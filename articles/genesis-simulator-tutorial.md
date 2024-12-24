---
title: "生成AIをネイティブでサポートする（予定の）物理シミュレータ: Genesis"
emoji: "⚛️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Python", "Genesis"]
published: true
---

# はじめに

先日発表された物理 AI エンジン「Genesis」が話題になっていたので、その導入方法や基本的な使い方を解説します。

https://x.com/zhou_xian_/status/1869511650782658846

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
  - uv の Python 経由だと OpenGL のエラーでビューワが起動しない不具合がありました [Issue#11](https://github.com/Genesis-Embodied-AI/Genesis/issues/11)

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

# その他の機能

## パラレルシミュレーション

GPU を使ってシミュレーションを高速化する最大の利点は、シーンレベルの並列処理によって何千もの環境で同時にロボットを訓練できるようになることです。

![](/images/genesis-simulator-tutorial/e09f28ae-b040-48f9-a2e5-3711a3e292a3.webp)

シーンを構築するときに、`n_envs` というパラメータを渡すだけで、シミュレータに必要な環境の数を設定できます。

```python
import torch

B = 20
scene.build(n_envs=B, env_spacing=(1.0, 1.0))

# コントローラを通して入力する際にもバッチ数を指定する
franka.control_dofs_position(torch.zeros(B, 9, device=gs.device))
```

コントローラーの入力にもバッチ数を指定する必要があることに注意してください。

ですが指定を忘れても genesis 側で自動的に次元を追加してくれて、警告とともに実行自体はできるのはポイント高いです。

```
[Genesis] [17:51:31] [WARNING] Input tensor is converted to torch.Size([20, 9]) for an additional batch dimension
```

## 逆運動学とモーションプランニング

逆運動学ではロボットの先端（エンドエフェクタ）を目標位置に移動させるための関節の角度や動きを計算し、モーションプランニングで障害物や環境を考慮しながら、ロボットが目標地点に到達するための経路を計画します。

![](/images/genesis-simulator-tutorial/3fd56437-dcdc-4579-a5f7-7a03f252fd24.webp)

モーションプランニングを実行するには、事前に OMPL（Open Motion Planning Library）モジュールを[インストール](https://genesis-world.readthedocs.io/en/latest/user_guide/overview/installation.html#optional-motion-planning)する必要があるので注意してください。

```python
# エンドエフェクタのリンクを取得
end_effector = franka.get_link('hand')

# 把持する手前の位置と姿勢を逆運動学で計算
qpos = franka.inverse_kinematics(
    link = end_effector,
    pos  = np.array([0.65, 0.0, 0.25]),
    quat = np.array([0, 1, 0, 0]),
)
# グリッパーを開く位置
qpos[-2:] = 0.04
path = franka.plan_path(
    qpos_goal     = qpos,
    num_waypoints = 200, # 2s duration
)
# 計画された経路を実行
for waypoint in path:
    franka.control_dofs_position(waypoint)
    scene.step()

# コントローラによる制御では目標位置と現在位置の間にずれが生じるため、
# 最終的な位置に到達するための微調整
for i in range(100):
    scene.step()
```

## 剛体以外のシミュレーション

genesis ではこれまでに行った剛体ミュレーションの他に、流体力学などの物理ソルバーもサポートしています。

![](/images/genesis-simulator-tutorial/1177fab1-8796-4e46-8a44-918d2c171c32.webp)
*Image: Genesis HP*

# 今後の開発ロードマップ

公式ドキュメントには[ロードマップ](https://genesis-world.readthedocs.io/en/latest/roadmap/index.html)の記載もあります。

中でも「包括的な生成フレームワーク」は、ユーザーが自然言語で記述したプロンプトを、モーションなどさまざまなモダリティのデータに変換する機能とのことで非常に楽しみです。

### 進行中で近日中にリリース予定の機能

- 微分可能で物理ベースの触覚センサーモジュール
- 微分可能な剛体シミュレーション
- タイルレンダリング
- 高速な JIT カーネルコンパイル
- 包括的な生成フレームワーク
  - キャラクターの動作
  - カメラモーション
  - インタラクティブなシーン
  - 顔のアニメーション
  - 移動ポリシー
  - 操作ポリシー
- 大規模な環境に対応する無制限 MPM（Material Point Method）シミュレーション

### 希望されているが現在取り組んでいない機能

- Windows でのビューアおよびヘッドレスレンダリング
- インタラクティブな GUI システム
- さらなる MPM ベースの材料モデルのサポート
- より多くのセンサータイプの対応

# まとめ

生成 AI を活用したソフトウェア開発が盛り上がる中、AI を知能とするロボティクスへの応用が急速に広がっています。

従来の物理シミュレータはデザインやドキュメントの使い勝手に難があり、自分含め新参者には学習コストが高い印象がありました。しかし Genesis は掲げている長期ミッションにもある通り、そのハードルを下げ、非専門家や個人でも物理シミュレーションにアクセスしやすいプラットフォームを提供しています。

自分でも触り続けながらこのプロジェクトの可能性をさらに探求し、応用事例を深掘りしていきたいと考えています。

それではまたお会いしましょう！

# 余談

- 記事内で使用した動画を撮影するために使ったカメラのポジションコントロール難しすぎ？
- アドカレに参加しようと思ったけど、駆け込みでいい感じのカテゴリ枠がなくて野良で投稿
