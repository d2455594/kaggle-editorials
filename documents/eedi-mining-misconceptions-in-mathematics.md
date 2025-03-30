---
tags:
  - Kaggle
  - 教育
startdate: 2024-09-13
enddate: 2024-12-13
---
# Eedi - Mining Misconceptions in Mathematics
https://www.kaggle.com/competitions/eedi-mining-misconceptions-in-mathematics

**概要 (Overview)**

* **目的:** このコンペティションの目的は、教育プラットフォーム「Eedi」で収集されたデータを用いて、生徒が数学の多肢選択問題を間違えた際に、その**誤りの原因となった根本的な「誤解（Misconception）」**を特定するモデルを開発することです。
* **背景:** 生徒が単に問題を間違えたという事実だけでなく、なぜ間違えたのか（どのような誤解を持っているのか）を理解することは、個別最適化されたフィードバックを提供し、効果的な学習を支援する上で非常に重要です。Eediは診断的な質問を通じて生徒の理解度を測っており、このコンペティションは、誤答の背後にあるパターンを分析し、教育改善に繋げることを目指しています。
* **課題:** 同じ問題に対する異なる誤答選択肢は、しばしば異なる特定の数学的な誤解に基づいています。生徒が選んだ誤答から、最も可能性の高い誤解を推定するには、問題の内容と生徒の典型的なエラーパターンを理解する必要があります。参加者は、生徒の回答履歴（どの問題を、どの選択肢で間違えたか）に基づいて、その誤答に対応する誤解の種類を予測することが求められます。これは**多クラス分類問題**として扱われます。

**データセットの形式 (Dataset Format)**

提供される主なデータは、生徒のEediプラットフォーム上での活動ログ（主に多肢選択問題への回答）で、表形式（CSVファイル）で提供されます。

1.  **トレーニングデータ (`train.csv` など):**
    * 生徒の回答記録を含みます。主要な列として以下のようなものが考えられます。
        * `user_id`: 生徒の識別子。
        * `question_id`: 問題の識別子。
        * `chosen_answer`: 生徒が選択した回答肢（例: 'A', 'B', 'C', 'D'）。
        * `is_correct`: その回答が正解だったかどうか（True/False）。
        * `misconception_id`: **予測対象のターゲット変数**。生徒が**不正解**だった場合に、その選択肢を選んだ原因と考えられる誤解のID。正解の場合は通常nullやNaNが入ります。

2.  **問題メタデータ (`questions.csv`, `question_metadata.csv` など):**
    * 各問題に関する詳細情報。
        * `question_id`: 問題の識別子。
        * 問題文、選択肢の内容、正解の選択肢。
        * **重要な情報:** 特定の**不正解の選択肢**が、どの**誤解ID (`misconception_id`)** に対応するかを示すマッピング情報。これがモデル構築の鍵となります。
    * 誤解の種類とその説明をリスト化したファイル (`misconceptions.csv` など) が別途提供される可能性もあります。

3.  **テストデータ (`test.csv` など):**
    * トレーニングデータと同様の形式ですが、生徒が不正解だった回答記録のみが含まれ、`misconception_id` の列は含まれません。
    * 参加者は、このテストデータに対して `misconception_id` を予測します。

4.  **`sample_submission.csv`**:
    * 提出フォーマットのサンプル。通常、テストデータの各行に対応する `row_id` や回答セッションIDと、予測された `misconception_id` の列を持ちます。

**評価指標 (Evaluation Metric)**

* **指標:** **Macro-averaged F1 Score (マクロ平均F1スコア)**
* **計算方法:**
    1.  まず、**各誤解クラスごと**に F1スコア を計算します。F1スコアは適合率（Precision）と再現率（Recall）の調和平均であり、予測の正確さと網羅性のバランスを示します。(F1 = 2 * (Precision * Recall) / (Precision + Recall))
    2.  次に、全誤解クラスについて計算されたF1スコアを単純に平均します（マクロ平均）。
* **意味:** マクロ平均F1スコアは、データセット内での出現頻度に関わらず、**各誤解の種類（クラス）を平等に評価**します。これにより、まれな誤解の検出精度も、頻繁に見られる誤解の検出精度と同じ重みで最終スコアに貢献します。多クラス分類タスクにおいて、クラス間のデータ量不均衡の影響を抑えつつ、全体的な性能を評価するのに適しています。

要約すると、Eediコンペティションは、生徒の数学問題への誤答データから、その原因となった誤解の種類を特定する多クラス分類タスクです。データは生徒の回答ログと問題メタデータから成り、性能は各誤解クラスを平等に評価するマクロ平均F1スコアによって測られます。

---

**全体的な傾向:**

このコンペでは、数学の多肢選択式問題の不正解に対して、生徒が抱いている可能性のある誤概念を特定し、ランキング付けすることが課題でした。上位解法では、Retrieve-and-Rerankのフレームワークが広く採用されています。大規模言語モデル（LLM）を活用したアプローチが主流であり、特にQwenシリーズのモデルが頻繁に登場します。合成データの生成、知識蒸留、Chain of Thought（CoT）、ハードネガティブサンプリング、リストワイズランキングなどが重要なテクニックとして用いられています。また、 unseen な誤概念への対応も重要なポイントとなっています。

**各解法の詳細:**

**1位**

- **アプローチ:** Retrieve-and-Rerankシステム（4段階）。不正解と誤概念の関連性をランキング付け。
- **アーキテクチャ:**
    - **Retriever:** Qwen/Qwen2.5-14B エンコーダ（高Recall）。
    - **Ranker:** Fine-tuned Qwen/Qwen2.5-14B, Qwen/Qwen2.5-32B, Qwen/Qwen2.5-72B。
- **アルゴリズム:** MultipleNegativesRankingLoss (Retriever), Cross Entropy Loss (Ranker)。
- **テクニック:**
    - **合成データ:** クラスター化された誤概念に基づいてClaude 3.5 Sonnetで生成。LLMによるフィルタリング。外部の誤概念も活用。
    - **Chain of Thought (CoT):** Claude 3.5 Sonnetで生成し、Qwen 2.5シリーズモデルで蒸留。
    - **Train-Validation Split:** ConstructIdに基づくGroupKFold。
    - **Retriever:** LoRAファインチューニング、Temperature調整、バッチ内の正例は1つのみ、キュレーション前の合成データで事前学習。
    - **Ranker:** ポイントワイズランキング（14B, 32B）、リストワイズランキング（72B）。
    - **Rankerの工夫:** Few Shot Examples、Distillation/Pseudo Labeling、Negative Ratioの調整、Chain of Thought (CoT) の利用。
    - **Quantization:** AutoAWQによる量子化。タスク固有のキャリブレーションデータセットを使用。

**2位**

- **アプローチ:** Retrieve-and-Rerank。過去のEEDIコンペのメタデータ、合成データ生成、誤概念の拡張、Chain of Thought（CoT）を活用。
- **アーキテクチャ:**
    - **Retriever:** Linq-AI-Research/Linq-Embed-Mistral (Arcface loss), Qwen/Qwen2.5-14B, Qwen/Qwen2.5-32B, Qwen/QwQ-32B-Preview (MultipleNegativesRankingLoss)。
    - **Reranker:** Qwen2.5-14B-Instruct, Qwen2.5-72B-Instruct, Llama-3.3-70B-Instruct (リストワイズ)。
- **アルゴリズム:** Arcface loss, MultipleNegativesRankingLoss。
- **テクニック:**
    - **前処理:** 過去のEEDIコンペのメタデータを活用し、親Subject情報を追加。
    - **合成データ:** LLM（主にQwen-math, gpt-4o-mini）による質問生成、誤概念制約下での不正解生成、品質フィルタリング。複数の生成フェーズ。
    - **誤概念拡張:** LLM（llama3.1-70b-Instruct, qwen2.5-72b-Instruct）による誤概念の説明生成。
    - **Chain of Thought (CoT):** Qwen2.5-32B-Instruct-AWQで生成。
    - **Retriever:** CoTをオプションで入力に利用。ラストトークンPooling。ハードネガティブサンプリング。
    - **Reranker:** スライディングウィンドウ方式のリストワイズランキング。テスト時の拡張（TTA）。
    - **学習:** QLoRAによるファインチューニング。リストワイズ選択のランダム化、ネガティブサンプルのマイニング。
    - **Quantization:** intel/auto-roundライブラリを使用。vLLM互換。
    - **Inference:** vLLMを使用。プレフィックスキャッシュの有効化。

**3位**

- **アプローチ:** 2段階アプローチ（Retrieve-and-Rerank）。未知の誤概念への対応を重視。
- **アーキテクチャ:**
    - **Retriever:** Qwen-14B Embedder (アンサンブル)。
    - **Reranker:** Qwen-32B-instruct-AWQ (LoRAアンサンブル)。
- **アルゴリズム:** MultipleNegativesRankingLoss (Retriever), Cross Entropy Loss (Reranker)。
- **テクニック:**
    - **Retriever:** [@anhvth226](https://www.kaggle.com/anhvth226) のQwen-14B EmbedderとFlagEmbeddingで学習したQwen-14B Embedderのアンサンブル。
    - **Reranker:** 様々な学習パラメータとクロスバリデーションで学習した6つの異なるLoRAのアンサンブル。GPT4-miniで生成したデータも一部使用。
    - **魔法:** テストセットに多数の未知の誤概念が存在することに着目。LBプロービングにより既知と未知の誤概念の比率を推定し、未知の誤概念の予測確率を定数倍するポストプロセス。
    - **その他:** ランダムな誤概念の順序シャッフル（Rerankerへの入力）。

**4位**

- **アプローチ:** 未知の誤概念への対応を重視したRetrieve-and-Rerank。データ生成、誤概念生成、Retriever、Reranker、ポストプロセス。
- **アーキテクチャ:**
    - **データ生成:** Qwen2.5-72B-Instruct-AWQ。
    - **誤概念生成:** Qwen2.5-32B-instruct-AWQ。
    - **Retriever:** Fine-tuned Qwen2.5-14B-instruct, Qwen2.5-32B-instruct (LoRA)。
    - **Reranker:** Fine-tuned Qwen2.5-32B-instruct (LoRAアンサンブル)。
- **アルゴリズム:** MultipleNegativesRankingLoss (Retriever), Cross Entropy Loss (Reranker)。
- **テクニック:**
    - **検証戦略:** QuestionIdに基づくGroup KFold（5分割）。未知の誤概念に対するスコアも評価。
    - **データ生成:** 未知の誤概念に対応する質問、回答選択肢、不正解を生成。プロンプトに多数の例を含める。
    - **誤概念生成:** 質問、回答などから誤概念を生成。
    - **Retriever:** 質問テキスト、回答テキストに加えて、生成された誤概念もテキストに追加して学習。Qwen2.5-14B-instructとQwen2.5-32B-instructの出力を結合して検索。既知の誤概念と未知の誤概念を区別して検索。
    - **Reranker:** 複数のQwen2.5-32B-instructモデルをLoRAでファインチューニング。ネガティブサンプルの数を調整し、生成されたデータを学習データに追加。LoRAアンサンブル。
    - **ポストプロセス:** 学習データに存在した誤概念の予測スコアを減衰。
    - **高速化:** データ生成と推論にvLLMを使用。

**5位**

- **アプローチ:** 知識蒸留を用いたRetrieve-and-Rerank。未知の誤概念に対応するための合成データ生成。リストワイズランキングにLLMを活用。
- **アーキテクチャ:**
    - **知識蒸留:** Qwen 2.5 32B Instruct。
    - **Candidate generation (Biencoder):** dunzhang/stella_en_1.5B_v5 (SentenceTransformer)。
    - **Listwise reranking:** Fine-tuned Qwen 2.5 32B Instruct。
- **アルゴリズム:** CachedMultipleNegativesRankingLoss (Biencoder), Cross Entropy Loss (Reranker)。
- **テクニック:**
    - **クロスバリデーション:** SubjectIdに基づくGroupKFold（5分割）。
    - **合成データ生成:** 未学習のMisconceptionIdごとに、類似のMisconceptionNameを持つ既存のデータセット情報をfew-shotプロンプティング（gemini-1.5-pro）で生成。
    - **知識蒸留 (KD):** QuestionText、正解、不正解をLLMに入力して不正解の理由を生成。後続のモデルの入力として使用。
    - **Candidate generation:** Biencoderで各問題と不正解ペアに対するMisconceptionName候補を抽出。ハードネガティブは効果なし。
    - **Listwise reranking:** Biencoderの出力でソートされたMisconception候補を52個ずつLLM（Qwen 2.5 32B Instruct）に入力し、各オプションの確率を取得してランキング。トップ104候補を使用し、2回に分けて推論。
    - **アンサンブル:** 3つのフォールドでリストワイズリランカーの結果をアンサンブル。

**6位**

- **アプローチ:** Retrieve-and-Rerank。未知の誤概念への汎化性を重視。合成データ生成、Latexテキストの処理、知識蒸留、2段階Reranker。
- **アーキテクチャ:**
    - **Retriever:** gte-Qwen2-7B-instruct, Salesforce-SFR-Embedding-2_R, bge-multilingual-gemma2, Qwen2.5-14B-Instruct (LoRA)。
    - **Reranker:** Qwen 32B instruct (LoRA, AWQ)。
- **アルゴリズム:** MultipleNegativesRankingLoss (Retriever), Cross Entropy Loss, KL Divergence (Reranker)。
- **テクニック:**
    - **検証戦略:** 見られた誤概念と見られない誤概念に等しい重みを置くことを想定。
    - **合成データ生成:** Qwen 72B instructによるfew-shot生成。類似の学習データの誤概念からランダムに3つのfew-shot例を使用。
    - **Retriever:** pylatexencライブラリでLatexをプレーンテキストに変換。ハードネガティブマイニング。BNB 4bit量子化。
    - **Reranker:** `DataCollatorForCompletionOnlyLM`を使用。テスト誤概念を学習から除外。
    - **Distillation:** 異なるシードで学習したモデルのロジットを用いてKLダイバージェンス蒸留。
    - **First stage reranker:** 上位100個の類似誤概念からランダムに49個のネガティブサンプルを選択し、正解の誤概念をランダムな位置に配置して分類。
    - **Misconception rephrasing:** Qwen 32B-Instructで誤概念を一般的な形式に言い換え、元の誤概念のロジットと組み合わせる。
    - **Second stage reranker:** 最初のRerankerの上位ランクの誤概念の小さなウィンドウで、すべての可能な順列を適用し、ロジットを平均化して再度ランク付け。最適なウィンドウサイズは2。
    - **Quantization:** AWQ 4bit量子化。

**7位**

- **アプローチ:** クラスごとのガウスヒートマップ予測を行う3Dセグメンテーションモデル。シミュレーションデータでの事前学習と実験データでのファインチューニング。
- **アーキテクチャ:** U-Net (バックボーン: ResNet50d, EfficientNetV2-M), DeepLab (バックボーン: ResNet50d)。
- **アルゴリズム:** U-Net、ResNet、EfficientNet、DeepLab。
- **テクニック:**
    - 事前学習とファインチューニング。
    - パーセンタイルクリッピング、データセット固有のスケーリング。
    - スライディングウィンドウによるパッチ分割とランダムシフト。
    - シフト、CutMix、MixUp、RandomFlip、Affine、Rot90、コントラスト、ガンマ、ガウスノイズによる強力なデータ拡張。
    - 重み付きBCE損失。
    - EMA（指数移動平均）。
    - モデルスープのアンサンブル。
    - 4x TTA、4x スライディングウィンドウ推論。
    - ロジットの平均化、エッジアーティファクト軽減、ガウシアンブラー、WBF、ロジット空間での閾値処理、LBプロービング。
