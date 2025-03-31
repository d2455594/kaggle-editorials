---
tags:
  - Kaggle
startdate: 2023-07-12
enddate: 2023-10-11
---
# Kaggle - LLM Science Exam
https://www.kaggle.com/competitions/kaggle-llm-science-exam

**概要 (Overview)**

- **目的:** このコンペティションの目的は、様々な科学分野（化学、物理、生物学、地学など）に関する**多肢選択式の質問**に対して、大規模言語モデル（LLM）を活用して**正解の選択肢を予測する**モデルを開発することです。
- **背景:** LLMの能力が急速に進歩する中で、その知識や推論能力を、標準化された教育評価のようなタスクで測定・評価することに関心が高まっています。科学分野の質問に正確に答える能力は、より汎用的な人工知能の実現に向けた進歩を示すとともに、教育、情報検索、専門家システムの分野への応用可能性を示唆します。
- **課題:** 広範な科学知識（知識の幅と深さ）が要求されること。質問は単なる知識の検索だけでなく、複雑な概念の理解や多段階の推論を必要とすることが多いこと。不正解の選択肢（Distractors）が巧妙に作られており、正解を特定するには注意深い分析が必要であること。特に、提供される外部知識コーパス（例：Wikipedia記事）を効果的に活用し、質問に関連する情報を検索して回答を生成する（Retrieval-Augmented Generation - RAG）アプローチが重要となる場合があります。

**データセットの形式 (Dataset Format)**

提供される主なデータは、科学分野の多肢選択問題とその正解です。外部知識ソースとして利用可能なテキストコーパスも提供される場合があります。

1. **トレーニングデータ (`train.csv` など):**
    
    - トレーニング用の質問が含まれるCSVファイル。
    - 各行が一つの質問に対応し、以下の列を含む可能性があります。
        - `id`: 質問の一意な識別子。
        - `prompt`: 質問文。文脈情報を含む場合もあります。
        - `A`, `B`, `C`, `D`, `E`: 各選択肢のテキスト。（選択肢の数は5つが一般的ですが、異なる場合もあります）。
        - `answer`: **ターゲット変数**。正解の選択肢を示す文字（例: `A`, `B`, `C`, `D`, `E`）。
2. **テストデータ (`test.csv` など):**
    
    - トレーニングデータと同様の形式で `id`, `prompt`, `A`, `B`, `C`, `D`, `E` を含みますが、`answer` 列は含まれません。
    - 参加者は、このテストデータに含まれる各質問に対して正解の選択肢を予測します。
3. **外部知識データ (任意):**
    
    - 大量のテキストデータ（例: Wikipediaの記事ダンプ）が提供される場合があります。参加者は、テスト時にこれらの外部データを参照し、関連情報を検索して回答の精度を高めるアプローチ（RAG）を取ることが推奨されることがあります。
4. **`sample_submission.csv`**:
    
    - 提出フォーマットのサンプル。通常、`id` と `prediction` の列を持ちます。`prediction` 列には、モデルが最も正解であると予測する**上位3つの選択肢**を、確信度の高い順にスペース区切りで記述します（例: `A B C`）。

**評価指標 (Evaluation Metric)**

- **指標:** **Mean Average Precision at 3 (MAP@3)**
- **計算方法:**
    1. **各質問の評価:** モデルは各質問に対して、最も正解の可能性が高いと判断した上位3つの選択肢を順位付けして提出します。
    2. その質問に対する **Average Precision at 3 (AP@3)** を計算します。
        - 1位の予測が正解の場合: AP@3 = 1/1 = 1.0
        - 1位が不正解で、2位の予測が正解の場合: AP@3 = 1/2 = 0.5
        - 1位、2位が不正解で、3位の予測が正解の場合: AP@3 = 1/3 ≈ 0.33
        - 上位3つの予測に正解が含まれない場合: AP@3 = 0
    3. **最終スコア:** テストデータセットの**全ての質問に対するAP@3の平均値**が MAP@3 となります。MAP@3 = (全質問のAP@3の合計) / (総質問数)。
- **意味:** この指標は、モデルが単に正解を当てるだけでなく、正解であると確信している度合い（＝正解を上位にランク付けする能力）も評価します。正解が上位3位以内に入っていれば部分点が与えられるため、絶対的な正解を一つだけ選ぶよりも、モデルの「自信度」を考慮した評価が可能です。MAP@3 が**高い**ほど（最大1.0）、モデルが正解の選択肢をより正確に、かつ高い確信度を持って予測できていると評価されます。

要約すると、このコンペティションは、LLMを用いて科学分野の多肢選択問題に答えるタスクです。データは質問、選択肢、正解ラベル（トレーニング用）、および外部知識コーパスで構成され、性能は正解選択肢を上位3位以内に、かつ高い順位で予測する能力を測る MAP@3（高いほど良い）によって評価されます。

---

**全体的な傾向**

上位解法では、外部知識ソース（主にWikipedia）から関連情報を検索し、それをコンテキストとして言語モデルに入力して回答を選択するRetrieval-Augmented Generation (RAG) アプローチが支配的でした。検索（Retrieval）フェーズでは、Embeddingモデル（e5, gte, bgeなど）を用いた類似度検索（Dense Retrieval）や、BM25、TF-IDFなどを用いたキーワード検索（Sparse Retrieval）が単独または組み合わせて使用されました。Wikipediaのデータソースとしては、公開されているものに加え、独自に処理・構築されたコーパス（特にcirrussearch dumpやSTEM分野に特化したもの）が利用されました。検索結果をさらに絞り込む・並び替えるためのRerankerモデル（DeBERTaベースが多い）も有効な手法でした。回答選択のフェーズ（Reader/Ranker）では、DeBERTa V3 LargeなどのTransformerモデルや、Llama 2、Mistralといった大規模言語モデル（LLM）がファインチューニングされて用いられました。LLMのファインチューニングには、多くの場合LoRAやQLoRAといった効率的な手法が採用されました。モデルの学習形式は、Multiple Choice形式、各選択肢の正誤を判定するBinary Classification形式、正解選択肢のトークンを予測するCausalLM形式など多様でした。複数のコンテキストを組み合わせたり、難易度に応じてモデルを使い分ける段階的推論、そして複数モデルやコンテキストの結果を統合するアンサンブルがスコア向上の鍵となりました。

**各解法の詳細**

**1位**

- **アプローチ:** RAG + LLM (Binary Classification)。多様なEmbeddingモデルとWikipediaダンプによるコンテキスト検索。LLMはBinary形式でファインチューニング。他選択肢の情報を追加する工夫あり。
- **アーキテクチャ:** Retrieval: e5-base-v2, e5-large-v2, gte-base, gte-large, bge-large。Reader: Llama-2-7b/13b, Mistral-7B-v0.1, xgen-7b-8k-base (LoRAでFine-tuning)。Binary Classification Head。
- **アルゴリズム:** 類似度検索 (FAISS等)、LoRA、BCE Loss、Cosine LR Decay。
- **テクニック:**
    - **Retrieval:** 複数Embeddingモデル使用。Wikipediaダンプの比較検討（自作、cirrussearch）。クエリ生成方法（全体 vs 選択肢ごと）。高速類似度検索（GPU並列）。
    - **コンテキスト:** 複数（訓練時3、推論時5）コンテキスト利用。コンテキスト順序の重要性。
    - **LLM学習:** Binary形式（各選択肢が正解か否か予測）。LoRA（全Linear層）。キャッシュ（past_key_values）利用で推論高速化。別々のLR（Head vs LoRA）。
    - **他選択肢情報:** 他の選択肢の平均Logitsを追加情報として分類Headに入力（性能向上）。
    - **ツール:** H2O LLM Studio (fork) を利用。
    - **アンサンブル:** 5つの7Bモデルと1つの13Bモデルのブレンド（異なるコンテキストアプローチ）。

**2位**

- **アプローチ:** RAG + Reranker + Multiple Choiceモデル + XGBRankerによるスタッキング。MLMによる類似選択肢対策。
- **アーキテクチャ:** Knowledge DB: Wikipedia (graelo/wikipedia)。Retrieval: Lucene BM25 (pyserini)。Reranker: DeBERTa-V3-base。Reader: DeBERTaV2ForMultipleChoice, MistralForMultipleChoice。MLM: DeBERTa (詳細不明)。Stacking: XGBRanker。
- **アルゴリズム:** BM25、DeBERTa Multiple Choice、Masked Language Modeling (MLM)、XGBRanker。損失スケーリング（選択肢数による）。
- **テクニック:**
    - **データ準備:** Wikipediaを文単位に分割しLuceneでインデックス化。
    - **Reranker:** (Question, Answer)ペアに対して最適なコンテキストを予測するように学習（Teacherモデルで疑似ランク付与）。
    - **Reader:** 標準的なMultiple Choice形式で学習。
    - **MLM:** 選択肢間の単語差分が少ない問題に特化して使用（`sed.standard_sed_backtrace`で差分検出）。
    - **Cross Reference:** "None of the above"のような選択肢に対応するため、疑似データで学習した第2段階Ranker（詳細不明）。
    - **スタッキング:** Retrievalスコア、DeBERTa/MistralのLogits、"None of the above"フラグなどを特徴量としてXGBRankerで最終ランキング。

**3位**

- **アプローチ:** RAG + Reranker + 複数モデルアンサンブル + LLM (Zero-shot)。Wikipedia処理とReranker学習が特徴。
- **アーキテクチャ:** Retrieval: bge-small, all-MiniLM-L6-v2。Wikipedia処理: wikiextractor (改変)。Reranker: ibm/re2g-reranker-nq (事前学習済み) をFine-tuning。Reader: DeBERTa, ELECTRA, RoBERTa (Multiple Choice形式), LLM (Platypus2 70B, Xwin)。
- **アルゴリズム:** 類似度検索 (FAISS等)、Multiple Choice Fine-tuning、Zero-shot LLM推論。
- **テクニック:**
    - **Wikipedia処理:** wikiextractorを改変し数値欠落問題に対処。記事単位ではなく文章(passage)レベルで検索。
    - **Reranker学習:** Wikipedia passageを候補とし、(Question, Candidate)ペアで学習。Hard Negative Mining（ただし事前学習済みモデル使用 or 2段階学習が重要）。
    - **Reader:** 複数のTransformerモデル(DeBERTa, ELECTRA, RoBERTa)をアンサンブル。
    - **LLM活用:** 回答が難しい問題（上位モデルの予測信頼度が低いもの）に対してZero-shotでLLM (Platypus2, Xwin) を適用。
    - **アンサンブル:** 複数ReaderモデルとLLMの結果を統合。

**4位**

- **アプローチ:** 高品質なRetrievalとDeBERTaモデルの組み合わせ。RetrievalにElasticsearchと意味検索を併用。
- **アーキテクチャ:** Retrieval: Elasticsearch (BM25), Sentence-Transformers (msmarco-bert-base-dot-v5, all-mpnet-base-v2)。Reader: DeBERTa V3 Large。
- **アルゴリズム:** Elasticsearch BM25、類似度検索 (Sentence-Transformers)。
- **テクニック:**
    - **Wikipedia処理:** cirrussearch dumpを使用し、文単位で分割。
    - **Retrieval (Stage 1):** Elasticsearchで文単位のキーワード検索（プロンプト/選択肢の全単語からStopword除去）。
    - **Retrieval (Stage 2):** Stage 1の結果を3種類の方法でリランキングしコンテキスト生成（v3: Elasticsearchスコア、v5: 編集距離、v7: Semantic Searchスコア）。周辺文・連続文の結合処理。
    - **Reader:** 取得したコンテキストと質問・選択肢を入力。maxlenを段階的に増加（512→768→1280）。
    - **コンテキストアンサンブル:** 3種類のコンテキスト(v3, v5, v7)それぞれで推論し、結果をアンサンブル。
    - **Elasticsearch on Kaggle:** /kaggle/tempへのコピーと、読み取り専用ファイルのシンボリックリンクによる高速化。

**5位**

- **アプローチ:** Sparse (BM25) + Dense (instructor-xl, bge-large) RetrievalとLLM (Llama 2 70B, Mistral 7B) の組み合わせ。自作Wikipediaデータセットと70Bモデルの推論工夫が特徴。
- **アーキテクチャ:** Retrieval (Sparse): pyserini (BM25/Lucene)。Retrieval (Dense): instructor-xl, bge-large + FAISS (量子化)。Reader: Llama-2 70B-hf, Mistral-7B-v0.1 (QLoRAでFine-tuning)。
- **アルゴリズム:** BM25、類似度検索 (FAISS)、QLoRA、CausalLM形式 (次トークン予測)。TTA（選択肢順序変更）。
- **テクニック:**
    - **Wikipediaデータセット:** wikitextparser + カスタムテンプレート処理で独自に構築（数値・数式情報を保持）。段落単位で分割。
    - **QAデータセット:** ChatGPT 3.5で独自生成（科学分野に特化、難易度高設定）。
    - **Retrieval:** Sparse (BM25) と Dense (instructor-xl/全wiki文単位、bge-large/STEM wiki段落単位) の3種を組み合わせ。
    - **LLM学習:** QLoRAでFine-tuning。CausalLM形式（正解選択肢のトークン " A", " B", .. の確率を比較）。
    - **70Bモデル推論:** 量子化(4bit) + レイヤー単位データセット登録 + Layer-by-Layer推論 + xformersによるAttention効率化でKaggle Notebook実行を実現 (GPUメモリ約6GB)。
    - **TTA:** 選択肢の順序を5パターン試し平均化。共通部分の計算を一度にまとめる工夫で高速化。
    - **段階的推論:** Mistral-7B (複数コンテキスト) → Llama2-70B (複数コンテキスト) → Llama2-70B (長文コンテキスト) の3段階で、下位%の難問のみ後段モデルで再推論。

**6位**

- **アプローチ:** Retriever-Readerフレームワーク。カスタムSTEMコーパス、合成データ利用、Retriever/RerankerのFine-tuning、多様なReaderモデル（DeBERTa, LLM）と段階的LLM推論。
- **アーキテクチャ:** Retriever: thenlper/gte-base, BAAI/bge-base-en-v1.5 (Fine-tuning)。Reranker: DeBERTa-V3-base (Cross-Encoder)。Reader (DeBERTa): Span Classification形式, MultipleChoice形式, PET形式。Reader (LLM): Open-Orca/Mistral-7B-OpenOrca (LoRA), platypus2-70b-instruct (Zero-shot), sheep-duck-llama-2 (Zero-shot)。
- **アルゴリズム:** NCE Loss (Retriever)、Cross-Encoder Reranking、LoRA、Zero-shot LLM推論。
- **テクニック:**
    - **データ:** カスタムSTEM Wikipediaコーパス（カテゴリメタデータでフィルタリング）。合成MCQデータ生成（GPT-3.5/4, LLaMA2 70b, Falcon 180b）。
    - **チャンキング:** 短いチャンク（検索用）と長いチャンク（コンテキスト用、オーバーラップあり）の2種作成。
    - **Retriever Fine-tuning:** 合成MCQデータを利用しNCE Loss + Hard Negativesで学習。
    - **Reranker:** 上位10件をDeBERTaでリランキングし、上位2-4件をコンテキストとして選択。
    - **Reader (DeBERTa):** Span Classification（選択肢トークンMean Pooling + Cross-option情報）、MultipleChoice、PET（Pattern Exploiting Training）の3形式で多様性を確保。
    - **Reader (LLM):** Mistral-7BをLoRAでFine-tuning (Cross-option形式)。予測信頼度が低い上位5%の問題のみ70Bモデル（platypus2, sheep-duck-llama-2）でZero-shot推論。プロンプトエンジニアリング（yes/noトークン差分など）。

**7位**

- **アプローチ:** 多様なRetrieval手法と、DeBERTa V3 LargeおよびLLM（CausalLM, Reward Modeling）のアンサンブル。段階的推論による効率化。
- **アーキテクチャ:** Retrieval: gte-small + FAISS (Article->Sentence), TF-IDF (Sentence, Paragraph), Sentence-Transformer (gte-small)。Reader: DeBERTa-V3 Large, Mistral-7B-v0.1, Mistral-7B-OpenOrca, OpenOrca-Platypus2-13B, Llama2-chat-AYT-13B。
- **アルゴリズム:** 類似度検索 (FAISS)、TF-IDF、CausalLM形式、Reward Modeling形式、LoRA/QLoRA、Softmax。
- **テクニック:**
    - **Retrieval:** 4種類の異なる手法（Wikipediaダンプ、270Kデータ、Cohereデータ、TF-IDF、Embedding）を組み合わせ。
    - **LLM学習:** CausalLM形式（正解選択肢トークン予測）、Reward Modeling形式（正解選択肢ペアを"chosen"、他を"rejected"として学習、"yes"トークンlogit比較）。QLoRA使用。選択肢シャッフル。
    - **段階的推論:** DeBERTaモデルで簡単な問題（max probability > 0.7）を除外し、残りをLLMで推論。さらにDeBERTaモデル群も段階的に適用。
    - **アンサンブル:** 8つのDeBERTaモデルと3つのLLMモデル（異なるコンテキスト）の予測を重み付き平均（LLMの重み高め）。

**10位**

- **アプローチ:** 複数のWikipediaデータソースからの多様なコンテキスト検索と、単一DeBERTaモデルによる予測。Max Probability EnsembleとTF-IDF後処理。
- **アーキテクチャ:** Retrieval: FAISS (Embedding), TF-IDF。Reader: DeBERTa Large。
- **アルゴリズム:** 類似度検索 (FAISS ivfpq, indexflat)、TF-IDF (4,7-gram)。Max Probability Ensemble。
- **テクニック:**
    - **Retrieval:** 4種類のWikipediaデータ（wikiextractor dump, cirrus dump, 270K Cohere, 270K parsed）に対し、FAISS (Embedding) とTF-IDFを組み合わせて検索。スライディングウィンドウによる文分割。
    - **コンテキスト:** 推論時には訓練時より長いコンテキストを使用。
    - **アンサンブル:** 各コンテキスト検索手法で得られた予測確率の最大値を取るMax Probability Ensemble。
    - **後処理:** 特定条件下でTF-IDF (4,7-gram) を用いて予測を修正。

