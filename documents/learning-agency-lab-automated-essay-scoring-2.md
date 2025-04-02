---
tags:
  - Kaggle
  - 教育
  - LLM
  - DeBERTa
  - QWK
  - LightGBM
startdate: 2024-04-04
enddate: 2024-07-03
---
# Learning Agency Lab - Automated Essay Scoring 2.0
[https://www.kaggle.com/competitions/learning-agency-lab-automated-essay-scoring-2](https://www.kaggle.com/competitions/learning-agency-lab-automated-essay-scoring-2)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、学生が書いたエッセイの質を自動的に評価（採点）する機械学習モデルを開発することです。特に、様々なプロンプト（課題）に対して書かれたエッセイに対し、人間の採点者によるホリスティックな評価（全体的な質に基づくスコア）を再現することを目指します。
* **背景:** Automated Essay Scoring (AES) は、教育評価の効率化と客観性向上のために長年研究されてきた分野です。従来のAESシステムは、文法、語彙、エッセイの長さなどの表層的な特徴量に依存することが多かったですが、近年の大規模言語モデル (LLM) の発展により、内容の理解や論理構成といった、より深層的な評価が可能になることが期待されています。
* **課題:** 与えられたエッセイテキストに対し、1から6までの6段階評価のスコアを予測すること。人間の採点には主観性やばらつきが含まれるため、モデルはそのようなノイズを含むデータから学習する必要があります。また、異なるデータソース（過去のデータセットと新しいデータセット）が混在しており、それらの間の採点基準の差異（ドメインシフト）に対応することも重要な課題となりました。これは本質的には**順序回帰 (Ordinal Regression)** または単なる**回帰 (Regression)** タスクとして扱われます。

**データセットの形式 (Dataset Format)**

提供される主なデータは、エッセイテキストとそのスコアです。

1.  **トレーニングデータ:**
    * `train.csv`:
        * `essay_id`: 各エッセイの一意なID。
        * `full_text`: エッセイの全文。
        * `score`: 人間の採点者によって付けられたホリスティックなスコア（1, 2, 3, 4, 5, 6のいずれか）。
    * データソースに関する情報が重要であることが判明しました。データセットは主に、過去の大規模エッセイデータセットである Persuade Corpus 2.0 由来のデータ ("old" data) と、このコンペティションのために用意された比較的新しいデータ ("new" data / "Kaggle-only" data) の混合で構成されていました。これらは `prompt_name` などの情報から区別できる可能性がありました。
2.  **テストデータ:**
    * `test.csv`:
        * `essay_id`: 各エッセイの一意なID。
        * `full_text`: エッセイの全文。
    * `score` は提供されません。テストデータは主に "new" データソースからサンプリングされていると推測されました。
3.  **`sample_submission.csv`**:
    * 提出フォーマットのサンプル。`essay_id` と予測された `score` の列を持ちます。スコアは整数である必要があります。

**評価指標 (Evaluation Metric)**

* **指標:** **Quadratic Weighted Kappa (QWK)** (二次加重カッパ係数)
* **計算方法:** QWKは、2人の評価者（ここでは、モデルの予測スコアと真のスコア）間の一致度を、偶然期待される一致度と比較して評価する指標です。カテゴリカルな評価、特に順序性のある評価に適しています。
    1.  まず、混同行列 (Confusion Matrix) を作成します。行が真のスコア、列が予測スコアに対応します。
    2.  観測された一致度 (observed agreement) と、偶然による期待一致度 (expected agreement) を計算します。
    3.  重み行列 (Weight Matrix) を計算します。二次加重 (Quadratic Weighting) では、予測スコアと真のスコアの差の二乗に比例してペナルティ（不一致の重み）が大きくなります。つまり、|i - j|^2 / (N-1)^2 のような形で重みが計算されます（i, j はスコアカテゴリ、Nはカテゴリ数）。
    4.  これらの値を用いてカッパ係数を計算します: Kappa = (Observed Agreement - Expected Agreement) / (1 - Expected Agreement)。計算時には重み行列が考慮されます。
* **意味:** モデルの予測スコアが、真のスコアとどれだけ一致しているかを、偶然の一致を除外した上で評価します。特に二次加重により、予測が真のスコアから大きく外れるほど評価が低くなります。スコアは通常 -1 から 1 の範囲を取り、1に近いほど一致度が高く、0は偶然レベル、負は偶然以下を示します。スコアは**高い**ほど良いモデルと評価されます。

要約すると、このコンペティションは、学生のエッセイテキストから1〜6のホリスティックスコアを予測する回帰（または順序回帰）タスクです。データはエッセイテキストとスコアで構成され、異なるデータソースの混合が特徴です。性能は、予測と真のスコアの一致度を測るQuadratic Weighted Kappa (QWK、高いほど良い) によって評価されます。

---

**全体的な傾向**

このエッセイ自動採点コンペティションでは、データセットの特性（旧データと新データの混在と採点基準の違い）を理解し、それに対応することが極めて重要でした。上位のソリューションは、主に以下の戦略を採用しました。

* **モデルアーキテクチャ:** **DeBERTa-v3** (特に large, base) が圧倒的に支配的でした。他のTransformerモデルも試されましたが、DeBERTa系が最も良い結果を示しました。
* **学習戦略:**
    * **2段階学習:** 旧データ (Persuade Corpus由来) + 新データ (Kaggle Only) で事前学習し、その後、新データのみでファインチューニングするアプローチが非常に効果的でした。これは、テストデータが新データに近いという仮説に基づいています。
    * **データソース情報の活用:** 入力テキストにデータソースを示す特別なタグを追加したり、データソースごとに異なる分類ヘッドを使用したりする試みも見られました。
    * **タスク形式:** ほとんどのチームが**回帰 (Regression)** 問題としてスコアを予測しました。損失関数としてはMSEやBCE（ターゲットを[0,1]にスケーリング）、Huber Lossが用いられました。**順序回帰 (Ordinal Regression)** も有効なアプローチとして採用されました（特に2位）。
* **前処理:**
    * **特殊トークン:** DeBERTaのトークナイザが無視する改行文字 `\n` を `[BR]` のような特殊トークンに置き換える処理が有効でした。
    * **テキストクリーニング:** 特定のコードを用いたテキストクリーニングも試されました。
* **疑似ラベリング (Pseudo Labeling):** 新データでファインチューニングしたモデルを用い、旧データに対して新しいスコア（疑似ラベル）を予測し、そのデータを使って再度モデルを学習させる手法がスコア向上に貢献しました。複数ラウンド行うチームもありました。
* **アンサンブル:** 複数のDeBERTaモデル（異なるサイズ、Fold、Seed、ハイパーパラメータ、Pooling手法、学習データ）の予測値をアンサンブルすることが必須でした。手法は単純平均や重み付き平均 (Nelder-Mead最適化など) が主流でした。一部、LightGBMによるスタッキングも試されましたが、DeBERTaアンサンブルが強力でした。
* **閾値最適化 (Thresholding):** 回帰モデルの出力（浮動小数点数）を最終的な整数スコア（1〜6）に変換する際、**QWKスコアを最大化するように閾値を最適化**する後処理が極めて重要でした。`OptimizedRounder` や `scipy.optimize.minimize` を用いて、OOF (Out-of-Fold) 予測に対して最適な閾値（例: 1.5, 2.5, ..., 5.5 の代わりに 1.67, 2.45, ... のような値）を探索しました。
* **CV戦略:** データソースの違いを考慮したCV戦略が重要でした。Prompt IDとスコアに基づいたStratified K-Foldや、新データのみでの評価などが試されました。しかし、LBとの相関に苦労するチームも多く、最終的な提出モデル選択はCVとLBの両方を考慮する必要がありました。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/learning-agency-lab-automated-essay-scoring-2/discussion/516791)**

* **アプローチ:** DeBERTaアンサンブル。2段階学習（旧+新→新）。2段階疑似ラベリング。閾値最適化。
* **アーキテクチャ:** DeBERTa-v3-large/base。Pooling層 (CLS, GeM)。回帰ヘッド。
* **アルゴリズム:** MSE Loss, BCE Loss。閾値最適化 (scipy.optimize.minimize + Powell法)。アンサンブル (単純平均, Nelder-Mead, Hill Climbing)。
* **テクニック:**
    * **データ戦略:** 旧データと新データのスコア分布の違いを分析。旧+新データで事前学習し、新データでファインチューニング。
    * **疑似ラベリング:** 2ラウンド実施。Round 1: `mean(真スコア, 予測スコア)`。Round 2: 予測スコアのみ。ただし、Round 2はCVは向上するがLBが低下したため、最終選考ではRound 1のみ使用したモデルが主。
    * **学習:** 3-seed平均。差分学習率。Cosine Decay。Dropoutなし。特殊トークン追加。
    * **閾値最適化:** OOF予測に対してQWKを最大化するように閾値を最適化。開始点依存性を考慮し複数開始点から平均。
    * **アンサンブル:** 過学習を避けるため、閾値はモデルごとに計算し、アンサンブル重みで閾値もブレンド。単純平均も併用。

**[2位](https://www.kaggle.com/competitions/learning-agency-lab-automated-essay-scoring-2/discussion/516790)**

* **アプローチ:** DeBERTa-v3-large中心。2段階学習。MLM事前学習。順序回帰。閾値最適化。Votingアンサンブル。
* **アーキテクチャ:** DeBERTa-v3-large。Pooling層 (Attention)。カスタムモデル (CoPE, PET風, 階層的Pooling)。
* **アルゴリズム:** BCE Loss (回帰/順序回帰)。閾値最適化 (OptimizedRounder + Last Threshold Search)。Votingアンサンブル。
* **テクニック:**
    * **学習:** 2段階学習（Persuade→Kaggle-Only）。全データで10エポックMLM事前学習。
    * **順序回帰:** スコアを N-1 個のバイナリ変数に変換 (`score=3` -> `[1, 1, 0, 0, 0]`) し、BCE Lossで学習。
    * **閾値最適化:** OptimizedRounderに加え、最後の閾値 (5と6の間) をループで探索。全学習データで探索。
    * **その他モデル:** CoPE (Contextual Position Embedding) の導入、PET風プロンプト (`... essay is: [MASK].`)、階層的Pooling (文→段落→全体) も試行。
    * **アンサンブル:** 5つのモデル (通常回帰、順序回帰、PET風、CoPEなど) の最終予測 (整数) によるVoting。

**[3位](https://www.kaggle.com/competitions/learning-agency-lab-automated-essay-scoring-2/discussion/517014)**

* **アプローチ:** DeBERTaアンサンブル。データB (Kaggle Only) 重視の2段階学習。ソフトラベル利用。Nelder-Meadアンサンブル。
* **アーキテクチャ:** DeBERTa-v3-base/large。Pooling層 (MeanPooling, AttentionPooling, LSTMPooling)。回帰ヘッド。
* **アルゴリズム:** BCE Loss。Nelder-Meadアンサンブル重み最適化。
* **テクニック:**
    * **データ戦略:** データA (Persuade) とデータB (Kaggle Only) を分離。データBでの評価を重視。
    * **学習:**
        * Stage 1: データA+Bで学習、データBでの評価に基づきエポック選択。
        * Stage 2: Stage 1の重みをロードし、データBのみで学習、データAでの評価に基づきエポック選択（過学習防止）。
    * **ソフトラベル:** Stage 2学習時に、Stage 1のOOF予測値をソフトラベルとして利用。
    * **モデル設定:** MLM事前学習。レイヤーフリーズ (base: 9層, large: 6層)。特殊トークン (`[BR]`)。
    * **アンサンブル:** 10個のDeBERTaモデル (base/large, 1024/1536 length, 3種Pooling) のOOF予測に対してNelder-Meadで重みを最適化。

**[4位](https://www.kaggle.com/competitions/learning-agency-lab-automated-essay-scoring-2/discussion/516639)**

* **アプローチ:** データソース間の非互換性に着目。入力タグ付け、マルチヘッド、データソース分類補助タスク。大規模アンサンブル。
* **アーキテクチャ:** DeBERTa-large/v3-large, Qwen2-1.5B-Instruct。回帰ヘッド、データソース分類ヘッド。
* **アルゴリズム:** MSE Loss。重み付き平均アンサンブル。閾値最適化。
* **テクニック:**
    * **データソース対応:**
        * 入力テキストにタグ (`[A]` or `[B]`) を追加。
        * データソースを予測する補助分類ヘッドを追加。
        * データソースごとに異なる回帰ヘッドを使用するモデルも試行。
    * **学習:** 非Persuadeデータでのスコアに基づきEarly Stopping。
    * **推論:** 全てのテストサンプルを非Persuade (`[A]`) と仮定して推論。
    * **疑似ラベル/蒸留:** タグを入れ替えて疑似ラベルや蒸留を試行（若干改善）。
    * **効率化:** 動的マイクロバッチ（トークン数制限によるバッチ分割）、高速化DeBERTa実装。
    * **アンサンブル:** 3モデル x 5 Fold x 3 Seed = 45予測の平均。

**[5位](https://www.kaggle.com/competitions/learning-agency-lab-automated-essay-scoring-2/discussion/516922)**

* **アプローチ:** DeBERTa中心の2段階学習。Kaggle-Onlyデータを重視。LightGBMとのブレンド。
* **アーキテクチャ:** DeBERTa-v3-small/base/large。LightGBM (公開ノートブックベース)。
* **アルゴリズム:** (DeBERTaは不明)。重み付きブレンド。
* **テクニック:**
    * **データ戦略:** Kaggle-Onlyデータを StratefiedKFold で train/valid に分割。
    * **学習:**
        * Stage 1: Persuadeデータ + Kaggle-Only-train で学習。
        * Stage 2: Stage 1の重みをロードし、Kaggle-Only-train で学習し、Kaggle-Only-valid で評価。
    * **アンサンブル:** DeBERTaモデル群の予測と、公開ノートブック由来のLightGBM予測を重み付きでブレンド (例: DeBERTa 0.9 : LGBM 0.1)。

**[6位](https://www.kaggle.com/competitions/learning-agency-lab-automated-essay-scoring-2/discussion/516814)**

* **アプローチ:** 公開ノートブックベースのLightGBMスタッキング。データソース特徴量、プロンプト重み付けCV、特徴量エンジニアリング、ラベリング修正。
* **アーキテクチャ:** DeBERTa-v3-large (特徴抽出器)、LightGBM (2段目)。TF-IDF, CountVectorizer。SpaCy。
* **アルゴリズム:** (DeBERTaは不明)。LightGBM (QWK目的関数)。閾値最適化。
* **テクニック:**
    * **CV:** Kaggle-Only 5プロンプトのみで評価。テストセットのプロンプト分布を推定し、重み付けQWKで評価。
    * **特徴量:**
        * データソースフラグ (`persuade`) を追加。
        * Texstat特徴量、SpaCyベース特徴量 (文間類似度、単語/文/段落統計)。
        * TF-IDF/CountVectorizerはテストセットのみで学習 (Leak?)。
    * **DeBERTa:** max_length=1536。OOF予測をCDF (累積分布関数) に変換してLGBM入力に。
    * **ラベリング修正:** DeBERTa予測と真ラベルの差が大きい場合に、LGBM学習時のラベルを修正。
    * **アンサンブル:** (LGBMモデルのみ?)

**[7位](https://www.kaggle.com/competitions/learning-agency-lab-automated-essay-scoring-2/discussion/516872)**

* **アプローチ:** 公開ノートブックベースのLightGBMスタッキング。Kaggle-Onlyデータの重複による学習データ強調。ハイパーパラメータチューニング。
* **アーキテクチャ:** DeBERTa-v3-large (特徴抽出器)、LightGBM (2段目)。TF-IDF, CountVectorizer。
* **アルゴリズム:** LightGBM (QWK目的関数)。閾値最適化 (調整あり)。
* **テクニック:**
    * **データ強調:** Kaggle-Onlyデータ (4436件) を複数回 (例: 3回や5回) 重複させて学習データに加える。ただし、CV時のリークを防ぐため、Validデータに含まれるものはTrainデータから削除。
    * **LGBMチューニング:** 特徴量選択 (上位501個)、ハイパーパラメータ調整 (n_splits, lr, depth, leaves, colsample, reg_alpha/lambdaなど)。
    * **QWK最適化:** 学習時のQWKパラメータ (a, b) を調整。推論時の閾値 (a) も調整。
    * **アンサンブル:** (LGBMモデルのみ?)
