---
tags:
  - Kaggle
  - LLM
  - NLP
  - LogLoss
  - Llama
  - Gemma
startdate: 2024-05-03
enddate: 2024-08-13
---
# LMSYS - Chatbot Arena Human Preference Predictions
[https://www.kaggle.com/competitions/lmsys-chatbot-arena](https://www.kaggle.com/competitions/lmsys-chatbot-arena)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、あるプロンプトに対して2つの異なる大規模言語モデル（LLM）が生成した応答 (`response_a`, `response_b`) が与えられた場合に、**人間がどちらの応答をより好むか（あるいは引き分けか）を予測する**モデルを開発することです。これは、LLMの性能評価や改善に用いられるReward Model（報酬モデル）またはPreference Model（嗜好モデル）を構築するタスクに相当します。
* **背景:** 近年のLLM開発では、人間のフィードバックを用いてモデルをファインチューニングするRLHF（Reinforcement Learning from Human Feedback）やDPO（Direct Preference Optimization）といった手法が主流となっています。これらの手法では、人間の好みを正確にモデル化することが重要です。LMSYS Chatbot Arenaは、匿名化されたLLM同士を対戦させ、人間の投票によってランキングを作成するプラットフォームであり、本コンペティションはその蓄積された嗜好データを利用しています。人間の好みを予測する高精度なモデルは、より人間に有用で無害なLLMの開発に貢献します。
* **課題:** 人間の好みは主観的であり、応答の質（正確性、創造性、詳細さ、安全性など）、プロンプトの内容、モデルの個性、評価者のバイアスなど、多くの要因が絡み合って決定されます。提供されるプロンプトと応答のテキスト情報から、この複雑な嗜好判断を学習し、未知の応答ペアに対しても正確な予測を行うことが求められます。また、コンペ期間中にデータリークの問題が発生し、評価データセットが更新されるなど、外的要因への対応も必要となりました。

**データセットの形式 (Dataset Format)**

提供される主なデータは、プロンプト、2つのモデル応答、そして人間の評価（どちらを好んだか）です。

1.  **トレーニングデータ:**
    * `train.csv`: 人間の嗜好データ。
        * `id`: 各比較サンプルの一意なID。
        * `model_a`, `model_b`: 応答を生成したモデル名（匿名化されている場合あり）。
        * `prompt`: ユーザーがLLMに入力したプロンプト。複数ターンの対話履歴を含む場合があり、通常JSON形式で格納されている。
        * `response_a`, `response_b`: それぞれ `model_a`, `model_b` が生成した応答。プロンプト同様、複数ターンの対話履歴を含む場合があり、JSON形式。
        * `winner_model_a`, `winner_model_b`, `winner_tie`: **ターゲット変数**。人間による評価結果を示すフラグ。`winner_model_a=1` ならモデルAの勝ち、`winner_model_b=1` ならモデルBの勝ち、`winner_tie=1` なら引き分け。3つのうち必ず1つだけが1になる。
    * **追加データ:** コンペ中に主催者や参加者によって追加のデータセットが共有されました。
        * `33k dataset`: LMSYS提供の追加嗜好データ。
        * `ut dataset`: 参加者提供のデータセット。
        * `lmsys-1m dataset`: 大規模なチャットログデータ（嗜好ラベルなし）。擬似ラベリングなどに活用された。
        * その他、UltraFeedbackなどの公開Rewardデータセットも利用されました。
2.  **テストデータ:**
    * `test.csv`: 評価用データ。`id`, `model_a`, `model_b`, `prompt`, `response_a`, `response_b` カラムを含む。ターゲット変数列は含まれない。
3.  **`sample_submission.csv`**:
    * 提出フォーマットのサンプル。`id` と、`winner_model_a`, `winner_model_b`, `winner_tie` それぞれの予測確率（合計が1になる実数）の列を持つ。

**評価指標 (Evaluation Metric)**

* **指標:** **Log Loss (対数損失)**
* **計算方法:** 3クラス（Aが勝ち, Bが勝ち, 引き分け）分類問題として扱われ、各クラスに対する予測確率と実際の勝敗ラベル（例: A勝ちなら [1, 0, 0]）の間の対数損失が計算され、全テストサンプルで平均されます。
    $$ \text{LogLoss} = - \frac{1}{N} \sum_{i=1}^{N} \sum_{j=1}^{3} y_{ij} \log(p_{ij}) $$
    （N: サンプル数, y_ij: i番目のサンプルのクラスjの真ラベル(0 or 1), p_ij: i番目のサンプルのクラスjの予測確率）
* **意味:** 予測された確率分布が、実際の勝敗結果（真の分布）からどれだけ離れているかを示す指標です。予測確率が真の勝敗結果に近く、かつその予測に自信を持っている（確率が高い）ほど、Log Lossの値は小さくなります。値は0以上の実数を取り、**小さい**ほどモデルの性能が高いことを示します。

要約すると、このコンペティションは、LLMの2つの応答に対する人間の好みを予測する3クラス分類タスクです。データはプロンプト、応答ペア、勝敗ラベルで構成され、追加データの活用が鍵となり、性能はLog Loss（小さいほど良い）によって評価されます。

---

**全体的な傾向**

このコンペティションでは、人間の嗜好という複雑なターゲットを予測するために、最新の大規模言語モデル（LLM）を用いたファインチューニングが主流となりました。特に**Gemma-2-9B**が高い性能を示し、多くのチームがベースモデルとして採用しました。Llama-3-8BやQwen2-72Bなども有力な選択肢でした。

LLMのファインチューニングにあたり、通常のInstruction-Tunedモデル (`-it` モデル) だけでなく、**Reward Model (RM) や Preference Model として事前学習されたモデル** (`sfairXC/FsfairX-Gemma2-RM-v0.1`, `RLHFlow/pair-preference-model-LLaMA3-8B` など) を開始点とすることが、スコア向上に非常に効果的でした。

学習データ戦略としては、公式データに加えて**共有された追加データ (33k, ut data) や外部のRewardデータセット (UltraFeedbackなど) を活用**することが一般的でした。さらに、強力なベースラインモデルを用いて**大規模なラベルなしデータ (lmsys-1mなど) に擬似ラベルを付与**し、学習データを拡張する手法も上位解法で見られました。

モデルの学習においては、**全パラメータファインチューニング**と**LoRA/QLoRA**の両方が用いられました。入力は、プロンプトと2つの応答を指定の形式（例：特別な区切りトークンや指示テンプレートを使用）に整形して与えられました。複数ターンの対話や長いシーケンスへの対応（切り捨て方法など）も工夫されました。応答Aと応答Bを入れ替えたデータ (`Full Swap`) を同時に学習データに含める戦略も有効でした。

予測性能を高めるため、**複数モデルのアンサンブル**が広く採用されました。異なるベースモデル、異なる学習データ/設定、異なるFoldで学習したモデルの予測確率を平均化しました。推論時には、応答Aと応答Bを入れ替えて予測し平均を取る**TTA (Test Time Augmentation)** も効果的でした。

1位のチームは、大規模モデル（70B/72B）で学習したモデルの**予測分布 (logits) を教師として、より小さなモデル (Gemma-2-9B) を学習させる知識蒸留**を採用し、これが最終的なスコア向上に大きく貢献しました。

学習・推論の効率化のために、**Flash Attention, Triton Kernel, 8bit量子化, シーケンス連結 (padding削減)** などの技術が積極的に活用されました。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/lmsys-chatbot-arena/discussion/527629)**

* **アプローチ:** **知識蒸留** + 大規模モデル利用 + LoRA/QLoRA + アンサンブル。
* **アーキテクチャ/アルゴリズム:** ベースモデル: Llama3 70B, Qwen2 72B (教師モデル), Gemma-2-9B (生徒モデル)。AutoModelForSequenceClassification。LoRA/QLoRA (r=64, alpha=128, 全線形層ターゲット)。蒸留損失 + 分類損失。
* **テクニック:**
    * **事前学習:** UTデータセットでベースモデルを1エポック学習。
    * **教師モデル学習 & Logits抽出:** Kaggle+33kデータ (5-fold) で大規模モデル (70B/72B) を学習し、学習セットに対するlogits予測値を取得。
    * **知識蒸留:** Gemma-2-9BをKaggle+33kデータでファインチューニングする際に、教師モデルのlogitsとのKLダイバージェンスなどの蒸留損失を追加。
    * **アンサンブル:** 5-foldで学習したGemma-2-9Bモデルの**LoRAレイヤーを直接マージ**。
    * **推論:** 8bit量子化 (GPTQ)、TTA (入力長2000)。

**[2位](https://www.kaggle.com/competitions/lmsys-chatbot-arena/discussion/527685)**

* **アプローチ:** **Reward Model活用** + 全パラメータファインチューニング + **Full Swap** + 擬似ラベリング + アンサンブル。
* **アーキテクチャ/アルゴリズム:** ベースモデル: Gemma-2-9B-it, Gemma-2-27B-it, **RLHFlow/ArmoRM-Llama3-8B-v0.1**。カスタム分類ヘッド (Dropout + Linear + GELU + Linear)。
* **テクニック:**
    * **データ:** Kaggle + 33kデータ (21k重複削除後)。
    * **入力形式:** 指示テンプレート (2306.05685) を利用したPAB形式 (Prompt-ResponseA-ResponseB)。
    * **Full Swap:** 元サンプルと応答A/B入れ替えサンプルをペアで学習し、勾配をまとめて更新。
    * **擬似ラベリング:** Stage 1のアンサンブルモデルで外部データ (lmsys-1m, その他) 240kにソフトラベルを付与 (Stage 2)。
    * **学習:** Stage 1 (ベースライン) → Stage 2 (擬似ラベルで学習) → Stage 3 (Kaggle+21kデータで最終調整)。Gemma-2-9BはWindow Attention無効化。
    * **アンサンブル:** Stage 3のGemma-2-9BモデルとLlama3 RMモデルを2:1で加重平均。
    * **TTA:** Llama3モデルのみ推論時に応答A/Bをスワップ。
    * **効率化:** Flash Attention 2, Triton Kernel (RMSNorm, RoPE, GeLU/SiLU), シーケンス連結 (padding排除)。

**[3位](https://www.kaggle.com/competitions/lmsys-chatbot-arena/discussion/527766)**

* **アプローチ:** **Reward Model活用** + 大規模擬似ラベリング + QLoRA + アンサンブル。
* **アーキテクチャ/アルゴリズム:** ベースモデル: **sfairXC/FsfairX-Gemma2-RM-v0.1**, **RLHFlow/pair-preference-model-LLaMA3-8B**。QLoRA (r=64, alpha=16, 全線形層ターゲット)。AutoModelForSequenceClassification相当のカスタムヘッド。
* **テクニック:**
    * **データ:** Kaggle + 33kデータ。擬似ラベリング用に外部データ (orpo-dpo-mix-40k) + 自家生成データ (lmsys-1mプロンプトに対する応答ペア, 500k)。
    * **擬似ラベリング:** ベースラインモデル (Gemma2 RM, Llama3 RM) のアンサンブルで500k+データにソフトラベルを付与。
    * **学習:** 擬似ラベルデータとKaggle+33kデータを混合して最終モデルを学習。Gemma2モデルはSoftcapping無効化で性能向上。
    * **アンサンブル:** 最終的にGemma2-RMモデル (Mark作成) と別のGemma2モデル (Raja作成) をアンサンブル。
    * **TTA:** Gemma2モデルで応答A/BスワップTTA。
    * **効率化:** vLLMによる効率的なデータ生成（ターン数でバッチ化）。

**[4位](https://www.kaggle.com/competitions/lmsys-chatbot-arena/discussion/529067)**

* **アプローチ:** Gemma-2-9B-it ベース + LoRA + 擬似ラベリング + **独自プロンプト形式/切り捨て戦略** + TTA + 後処理。
* **アーキテクチャ/アルゴリズム:** ベースモデル: Gemma-2-9B-it。Gemma2ForSequenceClassification。LoRA (r=64, alpha=16, 全線形層含む複数モジュールターゲット)。
* **テクニック:**
    * **データ:** Kaggle + 33kデータ。擬似ラベリング用にUltraFeedback 30k。
    * **プロンプト形式/切り捨て:** 複数ターン対話を保持しつつ、最大長を超えた場合に最終ターンのプロンプト/応答A/応答Bが一定割合表示されるように切り捨て。情報量の少ない最終ターンは完全に破棄。
    * **学習:** Stage 1 (Kaggle+33k) → Stage 2 (Stage 1モデルでUltraFeedbackに擬似ラベル付けし、全データで再学習)。
    * **推論:** Max Lengthを3072に延長。TTA (応答A/Bスワップ)。
    * **後処理:** 応答AまたはBが空 ([null]など) の場合、非空応答が勝つように確率を固定 ([0.04, 0.88, 0.08] or [0.88, 0.04, 0.08])。応答AとBが同一の場合、引き分け確率を固定 ([0.06, 0.06, 0.88])。

**[5位](https://www.kaggle.com/competitions/lmsys-chatbot-arena/discussion/527669)**

* **アプローチ:** **Rewardデータでの事前学習** + Gemma-2-9Bファインチューニング + アンサンブル + TTA。
* **アーキテクチャ/アルゴリズム:** ベースモデル: Gemma-2-9B。H2O LLM Studio使用。
* **テクニック:**
    * **事前学習:** Gemma-2-9Bを公開Rewardデータ (主にUltraFeedback) で事前学習（二値分類タスクとして）。
    * **ファインチューニング:** 事前学習済みモデルを開始点として、Kaggle+33kデータでファインチューニング。
    * **アンサンブル:** 同じ設定で学習した2つのGemma-2-9Bモデル（片方は学習時に応答A/Bスワップ）をアンサンブル。
    * **TTA:** 推論時に片方のモデルで応答A/BスワップTTA。
    * **効率化:** GPU並列実行、INT8量子化 (bitsandbytes)、バッチ最適化。

**[9位](https://www.kaggle.com/competitions/lmsys-chatbot-arena/discussion/527704)**

* **アプローチ:** Gemma-2-9B-it と Llama3 Reward Model のLoRA + 擬似ラベリング + アンサンブル。
* **アーキテクチャ/アルゴリズム:** ベースモデル: Gemma-2-9B-it, **RLHFlow/pair-preference-model-LLaMA3-8B**。LoRA (Gemma: r=32, alpha=64; Llama3: r=128, alpha=128)。
* **テクニック:**
    * **データ:** Kaggle + 33kデータ。擬似ラベリング用にlmsys-chat-1m (45kサンプル抽出)。
    * **入力形式:** 最後の応答ペアのみを`<response_A>`, `<response_B>`とし、それ以前の対話履歴を`<context>`にまとめる形式。
    * **擬似ラベリング:** lmsys-chat-1mから抽出した45kデータにラベル付け（方法は不明記）。
    * **学習:** GemmaモデルはKaggle+33k+擬似ラベルで学習。Llama3モデルはKaggle+33kで学習。
    * **アンサンブル:** Gemma2モデル（応答スワップ版データで学習したものと通常版のアンサンブル）+ Llama3 RMモデル。
    * **推論効率化:** Llama3はトークン数が少ない/多いサンプルに限定して推論。8bit量子化、LoRAマージ、動的バッチサイズ。


