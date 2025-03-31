---
tags: 
startdate: 
enddate:
---
# Vesuvius Challenge - Ink Detection
https://www.kaggle.com/competitions/vesuvius-challenge-ink-detection

**概要 (Overview)**

- **目的:** このコンペティションの目的は、紀元後79年のヴェスヴィオ火山噴火によって炭化した古代ヘルクラネウムの巻物から得られた**3D X線CTスキャンデータ**を用いて、**巻物内部に書かれたインクを検出する**機械学習モデルを開発することです。
- **背景:** ヘルクラネウムの巻物は、物理的に開くと崩れてしまうほど脆いため、未開封のままです。これらには失われた古代のテキストが含まれている可能性があり、その解読は歴史的に非常に重要です。高解像度のX線CTスキャン技術により巻物の内部構造をデジタルデータ化できますが、パピルスとインクの密度差が非常に小さいため、インクの識別は極めて困難です。この「ヴェスヴィオ・チャレンジ」は、AI技術を活用してこの難問に挑む、より大きな取り組みの一部です。
- **課題:** CTスキャンデータから、パピルス繊維と区別がつきにくい**微かなインクの痕跡**を正確に識別することです。データは大規模な3次元ボリュームであり、巻物の損傷や変形も考慮する必要があります。これは、本質的には3Dデータに対する**セグメンテーション（領域分割）タスク**であり、インクが存在するピクセル（ボクセル）を特定することが求められます。

**データセットの形式 (Dataset Format)**

提供される主なデータは、巻物のCTスキャンデータの一部（フラグメント）とその中のインク位置を示すラベル（マスク）です。

1. **トレーニングデータ:**
    
    - 複数の巻物フラグメントから構成されます。
    - 各フラグメントは、**3Dボリュームデータ**を表す一連の2D画像（通常はTIFF形式）のスライスとして提供されます。これらはCTスキャンの連続した断面画像です。
    - `train/{fragment_id}/surface_volume/` : 各フラグメントのCTスキャンデータ（入力画像）。数十枚のTIFFファイルが含まれます。
    - `train/{fragment_id}/inklabels.png`: **ターゲット変数**となる、対応するインクの正解ラベルマスク（2値画像）。インクがあるピクセルが1（または255）、ないピクセルが0で示されます。
    - `train/{fragment_id}/mask.png`: 巻物領域を示すマスク。この領域内のみが評価対象となります。
2. **テストデータ:**
    
    - トレーニングデータと同様の形式で、複数のフラグメントのCTスキャンデータ（TIFF画像のスライス）が提供されます (`test/{fragment_id}/surface_volume/`)。
    - インクの正解ラベル (`inklabels.png`) は提供されません。
    - 参加者は、このテストデータ内の指定された領域 (`mask.png`) に対してインクの位置を予測します。
3. **`sample_submission.csv`**:
    
    - 提出フォーマットのサンプル。通常、`Id`（フラグメントIDとピクセル位置を示す識別子）と `Predicted`（予測結果、通常はランレングスエンコーディング形式）の列を持ちます。

**評価指標 (Evaluation Metric)**

- **指標:** **F0.5スコア**
- **計算方法:** Fbetaスコアは、適合率（Precision）と再現率（Recall）の加重調和平均です。F0.5スコアは、β=0.5として計算され、適合率を再現率よりも重視します。
    - Precision = TP / (TP + FP)
    - Recall = TP / (TP + FN)
    - F0.5 = (1 + 0.5²) * (Precision * Recall) / ((0.5² * Precision) + Recall) （TP: 真陽性、FP: 偽陽性、FN: 偽陰性）
- **意味:** モデルが予測したインク領域と実際のインク領域の一致度を測ります。F0.5スコアは、**偽陽性（インクでない箇所をインクと予測してしまうこと）を低く抑えること**（高い適合率）を特に重視する指標です。巻物の解読において、誤ってインクと判定された箇所が多いとテキストの解釈を誤らせる可能性があるため、適合率の高さがより重要視されていると考えられます。スコアは**高い**ほど、より正確に（特に偽陽性を少なく）インクを検出できていると評価されます。

要約すると、このコンペティションは、X線CTスキャンされた古代巻物の3Dデータから微かなインクを検出するセグメンテーションタスクです。データはCT画像スライスと正解マスクで構成され、性能は適合率を重視したF0.5スコア（高いほど良い）によって評価されます。

---

**全体的な傾向**

ヴェスヴィオ火山の噴火で炭化した巻物からインク（文字）を検出するこのセグメンテーションタスクでは、3次元のX線CTボリュームデータを扱うため、3D CNNや、複数スライスをチャンネルや時間軸として入力する2.5Dアプローチが上位解法で広く採用されました。特に**U-Netベースのアーキテクチャ**が主流であり、エンコーダーには3D ResNet系（ir-CSN, ResNet-C3Dなど）、SegFormer、EfficientNet、MaxViT、CoAt、Swin Transformerなど多様なモデルが用いられました。

データの特性として、**z軸（深さ方向）のインク位置がフラグメント間で異なる**こと、アノテーションにノイズが含まれること、また一部で**ラベル（マスク）と画像の0.5ピクセルのずれ**が報告されたことが挙げられます。これらに対処するため、深さ方向に不変なモデル構造の工夫（プーリング、Attention）、ソフトターゲット（複数アノテーションの平均）の使用、ピクセルずれ補正、信頼性の高いCV（交差検証）戦略の構築、ノイズに頑健な損失関数やAugmentationの選択などが重要なテクニックとなりました。

学習においては、大きなパッチサイズや高解像度入力、EMA（指数移動平均）やSWA（確率的重み平均化）、強力な正則化（Augmentation、Dropout、Label Smoothing）、段階的学習などが有効でした。推論では、TTA（Test Time Augmentation）、複数モデルのアンサンブル、閾値の最適化（固定値またはパーセンタイル）、後処理（小領域除去など）がスコア向上に貢献しました。

**各解法の詳細**

**1位**

- **アプローチ:** U-Net + MaxViT。ラベルの0.5ピクセルずれを発見し、対称化ラベルで学習後、小さなCNNで補正する独自手法。単一時刻と4時刻パネル入力モデルのアンサンブル。
- **アーキテクチャ:** U-Net + MaxViT_tiny_tf_512 Encoder。ずれ補正用Conv5x5。
- **アルゴリズム:** BCE Loss。AdamW。Cosine Annealing w/ Warmup。8-TTA (rot90, flip)。閾値最適化。連結成分除去。
- **テクニック:**
    - **ラベル処理:** ソフトターゲット（アノテーション平均）。0.5ピクセルシフトした対称化ラベル(y_sym)でU-Net学習。学習済み特徴量から小さなCNNで元の右下ズレ(y)へマッピング学習。
    - **入力:** Ash color画像。高解像度(1024x1024)。単一時刻(t=4) / 4時刻(t=1-4)パネル入力。
    - **Augmentation:** RandomRotate90, HorizontalFlip, ShiftScaleRotate（ラベルずれ補正により有効化）。
    - **アンサンブル:** 単一時刻モデルと4時刻パネルモデルの5-Fold平均 x 8TTA結果を重み付き平均。

**2位 (tattaka & mipypf part)**

- **アプローチ:** 2.5D/3D CNN。アップサンプリングなしの低解像度(1/32)予測。強い正則化。パーセンタイル閾値。
- **アーキテクチャ:** 2.5D: 2D CNN Backbone (ResNetRS, ConvNeXt, SwinV2など) + Pointwise Conv2D Neck + 3D CNN (ResBlockCSN)。3D: 3D CNN Backbone (ResNet50/152-irCSN)。Decoderはシンプル。
- **アルゴリズム:** BCE + Global Fbeta Loss。Label Smoothing。CutMix, MixUp, Manifold MixUp。EMA。TTA (flip, 時間ストライド毎切替)。Percentile Threshold。
- **テクニック:**
    - **データ:** Group K-Fold (Fragment 2分割)。低解像度(256^2/192^2)入力、ターゲットも1/32にダウンサンプル。サンプル毎正規化。高速化のためパッチ保存。
    - **入力層:** 中間35層または27層を使用。
    - **学習:** AMP、EMA、Label Smoothing、Drop Path、MixUp系。重度Augmentation (Flip, Rotate, ShiftScaleRotate, Elastic, Blur, Noise, Distortion, BrightnessContrast, Cutout, Channel Shuffle, Z軸シフト)。
    - **推論:** FP16。ストライド32。TTA。パーセンタイル閾値(0.9, 0.93)。マスク外領域除外。

**2位 (yukke42 part)**

- **アプローチ:** 3D CNN/Transformer Encoder + シンプルな2D/1D Decoder。低解像度予測。
- **アーキテクチャ:** Encoder: 3D CNN (ResNet18/34ベース), MViTv2-s。Decoder: 単層2D CNN or Linear + Patch Expanding。
- **アルゴリズム:** BCE (+Dice?)。Label Smoothing。CutMix, MixUp。
- **テクニック:**
    - **データ:** Fragment 2を垂直分割した4-Fold CV。
    - **モデル:** 3D CNNはMaxPool除去、Attention追加。MViTv2は各スケール出力を利用。Decoderはシンプル化。出力は1/2 or 1/4解像度→Bilinearでアップサンプル。
    - **学習:** AMP、torch.compile。Label Smoothing、Cutout、CutMix、MixUp。Augmentation (Flip, Rotate, ShiftScaleRotateなど)。Patch Size 224, Stride 112。
    - **推論:** FP16。Stride 75。予測のエッジ部分を無視。

**3位**

- **アプローチ:** 3Dエンコーダ(ir-CSN) + 2D U-Netデコーダ。複数クロップサイズ推論のアンサンブル。
- **アーキテクチャ:** Encoder: ir-CSN-r50/r152 (OpenMMLab)。Decoder: U-Net (軽量版、チャンネル256開始)。
- **アルゴリズム:** BCE + Hard Dice Loss。AdamW。OneCycle Scheduler。EMA。TTA (4回転)。
- **テクニック:**
    - **データ:** Fragment 2を3分割した5-Fold CV。入力層選択(24層 or 32層)。複数クロップサイズ(224, 384, 576, 640)。ストライドはサイズによる。入力正規化。ゼロ入力フィルタリング。
    - **入力処理:** 時間次元(z)をチャンネル次元に結合（オーバーラップあり）。
    - **学習:** Augmentation (Flip, Rotate, ShiftScaleRotateなど)、MixUp, CutMix。EMA。DDP、勾配蓄積（大規模パッチ）。
    - **推論:** 複数クロップサイズ(224, 384, 576)のモデルをアンサンブル。TTA (4回転)。閾値0.5。

**4位 (DS Annotation Experts)**

- **アプローチ:** 3Dモデルに独自の時間軸Augmentation (Random Crop & Paste) を適用。偽陽性の少ない重み選択。
- **アーキテクチャ:** 3D ResNet152, 3D ResNet200, 3D ResNeXt101 + U-NetライクDecoder。
- **アルゴリズム:** CrossEntropy Loss (Label Smoothing 0.3)。AdamW。Cosine Annealing w/ Warmup。CutMix。閾値最適化。
- **テクニック:**
    - **Temporal Augmentation:** 時間軸(z)でランダムに12-22層クロップ → ランダムな位置にペースト → ランダムに0-2層カットアウト。
    - **データ:** 4-Fold CV (Fragment 2分割)。入力層選択(21-42層)。画像クリッピング(50-200)。
    - **重み選択:** 各エポックで保存し、Validationスコアと偽陽性(FP)数を基に低FPの重みを選択。
    - **モデル:** Kinetics事前学習済み3D ResNet/ResNeXt利用。
    - **学習:** Label Smoothing、CutMix。
    - **Augmentation:** Flip, Rotate, RandomGamma, BrightnessContrast, Noise, Blur, ShiftScaleRotate, CoarseDropout。
    - **推論:** 複数モデルアンサンブル。閾値はCVで最適化（0.47または0.5）。

**5位**

- **アプローチ:** 3D ResNetモデル。Mosaic Augmentation。高解像度推論とTTA。
- **アーキテクチャ:** 3D ResNet18, 3D ResNet34。
- **アルゴリズム:** BCE Loss。AdamW。GradualWarmupScheduler。TTA (4回転 + h/v flip)。
- **テクニック:**
    - **データ:** 4-Fold CV (Fragment 2分割)。タイルサイズ256、ストライド128。入力32層(16-48)。空白タイル無視。
    - **Augmentation:** Mosaic Augmentation、Albumentations (Flip, Rotate90, ShiftScaleRotate, Elastic, Blur, Noise, Optical/Grid Distortion, PiecewiseAffine, BrightnessContrast)。
    - **学習:** 50エポック、FP16。
    - **推論:** タイルサイズ1024、ストライド512。TTA (4回転+Flip)。タイルのエッジ付近の予測を無視。閾値0.5。ノイズ除去。
    - **アンサンブル:** 2 Fold ResNet18 + 4 Fold ResNet34。

**6位**

- **アプローチ:** EfficientNet+UNetとSegFormerのアンサンブル。推論時の画像回転とパーセンタイル閾値が特徴。IR画像入力も試行。
- **アーキテクチャ:** CNN+UNet (EfficientNet_b6/b7_ns, EfficientNetV2_l)。 SegFormer (b3)。
- **アルゴリズム:** SoftBCEWithLogitsLoss。Adam。Cosine Schedule w/ Warmup。TTA (h flip)。Percentile Threshold。
- **テクニック:**
    - **推論フロー:** Fragment A, B結合→時計回り回転→推論(元+hflip TTA)→逆回転→分割・エンコード。
    - **入力:** IR画像(z>45平均)をチャンネル追加。SegFormerは3ch。
    - **閾値:** パーセンタイル閾値を使用（LB probingとValidationで0.96または0.95）。
    - **学習:** 高k-fold数(7, 10)が有効。勾配ノルムクリッピング。マスク外領域除外。
    - **アンサンブル:** 複数モデル(EffNet B6/B7/V2L, SegFormer B3)の平均。

**7位**

- **アプローチ:** nnU-Netフレームワークベースのカスタム3D Encoder + 2D Decoder U-Net。SEブロックによる深さ方向の重み付け。
- **アーキテクチャ:** 3D Encoder + 2D Decoder U-Net。Skip接続にSEブロック挿入（深さ次元Attention）。Encoderは各ステージ4 Conv、Decoderは2 Conv。
- **アルゴリズム:** Dice + BCE Loss (nnU-Net)。TTA (ミラーリング、90度回転)。後処理（連結成分解析、2D U-Netリファインメント）。
- **テクニック:**
    - **フレームワーク:** nnU-Net利用。
    - **データ準備:** Fragmentを25分割し5-Fold作成（前景ピクセル数均等化）。入力32層（強度分布の中間点周り）。Z-scoring正規化、パーセンタイルクリッピング、Fragment毎正規化。パッチサイズ 32x512x512。
    - **モデル:** 3D Encoder特徴マップをSEブロックで深さ方向に重み付けし平均化して2D Decoderへ。
    - **Augmentation:** nnU-Netデフォルト + 平面内回転、スケーリング、ノイズ、Blur、輝度/コントラスト、低解像度シミュレーション、ガンマ、ミラーリング。
    - **推論:** TTA (全軸ミラーリング + 90度回転、計8パターン)。
    - **後処理:** ①閾値上げ(0.6)+連結成分解析（Softmax 95パーセンタイル<0.8除去）、②後段2D U-Netによるリファインメント（高解像度入力）、の2種試行（Privateではマイナス効果）。
    - **アンサンブル:** 2 Fold x 2モデル（異なるBS/WD）のアンサンブル。

**8位**

- **アプローチ:** 3Dモデル(SEResNet, ResNet)と2Dモデル(SegFormer)のアンサンブル。時間軸方向TTA（Channel TTA）。固定/動的閾値。
- **アーキテクチャ:** 3D Encoder: 3D SEResNet101, 3D ResNet34。 2D Encoder: SegFormer-b3。 Decoder: UNet。
- **アルゴリズム:** BCE + Dice Loss。TTA (rot90, channel TTA)。固定閾値(0.55) or Percentile Threshold (上位3%)。
- **テクニック:**
    - **データ:** 3-Fold or 4-Fold CV。
    - **入力:** 3Dモデルは20ch入力(15-40)。SegFormerは3ch入力（11chをグループ化しAttention Pooling or 6chをグループ化しLogit平均）。
    - **前処理:** 最大ピクセル値クリッピング(0.78)。
    - **Augmentation:** リサイズ(3種)、Z次元圧縮、3D回転、Channel Dropout、Rotate/Flip、Albumentations。
    - **学習:** BCE+Dice損失。マスク外領域除外。
    - **推論:** FP16。TTA (90度回転)。Channel TTA（3D SEResNetのみ、Window 20, Stride 5でスライドし平均）。エッジピクセル除去。閾値（固定0.55 or 上位3%）。
    - **アンサンブル:** 5モデル（3D SEResNet101, SegFormer Attention/LogitAvg, 3D ResNet34 x2）の重み付き平均。

**9位**

- **アプローチ:** 2.5D U-Netモデルのアンサンブル。ResNet34dとPVTv2-b3エンコーダーを使用。
- **アーキテクチャ:** U-Net。Encoder: resnet34d, pvtv2-b3-daformer。
- **アルゴリズム:** BCE + Dice Loss (推定)。
- **テクニック:**
    - **データ:** Fragment 2を分割した2-Fold CV。入力32層または16層。
    - **モデル:** 2.5Dアプローチ。
    - **その他:** hengck氏の公開議論・コードを参考に実装。

**10位**

- **アプローチ:** 3D Encoder (ResNet3D) + 2D FPN Decoder。InkClassifier3DCNNでの事前学習。Attention Pooling。データサンプリング戦略。Denoiser後処理。
- **アーキテクチャ:** Encoder: ResNet3D (resnet10t?, InkClassifier3DCNNで事前学習)。 Decoder: FPN。 Pooling: Attention Pooling。 Denoiser: UNet (tu-resnet10t)。
- **アルゴリズム:** BCE Loss。AdamW。Attention Pooling。
- **テクニック:**
    - **事前学習:** InkClassifier3DCNNで3D Encoderを事前学習。
    - **データ:** 5-Fold CV (Fragment 2分割)。解像度224、ストライド56。入力16層(20-36層)。インク陽性:陰性=1:8でサンプリング。
    - **Pooling:** Depth方向のAttention Pooling（学習可能な重み）。
    - **Denoiser:** UNetベースのDenoiserで最終予測マスクからノイズ成分を減算。
    - **Augmentation:** Flip, Rotate, Affine, ToneCurve, BrightnessContrast, ShiftScaleRotate, Elastic, GridDistortion, Noise, Blur, MotionBlur, CoarseDropout, Normalize。
    - **推論:** TTA (rot90 x3 + original)。閾値はCVで決定。

