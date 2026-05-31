---
title: "docker compose の healthcheck で環境変数が空になる — $ と $$ の話"
date: 2026-05-31
tags: [docker, docker-compose, postgresql, healthcheck]
---

## はじめに

docker compose で PostgreSQL の `healthcheck` を書いたとき、`pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}` のように環境変数を埋め込んだら、なぜか変数が空文字になってヘルスチェックが永遠に通らない、という現象に出くわした。

不思議なのは、同じ `env_file` で渡している変数が、`db` サービスの `environment` にはちゃんと値が入って見えること。「値はあるのに空になる」という一見矛盾した状況だった。

結論から言うと、**docker compose には変数を展開するタイミングが2回あり、それぞれ参照する場所が違う**。これを理解すると現象がきれいに説明でき、`$$`（ドル記号2つ）で解決できる。同じ罠は healthcheck に限らず、compose ファイル内でシェルコマンドを書くところ全般で踏みうる。

## 問題の構成

最初に書いていた `docker-compose.yaml`（抜粋）はこうだった。

```yaml
db:
  image: postgres:17
  env_file: ./db/.env
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
    interval: 10s
    timeout: 5s
    retries: 5
```

`./db/.env` には接続情報を定義している。

```bash
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=dbsp_mon
```

`docker compose config`（展開後の最終的な設定を表示するコマンド）で確認すると、次のようになった。

```
WARN[0000] The "POSTGRES_USER" variable is not set. Defaulting to a blank string.
WARN[0000] The "POSTGRES_DB" variable is not set. Defaulting to a blank string.
...
  db:
    environment:
      POSTGRES_DB: dbsp_mon
      POSTGRES_PASSWORD: postgres
      POSTGRES_USER: postgres
    healthcheck:
      test:
        - CMD-SHELL
        - 'pg_isready -U  -d '      # ← 変数が消えて空になっている
```

`environment` には値が入っているのに、healthcheck の中だけ `pg_isready -U  -d ` と空になっている。この一見矛盾した出力が、原因の全てを物語っていた。

## 変数展開は2回ある

ポイントは、`CMD-SHELL` を使った healthcheck のコマンド文字列は、最終的に **コンテナの中のシェル（`/bin/sh -c "..."`）** に渡されて実行される、ということ。つまり「変数を展開しうる登場人物」が2人いる。

1. **Compose（ホスト側）** — ファイルを読むときに `${VAR}` を見つけたら展開しようとする。参照先は「ホストのシェル環境変数」または「compose ファイルと同じ階層にある `.env`」
2. **コンテナ内のシェル** — 渡された文字列の中の `$VAR` を、実行時にコンテナの環境変数で展開する

`env_file:` が注入するのは **コンテナの環境変数**。つまり 2 番の世界の住人だ。ところが `${POSTGRES_USER}` と普通に書くと、ファイル読み込み時に **1 番の Compose が先に食べてしまう**。ホスト側には `POSTGRES_USER` が無いので空文字になり、コンテナ内シェルに届く前に消える。これが `pg_isready -U  -d ` の正体であり、`variable is not set` 警告の意味だった。

一方 `environment:` に値が出ていたのは、`env_file` が正しくコンテナ用の環境変数として読み込まれた結果。**届く先（コンテナ）と、`${...}` の展開元（ホスト）が別物**なので、片方だけ空になる。矛盾ではなかった。

| 項目 | 何の値か | 展開する人・タイミング | 結果 |
|---|---|---|---|
| `environment:` | env_file の中身 | コンテナ起動時 | 値あり |
| healthcheck の `${...}` | ホスト環境変数 | ファイル読込時（ホスト） | 空 |

## $$ で展開する場所を移す

やりたいのは「Compose には手を出させず、文字列 `$POSTGRES_USER` をそのままコンテナまで運び、コンテナ内シェルに展開させる」こと。

そのために、Compose へ「この `$` は自分用ではない」と伝える。その合図が **`$` を2つ重ねる（`$$`）** 記法だ。

```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
```

展開の流れはこうなる。

```
記述:               $${POSTGRES_USER}
   ↓ Compose は "$$" を "$" に戻すだけで、中身は展開しない
コンテナに届く文字列: ${POSTGRES_USER}
   ↓ コンテナ内シェルが env_file 由来の値で展開
実行コマンド:        pg_isready -U postgres ...
```

## 直ったことの確認

修正後に `docker compose config` をもう一度実行すると、今度は警告が消え、出力はこうなった。

```
  db:
    healthcheck:
      test:
        - CMD-SHELL
        - pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}
```

`config` の出力に `$$` のまま残っているのは、「コンテナに届く時点では `$`（1つ）になる」ことを表す。空だった前回と違い、変数名が消えずに運ばれているのが確認できる。

静的チェックだけでなく、実挙動も見ておくと確実。

```bash
docker compose up -d
docker compose ps          # db が (healthy) になるか
```

`db` が `(healthy)` になり、`condition: service_healthy` で待たせている `api` がその後に起動すれば成功。

## まとめ

- docker compose の変数展開は **2 段階**ある。`${VAR}` は「ファイル読込時にホスト側」で、`$$VAR`（コンテナ内で `$VAR` になる）は「実行時にコンテナ内シェル」で展開される
- `env_file:` の変数は **コンテナの環境変数**。compose ファイル内の `${...}` 展開（ホスト側）からは見えない
- `environment` には値が出るのに healthcheck だけ空になるのは、両者の参照先が違うため。矛盾ではなく原因の証拠
- コンテナ内で実行されるコマンド（healthcheck の `CMD-SHELL` など）で env_file の変数を使うなら、`$$` でエスケープしてコンテナ内シェルに展開させる
- `docker compose config` で展開後の姿を確認するのが、この種のハマりの一番の早道

## 参考

- [postgres - Docker Hub](https://hub.docker.com/_/postgres) — healthcheck や環境変数の公式例
- `docker compose config` — 変数展開後の最終的な設定を確認するコマンド
- 補足: `pg_isready` は `-U` を省略すると OS のログインユーザー名を使うため、postgres イメージ上では `-U` 自体を省く書き方の例も多い
