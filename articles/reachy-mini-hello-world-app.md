---
title: "Reachy Mini Wireless で自作 Hello World アプリを作り、実機デプロイから HF 公開までやってみる"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ReachyMini", "Python", "ロボット", "HuggingFace"]
published: true
---

# tl;dr

- オープンソースの卓上ロボット Reachy Mini Wireless で、首を振ってアンテナを揺らすだけの最小 Hello World アプリを自作した
- アプリは `ReachyMiniApp` を継承して `run()` ループを書くだけ。雛形は公式 CLI `reachy-mini-app-assistant` が生成してくれる
- 開発機（Mac）から実機への SSH 鍵デプロイ、デーモン REST API での起動、Hugging Face Spaces への公開までの一連の流れをまとめた
- ハマりどころ（SSH の `known_hosts` / `MaxAuthTries`、初期パスワード、API パスの違い）も記録

:::message
これは「まずは動かす」ことを目的とした入門記事です。
:::

# Reachy Mini とは

[Pollen Robotics](https://www.pollen-robotics.com/)（現在は Hugging Face 傘下）が開発する、オープンソースの卓上ヒューマノイドロボットです。

https://huggingface.co/docs/reachy_mini/index

2 つのバリアントがあります。

- Lite: USB で母艦 PC に接続して動かす
- Wireless: 内部に Raspberry Pi CM4 を搭載し、バッテリー + Wi-Fi で単体動作

今回は Wireless を使います。Wireless だけが IMU（加速度・ジャイロ）を搭載しており、ほかに広角カメラ・マルチマイクアレイも備えるのでセンサー遊びに向いています。

ハードウェア構成はこんな感じです。

- 頭部: 6 自由度（Stewart プラットフォーム）
- 胴体: 垂直回転
- アンテナ: 2 本（物理ボタンとしても使える）

# アプリの仕組み

Reachy Mini のアプリは、ロボット内で常駐する デーモンがライフサイクルを管理します。

1. ダッシュボード or REST API から「アプリ起動」をリクエスト
2. デーモンがアプリを Python サブプロセスとして起動（`python -m your_app.main`）
3. アプリは接続済みの `ReachyMini` インスタンスと `stop_event` を受け取る
4. 停止時はデーモンが `SIGINT` を送り、グレースフルに終了

そしてアプリの実体は、`ReachyMiniApp` を継承して `run()` を実装したクラスです。

# 開発環境の準備

公式 CLI `reachy-mini-app-assistant` を使います。注意点として、`reachy-mini` の依存（`libusb-package` など）は Python 3.13 までの wheel しか無く、私の Mac の Python 3.14 だと解決に失敗しました。`uvx` で Python 3.11 を明示して回避します。

```bash
# 雛形生成も check も publish もこの CLI 経由
uvx --python 3.11 --from reachy-mini reachy-mini-app-assistant --help
```

:::message
ローカルに常駐させたくないので、本記事ではすべて `uvx`（使い捨て環境）で実行しています。`uv pip install reachy-mini` で常設してもOKです。
:::

# 1. アプリの雛形を作る

フォルダは手動で作らず、必ず CLI で生成します（エントリポイントや HF 用メタデータを正しく用意してくれるため）。

```bash
uvx --python 3.11 --from reachy-mini reachy-mini-app-assistant create reachy_mini_lab .
```

生成物はこんな構成です。

```
reachy_mini_lab/
├── index.html / style.css      # HF Space のランディングページ
├── pyproject.toml              # エントリポイント定義
├── README.md                   # reachy_mini_python_app タグ付き
└── reachy_mini_lab/
    ├── __init__.py
    ├── main.py                 # ここにアプリのロジックを書く
    └── static/                 # 任意の Web UI
```

`pyproject.toml` には、デーモンがアプリを発見するためのエントリポイントが入っています。

```toml:pyproject.toml
[project.entry-points."reachy_mini_apps"]
reachy_mini_lab = "reachy_mini_lab.main:ReachyMiniLab"
```

# 2. Hello World を書く

雛形から Web UI などを削ぎ落として、首を左右に振ってアンテナを揺らすだけの最小コードにしました。本質は `run()` の `while` ループだけです。

```python:reachy_mini_lab/main.py
import threading
import time

import numpy as np

from reachy_mini import ReachyMini, ReachyMiniApp
from reachy_mini.utils import create_head_pose


class ReachyMiniLab(ReachyMiniApp):
    # 今回は Web UI なし
    custom_app_url: str | None = None

    def run(self, reachy_mini: ReachyMini, stop_event: threading.Event):
        print("[reachy_mini_lab] hello! moving until stopped.")
        t0 = time.time()

        while not stop_event.is_set():
            t = time.time() - t0

            # 首をゆっくり左右に
            yaw = 30.0 * np.sin(2 * np.pi * 0.2 * t)
            head = create_head_pose(yaw=yaw, degrees=True)

            # アンテナを揺らす
            a = np.deg2rad(25.0 * np.sin(2 * np.pi * 0.5 * t))
            antennas = np.array([a, -a])

            reachy_mini.set_target(head=head, antennas=antennas)
            time.sleep(0.02)


if __name__ == "__main__":
    app = ReachyMiniLab()
    try:
        app.wrapped_run()
    except KeyboardInterrupt:
        app.stop()
```

ポイント。

- `run(reachy_mini, stop_event)` を実装するだけ。`reachy_mini` は接続済みで渡ってくる
- ループ内で `stop_event.is_set()` を見てグレースフルに抜ける
- 動きの基本は `set_target(head=..., antennas=...)`。なめらかな補間が欲しいときは `goto_target()`
- 頭の姿勢は `create_head_pose(yaw=, pitch=, roll=, degrees=True)` で作る
- 安全リミット（pitch/roll ±40°, yaw ±180° など）は SDK が自動クランプしてくれる

構造の検証は `check` でできます（一時 venv にインストールして検証までやってくれる）。

```bash
uvx --python 3.11 --from reachy-mini reachy-mini-app-assistant check reachy_mini_lab
# => [OK] App 'reachy_mini_lab' passed all checks!
```

# 3. 実機にデプロイする

Wireless の場合、アプリのパッケージを ロボット内の共有 venv（`/venvs/apps_venv/`）に pip install し、デーモンから起動します。

## SSH 鍵の準備（1Password で管理）

毎回パスワードを打つのは面倒なので SSH 鍵を使います。私は鍵を 1Password の SSH エージェントで管理しているので、鍵の実体はディスクに置かず、`~/.ssh/config` でこのロボット専用に「1 個だけ使う」設定にしました。

```ssh-config:~/.ssh/config
Host reachy-mini reachy-mini.local
    HostName reachy-mini.local
    User pollen
    IdentityAgent "~/Library/Group Containers/2BUA8C4S2C.com.1password/t/agent.sock"
    IdentitiesOnly yes
    IdentityFile ~/.ssh/reachy-mini.pub
```

:::details ハマりどころ: SSH 周り
- Host key verification failed: 初回は `known_hosts` に無いので弾かれる。`ssh-keyscan -t ed25519,rsa,ecdsa reachy-mini.local >> ~/.ssh/known_hosts` で登録
- Too many authentication failures: エージェントが多数の鍵を提示すると、ロボットの `MaxAuthTries` を超えて失敗する。`IdentitiesOnly yes` + 対象鍵の `IdentityFile` 指定で「1 個だけ」に絞る
- 初期 SSH 認証情報: 公式ドキュメント通り `username: pollen` / `password: root`（ユーザー名と紛らわしい）
- 1Password 管理の鍵を登録: 秘密鍵がディスクに無いので `ssh-copy-id` に `-f`（フィルタ用ログインをスキップ）を付ける
:::

```bash
# 公開鍵をロボットの authorized_keys に登録（初回のみパスワード入力）
ssh-copy-id -f -i ~/.ssh/reachy-mini.pub pollen@reachy-mini.local
```

## デプロイスクリプト

転送 → インストール → 起動をまとめた `deploy.sh` を用意しました。要点は以下です。

```bash:deploy.sh（抜粋）
# .venv を絶対に送らない（Mac 用バイナリで数百 MB / 無駄）
rsync -az --delete \
  --exclude='.venv' --exclude='__pycache__' --exclude='build' \
  --exclude='*.egg-info' --exclude='uv.lock' --exclude='deploy.sh' \
  ./ pollen@reachy-mini.local:/tmp/reachy_mini_lab/

# ロボットの共有 venv にインストール
ssh pollen@reachy-mini.local "/venvs/apps_venv/bin/pip install --quiet /tmp/reachy_mini_lab"

# デーモン REST API で起動
curl -fsS -X POST http://reachy-mini.local:8000/api/apps/start-app/reachy_mini_lab
```

実行すると、デーモンのログに起動が出ます。

```
reachy_mini.apps.manager.runner - INFO - App reachy_mini_lab is running
reachy_mini.apps.manager.runner - INFO - [reachy_mini_lab] hello! moving until stopped.
```

`print` した文字列がそのままロボット側のログに出ていて、実機で動いていることが確認できます 🎉

![デモ](/images/reachy-mini-hello-world-app/Untitled.gif)

止めるときは:

```bash
curl -X POST http://reachy-mini.local:8000/api/apps/stop-current-app
```

# 4. Hugging Face Spaces に公開する

Reachy Mini のアプリストアは Hugging Face Spaces で動いています。公開すると、Reachy Mini Control（公式デスクトップアプリ）の「Discover apps」からワンクリックでインストールできるようになります。

まず HF にログイン（Write 権限のトークンが必要）。

```bash
uvx --python 3.11 --from huggingface_hub hf auth login
```

公開はこれだけ。`check` も自動で走ります。

```bash
# まずは private で様子見
uvx --python 3.11 --from reachy-mini reachy-mini-app-assistant publish reachy_mini_lab "initial publish" --private
```

問題なければ public 化（App Store に載せるには public が必要）。

:::message alert
public = コードが全世界に見え、誰でもインストールできる状態になります。Reachy Mini のアプリは「各自のロボットがソースを pull して実行する」配布モデルなので、コードを隠したまま誰でも使わせる、は基本できません。隠したいロジックがあるなら、薄いクライアントだけ公開して本体は自前サーバーに置く構成に。
:::

## ハマりどころ

:::details monorepo に入れていると nested .git ができる
`publish` はアプリディレクトリ内に HF Space 用の `.git` を作ります。私はアプリを別の monorepo 配下に置いていたので「入れ子リポジトリ」になってしまいました。monorepo を正本にするなら、この `.git` は削除し、更新は `publish` を再実行する運用が楽です。
:::

:::details Control の Discover にすぐ出ない
公開直後はストアのインデックス同期にラグがあります。デーモンの `GET /api/apps/list-available` を叩くとカタログ（HF 上の全公開アプリ）が返るので、ここに自分のアプリが載っていれば反映済み。出ない場合は Control を再読み込み or 名前で検索。なお `/api/apps/list` ではなく `list-available` が正しいパスでした。
:::

URL 指定で直接インストールすることもできます。

```bash
curl -X POST http://reachy-mini.local:8000/api/apps/install \
  -H "Content-Type: application/json" \
  -d '{"name":"<user>/reachy_mini_lab","source_kind":"hf_space","url":"https://huggingface.co/spaces/<user>/reachy_mini_lab"}'
```

# まとめ

- `ReachyMiniApp` を継承して `run()` を書くだけで、Reachy Mini のアプリは作れる
- 雛形・検証・公開はすべて `reachy-mini-app-assistant` に任せられる
- 実機デプロイは「共有 venv に pip install → デーモン REST で起動」。SSH 鍵だけ整えれば `deploy.sh` 一発
- HF Spaces に公開すれば、Control のストアから誰でもインストール可能に

# 参考リンク

- Reachy Mini ドキュメント: https://huggingface.co/docs/reachy_mini/index
- アプリの作り方（Building & Publishing Apps）: https://huggingface.co/docs/reachy_mini/SDK/apps
- SDK リポジトリ: https://github.com/pollen-robotics/reachy_mini
