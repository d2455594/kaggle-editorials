---
tags:
  - Kaggle
  - 教育
startdate: 2024-09-13
enddate: 2024-12-13
---
# Eedi - Mining Misconceptions in Mathematics
https://www.kaggle.com/competitions/eedi-mining-misconceptions-in-mathematics

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
