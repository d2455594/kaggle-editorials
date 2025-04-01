---
tags:
  - Kaggle
startdate: 2023-05-10
enddate: 2023-06-09
---
# Parkinson's Freezing of Gait Prediction
[https://www.kaggle.com/competitions/tlvmc-parkinsons-freezing-gait-prediction](https://www.kaggle.com/competitions/tlvmc-parkinsons-freezing-gait-prediction)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、パーキンソン病患者の日常生活における**すくみ足 (Freezing of Gait, FOG)** イベントを、**ウェアラブルセンサー（加速度計）**から得られた時系列データを用いて検出・予測する機械学習モデルを開発することです。特に、歩行開始時のすくみ足 (StartHesitation)、方向転換時のすくみ足 (Turn)、歩行中のすくみ足 (Walking) の3種類のイベントを対象とします。
* **背景:** FOGはパーキンソン病患者によく見られる症状で、突然足が動かなくなり転倒のリスクを高めます。ウェアラブルセンサーによる客観的なFOG検出は、症状のモニタリングや治療法の評価、転倒予防システムの開発に役立ちます。センサーデータは個人の動きを継続的に記録しますが、日常生活の多様な動きの中からFOG特有のパターンを正確に識別することは困難です。
* **課題:** 加速度センサーデータ（AccV, AccML, AccAP）からなる**時系列データ**から、FOGイベント（StartHesitation, Turn, Walking）が発生する**時間区間を正確に特定**することです。データには個人差や活動内容による変動が含まれ、またFOGイベントは比較的まれに発生するため、クラス不均衡の問題もあります。提供されるデータには2種類（tdcsfog, defog）があり、それぞれ異なる特性を持つ可能性があります。これは、本質的には多クラス（またはマルチラベル）の**時系列セグメンテーションタスク**です。

**データセットの形式 (Dataset Format)**

提供される主なデータは、被験者ごとの加速度センサーデータと、FOGイベントの時間区間を示すアノテーションです。

1.  **センサーデータ (tdcsfog/ & defog/ ディレクトリ):**
    * 各サブディレクトリは被験者ごとのデータを含みます。
    * ファイル形式は `.csv` です。
    * 各ファイルには、`Time` (タイムスタンプ)、`AccV` (垂直方向加速度)、`AccML` (左右方向加速度)、`AccAP` (前後方向加速度) の列が含まれます。これらがモデルの**入力特徴量**となります。
    * `tdcsfog` と `defog` は異なる研究プロジェクトまたはデバイスから収集されたデータである可能性があり、サンプリングレートや単位が異なる場合があります (`tdcsfog` は 128Hz, `defog` は 100Hz で m/s² と g の違いあり)。
2.  **アノテーションデータ (`tdcsfog_metadata.csv`, `defog_metadata.csv`, `events.csv`):**
    * `tdcsfog_metadata.csv`, `defog_metadata.csv`: 各センサーデータファイルに対応するメタデータ（被験者ID、記録IDなど）。
    * `events.csv`: **ターゲット変数**となるFOGイベントのアノテーション情報。`Id` (記録ID)、`Start` (イベント開始時間)、`End` (イベント終了時間)、`Type` (StartHesitation, Turn, Walkingのいずれか) が含まれます。`Start` から `End` までの区間が、対応する `Type` のFOGイベントが発生した期間です。
    * `tasks.csv`: `events.csv` に関連し、特定のタスク（歩行、方向転換など）が行われていた期間を示します。`Valid` 列と `Task` 列は、ラベル付けや評価の対象となる区間を示すために重要です (`defog` データに主に関連)。
3.  **被験者情報 (`subjects.csv`):**
    * 各被験者のメタデータ（年齢、性別、罹患期間など）。特徴量として利用されることもあります。
4.  **`sample_submission.csv`:**
    * 提出フォーマットのサンプル。`Id` (記録IDとタイムステップを組み合わせた識別子) と `StartHesitation`, `Turn`, `Walking` の各列に、そのタイムステップで各イベントが発生している確率（またはバイナリ）を予測します。

**評価指標 (Evaluation Metric)**

* **指標:** **クラスごとの平均適合率 (Average Precision, AP) の平均 (mean Average Precision, mAP)**
* **計算方法:**
    1.  各FOGイベントタイプ（StartHesitation, Turn, Walking）について、Average Precision (AP) を計算します。APはPrecision-Recall曲線の下の面積であり、予測確率の閾値を変化させた際の適合率と再現率の関係を統合した指標です。イベント検出タスクで広く用いられます。
    2.  3つのイベントタイプのAPを算術平均して、最終的なmAPスコアを算出します。
        mAP = (AP<sub>StartHesitation</sub> + AP<sub>Turn</sub> + AP<sub>Walking</sub>) / 3
* **意味:** モデルが各FOGイベントタイプをどれだけ正確に検出できているかを総合的に評価します。mAPは、イベントの有無だけでなく、予測確率の信頼度も考慮する指標です。スコアは **0から1** の範囲を取り、**高い**ほど性能が良いことを示します。この指標は、偽陽性（FOGでないのにFOGと予測）と偽陰性（FOGなのに見逃す）の両方をバランス良く評価しつつ、特にランキング（確率の高い予測が実際にFOGであるか）の質を重視します。

要約すると、このコンペティションは、ウェアラブル加速度センサーの時系列データから3種類のパーキンソン病FOGイベントを検出するマルチラベル時系列セグメンテーションタスクです。データはセンサー記録とイベントアノテーションで構成され、性能は各イベントタイプのAverage Precisionの平均 (mAP、高いほど良い) によって評価されます。

---

**全体的な傾向**

パーキンソン病患者の加速度センサーデータからFOGイベント（すくみ足）を検出するこのタスクでは、時系列データの特性を捉えるモデルが中心となりました。特に**RNN (GRU, LSTM)** や **Transformer**、**1D CNN** が広く採用されています。多くのチームが、特性の異なる可能性のある `tdcsfog` と `defog` データセットに対して、**別々のモデルを学習させる**アプローチを取りました（ただし、統合して単一モデルで扱う解法も存在）。

データの前処理として、**正規化**（Mean-Std, RobustScaler, StandardScaler）、**リサンプリング**（特にtdcsfogとdefogの周波数を合わせるため）、**ウィンドウ（シーケンス）分割**が重要でした。推論時に学習時よりも**長いシーケンス長を用いる**テクニックが複数の上位解法で有効と報告されており、これがスコア向上に大きく寄与した可能性があります。

特徴量エンジニアリングとしては、加速度の差分や累積和、時間に関する特徴量（正規化時間、Sin/Cos時間）などが試されました。**Augmentation**（ノイズ付加、タイムワープ、フリップ、マグニチュード変更など）も多くの解法で用いられました。Defogデータに含まれるラベルなし区間（`notype`）や `Valid`/`Task` フラグの扱い、`Event` 列を利用した**疑似ラベリング**もスコア向上に貢献したケースが見られます。

モデルアーキテクチャでは、単純なRNNやCNNだけでなく、**Residual接続**、**Attention機構**、**Squeeze-and-Excitation**、**U-Net構造**、**WaveNetブロック**などを組み込んだより複雑なものが試されました。Vision Transformerのように時系列データを**パッチ化**してTransformerに入力するアプローチも有効でした。

学習においては、適切な損失関数（BCE Loss, Cross Entropy）、オプティマイザ（AdamW, Ranger）、学習率スケジュール（Cosine Annealing, Linear Warmup）の選択が重要です。**アンサンブル**（複数モデル、複数Fold、複数シード）は、最終的なスコアを向上させるための一般的なテクニックでした。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/tlvmc-parkinsons-freezing-gait-prediction/discussion/416026)**

* **アプローチ:** tdcsfogとdefogで別モデル。Transformer EncoderとBiLSTMの組み合わせ。Vision Transformer風のパッチ化。ターゲットの解像度低減。
* **アーキテクチャ:** FOGEncoder (Transformer Encoder Layer x5 + Positional Encoding) + BiLSTM x2 + Dense。
* **アルゴリズム:** BCE Loss (マスク適用)。Adam (カスタム学習率スケジュール、Warmup付き)。
* **テクニック:**
    * **前処理:** Mean-Std正規化。ゼロパディング。パッチ化 (例: 15552点を864x(18x3=54)のパッチに)。ターゲットもパッチごとにmax poolingで解像度低減。
    * **入力:** 加速度3軸のみ。メタデータ未使用。
    * **モデル:** tdcsfog用4モデル、defog用4モデルを学習し、それぞれ平均でアンサンブル。モデルごとにハイパーパラメータ（次元数、Head数、Layer数など）や学習データ分割を変更。
    * **Augmentation:** 学習時にPositional Encodingをランダムにシフト。
    * **推論:** 予測結果を元の解像度に復元 (`tf.tile`)。

**[2位](https://www.kaggle.com/competitions/tlvmc-parkinsons-freezing-gait-prediction/discussion/416057)**

* **アプローチ:** tdcsfogとdefogで別モデル。両方ともGRUベース。短いシーケンスで学習し、長いシーケンスで推論。Defogでは疑似ラベル活用。
* **アーキテクチャ:** GRU (多層)。
* **アルゴリズム:** BCEWithLogitsLoss (重み付きあり)。AdamW。Linear Schedule with Warmup。StratifiedGroupKFold (Subject単位)。
* **テクニック:**
    * **特徴量:** 加速度3軸 + 差分 + 累積和 (tdcsfog)。加速度3軸 + 差分 (defog)。
    * **前処理:** RobustScaler (tdcsfog), StandardScaler (defog)。IDごとに正規化。
    * **シーケンス:** 学習時は短く (tdcsfog: 1000, defog: 5000)、推論時は長く (tdcsfog: 3000/5000, defog: 15000/30000)。オーバーラップさせてシーケンス作成。
    * **疑似ラベル:** Defogの `notype` データを `Event` 列に基づいて疑似ラベリングし学習に利用（複数ラウンド実施）。
    * **重み付き損失:** 特定ターゲットの損失重みを上げて学習するモデルを作成し、そのターゲットの予測のみを使用。
    * **アンサンブル:** 複数モデル（均一損失、重み付き損失）の結果を重み付き平均。
    * **モデル選択:** CVとPublicスコアの相関が後半で崩れたため、Publicスコアでモデル選択、CVでシーケンス長を決定。

**[3位](https://www.kaggle.com/competitions/tlvmc-parkinsons-freezing-gait-prediction/discussion/417717)**

* **アプローチ:** tdcsfogとdefogに単一モデルを適用。Transformer + RNN。
* **アーキテクチャ:** DeBERTa/Vision Transformer/Vision Transformer (Relative Position) Encoder + LSTM/GRU Decoder (通常2-4層、RNNは単層)。
* **アルゴリズム:** (詳細不明だが、一般的なシーケンスモデルの学習法と推測される)
* **テクニック:**
    * **パッチ化:** パッチサイズ7-13、シーケンス長192-384パッチ。
    * **Augmentation:** ストレッチ、クロッピング、アブレーション、累積ガウスノイズなど、重度のAugmentation。

**[4位](https://www.kaggle.com/competitions/tlvmc-parkinsons-freezing-gait-prediction/discussion/416410)**

* **アプローチ:** tdcsfogとdefogを統合し単一モデルで学習。Residual接続を持つ多層BiGRU。
* **アーキテクチャ:** MultiResidualBiGRU (入力FC -> LayerNorm -> ResidualBiGRUブロック x N層 -> 出力FC)。ResidualBiGRUブロック内 (GRU -> FC -> LayerNorm -> ReLU -> FC -> LayerNorm -> ReLU -> Skip Connection)。
* **アルゴリズム:** Cross Entropy Loss。Ranger Optimizer (Adam, AdamWも試行)。Cosine Annealing Schedule。バッチサイズ1。混合精度学習、勾配クリッピング。
* **テクニック:**
    * **前処理:** 加速度3軸のみ使用。50Hzにダウンサンプリング。単位変換 (defog)。`Valid` と `Task` でマスク作成。シーケンスごとにStandardScalerで正規化。
    * **4クラス分類:** FOG 3クラス + "no-activity" クラスを追加して学習。
    * **CV:** 主モデルはTrain/Validation Split (80/20)。後にStratified K-Fold (k=5) でアンサンブルも試行（スコア微減）。
    * **入力:** ダウンサンプルしたシーケンス全体を一度に入力（バッチサイズ1）。
    * **GRU初期化:** 被験者メタデータ（年齢、性別など）をFCで射影し、GRUの初期隠れ状態として使用する試み（プロトタイプ）。1D Convでの特徴抽出も試行。

**[5位](https://www.kaggle.com/competitions/tlvmc-parkinsons-freezing-gait-prediction/discussion/418275)**

* **アプローチ:** WaveNetブロックとGRUの組み合わせ。tdcsfog/defog共通モデル。事前学習の試み。ONNX変換によるCPU推論。
* **アーキテクチャ:** Wave_Block (Dilated Conv) x3 + BiGRU (4層) + Dense。
* **アルゴリズム:** (損失関数は明記されていないが、BCE Lossなどが一般的)。(Optimizer, Schedulerは不明)。GroupKFold (Subject単位)。
* **テクニック:**
    * **前処理:** ウィンドウ分割 (サイズ2000, オーバーラップ500)。tdcsfogを100Hzにリサンプリング (librosaバージョンに注意)。
    * **データセット:** tdcsfog/defogからランダムにウィンドウを選択してバッチ作成。
    * **事前学習:** ラベルなしデータで時系列の次ステップ予測タスクによる事前学習を試行 (WaveNet部分のみ)。
    * **モデルバリアント:** `Valid` フラグの扱い方や重み保存戦略が異なる複数のモデル (v1, v2, v3) を学習。
    * **推論:** 長いウィンドウ (16000 or 20000) で分割して推論。ONNXに変換してCPUで高速化。複数モデル/重みを平均でアンサンブル。
    * **結果:** 事前学習モデルを含むアンサンブルがPublic LBでは良かったがPrivate LBでは低下。GPU推論では事前学習モデルが有効だった可能性。

**[6位](https://www.kaggle.com/competitions/tlvmc-parkinsons-freezing-gait-prediction/discussion/415992)**

* **アプローチ:** スペクトログラム、ウェーブレット、1D Convベースの複数モデルのアンサンブル。tdcsfog/defog共通モデル。
* **アーキテクチャ:**
    * **Spectrogram:** STFT -> 2D CNN (例: ResNet18) -> U-Net Decoder -> Pooling -> Transformer Encoder -> Dense。
    * **Wavelet:** CWT -> 2D CNN (例: ResNet18/34) -> Pooling -> Transformer Encoder -> Dense。
    * **1D Conv:** 1D CNN + メタデータ特徴量 + Cumsum特徴量。
* **アルゴリズム:** (詳細不明だが、画像/時系列モデルの標準的な学習法)。ネストしたCV (Outer/Inner Folds)。
* **テクニック:**
    * **特徴量:** 加速度3軸 + 時間特徴量 (`pct_time`)。一部モデルでメタデータも使用。
    * **スペクトログラム/ウェーブレット:** 時間次元のダウンサンプリング。周波数エンコーディング追加。低周波数帯のみ使用 (0-15Hz)。
    * **リサンプリング:** 32, 64, 128Hzなど様々な周波数で試行。Defogをtdcsfogに合わせる。
    * **Transformer:** 自己注意に距離ベースのバイアスマスクを適用。
    * **Augmentation:** Time Stretch, Gaussian Noise, Pitch Shift, Wave Scale Aug, Time Feature Shift Augなど (スペクトログラム/ウェーブレット)。
    * **1D Conv:** Defog/tdcsfogの時間軸整列、Outlier被験者の重み低減、Snapshot Ensembling、`notype` データ活用 (max pooling予測)。
    * **アンサンブル:** 複数モデルタイプの結果を重み付き平均 (GP_minimizeで重み探索)。

**[8位](https://www.kaggle.com/competitions/tlvmc-parkinsons-freezing-gait-prediction/discussion/416021)**

* **アプローチ:** ベースライン (Mayukh18氏) の改良。1D-ResNetを使用。5-Fold CVアンサンブル。tdcsfog/defog共通モデル。
* **アーキテクチャ:** 1D-ResNet (3チャネル入力)。
* **アルゴリズム:** (損失関数は不明、ベースラインはBCE)。(Optimizerは不明、ベースラインはAdam)。ReduceLROnPlateauスケジューラ (LR=0.001から開始)。
* **テクニック:**
    * **データセット:** ベースラインのDatasetクラスを1D-ResNet用に変更。
    * **ウィンドウ:** 1000msのカット、未来予測50ms。(ベースラインから変更)。
    * **CV:** ベースラインのFold定義に基づき5つのモデルを学習し、アンサンブル。

**[10位](https://www.kaggle.com/competitions/tlvmc-parkinsons-freezing-gait-prediction/discussion/416513)**

* **アプローチ:** Squeeze-and-Excitation付き1D U-Net。非常に長いコンテキストウィンドウ。tdcsfog/defog共通モデル。
* **アーキテクチャ:** 1D U-Net (5 Encoder/Decoderペア) + Squeeze-and-Excitationブロック。
* **アルゴリズム:** (損失、Optimizer、Schedulerは不明)。
* **テクニック:**
    * **特徴量:** 生の加速度3軸 (正規化なし) + 時間特徴量 (NormalizedTime, SinNormalizedTime)。周波数特徴量は効果なし。
    * **コンテキスト長:** 10240サンプルという非常に長いウィンドウで処理。
    * **Augmentation (学習時):** Random Low Pass Filtering, Random Time Warp, Random Flip (AccML), Random Magnitude Warping, Noisy Time Features。
    * **Augmentation (推論時 - TTA):** AccMLフリップ有無、オーバーラップウィンドウ推論 (Stride=Window/8)。計16回の予測を平均。
    * **正規化:** 正規化は有害と判断し、行わない。サンプルレートや単位も統一せず。
    * **アンサンブル:** 同じハイパーパラメータ、同じFoldだが異なる乱数シードで学習した2つのモデルをアンサンブル（スコアの良いものを選別）。


