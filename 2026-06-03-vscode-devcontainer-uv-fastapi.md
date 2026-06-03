---
title: "VSCode Dev Container で uv×FastAPI の import 警告を消す"
date: 2026-06-03
tags: [VSCode, DevContainer, Docker, FastAPI]
format: tutorial
---

## 概要

ホスト側に `.venv` を作らない構成（依存をイメージ内のシステム Python `/usr/local` に入れる）で FastAPI 開発環境を組むと、VSCode 上で `Import "fastapi" could not be resolved` という警告が残る。アプリはコンテナ内で正しく動いているのに、エディタだけが import を解決できない状態。

このチュートリアルでは、VSCode の Dev Container を使ってこの警告を消す。ホスト側の構成（`.venv` を作らない方針）は一切変えず、エディタの言語サーバーだけをコンテナ内に移すのがポイント。

完成後の状態:

- `Import "fastapi" could not be resolved` の警告が消える
- ホスト側に `.venv` は作らないまま
- ソースコードはホスト側のファイルを編集し続ける（保存先・git 管理はホストのまま）

## なぜ警告が出るのか

この警告はエディタ（Pylance）が出している。Pylance はホスト側の Python 環境を見て import を解決するが、今回の構成ではホストに `.venv` を作らず、`fastapi` はイメージ内（`/usr/local`）にしか無い。そのため Pylance が `fastapi` を見つけられず警告する。アプリの実行はコンテナ内で完結しているので、動作には影響しない。

## Dev Container の考え方

登場するものは 2 種類あり、置き場所と役割が違う。

| | 正体 | 置き場所 | 共有方針 |
|---|---|---|---|
| ソースコード（`main.py` 等） | テキスト（成果物） | ホスト | マウントで共有する |
| 環境（`.venv` / インストール済みパッケージ） | 環境ごとに組み立てる成果物 | コンテナ内（`/usr/local`） | 共有しない |

Dev Container の「Reopen in Container」は、**VSCode の裏側（言語サーバー＝ import 解決やデバッガを動かす部分）をコンテナ内で動かす**機能。これにより `/usr/local` の `fastapi` が見えて警告が消える。

重要なのは、コンテナに入っても**編集する対象はバインドマウントされたホスト側のソース**だということ。「コードはホスト、依存を見る目だけコンテナに移す」というイメージで、マウントを外す必要はない（外すとリビルドで編集が消える・git がソースを追えない・ホットリロードが効かなくなる）。

## 前提

- ホストに `.venv` を作らない構成で、`docker compose up` により FastAPI が起動する状態
- `docker-compose.yaml` がプロジェクトルートにあり、`backend` サービスが `./backend:/app` をマウントしている
- VSCode に拡張機能「Dev Containers」がインストール済み（インストールしただけでは何も起きない。設定ファイルと「Reopen in Container」操作が必要）

## ステップ 1: devcontainer.json を置く場所を決める

`devcontainer.json` は **VSCode で開くフォルダ（ワークスペースのルート）直下の `.devcontainer/`** に置く。`devcontainer.json` 内のパスもこの位置からの相対になる。

| 置き場所 | 開くフォルダ | 向いているケース |
|---|---|---|
| `backend/.devcontainer/` | `backend/` 単独 | バックエンドだけに集中するとき |
| ルートの `.devcontainer/` | プロジェクトルート | フロントもまとめてモノレポにしていくとき |

フロントエンドを後から足す前提なら、ルート（`docker-compose.yaml` と同じ階層）に置くのがよい。

## ステップ 2: devcontainer.json を書く

既存の `docker-compose.yaml` を再利用する形で書く。

```json
// .devcontainer/devcontainer.json
{
    "name": "backend",
    "dockerComposeFile": "../docker-compose.yaml",
    "service": "backend",
    "workspaceFolder": "/app",
    "customizations": {
        "vscode": {
            "extensions": [
                "ms-python.python",
                "ms-python.vscode-pylance"
            ],
            "settings": {
                "python.defaultInterpreterPath": "/usr/local/bin/python"
            }
        }
    }
}
```

各プロパティの意味:

| プロパティ | 意味 |
|---|---|
| `dockerComposeFile` | 使う compose ファイル。**`devcontainer.json` から見た相対パス**。`.devcontainer/` の中にあるので一つ上の `../docker-compose.yaml` |
| `service` | compose の中で、エディタが入るサービス名 |
| `workspaceFolder` | コンテナ内で VSCode が開くディレクトリ。compose のマウント先 `/app` と一致させる |
| `customizations.vscode.extensions` | コンテナ側に入れる拡張機能の **ID**（`publisher.name` 形式）。表示名「Python」ではなく `ms-python.python` |
| `customizations.vscode.settings` | コンテナ内 VSCode の設定。`python.defaultInterpreterPath` に **python 実行ファイルのパス**（ディレクトリ `/usr/local` ではなく `/usr/local/bin/python`）を指定する |

書くときに間違えやすい点:

- `dockerComposeFile` のパスは `.devcontainer/` 基準。ルートの compose なら `../` が要る
- `extensions` は表示名ではなく拡張機能 ID。ID は拡張機能ページの「Copy Extension ID」や Marketplace URL の `itemName=` で確認できる
- `settings` は文字列ではなく**オブジェクト**（`"設定キー": 値` のペアを `{ }` で囲む）。インタープリタの値はディレクトリではなく実行ファイルのパス

## ステップ 3: コンテナで開き直す

VSCode のコマンドパレット（`F1` または `Ctrl + Shift + P`）を開き、`Reopen in Container` と打って **「Dev Containers: Reopen in Container」** を実行する。

実行すると順に起きること:

1. compose のコンテナが起動する
2. VSCode がそのコンテナに接続し直す（ウィンドウが再読み込みされる）
3. 拡張機能（Python / Pylance）がコンテナ側にインストールされる（初回は時間がかかる）
4. コンテナ内の `/usr/local` の `fastapi` が見えるようになる

## ステップ 4: 動作確認

確認ポイント:

1. VSCode の左下隅に緑色で **「Dev Container: backend」** のような表示が出ている（コンテナ内で開いている印）
2. `main.py` の `from fastapi import FastAPI` の警告が消えている

`workspaceFolder` が `/app`（= `./backend` のマウント先）なので、コンテナ内には backend のソースだけが見える。これは想定どおりで、backend コンテナには backend のコードしか存在しないため。

## ホスト側に戻すには

コンテナから抜けてホスト側で開き直すには、コマンドパレットから **「Dev Containers: Reopen Folder Locally」** を実行する。左下の緑の表示が消えればホスト側に戻っている。再びコンテナで開きたくなったら「Reopen in Container」を選べば、`.devcontainer/` の設定が再利用される。

なお、`docker compose up` で起動したコンテナはそのまま動き続ける。止めるときは `docker compose down` を使う。

## おわりに

「ホストに `.venv` を作らない」構成を崩さずに、エディタの言語サーバーだけをコンテナへ移すことで import 警告を解消できた。コードはホスト、依存を見る役だけコンテナ、という分担が核心になる。

発展課題:

- フロントエンド（React）を足すときの、モノレポでのワークスペース・コンテナの持ち方
- pytest による最小テストと GitHub Actions での CI
