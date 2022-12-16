---
title: "p5.jsのExamplesをReactで写経してみた"
emoji: "🖼"
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["react", "typescript", "p5js"]
published: true
---
こんにちは、にしざかです。
普段はWebのインフラからフロントまで薄く広く開発しています。デザインはできないです。

最近のNFTプロジェクトの盛り上がりを感じる中で、自分もクリエイターとして作品作りをしたいな、でも何のメディアで作ろうかなと漠然と考えていました。いわゆる2次元のキャンバスに描かれるようなアート作品を0から作ることは、自分のスキルとマッチしていないのが明らかなので何から手をつけようか調べていました。

そんな時に[Generative Art](https://ja.wikipedia.org/wiki/%E3%82%B8%E3%82%A7%E3%83%8D%E3%83%AC%E3%83%BC%E3%83%86%E3%82%A3%E3%83%96%E3%82%A2%E3%83%BC%E3%83%88)という分野があることを知りました。ソフトウェアのアルゴリズムを使って、人工物とも自然物とも言えない有機的な作品を作ることができます。これなら開発の楽しさを味わいながら面白い作品が作ることができそうです。そんなクリエイティブコーディングをサポートしてくれるツールの1つがp5.jsです。

（NFTプロジェクトはその背景にあるストーリーやナラティブ、そこに参加しているユーザーコミュニティに意味があると感じています。ただ、この記事ではNFT化されたコンテンツ、特にアート系の作品そのものへの興味がモチベーションです。）

# p5.jsについて
https://p5js.org/
[Processing](https://processing.org/)をJSに移植したライブラリです。JSで実行できるクリエイティブコーディングのツールを謳っていて、例えばこんな作品が作れます。

![Sine Cosine](https://storage.googleapis.com/zenn-user-upload/7b7d1a21d27644a5e7c72751.gif)

ちなみに、描画したcanvasをキャプチャするときはこちらのコードを使いました。各フレームでpngが保存されるので、あとはQuickTimePlayerで動画化してgifに変換するだけ。
https://editor.p5js.org/jnsjknn/sketches/B1O8DOqZV

# 実装
公式にあるExamplesの実装を写経したログを残します。
[コード](https://github.com/nszknao/generative-art)

## 環境
- Next.js
- TypeScript

## 完成形
p5.js用のラッパーComponentにsketchを渡して描画します。sketchには描画するコンテンツや、セットアップ時の処理などが書かれています。

p5をimportしたときに`ReferenceError: window is not defined`が出たので、SSRを無効化しています。

```js:index.tsx
import dynamic from "next/dynamic";
import type { NextPage } from "next";

const P5Wrapper = dynamic(() => import("src/P5Wrapper"), { ssr: false });
import { sineCosine } from "src/sketches/sine-cosine";

const IndexPage: NextPage = () => {
  return <P5Wrapper sketch={sineCosine} />;
};

export default IndexPage;
```

## p5.js用のラッパーComponent
本家のExamplesでは生のJSを使った例が載っているので、Reactで簡単に使えるようラッパーを用意しました。

```js:P5Wrapper.ts
import p5 from "p5";
import React, { createRef, useEffect, useState } from "react";

interface Props {
  sketch: any;
}

const P5Wrapper: React.VFC<Props> = (props) => {
  const [instance, setInstance] = useState<p5>();
  const wrapper = createRef<HTMLDivElement>();

  useEffect(() => {
    if (wrapper.current === null) return;
    setInstance(new p5(props.sketch, wrapper.current));
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [props.sketch]);

  return <div ref={wrapper} />;
};

export default P5Wrapper;
```

## sketchファイル
sketchファイルでは描画するコンテンツを定義しています。

`setup()`でcanvasを作成し、`draw()`でお絵描きしてます。静的な描画だけではなくてアニメーションや、WebGLを使った3Dグラフィックスにも対応しているので、[公式のリファレンス](https://p5js.org/reference/)を見ながらクリエイティブ活動を楽しみめます。

```js:sine-cosine.ts
import type p5 from "p5";

export const sineCosine = (p: p5) => {
  p.setup = () => {
    p.createCanvas(720, 400, "webgl");
  };

  p.draw = () => {
    p.background(250);
    p.rotateY(p.frameCount * 0.01);

    for (let j = 0; j < 5; j++) {
      p.push();
      for (let i = 0; i < 80; i++) {
        p.translate(
          p.sin(p.frameCount * 0.001 + j) * 100,
          p.sin(p.frameCount * 0.001 + j) * 100,
          i * 0.1
        );
        p.rotateZ(p.frameCount * 0.002);
        p.push();
        p.sphere(8, 6, 4);
        p.pop();
      }
      p.pop();
    }
  };
};
```

# p5.jsのソース深堀り
sketchの書き方で気になる箇所があったので、ソース見ながら実装を見ていきたいと思います。
https://github.com/processing/p5.js/

### p.push() / p.pop()
Strokeの幅や色といった描画スタイルの設定を保存・復元できるAPI。グローバルにスタイルの履歴`_styles`を持たせている。描画を担っているのがRendererオブジェクト`_render`ぽいので、`rect()`などの描画イベントが呼ばれたタイミングでスタイルを流し込んでいそう。

```js:p.push()
p5.prototype.push = function() {
  this._styles.push({
    props: {
      _colorMode: this._colorMode
    },
    renderer: this._renderer.push()
  });
};
```

```js:p.pop()
p5.prototype.pop = function() {
  const style = this._styles.pop();
  if (style) {
    this._renderer.pop(style.renderer);
    Object.assign(this, style.props);
  } else {
    console.warn('pop() was called without matching push()');
  }
};
```

# 今後の展望
- パラメータを操作できるインターフェースを実装
- 動物やキャラクターをモチーフにしたアイコンを自動生成
- Three.jsなどの他のグラフィックライブラリを試す

おすすめのライブラリなどありましたら、ぜひ教えてください！