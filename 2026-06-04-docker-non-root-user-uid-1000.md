---
title: "Docker開発環境で `USER` を非rootにする理由 — `useradd -u 1000` は何のエラーを防ぐのか"
date: 2026-06-04
tags: [docker, wsl, permission, fastapi]
format: blog-post
---

## はじめに

Dockerfile でよく見かける、この2行。

```dockerfile
RUN useradd -m -u 1000 appuser
USER appuser
```

「なんとなくセキュリティに良さそう」で写経しがちだが、開発環境においてこれが**具体的にどんなエラーを防いでいるのか**、そして**どの環境では効果があり、どの環境では不要なのか**を整理する。

結論を先に言うと、これは主に **「コンテナが生成したファイルがホスト側で root 所有になり、`sudo` なしで消せなくなる」問題**を防ぐための設定であり、**WSL や Linux では効果が大きく、macOS ではほぼ不要**という、環境依存の強い対策である。

## 防ごうとしている問題：root所有ファイルが消せなくなる

Docker の公式イメージの多くは、コンテナ内のプロセスを **root（UID 0）**として実行する。開発時はソースコードを bind マウントで共有することが多い。

```yaml
services:
  backend:
    build: ./backend
    volumes:
      - ./backend:/app   # ホストのコードをコンテナに共有
```

この状態でコンテナ内のプロセス（例: `fastapi dev` のホットリロード）が `__pycache__` やログ、生成物などを `/app` 配下に書き出すと、その**ファイルはコンテナの実行ユーザー（root, UID 0）の所有**になる。

bind マウント経由なので、このファイルは**ホスト側にもそのまま現れる**。そしてホスト側でもオーナーは UID 0 = root。結果、ホストの一般ユーザーからは：

```
$ rm -rf backend/__pycache__
rm: cannot remove 'backend/__pycache__/main.cpython-313.pyc': Permission denied

$ sudo rm -rf backend/__pycache__   # sudo を強いられる
```

という、地味だが毎回うっとうしい状況になる。git の管理対象に混ざったり、エディタから削除できなかったりと、開発体験を確実に削る。

## なぜ「環境依存」なのか

ここが最重要ポイント。**この問題が起きるかどうかは OS（より正確には Docker のファイル共有方式）によって変わる**。

| 環境 | root所有問題 | 理由 |
|---|---|---|
| **macOS（Docker Desktop）** | ほぼ起きない | bind マウントのファイルをホストユーザー所有として見せる仕組みがあり、コンテナ内 root が書いてもホスト側では自分の所有に見える |
| **WSL2 / Linux（ネイティブ ext4）** | **起きる** | 真の Linux 権限がそのまま適用される。コンテナ内 root が書けばホストでも root 所有になる |
| WSL2（`/mnt/c` 配下） | 起きにくい | Windows ドライブの drvfs はメタデータの扱いが特殊で、権限問題が表面化しにくい |

つまり、

- **Mac だけで開発している人** → この2行はあってもなくても体感は変わらない（任意）
- **WSL や Linux で開発する人** → この2行の有無で「`sudo` 地獄になるか否か」が決まる（推奨）

特に WSL は、I/O 性能のためにプロジェクトを**ネイティブ領域（`~/` 配下の ext4）に置くのが定石**であり、これはちょうど「問題が起きる側」に該当する。Mac と Windows(WSL) の二刀流なら、WSL 側に合わせて非root化しておくのが無難。

## 仕組み：効いているのは「chown」ではなく「実行時のUID一致」

ここで多くの人が誤解する点がある。「`chown -R appuser /app` でディレクトリの所有者を変えれば解決」と思いがちだが、**bind マウントする開発構成では、Dockerfile 内の `chown` は実行時に効かない**。

理由は、bind マウントは**コンテナ内の `/app` をホスト側のディレクトリで丸ごと上書き（マスク）する**から。ビルド時にイメージ内の `/app` を何 chown しようと、実行時に見えているのはホスト側の `./backend` であり、その所有権が支配する。

では何が効いているのか。答えは、

> **コンテナのプロセスが、ホストのユーザーと同じ UID で動いているか**

である。Linux/WSL のファイル所有権は名前ではなく **UID（数値）**で管理される。ホスト(WSL)の通常ユーザーはたいてい **UID 1000**（最初に作られた一般ユーザーに割り当てられる）。

- コンテナが **UID 1000** で動けば → 生成ファイルは UID 1000 所有 → ホストの自分のユーザー所有になる → **普通に消せる** ✅
- コンテナが **UID 0（root）**で動けば → 生成ファイルは root 所有 → 消せない ❌

`useradd -m -u 1000 appuser` の `-u 1000` が肝なのはこのため。**ホストユーザーと UID を一致させる**ことで、bind マウント越しの読み書きと所有権が噛み合う。

余談だが、`node:20` 公式イメージに最初から入っている `node` ユーザーも UID 1000 で作られている。React の Docker 構成で `USER node` がすんなり機能するのは、この UID 1000 が WSL のデフォルトユーザーと偶然一致するからである。

## 解決策

### 1. Dockerfile で非rootユーザーを作る（推奨）

ベースイメージに都合のいい非rootユーザーが無い場合（`uv` の Python イメージなど）は、自分で UID 1000 のユーザーを作る。

```dockerfile
RUN useradd -m -u 1000 appuser
USER appuser
```

**配置の注意**: 依存インストールなど **root 権限が必要な処理（システム領域への書き込み）は `USER` 切り替えより前**に済ませること。`USER` を早く置きすぎるとインストールが権限エラーで失敗する。

FastAPI + uv の例（`uv sync` はシステム領域 `/usr/local` に書くため root のまま実行し、その後で切り替える）:

```dockerfile
FROM ghcr.io/astral-sh/uv:python3.13-bookworm-slim

ENV UV_PROJECT_ENVIRONMENT="/usr/local/"
ENV UV_SYSTEM_PYTHON=1
ENV UV_LINK_MODE=copy
ENV UV_CACHE_DIR=/tmp/uv-cache          # /root 以下から逃がす（後述）

WORKDIR /app

COPY pyproject.toml uv.lock ./
RUN uv sync --frozen                     # ← root のまま（/usr/local に書くため）

COPY . .

RUN useradd -m -u 1000 appuser
USER appuser                             # ← インストール完了後に切り替え

CMD ["uv", "run", "--no-sync", "fastapi", "dev", "main.py", "--host", "0.0.0.0"]
```

開発用（bind マウントあり）であれば、`/app` はどうせ実行時にホスト側で上書きされるため、`chown -R appuser /app` は**不要**。効いているのは「UID 1000 で動くこと」だけなので、実質この2行で足りる。

### 2. キャッシュ・ホームの書き込み先に注意

非rootユーザーは `/root` 配下に書けない。ツールのキャッシュやホームが `/root` を指していると、実行時に Permission denied になることがある。上記例で `UV_CACHE_DIR` を `/root/.cache/uv` から `/tmp/uv-cache` に変えているのはこのため。

- 症状: 非root化した途端、ツールがキャッシュ書き込みで失敗する
- 対処: キャッシュ/一時ディレクトリの環境変数を `/tmp` など書き込める場所に向ける

### 3. Dockerfileを触らない手軽版：compose の `user:`

ユーザーを作らず、実行 UID だけ compose で指定する方法もある。

```yaml
services:
  backend:
    build: ./backend
    user: "1000:1000"
    volumes:
      - ./backend:/app
```

`useradd` 不要で最も手軽。ただしそのUIDが `/etc/passwd` に存在しないため `HOME` が未設定になり、一部ツールが警告・失敗することがある（キャッシュ先を `/tmp` に固定するなどで回避）。**きっちりやるなら Dockerfile 版、手軽さ優先なら compose 版**。

### 4. 本番イメージの場合

本番は bind マウントせず `COPY . .` でコードを焼き込むため、**`/app` の所有権が実行時にも効く**。この場合は非rootユーザーが書き込めるよう、明示的に所有権を渡す必要がある。

```dockerfile
COPY --chown=appuser:appuser . .
# または
RUN chown -R appuser:appuser /app
USER appuser
```

## まとめ

- `RUN useradd -m -u 1000 appuser` + `USER appuser` は、**コンテナ生成ファイルが root 所有になり消せなくなる問題**を防ぐ設定。
- 効くかどうかは環境依存。**WSL2 / Linux（ネイティブ領域）では効果大、macOS ではほぼ不要**。Mac/Windows 二刀流なら WSL に合わせて入れておくのが安全。
- 本質は「**コンテナの実行 UID をホストユーザー（通常 1000）と一致させる**」こと。bind マウント構成では Dockerfile 内の `chown` は実行時にマスクされるため、効いているのは UID 一致のほう。
- `USER` 切り替えは**root権限が要るインストール処理の後**に置く。キャッシュ/ホームが `/root` を指していないかも確認する。
- 開発用（bind マウント）なら chown 不要で2行だけ、本番用（焼き込み）なら `--chown` も必要、と使い分ける。
