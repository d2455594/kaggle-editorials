---
tags:
  - Kaggle
  - 物理学
  - GNN
  - SwinTransformer
startdate: 2023-01-20
enddate: 2023-04-20
---
# IceCube - Neutrinos in Deep Ice
[https://www.kaggle.com/competitions/icecube-neutrinos-in-deep-ice](https://www.kaggle.com/competitions/icecube-neutrinos-in-deep-ice)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、南極の氷床下に設置されたニュートリノ検出器「IceCube」で観測されたデータを用いて、検出器に入射した高エネルギーニュートリノの**到来方向（天頂角 zenith と方位角 azimuth）を高精度に推定する**機械学習モデルを開発することです。
* **背景:** IceCube検出器は、広大な氷体積内に配置された多数の光センサー（Digital Optical Modules - DOMs）で構成されています。ニュートリノが氷と相互作用すると、荷電粒子が生成され、これが氷中を移動する際にチェレンコフ光を発します。DOMはこの微弱な光を検出し、その到達時間と光量を記録します。これらの情報から、元のニュートリノのエネルギーや種類、そして特にその飛来方向を再構成することが、ニュートリノ天文学における重要な課題です。
* **課題:** 検出される光のパターンは非常に複雑でスパース（まばら）です。ニュートリノの種類やエネルギー、相互作用の種類によって光のパターンは大きく異なります（例：トラック状、カスケード状）。また、氷の光学的特性（透明度、散乱）の不均一性や、センサー自体のノイズ、宇宙線ミューオンなどの背景事象も考慮する必要があります。センサー群の時空間的なヒットパターンから、元のニュートリノの方向ベクトル (azimuth, zenith) を正確に推定する**回帰タスク**に挑みます。

**データセットの形式 (Dataset Format)**

提供される主なデータは、各ニュートリノイベントにおけるセンサーのヒット情報と、対応する真のニュートリノ到来方向です。

1.  **トレーニングデータ:**
    * `train/batch_*.parquet`: イベントごとのセンサーヒットデータを含むファイル群（多数のバッチに分割されている）。各ファイルには複数のイベントが含まれ、各イベントは多数のセンサーヒット（パルス）から構成されます。
        * 各ヒット情報には通常、センサーID (`sensor_id`)、ヒット時刻 (`time`)、記録された電荷量 (`charge`)、補助情報フラグ (`auxiliary`、Trueはノイズの可能性が高い) などが含まれます。
    * `sensor_geometry.csv`: 各センサーの3次元座標 (`x`, `y`, `z`) や種類などの情報。
    * `train_meta.parquet` (または類似ファイル): 各イベント (`event_id`) に対応する真のニュートリノ到来方向（`azimuth`, `zenith`）やバッチID (`batch_id`) などのメタデータ。
    * その他、氷の特性に関するデータが提供される場合もあります。
2.  **テストデータ:**
    * `test/batch_*.parquet`: トレーニングデータと同様の形式で、テストイベントのセンサーヒットデータが提供されます。
    * 到来方向の真値は提供されません。
3.  **`sample_submission.csv`**:
    * 提出フォーマットのサンプル。`event_id` と、予測された `azimuth` および `zenith` の列を持ちます。

**評価指標 (Evaluation Metric)**

* **指標:** **平均角度誤差 (Mean Angular Error)**
* **計算方法:** 予測されたニュートリノの方向ベクトルと、真の方向ベクトルとの間の角度差（球面上の測地線距離）を各イベントについて計算し、その平均値を取ります。
    * 方向ベクトルは、天頂角 (zenith, θ) と方位角 (azimuth, φ) から3次元デカルト座標 (x, y, z) に変換されます:
        * x = sin(θ)cos(φ)
        * y = sin(θ)sin(φ)
        * z = cos(θ)
    * 2つの単位ベクトル **a** と **b** の間の角度誤差 Angle( **a**, **b** ) は、内積を用いて `arccos(a · b)` で計算されます。
    * 最終的なスコア = (1 / N) * Σ Angle( **predicted\_vector**\_j, **true\_vector**\_j) （j は各イベント、N はイベント総数）
* **意味:** モデルが予測したニュートリノの到来方向が、真の方向から平均してどれだけずれているかを示します。角度誤差なので、単位はラジアンまたは度になります。スコアは**低い**ほど、モデルの予測精度が高いことを示します。

要約すると、このコンペティションは、IceCube検出器のセンサーヒットデータ（時空間点群データ）からニュートリノの到来方向（azimuth, zenith）を推定する回帰タスクです。データはイベントごとのセンサーヒット情報と真の方向ラベルで構成され、性能は予測方向と真の方向との平均角度誤差（低いほど良い）によって評価されます。

---

**全体的な傾向**

このコンペティションでは、センサーヒットのスパースな時空間データを効果的に処理し、ニュートリノの方向を推定することが求められました。上位解法では主に以下のアーキテクチャが活躍しました。

1.  **Graph Neural Networks (GNN):** DOMヒットをノード、ヒット間の近接性（空間的、時間的）をエッジとしてグラフを構築し、GNN（特に**DynEdge**, **EdgeConv**, **GravNet**, **GPS**など）を用いて特徴抽出を行うアプローチ。ベースラインとして提供され、多くのチームが改良を加えました。
2.  **Transformers:** DOMヒットをトークンのシーケンスとみなし、**自己注意機構 (Self-Attention)** を用いてパルス間の大域的な関係性を捉えるアプローチ。特に高性能を示しました。
3.  **Recurrent Neural Networks (RNN):** ヒットを時間順にソートし、**LSTM**や**GRU**で処理するアプローチも有効でした。
4.  **ハイブリッド/アンサンブル:** 上記のアーキテクチャを組み合わせたり（例: GNN+Transformer）、複数の異なるモデルの予測を統合（アンサンブル、スタッキング、ブレンディング）したりすることが、最終的なスコア向上に不可欠でした。

**データと特徴量の扱い**も重要な要素でした。

* **入力特徴量:** 基本的な (x, y, z, t, charge, auxiliary) に加えて、センサー特性（量子効率 QE）、氷の光学特性（散乱長、吸収長）、ヒット間の相対情報（距離、時間差、相対時空間隔 ds^2）、パルスのランクなどが特徴量として検討・利用されました。
* **データローディング/バッチング:** データセットが巨大なため、効率的なデータローディング（バッチごとの読み込み、キャッシュ）や、計算効率を高めるための**シーケンスバケッティング**（似た長さのイベントをまとめてバッチ化）が重要でした。
* **入力エンコーディング:** Transformerで連続値を扱うため、**Fourier Encoding** や線形層による埋め込みが用いられました。

**損失関数と学習戦略**も多様でした。

* **損失関数:** 方向ベクトル (3D) の回帰には **von Mises-Fisher (vMF) Loss** が広く使われました。評価指標である角度誤差を直接または間接的に組み込んだ損失関数も有効でした。角度空間をビン分割して**分類問題**として解き、Cross Entropy Loss（特に角度的な近さを考慮したラベルスムージング付き）を用いるアプローチもありました。
* **学習:** AdamWなどのオプティマイザ、Cosine Annealing スケジューラ、**SWA (Stochastic Weight Averaging)** がよく使われました。複数段階の学習（例: vMF Lossで初期学習し、角度誤差ベースのLossでファインチューニング）も行われました。
* **アンサンブル/スタッキング:** 単純なベクトル平均（vMFの場合）や重み付き平均に加え、XGBoostやMLP/LSTMなどの**メタモデル**を用いたスタッキング/ブレンディングが効果を発揮しました。予測角度に基づいてビン分割し、ビンごとに最適なブレンド比率を求める手法も用いられました。

**その他:**

* **0時刻問題:** シミュレーションデータの0時刻基準がリークとなりうることが指摘され、最初のヒット時刻を基準にするなどの検討が行われました。
* **TTA:** 検出器の回転対称性を利用したTTAが試されました。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/icecube-neutrinos-in-deep-ice/discussion/402976)**

* **アプローチ:** EdgeConv (GNN) と Transformer を組み合わせたハイブリッドモデル。スタッキングによるアンサンブル。
* **アーキテクチャ:** EdgeConvブロック + Transformerブロック。EdgeConvのエッジ選択は静的に入力時に計算。EdgeConvの更新式を電荷量やauxiliaryを考慮するように修正。
* **アルゴリズム:** 角度誤差とvMF Lossを組み合わせたカスタム損失関数 (`-θ - κ*cos(θ) + C(κ)`)。3層MLPによるスタッキング。
* **テクニック:**
    * **データ処理:** シーケンスバケッティング（DataLoaderのcollate\_fnで実装）。バッチごとのデータ読み込み。
    * **推論:** 長いシーケンス長 (6000) を使用。
    * **スタッキング:** ベースモデルの予測を入力として3層MLPで最終予測。

**[2位](https://www.kaggle.com/competitions/icecube-neutrinos-in-deep-ice/discussion/402882)**

* **アプローチ:** Transformerベースのソリューション。相対時空間隔バイアスが鍵。
* **アーキテクチャ:** BEiTビルディングブロックを用いたTransformer。一部モデルはGraphNetエンコーダーも使用。CLSトークンを使用。
* **アルゴリズム:** von Mises-Fisher Loss + 角度誤差損失 (後半エポック)。重み付きベクトル平均によるアンサンブル。
* **テクニック:**
    * **データ処理:** チャンクベースのデータローディングとキャッシング。長さマッチングサンプリング（シーケンスバケッティング）。auxiliary=True/False両方のパルスを使用し、長いイベントではランダムサンプリング。
    * **特徴エンコーディング:** Fourier Encoding（連続値入力用）。
    * **相対時空間隔バイアス:** ヒット間の相対時空間隔 `ds^2 = c^2 dt^2 - dx^2 - dy^2 - dz^2` を計算し、そのFourier EncodingをTransformerのAttentionバイアスとして導入。
    * **学習:** AdamW、Cosine Annealing + Warmup。SWA。
    * **推論:** 長いシーケンス長 (768) を使用。

**[3位](https://www.kaggle.com/competitions/icecube-neutrinos-in-deep-ice/discussion/402888)**

* **アプローチ:** 自己注意 (Self-Attention) モデルをベースとし、角度を分類問題として解く。vMFモデルとのアンサンブルにXGBoostを使用。
* **アーキテクチャ:** 自己注意モデル (NanoGPTベース)。MLP (vMF Loss用)。XGBoost (アンサンブル用)。
* **アルゴリズム:** カスタムCross Entropy Loss (ガウシアンカーネルで畳み込んだターゲットベクトルを使用)。vMF Loss (スタックモデル用)。HistGradientBoostingClassifier/XGBoost (アンサンブル用)。
* **テクニック:**
    * **データ処理:** シーケンス長<=256でサンプリング (auxiliary=False優先)。シーケンス長によるバッチグルーピング（高速化）。バッチごとのデータ読み込み。
    * **学習:** FP16 + FlashAttention (初期)、FP32 (後期)。より多くのデータで学習 (650バッチ)。
    * **損失関数:** 角度ビン分類 + 角度的近傍を考慮したラベルスムージング。
    * **アンサンブル:**
        * ベースモデル（分類）とスタックモデル（vMF）の予測の差が大きい場合はスタックモデルを優先するルールベース投票。
        * 両モデルの予測、信頼度、イベント長などを特徴量としてXGBoostを学習し、最終的な角度を予測。
        * さらに別のブースティング分類器で「悪い予測」を判定し角度を反転。
        * 複数モデル/チェックポイントの結果をブースティング分類器と投票で統合。

**[5位](https://www.kaggle.com/competitions/icecube-neutrinos-in-deep-ice/discussion/403398)**

* **アプローチ:** Transformer（分類）とGraphNet（回帰）の2つの異なるモデルを開発し、LSTMを用いたメタモデルでどちらの予測を採用するかを動的に選択するスタッキングアンサンブル。
* **アーキテクチャ:**
    * Transformer: 標準的なエンコーダー (8層、512次元)。2つの分類ヘッド (azimuth/zenith、各256ビン)。
    * GraphNet: DynEdgeベース。BatchNormをLayerNormに変更。KNN特徴量数を増加。
    * スタッキング: Bi-LSTM (3層)。Transformer/GraphNetの埋め込みを初期隠れ状態とし、パルスシーケンスを入力。
* **アルゴリズム:** Cross Entropy Loss (Transformer、角度的近傍を考慮したラベルスムージング付き)。vMF Loss (GraphNet)。Binary Cross Entropy (LSTMセレクタ)。
* **テクニック:**
    * **Transformer特徴:** (x, y, z, t, charge, auxiliary, is_core, rank) を使用。連続値は線形層、離散値は埋め込みで処理。
    * **GraphNet特徴:** ベースライン+追加特徴量 (ice特性など)。センサー1229を除外するフラグ。
    * **学習 (Transformer):** TPU、Cosine Schedule (短周期)。
    * **学習 (GraphNet):** AdaBelief、段階的LR減衰。
    * **スタッキング:** Transformer/GraphNetの埋め込みと予測を入力とし、LSTMでどちらのモデルの予測を採用するかを分類。

**[6位](https://www.kaggle.com/competitions/icecube-neutrinos-in-deep-ice/discussion/403153)**

* **アプローチ:** GNN、Transformer、LSTM、SAKT (LSTM+CNN+Transformer) の多様なモデルのアンサンブル。イベント特性に基づいたグループアンサンブル。
* **アーキテクチャ:** DynEdge (GNN)、Transformer (詳細不明)、LSTM (4層GRU/LSTM)、SAKT (LSTM+CNN+Transformer)。
* **アルゴリズム:** カスタムCross Entropy Loss (LSTM/SAKT、近傍ロス加算付き、48x48ビン)。vMF Loss (GNN/Transformer)。グループ別重み付きアンサンブル。
* **テクニック:**
    * **LSTM/SAKT損失:** Cross Entropyに近傍ビンの誤差も加算するカスタムロス。
    * **学習:** 複数ステップの学習スケジュール（LR変更、パルス数変更）。
    * **グループアンサンブル:** Line-fitによる予測やパルス数、センサー配置（垂直/複数列）に基づきイベントを6～13グループに分割。各グループでモデルの性能が異なるため、グループごとに最適なアンサンブル重みを適用。

**[8位](https://www.kaggle.com/competitions/icecube-neutrinos-in-deep-ice/discussion/403713)**

* **アプローチ:** GNN (DynEdge, GPS, GravNet) とLSTMのモデルのアンサンブル。特徴量エンジニアリングと洗練されたブレンディング手法。
* **アーキテクチャ:** DynEdge, GPS (Transformerベース), GravNet, LSTM/GRU (4層)。
* **アルゴリズム:** vMF Loss + (1 - CosineSimilarity) (GNN)。Sparse Categorical Cross Entropy / Categorical Cross Entropy + Label Smoothing (LSTM)。Decision Tree Regressorによるビン分割 + 重み最適化 (ブレンディング)。
* **テクニック:**
    * **データ処理:** イベントデータを`.pt`ファイルとして事前保存（ただしinode数に注意）。
    * **特徴量 (GNN):** 11特徴量 (座標、時間、電荷、QE、Aux、散乱長、前ヒットとの距離/時間差、散乱フラグ)。
    * **学習 (GNN):** AdamW、Cosine Schedule。
    * **学習 (LSTM):** 3段階学習 (高LR→低LR→ラベルスムージング付き低LR)。SWA。
    * **Augmentation (GNN):** z軸周り60度回転。
    * **TTA (GNN):** 6x回転TTA。
    * **ブレンディング:**
        * Alvorの手法: 予測zenithに基づいて10x10ビンを作成し、ビンごとに線形結合の重みを最適化。
        * Wojtekの手法: 複数モデルの予測や特徴量を入力としてDecision Tree Regressorを学習し、その予測値に基づいてビンを作成。ビンごとに線形結合の重みを最適化。

**[9位](https://www.kaggle.com/competitions/icecube-neutrinos-in-deep-ice/discussion/402849)**

* **アプローチ:** 複数のDynEdge (GNN) モデルのバリアントを作成し、MLPによるスタッキングとセグメント別ブレンディングを組み合わせる。
* **アーキテクチャ:** DynEdge (Vanilla, Bigman, DynEmb の3種)。MLP (スタッキング用)。
* **アルゴリズム:** vMF Loss (初期学習)。Euclidean Distance Loss (ファインチューニング)。MLPによるスタッキング。セグメント別ブレンディング。
* **テクニック:**
    * **モデルバリアント:**
        * Vanilla: KNN近傍数変更、Pooling追加、QHAdamオプティマイザ。
        * Bigman: 層数・幅増加、SiLU活性化関数、BatchNorm。
        * DynEmb: センサー座標に基づく埋め込みを追加、Weight Normalization、Lionオプティマイザ、FP16。
    * **学習:** 2段階学習 (vMF Loss, 長シーケンス → Euclidean Distance Loss, 短シーケンス)。SWA。
    * **スタッキング:** 6つのモデル (各バリアント2種) の予測をMLPに入力して最終予測。
    * **セグメント別ブレンディング:** パルス数でイベントを10分割し、サブセットで学習した4モデルをセグメントごとにブレンド。
    * **最終提出:** スタックモデルとセグメント別ブレンドモデルを60/40でブレンド。

**[10位](https://www.kaggle.com/competitions/icecube-neutrinos-in-deep-ice/discussion/402969)**

* **アプローチ:** 複数のLSTMベースモデルのアンサンブル。Nelder-Mead法による重み最適化。
* **アーキテクチャ:** GRU (3層 BiGRU)、LSTM (3層 BiLSTM、埋め込み層あり/なしのバリエーション)。
* **アルゴリズム:** Cross Entropy Loss (ビン分類)、von Mises-Fisher 3D Loss (方向回帰)。Hill Climbing / Nelder-Mead (アンサンブル重み最適化)。
* **テクニック:**
    * **入力特徴量:** 9特徴量 (座標, t, charge, auxiliary, is\_main\_sensor, is\_deep\_veto, is\_deep\_core)。
    * **出力:** 31x31のビン分類、または3D方向ベクトル+κ。
    * **データ処理:** パディングされたシーケンスを`pack_padded_sequence`で処理。シーケンス長<=128。
    * **学習:** Adam、Cosine Annealing LR + Warmup。
    * **アンサンブル:** 4種類のアーキテクチャを複数回学習させ、計8モデルをNelder-Meadで重み最適化してブレンド。
    * **使用ライブラリ:** PyTorch Lightning (TensorFlowより良好だった)。
