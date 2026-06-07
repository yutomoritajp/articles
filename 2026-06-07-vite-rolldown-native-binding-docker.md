---
title: "Docker × Vite で起きた rolldown ネイティブバインディングエラーと、package-lock.json の正体"
date: 2026-06-07
tags: [docker, npm, vite, troubleshooting]
format: blog-post
---

## はじめに

React + Vite のフロントエンドを Docker で動かしていて、`docker compose up` のたびに次々とエラーに遭遇した。最初は Tailwind が見つからないエラー、直したと思ったら今度は rolldown のネイティブバインディングが見つからないエラー。

追いかけていくと、根っこは2つの問題に行き着いた。

1. **Docker の名前付きボリュームが node_modules を上書きしていて、再ビルドしても更新されない**
2. **別 OS（Mac）で生成した `package-lock.json` を Linux コンテナに持ち込んでいた**

この記事では、その調査の流れと、最終的に分かった `package.json` / `package-lock.json` の役割をまとめる。同じように「Docker で npm 依存が反映されない」「ネイティブバインディングが見つからない」で詰まった人の役に立つはず。

## 環境

- ホスト: Windows + WSL2（Linux x64）。Mac でも開発する想定
- コンテナ: `node:20-slim`
- Vite 8 系（内部バンドラに rolldown を使用）
- node_modules を名前付きボリューム（`<project>_react_node_modules`）で管理
- ソースは `./frontend` をコンテナの `/app` にバインドマウント

## 症状1: `Could not resolve '@tailwindcss/vite'`

最初のエラーはこれ。

```
vite.config.js (3:24) [UNRESOLVED_IMPORT] Could not resolve '@tailwindcss/vite' in vite.config.js
```

`vite.config.js` では Tailwind v4 のプラグインを import している。

```js
import tailwindcss from '@tailwindcss/vite'
```

`package.json` には `@tailwindcss/vite` が記載されているのに、`node_modules` に実体が無い状態だった。Dockerfile はビルド時に `npm install` を実行しているので一見おかしい。

### 原因: 名前付きボリュームが node_modules を隠していた

Dockerfile の流れはこうなっている。

```dockerfile
COPY --chown=node:node package*.json ./
RUN npm install          # ← イメージ内の /app/node_modules にインストール
COPY --chown=node:node . .
CMD ["npm", "run", "dev"]
```

ビルド時の `npm install` は**イメージ内**の `/app/node_modules` に依存を入れる。ところが起動時、compose が名前付きボリュームを `/app/node_modules` に**上書きマウント**する。このボリュームは Tailwind を `package.json` に追記する前に作られた古い内容のままだったため、`@tailwindcss/vite` が存在しなかった。

ここで重要な挙動がある。

> **名前付きボリュームは「空のとき（初回作成時）だけ」イメージの内容がコピーされる。一度データが入ったボリュームには、イメージを再ビルドしても中身がコピーされない。**

つまり、依存を追加してイメージを再ビルドしても、既存のボリュームが古いままなら反映されない。ここが最大のハマりどころだった。

### 対処: ボリュームを消してから再ビルド

```bash
docker compose down -v          # -v でボリュームごと削除
docker compose build --no-cache frontend
docker compose up -d            # 空ボリュームが新規作成され、イメージの node_modules がコピーされる
```

ポイントは `down -v` でボリュームを消すこと、そして `--no-cache` で `RUN npm install` のキャッシュを無効化することの2点。これで Tailwind のエラーは消えた。

## 症状2: `Cannot find module '@rolldown/binding-linux-x64-gnu'`

次に出たのがこれ。

```
Error: Cannot find native binding. npm has a bug related to optional dependencies
(https://github.com/npm/cli/issues/4828).
  [cause]: Error: Cannot find module '@rolldown/binding-linux-x64-gnu'
```

Vite 8 は内部バンドラに **rolldown**（Rust 製）を使っており、プラットフォーム固有のネイティブバイナリ（`@rolldown/binding-linux-x64-gnu` など）が必要になる。それが入っていないため起動に失敗していた。

### 原因: Mac で生成した package-lock.json を Linux に持ち込んでいた

これは npm の有名なバグ（[npm/cli#4828](https://github.com/npm/cli/issues/4828)）に起因する。

`package-lock.json` を Mac 上（コンテナの外）で生成すると、Linux 用の optional dependency がロックに記録されないことがある。その lock を `COPY package*.json ./` でコンテナに持ち込むと、`npm install` は lock に従うため Linux コンテナでも Linux 用バイナリを入れない。エラーメッセージ自身が「`package-lock.json` と `node_modules` を消して `npm i` し直せ」と言っている通りだ。

### 対処: Linux 環境で lock を作り直す

```bash
rm frontend/package-lock.json          # Mac 産の壊れた lock を削除
docker compose down -v
docker compose build --no-cache frontend
docker compose up -d
```

`package-lock.json` を消した状態でビルドすると、`COPY package*.json` では `package.json` だけがコピーされ、`RUN npm install` が Linux 環境で走って Linux 用バイナリを含む新しい lock を生成する。これで起動できるようになった。

## ハマりどころ: 生成された lock がホストに出てこない

起動はできたが、`package-lock.json` がホスト側に無いままだった。これは想定どおりで、ビルド中に生成された lock は**コンテナ（イメージ）の中**にあり、ホストには自動では出てこない。

ホストに取り戻すには、起動中のコンテナの中で `npm install` を実行するのが確実だった。

```bash
docker compose exec frontend npm install
```

この構成は `./frontend` を `/app` にバインドマウントしているため、コンテナ内の `/app` で `npm install` すると、生成される `package-lock.json` がバインド先のホスト `frontend/` に直接書き出される。あとは Git にコミットすればよい。

```bash
ls frontend/package-lock.json
git add frontend/package-lock.json
```

## package.json と package-lock.json の役割

一連のトラブルで「lock が無くても起動できる」ことに気づき、改めて2つのファイルの違いを整理した。

### package.json — 「何が欲しいか」の宣言

- 人間が手で書くファイル
- 依存を**範囲（バージョンの幅）**で書く。例: `"react": "^19.2.6"`（19.x.x の最新ならOK）
- プロジェクトのメタ情報（名前、scripts など）も持つ

### package-lock.json — 「実際に何が入ったか」の記録

- `npm install` が自動生成するファイル（手で書かない）
- 実際にインストールされた**正確なバージョンを1つに固定**する
- 直接依存だけでなく、依存の依存（推移的依存）まで全部記録する
- 取得元 URL やハッシュ（整合性チェック用）も持つ

### なぜ両方必要か

`package.json` だけだと範囲指定（`^19.2.6` など）に幅があるため、**いつ・どこで install するかで入るバージョンが変わりうる**。「自分の環境では動くのに別環境では壊れる」の温床になる。

`package-lock.json` があると、`npm install` は範囲から選び直さず lock の通りに入れるため、**いつでも完全に同じ依存ツリーが再現できる**。これが lock の存在理由。だから Git にコミットして共有する（`.gitignore` には入れない）。

| | package.json | package-lock.json |
|---|---|---|
| 書く人 | 人間 | npm（自動） |
| バージョン | 範囲（`^`, `~`） | 確定（1つ） |
| 範囲 | 直接依存中心 | 全依存ツリー |
| 役割 | 要求の宣言 | 実態の固定 |
| 起動への必須性 | 必要 | 無くても起動はできる |

### lock が作られる/更新されるタイミング

`node_modules` の中身が変わる操作をすると、結果が lock に書き戻される。

| コマンド | 動作 |
|---|---|
| `npm install` | lock が無ければ新規生成、あれば更新 |
| `npm install <pkg>` | パッケージ追加＋lock 更新 |
| `npm uninstall <pkg>` | パッケージ削除＋lock 更新 |
| `npm update` | バージョン上げ＋lock 更新 |

逆に `npm ci` は **既存の lock の通りに入れるだけで lock を変更しない**（lock が無いとエラー）。lock を作りたいときは `npm install`、lock 通りに忠実に入れたい CI・本番では `npm ci`、と使い分ける。

## Mac と Windows の両方で開発する場合の注意

このトラブルの本質は「OS の違い」ではなく「**Docker コンテナの中で動く Linux のアーキテクチャ**」にある。開発も install もコンテナの中で走るので、ホストが Mac でも Windows でも、本来はコンテナのプラットフォームさえ揃えれば lock を1つで共有できる。

ただし CPU アーキテクチャだけは注意が必要。

| ホスト | デフォルトのコンテナ | 必要なバイナリ |
|---|---|---|
| Windows (x64) | linux/amd64 | `linux-x64-gnu` |
| Mac (Apple Silicon) | linux/arm64 | `linux-arm64-gnu` |
| Mac (Intel) | linux/amd64 | `linux-x64-gnu` |

Apple Silicon の Mac だと同じ Docker でもコンテナが arm64 になり、必要なネイティブバイナリが Windows(x64) と変わる。複数 OS で開発するなら、いちばんシンプルなのはコンテナのプラットフォームを固定する方法。

```yaml
# compose.yml の frontend サービスに追記
services:
  frontend:
    platform: linux/amd64
```

これで Mac でも Windows でも同じ x64 コンテナになり、Linux x64 基準の lock を1つコミットしておけば両対応できる（Apple Silicon ではエミュレーションになり多少遅くなるトレードオフはある）。

## まとめ

- Docker の名前付きボリュームは**空のときしかイメージから初期化されない**。依存を変えたら `down -v` でボリュームを消してから再ビルドする
- `package-lock.json` は OS / CPU 固有の情報を含むことがある。**別 OS で作った lock を持ち込むとネイティブバインディングが見つからない**問題が起きる（npm/cli#4828）
- lock は「動かすため」ではなく「**いつでも同じ状態を再現するため**」にある。だから Git にコミットして共有する
- lock は `npm install` で作られる。CI・本番で忠実に再現したいときは `npm ci`
- 複数 OS で開発するなら `platform: linux/amd64` などでコンテナのアーキテクチャを固定すると lock を1つで共有しやすい
