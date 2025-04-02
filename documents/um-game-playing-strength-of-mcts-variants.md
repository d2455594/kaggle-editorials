---
tags:
  - Kaggle
  - MCTS
  - LightGBM
  - DeepTables
  - RMSE
  - DeBERTa
startdate: 2024-09-06
enddate: 2024-12-03
---
# UM - Game-Playing Strength of MCTS Variants
[https://www.kaggle.com/competitions/um-game-playing-strength-of-mcts-variants](https://www.kaggle.com/competitions/um-game-playing-strength-of-mcts-variants)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、様々なルールを持つボードゲームにおいて、異なる構成（パラメータ）を持つ2つの**モンテカルロ木探索（Monte Carlo Tree Search, MCTS）**エージェントが対戦した場合の**勝敗結果（エージェント1から見た期待効用値）を予測**する回帰モデルを開発することです。
* **背景:** MCTSは、囲碁AI「AlphaGo」などで知られるように、完全情報ゲームにおいて非常に強力な探索アルゴリズムです。しかし、その性能は選択戦略（UCB1など）、探索定数、プレイアウト戦略（ランダム、N-Gramなど）、スコア境界の扱いといった様々なパラメータ設定や、ゲーム自体の特性（複雑さ、引き分けやすさなど）に大きく依存します。事前にMCTSのバリアント間の強さを予測できれば、特定のゲームに最適なアルゴリズム構成の選択や、アルゴリズム自体の改善に繋がる可能性があります。
* **課題:** 多種多様なゲームルール（汎用ゲームシステムLudiiで記述）と、多数のMCTSパラメータの組み合わせに対して、対戦結果を予測する必要があります。入力として、ゲームルールテキスト、MCTSエージェントの構成文字列、および事前計算されたゲーム特性（ランダムプレイ時の勝率やゲームの複雑性など）が与えられます。これらの異種混合データから、実際のMCTSエージェント間の勝敗に関わる特徴量を抽出し、-1（敗北）から1（勝利）の間の連続値を予測する回帰問題です。特に、テキスト形式のゲームルールやカテゴリ的なエージェント構成をどのように効果的にモデルに取り込むかが課題となりました。

**データセットの形式 (Dataset Format)**

提供される主なデータは、ゲームルール、エージェント構成、ゲーム特性、および対戦結果です。

1.  **`train.csv`, `test.csv`**:
    * 各行が特定のゲームにおける特定のMCTSエージェントペアの対戦結果（または予測対象）を示します。
    * `Id`: 各データ行の一意なID。
    * `GameRulesetName`: ゲームルールセットの名前（例: `Breakthrough.lud`）。
    * `agent1`, `agent2`: 対戦する2つのMCTSエージェントの構成を示す文字列。形式は `MCTS-<selection>-<exploration_const>-<playout>-<score_bounds>`（例: `MCTS-UCB1Tuned-0.6-Random-true`）。
    * `utility_agent1`: **ターゲット変数**。エージェント1から見た期待効用値。複数回の対戦結果（勝利: +1, 引き分け: 0, 敗北: -1）の平均値で、-1から1の間の連続値。
    * `LudRules`: Ludiiシステムで解釈される形式のゲームルールテキスト。
    * `EnglishRules`: 自然言語（英語）で記述されたゲームルールテキスト。
    * **ゲーム特性特徴量:** ランダムプレイシミュレーション等に基づいて計算された多数の特徴量。
        * `AdvantageP1`: ランダムプレイにおけるプレイヤー1の有利さ（勝率 - 敗率）。
        * `Balance`: ゲームのバランス（(勝率1 + 勝率2)/2）。
        * `Completion`: ゲームが終了する確率。
        * `Drawishness`: 引き分けになる確率。
        * `StateTreeComplexity`, `GameTreeComplexity`: ゲームの複雑性指標。
        * など、多数。
    * **エージェント性能特徴量:** 対戦シミュレーション時のエージェントの探索効率などを示す特徴量。
        * `MovesPerSecond`, `PlayoutsPerSecond`: 1秒あたりの着手数、プレイアウト数。
        * `DurationActions`, `DurationTurns`, `DurationTurnsStdDev`: 行動やターンの持続時間に関する統計量。
        * `Timeouts`: タイムアウト発生回数。
        * など、多数。
2.  **外部データ / 生成データ:**
    * 一部の参加者は、GAVELアルゴリズムやLLMを用いて**新しいゲームルールセットを生成**し、それらに対してMCTSエージェントの対戦シミュレーションを実行して**追加の学習データを生成**しました。
3.  **`sample_submission.csv`**:
    * 提出フォーマットのサンプル。`Id`と予測値`utility_agent1`の列を持ちます。

**評価指標 (Evaluation Metric)**

* **指標:** **RMSE (Root Mean Squared Error, 二乗平均平方根誤差)**
* **計算方法:** モデルが予測した`utility_agent1`の値と、実際の（隠された）`utility_agent1`の値との間の二乗誤差を計算し、その平均値の平方根を取ります。
    * RMSE = √ [ (1/N) * Σ (予測値_i - 真の値_i)² ]
* **意味:** 予測値と真の値の間の平均的な誤差の大きさを測る標準的な回帰指標です。誤差の二乗を平均するため、大きな誤差の影響をより強く受けます。スコアは**低い**ほど、モデルの予測精度が高いことを示します。

---

**全体的な傾向**

このMCTSエージェントの強さ予測タスクでは、表形式データに対する強力なアルゴリズムである**勾配ブースティング木 (GBDT)**、特に**CatBoost**と**LightGBM**が広く用いられ、高い性能を示しました。ニューラルネットワーク（NN）系のモデル（MLP, TabM, DeepTablesなど）も試されましたが、GBDTが優位な状況でした。最終的な予測は、これらのモデルを複数組み合わせた**アンサンブル**によって行われるのが一般的でした。

**特徴量エンジニアリング**が重要な役割を果たしました。提供された多数の数値特徴量（ゲーム特性、エージェント性能特性）はそのまま、あるいは四則演算などで組み合わせて利用されました。`agent1`, `agent2` の文字列は、MCTSの各構成要素（選択戦略、探索定数、プレイアウト戦略、スコア境界）に分解され、**カテゴリ特徴量**として扱われました。さらに、両エージェントの構成要素の組み合わせから**相互作用特徴量**を作成することも有効でした。ゲームルールテキスト (`LudRules`, `EnglishRules`) からは、**TF-IDF + SVD/PCA**や、テキストの**可読性指標**（ARI, McAlpine EFLAW, CLRIなど）が特徴量として抽出されました。

特に1位の解法では、各ゲームの**開始局面で短時間のMCTSを実行**し、その結果（評価値、探索効率など）を**追加特徴量**として利用するという独自のアプローチが採用され、これが大きな性能向上に繋がったと報告されています。

**データ拡張**として、`agent1`と`agent2`を入れ替え、一部の特徴量（`AdvantageP1`, ターゲット`utility_agent1`）の符号や値を反転させたデータを生成する**Flip Augmentation**が非常に効果的であり、多くのチームで採用されました。この拡張は学習データの倍増だけでなく、推論時の**TTA (Test Time Augmentation)** としても利用されました。他の拡張手法（Self-play, Transitivity）も試されました。

モデル構築においては、**スタッキング**（Stage 1モデルの予測値を特徴量としてStage 2モデルを学習）や、**Isotonic Regression**による予測値のキャリブレーション（後処理）も有効なテクニックでした。予測値全体を定数倍するスケーリングや、範囲を[-1, 1]にクリップする後処理も試されました。

**交差検証（CV）** 戦略としては、`GameRulesetName`でグループ化し、同一ゲームのデータが異なるFoldに分割されないようにする**GroupKFold**が基本でした。一部では、ルールの類似性に基づいてゲームをクラスタリングしてからGroupKFoldを行ったり、ターゲット値で層化（Stratification）したりする工夫も見られました。ただし、CVスコアとリーダーボード（LB）スコアの相関が低い場合もあり、信頼できるCVの構築が課題となることもありました。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/um-game-playing-strength-of-mcts-variants/discussion/549801)**

* **アプローチ:** 開始局面での短時間MCTSによる追加特徴量生成 + GBDT/NNモデル + Isotonic Regressionによるスタッキングアンサンブル。
* **アーキテクチャ:** Base Models: CatBoost, LightGBM, TabM。Stacking Model: Isotonic Regression (Centered Isotonic Regression)。
* **アルゴリズム:** CatBoost, LightGBM, TabM (回帰)。Isotonic Regression。
* **テクニック:**
    * **追加特徴量:** 各ゲーム開始局面でUCB1Tuned-Random MCTSを15秒実行し、評価値や探索速度などを特徴量化（スコアを約0.045 CV, 0.012 LB改善）。
    * **データ:** 自作の追加学習データ（GAVELアルゴリズムとLLMでルール生成＋対戦シミュレーション、14k行）を使用（重み付けあり）。
    * **データ拡張:** MCTS特徴量の複数回計算とランダム選択。提供特徴量の複数回再計算とノイズ付与。
    * **特徴量選択:** Ludiiバージョン差異が疑われる43特徴量を除外するとCVが改善。
    * **スタッキング:** 各ベースモデルのOOF予測をIsotonic Regressionに入力。
    * **ハイパーパラメータ:** Optunaで最適化。TabMは論文外の広い隠れ層/低LR/小バッチサイズが有効。LightGBMは`boosting=dart`が重要。
    * **アンサンブル:** 各スタックモデルは内部で20モデル（10 fold x 2 seed）のアンサンブル。最終的にCatBoost/LightGBM/TabMのスタック出力をNelder-Meadで重み最適化してアンサンブル。
    * **CV戦略:** GroupKFoldShuffle (`GameRulesetName`)。

**[2位](https://www.kaggle.com/competitions/um-game-playing-strength-of-mcts-variants/discussion/549718)**

* **アプローチ:** Flip Augmentation + GBDT/NNモデルの重み付きアンサンブル。CVとLB乖離からLB重視へ。
* **アーキテクチャ:** CatBoost (x2), LightGBM (x2), MLP (x2), AE+MLP。
* **アルゴリズム:** CatBoost, LightGBM, MLP, AutoEncoder (回帰)。OLSによるアンサンブル重み決定と最終スケーリング。
* **テクニック:**
    * **データ拡張:** **Flip Augmentation**を使用（学習データ倍増）。
    * **特徴量:** `LudRules`から可読性指標 (ARI, McAlpine EFLAW, CLRI) を抽出。カテゴリ特徴量はOne-hotエンコード。一部GBDTモデルではルールセットグループのダミー変数も使用。NN入力は標準正規分布にスケーリング。
    * **CV戦略:** `LudRules`のTF-IDF+KMeansでルールをクラスタリングし、そのグループIDでGroupKFold (5-split)。
    * **アンサンブル:** 7モデルの予測値をOLSで重み付け（一部NNモデルの重みが負になったが、LBスコア改善のため採用）。6つの異なるシードで学習したモデルセットをさらに等価重みでアンサンブル。NNモデルのFlip Augmentation版予測はCVでは除外したが、最終的なLB重視提出では負の重みで追加。
    * **後処理:** 予測値を[-1, 1]にクリップ後、OLSでスケーリング。
    * **LB重視:** CVベース提出よりも、LBスコアに合わせてNNのAugmentation版予測や等価重み予測を追加した提出の方がPrivateスコアも良かった。

**[3位](https://www.kaggle.com/competitions/um-game-playing-strength-of-mcts-variants/discussion/549588)**

* **アプローチ:** Flip Augmentation + 2段階スタッキング。LightGBMとCatBoostのアンサンブル。
* **アーキテクチャ:** Stage 1: LightGBM, CatBoost。Stage 2: LightGBM, CatBoost。
* **アルゴリズム:** LightGBM, CatBoost (回帰)。
* **テクニック:**
    * **データ拡張:** **Flip Augmentation**。
    * **CV戦略:** StratifiedGroupKFold (`GameRulesetName`, ターゲット値で層化)。
    * **特徴量:** yunsuxiaozi氏のノートブックから可読性指標などを追加。`EnglishRules`のTF-IDF+SVD特徴量も追加（パラメータ探索重要）。
    * **特徴量選択:** Null Importanceで不安定な特徴量を除去。
    * **スタッキング:** Stage 1モデルのOOF予測をStage 2モデルの特徴量として使用。
    * **ハイパーパラメータ:** Optunaで最適化。
    * **アンサンブル:** LightGBMとCatBoostのブレンド。3シード平均。
    * **後処理:** 最終予測値に係数(1.12)を乗算（予測値の平均がターゲット平均より低いため）。

**[4位](https://www.kaggle.com/competitions/um-game-playing-strength-of-mcts-variants/discussion/549603)**

* **アプローチ:** 多数の公開ノートブックの予測結果を**カスケード式にアンサンブル**。LBスコアに基づき手動で重みを調整。
* **アーキテクチャ:** アンサンブル対象として、公開ノートブック由来のGBDT (LightGBM, CatBoost) やNN (DeepTables) モデルを利用。
* **アルゴリズム:** 回帰モデルのアンサンブル (手動重み調整)。
* **テクニック:**
    * **モデルソース:** yunsuxiaozi氏、hideyukizushi氏、yekenot氏など多数の公開ノートブックの予測結果を利用。
    * **アンサンブル戦略:** ベースとなるアンサンブルから開始し、他のモデルの予測結果を段階的に追加していく。各ステップで、追加する予測結果に対する重み（正または負）を手動で調整し、Public LBスコアが改善するように進める。
    * **後処理:** 最終的なアンサンブル予測値に対し、定数倍のスケーリング（x1.25）とクリッピング（-0.98〜0.98）、わずかなシフト（±0.005）を適用。
    * **LB重視:** CVスコアは考慮せず、Public LBスコアのみを指標としてアンサンブルを構築（LBオーバーフィット戦略）。
    * **備考:** このアプローチは他の参加者の成果に大きく依存しており、モデル構築や特徴量エンジニアリングの独自性は低い。

**[5位](https://www.kaggle.com/competitions/um-game-playing-strength-of-mcts-variants/discussion/549585)**

* **アプローチ:** CatBoost, LightGBM (Dart), DeepTablesモデルの重み付きアンサンブル。Flip Augmentation。調整済みAdvantage特徴量。
* **アーキテクチャ:** CatBoost, LightGBM (Dart), DeepTables (NN)。
* **アルゴリズム:** CatBoost, LightGBM, DeepTables (回帰)。重み付き平均アンサンブル。
* **テクニック:**
    * **データ拡張:** **Flip Augmentation**（学習とTTA）。
    * **特徴量:** `AdvantageP1`を調整（`Completion`と`Drawishness`を考慮）。TF-IDF特徴量（`EnglishRules`, `LudRules`, `agent1`, `agent2`）。
    * **CV戦略:** Stratified Group 10-Fold (`GameRulesetName`グループ、ターゲット値で層化）。
    * **学習:** GBDTはStackingも使用。DeepTablesはMin-Maxスケーリング、LRスケジューラ、Adam。
    * **アンサンブル:** 各メンバーのパイプライン（Anil: CatBoost, Sercan: CatBoost+LightGBM+DeepTablesのブレンド）の予測結果をさらに重み付きでブレンド。

**[6位](https://www.kaggle.com/competitions/um-game-playing-strength-of-mcts-variants/discussion/549582)**

* **アプローチ:** Zero-costデータ生成（Flip Augmentation + Balance反転）、自動特徴量生成（OpenFE）、GBDT/NNの2段階スタッキングアンサンブル。LB重視の後処理。
* **アーキテクチャ:** Base: CatBoost。Stage 2: CatBoost (OOF特徴量), LightGBM (OOF特徴量), CatBoost (OOFベースライン), MLP (OOF特徴量 + 元特徴量)。
* **アルゴリズム:** CatBoost, LightGBM, MLP (回帰)。RMSE損失。MADGRAD (MLP)。
* **テクニック:**
    * **データ拡張:** **Flip Augmentation** に加え、`Balance` 特徴量も反転させることで、元データと拡張データのOOFスコア差を埋める。Self-play拡張は効果なし。TTAも可能だったが最終提出では不使用。
    * **特徴量:** エージェント文字列分解、公開ノートブック由来の特徴量、エージェント間相互作用特徴量。**OpenFE**で自動生成した特徴量（Top500）も使用。ルールテキスト特徴量は効果なし。
    * **CV戦略:** Stratified Group K-Fold (`GameRulesetName`, ターゲット値, `agent1`で層化)。5 folds。拡張データは学習のみに使用し、検証は元データのみ。
    * **スタッキング:** Nested CV (5x5 Fold) でCatBoostのOOF予測を生成し、Stage 2モデルの特徴量またはベースラインとして使用。
    * **NN:** カテゴリ特徴量はEmbedding、数値特徴量はQuantile Transformerで処理後、MLPに入力。
    * **アンサンブル:** Stage 1 CatBoostとStage 2モデル群の重み付きアンサンブル（重みはCVベースでScipy Minimizeにより最適化）。
    * **後処理:** Public LB probingに基づき、最終予測値を `clip(pred * 1.175, -1, 1)` で調整（LBオーバーフィット）。

**[7位](https://www.kaggle.com/competitions/um-game-playing-strength-of-mcts-variants/discussion/549617)**

* **アプローチ:** Flip Augmentation + GBDT/NNアンサンブル。数値特徴量のクロス特徴量生成。
* **アーキテクチャ:** LightGBM, CatBoost, NN (PyTorch実装、CIN+FMプーリング)。
* **アルゴリズム:** LightGBM, CatBoost, NN (回帰)。RMSE損失。
* **テクニック:**
    * **データ拡張:** **Flip Augmentation**（学習とTTA）。
    * **CV戦略:** GroupKFold (`GameRulesetName`) + ターゲット値で層化。
    * **特徴量:** Top20数値特徴量のペアワイズな積・商を追加（クロス特徴量、効果大）。Target Encoding。LightGBMの葉インデックス特徴量 (depth 5, 200 trees) をNNに追加。
    * **学習:** Treeモデルは深めに設定 (LGBM 16, CBT 10)、正則化を強めに (lambda=10)。NNはビン化数値特徴量とカテゴリ特徴量入力。ラベル摂動 (Multinomial)。
    * **アンサンブル:** LightGBM (10 folds) + NN (5 folds) の重み付きブレンド (1:0.75)。
    * **後処理:** 予測値に係数を乗算 (a*x + b の形式でOOFに基づき最適化)。

**[8位](https://www.kaggle.com/competitions/um-game-playing-strength-of-mcts-variants/discussion/549616)**

* **アプローチ:** 単一のCatBoostモデル。Polarsによる高速な特徴量エンジニアリング。Flip Augmentationと、より複雑な関係性（推移律など）に基づくデータ拡張。
* **アーキテクチャ:** CatBoost。
* **アルゴリズム:** CatBoost (回帰)。RMSE損失。
* **テクニック:**
    * **特徴量:** 公開ノートブック由来の可読性指標など。ゲームボード形状、ピースタイプ、ランダム性有無、ルール複雑性、特殊移動パターンなど、ゲームルールに基づいた特徴量エンジニアリング。
    * **データ拡張:** **Flip Augmentation**。さらに、`A > B` かつ `B = C` ならば `A > C` のような**推移律**に基づくデータ拡張も実装（最終アンサンブルには含めず）。
    * **CV戦略:** 不明（記述なし）。
    * **学習:** 10 Foldで学習。
    * **後処理:** 予測値に係数 (1.2) を乗算（Public LBで最適化）。
    * **その他:** Feature Engineeringの効率化のためPolarsを多用。VS Code拡張機能の開発デモとして参加。

**[9位](https://www.kaggle.com/competitions/um-game-playing-strength-of-mcts-variants/discussion/549624)**

* **アプローチ:** DeBERTa-v3-large, DeBERTa-v2-xlarge (LoRA) モデルのアンサンブル。文字レベルでのトークナイザ差分吸収。Mixtralによるデータセット改良。
* **アーキテクチャ:** DeBERTa-v3-large (通常版、LayerDrop+MultiSample Dropout版)、DeBERTa-v2-xlarge (LoRA)。
* **アルゴリズム:** Token Classification? (MCTSコンペのファイルと混同している可能性。これはPIIコンペの解法詳細)。
* **テクニック:** (MCTSコンペのファイルではないため、省略)

**[10位](https://www.kaggle.com/competitions/um-game-playing-strength-of-mcts-variants/discussion/549605)**

* **アプローチ:** CatBoostモデルのアンサンブル。Flip Augmentation。AdvantageP1のビニング。
* **アーキテクチャ:** CatBoost。
* **アルゴリズム:** CatBoost (回帰)。RMSE損失。
* **テクニック:**
    * **データ拡張:** **Flip Augmentation**（学習とTTA）。
    * **特徴量:** `AdvantageP1`を10分割にビニングした特徴量を追加。
    * **CV戦略:** 不明（記述なし）。
    * **学習:** 10 Foldで学習。
    * **アンサンブル:** 複数のCatBoostモデル（パラメータやシード違い？）のアンサンブル。
    * **後処理:** 予測値に係数 (1.25 or 1.3) を乗算（予測値が0に寄る傾向を補正）。
    * **その他:** NNモデルは効果がなかった。
