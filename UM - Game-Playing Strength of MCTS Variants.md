---
tags:
  - Kaggle
url: https://www.kaggle.com/competitions/um-game-playing-strength-of-mcts-variants
startdate: 2024-09-06
enddate: 2024-12-03
---
**全体的な傾向:**

このコンペでは、モンテカルロ木探索（MCTS）の変種のゲームプレイングの強さを予測することが課題です。上位解法は、主に古典的な機械学習モデル（CatBoost、LightGBM、TabM、線形回帰）と、ニューラルネットワーク（MLP、DeepTables）のアンサンブルを使用しています。特徴量エンジニアリングが非常に重要であり、ゲームのルールセットから抽出される特徴量や、MCTSのシミュレーション結果から得られる特徴量などが活用されています。データ拡張も一部で試みられています。

**各解法の詳細:**

**1位**

- **アプローチ:** 各ゲームの開始局面で数秒間の木探索を実行し、ゲームのバランスと探索速度に関する追加の特徴量を計算。既存の特徴量と合わせて、CatBoost、LightGBM、TabM、アイソトニック回帰のスタックアンサンブルに投入。
- **アーキテクチャ:** CatBoost、LightGBM、TabM、アイソトニック回帰のスタックアンサンブル。
- **アルゴリズム:** UCB1Tuned（木探索の選択戦略）、ランダムプレイアウト（木探索のプレイアウト戦略）、Nelder-Mead法（アンサンブルの重み調整）。
- **テクニック:**
    - **開始局面の評価:** UCB1Tunedを用いた木探索によるゲームバランスと探索速度の評価。
    - **特徴量エンジニアリング:** ゲームルールセットから派生する特徴量（ARI、McAlpine EFLAW、CLRIなど）、MCTSのシミュレーション結果（MovesPerSecond、PlayoutsPerSecondなど）、追加の探索速度メトリクス。
    - **データ拡張:** 木探索関連の特徴量を複数回計算し、トレーニング時にランダムに選択。非決定的な特徴量を再計算し、範囲内でランダムサンプリング。
    - **特徴量選択:** 値が一貫していない特徴量（Ludiiプレイヤーのバージョン差異の可能性）を削除。
    - **アイソトニック回帰:** OOF予測を特徴量として使用するアイソトニック回帰モデルをポスト処理として適用。
    - **TabM:** 新しいtabular深層学習ライブラリを活用。
    - **ハイパーパラメータチューニング:** Optunaによる探索。
    - **アンサンブル:** 20モデルのアンサンブル（10分割CVを2シードで実施）。

**2位**

- **アプローチ:** 伝統的な5分割CVトレーニング。追加のオフライン生成トレーニングデータは不使用。高度な特徴量エンジニアリングは行わず（LudRulesからのテキストスコアを除く）。特徴量選択は限定的。データ拡張を重視。
- **アーキテクチャ:** CatBoost (2種類), LightGBM (2種類), MLP (2種類), Auto-Encoder + MLP。
- **アルゴリズム:** Tfidf Vectorization、Kmeansクラスタリング（CV分割）、Ordinary Least Squares (OLS) 回帰（アンサンブルの重み）。
- **テクニック:**
    - **データ準備:** 重複および非情報的な特徴量を削除。LudRulesからARI、McAlpine EFLAW、CLRIスコアを計算。LudRulesに基づくグループでソートし、類似ゲームをクラスタリングしてCV分割。エージェントのフリップ、AdvantageP1の反転、utility_agent1の符号反転によるデータ拡張。カテゴリカル特徴量のOne-Hotエンコーディング。連続変数の標準正規分布へのマッピング。
    - **モデル:** 7つのモデルのアンサンブル（CatBoost、LightGBM、MLP、AE+MLP）。
    - **アンサンブル:** CV OLS回帰による重み付け。キャップされたアンサンブル予測を再度CV OLS回帰でスケーリング。異なるシードでトレーニングされた6つのモデルセットの平均。
    - **LBオーバーフィット:** NNモデルの拡張データ予測を含め、LBで直接重みを調整。等しい重みの予測セットを追加し、LBで直接負の重みを調整。

**3位**

- **アプローチ:** 2段階フリップ拡張スタッキング。フリップ拡張データでトレーニングした1段階目モデルのOOF予測を、2段階目の特徴量として使用。
- **アーキテクチャ:** CatBoost、LightGBM (Dartモード)。
- **アルゴリズム:** StratifiedGroupKFold（CV分割）、Optuna（ハイパーパラメータチューニング）。
- **テクニック:**
    - **データ拡張:** agent1とagent2の入れ替え、AdvantageP1の反転、utility_agent1の符号反転。
    - **CV:** GameRulesetNameによるStratifiedGroupKFold（8分割）。utility_agent1の少数クラスを近傍クラスに修正。
    - **特徴量エンジニアリング:** yunsuxiaozi氏のノートブックから特徴量を追加。EnglishRulesのTF-IDF-SVD特徴量（max_featuresを調整）。
    - **特徴量選択:** Null importance feature selection。
    - **ハイパーパラメータチューニング:** Optuna。
    - **モデリング:** 2段階モデリング。1段階目はフリップ拡張データでトレーニングし、OOF予測を生成。2段階目はOOF予測を特徴量として使用。
    - **アンサンブル:** LightGBMとCatBoostの3シード平均アンサンブル。
    - **後処理:** 予測値に係数（1.12）を乗算。

**4位**

- **アプローチ:** パブリックLBのスコアに合わせて調整された、他のKagglerの共有ノートブックの組み合わせ（Kaggle Learning）。スタッキング戦略（CVに基づきつつLBにオーバーフィット）。
- **アーキテクチャ:** DeepTables NN、LightGBM、CatBoost、XGBoost、RandomForest、Ridge、Lasso、KNN、SVR、LightAutoML (MLP, Dense Light, Dense, ResNet)、TabPFN、TabNet。
- **アルゴリズム:** Repeated Stratified K-Fold CV。
- **テクニック:**
    - **アンサンブル:** 複数の公開ノートブックの予測を組み合わせる。カスケードマージと手動回帰（クリッピングあり）。
    - **CV:** 5分割GKF。
    - **LBへのオーバーフィット:** 手動で重みを調整。

**5位**

- **アプローチ:** 修正されたAdvantageP1特徴量、CatBoost、2つのLGBM Dartモデルのアンサンブル。異なるデータ/特徴量でトレーニング。
- **アーキテクチャ:** CatBoost、LightGBM (Dartモード)。
- **アルゴリズム:** Stratified Group 10-Fold（CV分割）。
- **テクニック:**
    - **特徴量エンジニアリング:** 調整されたAdvantageP1特徴量。
    - **データ拡張:** agent1とagent2の入れ替え、AdvantageP1の反転、utility_agent1の符号反転（TTAとして推論時に使用）。
    - **モデリング:** CatBoostとLightGBM Dartモデル。
    - **アンサンブル:** AnilとSercanのモデルセットの重み付きブレンド。

**6位**

- **アプローチ:** ゼロコストデータ生成、適切なモデリング、手動および自動の特徴量生成、2段階モデルによるアンサンブル。
- **アーキテクチャ:** CatBoost、LGBM、XGB、DNN (MLP)、OpenFE (自動特徴量生成)。
- **アルゴリズム:** StratifiedGroupKFold（CV分割）、OpenFE（自動特徴量生成）。
- **テクニック:**
    - **ゼロコストデータ生成:** agentの入れ替え、AdvantageP1の反転、utility_agent1の反転。
    - **手動特徴量生成:** agent文字列の分割、litsea氏のノートブックからの特徴量、クロスエージェント特徴量。
    - **自動特徴量生成:** OpenFEライブラリを使用。
    - **特徴量選択:** CV重要度に基づく。
    - **モデリング:** Catboost、LGBM、XGB、DNN。2段階モデル（Catboost OOFを特徴量または初期化として使用）。
    - **アンサンブル:** 重み付きアンサンブル（scipy minimize関数で重みを選択）。LB分布へのマッチング（クリッピング、スケーリング）。

**7位**

- **アプローチ:** TreeモデルとNNモデルのアンサンブル。フリップ拡張を2段階で行うスタッキング。
- **アーキテクチャ:** LGB、CatBoost、NN (DeepTables)。
- **アルゴリズム:** Stratified Group K-Fold（CV分割）。
- **テクニック:**
    - **データ拡張:** agent1とagent2の入れ替え、AdvantageP1の反転、ラベルの反転（非常に低い確率）。テスト時拡張（TTA）。
    - **特徴量エンジニアリング:** 数値特徴量の交差特徴量、ターゲットエンコーディング。
    - **モデリング:** LGB、CatBoost、DeepTables NN。
    - **2段階モデリング:** フリップ拡張データで1段階目を学習し、OOF予測を生成。OOF予測を2段階目の特徴量として使用。
    - **アンサンブル:** 重み付き平均。

**8位**

- **アプローチ:** 10日間の挑戦。MCTS Starterをベースに、polarsによる高速化、ゲームルールに基づいた特徴量エンジニアリング、データ拡張、CatBoostのアンサンブル。
- **アーキテクチャ:** CatBoost。
- **アルゴリズム:** GroupKFold（CV分割）。
- **テクニック:**
    - **高速化:** polarsライブラリによるデータ処理の高速化。
    - **特徴量エンジニアリング:** ボード形状、駒の種類、ランダム性の有無、ルール複雑性、特殊移動パターンなど、ゲームルールに基づいた特徴量。
    - **データ拡張:** agent1とagent2の入れ替え、より複雑な同値関係と推移律に基づく拡張。
    - **後処理:** 予測結果に係数（1.2）を乗算（LBで最適化）。

**9位**

- **アプローチ:** 様々な拡張と多数のモデリングトリック。修正されたAdvantageP1特徴量、CatBoost、2つのLGBM Dartモデルのアンサンブル。
- **アーキテクチャ:** CatBoost、LightGBM (Dartモード)。
- **アルゴリズム:** 8分割GroupKFold（CV分割）。
- **テクニック:**
    - **データ拡張:** agent1とagent2の入れ替え、AdvantageP1の反転、utility_agent1の符号反転（TTAとして推論時に使用）。
    - **特徴量エンジニアリング:** AdvantageP1の修正。
    - **モデリング:** CatBoostとLGBM Dartモデル。utility_agent1 - AdvantageP1 を予測。AdvantageP1を最終アンサンブルに追加。Wins/Lossesの予測を試行。MultiRMSEを試行。エージェントごとのモデルを試行。フルフィット。分類器の試行。疑似ラベリングの試行。
    - **アンサンブル:** 重み付き平均（クリッピングあり）。

**10位**

- **アプローチ:** MCTS Starterをベースに、データ拡張（agent順序の入れ替え）、AdvantageP1のビニング、フォールド数の増加。予測値に定数を乗算（オーバーフィット）。
- **アーキテクチャ:** CatBoost（MCTS Starterをベース）。
- **アルゴリズム:** GroupKFold（CV分割）。
- **テクニック:**
    - **データ拡張:** agent順序の入れ替え、AdvantageP1の反転、utility_agent1の符号反転。
    - **特徴量エンジニアリング:** AdvantageP1のビニング（10個のビン）。
    - **CV:** 10分割GroupKFold。
    - **アンサンブル:** 複数のモデルの平均。
    - **後処理:** 予測値に定数（1.25または1.3）を乗算（LBで最適化）。