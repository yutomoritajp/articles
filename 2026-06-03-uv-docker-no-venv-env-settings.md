---
title: "FastAPI×uv×Docker：環境変数で .venv を作らせない構成にする"
date: 2026-06-03
tags: [Docker, uv, FastAPI, Python]
format: tutorial
---

## 概要

`docker compose` で FastAPI の開発環境を組むと、`./backend:/app` のようにソースを丸ごとマウントしたとき **コンテナ内で uv が作った `.venv` がホスト側に漏れて壊れる**という問題が起きる。

この記事では、その問題を **uv の環境変数を Dockerfile に加えるだけ**で根本から解決する。`.venv` をマウントから「フタ」で隠すような小細工（`- /app/.venv` の匿名ボリューム）を使わず、**そもそも `/app` の中に `.venv` を作らせない**方針をとる。

完成後の状態:

- コンテナ内では依存がシステム Python（`/usr/local`）に直接入る → `.venv` ディレクトリ自体ができない
- マウントしてもホスト側 `backend/.venv` は生成されない → 漏れようがない
- 匿名ボリュームの「フタ」も、古い `.venv` の掃除も不要

## 前提

- `backend/` に `pyproject.toml` / `uv.lock` / `main.py` があり、`docker compose` で FastAPI 開発サーバーが起動する状態
- OS: Linux / WSL2（macOS でもほぼ同様）
- ベースイメージは Python が `/usr/local` にある uv 公式系イメージ（`astral/uv:python3.x-*` 等）
- ホストに Python / uv は不要

## なぜ `.venv` がホストに漏れると困るのか

`compose.yaml` でソースをこう共有しているとする。

```yaml
volumes:
  - ./backend:/app
```

これは「ホストの `./backend` とコンテナの `/app` を双方向で共有する」設定。便利な一方で、コンテナ内で uv が作る `.venv` も、この共有を通じてホスト側 `backend/.venv` に書き出される。

`.venv` はその場の環境に**絶対パスを焼き込む**成果物で、`.venv/bin/python` はコンテナ内のパス（例: `/usr/local/bin/python3.x`）を指すシンボリックリンクになる。ホスト（WSL）にそのパスは無いのでリンクが迷子になり、VSCode が「Python が見つからない＝ライブラリも見つからない」と警告する。

整理すると、`pyproject.toml` / `uv.lock` が「何を入れるか」のソース（OS非依存のテキスト）であり、`.venv` はそこから各環境が `uv sync` で組み立てる成果物にすぎない。**成果物である `.venv` をホストとコンテナで共有しようとしたこと自体が間違い**だった。

## 考え方：コンテナの中で二重に隔離しない

よくある対処は、マウントに匿名ボリュームを重ねて `.venv` だけ共有から切り離す方法。

```yaml
volumes:
  - ./backend:/app
  - /app/.venv   # ← .venv だけ匿名ボリュームでフタをする
```

これは動くが、「より深いパスのマウントが優先される」「匿名ボリュームはイメージの中身で初期化される」という Docker の暗黙挙動に依存していて分かりにくく、匿名ボリュームが残って「再ビルドしても古い `.venv` が使われる」という地雷もついてくる。

そこで発想を変える。**コンテナ自体がすでに隔離環境なのだから、その中でさらに `.venv` で隔離する必要はない。** 依存をシステム Python に直接入れてしまえば、`/app` の中に `.venv` は生まれず、マウントしても漏れようがない。これを uv の環境変数で指示する。

## ステップ 1: Dockerfile に uv の環境変数を加える

`backend/Dockerfile` に、以下の環境変数を追加する。

```dockerfile
# backend/Dockerfile
FROM astral/uv:python3.13-bookworm-slim

# uv に .venv を作らせず、システム Python に直接インストールさせる設定
ENV UV_PROJECT_ENVIRONMENT=/usr/local/
ENV UV_SYSTEM_PYTHON=1
ENV UV_LINK_MODE=copy
ENV UV_CACHE_DIR=/root/.cache/uv

WORKDIR /app

# 依存リスト(レシピ)だけ先にコピー → レイヤーキャッシュを効かせる
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen

# アプリ本体をコピー
COPY . .

CMD ["uv", "run", "fastapi", "dev", "main.py", "--host", "0.0.0.0"]
```

### 各環境変数の意味

| 環境変数 | 役割 | 漏れ対策に効くか |
|---|---|---|
| `UV_PROJECT_ENVIRONMENT=/usr/local/` | venv の場所をシステム Python（`/usr/local`）にする。実質 `.venv` を作らずシステムに直接インストールする | ◎ これが本体 |
| `UV_SYSTEM_PYTHON=1` | uv が独自に Python を落とさず、イメージにある Python を使う | 補助 |
| `UV_LINK_MODE=copy` | キャッシュから環境へ展開する方式を hardlink から copy に。別ファイルシステム間で出る警告を抑制する | 直接は無関係（実用上の保険） |
| `UV_CACHE_DIR=/root/.cache/uv` | キャッシュ置き場を固定する。キャッシュ用ボリュームを当てやすくする | 無関係（ビルド高速化用） |

漏れ対策の本体は `UV_PROJECT_ENVIRONMENT` であり、`UV_SYSTEM_PYTHON` はその補助。残り 2 つは高速化・警告抑制のためのおまけと割り切ってよい。

### 補足：`/opt/venv` 版との違い

「venv を作らない」のではなく「venv を `/app` の外に出す」だけでも漏れは防げる。

```dockerfile
ENV UV_PROJECT_ENVIRONMENT=/opt/venv
```

こちらは venv の境界が残る分わかりやすい。一方 `/usr/local/` 版は venv 自体を作らない最小構成で、より Docker らしい。どちらも「`/app` の中に `.venv` を作らない」点で漏れ対策としては同等。本記事では後者を採用する。

## ステップ 2: compose.yaml を build に切り替える

`image:` で借りるのをやめ、ステップ 1 の Dockerfile を `build:` する。`.venv` は `/app` の外（システム側）にあるので、**マウントの「フタ」（`- /app/.venv`）は不要**になる。

```yaml
# compose.yaml（backend サービスの差分）
services:
  backend:
    build: ./backend        # image: → build: に変更
    volumes:
      - ./backend:/app       # ソースはホットリロード用に共有。これだけでよい
    ports:
      - "8000:8000"
    # command / working_dir は Dockerfile 側(CMD/WORKDIR)に集約したので削除
```

## ステップ 3: ホストに残った壊れた .venv を掃除する

これまでのマウント運用でホスト側に `root` 所有の `.venv` / `__pycache__` が残っている場合、ホストユーザーでは消せないことがある。使い捨てコンテナを root として動かして掃除する。

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

## ステップ 4: 動作確認

ビルドし直して起動する。

```bash
docker compose up --build
```

確認ポイント:

1. `http://localhost:8000` で `{"hello": "world"}` が返る
2. ホスト側 `backend/.venv` が**生成されない**

```bash
$ ls backend/.venv
ls: cannot access 'backend/.venv': No such file or directory
```

3. コンテナ内では依存がシステムに入っている

```bash
$ docker compose exec backend python -c "import fastapi; print(fastapi.__file__)"
/usr/local/lib/python3.13/site-packages/fastapi/__init__.py
```

`.venv` ではなく `/usr/local/lib/.../site-packages` から読まれていれば成功。

## トラブルシューティング

- **`uv sync --frozen` でエラー** → `uv.lock` が古い可能性。`uv lock`（または `uv add`）で更新してから再ビルドする
- **hardlink 関連の警告が出る** → `UV_LINK_MODE=copy` が設定されているか確認する
- **`/usr/local` に入らない / Python が見つからない** → ベースイメージの Python が `/usr/local` にあるか確認する。別の場所にある場合は `UV_PROJECT_ENVIRONMENT` をそのパスに合わせるか、`/opt/venv` のような専用パスを指定する

## おわりに

`.venv` を匿名ボリュームで隠す小細工に頼らず、**uv の環境変数で「そもそも `/app` に `.venv` を作らせない」**ことで、ホストへの漏れを根本から断った。コンテナを隔離境界とみなし、その中で二重に venv を切らない、という整理が核心になる。

発展課題:

- 本番用にマルチステージビルド（`uv sync --no-dev` で開発依存を除く）
- VSCode の補完を効かせたい場合は Dev Container（Reopen in Container）でエディタごとコンテナに入る
- pytest による最小テストと GitHub Actions での CI
