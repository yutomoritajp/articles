---
title: "postgres:18 の Docker イメージでデータ保存先が変わった件と起動エラーの対処"
date: 2026-06-02
tags: [postgres, docker, docker-compose, troubleshooting]
format: technical-memo
---

## 背景

`docker compose up -d` で `postgres:18` のコンテナを起動しようとしたところ、DB コンテナが起動直後に exit (1) で落ちる。`docker logs` を見ると、データ保存先のレイアウトに関する長いエラーメッセージが出ていた。原因と対処を記録する。

## エラーメッセージ

```
Error: in 18+, these Docker images are configured to store database data in a
       format which is compatible with "pg_ctlcluster" (specifically, using
       major-version-specific directory names).

       Counter to that, there appears to be PostgreSQL data in:
         /var/lib/postgresql/data (unused mount/volume)

       The suggested container configuration for 18+ is to place a single mount
       at /var/lib/postgresql which will then place PostgreSQL data in a
       subdirectory, allowing usage of "pg_upgrade --link" without mount point
       boundary issues.
```

## 原因：18 でデータ保存先の仕様が変わった

公式イメージの `postgres:18` から、データの置き場所と推奨マウント先が変更された。

| バージョン | マウント先 | `PGDATA`（データ実体） |
|---|---|---|
| 17 以下 | `/var/lib/postgresql/data` | `/var/lib/postgresql/data` |
| 18 以降 | `/var/lib/postgresql` | `/var/lib/postgresql/{MAJOR}/docker`（例: `/var/lib/postgresql/18/docker`） |

メジャーバージョン別のサブディレクトリにデータを置くことで、`pg_upgrade --link` をマウント境界の問題なく実行できるようにするための変更。`pg_ctlcluster` / `postgresql-common` の慣習に合わせた形。

注意点として、**17 以下で `/var/lib/postgresql` にマウントするとデータが永続化されない**（匿名ボリューム扱いになる）。バージョンとマウント先はセットで考える必要がある。

## エラー文の読み方

判断材料になる行は次の通り。

- `in 18+, these Docker images are configured ...` — 18+ で仕様が変わったという宣言。`major-version-specific directory names` が新方式。
- `Counter to that, there appears to be PostgreSQL data in: <パス> (unused mount/volume)` — **ここが核心**。指定したマウント先が 18 から見て間違った場所で、そこにあるデータは使われない、という意味。このパス表示が現状を表す。
- `The suggested container configuration for 18+ is to place a single mount at /var/lib/postgresql` — 解決策そのもの。

## 対処

### 1. マウント先を新仕様に直す

`docker-compose.yaml` のマウント先を `/var/lib/postgresql/data` から `/var/lib/postgresql` に変更する。

```yaml
  db:
    image: postgres:18
    volumes:
      - postgres_data:/var/lib/postgresql   # 旧: /var/lib/postgresql/data
```

### 2. 旧レイアウトのボリュームを削除する

マウント先だけ直しても、**既存ボリュームに旧レイアウト（マウント直下に `PG_VERSION` や `base/` がある形）でデータが残っている**と、今度は次のエラーになる。

```
Counter to that, there appears to be PostgreSQL data in:
  /var/lib/postgresql        ← パスがマウント先に変わっている
```

このパス表示が `/var/lib/postgresql/data` から `/var/lib/postgresql` に変わっていたら「マウントは直ったが中身が古い」サイン。開発用でデータを残す必要がなければ、ボリュームを削除してクリーンに初期化する。

```bash
docker compose down
docker volume rm sandbox_postgres_data   # ボリューム名は環境に合わせる
docker compose up -d
```

これで新仕様のレイアウトでデータが初期化され、正常に起動する。

## 既存データを残したまま 18 を使いたい場合（逃げ道）

データを消したくない場合は、環境変数で旧パスを明示すれば従来通り `:/var/lib/postgresql/data` マウントのまま動かせる。

```yaml
  db:
    image: postgres:18
    environment:
      PGDATA: /var/lib/postgresql/data
    volumes:
      - postgres_data:/var/lib/postgresql/data
```

## 参考

- [Docker Hub: postgres](https://hub.docker.com/_/postgres) — 「The defined `VOLUME` was changed in 18 and above to `/var/lib/postgresql`.」
- [docker-library/postgres PR #1259](https://github.com/docker-library/postgres/pull/1259) — 変更そのもの
- [docker-library/postgres issue #37](https://github.com/docker-library/postgres/issues/37) — 背景の議論
