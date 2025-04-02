---
tags:
  - Kaggle
  - DeBERTa
  - LightGBM
startdate: 2023-10-03
enddate: 2024-01-10
---
# Linking Writing Processes to Writing Quality
[https://www.kaggle.com/competitions/linking-writing-processes-to-writing-quality](https://www.kaggle.com/competitions/linking-writing-processes-to-writing-quality)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、学生がエッセイを書く際の**キーボード入力ログデータ**（キーストロークのタイミング、種類、テキストの変化など）のみを用いて、その**エッセイの品質（スコア）を予測する**機械学習モデルを開発することです。
* **背景:** ライティングプロセス（どのように文章が書かれるか）と最終的な文章の質との関係を理解することは、教育分野において重要です。キーボード入力ログは、学生の思考プロセス、計画、修正のパターンなどに関する洞察を提供する可能性があります。このコンペティションは、AIを活用してライティングプロセスデータからエッセイの品質を評価する新しい方法を探る試みです。
* **課題:** 提供されるキーボード入力ログは詳細な時系列データですが、ノイズが多く、また量も限られています（トレーニングデータは約2500件）。このデータから、最終的なエッセイ全体の品質という**抽象的で総合的な評価値（6段階のスコア）**を正確に予測することが求められます。特に、タイピングの癖だけでなく、文章構成や修正プロセスといった高次の情報をどのように抽出・モデル化するかが鍵となります。

**データセットの形式 (Dataset Format)**

提供される主なデータは、各学生（IDで識別）のエッセイ執筆時のキーボード入力ログと、対応するエッセイの評価スコアです。

1.  **トレーニングデータ:**
    * `train_logs.csv`: 各エッセイ(`id`)のキーボード入力イベントの時系列ログ。
        * `event_id`: イベントの連番。
        * `down_time`, `up_time`: キーを押した時刻、離した時刻（ミリ秒）。
        * `action_time`: キーを押してから離すまでの時間。
        * `activity`: イベントの種類（Nonproduction, Input, Remove/Cut, Paste, Replace, Move）。
        * `down_event`, `up_event`: 押された/離されたキーの名前。
        * `text_change`: テキストの変化内容。
        * `cursor_position`: イベント後のカーソル位置。
        * `word_count`: イベント後の単語数。
    * `train_scores.csv`: 各エッセイ(`id`)に対応する評価スコア（0.5から6.0までの0.5刻み）。これが**ターゲット変数**となります。
2.  **テストデータ:**
    * `test_logs.csv`: テスト用のエッセイのキーボード入力ログ。トレーニングデータと同じ形式ですが、`id`は匿名化されている可能性があります。
    * 評価スコアは提供されません。参加者はこのログデータからスコアを予測します。
3.  **`sample_submission.csv`**:
    * 提出フォーマットのサンプル。`id`（エッセイID）と`score`（予測スコア）の列を持ちます。

**評価指標 (Evaluation Metric)**

* **指標:** **RMSE (Root Mean Squared Error; 二乗平均平方根誤差)**
* **計算方法:** 各エッセイの予測スコア ($$ \hat{y}_i $$) と実際のスコア ($$ y_i $$) の差の二乗を計算し、その平均値の平方根を取ります。
    $$ \text{RMSE} = \sqrt{\frac{1}{n} \sum_{i=1}^{n} (\hat{y}_i - y_i)^2} $$
    （n: エッセイの数）
* **意味:** 予測スコアと実際のスコアの間の平均的な誤差の大きさを測ります。RMSEの値が**小さい**ほど、モデルの予測精度が高いことを示します。スコア予測の誤差を二乗するため、大きな誤差に対してよりペナルティが大きくなる特徴があります。

要約すると、このコンペティションは、キーボード入力の時系列ログデータからエッセイの品質スコアを予測する回帰タスクです。データはログファイルとスコアファイルで構成され、性能はRMSE（小さいほど良い）によって評価されます。

---

**全体的な傾向**

このコンペティションでは、キーボード入力ログからエッセイの品質を予測するため、**詳細な特徴量エンジニアリング**が極めて重要でした。特に、ログデータから**最終的なエッセイテキストを再構築**し、そのテキスト自体（単語数、文の長さ、TF-IDF、句読点、文法的な特徴など）や、タイピングプロセス（速度、ポーズ、修正回数、カーソル移動など）に関する特徴量を大量に作成するアプローチが上位解法で共通して見られました。

データセットのサイズが比較的小さい（約2500サンプル）ため、**外部データセット**（他のエッセイ評価コンペのデータやテキストコーパス）を活用して事前学習を行ったり、それらのデータで学習したモデルの予測値を特徴量として利用したりする戦略が有効でした。

モデルとしては、**LightGBM、XGBoost、CatBoost**といった勾配ブースティング木（GBT）モデルが主力として広く使われました。一方で、**ニューラルネットワーク**（MLP、Transformerベースのモデル like DeBERTa）も、特に再構築されたテキスト情報を扱うためや、GBTモデルとのアンサンブル目的で用いられました。DeBERTaなどのTransformerモデルを再構築エッセイに適用し、その出力（埋め込みや予測スコア）を特徴量としてGBTモデルに入力する手法も効果的でした。

**信頼性の高い交差検証（CV）戦略**の構築と、CVスコアを信頼することが、LB（公開リーダーボード）のシェイク（順位変動）を乗り越える上で重要視されました。StratifiedKFoldやRepeatedStratifiedKFoldなどが用いられました。

最終的なスコア向上には、異なるモデルや異なる特徴量セット、異なるシード値で学習させた**複数モデルのアンサンブル**が不可欠でした。単純平均、加重平均、スタッキング（Ridge回帰など）が用いられました。データクリーニングや後処理（予測値のクリッピング）も行われました。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/linking-writing-processes-to-writing-quality/discussion/466873)**

* **アプローチ:** データクリーニング + 詳細な特徴量エンジニアリング + **8つの外部データセット活用** + モデルアンサンブル。
* **アーキテクチャ/アルゴリズム:**
    * 特徴量エンジニアリング用: Fuzzy matchingを用いたエッセイ再構築、LightGBM（外部データスコア予測用）。
    * 最終予測用: LGBM (Regressor/Classifier), XGBoost (Regressor/Classifier), CatBoostRegressor, BaggingRegressor, TabNet, LightAutoML (DenseNet, ResNet, FTTransformer)。
    * アンサンブル: 複数モデルの予測値の**単純平均**。
* **テクニック:**
    * **データクリーニング:** 時間の矛盾修正、Unicodeエラー修正(ftfy)、特定イベント除去。
    * **特徴量エンジニアリング(FE):** 378個の特徴量。キー入力レイテンシ統計、イベント/アクティビティカウント、特定単語数到達時間、ポーズ関連特徴、テキスト統計（単語/文の長さ、句読点エラー）、修正バースト関連、**TF-IDF**（アクティビティ、イベント、キーレイテンシ、再構築テキスト(単語/文字レベル)）、**外部データ予測スコア**。TF-IDFはTruncatedSVDで次元削減(64次元)。
    * **外部データ:** CommonLit, ASAP-AES/SAS, HuggingFace datasets, Feedback Prize, IELTS, Persuade Corpus 2の計8データセット。各データセットで匿名化エッセイのTF-IDF特徴量からLightGBMでスコアを予測し、その予測値を特徴量として追加。
    * **CV:** Nested CV (6 bags x 5 folds, スコアで層化)。
    * **後処理:** 予測値を[0.5, 6.0]にクリッピング。
    * **その他:** CVスコアを信頼。

**[2位](https://www.kaggle.com/competitions/linking-writing-processes-to-writing-quality/discussion/466775)**

* **アプローチ:** 公開ノートブックの特徴量 + **事前学習済みDeBERTaによるコンテキスト特徴量** + モデルアンサンブル。CVスコア信頼。
* **アーキテクチャ/アルゴリズム:**
    * 特徴量生成用: **DeBERTaベース回帰モデル**（匿名化文字'q'で学習）。
    * 最終予測用: LightGBM, XGBoost, CatBoost, MLP, AutoInt, DenseLight (LightAutoMLベース)。
    * アンサンブル: 複数モデルの予測値の**加重平均**（重みはCVに基づきランダム気味に設定）。
* **テクニック:**
    * **特徴量エンジニアリング(FE):** 公開ノートブックの特徴量から重要度上位を選択(Top 64/128/256の和集合)。**DeBERTaコンテキスト特徴量**: 再構築エッセイでDeBERTa回帰モデルを学習し、最終層の一つ手前の層の出力(128次元ベクトル)を特徴量として連結。
    * **モデル学習:** GBDTとNNモデルを学習。NNはLightAutoMLフレームワーク利用。
    * **CV:** StratifiedKFold (5〜10 fold)。複数シード(10〜15)で学習。
    * **後処理:** 予測値を[0.5, 6.0]にクリッピング。

**[3位](https://www.kaggle.com/competitions/linking-writing-processes-to-writing-quality/discussion/466906)**

* **アプローチ:** **GBTアンサンブル** + **MLM事前学習済みDeBERTaアンサンブル** の加重平均 (40/60)。
* **アーキテクチャ/アルゴリズム:**
    * GBTアンサンブル: LGBM, XGBoost, CatBoost, LightAutoML, Shallow NN。
    * DeBERTaアンサンブル: **DeBERTa-v3-base/large**。MLM事前学習 + Fine-tuning。Squeezeformer層追加。
    * アンサンブル: 各アンサンブル内でRidge回帰(OOF予測使用)で重み決定。最終的なGBT/DeBERTaの重みは手動調整(40/60)。
* **テクニック:**
    * **データ前処理/FE:** エッセイ再構築。**難読化文字'q'を'i'や'X'に変更**（DeBERTaのトークナイゼーション改善）。カスタムトークナイザーも試行。GBT用には公開ノートブックの165特徴量を使用。
    * **DeBERTa学習:** Persuade Corpusを難読化して**MLMで事前学習**。その後、コンペの再構築エッセイと追加特徴量（カーソル位置など）でFine-tuning。Dropout/Augmentation強化。
    * **CV:** 8-fold CV。GBMとDeBERTaでCV/LB比が異なる（ドメインシフトの可能性）。
    * **後処理:** 予測値を[0.5, 6.0]にクリッピング（効果は限定的）。

**[4位](https://www.kaggle.com/competitions/linking-writing-processes-to-writing-quality/discussion/466961)**

* **アプローチ:** **特徴量エンジニアリング**に注力 + NNとGBTのアンサンブル。CVとLBの信頼性確保。
* **アーキテクチャ/アルゴリズム:**
    * NN: MLP, DenseLight, AutoInt, NODE (LightAutoMLベース), 1DCNN。
    * GBT: LGBM, CatBoost, XGBoost。
    * アンサンブル: NNアンサンブルとGBTアンサンブルを**等価重み**で結合。各モデルの重みはOptunaで最適化。
* **テクニック:**
    * **特徴量エンジニアリング(FE):** 再構築エッセイの構造に着目。段落/文の長さ（単語数/文字数）、大文字数、名詞数（推定）、非連続的な単語追加数、文頭単語の重複、時間窓特徴量（指定時間内に書かれた単語数）、句読点統計、疑問符/感嘆符数、複数文字置換数、カーソル位置統計（標準偏差、移動回数）、特定長単語の執筆時間、公開ノートの特徴量、**TF-IDF**(ngram=(1,1), 20特徴量)。
    * **モデル学習:** NNは10エポック学習、SWA使用、Early Stopping（最良3つの検証スコア）。GBTは1500イテレーション、Early Stoppingなし。
    * **CV:** StratifiedKFold (スコアで層化)。5つの異なるシードで学習。
    * **シェイクダウン対策:** CVとLBの差を減らす、多様な特徴量セット、NN(Early Stoppingあり)とGBT(Early Stoppingなし)を別々にアンサンブル。

**[6位](https://www.kaggle.com/competitions/linking-writing-processes-to-writing-quality/discussion/467848)** (CPU Only)

* **アプローチ:** CPUのみ使用。**特徴量数を抑制** + **多様なモデル**による**階層的アンサンブル**。CVスコア信頼。
* **アーキテクチャ/アルゴリズム:**
    * Level-0: LGBM(Classifier/Regressor, ExtraTrees含む), CatBoostRegressor, XGBoostRegressor, RandomForestRegressor, Ridge, Lasso, KNN, SVR, LightAutoML(MLP, Dense Light, Dense, ResNet), TabPFN, TabNet。 (計16種類以上)
    * Level-1: ExtraTree + Bagging, BayesianRidge, MLP。 (Level-0のOOF予測を入力)
    * Level-2: Level-1予測の**幾何平均ブレンド**。
* **テクニック:**
    * **FE & 選択:** エッセイ再構築。特徴量数を**169個**に抑制。公開ノートブックの特徴量に加え、**文字レベルTF-IDF(ngram=(1,5)) + TruncatedSVD(29次元)**、Ctrlキー押下回数などを追加。相関の高い特徴量を除去。
    * **モデル多様性:** 多くの異なるモデルタイプを使用。特徴量バギング、ランダムシード平均化も実施。
    * **CV:** Repeated Stratified K-Fold (5-fold x n_repeats)。
    * **後処理:** 予測値を[0.5, 6.0]にクリッピング。

**[7位](https://www.kaggle.com/competitions/linking-writing-processes-to-writing-quality/discussion/466941)**

* **アプローチ:** ベースライン特徴量 + 再構築エッセイ特徴量 + **エッセイ予測値の特徴量化** + モデルアンサンブル。
* **アーキテクチャ/アルゴリズム:**
    * 特徴量生成用: カスタムBPEトークナイザー、CountVectorizer/TfidfVectorizer、LGBM (分類/回帰)。
    * 最終予測用: LGBM, DenseLight, TabNet。
    * アンサンブル: Nelder-Mead最適化による加重平均。
* **テクニック:**
    * **FE:** 公開ノートブックのTop100特徴量 + 再構築エッセイ特徴量（カンマ/引用符/スペース/ドット数） + **BOW**（単語レベル、段落レベル） + **エッセイ予測値特徴量**（カスタムBPEトークンに対するCountVec/TfidfVecで学習したLGBM分類器・回帰器の予測値）。
    * **ラベル前処理:** スコアを3.7以上/未満で**二値化**したラベルを作成し、分類モデルの特徴量生成に利用。
    * **モデル学習:** Optunaでハイパーパラメータ調整。
    * **CV:** GroupKFold (5-fold)。

**[10位](https://www.kaggle.com/competitions/linking-writing-processes-to-writing-quality/discussion/467328)**

* **アプローチ:** **2つの異なる特徴量セット**でモデル群を学習し、加重平均アンサンブル。
* **アーキテクチャ/アルゴリズム:**
    * 特徴量生成用: PCA, TF-IDF, CountVectorizer, **distilroberta**。
    * 最終予測用: LightGBM, XGBoost, CatBoost, DenseLight (LightAutoMLベース)。
    * アンサンブル: 2つの特徴量セットで学習したモデル群の予測を**加重平均** (重みはCV/LBを見ながら調整)。
* **テクニック:**
    * **特徴量セット:**
        * Set 1: 公開特徴量 + PCA, TF-IDF, CountVectorizer, 追加時間関連特徴など (1504個)。
        * Set 2: 公開特徴量 + **distilroberta予測値**。
    * **distilroberta:** 再構築エッセイを使用。'q'の連続数を数字に変換する前処理（例: "qqq qq" -> "3 2"）を適用してトークン数を削減。軽量モデル(distilroberta)の方がdeberta-largeより良い結果。
    * **モデル学習:** GBDTとDenseLightを使用。LGBM/XGBoostはOptunaでチューニング。
    * **CV:** 10-fold CV。5つの異なるシードで学習し平均。
