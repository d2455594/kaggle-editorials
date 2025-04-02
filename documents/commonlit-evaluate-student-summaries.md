---
tags:
  - Kaggle
  - LLM
  - DeBERTa
startdate: 2023-07-13
enddate: 2023-10-12
---
# CommonLit - Evaluate Student Summaries
[https://www.kaggle.com/competitions/commonlit-evaluate-student-summaries](https://www.kaggle.com/competitions/commonlit-evaluate-student-summaries)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、与えられた文章（プロンプト）に対する生徒（3年生から12年生）の要約文を評価する機械学習モデルを開発することです。評価は、**内容 (content)** と**表現 (wording)** の2つの側面について行われ、それぞれのスコアを予測します。
* **背景:** CommonLit は、すべての生徒の読解力、文章力、批判的思考力を向上させることを目指す非営利の教育テクノロジー組織です。生徒の要約能力を評価することは教育的に重要ですが、教師にとっては時間のかかる作業です。このコンペティションは、AIを活用して要約文の評価を自動化し、教師の負担軽減と生徒への迅速なフィードバック提供を可能にすることを目指しています。
* **課題:**
    * **評価の主観性とノイズ:** 文章の品質評価には主観が伴い、評価者間での評価のばらつきがデータに含まれている可能性があります。モデルはこのようなノイズに対して頑健である必要があります。
    * **二つの評価軸:** 「内容」（要約が元の文章の主旨を捉えているか）と「表現」（文法、構成、明瞭さなど）という異なる側面を捉え、それぞれに対応するスコアを精度良く予測する必要があります。
    * **ドメインシフトと多様性:** 訓練データには4種類のプロンプトしかないのに対し、テストデータにははるかに多くの（122種類と推定される）未知のプロンプトが含まれています。モデルは未知のトピックや多様な学年の生徒の文章に対応できる高い汎化性能を持つ必要があります。
    * **自然言語処理特有の課題:** 文脈理解、同義語認識、文法構造の把握に加え、生徒が書きがちなスペルミスや文法的な誤りなどにも対応する必要があります。
    * **入力長の管理:** プロンプトの全文を入力に含めるとシーケンス長が非常に長くなるため、効率的な処理と長いコンテキストを扱えるモデルアーキテクチャが必要です。

**データセットの形式 (Dataset Format)**

提供されるデータは、生徒の要約文、要約対象の元の文章（プロンプト）、および専門家による評価スコアです。

1.  **トレーニングデータ:**
    * `summaries_train.csv`: 生徒の要約文と評価スコア。
        * `student_id`: 生徒の一意なID。
        * `prompt_id`: 対応するプロンプトのID。
        * `text`: 生徒が書いた要約文。
        * `content`: **ターゲット変数1**。内容に関する評価スコア（連続値）。
        * `wording`: **ターゲット変数2**。表現に関する評価スコア（連続値）。
    * `prompts_train.csv`: 要約対象の文章（プロンプト）に関する情報。
        * `prompt_id`: `summaries_train.csv` と対応するID。
        * `prompt_question`: 生徒に提示された、要約を作成するための具体的な問い。
        * `prompt_title`: プロンプト文章のタイトル。
        * `prompt_text`: プロンプト文章の全文。
2.  **テストデータ:**
    * `summaries_test.csv`: 評価対象となる生徒の要約文。`content` と `wording` スコアは含まれない。
    * `prompts_test.csv`: テストデータに対応するプロンプト情報。
3.  **`sample_submission.csv`**:
    * 提出フォーマットのサンプル。
        * `student_id`: 生徒ID。
        * `content`: 予測した内容スコア。
        * `wording`: 予測した表現スコア。

**評価指標 (Evaluation Metric)**

* **指標:** **MCRMSE (Mean Columnwise Root Mean Squared Error)** 平均列単位二乗平均平方根誤差。
* **計算方法:**
    * ターゲットとなる各列（`content` と `wording`）に対して、個別にRMSE (Root Mean Squared Error) を計算します。
    * RMSE は、予測値と真の値の差の二乗の平均値の平方根です (`sqrt(mean((predicted - actual)^2))`)。
    * 最後に、計算された各列のRMSEの平均値を取ります。
    * `MCRMSE = (RMSE_content + RMSE_wording) / 2`
* **意味:** 予測された内容スコアと表現スコアが、それぞれ真のスコアから平均的にどれだけ離れているかを示す指標です。RMSEに基づいているため、誤差が大きい予測に対してより大きなペナルティを与えます。スコアは**低い**ほど（0に近いほど）、モデルの予測精度が高いことを示します。

要約すると、このコンペティションは、生徒の要約文を内容と表現の2軸で評価する回帰タスクです。データは要約文、プロンプト、評価スコアで構成され、性能はMCRMSE（低いほど良い）によって評価されます。未知のプロンプトへの汎化性能が重要な課題です。

---

**全体的な傾向**

このコンペティションでは、生徒の要約文を評価するために、**大規模言語モデル (LLM) のファインチューニング**、特に **DeBERTa-v3-large** を使用するアプローチが上位解法を席巻しました。モデルの性能を最大限に引き出すために、いくつかの重要なテクニックが共有されました。

まず、**入力形式の工夫**が不可欠でした。生徒の要約文 (`text`) だけでなく、要約対象の文章である**プロンプトの情報 (`prompt_question`, `prompt_title`, `prompt_text`) を連結してモデルに入力**することが、スコア向上に大きく貢献しました。これに伴い、入力シーケンス長 (`max_length`) を **1024トークン以上に設定**し、長いコンテキストを扱えるようにすることが一般的でした。

次に、**プーリング層の選択**も重要でした。標準的なCLSトークンの利用に加え、Mean Pooling や Attention Pooling などが試されましたが、特に **Head Masking** と呼ばれる、**入力中の要約文部分に対応するトークンのみを対象として Mean Pooling を行う**手法が、2位チームによって非常に効果的であったと報告されています。

訓練データのプロンプトが4種類しかないという課題に対しては、**データ拡張**や**半教師あり学習**のアプローチが取られました。**LLM を用いて新たなプロンプトテキストや、それに対する様々な品質の要約文を生成**したり、**Meta Pseudo Labels** のような手法でラベルなしデータ（他の CommonLit テキストなど）を活用したりする試みが見られました（特に1位チーム）。また、過去の類似コンペ (Feedback Prize) のデータから**疑似ラベルを生成し、補助タスクとして利用**する手法も有効でした（2位チーム）。

学習テクニックとしては、**Layer-wise Learning Rate Decay (LLRD)**、**EMA (Exponential Moving Average)**、**モデルの一部レイヤーのフリーズ**、**Multi-sample Dropout** などが効果的とされました。一方で、AWP や SWA は効果が見られなかったという報告もありました。

最終的な提出は、異なる設定やシードで学習された複数の DeBERTa モデルの予測を**アンサンブル（平均化）** するのが一般的でした。LightGBM などの GBDT モデルによるスタッキングは、あまり有効ではなかったようです。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/commonlit-evaluate-student-summaries/discussion/447293)**

* **アプローチ:** DeBERTa-v3-large。LLMによるデータ拡張とMeta Pseudo Labelsによる半教師あり学習。
* **アーキテクチャ:** microsoft/deberta-v3-large (4-fold)。
* **アルゴリズム:** Transformerベースの回帰、半教師あり学習。
* **テクニック:** LLMで新規プロンプトと多様な品質の要約文を生成。**Meta Pseudo Labels** (3ラウンド) を使用し、ラベルなしデータ (CommonLitウェブサイト等から収集？) を活用。入力にプロンプト情報を含む。2段階学習 (疑似ラベルデータのみで学習 → 訓練データのみで学習)。入力長ソートによる推論高速化。

**[2位](https://www.kaggle.com/competitions/commonlit-evaluate-student-summaries/discussion/446573)**

* **アプローチ:** DeBERTa系アンサンブル。Head Masking Pooling、Prompt Augmentation、補助タスク学習、疑似ラベル。
* **アーキテクチャ:** microsoft/deberta-v3-large, base, OpenAssistant/reward-model-deberta-v3-large-v2, microsoft/deberta-large のアンサンブル。カスタムヘッド (Mean Pooling + LSTM Layer/Sequence Pooling, Mean Pooling + Linear)。
* **アルゴリズム:** Transformerベースの回帰、マルチタスク学習風。
* **テクニック:** **Head Masking Mean Pooling** (入力中の要約文部分のみPooling)。LLMによるPrompt Question Augmentation。Feedback Prize 3.0データを用いた**補助タスク学習** (疑似ラベル)。疑似ラベルデータを用いた高maxlen (最大2048) 学習。LLRD。Embedding/下位レイヤーフリーズ。Multi-sample Dropout。

**[3位](https://www.kaggle.com/competitions/commonlit-evaluate-student-summaries/discussion/446686)**

* **アプローチ:** DeBERTa-v3-largeアンサンブル (10 checkpoints)。データ前処理とAugmentation。
* **アーキテクチャ:** microsoft/deberta-v3-large。カスタムプーリング (CLS + Mean Pooling(text part))。
* **アルゴリズム:** Transformerベースの回帰。
* **テクニック:** 入力にプロンプト情報を含む。データ前処理・クリーニング (類似要約の発見・統合)。Augmentation (類似要約に基づく**逆オートコレクト風**)。EMA。LLRD。token_type_ids使用。

**[4位](https://www.kaggle.com/competitions/commonlit-evaluate-student-summaries/discussion/446524)**

* **アプローチ:** DeBERTa系アンサンブル (7モデル、Full-train)。多様な入力形式と補助損失。
* **アーキテクチャ:** microsoft/deberta-v3-large, deberta-v3-large-squad2。多様なプーリング (CLS, Attention Pooling, Mean Pooling(text part))。
* **アルゴリズム:** Transformerベースの回帰、マルチタスク学習風。
* **テクニック:** 1 full-train x 7 モデルアンサンブル。入力形式の多様化（プロンプトテキストの位置変更、カスタム指示文追加など）。**Arcface補助損失** (プロンプトID分類)。テキスト前処理 (句読点後のスペース確保)。推論時maxlen拡張。

**[5位](https://www.kaggle.com/competitions/commonlit-evaluate-student-summaries/discussion/446584)**

* **アプローチ:** DeBERTa-v3-large + LightGBMの重み付き平均。
* **アーキテクチャ:** microsoft/deberta-v3-large, LightGBM。
* **アルゴリズム:** Transformerベースの回帰, Gradient Boosting。
* **テクニック:** DeBERTaは全プロンプト情報入力。maxlen 1536。Freeze (Embedding + 18層)。Dropoutなし。LGBMは公開ノートブックベースの特徴量。アンサンブル重みはNelder-Meadで最適化。LLMによるデータ拡張は効果なし。

**[7位](https://www.kaggle.com/competitions/commonlit-evaluate-student-summaries/discussion/446534)**

* **アプローチ:** DeBERTa系アンサンブル。多様な入力形式とモデル。
* **アーキテクチャ:** microsoft/deberta-v3-large, deberta-large。
* **アルゴリズム:** Transformerベースの回帰。
* **テクニック:** 入力形式の多様化 (プロンプト情報の順序変更など)。Freezeレイヤー。LGBMブレンド。CV重視。

**[8位](https://www.kaggle.com/competitions/commonlit-evaluate-student-summaries/discussion/446712)**

* **アプローチ:** DeBERTa-v3-largeアンサンブル (6モデル、Full-train x 3実験 x 2シード)。特殊トークン使用。
* **アーキテクチャ:** microsoft/deberta-v3-large (ContextPooler使用)。
* **アルゴリズム:** Transformerベースの回帰。
* **テクニック:** 入力にプロンプト情報を含む。**特殊トークン** (`<question>`, `<summary>`, `<text>`) をTokenizerに追加して使用。maxlen 1024学習、1536推論。Gradient Accumulation。FC層Dropout Off。

**[9位](https://www.kaggle.com/competitions/commonlit-evaluate-student-summaries/discussion/446539)**

* **アプローチ:** カスタムDeBERTa+LSTMモデルアンサンブル。CV戦略の工夫。
* **アーキテクチャ:** カスタムモデル (DeBERTa-v3-large x2 + LSTM)。一方のDeBERTaは全体入力、もう一方は要約文のみ入力し、その出力をLSTMに入力、最終的に結合。
* **アルゴリズム:** Transformerベースの回帰、RNN。
* **テクニック:** 入力は `summary_text` と `prompt_question + [SEP] + prompt_text` のペア。maxlen 1680学習、4200推論。EMA。CV戦略 (特定の不安定なプロンプトID `814d6b` を評価から除外するかどうかでモデルを分ける)。複数シードアンサンブル。
