---
tags:
  - Kaggle
  - 教育
  - MAP3
  - DeBERTa
  - LLM
startdate: 2023-07-12
enddate: 2023-10-11
---
# Kaggle - LLM Science Exam
[https://www.kaggle.com/competitions/kaggle-llm-science-exam](https://www.kaggle.com/competitions/kaggle-llm-science-exam)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、大規模言語モデル (LLM) を活用して、大学レベルの科学（STEM：科学・技術・工学・数学）に関する多肢選択問題（5択）に高精度で解答するシステムを構築することです。
* **背景:** LLMは近年目覚ましい発展を遂げ、多様なタスクで高い能力を示していますが、専門的な知識を要する問題や、微妙なニュアンスを理解する必要がある問題に対して、その推論能力と知識の正確性をさらに向上させる必要があります。特に、外部知識を効果的に参照し、幻覚（Hallucination）を抑えつつ正確な解答を導き出す技術（Retrieval-Augmented Generation - RAG）が重要となっています。
* **課題:** 与えられた問題文（prompt）と5つの選択肢（A～E）に対して、最も適切な選択肢を特定することです。問題は専門的な科学知識を必要とし、しばしば選択肢間の差異が僅かであったり、複雑な推論が求められたりします。外部知識ソース（Wikipediaなど）を効率的かつ正確に検索し、その情報をLLMの推論プロセスに統合するRAGパイプラインの設計・最適化が中心的な課題となります。

**データセットの形式 (Dataset Format)**

提供される主なデータは、多肢選択問題のセットと、外部知識ソースとしてのWikipediaデータです。

1.  **問題データ:**
    * `train.csv`, `test.csv`: 各問題のデータが含まれます。
        * `id`: 問題の一意なID。
        * `prompt`: 問題文。
        * `A`, `B`, `C`, `D`, `E`: 5つの選択肢のテキスト。
        * `answer`: 正解の選択肢（`train.csv`のみ）。
    * 追加のトレーニングデータセットが公開されることもあります（例: 60k STEM questions）。
2.  **外部知識ソース (Wikipedia):**
    * 公開データセットとして、前処理（チャンク分割、埋め込みベクトル計算済みなど）されたWikipediaダンプが提供されることがあります。
        * 例: `wikipedia-22-12-en-embeddings-all-minilm-l6-v2`
    * 参加者は、自身でWikipediaダンプ（通常版またはCirrusSearch版など）をダウンロードし、前処理（クリーニング、チャンク分割、インデックス作成）を行って利用することもできます。
3.  **`sample_submission.csv`**:
    * 提出フォーマットのサンプル。`id`と、予測される選択肢の順位（最も確信度が高いものから順に3つ）を示す`prediction`列（例: `A B C`）を持ちます。

**評価指標 (Evaluation Metric)**

* **指標:** **Mean Average Precision @ 3 (MAP@3)**
* **計算方法:** 各問題に対して、モデルは5つの選択肢を確信度の高い順にランク付けします。
    1.  **Precision@k:** 上位k個の予測の中に正解が含まれていれば 1/k、そうでなければ 0。
    2.  **Average Precision@3 (AP@3):** 上位3つの予測に対するPrecisionの平均値。具体的には、(Precision@1 + Precision@2 + Precision@3) / 3 を計算する際に、正解がk番目に見つかった場合のみ Precision@k に 1/k を設定し、それ以外は 0 として計算する場合が多い（詳細な計算式は要確認だが、上位3つのランク付けの質を評価する）。
    3.  **Mean Average Precision@3 (MAP@3):** 全てのテスト問題におけるAP@3の平均値。
* **意味:** モデルが予測した上位3つの選択肢の中に、どれだけ確実に正解を含めることができているかを評価します。単に正解を選ぶだけでなく、確信度の順序付けも重要になります。スコアは**高い**ほど良いモデルと評価されます。

要約すると、このコンペティションは、外部知識（Wikipedia）を活用するRAGアプローチを用いて、科学分野の5肢択一問題に解答するLLM応用タスクです。データは問題文・選択肢とWikipediaコーパスで構成され、性能は上位3位までの予測精度を評価するMAP@3（高いほど良い）によって測られます。

---

**全体的な傾向**

このコンペティションでは、**Retrieval-Augmented Generation (RAG)** がほぼ必須のアプローチとなりました。質問に関連する情報をWikipediaから検索 (Retrieve) し、その情報をコンテキストとしてLLM (Reader) に与えて解答させるパイプラインが主流でした。

* **Retrieval (検索):**
    * **知識ソース:** Wikipediaダンプ。公開データセットに加え、多くのチームが独自に最新版を処理・クリーニング（特に数値や数式、Luaスクリプト展開の改善）して使用しました。CirrusSearchダンプ（レンダリング済みテキスト）の利用も見られました。
    * **インデックスと検索:**
        * **密検索 (Dense):** Sentence Transformer系モデル (`all-MiniLM-L6-v2`, `e5`, `gte`, `bge`, `instructor` など) で埋め込みベクトルを作成し、FAISSなどで類似度検索。
        * **疎検索 (Sparse):** BM25 (TF-IDFベース) も有効であり、密検索と併用されました。Pyserini (Lucene) が利用されました。
        * **ハイブリッド:** 密検索と疎検索の結果を組み合わせる。
    * **チャンク戦略:** 記事全体、段落、文、固定長など様々な粒度でのチャンク化。短いチャンクで検索し、周辺情報を含めてコンテキストとする戦略も。
    * **クエリ:** 質問文 (`prompt`) だけでなく、選択肢 (`A`~`E`) も含めて検索クエリを生成（例: `"prompt A"`, `"prompt B"`）。
    * **Reranker:** 検索結果をCross-Encoderモデル (DeBERTaベースなど) で再ランク付けし、より関連性の高いコンテキストを上位にする。Rerankerモデルの独自学習も有効でした（特にハードネガティブを利用）。
* **Reader (解答生成/選択):**
    * **モデル:** **DeBERTa-v3-large/base** が強力なベースラインとして広く使われました。さらに、**Llama-2 (7B, 13B, 70B)**, **Mistral (7B)**, Falcon (180B), Platypus2 (70B), sheep-duck-llama-2 (70B), Xwin などの**LLM**がファインチューニングまたはゼロショット/フューショットで用いられました。70Bクラスのモデルを時間内に実行するための工夫（量子化、レイヤー単位処理、KVキャッシュ）も重要でした。
    * **学習:**
        * **データ:** 公開データセット (60k, 40k, MMLUなど) や、GPT-3.5/4で独自生成したデータセットを使用。
        * **ファインチューニング:** **QLoRA**による効率的な学習が主流。
        * **入力形式:** コンテキスト、質問、選択肢を整形して入力。
        * **出力/損失:**
            * **Multiple Choice:** DeBERTaで標準的な、選択肢ごとのスコアを予測する形式 (Cross Entropy Loss)。
            * **Causal LM (Token Prediction):** 次トークンとして選択肢ラベル (A-E) の確率を予測。
            * **Binary Classification:** 各選択肢が正解か否かを独立に予測。選択肢間の比較情報を付加する工夫も。
            * **Reward Modeling:** 正解/不正解ペアで学習。
* **アンサンブルと後処理:**
    * **多様なモデル/コンテキスト:** 異なるRetriever/Reranker、異なるReaderモデル (DeBERTa vs LLM, 異なるLLM)、異なるコンテキスト戦略 (Wikipediaダンプ、チャンクサイズ、取得数 k) の結果をアンサンブルすることがスコア向上に不可欠でした。
    * **アンサンブル手法:** 重み付き平均、確率の最大値を取る、XGBRankerなどの学習ベースの手法。
    * **段階的推論:** 軽量モデルで簡単な問題を処理し、難問のみを大規模モデルで処理する戦略で、計算時間と精度を両立。
    * **TTA:** 選択肢の順序をシャッフルして複数回推論し、結果を平均化。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/kaggle-llm-science-exam/discussion/446422)**

* **アプローチ:** RAGパイプライン。多様なRetrieverとLLM (7B/13B) のファインチューニングモデルのアンサンブル。H2O LLM Studioを活用。
* **アーキテクチャ:** Retriever (`e5`, `gte`, `bge` 系)、Reader (Llama-2 7B/13B, Mistral-7B, XGen-7B)。
* **アルゴリズム:** 埋め込み類似度検索 (FAISS)。Binary Classification Head (Reader)。重み付き平均アンサンブル。
* **テクニック:**
    * **Retrieval:** 複数モデル、複数Wikipediaダンプ (通常版、CirrusSearch版) を使用。クエリは`"prompt A"`なども利用。GPU上での高速類似度検索。
    * **Reader:** 7B/13BモデルをQLoRAでファインチューニング。バイナリ分類形式（各選択肢が正解か否か）。KVキャッシュ活用による高速化。他選択肢の平均Logitを入力に追加する工夫も。
    * **コンテキスト:** 訓練時は3チャンク、推論時は5チャンクを使用。コンテキストの順序も重要。
    * **データ:** 公開データセット。RAGで生成したコンテキストで学習。
    * **アンサンブル:** 5つの7Bモデル + 1つの13Bモデルをブレンド。

**[2位](https://www.kaggle.com/competitions/kaggle-llm-science-exam/discussion/448256)**

* **アプローチ:** DeBERTa中心のRAG。カスタムReranker、MLM、Mistral、XGBRankerによる段階的アンサンブル。
* **アーキテクチャ:** Retriever (BM25 on Lucene via pyserini)、Reranker (DeBERTa-v3-base)、Reader (DeBERTa-v3-large, Mistral-7B)、MLM (DeBERTa-v3-large)、XGBRanker (アンサンブル)。
* **アルゴリズム:** BM25検索、Cross-Encoder再ランク付け、Multiple Choice、Masked Language Modeling、勾配ブースティング (XGBRanker)。
* **テクニック:**
    * **Retrieval:** 全Wikipedia (graelo/wikipedia) を文単位でチャンク化しLuceneでインデックス化 (BM25)。
    * **Reranker:** DeBERTaベース。教師モデル(DeBERTa-large)の予測に基づき、最適なコンテキストを予測するように学習。
    * **Reader:** DeBERTa (Multiple Choice)、Mistral (Multiple Choice)。データセットの選択肢数に応じて損失をスケーリング。
    * **MLM:** 選択肢間の差異が小さい場合に特化したモデル。ペアワイズのアライメントで差異を特定し予測。
    * **Cross Reference:** "None of the above" のような選択肢に対応するための第2段階ランカー。
    * **アンサンブル:** XGBRankerを用いて、Retrieverスコア、Readerのlogit、"None of the above"フラグなどを特徴量として最終ランクを学習。

**[3位](https://www.kaggle.com/competitions/kaggle-llm-science-exam/discussion/446358)**

* **アプローチ:** RAGパイプライン。独自Wikipedia処理、チューニング済みReranker、MLMアンサンブル + LLMによる難問処理。
* **アーキテクチャ:** Retriever (TF-IDF + `bge-small`, `all-MiniLM-L6-v2`)、Reranker (ibm/re2g-reranker-nq ファインチューン)、Reader (DeBERTa-v3-large x5, Electra x3, RoBERTa x5)、LLM (Platypus2 70B, Xwin 7B)。
* **アルゴリズム:** 密+疎検索、Cross-Encoder再ランク付け、Multiple Choice。
* **テクニック:**
    * **Wikipedia処理:** wikiextractorを修正し数値欠損を改善。記事単位ではなく、全文章を対象に検索 (passage level)。
    * **Retrieval:** TF-IDF + 2種の埋め込みモデルで検索し、結果を結合。
    * **Reranker:** 公開データセット (70k) を用いて学習。ハードネガティブが重要だが、事前学習済みモデルから開始するか2段階学習が必要。
    * **Reader (MLM):** 3種類のアーキテクチャ (DeBERTa, Electra, RoBERTa) をアンサンブル。
    * **Reader (LLM):** MLMアンサンブルで確信度の低い難問 (Top 500) のみLLM (Platypus2, Xwin) で再推論。
    * **アンサンブル:** MLMアンサンブルの結果とLLMの結果を単純に置換（LLM使用時）。

**[4位](https://www.kaggle.com/competitions/kaggle-llm-science-exam/discussion/446307)**

* **アプローチ:** RAGパイプライン。Elasticsearchによるキーワード検索と複数段階の再ランク付けを組み合わせた高品質なコンテキスト生成。DeBERTaモデル。
* **アーキテクチャ:** Retriever (Elasticsearch + Sentence Transformers)、Reader (DeBERTa-v3-large)。
* **アルゴリズム:** キーワード検索 (Elasticsearch)、編集距離、埋め込み類似度検索。Multiple Choice。コンテキストアンサンブル。
* **テクニック:**
    * **Wikipedia処理:** CirrusSearchダンプを文単位で分割しElasticsearchでインデックス化。
    * **Retrieval (Stage 1):** 質問/選択肢を文単位で分割し、キーワード抽出（Stopword除去）してElasticsearchで類似文を検索。
    * **Retrieval (Stage 2 - Context生成):** 3種類のコンテキストを生成。v3: Elasticsearchスコア順。v5: 質問文との編集距離順。v7: Sentence Transformer (`msmarco-bert-base-dot-v5`, `all-mpnet-base-v2`) による埋め込み類似度順。
    * **Reader:** DeBERTa-v3-largeを使用。入力トークン長を拡張 (512→768→1280)。プロンプトと選択肢を先に配置し、残りをコンテキストで埋める。
    * **コンテキストアンサンブル:** 3種類 (v3, v5, v7) のコンテキストでそれぞれ推論し、結果をアンサンブル。
    * **効率化:** Elasticsearchのインデックスファイルを `/kaggle/input` にシンボリックリンクし、`/kaggle/temp` のI/O不安定性を回避。

**[5位](https://www.kaggle.com/competitions/kaggle-llm-science-exam/discussion/446293)**

* **アプローチ:** RAGパイプライン。独自処理のWikipedia + 疎密両方の検索 + Llama 2 70Bモデル。
* **アーキテクチャ:** Retriever (BM25/Pyserini, instructor-xl, bge-large)、Reader (Llama-2 7B/13B/70B, Mistral-7B)。
* **アルゴリズム:** BM25検索、埋め込み類似度検索 (FAISS)、QLoRA、段階的推論。
* **テクニック:**
    * **Wikipedia処理:** wikitextparser + カスタムテンプレート処理で数値・数式情報を保持。段落単位で分割。
    * **Retrieval:** 疎 (BM25/Pyserini) + 密 (instructor-xl/FAISS on 全文, bge-large/FAISS on STEM 270k) の3種の結果を結合。コンテキストに記事タイトルを追加。
    * **Reader:** Llama-2 70B をQLoRA (4bit量子化) + xformers でKaggle環境で実行。Mistral-7Bも使用。Instruction Tuningなしのベースモデルを使用。出力は次トークン予測 (A-E)。
    * **70Bモデル実行:** 量子化重みをデータセットとして登録し、レイヤー単位で推論。xformersでメモリ効率化 (約6GB使用)。
    * **TTA:** 選択肢の順序を5パターン入れ替えて推論し平均化。推論効率化のため、共通部分 (コンテキスト+質問) の計算は1回に。
    * **段階的推論:** 1st: Mistral-7Bアンサンブル → 2nd: Llama-70B (下位40%) → 3rd: Llama-70B + 長コンテキスト (下位5%)。

**[6位](https://www.kaggle.com/competitions/kaggle-llm-science-exam/discussion/447647)**

* **アプローチ:** RAGパイプライン。カスタムSTEM Wiki + ファインチューン済みRetriever/Reranker + DeBERTa/LLMアンサンブル + 難問用70Bモデル。
* **アーキテクチャ:** Retriever (`gte-base`, `bge-base-en-v1.5` ファインチューン)、Reranker (DeBERTa-v3-base ファインチューン)、Reader (DeBERTa-v3-large/base, OpenOrca/Mistral-7B-OpenOrca ファインチューン)、LLM (platypus2-70b-instruct, sheep-duck-llama-2 70B)。
* **アルゴリズム:** 埋め込み類似度検索、NCE Loss (Retriever学習)、Cross-Encoder再ランク付け、Multiple Choice (DeBERTa)、LoRA (Mistral学習)、ゼロショット推論 (70B LLM)。
* **テクニック:**
    * **STEM Wiki Corpus:** Wikipediaカテゴリメタデータを用いてSTEM関連 (~500k) 記事を抽出し、Wikipedia-APIで取得。セクション分割後、約300トークンでチャンク化（Overlapあり/なしの2種）。
    * **Retriever学習:** GPT等で生成した合成MCQデータを利用。NCE Loss + In-batch Hard Negativesで学習。2種のRetrieverの結果を統合 (Top 10)。
    * **Reranker:** DeBERTa-baseをCross-Encoderとして学習。Top 2~4のコンテキストを選択。
    * **Reader (DeBERTa):** Span Classification（選択肢トークンをPooling + Cross-option情報）、標準Multiple Choice、PET (Pattern Exploiting Training) など多様な学習戦略。
    * **Reader (Mistral):** LoRAでファインチューニング。Cross-option設定。
    * **Reader (70B LLM):** DeBERTa/Mistralで確信度の低い難問 (Top 5-10%) のみゼロショット推論。2トークン (yes/noなど) の確率差で解答。

**[7位](https://www.kaggle.com/competitions/kaggle-llm-science-exam/discussion/447155)**

* **アプローチ:** 複数のRetriever戦略 + DeBERTa/LLMアンサンブル + 段階的推論。
* **アーキテクチャ:** Retriever (TF-IDF, Sentence Transformer/gte-small, Cohere TF-IDF)、Reader (DeBERTa-V3-large, Mistral-7B, OpenOrca-Platypus2-13B, Llama2-chat-AYT-13B)。
* **アルゴリズム:** 密+疎検索、Multiple Choice、Causal LM (Token Prediction)、Reward Modeling。重み付き平均アンサンブル。
* **テクニック:**
    * **Retrieval:** 4種類の戦略を併用。
        * 記事検索→文検索 (TF-IDF on Wikipedia-20230801)。
        * TF-IDF on MB 270K。
        * Sentence Transformer (gte-small) on MB 270K。
        * Cohere TF-IDF (BERT tokenizer, 段落単位)。
    * **Reader (DeBERTa):** 複数モデル (ベース、squad2ファインチューン版)、複数max_length (512, 1024)。
    * **Reader (LLM):** CausalLM形式 (次トークン予測 A-E) とReward Modeling形式 (正解らしさスコアリング) の両方を学習・利用。QLoRA使用。選択肢シャッフル学習。
    * **段階的推論:** DeBERTaで容易な問題をフィルタリング (max prob > 0.7) し、残りをLLMで推論。
    * **アンサンブル:** 8つのDeBERTaモデルと3つのLLMモデルを重み付き平均（LLMの重みを高く）。重みは検証データで決定。

**[10位](https://www.kaggle.com/competitions/kaggle-llm-science-exam/discussion/446248)**

* **アプローチ:** 複数のRetriever + 単一DeBERTaモデル + Max Probabilityアンサンブル + TF-IDF後処理。
* **アーキテクチャ:** Retriever (FAISS/embedding, TF-IDF)、Reader (DeBERTa-v3-large)。
* **アルゴリズム:** 密+疎検索、Multiple Choice、Max Probabilityアンサンブル、TF-IDF後処理。
* **テクニック:**
    * **Retrieval:** 4種類のWikipediaデータ/検索手法を併用。
        * Dumpデータ1 (wikiextractor): 記事検索(FAISS)→文検索(FAISS, sliding window)。
        * Dumpデータ2 (Cirrus): 同上。
        * 270kデータ1 (Cohere): TF-IDF。
        * 270kデータ2 (parsed): 埋め込み検索 (FAISS)。
    * **Reader:** 単一のDeBERTa-largeモデルを使用。推論時は訓練時より長いコンテキストを使用。
    * **アンサンブル:** 複数のRetrieverからの予測確率に対し、選択肢ごとに最大確率を取り、その順序で最終予測を決定 (Max Probability Ensemble)。
    * **後処理:** 特定条件下で、TF-IDF ((4,7)-gram) を用いて明らかに間違っている予測を修正。
