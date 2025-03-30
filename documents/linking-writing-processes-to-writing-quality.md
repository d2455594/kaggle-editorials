---
tags:
  - Kaggle
  - テキスト生成
  - LLM
  - NLP
startdate: 2023-10-03
enddate: 2024-01-10
---
# Linking Writing Processes to Writing Quality
https://www.kaggle.com/competitions/linking-writing-processes-to-writing-quality

**全体的な傾向**

上位解法では、キーボード操作ログから再構築されたエッセイテキストと、そこから抽出された様々な特徴量を活用するアプローチが主流でした。Transformerベースの言語モデル（特にDeBERTa）や、Gradient Boosting Machine（LightGBM、XGBoost、CatBoost）などの機械学習モデルが広く用いられています。データクリーニング、外部データの利用、特徴量エンジニアリング、モデルアンサンブルなどが重要なテクニックとして用いられました。

**各解法の詳細**

**1位**

- **アプローチ:** 徹底的なデータクリーニング、豊富な特徴量エンジニアリング、外部データの利用、多様なモデルのアンサンブル。CV（交差検証）を重視。
- **アーキテクチャ:** LightGBM、XGBoost、CatBoost、BaggingRegressor、TabNet、LightAutoML（DenseNet、ResNet、FTTransformer）。
- **アルゴリズム:** TF-IDF、Truncated SVD、LightAutoML、Optuna（ハイパーパラメータチューニング）、Forward Selection（アンサンブル）。
- **テクニック:**
    - **データクリーニング:** イベントの除去、up/downタイムの修正、Unicodeエラーの修正、異常データの除去。
    - **エッセイの再構築:** カーソル位置とテキスト変更情報を利用。
    - **特徴量エンジニアリング:** キー入力遅延、イベントカウント、時間経過、ポーズ関連、テキスト統計量、句読点エラー、リビジョンバースト、外部エッセイスコア予測（TF-IDF+LightGBM）、単語/文字レベルTF-IDF。
    - **外部データ:** 複数のエッセイ評価データセットを利用。
    - **モデル:** 回帰モデルと分類モデルの両方を使用。多様なモデル（Tree系、NN系）。
    - **アンサンブル:** 線形回帰、ロジスティック回帰、平均、Forward Selection。Nested CV。
    - **後処理:** 予測値のクリッピング。

**2位**

- **アプローチ:** 2段階学習（Kaggle-Persuadeで事前学習後、Kaggle-Onlyでファインチューニング）。MLM（Masked Language Modeling）による事前学習。最適な閾値を探索し、モデルをアンサンブル。
- **アーキテクチャ:** DeBERTa-v3-largeを主に使用。
- **アルゴリズム:** BCEWithLogitsLoss（順序回帰）、AdamWオプティマイザ。OptimizedRounder（閾値探索）。
- **テクニック:**
    - **2段階学習:** テストセットと類似性の高いKaggle-Onlyデータでファインチューニング。
    - **MLM:** 全学習データで10エポックのMLM事前学習。
    - **CV:** 4分割SKF。Kaggle-PersuadeとKaggle-Onlyの両データで検証。
    - **閾値探索:** OptimizedRounderを使用し、最終閾値を調整。
    - **モデル:** CoPE（Editing）、順序回帰、Petモデル（prompt tuning）、文埋め込みの平均プーリングなど、多様なモデルを試行。
    - **アンサンブル:** 投票アンサンブル。

**3位**

- **アプローチ:** MLM事前学習済みDeBERTaとGBDTのブレンド。エッセイテキストの難読化解除とトークナイザの工夫。CVとLBの相関を分析。
- **アーキテクチャ:** DeBERTa-v3-base、DeBERTa-v3-large、LGBM、XGBoost、CatBoost、LightAutoML、Shallow NN。Squeezeformerレイヤー（DeBERTaモデル内）。
- **アルゴリズム:** LightGBM、XGBoost、CatBoost、LightAutoML、Ridge回帰（アンサンブル）、Optuna（ハイパーパラメータチューニング）。
- **テクニック:**
    - **エッセイテキストの再構築:** `q` を `i` または `X` に置換。カスタムトークナイザの訓練。
    - **特徴量エンジニアリング (GBT):** 公開カーネルの165特徴量を使用。
    - **DeBERTaの訓練:** PersuadeコーパスでMLM事前学習。カーソル位置などの追加特徴量。Squeezeformerレイヤー。ヘビードロップアウト/オーグメンテーション。
    - **アンサンブル:** DeBERTaアンサンブルとGBTアンサンブルを40:60で重み付け。OOF予測に対する正のRidge回帰でブレンド重みを決定。

**4位**

- **アプローチ:** 特徴量エンジニアリングを重視。Persuadeデータと非Persuadeデータの非互換性を考慮。データソースのタグ付け、データソース分類ヘッドの追加、非Persuadeデータのスコアに基づく早期停止。
- **アーキテクチャ:** MLP、DenseLight、AutoInt、NODE（LightAutoML）、1DCNN、LGBM、CatBoost、XGBoost。
- **アルゴリズム:** LightAutoML、Optuna（アンサンブルの重み決定）。
- **テクニック:**
    - **特徴量エンジニアリング:** 段落長、文長、大文字数、名詞数、非連続追加単語数、同じ単語で始まる文の数、時間窓ベースの特徴量、句読点統計量、TF-IDF、カーソル位置ベースの特徴量、アクション時間。
    - **モデリング:** NN系モデルとTree系モデルのアンサンブル。
    - **学習戦略:** LightAutoMLは10エポック。Stochastic Weighted Averaging。早期停止。Tree系モデルは1500イテレーション。
    - **CV:** スコアによる層化K分割。
    - **アンサンブル:** NNとTree系を独立して重み最適化（Optuna）。

**6位**

- **アプローチ:** CV（交差検証）を重視。CPUのみで計算。エッセイ再構築、特徴量エンジニアリングと選択、多様なモデルのアンサンブル。
- **アーキテクチャ:** LightGBM Classifier/Regressor、CatBoost Regressor、XGBoost Regressor、RandomForest Regressor、Ridge、Lasso、KNN、SVR、LightAutoML（MLP、Dense Light、Dense、ResNet）、TabPFN、TabNet。
- **アルゴリズム:** Repeated Stratified K-Fold CV、TruncatedSVD（次元削減）、python-minifier（コード最小化）。
- **テクニック:**
    - **エッセイ再構築:** `essay constructor notebook` を参照。
    - **特徴量エンジニアリングと選択:** 文字レベルTF-IDF、'Ctrl' キー押下回数、相関の高い特徴量の削除など。特徴量は169個に限定。
    - **検証:** Repeated Stratified K-Fold CV。
    - **モデル:** 分類モデルと回帰モデルの両方を使用。多様なモデル（Tree系、線形モデル、KNN、SVR、LightAutoML、TabPFN、TabNet）。特徴量バギング。Random seed averaging。
    - **アンサンブル:** 幾何平均ブレンド。
    - **後処理:** 予測値を0.5から6.0の範囲にクリッピング。

**7位**

- **アプローチ:** エッセイ予測を特徴量として利用。公開ノートブックをベースに改善。Kaggle-onlyデータの重複、ハイパーパラメータ調整。
- **アーキテクチャ:** LGBM、DenseLight、TabNet。
- **アルゴリズム:** Optuna（ハイパーパラメータチューニング）、Nelder-Mead（アンサンブルの重み最適化）。
- **テクニック:**
    - **ベースライン特徴量:** 公開ノートブックの165特徴量から上位100個を選択。
    - **エッセイ特徴量エンジニアリング:** カンマ数、引用符数、スペース数、ドット数など。単語と段落のBag-of-Words。
    - **ラベル前処理:** ラベルを二値化（3.7以上を1、それ以外を0）。
    - **エッセイ予測を特徴量として利用:** カスタムBPEトークナイザでCountVectorizerとTfidfVectorizerを訓練し、LGBMで二値分類と回帰予測を行い、それらを新たな特徴量として追加。
    - **モデル:** 全てのモデルで同じ特徴量を使用し、Optunaでチューニング。
    - **アンサンブル:** Nelder-Mead最適化で重みを決定。

**10位**

- **アプローチ:** 2つの異なる特徴量セットでLightGBM、XGBoost、CatBoost、DenseLightを訓練し、それらの予測を異なる重みで平均化。重み比率はLocal CVとPublic LBを観察しながら調整。
- **アーキテクチャ:** LightGBM、XGBoost、CatBoost、DenseLight（LightAutoML）。DistilRoBERTa（特徴量セット2）。
- **アルゴリズム:** Optuna（LightGBM、XGBoostのハイパーパラメータチューニング）。
- **テクニック:**
    - **特徴量セット1:** 公開ノートブックの様々な特徴量、PCA、TF-IDF、CountVectorizer、追加のup_time_lagged特徴量を含む1504特徴量。
    - **特徴量セット2:** 公開ノートブックの特徴量とDistilRoBERTaの予測。
    - **DistilRoBERTa:** 再構築されたエッセイを前処理（'q'の連続数をカウント）。
    - **学習戦略:** 両方の特徴量セットで、10分割CVと5シードの平均。
    - **アンサンブル:** 重み付き平均。