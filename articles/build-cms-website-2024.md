---
title: "2024年にCMS付きのWebサイトをスクラッチで作る"
emoji: "🗂️"
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["nextjs", "cms", "sanity"]
published: false
---

# tl;dr

- CMS付きのWebサイトをスクラッチで構築し、Next.jsやSanityなどの技術スタックを使用
- Sanityはコンテンツ編集体験が優れ、ドキュメントも豊富でカスタマイズ性が高い
- ノーコードとスクラッチの選択を比較し、コストと拡張性のバランスを考慮して決定

# 作ったもの

大学の同期 4 人が集まってできた技術系グループ Active Club（通称アクティ部）のホームページです。今は皆会社で働いていますが、空いた時間でハード・ソフト問わずものづくりや技術全般の発信をしていきます。

内容は、トップページ・ブログコンテンツ・概要ページ・お問い合わせページとシンプルな構成です。

https://activeclub.jp/

採用した技術スタックは、Next.js・Sanity・shadcn/ui・Vercel あたりです。
コードは[GitHub で公開](https://github.com/activeclub/homepage)しています。

## 要件

### Web ページのホスト、ブログコンテンツを編集・管理できる

つまり CMS 機能を持った Web ページにします。

### 他プラットフォームで管理されている記事もキュレーションする

過去にメンバーが個人的に他プラットフォーム（Zenn など）で投稿したコンテンツも、ブログの一覧ページにキュレーションして表示します。

# 設計のポイント

## ノーコードで作るか、スクラッチで作るか

最近では CMS 機能を持つノーコードツールが多数存在し、STUDIO、Webflow、WordPress、Ghost、Squarespace などが候補に挙がりました。選定のポイントは以下の通りです。

- 運用コストの低さ
- 拡張性

### 運用コストの低さ

今回は、データベースを含めたインフラレイヤーを自分たちで管理しないことを意識しました。

フロントの UI デザインに関する変更は、ノーコードでもスクラッチでも必要になることが多く、基本的な手間は変わらないと考えました。しかしインフラを自分たちで管理すると、監視の手間や設計変更時の影響範囲が非常に大きくなります。

ノーコードを選択すれば問題は回避できますが、今どきスクラッチでもやりようはあるのではと思いました。具体的には、SanityにCMS管理を丸投げし、VercelなどのPaaSでホスティングの手間を軽減することで、バックエンドをプロバイダーに任せ、コストを抑えつつアプリ開発に集中できる体制を整えました。

### 拡張性

運用中に新しい機能が必要になることはよくあります。ノーコードツールを使用することで初期開発コストは低くなりますが、追加機能を無理に組み込むと、互換性の問題や予期しない動作によるバグが発生しやすくなります。特に、複雑なカスタマイズを行う際にツールの制約が原因で不具合が生じることが多いです。ノーコードは簡単に始められる反面、拡張性を持たせづらいトレードオフがあります。

今回は、他プラットフォームで管理されている記事もキュレーションする要件がありました。そのため、CMS で作成した記事と他プラットフォームの記事を同じレベルで一覧表示する必要があります。そこまでは既存 CMS でも対応できるのですが、ブログ詳細ページの本文に「元記事はこちら [記事へのリンク]」のようなテキストを入れる必要が出てきます。ユーザー体験の中に不要なページが挟まるのはなるべく避けたかったので、ノーコードを使う選択肢は外しました。

と、つらつら書きましたが最終的には、技術系の発信をするサイトだし、私自身もフロントエンド開発が本職なので「やっぱ自分たちの Web サイトはスクラッチで作るか〜〜」というお気持ちで決めたところはあります🤣

## ヘッドレス CMS はどれがいいか

最終的に Sanity を選んだわけですが、決め手となったポイントはこちらです。

- コンテンツの編集体験
- ドキュメントの豊富さ
- なんでもカスタマイズできそうという安心感

### コンテンツの編集体験

いわゆる WYSIWYG エディタといい、ユーザが入力した内容がで即時にスタイリングと共にプレビューされます。Sanity では Visual Editing と呼ばれているのですが、地味に大きいのが、編集とプレビューが同じ画面内で再現できることです。

![](/images/build-cms-website-2024/sanity-visual-editing-demo.webp)

他のヘッドレス CMS でもプレビューすることはできましたが、別ウィンドウで開く必要がありました。（最近はもう少し便利になったかも？）

リモート環境でビルド済みのコンテンツがリアルタイムに更新されるのは一見魔法のように見えますが、どうやって実現してるんだ？セキュリティ的には大丈夫なの？というのが気になったので調べた内容は末尾に置いておきます。

### ドキュメントの豊富さ

Sanity 固有の機能に関する説明にとどまらず、Next.js、Gatsby、React などのモダンフレームワークでの連携方法についても多くの記事が見つかります。

変化が激しいフロントエンド界隈ですが、Official タグがついている記事であればフレームワークの最新バージョンに追随してくれているものが多かった印象です。私のような新参者にも優しくとても助かりました。

参考に、NextJS ユーザであればこちらの記事が参考になりました。空っぽの状態からスキーマの作成、型の生成、フロントからのデータフェッチまで網羅されています。

https://www.sanity.io/plugins/next-sanity

### なんでもカスタマイズできそうという安心感

Sanity の存在を知ってまだ 1 週間程度なので何がカスタマイズできるかを詳細に語ることはできないのですが、フロントエンド UI だけでなくコンテンツ管理側も含めて拡張性の余地が残されている印象を受けました。

たとえばコンテンツ管理をするダッシュボード画面を Sanity Studio というのですが、このダッシュボード内に独自のボタンやアクションを追加できる Studio Tools があります。カスタムボタンを追加して、特定の外部 API を呼び出すアクションを実行するなど、独自の業務プロセスに合わせた操作を容易に追加できます。

https://www.sanity.io/docs/studio-tools

一例ですが、SSR でビルド時にブログコンテンツを取得している場合、更新があれば再ビルドが必要です。そのため、ダッシュボードから記事の「更新」ボタンを押してもサイトには即時反映されません（ISR すればいいやんというのは置いといて）。そこで、ダッシュボードに「再ビルド」ボタンを独自に追加することで、更新時に CI にビルドアクションを投げるといったカスタマイズが可能です。

## 編集した内容のリアルタイム更新を Sanity がどのように実現しているのか

気になったのは 2 点です。コンテンツを取得している部分のコードを合わせて記載します。

### どうやって実現してる？

Sanity Studio 上での編集を検知してリフェッチしているわけですが、以下の Loaders 機能がライブアップデート機能を提供しているようです。

https://www.sanity.io/docs/loaders

詳細の技術仕様までは記載がありませんでしたが、NextJS ユーザの場合は next-sanity から提供されるクライアントを使用することで Loaders の恩恵を受けられるとのことです。

### セキュリティ的には大丈夫？

NextJS の Draft Mode が有効になっている場合に、Sanity 上で発行したトークンをリクエストに含めてコンテンツを取得します。トークンを持っている主体しか下書きのコンテンツは取得できないので、環境変数に埋め込んでいればひとまず問題なさそうです。

```ts
import { createClient, QueryOptions, type QueryParams } from "next-sanity";
import { draftMode } from "next/headers";

import { apiVersion, dataset, projectId, token } from "./env";

export const client = createClient({
  projectId,
  dataset,
  apiVersion,
  useCdn: false,
  perspective: "published",
  resultSourceMap: "withKeyArraySelector",
  stega: {
    enabled: true,
    studioUrl: `${process.env.NEXT_PUBLIC_TEST_BASE_PATH || ""}/studio#`,
  },
});

export async function sanityFetch<QueryResponse>({
  query,
  params = {},
  revalidate = 60, // default revalidation time in seconds
  tags = [],
}: {
  query: string;
  params?: QueryParams;
  revalidate?: number | false;
  tags?: string[];
}) {
  const isDraftMode = (await draftMode()).isEnabled;

  if (isDraftMode && !token) {
    throw new Error("Missing environment variable SANITY_API_READ_TOKEN");
  }

  const queryOptions: QueryOptions = {};
  let maybeRevalidate = revalidate;

  if (isDraftMode) {
    queryOptions.token = token;
    queryOptions.perspective = "previewDrafts";
    queryOptions.stega = true;

    maybeRevalidate = 0; // Do not cache in Draft Mode
  } else if (tags.length) {
    maybeRevalidate = false; // Cache indefinitely if tags supplied
  }

  return client.fetch<QueryResponse>(query, params, {
    ...queryOptions,
    next: {
      revalidate: maybeRevalidate,
      tags,
    },
  });
}
```

# まとめ

気がついたら Sanity の紹介がメインになっていましたが、今回の記事では、CMS付きのWebサイトをスクラッチで作成するプロジェクトについて解説しました。ノーコードツールとスクラッチ開発の選択肢を比較し、最終的にSanityやVercelを採用することで、運用コストと拡張性のバランスを取れたと思います。

Sanityの選定理由としては、コンテンツ編集体験の優秀さ、豊富なドキュメント、そしてカスタマイズ性の高さが挙げられます。これにより、必要な機能を柔軟に追加しながら、効率的に開発を進めることができました。

このプロジェクトが、同様のCMS開発を考えている方々にとって参考になれば幸いです。

それではまたお会いしましょう！
