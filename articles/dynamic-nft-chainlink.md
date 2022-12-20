---
title: "GitHubのコントリビューションによって動的に変わるNFTを作ってみた"
emoji: "👘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["blockchain", "ethereum", "solidity", "chainlink"]
published: true
---

:::message
本記事は[Ethereum Advent Calendar 2022](https://qiita.com/advent-calendar/2022/ethereum)の19日目の記事です。
:::

# はじめに
本記事では、GitHubのコントリビューションに応じて見た目が変化するNFTについて技術的な背景も含めて紹介します。

いわゆるDynamic NFTですが、既存サービスとNFTの接続点として非常に可能性を感じています。Dynamic NFTでは、ブロックチェーンの外にあるオフチェーンデータをイベントにしてオンチェーンのパラメータを変更することができます。

ゲーム内での活動によってレベルアップし、見た目が変化するような演出は既存サービスでは当たり前に行われていますが、同じような仕様をオンチェーンのNFTでも実現できるようになります。

詳細な仕組みや具体的な応用事例の紹介は以下の記事が詳しいです。
https://blog.chain.link/what-is-a-dynamic-nft/
https://twitter.com/keccak255/status/1588378297658118144

今回はエンジニアとは切っても切れないGitHubへのコントリビューション（=オフチェーンデータ）によって動的に変化するNFTを作ってみました。

# GitHubコントリビューションNFTの開発

## 仕様
- 対象のNFTは誰でもフリーミント可能
- NFTのメタデータには、GitHubへのコントリビューションを表す草画像を使用
- GitHubへのコミットのイベントを検知して、草画像のカラーテーマがランダムで更新される

簡単な概念モデルを作成しました。
![概念モデル](/images/459d3937-fd8c-41da-b953-d8bf71124923.png)

## スタック
- OpenZeppelin, Foundry, Hardhat
- Next.js, wagmi, Tailwind CSS

## 開発ステップ
### NFTを作成
ERC721トークンで実装しました。

メタデータとしてGitHubの草画像をSVG形式で作成しています。
GitHub API v4で取得したコントリビューションの情報をもとに、スマートコントラクト上でSVGを構築し、そのままオンチェーン上に保存しています。

このあたりのSVG構築のソースは、Nounsを参考にさせてもらいました。
https://github.com/nszknao/dapps/blob/main/packages/contract/contracts/libs/MultiPartRLEToSVG.sol

GraphQLで提供されているGitHub API v4では、userに紐づくcontributionCalendarが生えているので、GitHubのページをクロールしてsvgを持ってくる必要は無くてひと安心でした。

GitHub APIを呼んでいるサーバーは、Next.jsのAPI Routesに実装してあります。
草情報は週ごとのコミット数によって0-4のレベルが割り振られたデータがカレンダー形式で返却されますが、SVG構築がしやすいよう連長圧縮をかけています。
（Nounsのソースでも使われていてそのまま流用したほうが実装コストが低いと判断しました。必ずしも圧縮後のサイズが小さくなるわけでは無いです）
https://ja.wikipedia.org/wiki/%E9%80%A3%E9%95%B7%E5%9C%A7%E7%B8%AE

テーマカラーもこの部分でランダムに決定していて、オリジナルのグリーンだけではない草画像がコミットのたびに作成されます。
テーマカラーに含まれる具体的なカラーコードはこちらを参考にしました。
https://github.com/williambelle/github-contribution-color-graph/blob/master/src/js/contentscript.js#L3-L37

草画像の完成イメージはこんな感じです。
![GitHub草画像](/images/0db985da-3755-4965-8bdc-eb85003f0383.png)

ただコミットのイベント検知を再現する部分はGitHubとの連携が必要になり、この記事のトピックからずれてくるため、疑似的にイベントを検知できるようコントラクト上で発火させたい関数をpublicにしています。

https://github.com/nszknao/dapps/blob/main/packages/contract/contracts/GitHubContributionNFT.sol#L63

### コントリビューションを検知してオンチェーンパラメータを更新
肝心のメタデータ更新はChainlinkのオラクルを使って再現しています。

https://github.com/nszknao/dapps/blob/main/packages/contract/contracts/GitHubContributionNFT.sol#L63-L94

コミットイベントをあったときに`commitGitHub`が呼ばれ、オラクルがAPIレスポンスを持って`fulfill`をコールバックします。

余談ですが、独自のノードを立てているわけでは無いので流れはシンプルですが、外部との接続が絡むとローカルテストが複雑になるのはあるあるかと思います。
その際、ローカルから他ネットワークにフォークしてコントラクト関数を実行できるFork Testingが便利でした。
https://book.getfoundry.sh/forge/fork-testing

今回はGoeliテストネットにデプロイしたのですが、テストネットで使えるオラクルをいくつか公式から提供してくれていたので、こちらを利用しました。
https://docs.chain.link/any-api/testnet-oracles/

ドキュメントを見た感じ、これらのオラクルでは対応しているレスポンスが、
- 単一の値（型はuint256, int256, bool, string, bytesそれぞれあります）
- uint256の複数の値
となっているので、これ以外のレスポンスをとるAPIと接続する際は独自ノードを作成する必要があります。

今回はオラクルを体験してみるという目的でシンプルな構成にしたかったので、用意されているノードをありがたく使わせてもらいました。

tokenIdが絡むためどうしてもステートフルになり、コールバック関数の中で草画像データとtokenIdの２種類のデータが必要で、危うく独自ノードの道に進みかけましたが、Chainlinkにリクエストを送る際に作成される`requestId`に紐づける形でtokenIdを保存する案が浮上して事なきを得ました。

# まとめ
GitHubの草画像を使ったDynamic NFTを作成しました。
コントラクトとAPIの実装は何とか完了したのですが、フロントエンドの実装も絶賛進めているので完成したタイミングでまた更新します！

# 参考
https://zenn.dev/yuichkun/articles/b207651f5654b0
