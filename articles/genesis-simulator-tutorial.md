---
title: "生成AIをネイティブサポートする物理シミュレータ「Genesis」"
emoji: "⚛️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Python", "Genesis"]
published: false
---

# はじめに

先日発表された物理 AI エンジン「Genesis」が話題になっていたので、その導入方法や基本的な使い方を解説します。

https://x.com/_philschmid/status/1869639246434246966

# Genesis とは

NVIDIA やカーネギーメロン大学などの研究者たちによって開発された、汎用ロボット・フィジカル AI アプリケーションのために設計された物理プラットフォームです。
https://genesis-embodied-ai.github.io/

自然言語によるデータ生成といった AI 機能をネイティブでサポートしているなど、他の物理エンジンにはない特徴を持っています。

- 🐍 100% Python
- 👶 簡単インストール＆シンプルな API 設計
- 🚀 並列化シミュレーションによる高速化
- 💥 多様な物理現象を扱う統一フレームワーク
- 📸 フォトリアルなレイトレーシングレンダリング
- 📐 微分可能なシミュレータ
- ☝🏻 物理的に正確で微分可能な触覚センサー
- 🌌 自然言語でさまざまなデータを生成可能

# 環境構築

私が試した環境はこちらです。

- Apple M1 (macOS Sequoia 15.2)
- Python 3.11
- Miniconda がインストール済み
  - uv の Python 経由だと OpenGL のエラーでビューワが起動しない不具合があります [Issue#11](https://github.com/Genesis-Embodied-AI/Genesis/issues/11)

## インストール

:::details uv で試したけど失敗した図

公式ドキュメントの[Getting Started](https://genesis-world.readthedocs.io/en/latest/#getting-started)に沿って進めます。

```bash
uv add genesis-world
```

PyTorch が無い場合はインストール。

```bash
uv add torch
```

動作確認のため、[👋🏻 Hello, Genesis](https://genesis-world.readthedocs.io/en/latest/user_guide/getting_started/hello_genesis.html)のサンプルコードを実行してみます。

```python:hello.py
import genesis as gs
gs.init(backend=gs.cpu)

scene = gs.Scene(show_viewer=True)
plane = scene.add_entity(gs.morphs.Plane())
franka = scene.add_entity(
    gs.morphs.MJCF(file='xml/franka_emika_panda/panda.xml'),
)

scene.build()

for i in range(1000):
    scene.step()
```

```bash
uv run python hello.py
```

実行は完了したのですが、以下の警告とともにビューアが表示されませんでした。

```
[Genesis] [17:00:00] [WARNING] Non-linux system detected. In order to use the interactive viewer, you need to manually run simulation in a separate thread and then start viewer. See `examples/render_on_macos.py`.
```

Linux 以外のマシンでインタラクティブビューアを使用するには、手動で別スレッドでシミュレーションを実行し、ビューアを起動する必要があるようです。

警告にある通り、[examples/render_on_macos.py](https://github.com/Genesis-Embodied-AI/Genesis/blob/main/examples/render_on_macos.py) を代わりに実行してみます。

```bash
uv run python render_on_macos.py --vis
```

[#11](https://github.com/Genesis-Embodied-AI/Genesis/issues/11)のバグを踏んでしまい、解決に時間がかかりそうなので Miniconda で環境を作り直してみます。

:::

公式ドキュメントの[Getting Started](https://genesis-world.readthedocs.io/en/latest/#getting-started)の手順はシンプルなのですが、私の環境ではインストールの順番によって動かないことがあったので注意が必要です。[Issue#61](https://github.com/Genesis-Embodied-AI/Genesis/issues/61)も報告されてました。

PyTorch が無い場合は先にインストール。

```bash
conda install pytorch::pytorch torchvision torchaudio -c pytorch
```

genesis-world をインストール。

```bash
pip install genesis-world
```

動作確認のため、[👋🏻 Hello, Genesis](https://genesis-world.readthedocs.io/en/latest/user_guide/getting_started/hello_genesis.html)のサンプルコードを実行してみます。

```python:hello.py
import genesis as gs
gs.init(backend=gs.cpu)

scene = gs.Scene(show_viewer=True)
plane = scene.add_entity(gs.morphs.Plane())
franka = scene.add_entity(
    gs.morphs.MJCF(file='xml/franka_emika_panda/panda.xml'),
)

scene.build()

for i in range(1000):
    scene.step()
```

```bash
python hello.py
```

実行は完了したのですが、以下の警告とともにビューアが表示されませんでした。

```
[Genesis] [17:00:00] [WARNING] Non-linux system detected. In order to use the interactive viewer, you need to manually run simulation in a separate thread and then start viewer. See `examples/render_on_macos.py`.
```

Linux 以外のマシンでインタラクティブビューアを使用するには、手動で別スレッドでシミュレーションを実行し、ビューアを起動する必要があるようです。

警告にある通り、[examples/render_on_macos.py](https://github.com/Genesis-Embodied-AI/Genesis/blob/main/examples/render_on_macos.py) を代わりに実行してみます。

```bash
python render_on_macos.py --vis
```

正常にビューワが立ち上がり、アームが床に自由落下する様子が確認できました。

![](/images/genesis-simulator-tutorial/ca54f8d2-d086-4fc0-a911-ee4fda74a8fc.webp)

# 基本的な使い方

## 初期化

最初のステップで、genesis をインポートして初期化する必要があります。

```python
import genesis as gs
gs.init(backend=gs.cpu, precision="32")
```

- **バックエンドデバイス**: クロスプラットフォームに対応していて、ここでは`gs.cpu`を指定していますが、`gs.cuda`などの他のバックエンドに切り替えることができます。
- **精度**: デフォルトでは f32 精度ですが、より高い精度が必要な場合は`"64"`を指定することで f64 精度に切り替えることができます。

## シーンの作成

すべてのオブジェクト、ロボット、カメラなどはシーンに配置されます。

```python
scene = gs.Scene(
    sim_options=gs.options.SimOptions(
        dt=0.01,
        gravity=(0, 0, -10.0),
    ),
    show_viewer=True,
    viewer_options=gs.options.ViewerOptions(
        camera_pos=(3.5, 0.0, 2.5),
        camera_lookat=(0.0, 0.0, 0.5),
        camera_fov=40,
    ),
)
```

この例では、シミュレーション dt を各ステップで 0.01s に設定し、重力を設定し、ビューワの初期カメラポーズを設定します。

## オブジェクトの追加

Genesis では、すべてのオブジェクトとロボットは Entity として表されます。 オブジェクト指向で設計されているため、ハンドルや割り当てられたグローバル ID を使用する代わりに、メソッドを通じて直接これらの Entity を操作することができます。

```python
plane = scene.add_entity(gs.morphs.Plane())
franka = scene.add_entity(
    gs.morphs.MJCF(file='xml/franka_emika_panda/panda.xml'),
)
```

`add_entity`の第一引数は Morph 型で、以下のようなプリミティブタイプを持ちます。

- `gs.morphs.Box`: ボックス
- `gs.morphs.Sphere`: 球
- `gs.morphs.Cylinder`: シリンダー
- `gs.morphs.Plane`: 平面

![](/images/genesis-simulator-tutorial/b1b1cebc-bf64-4118-8ebf-d659451eb0b2.webp)

また、他のツールで作成した 3D モデルを読み込むこともできます。現在サポートされているフォーマットはこちらです。

- `gs.morphs.MJCF`: MuJoCo XML ファイル
- `gs.morphs.URDF`: URDF(Unified Robotics Description Format) ファイル
- `gs.morphs.Mesh`: メッシュアセット（\*.obj, \*.ply, \*.stl, \*.glb, \*.gltf）

## シミュレーションの実行

これまでに追加したアセットをビルドし、シミュレーションを実行します。

```python
scene.build()
for i in range(1000):
    scene.step()
```

まず`scene.build()`を呼び出してシーンをビルドする必要があることに注意が必要です。
これは、genesis が実行ごとに GPU カーネルをその場でコンパイルする JIT コンパイラを採用していて、プロセスを開始する明示的なステップが必要なためです。

# 注意点とトラブルシューティング

# まとめ

それではまたお会いしましょう！
