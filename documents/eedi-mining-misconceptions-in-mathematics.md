---
tags:
  - Kaggle
startdate: 2024-09-13
enddate: 2024-12-13
---
# Eedi - Mining Misconceptions in Mathematics
[https://www.kaggle.com/competitions/eedi-mining-misconceptions-in-mathematics](https://www.kaggle.com/competitions/eedi-mining-misconceptions-in-mathematics)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、生徒が数学の多肢選択問題 (MCQ) で誤った答えを選んだ際に、その誤答を引き起こした可能性のある**根本的な誤解 (Misconception)** を特定し、関連性の高い順に最大25個推奨するモデルを開発することです。
* **背景:** Eediは、AIを活用して生徒一人ひとりの学習ニーズに応じた教育を提供するプラットフォームです。生徒がどこでつまずいているのか、なぜ間違えたのか（誤解）を正確に把握することは、効果的な指導やフィードバックのために不可欠です。このタスクを自動化することで、教師の負担を軽減し、より個別化された学習支援を実現することを目指しています。
* **課題:**
    * **誤解の特定と推論:** 特定の誤答と、2500以上存在する誤解の中から最も関連性の高いものを結びつける、高度な推論能力が求められます。LLMは問題解決能力は高いものの、誤った思考プロセスを特定することは苦手とする場合があります。
    * **誤解間の類似性:** 誤解リストには、概念的・計算的に非常に似通ったものが多く含まれており、これらの微妙な違いを区別する精度が必要です。
    * **未知の誤解への汎化:** テストデータには訓練データに存在しない誤解が含まれる可能性が高く、モデルはこれらの未知の誤解に対しても頑健に機能する必要があります。
    * **数学的内容の複雑性:** 問題文や選択肢は数学的な記号 (LaTeX) や専門用語を含むため、これらを正確に解釈する必要があります。
    * **ランキング精度:** 単に誤解を特定するだけでなく、関連性の高い順に正確にランキングすることが評価指標 (MAP@25) で求められます。

**データセットの形式 (Dataset Format)**

提供されるデータは、問題、生徒の解答履歴、誤解リスト、およびそれらの関連情報です。

1.  **トレーニングデータ:**
    * `train.csv`: 生徒の解答記録。`QuestionId`, `UserId`, `AnswerValue` (生徒の解答), `IsCorrect`, `CorrectAnswer`, `AnswerId` を含む。
    * `questions.csv`: 問題の詳細。`QuestionId`, `QuestionText` (問題文), `AnswerValue` (選択肢番号), `AnswerText` (選択肢テキスト) を含む。
    * `misconceptions.csv`: 誤解のリスト。`MisconceptionId`, `MisconceptionName` (誤解の説明テキスト) を含む。
    * `question_misconceptions.csv`: 問題の誤答と誤解の間の**正解ラベル**。`QuestionId`, `AnswerValue` (誤答), `LinkedMisconceptionId` (関連する誤解IDのリスト) を含む。これがモデル学習の主要な教師データとなります。
    * `subject_metadata.csv`, `student_metadata.csv`: 科目や生徒に関する追加情報（任意で使用）。
2.  **テストデータ:**
    * `test.csv`: 予測対象となる誤答データ。`QuestionId`, `UserId`, `AnswerValue` を含む。
3.  **`sample_submission.csv`**:
    * 提出フォーマットのサンプル。
        * `QuestionId_AnswerValue`: 問題IDと誤答選択肢の組み合わせID。
        * `misconception_id`: **ターゲット変数**。予測された関連性の高い誤解IDを、スペース区切りで最大25個、関連度順に並べた文字列。

**評価指標 (Evaluation Metric)**

* **指標:** **MAP@25 (Mean Average Precision at 25)** 平均適合率の平均@25。
* **計算方法:**
    * 各テストケース（`QuestionId_AnswerValue` のペア）に対して、モデルは関連度順に最大25個の誤解IDを予測します。
    * 各テストケースについて、予測リストと正解リストを比較し、Average Precision @ 25 (AP@25) を計算します。これは、予測リストの上位から見ていき、正解の誤解IDが見つかるたびにその時点での適合率 (Precision) を算出し、それらの平均を取ることで得られます（正解リストの総数で正規化される場合あり）。
    * 最終的なスコアは、**全てのテストケースにわたるAP@25の平均値 (MAP@25)** となります。
* **意味:** モデルが、各誤答に対して関連性の高い誤解を、リストの上位にどれだけ正確にランク付けできているかを評価する指標です。単に正解を含むだけでなく、その順序が重要視されます。スコアは**高い**ほど良い性能を示します (最大1.0)。

要約すると、このコンペティションは、生徒の数学MCQにおける誤答がどの誤解に起因するかを特定し、関連度順にランク付けするランキングタスクです。データは問題、解答履歴、誤解リストから成り、性能はMAP@25（高いほど良い）によって評価されます。未知の誤解への対応とランキング精度が鍵となります。

---

**全体的な傾向**

このコンペティションでは、誤答とその原因となる誤解を関連付けるために、**Retrieve & Rerank** フレームワークが圧倒的な成功を収めました。これは、まず候補となる誤解を広く検索 (Retrieve) し、次にそれらをより精密に順位付け (Rerank) する2段階のアプローチです。両ステージで**大規模言語モデル (LLM)**、特に **Qwen シリーズ (14B, 32B, 72B)** や **Llama シリーズ** のファインチューニングが鍵となりました。

**Retriever** ステージでは、LLMを Embedding モデルとしてファインチューニングし、問題・誤答ペアと誤解テキスト間の類似度を計算して候補を抽出しました。損失関数として MultipleNegativesRankingLoss や ArcFace が用いられ、**ハードネガティブマイニング**（類似しているが正解ではない誤解を学習に使う）が重要でした。

**Reranker** ステージでは、Retrieverが抽出した候補（数十個）を、より強力なLLMを用いて精密に順位付けしました。入力には問題文、正答、誤答、候補となる誤解テキストに加え、LLM自身に生成させた**Chain-of-Thought (CoT) や Rationale (根拠説明)**、あるいは**Few-shot例**を含めることが有効でした。Rerankの手法としては、候補を一つずつ評価する **Pointwise** と、複数の候補を同時に評価する **Listwise** があり、特に後者が高い性能を示しました。Listwiseでは、LLMに直接ランキングを生成させるか、特定の選択肢トークン（例: 'A', 'B', ... や 'Yes', 'No'）の出力確率を用いてスコアリングする手法が取られました。

**未知の誤解**への対応が本コンペの核心であり、多くのチームが **LLM を用いた合成データ生成**に取り組みました。訓練データに存在しない誤解に対して、LLMに新しい問題文、選択肢、誤答例を生成させ、これを訓練データに追加することで、汎化性能を大幅に向上させました。生成データの品質を担保するために、さらに別のLLMを用いたフィルタリング（LLM-as-a-judge）も行われました。また、**誤解テキスト自体の拡張**（LLMによる説明文生成）もRetrieverの性能向上に寄与しました。

推論時間制約（9時間）の中で大規模なRerankerモデル（特に72Bクラス）を利用するため、**AWQ** や **AutoRound** による**4bit量子化 (Quantization)** と、**VLLM** ライブラリを用いた高速推論が不可欠でした。

**アンサンブル**も精度向上の常套手段で、Retriever/Rerankerともに、異なるバックボーン、学習データ、設定、Fold/Seedで学習した複数のモデル出力を組み合わせることで、最終的な予測の頑健性と精度を高めていました。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/eedi-mining-misconceptions-in-mathematics/discussion/551688)**

* **アプローチ:** Retrieve (Qwen2.5-14B) + Cascade Rerank (14B -> 32B -> 72B)。合成データとCoT活用。
* **アーキテクチャ:** Retriever: Qwen2.5-14B。Reranker: Qwen2.5-14B (Pointwise), Qwen2.5-32B (Pointwise), Qwen2.5-72B (Listwise)。CoT生成: Qwen2.5-Math-7B, 14B, 32B。
* **アルゴリズム:** Embedding (MNRL), Pointwise/Listwise Reranking (CE Loss)。
* **テクニック:** **合成データ生成** (クラスタリング+Few-shot+GPT-4oフィルタリング)。**CoT生成** (Claude 3.5 Sonnet) と入力への活用。RetrieverでRecall重視。RerankerでFew-shot例入力。疑似ラベルによる蒸留。高ネガティブ比率学習。Quantization (AutoAWQ)。VLLM推論。

**[2位](https://www.kaggle.com/competitions/eedi-mining-misconceptions-in-mathematics/discussion/551651)**

* **アプローチ:** Retrieve (Mistralベース, Qwen2.5-14B x2) + Sliding Window Rerank (14B -> 72B/Llama70B)。合成データと誤解拡張、CoT活用。
* **アーキテクチャ:** Retriever: Linq-Embed-Mistral, Qwen2.5-14B, Qwen2.5-32B, Qwen/QwQ-32B。Reranker: Qwen2.5-14B, 72B, Llama-3.1-70B。CoT生成: Qwen2.5-32B。
* **アルゴリズム:** Embedding (ArcFace, MNRL), Listwise Reranking。
* **テクニック:** 合成データ生成 (複数世代、GPT-4o-miniフィルタリング)。**誤解テキスト拡張** (LLMで説明文生成)。**CoT生成**と入力への活用。**Listwise Reranker** (Sliding Window式: 8-17位を14Bで、1-10位を72B/70Bで再ランク)。Quantization (AutoRound)。VLLM推論 (Prefix Caching)。TTA (候補順序反転)。

**[3位](https://www.kaggle.com/competitions/eedi-mining-misconceptions-in-mathematics/discussion/551498)**

* **アプローチ:** Retrieve (Qwen-14B x2) + Rerank (Qwen-32B)。**後処理による未知誤解ブースト**。
* **アーキテクチャ:** Retriever: Qwen2.5-14B (FlagEmbeddingベース?)。Reranker: Qwen2.5-32B-instruct-AWQ (LoRA)。
* **アルゴリズム:** Embedding, Listwise Reranking風 (Yes/Noトークンlogit)。
* **テクニック:** 合成データ生成 (GPT4-mini)。**後処理**: LBプロービングに基づき、未知誤解のスコアを定数倍 (約75%が未知と推定)。Reranker入力候補のランダムシャッフル。LoRAアンサンブル。

**[4位](https://www.kaggle.com/competitions/eedi-mining-misconceptions-in-mathematics/discussion/551559)**

* **アプローチ:** Retrieve (Qwen2.5-14B, 32B) + Rerank (Qwen2.5-32B x3)。合成データと誤解生成活用。
* **アーキテクチャ:** Retriever: Qwen2.5-14B-instruct, Qwen2.5-32B-instruct。Reranker: Qwen2.5-32B-instruct (LoRA)。
* **アルゴリズム:** Embedding (MNRL), Listwise Reranking風 (Yes/No?)。
* **テクニック:** **合成データ生成** (未知誤解ターゲット、Qwen2.5-72B使用)。**誤解生成** (LLMでQuestion/AnswerからMisconceptionを生成しRetriever入力に追加)。Retrieverは複数Fold出力を連結。Rerankerは複数ネガティブ数、合成データ有無で学習したモデルをLoRAマージ等でアンサンブル。VLLM推論。後処理 (既知誤解スコア低減)。

**[5位](https://www.kaggle.com/competitions/eedi-mining-misconceptions-in-mathematics/discussion/551391)**

* **アプローチ:** Retrieve (stella_1.5B) + Rerank (Qwen2.5-32B)。合成データとKD/Rationale活用。
* **アーキテクチャ:** Retriever: dunzhang/stella_en_1.5B_v5。Reranker: Qwen2.5-32B-Instruct。KD/Rationale生成: Qwen2.5-32B-Instruct。
* **アルゴリズム:** Embedding (Cached MNRL), Listwise Reranking (単一トークン選択肢確率)。
* **テクニック:** 合成データ生成 (未知誤解ターゲット、gemini-1.5-pro使用、類似MisconceptionをFew-shot例に)。**知識蒸留/Rationale生成**と入力への活用。RerankerはListwise (52候補同時入力、アルファベットトークン確率でソート)。Quantization (GPTQ)。VLLM推論。複数Foldアンサンブル。

**[6位](https://www.kaggle.com/competitions/eedi-mining-misconceptions-in-mathematics/discussion/551565)**

* **アプローチ:** 問題分解 (P(efs=1)分類 + E[time|1]回帰) + NNスタッキング。
* **アーキテクチャ:** Stage 1: GBDT (LGBM/XGB/CatBoost)。Stage 2: TabM + 1層NN。
* **アルゴリズム:** GBDT, MLP (TabM)。
* **テクニック:** Stage 2 NN入力: TabMリスクスコア + Stage 1 GBDT予測の2次多項式特徴量。NNは評価指標近似損失 (Sigmoid使用) で学習。複数シードCV (20 fold x 5 seeds)。

**[7位](https://www.kaggle.com/competitions/eedi-mining-misconceptions-in-mathematics/discussion/551388)**

* **アプローチ:** 3パイプライン (Retriever+Reranker) のVoting Ensemble。
* **アーキテクチャ:** Retriever: SFR-Embedding-2_R, public Qwen14B。Reranker: Qwen2.5-32B AWQ+LoRA (40オプション選択 or Binary選択)。Rationale生成: Qwen35B。
* **アルゴリズム:** Embedding, Listwise/Pointwise Reranking。
* **テクニック:** 合成データ生成。Rationale生成と活用。Retriever候補フィルタリング。Rerankerは異なる形式 (40オプション選択 vs Binary Yes/No) を使用。VLLM推論。最終段で3パイプラインのTop25リストをRankベースでVoting。後処理 (未知誤解ブースト)。

**[8位](https://www.kaggle.com/competitions/eedi-mining-misconceptions-in-mathematics/discussion/551412)**

* **アプローチ:** Retrieve (Qwen2.5-14B x4) + Rerank (Qwen2.5-32B)。合成データ活用。
* **アーキテクチャ:** Retriever: Qwen2.5-14B。Reranker: Qwen2.5-32B。
* **アルゴリズム:** Embedding (LoRA FT), Pointwise Reranking (Yes logit, LoRA FT)。
* **テクニック:** 合成データ生成 (ChatGPT)。Iterative Hard Negative Mining。Rerankerは全データで学習。Wise-FTによるLoRAマージ。Quantization (AWQ)。VLLM推論 (Prefix Caching)。

**[9位](https://www.kaggle.com/competitions/eedi-mining-misconceptions-in-mathematics/discussion/551420)**

* **アプローチ:** Retrieve (Qwen2.5-14B) + Rerank (14B, 32B)。合成データとRationale活用。
* **アーキテクチャ:** Retriever: Qwen2.5-14B-Instruct。Reranker: Qwen2.5-14B-Instruct, Qwen2.5-32B-Instruct AWQ。Rationale生成: Qwen2.5-32B-Instruct AWQ。
* **アルゴリズム:** Embedding (LoRA FT, Contrastive Learning), Pointwise/Causal LM Reranking。
* **テクニック:** 合成データ生成 (GPT-4o)。Rationale生成と入力への活用。Retrieverはハードネガティブマイニング。RerankerはBinary分類またはCausal LM (Yesトークンlogit)。AWQ量子化。VLLM推論。

**[10位](https://www.kaggle.com/competitions/eedi-mining-misconceptions-in-mathematics/discussion/551722)**

* **アプローチ:** Retrieve (Qwen2.5-14B x3) + Rerank (Qwen32B AWQ+LoRA)。合成データ活用。
* **アーキテクチャ:** Retriever: Qwen2.5-14B-Instruct, Qwen2.5-Coder-14B-Instruct。Reranker: Qwen2.5-32B AWQ (LoRA)。
* **アルゴリズム:** Embedding (LoRA FT), Listwise Reranking (多肢選択式、CE損失)。
* **テクニック:** 合成データ生成 (GPT-4O)。Retrieverは合成データで事前学習後、訓練データでファインチューニング。Rerankerは多肢選択式 (Top25オプションの確率予測)。LoRAマージによるアンサンブル。
