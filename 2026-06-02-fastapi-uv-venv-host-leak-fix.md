---
title: "FastAPI×uv×Docker：ホストに .venv を漏らさない構成にする（前回記事の続き）"
date: 2026-06-02
tags: [Docker, uv, FastAPI, VSCode]
format: tutorial
---

## 概要

[前回の記事](https://zenn.dev/yutomoritajp/articles/49fa4f5509dcc1)で、`docker run` から `docker compose` へ段階的に FastAPI の最小構成を組み立てた。その構成は「とりあえず動く」が、VSCode（WSL）で開くと **「インストールしたはずのライブラリが見つからない」という警告**が出る。

この記事では、その原因を理解したうえで、`Dockerfile` を導入して **`.venv` をホストに漏らさない構成**に作り替える。前回記事のゴールにあった「なるべくホスト側の環境に依存しないようにする」を、ここで初めて本当の意味で達成する。

完成後の状態:

- コンテナ内で `uv sync` した `.venv` が **イメージの中に閉じる**
- ホストの `backend/.venv` は空のまま → VSCode の誤警告が消える
- ソースコードはマウントで共有されるので、ホットリロードはそのまま効く

## 前提

- [前回記事](https://zenn.dev/yutomoritajp/articles/49fa4f5509dcc1)のステップ1〜5まで完了していること（`backend/` に `pyproject.toml` / `uv.lock` / `main.py` があり、`compose.yaml` で開発サーバーが起動する状態）
- OS: Linux / WSL2（macOS でもほぼ同様）
- VSCode を WSL（または Remote）で開いている
- ホストに Python / uv は不要

## なぜ警告が出るのか

### 症状

VSCode で `from fastapi import FastAPI` に警告が付き、補完も効かない。「`pip install` したのに見つからない」状態。

### 原因：`.venv` がコンテナ専用品なのにホストへ漏れている

前回の `compose.yaml` はこうなっていた。

```yaml
volumes:
  - ./backend:/app
```

これは「ホストの `./backend` を、コンテナの `/app` に**丸ごと共有**する」設定。便利な一方で、**コンテナ内で `uv` が作った `.venv` も、この共有を通じてホスト側 `backend/.venv` に書き出される**。

`.venv` の中身を見ると問題が分かる。

```bash
$ cat backend/.venv/pyvenv.cfg
home = /usr/local/bin
version_info = 3.14.5

$ ls -la backend/.venv/bin/python
backend/.venv/bin/python -> /usr/local/bin/python3.14
```

`.venv/bin/python` は実体ではなく **コンテナ内のパス `/usr/local/bin/python3.14` を指すシンボリックリンク**。ところがホスト（WSL）にそんなパスは存在しない。

```bash
$ ls /usr/local/bin/python3.14
ls: cannot access '/usr/local/bin/python3.14': No such file or directory
```

→ リンク先が迷子になり、VSCode が「Python が見つからない＝ライブラリも見つからない」と警告する。`.venv` はそもそも **作られた環境に絶対パスを焼き込む**ため、別環境へ持ち出すと壊れる。コンテナで作ったものをホストで使い回すのが間違いだった。

### 考え方：`.venv` は成果物、`pyproject.toml` / `uv.lock` がソース

整理するとこうなる。

| ファイル | 役割 | 扱い |
|---|---|---|
| `pyproject.toml` / `uv.lock` | 「何を入れるか」のレシピ（**ソース**） | Git 管理する。OS非依存のテキスト |
| `.venv` | レシピから組み立てた**成果物** | 各環境が `uv sync` で再生成。共有・コミットしない |

`uv` はどの環境でも `uv sync`（`uv run` でも自動実行）で `uv.lock` を読み、**その環境用の `.venv` を作り直す**。だから `.venv` を持ち回る必要はゼロ。前回構成は「成果物（.venv）を作って共有ストレージに残す」形になっていた点が、本質的な誤りだった。

> 方針：**`.venv` はイメージの中で作って閉じ込める。ホストにはソース（コード）だけを共有する。**

これを実現するために `Dockerfile` を導入する。

## ステップ6：Dockerfile を作る

`backend/Dockerfile` を作成する。

```dockerfile
# backend/Dockerfile
FROM astral/uv:python3.14-trixie

WORKDIR /app

# 依存リスト(レシピ)だけ先にコピー → レイヤーキャッシュを効かせる
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen

# アプリ本体をコピー
COPY . .

# .venv はこのイメージ内に焼き込まれる（ホストには漏れない）
CMD ["uv", "run", "fastapi", "dev", "main.py", "--host", "0.0.0.0"]
```

### COPY の向きに注意

`Dockerfile` の `COPY` は **必ず「ホスト → イメージ」の一方向**。`compose.yaml` の `volumes`（実行時にホストと双方向共有）とは別物。

| 仕組み | 向き | タイミング |
|---|---|---|
| Dockerfile の `COPY` | ホスト → イメージ（コピー） | **ビルド時**。以後ホストと独立 |
| compose の `volumes` | ホスト ⇄ コンテナ（同じ実体を共有） | **実行時**。変更が双方向に反映 |

`.venv` がホストに漏れたのは後者（`volumes`）が原因。Dockerfile の `COPY`＋`uv sync` で作る `.venv` は前者なので**イメージに閉じてホストへ漏れない**。これが修正の核心。

### なぜ COPY を2回に分けるのか（レイヤーキャッシュ）

`COPY . .` を先にやって `uv sync` を後に書くと、**`main.py` を1文字直すたびに依存を全部入れ直す**ことになる。Docker は各行（レイヤー）をキャッシュし、**ある行の入力が変わるとそれ以降を全部再実行する**ためだ。

そこで「**変わりにくいもの（依存リスト）を先に、変わりやすいもの（ソース）を後に**」COPY する。

```dockerfile
COPY pyproject.toml uv.lock ./   # 依存リストが変わらなければ…
RUN uv sync --frozen             # …ここはキャッシュ流用（重いインストールをスキップ）
COPY . .                         # main.py を直したらここだけ再実行
```

`uv add` で新しいライブラリを足して `uv.lock` が変わったときだけ `uv sync` が走り直す。理にかなったタイミングだけで再ビルドされる。

| 部分 | 意味 |
|---|---|
| `FROM astral/uv:python3.14-trixie` | uv + Python 3.14 入りの公式イメージを土台にする |
| `WORKDIR /app` | 作業ディレクトリを `/app` に設定 |
| `COPY pyproject.toml uv.lock ./` | レシピだけ先にコピー（キャッシュ目的） |
| `RUN uv sync --frozen` | `uv.lock` の通りに `.venv` を作る。`--frozen` はロックを書き換えない（再現性確保） |
| `COPY . .` | 残りのソースをコピー |
| `CMD [...]` | 開発サーバーの起動コマンド |

## ステップ7：.dockerignore で .venv を除外する

`COPY . .` のとき、**ホストに残っている壊れた `.venv` までイメージに入り込み、せっかく `uv sync` で作った正しい `.venv` を上書きしてしまう**。これを防ぐため `backend/.dockerignore` を作る。

```gitignore
# backend/.dockerignore
# 環境固有の成果物はイメージに持ち込まない
.venv
__pycache__
*.py[oc]
.git
```

これで `COPY . .` はソースコードだけを持ち込み、`.venv` はステップ6でビルドした中身が保たれる。

## ステップ8：compose.yaml を build に切り替える

`image:` で借りるのをやめ、ステップ6の `Dockerfile` を `build:` する。さらに **`- /app/.venv` で `.venv` をマウントからくり抜く**。

```yaml
# compose.yaml（backend サービスの差分）
services:
  backend:
    build: ./backend          # image: → build: に変更
    depends_on:
      - db
    volumes:
      - ./backend:/app         # ソースはホットリロード用に共有
      - /app/.venv             # ★ .venv はイメージ内のものを使い、ホストと切り離す
    ports:
      - "8000:8000"
    # command と working_dir は Dockerfile 側(CMD/WORKDIR)に集約したので削除
```

### `- /app/.venv` が効く理由

| 書き方 | 効果 |
|---|---|
| `./backend:/app` | ホストの `backend` をコンテナ `/app` に共有（双方向） |
| `/app/.venv` | `/app/.venv` だけ**匿名ボリューム**で覆う。より深いパスのマウントが優先され、`.venv` だけホスト共有から切り離される。中身はイメージの `.venv` で初期化される |

結果、`/app` 直下のソースはホストと共有されつつ、`/app/.venv` だけは**イメージが持つ正しい `.venv`** が使われる。ホスト側 `backend/.venv` は空のままになる。

## ステップ9：ホストに残った壊れた .venv を掃除する

これまでのマウント運用で、ホストには `root` 所有の `.venv` / `__pycache__` が残っている。これらはコンテナが書いたものなのでホストユーザーでは消せないことがある。**使い捨てコンテナを root として動かして掃除**する。

```bash
docker run --rm -v $(pwd)/backend:/app -w /app \
  alpine \
  rm -rf .venv __pycache__
```

`.gitignore` に `.venv` と `__pycache__` が入っていることも確認しておく（`uv init` の雛形には既に入っている）。

```gitignore
# backend/.gitignore（抜粋）
__pycache__/
*.py[oc]
.venv
```

## ステップ10：動作確認

ビルドし直して起動する。古い `.venv` ボリュームが残っているなら `-v` で消してからにする。

```bash
docker compose down -v
docker compose up --build
```

確認ポイント:

1. `http://localhost:8000` で `{"hello": "world"}` が返る（前回と同じく動く）
2. ホスト側 `backend/.venv` が**空、または存在しない**

```bash
$ ls backend/.venv
# 何も出ない、または No such file or directory
```

VSCode 側の警告については、構成に応じて次のどちらかになる。

- **コンテナ内で開発するなら**：「Reopen in Container」で VSCode をコンテナ側に入れる。エディタとランタイムが同じ環境＝同じ `.venv` を見るので、補完も型チェックも完全に効く（こちらが本筋）
- **WSL で補完だけ効かせたいなら**：コンテナとは別に WSL 内にも開発用 `.venv` を作る（`uv sync` を WSL で実行）。ただし二重管理になる

いずれにせよ、**ホストにコンテナ専用の壊れた `.venv` が漏れる**という今回の問題は解消する。

## トラブルシューティング

- **`uv sync --frozen` でエラー** → `uv.lock` が古い可能性。ホストで `uv lock`（または `uv add` で更新）してから再ビルドする
- **再ビルドしたのに古い `.venv` が使われる** → 匿名ボリュームが残っている。`docker compose down -v` でボリュームごと削除してから `up --build`
- **`COPY . .` で `.venv` が混入する** → `backend/.dockerignore` に `.venv` が入っているか確認
- **`backend/.venv` をホストで消せない（Permission denied）** → ステップ9のとおり `docker run --rm ... alpine rm -rf` で消す

## おわりに

前回の「とりあえず動く」構成に対して、

- `.venv` は成果物、`pyproject.toml` / `uv.lock` がソース、という整理
- `Dockerfile` で `.venv` をイメージに焼き込む（レイヤーキャッシュの定石つき）
- `.dockerignore` と匿名ボリュームで `.venv` をホストから切り離す

を足したことで、「ホスト側の環境に依存しない」という当初のゴールがようやく完成した。

次の発展課題:

- 本番用にマルチステージビルド（`uv sync --no-dev` で開発依存を除く）
- Dev Container（`.devcontainer/devcontainer.json`）でエディタごとコンテナに入れる
- pytest による最小テストと GitHub Actions での CI
