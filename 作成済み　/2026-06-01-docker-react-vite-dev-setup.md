---
title: "ホストにNode.jsを入れずにDockerでReact(Vite)開発環境を作る"
date: 2026-06-01
tags: [docker, react, vite, frontend]
format: tutorial
---

## 概要

ホストマシンに Node.js をインストールせず、Docker だけで React(Vite) の開発環境を立ち上げる。最終的にブラウザで `http://localhost:5173/` を開くと React の初期画面が表示される状態を目指す。

設計方針は「最小構成から始め、不便を感じたら設定を足す」。よくわからないおまじないを減らし、各コマンドの意味を理解しながら進める。将来的に FastAPI と `docker compose` で繋ぐことを見据えて、フロントエンドは `frontend/` というフォルダ名で作る。

所要時間の目安: 15〜20分。

完成後にブラウザで見える状態:

```
  VITE v8.0.16  ready in 154 ms

  ➜  Local:   http://localhost:5173/
  ➜  Network: http://172.x.x.x:5173/
```

## 前提

- OS: Linux / WSL2（macOS でもほぼ同様）
- ツール: Docker がインストール済みで `docker run` が使えること
- ホストに Node.js は **不要**（入れないのが本記事の目的）
- 作業ディレクトリは任意（ここではプロジェクトのルートで作業する想定）

## なぜビルドツール(Vite)を使うのか

React は JSX を含むため、ブラウザがそのまま実行できない（トランスパイルが必要）。これを自前で webpack や esbuild から組むと、かえって設定（おまじない）が増える。

そこでビルドツールに **Vite** を使う。Vite は現在もっとも一般的な選択肢で、設定ファイル `vite.config.js` が数行で読める。「おまじないを減らす」目的でも、結果的にこちらの方が中身を理解しやすい。

## 鶏が先か卵が先か問題

Vite プロジェクトを作るには通常 `npm create vite` を打つが、それにはホストに Node.js が必要になる。Node を入れたくないのに、最初のプロジェクトをどう作るか。

解決策は「Docker を使い捨ての Node 環境として使う」こと。必要なときだけ Node 入りのコンテナを借りて、作業結果（フォルダ）だけをホストに残す。

## ステップ 1: 使い捨てコンテナで Vite プロジェクトを作る

以下の 1 行で、ホストに Node を入れないまま `frontend/` を生成する。

```bash
docker run --rm -it -v $(pwd):/app -w /app node:20 \
  npm create vite@latest frontend -- --template react
```

各オプションの意味:

| 部分 | 意味 |
|---|---|
| `docker run --rm` | コンテナを起動し、終了したら即破棄する（`--rm`） |
| `-it` | 対話的に操作できるようにする |
| `-v $(pwd):/app` | ホストの現在のフォルダを、コンテナ内の `/app` に共有する |
| `-w /app` | コンテナ内の作業ディレクトリを `/app` にする |
| `node:20` | Node.js 入りの公式イメージを借りる |
| `npm create vite@latest frontend -- --template react` | その中で雛形作成コマンドを実行する |

最後のコマンドの読み解き:

- `npm create vite` … `create-vite` という「雛形作成専用ツール」を一時的に取得して実行する（`npm init vite` の別名）
- `@latest` … `create-vite` のバージョン指定。古いキャッシュを使わず最新で実行する
- `frontend` … 作るフォルダ名（プロジェクト名）。将来 `backend` と並べる想定でこの名前にする
- `--` … **「ここから先は npm ではなく `create-vite` 本体に渡す引数」という区切り**。これが無いと後続の `--template` を npm 自身のオプションと誤解する
- `--template react` … React の雛形を使う（TypeScript 版なら `react-ts`）

実行すると、対話で「Install with npm and start now?」と聞かれることがある。ここで `Yes` にすると、雛形作成後に `npm install` と `npm run dev` まで自動で走り、開発サーバーが起動してしまう。学習目的では一度サーバーを止めて（`Ctrl+C`）、起動は次のステップで明示的に行うとよい。

### 確認

```bash
ls frontend
```

以下のように表示されれば成功。

```
README.md  eslint.config.js  index.html  node_modules  package-lock.json  package.json  public  src  vite.config.js
```

- `index.html` … 入口の HTML（Vite の起点）
- `src/` … React のコード本体
- `vite.config.js` … Vite の設定ファイル（数行で読める）
- `package.json` … 依存とコマンド定義
- `node_modules/` … インストールされた依存の実体

> `node_modules/` がホスト側にも出来ているのは、`-v` でフォルダを共有していたから。コンテナ内で `npm install` した結果がホストに書き込まれた。`-v` の共有は双方向に効く。

## ステップ 2: 開発サーバーを起動する

開発サーバーを正しく起動するには、2 つの壁を越える必要がある。

**壁① ポートを外に出す**

コンテナは独立した小さなマシンのようなもの。中で 5173 番ポート（Vite のデフォルト）がリッスンしていても、`-p` で明示的に繋がないとホストから到達できない。

```
-p 5173:5173
   ↑ホスト   ↑コンテナ
```

**壁② Vite を外向きにする**

デフォルトの Vite は `localhost`（127.0.0.1）だけで待ち受ける。しかしコンテナ内の `localhost` はコンテナ自身を指すため、ホストからの接続は届かない。`--host` を付けて `0.0.0.0`（どの窓口からの接続も受ける）にする必要がある。

両方を反映した起動コマンド:

```bash
docker run --rm -it -v $(pwd)/frontend:/app -w /app -p 5173:5173 node:20 \
  npm run dev -- --host
```

ステップ 1 との違い:

- `-v $(pwd)/frontend:/app` … 今度は `frontend` フォルダの中身を `/app` に共有する
- `-p 5173:5173` … 壁①を越える
- `npm run dev -- --host` … 壁②を越える（`--` の後ろが Vite への引数なのは雛形作成時と同じ理屈）

`npm create vite`（作る）と `npm run dev`（起動する）は別のコマンドである点に注意。

## ステップ 3: 動作確認

起動コマンド実行後、ターミナルに次のようなログが出る。

```
> frontend@0.0.0 dev
> vite

  VITE v8.0.16  ready in 154 ms

  ➜  Local:   http://localhost:5173/
  ➜  Network: http://172.x.x.x:5173/
```

ブラウザで `http://localhost:5173/` を開き、React のロゴが表示される画面が出れば成功。

## ステップ 4: 長い docker run を compose に畳む

ここまでで動くようにはなったが、起動のたびに長い `docker run` を手で打つのは不便。これが「不便を感じたら設定を足す」タイミング。毎回打つ起動コマンドを `compose.yaml` に一度だけ書いておき、`docker compose up` の一言で起動できるようにする。

注意点として、compose が畳むのは **毎回打つ「起動」コマンド（ステップ 2）** であって、最初の 1 回だけの「雛形作成」（ステップ 1）ではない。

### docker run の各部分を compose に翻訳する

| docker run の部分 | compose の書き方 | 補足 |
|---|---|---|
| `node:20` | `image: node:20` | 借りるイメージ |
| `-w /app` | `working_dir: /app` | 作業ディレクトリ |
| `-v $(pwd)/frontend:/app` | `volumes: - ./frontend:/app` | `$(pwd)` が `.`（composeファイルの場所基準）になる |
| `-p 5173:5173` | `ports: - "5173:5173"` | ポート公開 |
| `npm run dev -- --host` | `command: npm run dev -- --host` | 実行コマンド |
| `--rm` | （不要） | compose が管理。`docker compose down` で消す |
| `-it` | （不要） | `docker compose up` がログに繋いでくれる |

`--rm` と `-it` が消えるのは、コンテナの生死を compose が管理してくれるため。これが compose を使う旨味の一つ。

### ファイルを置く場所

`compose.yaml` は frontend の外側（プロジェクトルート）に置く。将来 FastAPI を `backend` サービスとして同じファイルに足せるようにするため。

```
sandbox/
├── compose.yaml        ← 新規作成するファイル
└── frontend/           ← 中身ごとそのまま残す（compose が作り直すわけではない）
    ├── package.json
    └── ...
```

> `compose.yaml` は「起動の手順書」でしかなく、動かす実体（`package.json` や `src/`）は `frontend/` の中にある。`volumes: - ./frontend:/app` は **既存の** `frontend/` の中身をコンテナに共有するという意味なので、frontend は中身ごと残す。フォルダだけ空に残してもダメで、中身が無いと ENOENT で起動できない。ファイル名は新しい `compose.yaml` が正式（昔の `docker-compose.yml` でも動く）。

### compose.yaml の中身

```yaml
services:
  frontend:
    image: node:20
    working_dir: /app
    volumes:
      - ./frontend:/app
    ports:
      - "5173:5173"
    command: npm run dev -- --host
```

`services:` の下に `frontend:` というサービスを 1 つ定義しているだけ。中身は対応表の通りで、新しい概念は増えていない。

### 起動

```bash
docker compose up
```

長い `docker run` の代わりにこれだけで、ステップ 3 と同じく `http://localhost:5173/` で React の画面が表示される。停止は `Ctrl+C`、後片付けは `docker compose down`。

> 補足: compose は出来合いのイメージを借りる（`image:`）だけでなく、Dockerfile を焼いて使う（`build:`）こともできる。今は `node:20` をそのまま借り、コードは `-v` で共有しているため Dockerfile は不要。Dockerfile が要るのは、依存やツールをイメージに焼き込みたいとき、本番用にビルドして配信するとき、コードに依存せず自己完結したイメージを配布したいとき。開発の最小構成では無くても動く。

## トラブルシューティング

- **`Could not read package.json ... open '/app/package.json'` (ENOENT)**
  → 共有フォルダの指定が 1 階層ズレている。`-v $(pwd):/app` ではなく `-v $(pwd)/frontend:/app` を指定する。`package.json` は `frontend/` の中にあるため、`/app` 直下には無い。

- **ブラウザで `http://localhost:5173/` に繋がらない**
  → `-p 5173:5173`（壁①）と `--host`（壁②）の両方が付いているか確認する。片方だけでは繋がらない。起動ログに `Network:` の行が出ていれば `--host` は効いている。

- **`Need to install the following packages: create-vite@9.0.7` と毎回聞かれる**
  → `--rm` でコンテナを毎回破棄しているため、`create-vite` がキャッシュに残らず毎回ダウンロード確認が出る。意味が分かっていれば `y` で進めてよい。

## おわりに

これで、ホストに Node.js を入れずに Docker だけで React(Vite) の開発環境が立ち上がり、長い `docker run` を `compose.yaml` に畳んで `docker compose up` で起動できるようになった。

次に感じそうな不便と、それに応じた発展課題は以下のとおり。

- コードを編集しても画面が自動更新されない場合 → ホットリロード（ファイル監視）の設定を足す。Docker × WSL2 では監視イベントが伝わらず polling 設定が必要になることがある
- node_modules をホストと共有している構成が不便になったら → Dockerfile で依存を焼き込む
- 当初の目的どおり FastAPI を `backend` サービスとして `compose.yaml` に追加し、前後を `docker compose` で繋ぐ

一度に完璧を目指さず、動くものを作って確認し、不便を感じたら足す。作り直しながら理解を深めていく。
