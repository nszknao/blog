---
title: "（仮）話し相手エージェント"
emoji: "🦙"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["ai", "LLM"]
published: false
---

:::message
本記事は[AI Agent Hackathon with Google Cloud](https://zenn.dev/hackathons/2024-google-cloud-japan-ai-hackathon)に提出したプロジェクトの記事です。
:::

# プロジェクト概要

子供の疑問に答える AI エージェント「（仮）話し相手エージェント」を開発しました。

## 課題と対象ユーザー

私たちのプロジェクトが対象とするユーザーは、好奇心旺盛な子供とその親です。特に、子供が日常生活の中で「これなに？」と感じた疑問を、すぐに解消できる環境を求める家庭に焦点を当てています。

現在、これらの課題が存在すると考えています。

- 子供の疑問に対して、親が時間的に余裕がなく答えられない場合や、知識が足りずに答えられない場合がある
- 興味を持った瞬間を逃すと、子供の関心が他のことに移りやすく、深い学びにつながらない
- 触覚や実体験を通じた学習が必要な場合があり、Web 検索などデジタルツールだけでは不十分なケースがある

# ソリューションと技術

これらの課題に対するソリューションとして、私たちは以下の特徴を持つ AI エージェントを開発しました。

### 子供の疑問に応える直感的なインターフェース

- 子供が「これなに？」と話しかけるだけで動作
- マイク・カメラ・スピーカーを搭載し、音声・映像の両方から情報を取得

### インタラクティブな学習体験

- 疑問に対してリアルタイムに回答を生成し、関連する追加情報も提供
- 子供がさらに深堀りした質問をした際も柔軟に対応

このプロジェクトにより、子供の好奇心を育むと同時に、親の負担を軽減します。

![](/images/zenn-ai-agent-hackathon/fc5f31c2-dec5-46bf-b910-6857bd7b904b.webp)
_利用イメージ_

## 技術スタック

TODO: 技術スタックについて記載

## システムアーキテクチャ

TODO: アーキテクチャ図を添付

- 入力層:
  - マイク: 子供の質問を音声として取得
  - カメラ: 周囲の映像を取得
- 処理層:
  - 音声認識: 質問内容をテキストに変換
  - 映像認識: カメラ映像から Google Cloud や Gemini API を使って対象物を特定
- 出力層:
  - スピーカー: 音声で回答を伝える

# 実装とデザイン

## 入力層

実際に使用したハードウェはこちら。

- Raspberry Pi 3 Model B
- マイク
- カメラ
- スピーカー

## 処理層

### 音声認識

SpeachRecognition をインストール

```bash
uv add SpeechRecognition
```

SpeachRecognition でマイクを使う場合は PyAudio もインストールが必要

```bash
sudo apt-get install portaudio19-dev

uv add PyAudio
```

pyttsx3 をインストール

```bash
sudo apt install espeak-ng libespeak1

uv add pyttsx3
```

オフラインで動くのが良いです

:::details flac インストール

```bash
OSError: FLAC conversion utility not available - consider installing the FLAC command line application by running `apt-get install flac` or your operating system's equivalent
```

```bash
sudo apt install flac
```

:::

### 映像認識

TODO: 映像認識のライブラリやモデルについて記載

## 出力層

TODO: スピーカーから音声を出力する方法について記載

# デモンストレーション

TODO: GitHub リポジトリへのリンクを貼る
TODO: デモ動画を埋め込む

# 課題と展望

必要最低限の機能を実装した今回の MVP をより使いやすく、より多くのユーザーに利用してもらうために、以下のようなユーザー体験の改善点があると考えています。

- AI の回答をもっと直感的でわかりやすくするために、画像や動画を活用して説明できるよう、ディスプレイを搭載する
- 外出時のインターネットに接続できない環境では、ローカル LLM で回答を生成する（マシンスペックの制限有り）

## 想定されるリスク

#### AI エージェントに過度に依存することで、自ら調べたり考える力が低下する可能性がある

#### AI に任せることによる親子のコミュニケーション減少への懸念

# チーム

大学の同期 4 人が集まってできた技術系グループ Active Club（通称アクティ部）です。
https://www.activeclub.jp

# 参考文献・リンク

TODO: 参考にした情報やリンクについて記載
