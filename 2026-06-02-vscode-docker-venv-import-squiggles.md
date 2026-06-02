---
title: "Docker 開発で VSCode に出る import の黄色い波線を消す（コンテナ内 venv を Pylance に見せる）"
date: 2026-06-02
tags: [vscode, docker, python, pylance, fastapi]
format: technical-memo
---

## 背景

FastAPI を Docker コンテナ上で開発していると、VSCode のエディタ上で `import fastapi` などに黄色い波線（import 解決エラー）が出る。アプリ自体はコンテナ内で正常に動くのに、エディタだけが警告を出す状態。原因と、エディタを汚さない軽量な対処をまとめる。

前提の構成は次の通り。コンテナ内で `uv` を使い、ソースを bind mount している。

```yaml
# docker-compose.yaml（抜粋）
services:
  backend:
    image: ghcr.io/astral-sh/uv:python3.13-bookworm-slim
    working_dir: /app
    volumes:
      - ./backend:/app
    command: uv run fastapi dev main.py --host 0.0.0.0
```

## 原因：ライブラリは見えるがインタプリタが壊れている

波線の原因は「ライブラリが無いから」ではない。調べると次の状態だった。

- `backend/.venv` の中に fastapi / sqlmodel / sqlalchemy / pydantic は**すべて入っている**
  - bind mount (`./backend:/app`) により、コンテナ内で `uv run` が作った `.venv` がそのままホスト側に見えているため
- ところが venv の Python インタプリタがホストでは壊れている

```bash
# venv の python はコンテナ内のパスを指す symlink
$ ls -la backend/.venv/bin/python
backend/.venv/bin/python -> /usr/local/bin/python3.13

# ホストにそのパスは存在しない（このMacのPythonは Homebrew の /opt/homebrew）
$ ls /usr/local/bin/python3.13
ls: /usr/local/bin/python3.13: No such file or directory
```

```ini
# backend/.venv/pyvenv.cfg もコンテナ前提
home = /usr/local/bin
version_info = 3.13.11
```

つまり「**ライブラリ実体はホストから見えるが、インタプリタはホストでは動かない**」。VSCode の Pylance は壊れたインタプリタから site-packages を辿れず、import を解決できずに波線を出す。

## 対処の選択肢

| 方法 | 内容 | 向き |
|---|---|---|
| A. extraPaths 追加（軽量） | bind mount 済みの site-packages を Pylance の検索パスに直接追加。インタプリタが壊れていても import 解決が効く。ホストに何も入れない | 最小対応で波線を消したい |
| B. Dev Containers | VSCode 自体をコンテナにアタッチし、Pylance をコンテナ内で動かす | 根本解決・堅牢。ホストを汚さない |
| C. ホストにも venv を作る | ホストで `uv sync` し、ホスト用の正常な venv を VSCode に指定 | コンテナと二重管理になり非推奨 |

## 対処 A：extraPaths でコンテナ内 venv を見せる（軽量）

`.vscode/settings.json` に、bind mount で見えている site-packages を Pylance の検索パスとして追加する。これだけで波線が消え、ホストには何もインストールしない。

```json
{
  // bind mount (./backend:/app) 経由でホストに見えている、
  // コンテナ内 venv の site-packages を Pylance の検索パスに追加する。
  // インタプリタが壊れていても import 解決が効く。
  "python.analysis.extraPaths": [
    "./backend/.venv/lib/python3.13/site-packages"
  ]
}
```

ポイント:

- `extraPaths` は「インタプリタが正常に選択されていなくても」Pylance に検索場所を直接教えられるので、壊れた symlink の影響を受けない
- パスに Python バージョン（`python3.13`）が含まれるため、バージョンを上げたら更新が必要
- 参照しているのは bind mount 経由のコンテナ内 venv なので、実質「コンテナ側のライブラリ」を見ている

## 対処 B：Dev Containers（根本解決）

VSCode 自体をコンテナの中に入れて作業する仕組み。画面（UI）だけホストに残し、言語サーバー・ターミナル・デバッガはコンテナ内で動かす。Pylance がコンテナ内で動くので、本物のインタプリタとライブラリをそのまま使え、import 解決のズレが原理的に起きない。

手順は「拡張機能 Dev Containers を入れる → `.devcontainer/devcontainer.json` を置く → コマンドパレットで Reopen in Container」。

```json
{
  "name": "backend",
  "dockerComposeFile": "../docker-compose.yaml",
  "service": "backend",
  "workspaceFolder": "/app",
  "customizations": {
    "vscode": {
      "extensions": ["ms-python.python", "ms-python.vscode-pylance"],
      "settings": {
        // コンテナ内では symlink が正しく解決できるので、
        // venv の python をそのままインタプリタに指定できる
        "python.defaultInterpreterPath": "/app/.venv/bin/python"
      }
    }
  }
}
```

トレードオフ:

- メリット: 補完・デバッグ・ライブラリ解決が完全一致、ホストに Python を入れなくてよい、開発環境をリポジトリで再現できる
- デメリット: 拡張機能の導入が必要、Reopen に時間がかかる、操作感が少し変わる

## まとめ

- 黄色い波線の原因は「ライブラリ不足」ではなく「**コンテナ内 venv のインタプリタがホストでは動かない**」こと
- すぐ消したいなら **A（extraPaths）**、根本的に揃えたいなら **B（Dev Containers）**
- ホストにも venv を作る C は二重管理になるので避ける

## 参考

- [Developing inside a Container (VS Code)](https://code.visualstudio.com/docs/devcontainers/containers)
