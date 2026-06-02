---
title: "FastAPI + SQLModel に Alembic を導入してスキーマをマイグレーションする"
date: 2026-06-02
tags: [fastapi, alembic, sqlmodel, postgresql]
format: tutorial
---

## 概要

FastAPI 公式チュートリアルの `create_all` でテーブルを作る構成（[前回記事](2026-06-02-fastapi-postgresql-connection.md)）から一歩進め、**Alembic** でスキーマ変更をマイグレーション管理する。

`create_all` には「既存テーブルを変更できない（`ALTER TABLE` をしない）」「変更履歴が残らない」「ロールバックできない」という限界がある。Alembic を入れると、既にあるテーブルにカラムを追加し、その変更を revision として記録し、双方向（upgrade / downgrade）に行き来できるようになる。

このチュートリアルでは、既存の `hero` テーブルに `team` カラムを追加するマイグレーションを作成・適用・ロールバックするところまでを行う。所要時間の目安は 30〜45 分。

### create_all と Alembic の違い

| | create_all | Alembic |
|---|---|---|
| テーブル新規作成 | できる | できる |
| 既存テーブルの変更（カラム追加等） | **できない**（既存テーブルは無視） | できる |
| 変更の履歴 | 残らない | revision として 1 つずつ残る |
| 切り戻し（ロールバック） | 不可 | downgrade できる |

## 前提

- OS: macOS / Linux
- Docker / Docker Compose
- FastAPI + SQLModel + PostgreSQL が Docker Compose で動いている（[前回記事](2026-06-02-fastapi-postgresql-connection.md)の状態）
- `backend/main.py` に `Hero` モデルと `engine` が定義済み
- alembic コマンドはコンテナ内の uv 経由で実行する（`docker compose exec backend uv run alembic ...`）

## ステップ 1: alembic を依存に追加する

`backend/pyproject.toml` の `dependencies` に 1 行追加する。

```toml
dependencies = [
    "fastapi[standard]>=0.136.3",
    "sqlmodel>=0.0.38",
    "psycopg[binary]>=3.2",
    "alembic>=1.13",
]
```

依存を変更したら backend を再起動して再同期する。

```bash
docker compose restart backend
docker compose exec backend uv pip list | grep -i alembic
```

`alembic` がリストに出れば準備完了。

## ステップ 2: マイグレーション環境を作る

`alembic init` でマイグレーション環境一式を生成する。`working_dir` が `/app`（＝ホストの `./backend`）なので、`backend/` 配下に作られる。

```bash
docker compose exec backend uv run alembic init alembic
```

生成物:

```
backend/
  alembic.ini          # 設定ファイル（DB URL など）
  alembic/
    env.py             # 実行時スクリプト（autogenerate の心臓部）
    script.py.mako     # 生成されるマイグレーションの雛形
    versions/          # マイグレーションファイルが溜まる場所
```

> `docker compose exec` はコンテナ内で root として動くため、生成ファイルがホスト側で root 所有になることがある。編集時に permission denied が出たら所有権を確認する。

## ステップ 3: 3 つの設定ファイルを編集する

Alembic 公式は素の SQLAlchemy（`Base.metadata`）前提で書かれているため、SQLModel 用に読み替えながら 3 箇所を編集する。

### 3-1. alembic.ini — 接続先 URL（connection）

`alembic.ini` の `sqlalchemy.url` がダミー値なので、PostgreSQL の接続 URL に変更する。

```ini
sqlalchemy.url = postgresql+psycopg://postgres:postgres@db:5432/sample_db
```

`main.py` の接続 URL と同じ値。ホストは `localhost` ではなく Compose のサービス名 `db`。

### 3-2. env.py — target_metadata（autogenerate の心臓部）

`alembic/env.py` の `target_metadata = None` の周辺を SQLModel 用に書き換える。冒頭の import に 2 行追加し、`target_metadata` を設定する。

```python
from sqlmodel import SQLModel
from main import Hero  # noqa: F401  Heroをimportしてmetadataに登録させる

# ...

target_metadata = SQLModel.metadata
```

- `target_metadata = SQLModel.metadata` … 「あるべきスキーマの設計図」。autogenerate はこれと実 DB を突き合わせて差分を出す
- `from main import Hero` … **必須**。`Hero` を import して初めて metadata に hero テーブルが登録される。忘れると metadata が空になり、autogenerate が「テーブルが無い」と誤認して**既存テーブルを DROP するマイグレーションを生成**してしまう

> `env.py` から `from main import Hero` が解決できるのは、`alembic.ini` に `prepend_sys_path = .` があり `/app` が import パスに入っているため。

### 3-3. script.py.mako — import sqlmodel（生成ファイルの雛形）

`alembic/script.py.mako` の `import sqlalchemy as sa` の下に 1 行追加する。

```python
from alembic import op
import sqlalchemy as sa
import sqlmodel
${imports if imports else ""}
```

SQLModel の `str` カラムは内部で `sqlmodel.sql.sqltypes.AutoString` 型になる。autogenerate が生成するマイグレーションはこの型を参照するため、雛形に `import sqlmodel` を入れておかないと適用時に `NameError` になる（SQLModel + Alembic 定番の罠）。

## ステップ 4: モデルにカラムを追加する

差分の素として、`backend/main.py` の `Hero` に新しいカラムを 1 本足す。既存行があるため **NULL 許可**（`| None`、`default=None`）にする。NOT NULL だと既存行の値が決まらず失敗する。

```python
class Hero(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    name: str = Field(index=True)
    age: int | None = Field(default=None, index=True)
    secret_name: str
    team: str | None = Field(default=None)  # 追加（NULL許可）
```

## ステップ 5: マイグレーションを自動生成する

モデル（team あり）と実 DB（team なし）の差分から、マイグレーションを生成する。

```bash
docker compose exec backend uv run alembic revision --autogenerate -m "add team to hero"
```

出力に以下が出れば差分検出に成功:

```
INFO  [alembic.autogenerate.compare] Detected added column 'hero.team'
Generating /app/alembic/versions/xxxxxxxx_add_team_to_hero.py ...  done
```

このコマンドはファイルを書くだけで、まだ DB には適用しない。生成されたファイルの中身:

```python
from alembic import op
import sqlalchemy as sa
import sqlmodel  # ← script.py.mako に入れておいたので入る

def upgrade() -> None:
    op.add_column('hero', sa.Column('team', sqlmodel.sql.sqltypes.AutoString(), nullable=True))

def downgrade() -> None:
    op.drop_column('hero', 'team')
```

`upgrade()` が team を追加し、`downgrade()` が team を削除する。双方向に定義されている点が重要。

## ステップ 6: マイグレーションを適用する

```bash
docker compose exec backend uv run alembic upgrade head
```

`head` は最新リビジョンまで DB を進める指定。出力:

```
INFO  [alembic.runtime.migration] Running upgrade  -> xxxxxxxx, add team to hero
```

ここで 2 つのことが起きる:

1. `hero` テーブルに `team` カラムが `ALTER TABLE` で追加される（create_all にはできなかった操作）
2. `alembic_version` テーブルが作られ、適用済みリビジョンが記録される（履歴管理の実体）

## ステップ 7: 動作確認

### テーブル構造に team が増えたか

```bash
docker compose exec db psql -U postgres -d sample_db -c "\d hero"
```

カラム一覧に `team | character varying | nullable` が出れば成功。既存行は壊れず、`team` だけ NULL になっている。

### 適用済みリビジョンの記録

```bash
docker compose exec db psql -U postgres -d sample_db -c "SELECT * FROM alembic_version;"
```

生成したリビジョン ID が 1 行出れば、DB 自身が「今このリビジョンまで適用済み」と覚えている証拠。

## ステップ 8: ロールバックを試す

`downgrade()` を実行して 1 つ前のリビジョンに戻す。

```bash
docker compose exec backend uv run alembic downgrade -1
```

出力:

```
INFO  [alembic.runtime.migration] Running downgrade xxxxxxxx -> , add team to hero
```

`\d hero` で `team` カラムが消えていればロールバック成功。`alembic_version` も空（ベース状態）に戻る。再び前に進めたいときは:

```bash
docker compose exec backend uv run alembic upgrade head
```

同じマイグレーションを upgrade / downgrade で行き来できることが、`create_all` との決定的な差。

## トラブルシューティング

- **`NameError: name 'sqlmodel' is not defined`（upgrade 時）**
  生成されたマイグレーションが `sqlmodel.sql.sqltypes.AutoString` を参照しているのに、ファイル先頭に `import sqlmodel` が無い。`script.py.mako` に `import sqlmodel` を追加する（ステップ 3-3）。**注意**: 雛形を直しても、既に生成済みのファイルには反映されない。該当ファイルに手で `import sqlmodel` を足すか、ファイルを削除して再生成する（まだ適用していなければ削除して問題ない）。

- **autogenerate が既存テーブルを DROP するマイグレーションを生成する**
  `env.py` の `target_metadata` が `None` のまま、または `from main import Hero` を忘れている。モデルが metadata に登録されず「テーブルが無い」と誤認している。ステップ 3-2 を確認する。

- **autogenerate が空のマイグレーションを生成する（`upgrade` が `pass` だけ）**
  モデルと DB に差分が無い。既存テーブルにカラムを追加したいなら、先に `main.py` のモデルへカラムを足してから autogenerate する（ステップ 4）。

- **`ModuleNotFoundError: No module named 'sqlModel'`**
  import のスペルミス。パッケージ名は全部小文字の `sqlmodel`、クラス名は全部大文字の `SQLModel`。正しくは `from sqlmodel import SQLModel`。

- **生成ファイルが root 所有で編集できない**
  `docker compose exec` はコンテナ内で root として動くため。ホスト側で所有権を変更するか、コンテナ内から編集する。

## おわりに

`create_all` から Alembic に移行し、既存テーブルへのカラム追加・履歴記録・ロールバックができるようになった。SQLModel と組み合わせる際は「`target_metadata = SQLModel.metadata` ＋ モデルの import」「`script.py.mako` に `import sqlmodel`」の 2 点が定番のポイント。

次に試すと面白い発展課題:

- 接続 URL を `alembic.ini` 直書きから環境変数に寄せ、`main.py` との重複を解消する
- `create_all`（`on_startup`）を完全に外し、スキーマの所有権を Alembic に一本化する
- 履歴をゼロから再現可能にするため、`hero` テーブルの作成も含めた初期マイグレーションを用意する
- マイグレーションの命名規約（制約の naming convention）を設定し、`ALTER` 時の事故を防ぐ
