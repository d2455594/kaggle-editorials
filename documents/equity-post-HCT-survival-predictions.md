---
tags:
  - Kaggle
startdate: 2024-12-05
enddate: 2025-03-06
---
# CIBMTR - Equity in post-HCT Survival Predictions
[https://www.kaggle.com/competitions/equity-post-HCT-survival-predictions](https://www.kaggle.com/competitions/equity-post-HCT-survival-predictions)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、同種造血細胞移植 (allogeneic Hematopoietic Cell Transplantation, HCT) を受けた患者の移植後のイベントフリー生存 (Event-Free Survival, EFS) を予測する機械学習モデルを開発することです。特に重要なのは、人種や民族グループ間で**公平 (Equitable)** な予測、つまり特定グループに偏らない予測精度を持つモデルを構築することです。
* **背景:** CIBMTR (Center for International Blood and Marrow Transplant Research) は、HCTおよび細胞療法に関するデータを収集・分析する国際的な研究組織です。HCTは多くの生命を脅かす疾患に対する治療法ですが、治療成績は人種・民族的背景を含む様々な要因によって影響を受ける可能性があります。AI予測モデルが臨床現場で責任を持って使用されるためには、異なる人口グループ間で公平な性能を発揮することが不可欠です。なお、本コンペティションで使用されたデータは、プライバシー保護のため **SurvivalGAN** アルゴリズムによって生成された**合成データ**です。
* **課題:**
    * **生存時間分析と打ち切り:** イベント発生有無 (`efs`) とイベント発生までの時間 (`efs_time`) の両方を考慮し、患者のリスクを順位付けする必要があります。イベントが発生しなかった（生存または追跡不能）打ち切りデータ (`efs=0`) の扱いが重要です。
    * **公平性の担保:** モデルの予測が特定の人種・民族グループ (`recipient_race_group`) に偏らず、グループ内で公平なランキング精度を達成することが求められます。これは評価指標（Stratified Concordance Index）によって直接評価されます。
    * **合成データの特性理解:** 実データではなく、SurvivalGANによって生成された合成データであるという特性を考慮したモデリングが必要です。
    * **評価指標の特殊性:** Stratified Concordance Index は、グループ内でのペア比較のみを行うため、通常のC-indexとは異なる挙動を示します。グループ間の予測値の絶対的な差は評価されません。

**データセットの形式 (Dataset Format)**

提供されるデータは、HCT患者の合成臨床データとイベント情報です。

1.  **トレーニングデータ (`train.csv`):**
    * `PatientID`: 患者の一意なID。
    * `efs_time`: イベント発生または最終フォローアップまでの日数（合成値）。
    * `efs`: イベント発生フラグ (0: 打ち切り/生存, 1: イベント発生)。
    * `donor_age_at_hct`, `recipient_age_at_hct`, `recipient_gender`, `recipient_race_group`, `donor_recipient_sex_match`, `disease_group`, `hla_match` など、多数の患者、ドナー、疾患、移植に関連する特徴量（数値およびカテゴリ）。HLA関連の特徴が多い。
2.  **テストデータ (`test.csv`):**
    * `PatientID` と訓練データと同様の特徴量が含まれます。`efs_time` と `efs` は含まれません。
3.  **`sample_submission.csv`**:
    * 提出フォーマットのサンプル。
        * `PatientID`: 患者ID。
        * `risk_score`: **ターゲット変数**。モデルによって予測されたリスクスコア（連続値、値が大きいほどリスクが高い）。評価指標はこのスコアの順序に基づいて計算されます。

**評価指標 (Evaluation Metric)**

* **指標:** **Stratified Concordance Index (層化コンコーダンス指数)**
* **計算方法:**
    * 通常の Concordance Index (C-index) を拡張し、公平性を考慮した指標です。
    * 患者ペア (i, j) の比較は、**同じ人種/民族グループ (`recipient_race_group`) に属する場合にのみ**行われます。
    * 比較可能なペアの中で、イベント発生までの時間が短い患者のリスクスコアが、時間が長い（または打ち切られた）患者のリスクスコアよりも高く予測されている場合に「コンコーダント（一致）」とカウントします。
    * （イベント時間 `T`, リスクスコア `R` として、ペア(i,j)が比較可能かつ `T_i < T_j` のとき `R_i > R_j` ならコンコーダント）
    * 最終的なスコアは、比較可能な全ペアにおけるコンコーダントなペアの割合として計算されます。
* **意味:** この指標は、異なる人種/民族グループ間の予測レベルの違いを無視し、**各グループ内部でのランキング精度**のみを評価します。これにより、特定グループに対するモデルの予測バイアスを評価から排除し、グループ内での公平な予測能力を測ることができます。スコアは 0.5 (ランダム) から 1.0 (完璧な順序付け) の範囲を取り、**高い**ほど良い性能を示します。

要約すると、このコンペティションは、合成されたHCT患者データを用いて、イベントフリー生存に関するリスクを予測・順序付けするタスクです。特徴は、公平性を重視した Stratified Concordance Index (高いほど良い) が評価指標である点と、データが合成データである点です。

---

**全体的な傾向**

このコンペティションでは、Stratified Concordance Index という特殊な評価指標と合成データという特性に対応するため、多くのチームが共通の戦略を採用しました。その核心は、問題を**2つのサブタスクに分解**することです。

1.  **イベント発生予測 (分類):** 患者がイベントを経験するかどうか (`efs=0` vs `efs=1`) を予測する二値分類タスク。
2.  **イベント時間予測 (回帰/ランク学習):** イベントが発生した場合 (`efs=1`) の生存時間 (`efs_time`) を予測、またはその相対的な順位（ランク）を予測するタスク。

そして、これら2つのモデルの予測値を**後処理で結合 (Merge)** し、最終的な単一の `risk_score` を生成するアプローチが大多数の上位解法で見られました。結合関数の形式 (`P(0)`と`Rank(time|1)`の組み合わせ方) やパラメータのチューニングがスコアを左右する重要な要素となりました。

モデルとしては、**GBDT (LightGBM, XGBoost, CatBoost)** が両方のサブタスクで安定した性能を示し、広く使われました。特に回帰/ランク学習タスクで強みを発揮しました。一方で、**ニューラルネットワーク (MLP, TabNet, TabM)** も、特に分類タスクや特殊な損失関数（ペアワイズランク損失など）を用いる場合に有効でした。ODST (Oblivious Decision Tree) を組み込んだNNや、GNNを試みたチームも存在しました。

ターゲット変数の扱いにも工夫が見られました。回帰/ランク学習タスクでは、`efs_time` を直接予測する代わりに、**ランクパーセンタイルに変換**したり、さらに**逆正規累積分布関数 (Inverse Normal CDF)** で変換して正規分布に近づけたりする手法が有効でした。分類タスクでは、単純な `efs` ラベルだけでなく、特定の時間閾値を用いたり、打ち切りデータをKaplan-Meier推定値で重み付けしたりする試みもありました。

**特徴量エンジニアリング**は比較的限定的で、カテゴリ変数のOne-Hotエンコーディングや、連続変数のビン化/カテゴリ化などが主でした。HLA関連特徴量の合計値を再計算するなどの試みもありましたが、大きな効果は報告されていません。

**交差検証 (CV)** は、Stratified K-Fold（`efs`や`race_group`で層化）が基本でしたが、ランダムシードによるばらつきが大きく、安定した評価を得るために複数シードでの平均を取るなどの工夫が必要でした。

評価指標がグループ内比較のみを行うため、モデル側での明示的な**公平性制約**はあまり見られませんでしたが、ターゲット変換によってグループ間の分布差を暗に調整しようとするアプローチは存在しました。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/equity-post-HCT-survival-predictions/discussion/566550)**

* **アプローチ:** 問題分解 (P(efs=0)分類 + efs_timeランク回帰) + 結合関数。
* **アーキテクチャ:** 分類器: LGBM, XGB, CatBoost, NN, TabM, GNN (KNN+GraphSAGE), RankLoss NN/TabM/GNN。回帰器: LGBM, XGB, CatBoost。
* **アルゴリズム:** GBDT, MLP, TabM, GNN, Rank Loss。
* **テクニック:** 回帰器でefs=0データも重み付き利用。CV: 10 fold StratifiedKFold (efs+race)。特徴量: OHE, 連続→カテゴリコピー。結合関数 `(1-y^b)*x^a+y^b` (x=P(efs=0), y=rank_pred) のパラメータとアンサンブル重みをOptunaで最適化。Rank Lossモデル/GNNの予測値シフト補正。

**[2位](https://www.kaggle.com/competitions/equity-post-HCT-survival-predictions/discussion/566522)**

* **アプローチ:** 問題分解 (efs分類 + efs_time回帰) + 結合 + NNによる再スコアリング。
* **アーキテクチャ:** 分類器: RealMLP, HistGBM, CatBoost。回帰器: XGBoost, HistGBM。再スコアリング: NN (MLP)。
* **アルゴリズム:** GBDT, MLP。
* **テクニック:** 回帰器はefs=1を特徴量として追加、推論時efs=1固定。結合: `P(efs=1) * sigmoid(-reg_pred)`。NNはコンペ指標近似損失+補助分類損失で学習。最終はランクアンサンブル。AutoGluonでも高スコア達成。

**[3位](https://www.kaggle.com/competitions/equity-post-HCT-survival-predictions/discussion/566574)**

* **アプローチ:** 問題分解 (efs分類 + efs=1回帰 + efs=0回帰) + 結合。
* **アーキテクチャ:** 分類器: NN (ODST), TabM。回帰器: CatBoost, LGBM, XGBoost。
* **アルゴリズム:** GBDT, MLP (ODST), TabM。
* **テクニック:** ターゲット変換 (efs=1は[0,1]ランク正規化, efs=0は定数[1.345, 1.355])。回帰タスクにBCE損失使用。結合関数: `P(0)*const + P(1)*pred1`。複数シード平均。外れ値除去(回帰のみ)。分類器をロジスティック回帰でスタッキング。CV: 4 fold x N Seeds。

**[4位](https://www.kaggle.com/competitions/equity-post-HCT-survival-predictions/discussion/566528)**

* **アプローチ:** 問題分解 (P(event)分類 + E[rank% | event=1]ランク回帰) + 結合。
* **アーキテクチャ:** TabM, CatBoost, XGBoost (分類器・回帰器ともに)。
* **アルゴリズム:** GBDT, TabM。
* **テクニック:** 分類器はefs=0をKaplan-Meier推定値で重み付け。回帰器はefs=1データのみ使用、ターゲットは rank(-time)/N を**逆正規CDF変換**。結合関数: `P(0)*(s0/2) + P(1)*(s0 + (1-s0)*E[rank%|1])` (s0はグループ内P(0)率)。重み付き平均でアンサンブル。

**[5位](https://www.kaggle.com/competitions/equity-post-HCT-survival-predictions/discussion/568339)**

* **アプローチ:** 問題分解 (A: P(efs_time<T)分類 + B: E[logit(rank%) | efs=1, time<T]回帰) + 結合 + 他ターゲットモデルとのアンサンブル。
* **アーキテクチャ:** LGBM, CatBoost, TabM, ODST Pairwise NN, AutoGluon (XGBoost, NN含む)。
* **アルゴリズム:** GBDT, TabM, MLP, Pairwise Rank Loss。
* **テクニック:** 独自のA/Bターゲット定義 (特定の時間閾値`T`を使用)。結合関数: `(A * B) - ((1 - A) * S_RATIO)`。NelsonAalenベースの多様なターゲット変換 (+Flat Shift, *Multiplier, SampleWeight Multiplier) を試行し、複数モデルを学習。Cox Lossモデル、ODST Pairwise NNモデルも使用。**Greedy Ensemble Selection**で多数のモデルから最終アンサンブルを構築。予測値(logit)のままでのアンサンブルと後処理(logitの線形変換)。

**[6位](https://www.kaggle.com/competitions/equity-post-HCT-survival-predictions/discussion/566686)**

* **アプローチ:** 問題分解 (P(efs=1)分類 + E[efs_time | efs=1]回帰) + NNスタッキング。
* **アーキテクチャ:** Stage 1: GBDT (LGBM/XGB/CatBoost)。Stage 2: TabM (評価指標近似損失) + 1層NN。
* **アルゴリズム:** GBDT, TabM, MLP。
* **テクニック:** Stage 2 NNへの入力: TabMリスクスコア + Stage 1 GBDT予測の2次多項式特徴量。複数シード平均。CV: 20 fold x 5 seeds。

**[9位](https://www.kaggle.com/competitions/equity-post-HCT-survival-predictions/discussion/566948)**

* **アプローチ:** 問題分解 (efs=1でのランク回帰 + P(efs=1) 分類) + 結合。
* **アーキテクチャ:** LGBM, CatBoost, XGBoost。
* **アルゴリズム:** GBDT。
* **テクニック:** efs=1データのみでランク回帰モデルを学習。最終予測 = `rank_pred * P(efs=1)` というシンプルな結合。時間不足のため詳細分析は限定的。
