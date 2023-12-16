---
title: "プロジェクトのナレッジに特化したChatbotを作った話"
emoji: "🦙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["LLM", "LangChain", "LlamaIndex"]
published: true
---

:::message
本記事は[LLM Advent Calendar 2023 シリーズ2](https://qiita.com/advent-calendar/2023/llm)の17日目の記事です。
:::

# tl:dr
- LLMの外部知識をRetrieval-Augmented Generation（RAG）で与えてみる
- 技術的にはLlamaIndexのシンプルなインデックスとLangChainを組み合わせ
- 検証用のデータも作成して改善ループが回る仕組みを作ろう

# モチベーション
## 問い合わせに対応する時間的コストを減らしたい
Chatbot導入前に発生していた問題を洗い出すことで、どのような状態になりたいのかあるべき姿を明確にします。

弊社では社内メンバーやクライアントとのコミュニケーション手段として主にSlackを使っているのですが、プロジェクトの数が増えるにつれて問い合わせ対応にかける時間が増えてきました。

問い合わせの内容は大きく3つに分類できます。
- プロダクトの仕様に対する質問
- 追加機能などの要望
- バグ報告

このうち後半の「要望」や「バグ報告」に関しては、ある程度フォーマットを定めることでその後のキャッチボールを減らすことができます。つまりAI導入云々の前に既存のツールを活用することで簡単にペインを解消できます。

ただ、最初の「質問」への応答に関しては一工夫が必要になります。（ドキュメントを確認してくださいと言うのは簡単ですが、クライアントに負担を強いることはできれば避けたい）

プロジェクトが大きくなればその分確認しないといけないドキュメントの分量も増えますし、全体を理解して説明できる人がPdMなど属人化するのを避けるためにも、何らかのシステマチックな解決策が必要だと考えました。

## RAGを使った仕組みを一通り作りたい
あとは技術者としての興味で、RAGでできることを肌感として理解しておきたかったからです😎


# 作ったもの
RAGの全体像のイメージはこちらです。

![RAGの全体像](https://docs.llamaindex.ai/en/latest/_images/basic_rag.png)
*RAGの全体像[^1]*

ユーザーのクエリを元にコーパス（Your data）から関連データ（relevant data）を抽出し、元のクエリと一緒にLLMにプロンプトを渡して結果を取得します。

LlamaIndexの[ドキュメント](https://docs.llamaindex.ai/en/latest/getting_started/concepts.html)にあるRAGアプリケーションの開発ステップに沿って設計しました。

![RAGの開発ステップ](https://docs.llamaindex.ai/en/latest/_images/stages.png)
*RAGの開発ステップ[^1]*

## Loading
まず、外部のデータソースからRAGのパイプラインにデータを取り込みます。

プロジェクトで管理しているドキュメントのデータを使用するのがシンプルで早いと思います。弊社はNotionをメインで使っているのですが、現場のツールに合わてデータローダをLlamaHubやLangChainで探してみてください。
https://llamahub.ai/

生のデータを持ってくるだけであればコピペで終わるのですが、今回はEvaluatingステップでRAGパイプラインの評価も行いたいと考えました。そこで評価用のデータセットも作成したのですが、詳細は[おまけ](#検証用のデータセット作成)に置いておきます。

生データがなくてとりあえず開発したパイプラインの評価がしたい場合は、検証データもセットで提供されているパブリックデータを使うこともできます。以下はBeIRという情報検索分野のベンチマークに使われているデータセットの1つです。
https://huggingface.co/datasets/BeIR/fiqa

:::details FiQAデータセットについて by ChatGPT
「FiQA - Financial Opinion Mining and Question Answering」データセットは、金融関連のテキストデータを中心に構築されたデータセットで、意見マイニング（Opinion Mining）と質問応答（Question Answering）のタスクに特化しています。このデータセットは、金融分野における自然言語処理（NLP）の応用を促進することを目的としています。
:::

このデータセットには元のコーパスも含まれているので、ユーザーからクエリされた質問を含むコンテキストをDBなどのソースデータから抽出するロジックもテストできました。

ちなみに、パブリックになっている日本語データセットは少ないです。英語のデータセットを利用する際に注意しなければならないのが、言語やドメインによる性能の悪化です。そのため開発したパイプラインをプロジェクトに導入する前に試験運用は必要になりますが、まずは素早いイテレーションを回す仕組みを作る、と言う意味では活用する意味はあると思います。

### コード
HuggingFaceのデータセットをLlamaIndexのDocumentとして読み込みます。
```Python
from datasets import load_dataset
from langchain.schema import Document
from llama_index.schema import Document as LlamaIndexDocument

# columns: _id, title, text
fiqa_corpus = load_dataset("BeIR/fiqa", "corpus")

# LlamaIndexのDocument形式に変換
docs: list[LlamaIndexDocument] = []
for data in fiqa_corpus["corpus"]:
  langchain_doc = Document(page_content=data["text"], metadata={"_id": data["_id"]})
  docs.append(LlamaIndexDocument.from_langchain_format(langchain_doc))
```

## Indexing
Loadingステップで読み込んだデータソースを横断的に検索できるようなデータ構造を作成します。

今回はシンプルなVectorStoreIndexしか利用していませんが、コーパスのデータ構造に応じてインデックスを設計することで精度の向上が期待できます。[^4]

当たり前ですが元のソースであるプロジェクトのドキュメントが更新されればこのインデックスも再作成する必要があります。そのため本番運用の際にはインデックス更新フローのシステム設計も考慮しなければなりません。このあたりの運用についても調査したことを[おまけ](#ドキュメントの更新に対応)に記載したので、ドキュメントをNotionで管理している方は参考にしていただければ幸いです！

### コード
```Python
from langchain.chat_models import ChatOpenAI
from langchain.embeddings import OpenAIEmbeddings
from llama_index import LLMPredictor
from llama_index.service_context import ServiceContext
from llama_index.storage.storage_context import StorageContext
from llama_index.node_parser.text.sentence import (
    DEFAULT_CHUNK_SIZE,
    SENTENCE_CHUNK_OVERLAP,
    SentenceSplitter,
)
from llama_index.callbacks.base import CallbackManager
from llama_index.vector_stores import PineconeVectorStore, SimpleVectorStore

# 評価後にパラメータを変更できるように自分で定義しておく
llm_predictor = LLMPredictor(ChatOpenAI(api_key=OPENAI_API_KEY))
embed_model = OpenAIEmbeddings(api_key=OPENAI_API_KEY)
node_parser = SentenceSplitter(
    chunk_size=DEFAULT_CHUNK_SIZE,
    chunk_overlap=SENTENCE_CHUNK_OVERLAP,
    callback_manager=CallbackManager(),
)
service_context = ServiceContext.from_defaults(
    llm_predictor=llm_predictor,
    embed_model=embed_model,
    node_parser=node_parser,
)

vector_store = SimpleVectorStore()
storage_context = StorageContext.from_defaults(vector_store=vector_store)

# Indexを作成
index = VectorStoreIndex.from_documents(
    docs,
    storage_context,
    service_context,
    show_progress=True
)
```

OpenAIのapikeyはモデルのインスタンスを作成するときに明示的に渡すようにしてます。環境変数に埋め込めばコード上はすっきりするのですが、どこでapiコールが走っているのかが隠蔽されてしまうのを防ぐためです。

## Storing
一度作成したIndexを外部のストレージなどに永続化することで、次回以降に埋め込み表現やインデックスを作成するコストを抑えることができます。

### コード
```Python
# ローカルディスクに永続化
index.storage_context.persist(persist_dir="./path/to/folder")

# ローカルディスクから読み込み
from llama_index import load_index_from_storage

vector_store = SimpleVectorStore.from_namespaced_persist_dir(persist_dir="./drive/MyDrive/storage_context")
storage_context = StorageContext.from_defaults(vector_store=vector_store)
index = load_index_from_storage(storage_context, service_context)
```

## Querying
インデックスに対してクエリを投げて検索結果を取得します。

### コード
```Python
query_engine = vector_index.as_query_engine()
ourput = query_engine.query("How to deposit a cheque issued to an associate in my business into my business account?")
output.response # response string
```

## Evaluating
開発したパイプラインの精度を評価する大事なステップです。RAGパイプラインの評価では、大きくRetrievalとGenerationの軸に分けることが多いです[^2]。

### Retrieval
クエリの内容に関連するコンテキストをコーパスから適切に抜き出しているかを評価します。RAG全体像の図でいうYour dataから抽出したrelevant dataの精度です。

- 再現率(Recall): 全ての適合アイテムのうち、検索結果にどれだけ適合アイテムが含まれるか
- 適合率(Precision): 検索結果に含まれる適合アイテムの比率

他にも適合アイテムのランキングを考慮した指標などもありますが、具体的な解説は他の記事に譲ります。

### Generation
応答がコンテキストの内容を使用しているかを評価します。RAG全体像の図でいうresponseテキストの精度ですね。

- Relevancy: コンテキストを考慮し、検索結果がクエリとどの程度一致するか
- Faithfulness: 検索結果がコンテキストとどの程度一致するか

つまり検索結果が的外れな回答をしてないか・テキトーな嘘を言っていないかを、抜き出したコンテキストをもとに定量的に判断します。

そしてこの「定量的」な判断にもLLMを活用しているのが面白いです。つまりコンテキストも数値ではなくテキストの配列であり、検索結果と比較して0-1などの数値で評価するのを人力でやるのはスケールしないため、LLM審判に評価を任せます。

Indexステップで軽く触れた精度改善の施策などそれぞれの打ち手を、ここで計測した指標を元に定量評価→分析することでより堅牢なパイプラインに仕上げることができます！（願望）

### コード
今回はRAG評価のためのライブラリRagas[^3]で実装しました。
```Python
from ragas.langchain import RagasEvaluatorChain
from ragas.metrics import (
    AnswerRelevancy,
    Faithfulness,
    ContextRecall,
    ContextPrecision,
)
from ragas.llms import OpenAI
import pandas as pd

embeddings_model = OpenAIEmbeddings(model="text-embedding-ada-002", openai_api_key=OPENAI_API_KEY)
llm = OpenAI(model="gpt-3.5-turbo-1106", api_key=OPENAI_API_KEY)

faithfulness = Faithfulness(llm=llm)
answer_relevancy = AnswerRelevancy(embeddings=embeddings_model, llm=llm)
context_precision = ContextPrecision(llm=llm)
context_recall = ContextRecall(llm=llm) # need ground_truths

# LangChainでRagas用の評価チェーンを作成
eval_chains = {
    m.name: RagasEvaluatorChain(metric=m) for m in [
          faithfulness,
          answer_relevancy,
          context_precision,
          context_recall,
        ]
}

# 指標ごとの平均値を算出
df_eval = pd.DataFrame(columns=eval_chains.keys())
for question in fiqa_eval["baseline"]["question"]:
  output = qa_chain({"query": question})
  row_eval = {}
  for name, eval_chain in eval_chains.items():
    row_eval[name] = eval_chain(output)[f"{name}_score"]
  df_eval = pd.concat([df_eval, pd.DataFrame([pd.Series(row_eval)])])

# OUTPUT
# ---
# faithfulness         0.822222
# answer_relevancy     0.872568
# context_precision    0.816667
# context_recall       0.812532
df_eval.mean()
```

## おまけ
### 検証用のデータセット作成
LLMアプリケーションを動かすモデルの根底には確率的な性質が伴うため、よりロバストなモデルを作成するためにはプロダクションの分布にマッチしたテストデータで検証することが重要です。

RAGパイプラインの検証に必要な項目はこちら。
| カラム | 説明 |
| ---- | ---- |
| question | 質問 |
| contexts | 回答を生成するために必要なデータ（relevant data） |
| ground_truths | 正解の回答 |
| answer | RAGが生成した回答 |

今回は自前でテストデータを作成したのですが、既存のOSSで自動生成する便利ツールもあるにはあります。
https://api.python.langchain.com/en/latest/evaluation/langchain.evaluation.qa.generate_chain.QAGenerateChain.html
https://docs.ragas.io/en/latest/concepts/testset_generation.html

ただ言語の問題などで得られた結果をそのまま使える状況にはならなかったので、プロンプトは参考にさせてもらいながらスクラッチで実装しました。

RagasのTestsetGeneratorではground_truthsとcontextsを別のプロンプトで作成していたのですが、まとめて出力させた方が関係の薄いコンテキストを拾ってくる割合は減りました。

```Python
from langchain.llms import OpenAI, OpenAIChat
from langchain.chains import LLMChain
from langchain.prompts import PromptTemplate
from langchain_core.pydantic_v1 import BaseModel, Field
from langchain.output_parsers import PydanticOutputParser
import pandas as pd

class TestDataset(BaseModel):
    question: str = Field(description="question")
    ground_truths: list[str] = Field(description="ground truths")
    relevant_sentences: list[str] = Field(description="relevant sentences")

parser = PydanticOutputParser(pydantic_object=TestDataset)

llm = OpenAI(temperature=0, api_key=OPENAI_API_KEY, model="text-davinci-003")
prompt = PromptTemplate(
    template="""You are a teacher coming up with questions to ask on a quiz.
Given the following document, please generate a question and answer based on that document in Japanese.

In addition, please extract relevant sentences from the provided document that can potentially help your answer.
While extracting candidate sentences you're not allowed to make any changes to sentences from given context.

Document Format:
<Begin Document>
{doc}
<End Document>

Be sure to follow the output instructions and ensure correct JSON format.
Be aware that your results often end up outputting in the middle of a statement.
{format_instructions}

These questions should be detailed and be based explicitly on information in the document. Begin!""",
    input_variables=["doc"],
    partial_variables={"format_instructions": parser.get_format_instructions()},
)
llm_chain = LLMChain(prompt=prompt, llm=llm)

# テストデータを保存するDataFrame
df_baseline = pd.DataFrame()
for i, node in enumerate(nodes):
  try:
    context = node.get_content()
    res = llm_chain.predict(doc=context)
    test_dataset = parser.parse(res)

    row_baseline = {
        "question": test_dataset.question,
        "contexts": test_dataset.relevant_sentences,
        "ground_truths": test_dataset.ground_truths,
    }
    df_baseline = pd.concat([df_baseline, pd.DataFrame([pd.Series(row_baseline)])])
    print(f"Success generate dataset! {i}")
  except Exception:
    # 結果のパースに失敗するなど、エラーがあったら無視する
    pass
```

### ドキュメントの更新に対応
本番運用時にはドキュメントの更新に応じてインデックスも再作成が必要になります。ここでは簡単にNotionで作成したページの変更を検知する方法をいくつか紹介します。

- ZapierでNotionデータベースレコードの変更・追加を検知
  - Notionデータベースで管理されているドキュメントであれば有効な方法
  - 削除を検知できない
- Notionデータベースにインデックス作成ステータスを管理するカラムを追加
  - ステータスは作成前、作成済み、不要など
  - 運用の中でドキュメントを更新したらステータスを作成前にする
  - 定期バッチ処理でステータスを見に行って処理したら作成済みステータスにする
  - [Database automations](https://www.notion.so/help/database-automations)がSlack以外の外部サービスに通知を送れるようになればバッチの仕組みも必要なくなりそう🤔

上記の方法によって変更されたページのIDを取得できたら、Notion APIを使って本文を取得してインデックスを再作成→外部ストレージに保存すれば対応できます。

# まとめ
世の中のAI活用が進むことでドキュメンテーションの重要性は今後より一層増えるでしょう。

RAGを含めたLLMアプリケーションには確率的な性質を伴うため、100%期待する結果が得ることは難しいです。そのためまずはバージョン1を爆速で開発し、そこで得られたフィードバックを元に改善ループを回す仕組みが提供価値を最大化につながります。

今回はLlamaIndexを使ったシンプルな実装で終わってしまいましたが、運用の中で見つけた改善点やTipsはまた別記事で公開しようと思います。

それではまたお会いしましょう！

[^1]: https://docs.llamaindex.ai/en/latest/getting_started/concepts.html
[^2]: https://blog.llamaindex.ai/evaluating-multi-modal-retrieval-augmented-generation-db3ca824d428
[^3]: https://docs.ragas.io/en/latest/index.html
[^4]: https://docs.llamaindex.ai/en/stable/optimizing/production_rag.html
