---
title: "FastAPI×uv×Docker：最初からホストに .venv を作らせない構成"
date: 2026-06-03
tags: [Docker, uv, FastAPI, Python]
format: tutorial
---

## 概要

`docker compose` で FastAPI の開発環境を組むと、`./backend:/app` のようにソースを丸ごとマウントしたとき、コンテナ内で uv が作る `.venv` がホスト側に漏れて壊れるという問題が起きる。

よくある解決は「いったん `.venv` を作ってしまい、後から消す／匿名ボリュームでフタをする」というもの。しかし `.venv` は各環境が `uv sync` で組み立てる成果物にすぎず、ホストに共有すべきものではない。であれば、**そもそも一度も作らせなければ削除の手間も要らない**。

このチュートリアルでは、最小構成のセットアップから、ホスト側に `.venv` が一度も生まれない状態を作る。ポイントは 2 つ。

- セットアップ時は `uv add --no-sync` で `pyproject.toml` / `uv.lock`（ホストに残したいテキスト）だけ更新し、環境同期（= `.venv` 生成）はしない
- 実際の依存インストールはイメージビルド時に、`.venv` ではなくシステム Python（`/usr/local`）へ行う

完成後の状態:

- ホスト側 `backend/.venv` が一度も生成されない
- コンテナ内では依存がシステム Python に入る → `.venv` ディレクトリ自体ができない
- 移行用の「掃除ステップ」が不要

完成後のフォルダ構成:

```
.
├── docker-compose.yaml
└── backend/
    ├── .python-version
    ├── .gitignore
    ├── Dockerfile
    ├── README.md
    ├── main.py          ← FastAPI アプリ本体
    ├── pyproject.toml    ← 依存とプロジェクト定義
    └── uv.lock          ← 依存の固定
```

## 前提

- OS: Linux / WSL2（macOS でもほぼ同様）
- Docker がインストール済みで `docker run` / `docker compose` が使えること
- ホストに Python / uv は不要（パッケージ管理は uv をコンテナ経由で使う）
- 作業ディレクトリは任意（ここではプロジェクトのルートで作業する想定）

## ステップ 1: uv プロジェクトを作成する

`backend/` を uv プロジェクトとして作成する。`uv init` は雛形を配置するだけで `.venv` は作らない。

```bash
docker run --rm -it -v $(pwd):/app -w /app \
  astral/uv:python3.13-trixie \
  uv init backend
```

実行後、ホスト側に `backend/` 一式（`pyproject.toml` / `main.py` / `.python-version` / `.gitignore` / `README.md`）が生成される。この時点では `.venv` も `uv.lock` も無い。

## ステップ 2: FastAPI を依存に追加する（同期しない）

`--no-sync` を付けて依存を追加する。マウント先を `backend/` に変えている点に注意。

```bash
docker run --rm -it -v $(pwd)/backend:/app -w /app \
  astral/uv:python3.13-trixie \
  uv add --no-sync 'fastapi[standard]'
```

`--no-sync` の効果:

- `pyproject.toml` に `fastapi[standard]` を追記し、依存解決の結果として `uv.lock` を生成する
- 環境同期（インストール = `.venv` 生成）だけをスキップする

確認ポイント — ホストに `.venv` が無く、テキストだけ更新されていること。

```bash
$ ls backend/.venv
ls: cannot access 'backend/.venv': No such file or directory

$ cat backend/pyproject.toml
[project]
name = "backend"
version = "0.1.0"
description = "Add your description here"
readme = "README.md"
requires-python = ">=3.13"
dependencies = [
    "fastapi[standard]>=0.136.3",
]
```

`uv.lock` は生成される（依存解決は `--no-sync` でも走るため）。「何を入れるか」のテキストはここまでで揃い、「実際に入れる」工程はイメージビルドに先送りされる。

## ステップ 3: main.py を最小の FastAPI に書き換える

```python
# backend/main.py
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def read_root():
    return {"hello": "world"}
```

## ステップ 4: Dockerfile を書く

`backend/Dockerfile` を作成する。uv の環境変数で「`.venv` を作らず、システム Python（`/usr/local`）へ直接インストールする」よう指示するのが肝。

```dockerfile
# backend/Dockerfile
FROM astral/uv:python3.13-trixie

# uv に .venv を作らせず、システム Python に直接インストールさせる設定
ENV UV_PROJECT_ENVIRONMENT="/usr/local/"
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

各環境変数の役割:

| 環境変数 | 役割 |
|---|---|
| `UV_PROJECT_ENVIRONMENT="/usr/local/"` | venv の場所をシステム Python にする。実質 `.venv` を作らず直接インストールする（漏れ対策の本体） |
| `UV_SYSTEM_PYTHON=1` | uv が独自に Python を落とさず、イメージにある Python を使う（補助） |
| `UV_LINK_MODE=copy` | キャッシュから環境への展開を hardlink から copy に。別ファイルシステム間の警告を抑制（保険） |
| `UV_CACHE_DIR=/root/.cache/uv` | キャッシュ置き場を固定（ビルド高速化用） |

注意点:

- `ENV` の `=` の前後にスペースを入れない（`ENV KEY = "value"` は値が壊れる）
- `CMD` は `uv run fastapi dev ...`。`fastapi` を抜かさない

## ステップ 5: compose を書いてビルドする

`docker-compose.yaml` を作成する。`image:` で借りるのではなく Dockerfile を `build:` する。`.venv` は `/app` の外（システム側）にあるので、マウントの「フタ」（`- /app/.venv`）は不要。ソースのマウント 1 行だけでよい。

```yaml
# docker-compose.yaml
services:
  backend:
    build: ./backend
    ports:
      - 8000:8000
    volumes:
      - ./backend:/app
```

`working_dir` / `command` は Dockerfile 側（`WORKDIR` / `CMD`）に集約したので compose には書かない。

ビルドする。

```bash
docker compose build
```

## ステップ 6: 起動して動作確認

```bash
docker compose up
```

確認ポイント:

1. `http://localhost:8000` で `{"hello": "world"}` が返る
2. ホスト側 `backend/.venv` が一度も生成されていない

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

- `uv sync --frozen` でエラー → `uv.lock` が古い可能性。`uv add` / `uv lock` で更新してから再ビルドする
- hardlink 関連の警告が出る → `UV_LINK_MODE=copy` が設定されているか確認する
- `/usr/local` に入らない / Python が見つからない → ベースイメージの Python が `/usr/local` にあるか確認する。別の場所にある場合は `UV_PROJECT_ENVIRONMENT` をそのパスに合わせるか、`/opt/venv` のような専用パスを指定する

## おわりに

「ロックファイルはホスト、`.venv` はイメージ内」と役割を分けることで、ホスト側に `.venv` が生まれる瞬間が一度もなくなった。コンテナ自体が隔離境界なのだから、その中でさらに `.venv` を切る必要はない、という整理が核心になる。「作って消す」ではなく「最初から作らない」ので、移行用の掃除ステップも不要になる。

発展課題:

- 本番用にマルチステージビルド（`uv sync --no-dev` で開発依存を除く）
- pytest による最小テストと GitHub Actions での CI
- 依存をイメージに焼く構成のさらなる最適化
