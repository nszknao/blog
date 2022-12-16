---
title: "Blenderで作成した3DモデルをThree.jsで表示する"
emoji: "🔳"
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["react", "typescript", "threejs"]
published: false
---
こんにちは、にしざかです。
普段はWebのインフラからフロントまで薄く広く開発しています。デザインはできないです。

先日、こんな面白いサービスを見つけました。
https://oncyber.io/

自分だけのギャラリー空間を作成して、所有しているNFTを展示できるサービスです。3D空間をうろうろ歩き回りながら展示されているNFTアートを鑑賞でき、購入するためのOpenSeaへの導線が貼られています。

最近のブラウザゲームなどでは当たり前の体験かもしれませんが、ブラウザでここまでスムーズに動かせるのは個人的に驚きでした。併せて「（Blenderなどで作り込まれた3Dモデルさえあれば←大事）自分でも似たようなものを作れるのでは？」と思ったので、必要な技術要素含めて調べながら作ってみます。

この記事では、ブラウザ上で3D空間内をうろうろ歩き回るところまで扱います。モバイルのブラウザまで対応できたら嬉しい。（TODO）
（TODO: 完成イメージ）

なのでNFTの展示に必要な各種Walletとの連携などは対象外です。

また当方、3DモデリングやWebGLはど素人の状態でスタートしていますので、マシュマロ大歓迎です！（マサカリはﾁｮｯﾄｺﾜｲ）
既存サービスは存在しますが、オリジナルの車輪を再開発しようという気概でやっていき。

# 事前準備
## 前提
### 想定読者
- WebGLに興味がある人
- Three.jsを触ってみたい人

### 使用する技術、ライブラリ
- Next.js
- Three.js (react-three-fiber)

## 環境構築
react-three-fiberはdeprecatedなので注意。
```bash
yarn create next-app --typescript
yarn add three @react-three/fiber
```

# 実装
[コード](https://github.com/nszknao/try-threejs)

## まずは公式のチュートリアル
react-three-fiberの最初のチュートリアルを写経します。
https://docs.pmnd.rs/react-three-fiber/tutorials/events-and-interaction

回転する立方体が表示され、クリックすると拡大・縮小するサンプルです。

```js:EventsAndInteraction.tsx
import React, { useRef, useState, VFC } from "react";
import { Canvas, MeshProps, useFrame } from "@react-three/fiber";

const MyRotatingBox: VFC = () => {
  const myMesh = useRef<MeshProps>();
  const [active, setActive] = useState(false);

  useFrame(({ clock }) => {
    const a = clock.getElapsedTime();
    if (myMesh.current === undefined) return;
    myMesh.current.rotation.x = a;
  });

  return (
    <mesh
      scale={active ? 1.5 : 1}
      onClick={() => setActive(!active)}
      ref={myMesh}
    >
      <boxBufferGeometry />
      <meshPhongMaterial color="royalblue" />
    </mesh>
  );
};

export const EventsAndInteraction: VFC = () => {
  return (
    <Canvas>
      <MyRotatingBox />
      <ambientLight intensity={0.1} />
      <directionalLight />
    </Canvas>
  );
};
```

## 3D空間をうろうろ歩いてみる
参考にしたのはThree.jsの公式サンプル。
https://threejs.org/examples/misc_controls_pointerlock.html

```js:PointerLock.tsx
```