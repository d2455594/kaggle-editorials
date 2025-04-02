---
tags:
  - Kaggle
  - 気象学
  - SkillScore
  - Squeezeformer
startdate: 2024-04-19
enddate: 2024-06-16
---
# LEAP - Atmospheric Physics using AI (ClimSim)
[https://www.kaggle.com/competitions/leap-atmospheric-physics-ai-climsim](https://www.kaggle.com/competitions/leap-atmospheric-physics-ai-climsim)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、高忠実度気候モデル (ClimSim) のシミュレーションデータを用いて、大気中の複雑な物理プロセス（放射、雲、降水など）を正確かつ効率的にエミュレート（模倣）する機械学習モデルを開発することです。
* **背景:** 地球の気候変動を予測する気候モデルは、現代社会にとって極めて重要ですが、その計算コストは膨大です。特に、モデルの格子スケールよりも小さな現象（サブグリッドスケールプロセス）、例えば雲の形成や乱流などは、「パラメタリゼーション」と呼ばれる簡略化された物理モデルで表現されており、これが計算時間の大部分を占めるとともに、モデルの不確実性の主要な原因となっています。AIを用いてこれらのパラメタリゼーションを高速かつ忠実に置き換えることができれば、気候モデルの性能向上と高速化に繋がり、より詳細な気候予測が可能になります。
* **課題:** 大気の状態（温度、湿度、風速などの鉛直プロファイル、地表の状態など数百次元）を入力とし、物理プロセスによるそれらの変化量（加熱率、水蒸気変化率などの鉛直プロファイル、数百次元）を出力として予測する、高次元の回帰タスクです。入力と出力の間には複雑な非線形関係と物理的な制約が存在します。非常に大規模なデータセットを効率的に扱い、汎化性能の高いモデルを構築する必要があります。

**データセットの形式 (Dataset Format)**

提供される主なデータは、気候シミュレーションから得られた大気状態の入力変数と、それに対応する物理プロセスの出力変数（ターゲット）です。

1.  **トレーニングデータ & テストデータ:**
    * **データソース:** ClimSimデータセット。通常、E3SMのような気候モデルの高解像度シミュレーション結果から生成されます。Hugging Faceなどで公開されています。
    * **形式:** `.nc` (NetCDF) や `.tfrecord` 形式で提供されることが多いです。データ量が膨大（テラバイト級）なため、複数のファイルに分割されています。
    * **内容:** 各サンプル（特定の時間・場所における大気カラム）に対して、以下のような多数の変数が含まれます。
        * **入力 (Features):**
            * 鉛直プロファイル (通常60レベル): `state_t` (温度), `state_q0001` (比湿), `state_q0002` (雲水混合比), `state_q0003` (雲氷混合比), `state_u` (東西風), `state_v` (南北風), `pbuf_ozone` (オゾン質量混合比), `pbuf_CH4` (メタン質量混合比), `pbuf_N2O` (亜酸化窒素質量混合比) など。
            * スカラー値: `state_ps` (地表気圧), `pbuf_SOLIN` (太陽入射放射量), `pbuf_LHFLX` (地表潜熱フラックス), `pbuf_SHFLX` (地表顕熱フラックス) など。合計で556次元の入力変数。
        * **出力 (Targets):**
            * 鉛直プロファイル (通常60レベル): `ptend_t` (温度変化率), `ptend_q0001` (比湿変化率), `ptend_q0002` (雲水変化率), `ptend_q0003` (雲氷変化率), `ptend_u` (東西風変化率), `ptend_v` (南北風変化率)。
            * スカラー値: `cam_out_NETSW` (正味短波放射), `cam_out_FLWDS` (下向き長波放射), `cam_out_PRECSC` (雪水フラックス), `cam_out_PRECC` (降水フラックス) など。合計で368次元の出力変数。
    * **解像度:** 低解像度 (LR) 版と高解像度 (HR) 版が存在する可能性があります。
2.  **`sample_submission.csv`**:
    * 提出フォーマットのサンプル。`sample_id` と、予測される368次元のターゲット変数の列を持ちます。各ターゲット変数には提出時に適用される重み (`submission_weights.csv`) があります。

**評価指標 (Evaluation Metric)**

* **指標:** **Skill Score** (R2スコアに基づく加重平均)
* **計算方法:**
    1.  各ターゲット変数に対して、モデルの予測値と真の値との間で **決定係数 (R2スコア)** を計算します。R2 = 1 - (Σ(y\_true - y\_pred)^2 / Σ(y\_true - y\_mean)^2)。
    2.  各ターゲット変数には、物理的な重要性などに基づいて重みが割り当てられています (`submission_weights.csv`)。
    3.  最終的なSkill Scoreは、各ターゲット変数のR2スコアを対応する重みで加重平均した値になります。
    * Skill Score = Σ (weight\_i * R2\_score\_i) / Σ weight\_i
* **意味:** モデルが各ターゲット変数の変動をどれだけ正確に再現できているかを、変数の重要度に応じて重み付けして評価します。R2スコアは、1に近いほどモデルの予測が真の値の変動をよく説明していることを示し、0であれば平均値を予測するのと同じ、負であれば平均値以下の性能であることを意味します。Skill Scoreは**高い**ほど、気候モデルの物理プロセス全体をより忠実にエミュレートできていると評価されます。

要約すると、このコンペティションは、気候シミュレーションデータを用いて大気物理プロセスを模倣する高次元の多変数回帰 (Seq2Seq) タスクです。データは数百次元の入力・出力変数からなり、性能は主要な出力変数に対する重み付きR2スコアの平均（Skill Score、高いほど良い）によって評価されます。

---

**全体的な傾向**

このコンペティションでは、気候モデルの物理プロセスをエミュレートするという、高次元かつ大規模なデータセットを扱う回帰タスクに取り組みました。上位解法にはいくつかの共通する戦略が見られました。

* **データの活用:** Hugging Faceで提供されている**低解像度 (LR) データセット全体を使用**することが、高スコアを得る上で非常に重要でした。一部のチームは高解像度 (HR) データも補助的に利用しました。データサイズが巨大なため、効率的なデータローダー (例: WebDataset, HDF5) の実装が鍵となりました。
* **モデルアーキテクチャ:** 特定の支配的なアーキテクチャはなく、多様なシーケンスモデルが試されました。
    * **RNN:** **LSTM (特に双方向LSTM)** が多くのチームで非常に有効でした。GRUも使用されました。
    * **CNN:** **1D CNN** も広く使われ、特にU-Net構造やConvNeXtライクなブロック、FiLMレイヤーとの組み合わせが見られました。
    * **Transformer:** **Squeezeformer** (Conv+Attention) や標準的なTransformer Encoderも単独または他の構造と組み合わせて利用されました。
    * **ハイブリッド:** LSTM+Transformer, CNN+LSTM, CNN+Transformer+LSTMなど、複数の要素を組み合わせたアーキテクチャが上位で見られました。
* **特徴量エンジニアリングと前処理:**
    * **入力形式:** 入力は通常、(バッチサイズ, 鉛直レベル=60, 特徴次元数) の形式に整形されました。スカラー入力は60レベルに複製して結合されました。
    * **正規化:** **StandardScaler** が標準的に用いられました。ターゲット変数も正規化されました。対数変換（特に混合比q系）や、全レベル共通の統計量を用いる正規化も試されました。
    * **追加特徴量:** 鉛直方向の**差分特徴量**（1次、2次）や、物理的に意味のある量（**相対湿度**, 浮力, 熱流束, 風速, 氷の割合など）を追加することが有効でした。
* **損失関数:**
    * **MAE / SmoothL1Loss (Huber Loss):** MSEよりも性能が安定し、高いスコアに繋がりやすい傾向がありました。
    * **MSE:** MAE/SmoothL1Lossで学習後のファインチューニングに用いることでスコアが向上するケースがありました。
    * **補助損失:** 鉛直隣接レベル間の差分を予測する損失や、信頼度を予測する損失、時空間情報（緯度経度、時刻）を予測する損失などが試されました。
    * **マスキング:** 提出対象外のターゲットや後処理で修正するターゲット (`ptend_q0002`の一部) を損失計算から除外することが一般的でした。
* **学習戦略:**
    * **オプティマイザ/スケジューラ:** AdamWが主流で、Cosine Annealingスケジューラ（Warmup付き、複数サイクル）やReduceLROnPlateauがよく使われました。
    * **大規模バッチ:** 性能向上のため、1024, 2048, 7000といった大きなバッチサイズが採用されました。
    * **複数ステージ学習:** MAE/SmoothL1Loss → MSE Lossのような段階的学習。あるいはグループファインチューニング（ターゲット変数を物理的なグループに分け、グループごとに再学習）。
    * **EMA:** 指数移動平均 (Exponential Moving Average) が有効でした。
* **後処理:**
    * **Ptend Trick:** `ptend_q0002` の特定レベル（例: 12-28）の値を、物理的な妥当性を考慮して `state_q0002` から計算される値 (`-state_q0002 / 1200`) で置き換える処理がほぼ必須でした。
    * **非負制約:** `q0002`, `q0003` など物理的に非負であるべき変数が負になった場合に0にクリップする。
    * **ゼロ重みターゲット:** sample_submissionの重みが0のターゲットの予測値を0にする。
    * **逆正規化:** FP64精度で行うことが推奨されました。
* **アンサンブル:** 多数のモデル（異なるアーキテクチャ、ハイパーパラメータ、学習データ、Foldなど）の予測をアンサンブルすることが最終スコア向上に不可欠でした。手法としては、重み付き平均（Nelder-Mead最適化など）や、スタッキング（1D CNN, LightGBMなど）が用いられました。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/leap-atmospheric-physics-ai-climsim/discussion/523063)**

* **アプローチ:** Squeezeformerベースのモデル。損失関数の工夫と多様なデータ表現、後処理が鍵。TPU活用。
* **アーキテクチャ:** Squeezeformerブロック x12。入力エンコーダ (Linear+LayerNorm)、予測ヘッド (Dense+SwiGLU)。
* **アルゴリズム:** MAE損失。補助損失（時空間情報、信頼度）。マスク損失。重み付き平均アンサンブル。
* **テクニック:**
    * **損失:** MAEをメインとし、補助損失（正規化された緯度経度、時刻のsin/cos、各ターゲットのMAE予測=信頼度）を追加。ゼロ重み/Ptend Trick対象ターゲットはマスク。
    * **データ:** LR+HRデータ (2:1)。TFRecords形式。
    * **入力表現:** 3種類の特徴量表現 (レベル別正規化、全体正規化、対数変換風正規化) と風速 (Wind) を結合。
    * **正規化:** 特徴量/ターゲットにソフトクリッピングを適用（特にHRデータ対応）。
    * **後処理:** Ptend Trick。FP64での逆正規化。R2<0のターゲットをゼロ化。
    * **アンサンブル:** 13モデルの重み付き平均 (重みは公開LBベースで調整)。

**[2位](https://www.kaggle.com/competitions/leap-atmospheric-physics-ai-climsim/discussion/523055)**

* **アプローチ:** LSTM/MAMBA/CNNを組み合わせた多様なアーキテクチャのアンサンブル。SmoothL1Loss、補助損失、グループファインチューン、Hill Climbアンサンブル。
* **アーキテクチャ:** ResLSTM, CNN+LSTM, LSTM+MAMBA, ConvLSTMなど多様。ResNet/Transformer構造の導入。
* **アルゴリズム:** SmoothL1Loss (beta=0.5)。補助Diff Loss (鉛直隣接レベル差)。Cosine Annealing。Group Fine-tuning。Hill Climbアンサンブル。
* **テクニック:**
    * **損失:** SmoothL1Loss。補助Diff Loss (隣接レベル差のSmoothL1Loss)。
    * **学習:** Cosine Annealing (3, 9エポックで減衰)。AdamW。
    * **Group Fine-tuning:** 全体学習後、ターゲットを7グループに分割し、各グループを1エポック再学習。
    * **データ:** 全LRデータ使用。8年目の後半+9年目1月を検証データ。
    * **アンサンブル:** Hill Climb法で各モデルの重みを探索。最終的に17モデルの重み付き平均。
    * **後処理:** Ptend Trick。ゼロ重みターゲットのゼロ化。

**[3位](https://www.kaggle.com/competitions/leap-atmospheric-physics-ai-climsim/discussion/523077)**

* **アプローチ:** 複数チームメンバーによる多様なモデル (1D CNN, Transformer, LSTM) のアンサンブル。LightGBMによる2段階目のリファインメント。
* **アーキテクチャ:**
    * Pao: 1D CNN + Transformer + LSTM。入力層での特徴別線形変換。
    * Camaro: CNN+Transformer または Transformerのみ (CLIPエンコーダ風)。
    * Kmat: FiLM 1D UNet。特徴量ごとにヘッドを分岐。状態分類ブランチも。
    * 2nd Stage: LightGBM。
* **アルゴリズム:** Huber Loss。補助損失（隣接レベル差）。Cosine Annealing。重み付き平均アンサンブル。
* **テクニック:**
    * **特徴量:** 元の特徴量に加え、差分（1次、2次）、相対湿度、飽和水蒸気圧、氷の割合などを追加。
    * **入力処理 (Pao):** スカラー特徴量を各レベルの特徴量に結合。高さ埋め込み。
    * **損失 (Camaro, Kmat):** Huber Loss (delta=2 or 4 or 8)。
    * **学習:** 全LRデータ使用。AdamW。
    * **2nd Stage:** Camaroモデルの予測値と一部の入力特徴量を用いて、LightGBMで `ptend_q` 系の予測値をレベルごとに再予測（リファインメント）。
    * **アンサンブル:** 9モデル (Camaro x5, Pao x3, Kmat x1) の重み付き平均。

**[4位](https://www.kaggle.com/competitions/leap-atmospheric-physics-ai-climsim/discussion/523042)**

* **アプローチ:** ConvNeXtライクな1D CNNとTransformerモデルのアンサンブル。特徴量エンジニアリングとハイパーパラメータチューニング。
* **アーキテクチャ:**
    * ConvNeXt 1D: ConvNeXtブロックを1次元入力用に改変。BatchNorm使用。カスタムヘッド (Conv1D+Linear)。
    * Transformer: 標準的なTransformer Encoder。カスタムヘッド (ターゲット別にConv1D/Linear)。位置エンコーディング。
* **アルゴリズム:** SmoothL1Loss (beta=0.01 or 1)。Polynomial Decayスケジューラ。重み付き平均アンサンブル。
* **テクニック:**
    * **データ:** 全LRデータ使用。
    * **特徴量:** 差分特徴量 (x[i]-x[i-1], x[i]-x[i-2])、類似特徴量の平均/差分平均。
    * **前処理:** StandardScaler。値クリッピング (-100, 100)。スカラー値の60回複製。
    * **学習:** AdamW。Polynomial Decay + Warmup。モデルサイズに応じてハイパラ調整（LR, weight decay, loss beta）。
    * **アンサンブル:** 1D CNN系とTransformer系のモデルを複数ブレンド。
    * **後処理:** Ptend Trick。

**[5位](https://www.kaggle.com/competitions/leap-atmospheric-physics-ai-climsim/discussion/523040)**

* **アプローチ:** BiLSTMベースのシンプルなモデルのアンサンブル。異なるデータセット混合と特徴エンジニアリングで多様性を確保。
* **アーキテクチャ:** BiLSTM (3層または6層) がコア。MLPエンコーダ/デコーダ、GRU、1D Average Pooling、Attentionメカニズムなどを組み合わせたバリエーション。
* **アルゴリズム:** Huber Loss (delta=1) または MSE+MAE混合損失。重み付き平均アンサンブル。
* **テクニック:**
    * **データ:** LR + Aqua-Planet混合、またはLR + HR (1/15) 混合の2パターンでモデルを学習。
    * **入力処理 (LR+Aqua):** スカラー特徴量を60回複製して結合。MLP Encoder/Decoderによる特徴量変換。追加特徴量（相対湿度、温度差、風速差など）も一部モデルで使用。
    * **入力処理 (LR+HR):** q系、オゾン、CH4, N2Oを対数変換後、StandardScalerで正規化。ターゲットは正規化せず直接予測。
    * **学習:** AdamW。Huber LossまたはMSE/MAE混合損失。EMA。
    * **アンサンブル:** 異なるデータセット/特徴量/アーキテクチャを持つBiLSTMベースモデル群の重み付き平均。LR+HRモデルの重みをやや高く設定。
    * **後処理:** `ptend_q` (q0001, q0002, q0003) について、予測値と `-state / 1200` の最大値を取る。

**[7位](https://www.kaggle.com/competitions/leap-atmospheric-physics-ai-climsim/discussion/524111)**

* **アプローチ:** 多様なアーキテクチャ (LSTM, Transformer+LSTM, Conv1D, Squeezeformer) のアンサンブル。MAE+MSEの2段階学習。Nelder-Meadによる重み最適化。
* **アーキテクチャ:** Transformer+LSTM, LSTM, Conv1D (ResNetブロック+SE Block+Inception風並列畳み込み), Squeezeformer, FiLM 1D UNet, LSTM (別メンバー)。
* **アルゴリズム:** MAE Loss → MSE Loss ファインチューニング。Cosine Annealing + Warmup。Nelder-Meadアンサンブル重み最適化。
* **テクニック:**
    * **データ:** 全LRデータ使用。一部メンバーはKaggleデータのみで実験。
    * **特徴量:** 差分特徴量 (1次、2次)、相対湿度、気圧差、水蒸気圧、氷の割合、氷率差などを追加。緯度経度は最終的に不使用。
    * **損失:** MAEで学習後、MSEで5エポックファインチューニング (+0.002改善)。ターゲットマスキング。
    * **モデル詳細 (Ryota):** Transformer+LSTM, LSTM, Conv1Dモデル。活性化関数や正規化層を使い分け。
    * **モデル詳細 (sqrt4kaido):** Squeezeformer, LSTM。後処理で非負制約。
    * **モデル詳細 (e-toppo):** LSTM。補助損失 (ptend_RH)。後処理で非負制約、温度依存の氷制約。
    * **モデル詳細 (Rheinmetall):** LSTM。スカラー特徴量を隠れ状態初期値として入力。MSE Loss (スカラーヘッドは重み0.1)。
    * **アンサンブル:** 上位6つのシングルモデル予測をNelder-Meadで最適化した重みで加重平均。

**[8位](https://www.kaggle.com/competitions/leap-atmospheric-physics-ai-climsim/discussion/523223)**

* **アプローチ:** BiLSTMベースのシンプルなSeq2Seqモデルとその派生 (Transformer, Attention, TCN, CNN追加) のアンサンブル。
* **アーキテクチャ:** BiLSTM (6層) がコア。Transformer Encoder, Multihead Attention, TCN (Temporal Convolutional Network), CNNブロックなどをLSTMと組み合わせたバリエーション。
* **アルゴリズム:** SmoothL1Loss。Cosine Annealing + Warmup。ターゲットごとのアンサンブル重み最適化。
* **テクニック:**
    * **データ:** 全LRデータ (7.5年分) を使用。後半6ヶ月分を検証用。
    * **入力形式:** (bs, 60, 25) -> LSTM -> (bs, 60, hidden*2) -> Head -> (bs, 60, 14) -> 最終出力 (bs, 368) に整形。
    * **損失:** SmoothL1Loss (MSEより良好)。
    * **活性化関数:** GELU (ReLUより良好)。
    * **アンサンブル:** 複数モデルの予測に対し、ターゲット変数ごとに最適なモデルを選択/重み付けする方式と、単純なモデル平均の両方を試行。ターゲット別選択の方がやや良好。
    * **後処理:** Ptend Trick。ゼロ重みターゲットのゼロ化。

**[10位](https://www.kaggle.com/competitions/leap-atmospheric-physics-ai-climsim/discussion/523041)**

* **アプローチ:** 4人のメンバーによる多様なモデル (Conv+Transformer, CNN/U-Net, LSTM) のアンサンブル。スタッキングによる重み最適化。
* **アーキテクチャ:** Conv+Transformer (Residual Block + LSTM/MAMBA), Pixel-Shuffle UNet, LSTM, Transformer+Conv+LSTM, FiLM 1D UNet。
* **アルゴリズム:** Huber Loss, MAE, Confidence-aware MSE Loss。Cosine Annealing。スタッキング (1D CNN)。重み付き平均。
* **テクニック:**
    * **データ:** 基本はLR 1-7年データ。一部モデルは8年目データやHRデータ、疑似ラベルデータも使用。サブサンプリングやハードサンプルドーピングも試行。
    * **特徴量:** 気候不変特徴量 (相対湿度、浮力、正規化熱流束など)、差分特徴量。
    * **モデル工夫:** Tanh正規化 (Attention安定化)、Dropout削除、ターゲット別ヘッド分岐、状態分類ブランチ追加など。
    * **スタッキング:** モデル予測を (bs, n_models, 368) の入力とし、1D CNNで最終予測。10 Foldで学習。
    * **アンサンブル:** スタッキング結果と平均アンサンブル結果をさらに重み付けしてブレンド。
    * **後処理:** Ptend Trick。非負制約。
