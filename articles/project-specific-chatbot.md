---
title: "プロジェクトのナレッジに特化したChatbotを作った話"
emoji: "🦙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["LLM", "LangChain", "LlamaIndex"]
published: false
---

# tl:dr
- 社内のプロジェクトに特化したChatbotを作った話
- 技術的にはRetrieval-Augmented Generation（RAG）あたり
- xxすることで回答精度が向上したよ

# モチベーション
## 問い合わせに対応する時間的コストを減らしたい
Chatbot導入前に発生していた問題を洗い出すことで、どのような状態になりたいのかあるべき姿を明確にします。

弊社では社内メンバーやクライアントとのコミュニケーション手段として主にSlackを使っているのですが、プロジェクトの数が増えるにつれて問い合わせ対応にかける時間が増えてきました。

**TODO: 問い合わせ内容の3分類スライド**

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
LlamaIndexの[ドキュメント](https://docs.llamaindex.ai/en/latest/getting_started/concepts.html)にRAGアプリケーションのステップについて記載があったので、その順番に沿って設計しました。

![RAGの開発ステップ](https://docs.llamaindex.ai/en/latest/_images/stages.png)
*RAGの開発ステップ[^1]*

## Loading
まず、外部のデータソースからRAGのパイプラインにデータを取り込みます。

プロジェクトで管理しているドキュメントのデータを使用しても良いのですが、最後のEvaluatingステップでパイプラインの評価を行いたいので、検証データもセットで提供されているパブリックデータを使いました。
https://huggingface.co/datasets/BeIR/fiqa

:::details FiQAデータセットについて by ChatGPT
「FiQA - Financial Opinion Mining and Question Answering」データセットは、金融関連のテキストデータを中心に構築されたデータセットで、意見マイニング（Opinion Mining）と質問応答（Question Answering）のタスクに特化しています。このデータセットは、金融分野における自然言語処理（NLP）の応用を促進することを目的としています。
:::

パブリックになっている日本語データセットは少ないです。英語のデータセットを利用する際に注意しなければならないのが、言語やドメインによる性能の悪化です。そのため開発したパイプラインをプロジェクトに導入する前に試験運用は必要になりますが、まずは素早いイテレーションを回す仕組みを作る、と言う意味では活用する意味はあると思います。

またこのデータセットには元のコーパスも含まれているので、ユーザーからクエリされた質問を含むコンテキストをDBなどのソースデータから抽出するロジックもテストできました。以下の図でいう「relevant data」が抽出されたコンテキストです。

![RAGの全体像](https://docs.llamaindex.ai/en/latest/_images/basic_rag.png)
*RAGの全体像[^1]*

RAGの検証方法について調べる中で見つけたベンチマークに含まれるカラムはおおよそ以下のような構造でした。
- question
- answer
- contexts

質問に答えるときに利用したコンテキストをカラムに含めることで、検証の際に元のコーパス（たいていは膨大）を参照せずに済むと言うメリットは大きいと思います。ただ素人目にみたらこの「contexts」を抽出するのも難しいのでは？という疑問がありました。

RAGの全体像の図を見ても分かる通り、LLMにはプロンプトとしてqueryの他にrelevant dataしか与えられないため、ここの情報が適切に抽出できていることはレスポンスの精度に大きく影響します。そう言った意味でもFiQAのデータセットでは、コーパスの情報からcontextsを抽出するロジックも検証できるのは助かります。

（私自身NLPの専門家でははないので、テキトーなことを言っていたらご指摘ください🙏）

### 実行コード
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
Lodingステップで読み込んだデータソースを横断的に検索できるようなデータ構造を作成します。

**TODO: データの構造に応じたインデックスの設計について**

### 実行コード
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

### 実行コード
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

### 実行コード
```Python
query_engine = vector_index.as_query_engine()
query_engine.query("How to deposit a cheque issued to an associate in my business into my business account?")
```

## Evaluating
開発したパイプラインの精度を評価する大事なステップです。RAGパイプラインの評価では、大きく分けてRetrievalとGenerationの軸に分けることが多いです[^2]。

### Retrieval
クエリの内容に関連するコンテキストをコーパスから適切に抜き出しているかを評価します。

- 再現率(Recall): 全ての適合アイテムのうち、検索結果にどれだけ適合アイテムが含まれるか
- 適合率(Precision): 検索結果に含まれる適合アイテムの比率

他にも適合アイテムのランキングを考慮した指標などもありますが、具体的な解説は他の記事に譲ります。

### Generation
応答がコンテキストの内容を使用しているかを評価します。

- Relevancy: コンテキストを考慮し、検索結果がクエリとどの程度一致するか
- Faithfulness: 検索結果がコンテキストとどの程度一致するか

つまり検索結果が的外れな回答をしてないか・テキトーな嘘を言っていないかを、抜き出したコンテキストをもとに定量的に判断します。

そしてこの「定量的」な判断にもLLMを活用しているのが面白いです。つまりコンテキストも数値ではなくテキストの配列であり、検索結果と比較して0-1などの数値で評価するのを人力でやるのはスケールしないため、LLM審判に評価を任せます。

### 実行コード
今回はRAG評価のためのライブラリRagas[^3]を使用して実現しました。
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
          # context_recall,
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
df_eval.mean()
```

# まとめ
プロジェクトの中で決まったことをドキュメント化しておくことは今後より重要になるでしょう。

それではまたお会いしましょう！

[^1]: https://docs.llamaindex.ai/en/latest/getting_started/concepts.html
[^2]: https://blog.llamaindex.ai/evaluating-multi-modal-retrieval-augmented-generation-db3ca824d428
[^3]: https://docs.ragas.io/en/latest/index.html