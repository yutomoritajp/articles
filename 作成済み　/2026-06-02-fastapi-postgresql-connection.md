---
title: "FastAPI を PostgreSQL に接続する（SQLModel + Docker Compose）"
date: 2026-06-02
tags: [fastapi, postgresql, sqlmodel, docker]
format: tutorial
---

## 概要

FastAPI 公式チュートリアル [SQL (Relational) Databases](https://fastapi.tiangolo.com/tutorial/sql-databases/) は SQLite を使っているが、これを **PostgreSQL** に置き換えて接続するハンズオン。SQLModel でモデルを定義し、起動時に `create_all` でテーブルを自動生成して、`POST /heroes/` で実際にデータを書き込むところまでを行う。

Docker Compose で FastAPI（uv 管理）と PostgreSQL 18 を立ち上げる構成。所要時間の目安は 30〜45 分。

完成後、起動ログに以下が出れば接続成功:

```
INFO   Application startup complete.
INFO   Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
```

> このチュートリアルは「テーブルを `create_all` で作る」最短ルート版（公式準拠）。スキーマ変更履歴を管理する Alembic への発展は次回の題材とする。

## 前提

- OS: macOS / Linux
- Docker / Docker Compose
- 公式チュートリアルとの差分は「Create an Engine」セクションのみ。それ以外は公式コードをそのまま流用できる
- ローカルに Python・psql・uv は不要（すべてコンテナ内で完結）

## プロジェクト構成

```
sandbox/
├── docker-compose.yaml
├── backend/
│   ├── pyproject.toml
│   └── main.py
└── db/
    ├── .env
    └── data/        # Postgres のデータ永続化先
```

`db/.env`:

```
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=sample_db
```

## ステップ 1: docker-compose.yaml を用意する

backend（uv イメージで FastAPI を dev 起動）と db（postgres:18）を定義する。`depends_on` の `service_healthy` で、DB が受け付け可能になるまで backend の起動を待たせるのがポイント。

```yaml
services:
  backend:
    image: ghcr.io/astral-sh/uv:python3.13-bookworm-slim
    working_dir: /app
    ports:
      - 8000:8000
    volumes:
      - ./backend:/app
    command: uv run fastapi dev main.py --host 0.0.0.0
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:18
    env_file:
      - ./db/.env
    ports:
      - 5432:5432
    volumes:
      - ./db/data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 1m
      timeout: 30s
      retries: 5
```

## ステップ 2: 依存パッケージを宣言する

`backend/pyproject.toml` の `dependencies` に、SQLModel と PostgreSQL ドライバ（psycopg3）を追加する。

```toml
[project]
name = "backend"
version = "0.1.0"
requires-python = ">=3.13"
dependencies = [
    "fastapi[standard]>=0.136.3",
    "sqlmodel>=0.0.38",
    "psycopg[binary]>=3.2",
]
```

- `sqlmodel`: モデル定義 + ORM。内部で SQLAlchemy を使う
- `psycopg[binary]`: PostgreSQL ドライバ。`[binary]` 付きはビルド済みバイナリが入るのでコンパイル不要

> 公式チュートリアル（SQLite）には `psycopg` の追加は登場しない。PostgreSQL は外部ドライバが必須なため、ここが公式との差分になる。

## ステップ 3: main.py を書く

公式の各セクション（Create Models / Create an Engine / Create the Tables / Create a Session Dependency / Create Database Tables on Startup / Create a Hero）を順に組み立てる。**接続 URL 以外は公式コードとほぼ同一**。

```python
from typing import Annotated
from fastapi import FastAPI, Depends
from sqlmodel import Field, Session, SQLModel, create_engine, select

app = FastAPI()

@app.on_event("startup")
def on_startup():
    create_db_and_tables()

postgres_url = "postgresql+psycopg://postgres:postgres@db:5432/sample_db"
engine = create_engine(postgres_url)

class Hero(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    name: str = Field(index=True)
    age: int | None = Field(default=None, index=True)
    secret_name: str

def create_db_and_tables():
    SQLModel.metadata.create_all(engine)

def get_session():
    with Session(engine) as session:
        yield session

SessionDep = Annotated[Session, Depends(get_session)]

@app.post("/heroes/")
def create_hero(hero: Hero, session: SessionDep) -> Hero:
    session.add(hero)
    session.commit()
    session.refresh(hero)
    return hero
```

### 接続 URL の構造

```
postgresql+psycopg://postgres:postgres@db:5432/sample_db
```

| 部分 | 値 | 意味 |
|---|---|---|
| ドライバ | `postgresql+psycopg` | PostgreSQL を psycopg3 で接続する指定 |
| 認証 | `postgres:postgres` | `db/.env` の USER:PASSWORD |
| ホスト | `db` | **Compose のサービス名**（`localhost` ではない） |
| ポート | `5432` | |
| DB 名 | `sample_db` | `db/.env` の POSTGRES_DB |

最大のポイントは**ホスト名が `db`** であること。コンテナ同士は Compose ネットワーク内でサービス名を DNS 名として解決する。`localhost` と書くと backend コンテナ自身を指してしまい接続に失敗する。

### 公式（SQLite）からの読み替え

| | 公式（SQLite） | 本記事（PostgreSQL） |
|---|---|---|
| 接続 URL | `sqlite:///database.db` | `postgresql+psycopg://...@db:5432/sample_db` |
| `connect_args` | `{"check_same_thread": False}` が必要 | **不要（削除）** |

`check_same_thread` は SQLite 固有のスレッド制約を回避するための引数。PostgreSQL には無関係なので付けない。

### コードの要点

- `class Hero(SQLModel, table=True)`: `table=True` を付けたクラスが「DB テーブルになるモデル」。Pydantic の検証情報（`Field(index=True)` 等）と DB のカラム定義を 1 つの宣言に同居させられるのが SQLModel の特徴
- `SQLModel.metadata.create_all(engine)`: `Hero` 定義時に裏で登録された metadata（テーブルの台帳）を一括 CREATE する。引数に `Hero` を渡さなくても拾ってくれる
- `get_session()`: リクエストごとに Session を開いて `yield` で貸し出し、`with` が自動で後始末する。`SessionDep` を引数に書くだけで Session が刺さる
- `create_hero`: `add`（追加予約）→ `commit`（実際に INSERT）→ `refresh`（DB が採番した id をオブジェクトへ反映）の 3 点セットが書き込みの基本形

> `@app.on_event("startup")` は公式チュートリアル通りだが、最近の FastAPI では非推奨（deprecated）。起動時に `DeprecationWarning` が出るが動作する。`lifespan` への書き換えは発展課題。

## ステップ 4: 起動する

```bash
docker compose up -d
```

backend コンテナの `uv run` が起動時に `pyproject.toml` を見て依存を自動同期し、サーバーを立ち上げる。ログを確認:

```bash
docker compose logs backend --tail 20
```

以下が出れば成功。起動時に `on_startup` → `create_all` が走り、PostgreSQL に接続して `hero` テーブルが作られている:

```
Installed N packages
...
INFO   Application startup complete.
INFO   Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
```

> 依存を変更した後にコンテナが起動済みの場合は、`docker compose restart backend` で `uv run` を再実行させると再同期される。「pyproject を変えたら backend を restart」が基本リズム。

## ステップ 5: 動作確認

### テーブルができているか（Postgres 側を直接確認）

```bash
docker compose exec db psql -U postgres -d sample_db -c "\dt"
```

`hero` テーブルが一覧に出れば `create_all` 成功。

### API 経由でデータを書き込む

```bash
curl -X POST localhost:8000/heroes/ \
  -H "Content-Type: application/json" \
  -d '{"name":"Deadpond","secret_name":"Dive Wilson"}'
```

`id` が採番された JSON が返れば、`add → commit → refresh` が動いた証拠。ブラウザで http://localhost:8000/docs を開けば Swagger UI から同じことを試せる。

### 本当に DB に入ったか（Postgres 側で確認）

```bash
docker compose exec db psql -U postgres -d sample_db -c "SELECT * FROM hero;"
```

作成した行が見えれば、**API → PostgreSQL の往復が成立**している。

## トラブルシューティング

- **`ModuleNotFoundError: No module named 'psycopg'`**
  接続 URL に `+psycopg` と書くと、SQLAlchemy は `create_engine` の時点で `import psycopg` を実行してドライバを確保しようとする。接続(`connect`)より手前のエンジン生成段階で必要になるため、ドライバ未導入だとここで落ちる。`pyproject.toml` に `psycopg[binary]` を追加して再同期する（ステップ 2）。

- **`pyproject.toml` を編集したのに反映されない**
  コンテナが起動済みだと、走り続けている `uv run` は追記分を同期しない。`docker compose up -d` だけでは設定（image/volume/env）が変わらない限りコンテナを作り直さないため再同期も起きない。`docker compose restart backend` で `uv run` を再実行させる。

- **`pip list` に依存パッケージが出てこない（pip しか表示されない）**
  uv は専用の仮想環境 `/app/.venv` に依存を入れる。素の `pip` はその外側のシステム Python を見ているため、両者は別環境。確認は uv 経由で行う:
  ```bash
  docker compose exec backend uv pip list
  ```
  このコンテナでは `python` ではなく `uv run python`、`pip` ではなく `uv pip` を使う。

- **`Failed to hardlink files; falling back to full copy`**
  キャッシュとコピー先が別ファイルシステムのときに出る性能警告。動作には影響しない。抑制したい場合は `UV_LINK_MODE=copy` を設定する。

- **DB に接続できない（接続拒否）**
  接続 URL のホストが `localhost` になっていないか確認する。コンテナ間は Compose のサービス名 `db` で解決する。

## おわりに

公式チュートリアルの SQLite 構成から「Create an Engine」セクションだけを読み替えることで、FastAPI を PostgreSQL に接続し、`create_all` でテーブルを生成し、`POST /heroes/` でデータを書き込めるようになった。

次に試すと面白い発展課題:

- 残りの CRUD（`GET /heroes/` 一覧 / `GET /heroes/{id}` 単体 / `DELETE /heroes/{id}`）を公式に沿って実装する
- モデルを `HeroBase` / `HeroPublic` / `HeroCreate` / `HeroUpdate` に分離する（公式後半「Multiple Models」）
- 接続 URL を直書きから環境変数（`DATABASE_URL`）に移す
- `@app.on_event("startup")` を `lifespan` ハンドラに書き換える
- `create_all` を卒業して **Alembic** でスキーマ変更をマイグレーション管理する
