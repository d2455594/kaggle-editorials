---
tags:
  - Kaggle
  - 物体検出
  - UNet
  - EfficientNet
startdate: 2024-11-07
enddate: 2025-02-06
---
# CZII - CryoET Object Identification
[https://www.kaggle.com/competitions/czii-cryo-et-object-identification](https://www.kaggle.com/competitions/czii-cryo-et-object-identification)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、細胞の3次元構造を高解像度で可視化する技術である**クライオ電子線トモグラフィ (Cryo-Electron Tomography, CryoET)** によって得られたトモグラム（3D画像データ）から、内部に含まれる**複数の種類の生体分子粒子（タンパク質複合体など）を検出し、その中心位置と種類を特定する**AIモデルを開発することです。
* **背景:** CryoETは、細胞や組織の構造を分子レベルの解像度で研究するための強力な手法です。トモグラム内には多種多様な分子構造が混在しており、特定の粒子を正確に同定・位置特定することは、細胞生物学や構造生物学の研究において非常に重要です。しかし、手作業による粒子の同定は膨大な時間と専門知識を要するため、自動化・高精度化が強く求められています。
* **課題:** トモグラム画像はノイズが多く、また内部には多種多様な粒子が様々な向き・状態で存在するため、対象となる粒子を正確に検出・分類することは困難です。データは3次元であり、モデルは3D空間内の粒子の位置と種類を予測する必要があります。粒子は非常に小さく、互いに近接している場合もあります。また、評価指標（F4スコア）はRecall（再現率）を非常に重視するため、見逃しを少なくすることが求められました。

**データセットの形式 (Dataset Format)**

提供される主なデータは、CryoETトモグラム、粒子の注釈情報、およびシミュレーションデータです。

1.  **トレーニングデータ:**
    * `train_images/`: 7つの実験データセット（トモグラム）の3D画像データ（MRC形式）。ファイル名は `TS_***.mrc`。各ファイルが1つのトモグラムを表します。
    * `train_annotations/`: 各トレーニングトモグラムに対応する注釈ファイル（CSV形式）。
        * `dataset_id`: トモグラムのID。
        * `image_name`: トモグラムファイル名。
        * `x`, `y`, `z`: 粒子中心の3D座標。
        * `label`: 粒子の種類を示す文字列（例: `ribosome`, `apo-ferritin` など、計6種類）。
2.  **シミュレーションデータ:**
    * `simulation_data/images/`: シミュレーションによって生成された多数のトモグラムデータ（MRC形式）。
    * `simulation_data/annotations/`: 対応する注釈ファイル（CSV形式）。ラベルには、より詳細な情報（回転角度など）も含まれます。これは主に事前学習に利用されました。
3.  **テストデータ:**
    * `test_images/`: 評価対象となるトモグラムデータ（MRC形式）。
    * 注釈情報は提供されません。
4.  **`sample_submission.csv`**:
    * 提出フォーマットのサンプル。`image_name`, `x`, `y`, `z`, `label` の列を持ちます。各行が検出された1つの粒子に対応します。

**評価指標 (Evaluation Metric)**

* **指標:** **平均F4スコア (Average Fbeta score with β=4)**
* **計算方法:**
    1.  各粒子クラスごとに、予測された粒子中心座標と真の粒子中心座標をマッチングさせます。マッチングは、予測座標と真座標の間のユークリッド距離が、粒子クラスごとに定義された許容距離 (`τ`) 以内である場合に行われます（通常は粒子の半径に基づく）。最適なマッチング（ハンガリアンアルゴリズムなど）により、True Positives (TP), False Positives (FP), False Negatives (FN) をカウントします。
    2.  各クラスごとにF4スコアを計算します。FbetaスコアはRecallをPrecisionよりもβ倍重視する指標であり、F4スコアはRecallを極めて重視します。
        * Precision = TP / (TP + FP)
        * Recall = TP / (TP + FN)
        * F4 = (1 + 4²) * (Precision * Recall) / ((4² * Precision) + Recall) = 17 * (Precision * Recall) / (16 * Precision + Recall)
    3.  全クラス（スコアリング対象は5クラス）のF4スコアの平均値を最終スコアとします。
* **意味:** 予測された粒子と実際の粒子の一致度を評価します。F4スコアは、**Recall（再現率、真の粒子を見逃さないこと）を極めて強く重視**します。これは、科学的な発見のためには、可能な限り多くの粒子を検出することが重要であるというタスクの性質を反映しています。偽陽性（誤検出）もある程度許容されますが、見逃しに対するペナルティが非常に大きいです。スコアは**高い**ほど良い評価となります。

---

**全体的な傾向**

このCryoET画像からの粒子検出タスクでは、3次元のボリュームデータから小さな粒子を検出・分類する必要があるため、**3Dセグメンテーション**または**3Dオブジェクト検出**のアプローチが主流でした。特に**U-Netベースの3Dセグメンテーションモデル**が多くの上位チームで採用されました。MONAIライブラリの3D U-Net, SegResNet, DynUNet, FlexibleUNetなどが活用されました。一部では2.5Dアプローチ（2D Encoder + 3D情報集約）も試されました。

モデルは、各ボクセルがどの粒子クラスに属するか（または背景か）を示す確率マップ、あるいは粒子中心からの距離や方向を示すヒートマップなどを出力するように学習されました。

**データ処理**では、入力となる3Dトモグラムから**パッチ (Patch/Subvolume)** を切り出して学習・推論を行うのが一般的でした。パッチサイズは64^3から128^3程度が多く見られました。入力データの**正規化**（標準正規化、パーセンタイルクリッピング+MinMax正規化など）や、**データ拡張**（フリップ、回転、アフィン変換、ガウスノイズ、MixUp/CutMixなど）も重要でした。

**ラベル表現**として、粒子中心座標だけでなく、中心からの距離や形状を考慮した**ガウシアンヒートマップ**や、特定の半径を持つ**球体マスク**を教師信号として用いるアプローチが多く取られました。半径の大きさは粒子クラスごとに調整されました。

**損失関数**は、クラス不均衡（背景ボクセルが圧倒的に多い）に対処するため、**重み付きCross Entropy Loss**、**Dice Loss**、**Tversky Loss**、**Focal Loss**などが単独または組み合わせて使用されました。特に陽性（粒子）ボクセルの重みを高く設定することが効果的でした。

**事前学習**として、提供された**シミュレーションデータ**を用いることが一般的でした。シミュレーションデータで事前学習し、実験データでファインチューニングすることで、性能が向上しました。

**後処理**はスコア向上に不可欠でした。モデルが出力した確率マップやヒートマップから、最終的な粒子中心座標リストを生成するために、以下のステップが踏まれました。
1.  **閾値処理:** 各クラスの確率マップを二値化するための閾値を設定（クラスごとに最適化）。
2.  **連結成分解析 (Connected Component Analysis, CCA/CC3D):** 二値化マスクから連結した塊（クラスター）を抽出。
3.  **重心計算:** 各クラスターの重心 (Centroid) を計算し、粒子の候補位置とする。
4.  **フィルタリング/クラスタリング:** 小さすぎるクラスター（ノイズ）を除去したり、近接する候補点をクラスタリング（DBSCANなど）したりして、最終的な予測リストを精製。

最終提出は、異なるモデルアーキテクチャ、異なるFold、異なるシードなどで学習した**複数モデルの予測（ロジットまたは確率マップ）を平均化するアンサンブル**が標準的でした。**Test Time Augmentation (TTA)**（フリップ、回転など）も広く用いられました。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/czii-cryo-et-object-identification/discussion/561510)**

* **アプローチ:** 3D U-Net (ResNet/EfficientNet B3 Encoder) によるセグメンテーションと、MONAIの3D物体検出モデル (SegResNet/DynUnet) のアンサンブル。MONAIフレームワーク活用。
* **アーキテクチャ:** MONAI FlexibleUnet (ResNet34/EfficientNet-B3)、MONAI SegResNet/DynUnet。一部モデルは最終層手前の特徴マップを使用（Partly U-Net）。
* **アルゴリズム:** Segmentation: 重み付きCross Entropy Loss (陽性ピクセル重み256)。Object Detection: MONAIデフォルト?
* **テクニック:**
    * **データ:** 実験データのみ使用（シミュレーションデータ不使用）。7 Fold CV (実験ID分割)。
    * **学習:** MONAI Augmentations (RandomCrop, Flip, Rotate)。自作MixUp。Cosine LRスケジュール。AMP。
    * **セグメンテーションターゲット:** 粒子中心の単一ピクセルをターゲットとし、背景クラスの重みを1/256に低減。
    * **アンサンブル:** セグメンテーションモデルと物体検出モデルの出力をアンサンブル。異なる分布を持つ特徴マップをランクベースでスケール調整してから平均化し、物体検出後処理へ。
    * **推論:** TTA。JIT/TensorRTで高速化。閾値はクラスごとに最適化。

**[2位](https://www.kaggle.com/competitions/czii-cryo-et-object-identification/discussion/561568)**

* **アプローチ:** 軽量な3Dセグメンテーションモデル (UNet3D, VoxResNet, VoxHRNet, SegResNet, DenseVNet, UNet2E3D) の大規模アンサンブル。CC3Dとサイズフィルタリングによる後処理。
* **アーキテクチャ:** MONAIの多様な軽量3Dセグメンテーションモデル。InstanceNorm3d + PReLUが安定性に寄与。
* **アルゴリズム:** Tversky Loss, Dice Loss, Cross Entropy Loss (重み付き/スケーリングあり)。AdamW。
* **テクニック:**
    * **データ:** 7 Fold CV。
    * **学習:** パッチサイズ 128x200x200 or 128x256x256。RandCropByLabelClassesdでクラスバランスサンプリング。RandRotate90d, RandFlipd, RandShiftIntensitydなどのAugmentation。
    * **ラベル表現:** 各粒子中心にカスタム半径（粒子半径 x 0.4 or 0.5）の球体マスクを生成。
    * **アンサンブル:** Public LBスコアに基づき多数のモデルを選択し、平均アンサンブル。
    * **後処理:** 各クラスに閾値 (0.15) を適用して二値化 → CC3Dで連結成分抽出 → 各クラスの統計量に基づき小さすぎるクラスターを除去。

**[3位](https://www.kaggle.com/competitions/czii-cryo-et-object-identification/discussion/561417)**

* **アプローチ:** 3D U-Net (ResNet101 backbone) モデルのアンサンブル。CC3Dによる後処理。
* **アーキテクチャ:** segmentation_models_pytorch_3d のU-Net + ResNet101。
* **アルゴリズム:** Cross Entropy Loss (7クラス、beta-amylase含む)。AdamW?
* **テクニック:**
    * **データ:** 7 Fold CV。
    * **学習:** 入力パッチ 64x128x128。EMA使用。学習時ターゲット半径は元半径の半分。
    * **Augmentation:** Flip (xyz), Switch axis (xy), Mixup, Copy-paste? 異なるノイズ除去アルゴリズム適用画像をAugmentationとして使用。
    * **推論:** 入力パッチ 64x256x256（学習時より大きく）。TTA (Flip xyz, Rot90 xy)。
    * **アンサンブル:** 4 Fold の平均アンサンブル。
    * **後処理:** CC3Dで連結成分抽出。

**[4位](https://www.kaggle.com/competitions/czii-cryo-et-object-identification/discussion/561401)**

* **アプローチ:** Heatmapベースの3D粒子位置検出。2種類のUNetライクモデルのアンサンブル。TensorRTによる高速化。
* **アーキテクチャ:**
    * yu4uモデル: 2.5D UNet (2D Backbone + Depth Pooling)。最終層はPixel Shuffleで出力。ConvNeXt Nano backbone。
    * tattakaモデル: 2.5D UNet (ResNetRS-50 backbone)。Depth方向にPooling/Conv。DecoderにJoint Pyramid Upsampling (JPU) とSESC Attention。
* **アルゴリズム:** MSEベースのバランス損失 (陽性/陰性サンプルを別々に計算し合計)。
* **テクニック:**
    * **ラベル表現:** 粒子中心を1.0とするガウシアンヒートマップ (sigma=6 または粒子サイズ依存)。座標変換オフセット1.0を使用。
    * **学習:** 4 Fold CV?
    * **アンサンブル:** yu4uモデル(4 fold)とtattakaモデル(3 fold)の計7モデルをアンサンブル。
    * **推論:** TensorRTでモデルを高速化。マルチGPU + マルチプロセスで並列処理。Sliding window inference。
    * **後処理:** Non-Maximum Suppression (Max Pooling kernel=7) でローカル最大値を検出。クラス別閾値でフィルタリング。座標変換（中心補正、オフセット補正、スケール変換）。

**[5位](https://www.kaggle.com/competitions/czii-cryo-et-object-identification/discussion/561580)**

* **アプローチ:** DeepFinder風の軽量3D CNNアーキテクチャ。4シード学習のアンサンブル。CC3D + サイズフィルタリングによる後処理。
* **アーキテクチャ:** DeepFinderベースのカスタム3D CNN。チャンネル数削減 (28, 32, 36)。Trilinear補間によるUpsample/Downsample。最終層のみTransposeConv。初期層にBatchNorm3d。
* **アルゴリズム:** Label Smoothing Cross Entropy Loss (smoothing=0.01)。Adam (lr=1e-4, betas=(0.9, 0.999))。
* **テクニック:**
    * **データ:** 全7 volumeで学習。ランダムパッチサンプリング (128^3)。
    * **ラベル表現:** 半径 `log2(r)*0.8` の球体マスク。
    * **正規化:** (5, 99) パーセンタイルでクリップ後、MinMax正規化。
    * **Augmentation:** Flip (xyz), Rotate (z軸90/180/270度), Mean/Std Shift。
    * **学習:** FP16 + Gradient Clipping。
    * **推論:** TTA (Flip x3, Rotate x3)。Sliding Window (overlapあり)。
    * **後処理:** クラス別閾値で二値化 → CC3D → 訓練マスクの1/7以下のサイズのクラスターを除去。

**[6位](https://www.kaggle.com/competitions/czii-cryo-et-object-identification/discussion/561518)**

* **アプローチ:** 2.5D UNetと3D UNet/SegResNetのアンサンブル。シミュレーションデータでの事前学習。FocalTversky++損失。TensorRTによる高速化。
* **アーキテクチャ:** 2.5D UNet (EfficientNet-B2/V2-B2, ConvNeXt-Nano, ResNet34d encoder), MONAI 3D UNet, MONAI SegResNet。
* **アルゴリズム:** FocalTversky++ Loss。AdamW + Cosine Annealing LR。
* **テクニック:**
    * **事前学習:** シミュレーションデータ (polnetで自作？形状をターゲットに似せる) で事前学習。
    * **データ:** 7 Fold CV。
    * **学習:** 入力パッチ 64x128x128。前処理はOutlier除去 + MinMax正規化。ラベルは半径x0.5の球体マスク。
    * **損失関数:** FocalTversky++ がBCE/Focal/Dice/Tversky単体より優れていた（予測値が0/1に偏りにくい）。
    * **アンサンブル:** 10モデルを平均アンサンブル。
    * **推論:** Sliding Window (overlap 0.25)。TTA。エッジ8%の予測を破棄。TensorRTで高速化。マルチGPU+マルチプロセス。
    * **後処理:** クラス別閾値で二値化 → CC3D → 重心計算。

**[7位](https://www.kaggle.com/competitions/czii-cryo-et-object-identification/discussion/561447)**

* **アプローチ:** 3D U-Netベースモデルにガウシアンヒートマップ予測。シミュレーションデータでの事前学習。重み付きBCE Loss。重み付きボックス融合 (WBF) による後処理。
* **アーキテクチャ:** U-Net (ResNet50d, EfficientNetV2-M backbone), DeepLabV3+ (ResNet50d backbone)。一部モデルは高解像度特徴マップのみ使用。
* **アルゴリズム:** 重み付きBCE Loss。AdamW? ModelEMA (decay=0.995)。
* **テクニック:**
    * **事前学習:** シミュレーションデータ全体で事前学習（WBPデータをValidationに使用）。
    * **データ:** 4 Fold CV? Sliding windowで64個のパッチを生成、学習時はランダムシフト。
    * **ラベル表現:** ガウシアンヒートマップ。
    * **損失:** 重み付きBCE。クラスごとに重み設定（困難クラス重視）。陽性ピクセルにもクラスごとに重み設定。
    * **Augmentation:** Shift, CutMix, Mixup, Flip(xyz), Rot90(xy), Affine(事前学習時のみ), Contrast, Gamma, Gaussian Noise(FT時のみ)。
    * **推論:** 4x Sliding Window TTA。Flip TTA (8x or 3x)。Logit平均アンサンブル。
    * **後処理:** Gaussian Blur (k=5, s=1.0) → MaxPool (k=7) でローカル最大値検出 → **Weighted Box Fusion (WBF)** (半径0.5*粒子半径) で座標精製。クラス別閾値（ロジット空間）。

**[8位](https://www.kaggle.com/competitions/czii-cryo-et-object-identification/discussion/561515)**

* **アプローチ:** 4つの3D U-Netモデルスープのアンサンブル。シミュレーションデータでの事前学習。多様なトモグラムタイプでの学習。Watershedによる後処理。
* **アーキテクチャ:** MONAI 3D U-Net。異なるチャネル数/Residual Unit数設定。
* **アルゴリズム:** MONAI DiceCELoss。AdamW + LR ReduceOnPlateau。
* **テクニック:**
    * **データ:** 7 Fold CV (実験ID分割)。
    * **事前学習:** 6つのシミュレーションデータで事前学習（Gaussian Denoising適用）。
    * **学習:** パッチサイズ 128^3。RandCropByLabelClassesd, RandFlipd, RandRotate90dなどのAugmentation。異なるトモグラムタイプ（denoised, IsoNet-corrected, CTF-deconvolved）でも学習。EMA使用。
    * **モデルスープ:** 各Foldで学習したモデルの重みを平均化。
    * **アンサンブル:** 4つのモデルスープ（異なる設定、学習データ）をアンサンブル（Logitレベル）。
    * **推論:** Sliding Window (160x384x384, overlap 0.25, Gaussian)。TTA (Flip, Transpose)。
    * **後処理:** **Watershedセグメンテーション**を用いて接触/重なり粒子を分離。クラス別閾値でマスク生成 → 距離変換 → Local Maximaマーカー → Watershed。cucim/cupyで高速化。クラス別Blobサイズ閾値でFP除去。

**[9位](https://www.kaggle.com/competitions/czii-cryo-et-object-identification/discussion/561431)**

* **アプローチ:** 軽量3D ConvNeXtライクモデル + U-Net様Decoder。半径調整済みマスク。DBSCANによる後処理。
* **アーキテクチャ:** カスタム3D ConvNeXt Encoder (Stem 2x2, Kernel 3x3, Block数削減) + U-Net様Decoder (ConvBlock3D + Upsample)。
* **アルゴリズム:** BCE Loss。
* **テクニック:**
    * **データ:** 7 Fold CV?
    * **ラベル表現:** 粒子半径を調整（小粒子は/2、大粒子は/3）した球体マスク。
    * **学習:** 入力パッチ 32x256x256。正規化 (1-99パーセンタイルクリップ+MinMax)。Augmentation (Rot90, Flip xyz)。
    * **推論:** Sliding Window (32x320x320, z軸overlap 6)。TTA (Rot90 xy)。
    * **後処理:** クラス別閾値で二値化 → CC3D → **DBSCAN**で候補点クラスタリング/精製。
    * **アンサンブル:** 多数のモデル（Fold違い、パラメータ違い？）をアンサンブル。

**[10位](https://www.kaggle.com/competitions/czii-cryo-et-object-identification/discussion/561844)**

* **アプローチ:** MONAI 3D U-Netのアンサンブル。シミュレーションデータでの事前学習。KLダイバージェンス最小化に基づく後処理。
* **アーキテクチャ:** MONAI UNet (`channels=(48, 64, 80, 80, 128), stride_patterns=(2,2,2,1)`)。
* **アルゴリズム:** Tversky Loss + Multiclass CrossEntropy。AdamW + Cosine LR Scheduler。Optunaでハイパーパラメータ探索。
* **テクニック:**
    * **事前学習:** シミュレーションデータで事前学習。3種類の目的関数（クラス予測のみ、クラス+方向ベクトル予測、クラス+バグあり方向ベクトル予測）を試行。バグあり版が最も効果的だった。
    * **データ:** 7 Fold CV (実験ID分割)。
    * **学習:** Optunaで最適化。Mixup使用。
    * **アンサンブル:** 9つの3D UNetモデル（異なる事前学習、Fold、パラメータ）のLogit平均。
    * **後処理:** 閾値処理後、独自の**KLダイバージェンス最小化**に基づく手法で連結成分内の複数粒子を分離。粒子形状pdf(Q)をモデル予測pdf(P)上でKL(P||Q)が最小になるように配置。さらに、検出された境界上の点群から**Circumcenter**を計算し、そのクラスター中心を最終座標とする。この後処理でLB+0.015。
