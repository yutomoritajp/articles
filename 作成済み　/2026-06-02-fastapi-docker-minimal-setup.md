---
title: "Dockerだけで FastAPI の最小構成を作る"
date: 2026-06-02
tags: [FastAPI, Docker, uv, Python]
format: tutorial
---

## 概要

ホストマシンに Python も uv もインストールせず、Docker だけで FastAPI の開発環境を立ち上げる最小構成を作る。
完成すると次の状態になる。

- `http://localhost:8000` で `{"hello":"world"}` が返る
- `http://localhost:8000/docs` で Swagger UI が表示される
- `docker compose up` 一発で開発サーバーが起動する

同シリーズの [React (Vite) 編](https://zenn.dev/articles/49fa4f5509dcc1) と同じノリで、最終的に `docker compose` で frontend と backend を並べることも見据えている。

所要時間: 10〜15 分。

## 前提

- OS: Linux / WSL2（macOS でもほぼ同様）
- ツール: Docker がインストール済みで `docker run` が使えること
- ホストに Python / uv は **不要**
- 作業ディレクトリは任意（ここではプロジェクトのルートで作業する想定）

## パッケージ管理ツールの選択: uv

今回は [uv](https://docs.astral.sh/uv/) を採用する。理由：

- 公式 Docker イメージ `ghcr.io/astral-sh/uv:python3.13-bookworm-slim` が用意されており、`node:20` と同じ感覚で「借りるイメージ」として使える
- `uv init` → `uv add` → `uv run` の 3 コマンドで完結し、React 編の `npm create vite` / `npm run dev` と並べやすい
- `pyproject.toml` + `uv.lock` の 2 ファイルで依存関係が管理できる

## 完成後フォルダ構成

```
.
├── compose.yaml
└── backend/                 ← uv プロジェクト (Step1 で作成)
    ├── .python-version
    ├── .venv/               ← uv が作る仮想環境
    ├── README.md
    ├── main.py              ← FastAPI アプリ本体
    ├── pyproject.toml       ← 依存とコマンド定義
    └── uv.lock              ← 依存の固定
```

## ステップ 1: uv プロジェクトを作る

`backend/` を uv プロジェクトとして作成する。

```bash
docker run --rm -it -v $(pwd):/app -w /app \
  ghcr.io/astral-sh/uv:python3.13-bookworm-slim \
  uv init backend
```

各オプションの意味:

| 部分 | 意味 |
|---|---|
| `docker run --rm` | コンテナを起動し、終了したら即破棄する（`--rm`） |
| `-it` | 対話的に操作できるようにする |
| `-v $(pwd):/app` | ホストの現在のフォルダを、コンテナ内の `/app` に共有する |
| `-w /app` | コンテナ内の作業ディレクトリを `/app` にする |
| `ghcr.io/astral-sh/uv:python3.13-bookworm-slim` | uv + Python 3.13 入りの公式イメージを借りる |
| `uv init backend` | `backend/` フォルダを作り、`pyproject.toml` `main.py` などの雛形を配置する |

実行後、ホスト側に `backend/` 一式が生成される。

## ステップ 2: FastAPI を依存に追加する

`backend/` の中に入って FastAPI を依存として追加する。

```bash
docker run --rm -it -v $(pwd)/backend:/app -w /app \
  ghcr.io/astral-sh/uv:python3.13-bookworm-slim \
  uv add 'fastapi[standard]'
```

ポイント:

- `uv add` … 依存を `pyproject.toml` に追記し、`uv.lock` も更新する
- `fastapi[standard]` … FastAPI 本体に加え、`fastapi dev` CLI / `uvicorn` / `httpx` など開発に必要な周辺ツールを一式入れる extras
- マウント先を `backend/` に変えている点に注意（プロジェクト内で作業するため）

## ステップ 3: main.py を最小の FastAPI に書き換える

`uv init` で生成された `backend/main.py` は単純な Hello World スクリプトなので、最小の FastAPI アプリに置き換える。

```python
# backend/main.py
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def read_root():
    return {"hello": "world"}
```

## ステップ 4: 開発サーバーを起動する

```bash
docker run --rm -it -v $(pwd)/backend:/app -w /app -p 8000:8000 \
  ghcr.io/astral-sh/uv:python3.13-bookworm-slim \
  uv run fastapi dev main.py --host 0.0.0.0
```

### やっていること

1. ポートを外に出す（`-p 8000:8000`）
2. ホストからの接続を受け付ける（`--host 0.0.0.0`）
3. 開発サーバーの起動（`uv run fastapi dev main.py`）

補足:

- `uv run` … 「この pyproject の環境で実行する」コマンド。`.venv` が無ければその場で作って同期してから走らせる
- `fastapi dev` … FastAPI 公式 CLI のホットリロード付き開発サーバー。デフォルトは `127.0.0.1` バインドなので、コンテナ外から見るために `--host 0.0.0.0` を渡す必要がある

### 確認

ブラウザで以下にアクセスする。

- `http://localhost:8000` … `{"hello":"world"}` が返る
- `http://localhost:8000/docs` … Swagger UI が表示される

## ステップ 5: docker run を compose に落とし込む

ステップ 4 の `docker run` コマンドを `compose.yaml` にまとめる。

```yaml
# compose.yaml
services:
  backend:
    image: ghcr.io/astral-sh/uv:python3.13-bookworm-slim
    working_dir: /app
    volumes:
      - ./backend:/app
    ports:
      - "8000:8000"
    command: uv run fastapi dev main.py --host 0.0.0.0
```

### run コマンドとの対応関係

| `docker run` の部分 | compose の書き方 | 補足 |
|---|---|---|
| `ghcr.io/astral-sh/uv:...` | `image: ghcr.io/astral-sh/uv:...` | 借りるイメージ |
| `-w /app` | `working_dir: /app` | 作業ディレクトリ |
| `-v $(pwd)/backend:/app` | `volumes: - ./backend:/app` | `$(pwd)` が `.`（compose ファイルの場所基準）になる |
| `-p 8000:8000` | `ports: - "8000:8000"` | ポート公開 |
| `uv run fastapi dev main.py --host 0.0.0.0` | `command: ...` | 実行コマンド |
| `--rm` | （不要） | compose が管理。`docker compose down` で消す |
| `-it` | （不要） | `docker compose up` がログに繋いでくれる |

`docker compose up` で起動できる。

## ステップ 6 (おまけ): frontend と並べる

シリーズの最終目標である「React と FastAPI を `docker compose` でつなぐ」は、`services:` を 2 つ書くだけで実現できる。

```yaml
# compose.yaml
services:
  frontend:
    image: node:20
    working_dir: /app
    volumes:
      - ./frontend:/app
    ports:
      - "5173:5173"
    command: npm run dev -- --host

  backend:
    image: ghcr.io/astral-sh/uv:python3.13-bookworm-slim
    working_dir: /app
    volumes:
      - ./backend:/app
    ports:
      - "8000:8000"
    command: uv run fastapi dev main.py --host 0.0.0.0
```

`docker compose up` で 5173 と 8000 が同時に立ち上がる。

## 残課題

下記に思いつく残課題を記載する。不便さを感じたら対応したい。

- **ホットリロード**: `fastapi dev` は基本リロードあり。マウント越しのファイル変更検知が WSL2 等で不安定な場合は `--reload-dir` や polling を検討
- **`.venv` のホスト共有問題**: コンテナ内で作られる `.venv` は Linux バイナリなので、ホストの macOS / Windows からは使えない。`volumes: - backend-venv:/app/.venv` のように **`.venv` だけ named volume に逃がす** とホスト側ディレクトリが汚れない
- **本番用環境の作成**: `fastapi run` + multi-stage Dockerfile + `uv sync --frozen --no-dev` で別途構築

## おわりに

これで Docker だけで動く FastAPI の最小構成が手に入った。次は以下のような発展課題が考えられる。

- pytest による最小テストの追加
- Ruff / mypy など Lint・型チェックの設定
- GitHub Actions で CI を回す
- React (frontend) との API 連携（CORS 設定など）
