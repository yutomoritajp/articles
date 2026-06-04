---
title: "DockerだけでReact(Vite)開発環境を作る — ホストにNode不要のハンズオン"
date: 2026-06-04
tags: [docker, react, vite, frontend]
format: tutorial
---

## 概要

ホストマシンに Node.js をインストールせず、**Docker だけで React(Vite) の開発環境**を立ち上げるチュートリアル。最終的にブラウザで `http://localhost:5173` を開くと React の初期画面が表示され、コードを編集すると自動でリロード（ホットリロード）される状態を目指す。

将来 FastAPI や PostgreSQL と `docker compose` でつなぐことも見据え、`frontend/` というサブフォルダ構成で作る。

所要時間の目安: 20〜30 分（初回ビルドのダウンロード時間を含む）。

完成後のフォルダ構成:

```
.
├── docker-compose.yaml
└── frontend/
    ├── Dockerfile
    ├── index.html
    ├── package.json
    ├── src/
    ├── vite.config.js
    └── ...
```

## 前提

- OS: macOS / Linux / WSL2
- ツール: Docker がインストール済みで `docker run` / `docker compose` が使えること
- ホストに Node.js は **不要**（これがこの構成の主目的）
- パッケージマネージャは **npm** を使用（Node 公式イメージに最初から入っているため。yarn でも可だが lock ファイルとコマンドを片方に統一すること）

## ステップ 1: Vite の雛形を作る

ホストに Node を入れず、使い捨てコンテナの中で `create-vite` を実行して `frontend/` を作る。

```bash
docker run --rm -it -v $(pwd):/app -w /app node:20 \
  npm create vite@latest frontend -- --template react
```

各オプションの意味:

| 部分 | 意味 |
|---|---|
| `docker run --rm` | コンテナを起動し、終了したら即破棄する |
| `-it` | 対話的に操作できるようにする |
| `-v $(pwd):/app` | ホストの現在フォルダをコンテナ内の `/app` に共有する |
| `-w /app` | コンテナ内の作業ディレクトリを `/app` にする |
| `node:20` | Node.js 入りの公式イメージを借りる |
| `npm create vite@latest frontend -- --template react` | React の雛形を作る。`--` 以降は npm ではなく create-vite に渡す引数 |

**確認**: `frontend/` が生成され、中に `package.json` / `src/` / `index.html` / `vite.config.js` などがあれば成功。

> 注意: `npm create vite` は**雛形ファイルを作るだけで、依存はインストールしない**。この時点では `node_modules` は存在しない。インストールは後述の Dockerfile 内で行う。これを知らずに次の手順で `npm run dev` すると `vite: not found` で落ちるので注意。

## ステップ 2: vite.config.js にホスト設定を加える

生成された `frontend/vite.config.js` に `server.host` を追加する。

```js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

// https://vite.dev/config/
export default defineConfig({
  plugins: [react()],
  server: {
    host: true,
  },
})
```

**なぜ必要か**: Vite の開発サーバーはデフォルトで `localhost`（= `127.0.0.1`）だけを listen する。これは「自分自身からの接続しか受け付けない」という意味で、**コンテナ内の `localhost` とホスト(Mac)の `localhost` は別物**である。

そのため、ポートを公開（`-p 5173:5173`）して接続自体はコンテナまで届いても、Vite 側が「外部からの接続」とみなして弾いてしまい、ブラウザが真っ白になる。

`host: true` は Vite に「`localhost` だけでなく全ネットワーク（`0.0.0.0`）から接続を受け付けろ」と指示する設定。これで外（ホストのブラウザ）からの接続が通るようになる。

これは FastAPI（uvicorn）で `--host 0.0.0.0` を指定するのと**まったく同じ理屈**。コンテナ内のサーバーを外から繋ぐには「`0.0.0.0` で待ち受けろ」が必要になる。CLI で `npm run dev -- --host` と書くのと等価で、config に書くか起動コマンドに付けるかのどちらか一方でよい。

## ステップ 3: frontend/Dockerfile を作る

`frontend/Dockerfile` を新規作成する。

```dockerfile
FROM node:20

RUN mkdir /app && chown node:node /app
WORKDIR /app

USER node

COPY --chown=node:node package*.json ./
RUN npm install

EXPOSE 5173

COPY --chown=node:node . .

CMD ["npm", "run", "dev"]
```

各行のポイント:

- **`USER node`（非 root 実行）**: `node:20` イメージはデフォルトで root として動くため、コンテナが生成したファイルがホスト側で root 所有になり、後で編集・削除できなくなる事故が起きる（特に Linux）。標準で用意されている非 root ユーザー `node` に切り替えて回避する。
- **`COPY --chown=node:node`**: `COPY` は `USER` 指定に関わらずデフォルトで root 所有でコピーする仕様。`--chown` を付けて所有者を `node` に揃える。
- **`RUN npm install`**: ステップ 1 の雛形作成では依存が入っていないので、ここでインストールする。**これが抜けていると `vite: not found` で起動に失敗する**ので必須。
- **`COPY package*.json ./` → `RUN npm install` → `COPY . .` の順番**: Docker は各行の結果をキャッシュする。先に `package.json` だけコピーして install し、その後にソース全体をコピーすると、`src/` のコードを直しただけのときは `package.json` が変わっていないので **`npm install` をキャッシュで飛ばせて高速**になる。逆に `COPY . .` を先にやると、コードを 1 行直すたびに install が毎回走って遅い。

## ステップ 4: docker-compose.yaml を書く

プロジェクトルートに `docker-compose.yaml` を作成する。

```yaml
services:
  frontend:
    build: ./frontend
    ports:
      - 5173:5173
    volumes:
      - ./frontend:/app:cached
      - react_node_modules:/app/node_modules

volumes:
  react_node_modules:
```

各設定のポイント:

- **`build: ./frontend`**: `frontend/Dockerfile` を使ってイメージをビルドする。
- **`ports: - 5173:5173`**: 左がホスト側、右がコンテナ側のポート。`localhost:5173` でアクセスできる。
  - もしホスト側を別ポート（例 `3000:5173`）にしたい場合は、HMR の WebSocket がポート違いで失敗する。その場合は vite.config に `server.hmr.clientPort: 3000`（ホスト側ポート）と `server.port: 5173`（コンテナ側ポート）を明示する必要がある。ポートを揃えておけば不要。
- **`./frontend:/app:cached`（bind マウント）**: ホストのソースをコンテナに常時共有する。これにより**ホストでコードを編集すると即コンテナに反映**され、ホットリロードが効く。`:cached` は macOS 等での bind マウント I/O を軽減するオプション。
  - コンテナ側のパスは**絶対パス（`/app`）**で書くこと。`app` のようにスラッシュを抜くとエラーになる。
- **`react_node_modules:/app/node_modules`（名前付きボリューム）**: `node_modules` だけは bind マウントから除外し、専用のボリュームに逃がす。これには2つの理由がある。
  1. **上書き防止**: bind マウントだけだと、ホスト側の（空の or 存在しない）`node_modules` が、コンテナ内の install 済み `node_modules` を上書きしてしまい、また `vite: not found` になる。
  2. **OS 依存バイナリの衝突回避**: Vite/esbuild は OS・CPU 別のバイナリを持つ。ホスト(mac/arm)で作った `node_modules` をコンテナ(linux)で使うと動かない。ボリュームで分離すればこの衝突を防げる。
  - **`:` を忘れない**こと。`react_node_modules/app/node_modules` のようにコロンを抜くと、名前付きボリュームではなく単なる相対パスとして解釈されてしまう。
- **トップレベルの `volumes:` 宣言**: 名前付きボリューム `react_node_modules` は、サービス内で参照するだけでなくトップレベルの `volumes:` でも宣言する必要がある。忘れがちなので注意。

## ステップ 5: 起動して動作確認する

ビルドして起動する。

```bash
docker compose up --build
```

**`--build` について**: `docker compose up` はイメージが無ければ自動でビルドするが、一度ビルドした後に Dockerfile や package.json を変更しても、ただの `up` では古いイメージのまま使い回す。確実に反映させるため、初回や Dockerfile/package.json 変更時は `--build` を付けるのが安全。

| コマンド | 挙動 |
|---|---|
| `docker compose up` | イメージが無ければビルド、有れば既存を使い回す |
| `docker compose up --build` | 毎回必ずビルドし直す |
| `docker compose build` | ビルドだけする（起動はしない） |

**確認**: 起動ログに以下の **2 行（特に `Network:`）** が出れば、`host: true` が効いている証拠。

```
➜  Local:   http://localhost:5173/
➜  Network: http://172.x.x.x:5173/
```

ブラウザで `http://localhost:5173` を開き、React の初期画面が表示されれば成功。

**ホットリロードの確認**: `frontend/src/App.jsx` の文言を少し書き換えて保存し、ブラウザが自動で更新されれば HMR も効いている。

## トラブルシューティング

- **ブラウザが真っ白 / 繋がらない** → vite.config の `server.host: true` が入っているか確認。これが無いとコンテナ外から接続できない。
- **`vite: not found` で落ちる** → Dockerfile に `RUN npm install` があるか、`node_modules` 用の名前付きボリュームが正しくマウントされているか（コロン抜けに注意）確認する。
- **保存しても自動リロードされない（HMR が効かない）** → macOS / WSL2 では bind マウント経由のファイル変更検知（inotify）が効かないことがある。vite.config の `server.watch.usePolling: true` を追加するとポーリングで検知するようになる。CPU を少し食うので、効いているなら付けなくてよい。
- **ホスト側ポートを 5173 以外にしたら HMR が失敗する** → vite.config に `server.hmr.clientPort`（ホスト側の公開ポート）を明示する。
- **生成ファイルが root 所有になって消せない（Linux）** → Dockerfile の `USER node` と `COPY --chown=node:node` が入っているか確認する。

## おわりに

これでホストに Node を入れず、Docker だけで React の開発環境を立ち上げ、`localhost:5173` に初期画面を表示できるようになった。最初の素朴な構成にありがちな次の問題も、すべて解消されている。

- `npm install` を Dockerfile 内で実行 → 起動が安定
- `node_modules` を名前付きボリュームで分離 → 上書き事故・OS 依存バイナリ衝突を回避
- `USER node`（非 root）→ 権限事故を回避
- `server.host: true` → コンテナ外のブラウザから接続可能
- bind マウント → コードの編集が即反映（ホットリロード）

発展課題:

- **backend / db との接続**: 同じ compose ファイルに FastAPI や PostgreSQL のサービスを追加して連携させる。
- **Dev Containers**: VS Code ごとコンテナの中に入って開発する構成。`.devcontainer/devcontainer.json` を用意すると、ターミナルが最初からコンテナ内になり、ポート転送も VS Code が自動でやってくれる（この場合 `host: true` 無しでも繋がる）。
- **本番用ビルド**: `vite build` で静的ファイルを生成し、Nginx などで配信する本番用の構成を別途用意する。
