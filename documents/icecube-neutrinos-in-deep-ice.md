---
tags:
  - Kaggle
startdate: 2023-01-20
enddate: 2023-04-20
---
# IceCube - Neutrinos in Deep Ice
[https://www.kaggle.com/competitions/icecube-neutrinos-in-deep-ice](https://www.kaggle.com/competitions/icecube-neutrinos-in-deep-ice)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、南極の氷床下に設置された巨大な **IceCube ニュートリノ検出器**で観測されたデータを用いて、検出器に入射した**高エネルギーニュートリノの到来方向（天頂角 zenith と方位角 azimuth）**を高精度に推定するモデルを開発することです。
* **背景:** ニュートリノは宇宙を構成する基本的な粒子の一つですが、他の物質とほとんど相互作用しないため観測が非常に困難です。IceCube は、氷中でニュートリノが稀に起こす相互作用によって生じる二次粒子（主にミューオン）が発するチェレンコフ光を、多数の光検出器（DOM: Digital Optical Module）で捉えます。この光のパターンから元のニュートリノの情報を再構成することで、高エネルギー宇宙現象の解明を目指します。特に、ニュートリノの到来方向を正確に知ることは、その発生源（超新星爆発、活動銀河核など）を特定するために不可欠です。
* **課題:**
    * **膨大な背景ノイズ:** 宇宙線が大気と衝突して生成される大気ミューオンも同様の光信号を生成し、これがニュートリノ信号に対して圧倒的な背景ノイズとなります。
    * **スパースで不規則なデータ:** 検出器は広大な領域に分散しており、イベントごとに光を捉えるセンサーの数や位置、タイミングは大きく異なります。データは本質的にスパースな点群時系列データです。
    * **氷の不均一性:** 氷の光学的特性（透明度、散乱）は深さによって異なり、光の伝播に影響を与えます。
    * **角度予測の精度:** 天文学的な要求に応えるためには、非常に高い角度分解能（誤差1度未満）が求められます。球面上の角度を正確に予測する必要があります。
    * **大規模データ:** トレーニングデータは非常に大規模（数TB）であり、効率的なデータ処理と学習が必要です。

**データセットの形式 (Dataset Format)**

提供される主なデータは、各ニュートリノ候補イベントにおける光検出器の応答記録と、対応する真のニュートリノ到来方向です。

1.  **センサー形状データ (`sensor_geometry.csv`):**
    * 各光検出器（DOM）のID (`sensor_id`) とその3次元座標 (`x`, `y`, `z`) を含みます。
2.  **メタデータ (`train_meta.parquet`, `test_meta.parquet`):**
    * 各イベント (`event_id`) のメタ情報。
    * `batch_id`: イベントが含まれるバッチファイル番号。
    * `event_id`: イベントのユニークID。
    * `first_pulse_index`, `last_pulse_index`: 各イベントが `batch_XXX.parquet` ファイル内で占める行の範囲を示すインデックス。
    * `azimuth`, `zenith`: **ターゲット変数**となる、真のニュートリノ到来方向（ラジアン）。Azimuthは[0, 2π]、Zenithは[0, π]。トレーニングデータにのみ含まれます。
3.  **イベントデータ (`train/batch_XXX.parquet`, `test/batch_XXX.parquet`):**
    * 実際のセンサー応答データ。多数のバッチファイルに分割されています。
    * 各行が一つの光パルス検出に対応します。
    * `event_id`: 対応するイベントID。
    * `sensor_id`: パルスを検出したセンサーのID。
    * `time`: パルスの検出時刻 (ナノ秒)。基準時刻はイベントごとに異なる可能性があります。
    * `charge`: 検出された光量（光電子増倍管の電荷量）。
    * `auxiliary`: Trueの場合、ノイズパルスである可能性が高いことを示すフラグ。
    * **これらの `time`, `charge`, `auxiliary` と `sensor_geometry.csv` の座標 (`x`, `y`, `z`) がモデルの主要な入力特徴量となります。**
4.  **`sample_submission.csv`:**
    * 提出フォーマットのサンプル。`event_id`, `azimuth`, `zenith` の列を持ちます。

**評価指標 (Evaluation Metric)**

* **指標:** **平均角度誤差 (Mean Angular Error)**
* **計算方法:**
    1.  各イベントについて、予測された方向ベクトル (azimuth\_pred, zenith\_pred から計算) と真の方向ベクトル (azimuth\_true, zenith\_true から計算) の間の球面上の角度差 (arc cosine of the dot product) を計算します。
        Angle Error = arccos( (x<sub>pred</sub> * x<sub>true</sub>) + (y<sub>pred</sub> * y<sub>true</sub>) + (z<sub>pred</sub> * z<sub>true</sub>) )
        ここで、(x, y, z) は方向ベクトル（単位ベクトル）です。
        x = sin(zenith) * cos(azimuth)
        y = sin(zenith) * sin(azimuth)
        z = cos(zenith)
    2.  全てのテストイベントに対する角度誤差を平均して最終スコアとします。
* **意味:** 予測されたニュートリノ到来方向と真の方向との平均的なずれ（角度、ラジアン単位）を示します。スコアは **0に近い** ほど性能が良いことを示します。

要約すると、このコンペティションは、IceCube検出器のスパースな時空間ヒットパターンからニュートリノの到来方向（2つの角度）を高精度に回帰予測するタスクです。データはセンサー形状、イベントメタデータ、および多数のバッチファイルに分割されたセンサー応答ログで構成され、性能は予測方向と真の方向の平均角度誤差（小さいほど良い）によって評価されます。

---

**全体的な傾向**

IceCubeニュートリノ検出器のデータからニュートリノの到来方向を予測するこのタスクでは、センサーヒットのスパースな3D+時間情報を効果的に処理する必要がありました。上位解法では、**Graph Neural Networks (GNN)** と **Transformer** が主要なアーキテクチャとして採用され、一部では **LSTM/GRU** も有効でした。これらのモデルを単独で、あるいは組み合わせて使用し、最終的には複数のモデル予測を**アンサンブル**または**スタッキング**するアプローチが一般的でした。

**GNN** では、特に **DynEdge** (動的にエッジを構築するEdgeConv) や **GravNet**、**GPS (Graph Positioning System)** などが用いられました。入力特徴量として、センサーのXYZ座標、時間、電荷、auxiliaryフラグに加え、氷の特性（量子効率 QE、散乱長）や近傍ヒット情報（距離、時間差）などが工夫されました。エッジ選択の戦略（動的 vs 静的、KNNのパラメータ）や活性化関数 (GELU, SiLU)、正規化層 (BatchNorm vs LayerNorm) も検討されました。

**Transformer** は、センサーヒットのシーケンスを入力として、大域的な関連性を捉える能力が期待されました。入力特徴量のエンコーディング（**Fourier Encoding**, 線形層）、シーケンス長の問題（**シーケンス長バケッティング/パッキング**、長シーケンス推論）、Attention機構への工夫（**相対時空間隔バイアス**、FlashAttention）などが重要なテクニックとなりました。

**LSTM/GRU** は、時系列情報を処理する古典的なアプローチとして、単独または他のモデルとの組み合わせで用いられました。

**損失関数**は、角度予測タスク特有のものが多く試されました。**von Mises-Fisher (vMF) Loss**（3D方向ベクトルとその集中度κを予測）や、評価指標である**角度誤差 (Mean Angular Error)** そのもの、あるいは角度をビンに分割して分類問題として扱う**Cross Entropy Loss**（カスタム平滑化を含む）、**Cosine Similarity Loss** などが用いられ、これらを組み合わせることもありました。

**データ処理**も重要で、大規模データを効率的に扱うための**バッチごとのロード**や**キャッシュ戦略**、メモリ使用量を削減しつつ長いシーケンスを扱うための**シーケンス長マッチング/パッキング**が効果的でした。パルス選択（`auxiliary=False`優先、ランダムサンプリングなど）も精度に影響しました。

**アンサンブル/スタッキング**では、単純な加重平均のほか、予測角度によって重みを変えるビンベースのブレンディング、XGBoostやMLP、LSTMを用いたメタモデルによるスタッキングなど、高度な手法が用いられました。**Test Time Augmentation (TTA)** として、検出器の対称性を利用したZ軸周りの回転などが有効でした。

興味深い点として、シミュレーションデータの**基準時刻（time=0）が物理的な意味を持ち、モデルがそれをリークとして利用している**可能性が指摘されました。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/icecube-neutrinos-in-deep-ice/discussion/402976)**

* **アプローチ:** EdgeConv + Transformer の組み合わせ。静的エッジ選択。カスタム損失関数。MLPによるスタッキング。
* **アーキテクチャ:** EdgeConv層 + Transformer Encoder層。
* **アルゴリズム:** カスタム損失 (-θ - κ\*cos(θ) + C(κ))。vMF Lossも一部利用。
* **テクニック:**
    * **GNN:** EdgeConv modificaiton (絶対値情報 `xj` も使用)。入力に基づく静的エッジ選択。KNNパラメータ (x,y,z or x,y,z,t)。
    * **Transformer:** (詳細不明だが、標準的なEncoder構造と推測)。
    * **データ処理:** シーケンス長バケッティング (`collate_fn` で実装)。バッチごとのデータロード。
    * **スタッキング:** ベースモデルの出力を入力として3層MLPで最終予測。

**[2位](https://www.kaggle.com/competitions/icecube-neutrinos-in-deep-ice/discussion/402882)**

* **アプローチ:** Transformerベース。Fourier Encoding。相対時空間隔バイアス。シーケンス長マッチング。
* **アーキテクチャ:** Transformer Encoder (BEiT風ブロック、一部カスタム相対バイアス付き)。CLSトークン使用。
* **アルゴリズム:** von Mises-Fisher Loss + 角度誤差 Loss (後半)。AdamW (Cosine Annealing)。SWA。
* **テクニック:**
    * **データ処理:** チャンクベースデータロード + キャッシュ + **シーケンス長マッチング** (重要)。パルス選択 (aux=False優先、ランダムサンプリング)。
    * **入力エンコーディング:** **Fourier Encoding** (入力スケーリング係数=1024-4096が重要)。離散特徴はEmbedding。
    * **Attentionバイアス:** **相対時空間隔 (ds^2)** をFourier EncodingしてAttentionバイアスとして導入 (重要)。
    * **推論:** 長いシーケンス長 (768) で推論。
    * **アンサンブル:** 6モデルの出力ベクトルを加重平均 (vMFのκを考慮)。

**[3位](https://www.kaggle.com/competitions/icecube-neutrinos-in-deep-ice/discussion/402888)**

* **アプローチ:** Self-Attentionモデル (NanoGPTベース)。角度をビン分類。vMF Lossモデルとのスタッキング (XGBoost/HistGradientBoosting)。
* **アーキテクチャ:** Self-Attention (Transformer Encoder)。MLP (スタッキング用)。XGBoost/HistGradientBoosting (スタッキング用)。
* **アルゴリズム:** カスタムCross Entropy (ガウシアンカーネルで平滑化)。vMF Loss (スタッキング用MLP)。
* **テクニック:**
    * **入力:** パルスデータ (+透明度情報)。シーケンス長256にサンプリング (aux=False優先)。
    * **出力:** AzimuthとZenithそれぞれ128ビンに分類。
    * **学習:** シーケンス長パッキング。FlashAttention (FP16)。FP32に切り替え (安定性のため)。
    * **スタッキング:** Attentionモデル (分類) とMLPモデル (vMF) の予測を入力とし、XGBoost/HGBTで最終的な角度補正/選択を行う。入力にはモデルの信頼度指標 (確率/κ)、イベント長、Z座標なども使用。

**[5位](https://www.kaggle.com/competitions/icecube-neutrinos-in-deep-ice/discussion/403398)**

* **アプローチ:** TransformerとGraphNetの出力をLSTMメタモデルでスタッキング (どちらのモデル予測を採用するか選択)。
* **アーキテクチャ:** Transformer Encoder (8層)。GraphNet (DynEdgeベース、改良版)。Bi-LSTM (3層、スタッキング用)。
* **アルゴリズム:** Transformer: Cross Entropy (256ビンx2、カスタム平滑化)。GraphNet: vMF Loss or ユークリッド距離Loss。LSTM: 分類損失 (どちらが良いか)。
* **テクニック:**
    * **Transformer特徴量:** (x,y,z,t,charge,auxiliary,is_core,rank)。カテゴリ特徴はEmbedding、連続値は線形層。
    * **GraphNet特徴量:** (x,y,z,t,charge,auxiliary,QE,wrong_charge_dom,ice_absorption,ice_scatter)。kNN特徴量数を増加 (48)。LayerNorm化。
    * **スタッキング:** TransformerとGraphNetの最終層手前の埋め込みをLSTMの初期隠れ状態として入力。パルス列の一部もLSTMに入力。LSTMは2つのモデルのうちどちらの予測がより良いかを予測する。

**[6位](https://www.kaggle.com/competitions/icecube-neutrinos-in-deep-ice/discussion/403153)**

* **アプローチ:** GNN, Transformer, LSTM, SAKT (LSTM+CNN+Transformer) のアンサンブル。イベントのグループ別アンサンブル。
* **アーキテクチャ:** GNN (DynEdgeベース、改良版)。Transformer。LSTM。SAKT。
* **アルゴリズム:** LSTM/SAKT: Cross Entropy (48x48 or 48ビン、カスタム近傍損失)。GNN/Transformer: vMF Loss系。AdamW (Cosine Annealing)。
* **テクニック:**
    * **LSTMカスタム損失:** 正解ビンの周囲にも損失を与える重み付け。
    * **学習:** 複数ステップでLR変更、パルス数変更。
    * **グループアンサンブル:** イベントをパルス数やLineFitのZenith予測などに基づいて6グループ (後に13グループ) に分類し、グループごとに最適なモデル重みでアンサンブル。

**[8位](https://www.kaggle.com/competitions/icecube-neutrinos-in-deep-ice/discussion/403713)**

* **アプローチ:** GNN (DynEdge, GPS, GravNet) とLSTMのアンサンブル。高度なブレンディング。
* **アーキテクチャ:** DynEdge, GPS, GravNet。LSTM (複数層)。Decision Tree Regressor (ブレンディング用)。
* **アルゴリズム:** GNN: vMF Loss + (1 - CosineSimilarity)。LSTM: Cross Entropy (24ビンx2、ラベル平滑化)。AdamW (GNN), Adam (LSTM)。段階的LRスケジュール。
* **テクニック:**
    * **データ前処理:** イベントごとにptファイルとして保存。パルス数で層化分割。
    * **GNN特徴量:** 11特徴量 (座標, 時間, 電荷, QE, Aux, 散乱長, 前ヒット距離/時間差, 散乱フラグ)。パルス数256にサンプリング。
    * **GNN学習:** GELU活性化。LayerNorm。
    * **LSTM特徴量:** + 氷透明度/吸収。パルス選択 (non_aux優先+auxランダム)。
    * **TTA (GNN):** Z軸周り60度回転 x6。
    * **ブレンディング:** Decision Treeを用いて予測角度やパルス数などでBinを作成し、各Binで最適な線形結合の重みを学習。

**[9位](https://www.kaggle.com/competitions/icecube-neutrinos-in-deep-ice/discussion/402849)**

* **アプローチ:** 複数のGNN (DynEdgeベース) のアンサンブル + MLPスタッキング。代替アプローチとしてパルス数によるセグメント別モデル+ブレンド。
* **アーキテクチャ:** DynEdgeベース GNN (Vanilla, Bigman, DynEmb)。MLP (スタッキング用)。
* **アルゴリズム:** vMF Loss (1段階目) -> ユークリッド距離 Loss (2段階目)。QHAdam, Lion Optimizer。
* **テクニック:**
    * **GNNバリアント:** 層数/幅変更、SiLU活性化、BatchNorm、センサー埋め込み(DynEmb)、Weight Norm。
    * **学習:** 長期学習 + 短期Fine-tuning (ユークリッド距離Loss、短パルス数)。SWA。
    * **スタッキング:** 6つのGNNモデルの予測を入力としてMLPで学習。
    * **セグメント別:** パルス数で10ビンに分割、組み合わせで4モデル学習、セグメントごとにブレンド。
    * **アンサンブル:** スタッキングモデルとセグメント別ブレンドモデルを60:40で最終ブレンド。

**[10位](https://www.kaggle.com/competitions/icecube-neutrinos-in-deep-ice/discussion/402969)**

* **アプローチ:** 複数LSTM/GRUモデルのアンサンブル。角度ビン分類と方向ベクトル回帰の両方を試行。
* **アーキテクチャ:** GRU (3層、分類用)。LSTM (3層、回帰用)。Embedding層付きLSTM (3層、回帰用)。
* **アルゴリズム:** Cross Entropy (31x31ビン、分類用)。vMF Loss (回帰用)。Adam Optimizer (Cosine Annealing)。
* **テクニック:**
    * **入力特徴量:** 9特徴量 (座標, 時間, 電荷, auxiliary, is_main_sensor, is_deep_veto, is_deep_core)。
    * **データ処理:** シーケンスパッキング (`pack_padded_sequence`)。
    * **学習:** バッチサイズ2048、最大パルス数128。
    * **アンサンブル:** 8モデル (複数アーキテクチャ x 複数学習試行) の予測をHill Climbing / Nelder-Meadで重み最適化。


