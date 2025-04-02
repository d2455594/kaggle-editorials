---
tags:
  - Kaggle
  - UNet
startdate: 2023-03-16
enddate: 2023-06-15
---
# Vesuvius Challenge - Ink Detection
[https://www.kaggle.com/competitions/vesuvius-challenge-ink-detection](https://www.kaggle.com/competitions/vesuvius-challenge-ink-detection)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、紀元後79年のヴェスヴィオ火山噴火によって炭化した古代ヘルクラネウムの巻物から得られた**3D X線CTスキャンデータ**を用いて、**巻物内部に書かれたインクを検出する**機械学習モデルを開発することです。
* **背景:** ヘルクラネウムの巻物は、物理的に開くと崩れてしまうほど脆いため、未開封のままです。これらには失われた古代のテキストが含まれている可能性があり、その解読は歴史的に非常に重要です。高解像度のX線CTスキャン技術により巻物の内部構造をデジタルデータ化できますが、パピルスとインクの密度差が非常に小さいため、インクの識別は極めて困難です。この「ヴェスヴィオ・チャレンジ」は、AI技術を活用してこの難問に挑む、より大きな取り組みの一部です。
* **課題:** CTスキャンデータから、パピルス繊維と区別がつきにくい**微かなインクの痕跡**を正確に識別することです。データは大規模な3次元ボリュームであり、巻物の損傷や変形も考慮する必要があります。これは、本質的には3Dデータに対する**セグメンテーション（領域分割）タスク**であり、インクが存在するピクセル（ボクセル）を特定することが求められます。

**データセットの形式 (Dataset Format)**

提供される主なデータは、巻物のCTスキャンデータの一部（フラグメント）とその中のインク位置を示すラベル（マスク）です。

1.  **トレーニングデータ:**
    * 複数の巻物フラグメントから構成されます (`fragment_1`, `fragment_2`, `fragment_3`)。
    * 各フラグメントは、**3Dボリュームデータ**を表す一連の2D画像（通常はTIFF形式、65枚）のスライスとして提供されます。これらはCTスキャンの連続した断面画像です。
    * `train/{fragment_id}/surface_volume/`: 各フラグメントのCTスキャンデータ（入力画像）。フォルダ内に数十枚のTIFFファイルが含まれます。
    * `train/{fragment_id}/inklabels.png`: **ターゲット変数**となる、対応するインクの正解ラベルマスク（2値画像）。インクがあるピクセルが1、ないピクセルが0で示されます。
    * `train/{fragment_id}/mask.png`: 巻物領域を示すマスク。この領域内のみが評価対象となります。ピクセル値が255の部分が有効領域です。
    * `train/{fragment_id}/ir.png`: 赤外線写真。補助的な情報として利用可能。
2.  **テストデータ:**
    * トレーニングデータと同様の形式で、複数のフラグメントのCTスキャンデータ（TIFF画像のスライス）が提供されます (`test/{fragment_id}/surface_volume/`)。
    * インクの正解ラベル (`inklabels.png`) は提供されません。
    * 参加者は、このテストデータ内の指定された領域 (`mask.png`) に対してインクの位置を予測します。
3.  **`sample_submission.csv`**:
    * 提出フォーマットのサンプル。`Id`（フラグメントIDとピクセル位置を示す識別子）と `Predicted`（予測結果、ランレングスエンコーディング形式）の列を持ちます。

**評価指標 (Evaluation Metric)**

* **指標:** **F0.5スコア**
* **計算方法:** Fbetaスコアは、適合率（Precision）と再現率（Recall）の加重調和平均です。F0.5スコアは、β=0.5として計算され、適合率を再現率よりも重視します。
    * Precision = TP / (TP + FP)
    * Recall = TP / (TP + FN)
    * F0.5 = (1 + 0.5²) * (Precision * Recall) / ((0.5² * Precision) + Recall) = 1.25 * (Precision * Recall) / (0.25 * Precision + Recall)
    （TP: 真陽性、FP: 偽陽性、FN: 偽陰性）
* **意味:** モデルが予測したインク領域と実際のインク領域の一致度を測ります。F0.5スコアは、**偽陽性（インクでない箇所をインクと予測してしまうこと）を低く抑えること**（高い適合率）を特に重視する指標です。巻物の解読において、誤ってインクと判定された箇所が多いとテキストの解釈を誤らせる可能性があるため、適合率の高さがより重要視されていると考えられます。スコアは**高い**ほど、より正確に（特に偽陽性を少なく）インクを検出できていると評価されます。

---

**全体的な傾向**

ヴェスヴィオ火山の噴火で炭化した巻物からインク（文字）を検出するこのセグメンテーションタスクでは、3次元のX線CTボリュームデータを扱うため、**3D CNN**や、複数スライスをチャンネルや時間軸として入力する**2.5D CNN**アプローチが上位解法で広く採用されました。特に**U-Netベースのアーキテクチャ**が主流であり、エンコーダーには3D ResNet系（ir-CSN, ResNet3Dなど）、SegFormer、EfficientNet、MaxViT、ConvNeXt、MViTv2などが多様に用いられました。

データの特性として、**z軸（深さ方向）のインク位置がフラグメント間で異なる**こと、アノテーションにノイズが含まれること、また一部で**ラベル（マスク）と画像の0.5ピクセルのずれ**が報告されたことが挙げられます。これらに対処するため、**深さ方向に不変 (depth-invariant)** なモデル構造の工夫（3D Encoder + 2D Decoder、Depth Pooling、Attentionなど）、ソフトターゲット（複数アノテーションの平均）の使用、ピクセルずれ補正、信頼性の高いCV（交差検証）戦略の構築（例: fragment 2を分割）、ノイズに頑健な損失関数やAugmentationの選択などが重要なテクニックとなりました。

学習においては、**大きなパッチサイズ**や高解像度入力、EMA（指数移動平均）やSWA（確率的重み平均化）、強力な正則化（Augmentation、Dropout、Label Smoothing、CutMix/MixUp）、段階的学習（低解像度→高解像度）、**疑似ラベル**の活用などが有効でした。入力スライス数の選択（16〜48枚程度）も重要でした。

推論では、**TTA（Test Time Augmentation）**（回転、フリップ）、複数モデルのアンサンブル、**閾値の最適化**（固定値、パーセンタイル）、**後処理**（小領域除去など）がスコア向上に貢献しました。特に閾値設定はスコアへの影響が大きく、慎重な調整が求められました。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/vesuvius-challenge-ink-detection/discussion/417496)**

* **アプローチ:** 3Dモデル (UNETR) + 2Dモデル (SegFormer) の2段階構成。大きなクロップサイズ (1024x1024)。複数モデルのアンサンブル。
* **アーキテクチャ:** Stage 1: 3D UNETR。Stage 2: SegFormer-b5。
* **アルゴリズム:** BCE + Dice Loss。AdamW + Gradual Warmup Scheduler。SWA。
* **テクニック:**
    * **データ処理:** 中央16スライスを入力。大きなクロップサイズ (1024x1024)。ブランクでないクロップのみ使用。
    * **モデル構造:** 3D UNETRで3D情報を集約し、中間特徴量 (32ch) を生成。これをZ軸方向に最大値プーリングで2D特徴量 (32ch, 1024x1024) に変換し、SegFormer-b5に入力して最終的なセグメンテーションを行う。この構造が深さ不変性を実現。
    * **Augmentation:** フリップ、90度回転、輝度コントラスト、Channel Dropout、ShiftScaleRotate、ノイズ、ブラー、Coarse Dropout、Grid Distortion。
    * **学習:** Fragment 1を検証用とし、2, 3で学習後、全データで再学習。
    * **アンサンブル:** 9つのモデル（異なる1st/2nd stageモデル、異なる解像度、SWA有無など）をアンサンブル。
    * **後処理:** 閾値 (0.5) で二値化後、10000ピクセル未満の連結成分を除去。

**[2位](https://www.kaggle.com/competitions/vesuvius-challenge-ink-detection/discussion/417255)**

* **アプローチ:** 2.5Dおよび3D CNN。アップサンプリングなしの低解像度(1/32)予測。強い正則化。パーセンタイル閾値。
* **アーキテクチャ:**
    * 2.5D: 2D CNN Backbone (ResNetRS, ConvNeXt, SwinV2, ResNeXtなど) + Pointwise Conv2D Neck + 3D CNN (ResBlockCSN)。
    * 3D: 3D CNN Backbone (ResNet50/152-irCSN) + Pooling + Pointwise Conv2D。Decoderはシンプル。
* **アルゴリズム:** BCE + Global Fbeta Loss。Label Smoothing。CutMix, MixUp, Manifold MixUp。EMA。TTA (flip, 時間ストライド毎切替)。Percentile Threshold。
* **テクニック:**
    * **データ:** Group K-Fold (Fragment 2を3分割)。入力解像度256x256 (2.5D) or 192x192 (3D)。ターゲットも1/32にダウンサンプル。高速化のためパッチをnpyで保存。サンプル毎正規化。
    * **入力層:** 中央35層または27層を使用。
    * **学習:** AMP、EMA、Label Smoothing、Drop Path、MixUp系。強力なAugmentation (Flip, Rotate, ShiftScaleRotate, Elastic, GaussianBlur, GaussNoise, OpticalDistortion, GridDistortion, PiecewiseAffine, BrightnessContrast, Cutout, Channel Shuffle, Z軸シフト)。
    * **推論:** FP16。ストライド32。TTA (フリップ、時間ストライド毎切替)。パーセンタイル閾値(0.9, 0.93)を適用し、常に一定割合の陽性ピクセルを予測。マスク外領域除外。

**[3位](https://www.kaggle.com/competitions/vesuvius-challenge-ink-detection/discussion/417536)**

* **アプローチ:** 3D Encoder (ir-CSN) + 2D Decoder (U-Net)。チャンネル数と解像度を変えたモデルのアンサンブル。
* **アーキテクチャ:** 3D ir-CSN (ResNet50, ResNet152) Encoder + Mean Pooling (Z軸方向) + 2D U-Net Decoder。
* **アルゴリズム:** BCE + Dice Loss (重み付き平均)。AdamW + OneCycle Scheduler。EMA。
* **テクニック:**
    * **データ分割:** Fragment 2を3分割し、5 Fold CV。
    * **入力:** 24, 32, 48スライスを試行。24スライスが主。パッチサイズは224x224, 384x384, 576x576, 640x640を実験。大きいサイズは過学習傾向。クロップStrideはサイズに応じて調整。
    * **特徴量入力:** 3スライスを重ねて3チャンネル入力とし、事前学習済み重みを活用。
    * **Augmentation:** Tanaka氏ベースライン + rotate_limit=180。MixUp, CutMixも試行。
    * **学習:** クリーンな入力（全ゼロでない）のみ使用。DDP、勾配蓄積を使用（大解像度時）。
    * **推論:** 小さいStrideを使用。TTAなし（回転TTAは効果薄）。閾値0.5固定。
    * **アンサンブル:** 異なる入力サイズ（224, 384, 576）、異なるバックボーン（r50, r152）、異なるFoldの計15モデルをアンサンブル。

**[4位](https://www.kaggle.com/competitions/vesuvius-challenge-ink-detection/discussion/417779)**

* **アプローチ:** 3D ResNet/ResNeXt Encoder + U-Net様Decoder。時間方向のランダムクロップ/ペースト/カットアウトAugmentation。低FP重視のチェックポイント選択。
* **アーキテクチャ:** 3D ResNet (152, 200), 3D ResNeXt101 Encoder + U-Net様Decoder。
* **アルゴリズム:** CrossEntropyLoss (Label Smoothing 0.3)。AdamW + Cosine Annealing with Warmup。CutMix。
* **テクニック:**
    * **データ:** 4 Fold CV (Fragment 2を2分割)。入力256x256、22スライス (21-42層)。
    * **独自Augmentation:** Temporal Random Crop & Paste & Cutout。入力22層からランダムに12-22層をクロップし、ランダムな位置にペースト。さらに0-2層をランダムにゼロ埋め（Temporal Cutout）。これにより深さ方向の汎化を促進。
    * **チェックポイント選択:** CVスコアだけでなく、**偽陽性(FP)が少ない**エポックの重みを選択・アンサンブルする戦略。
    * **画像Augmentation:** フリップ、回転(360°)、輝度コントラスト、ガンマ、ノイズ、ブラー、ShiftScaleRotate、CoarseDropout。
    * **データ前処理:** ピクセル値を[50, 200]にクリッピング。
    * **アンサンブル:** 3種類の3Dバックボーン x 4 Foldのモデルをアンサンブル。

**[5位](https://www.kaggle.com/competitions/vesuvius-challenge-ink-detection/discussion/417642)**

* **アプローチ:** 3D ResNetモデルのアンサンブル。Mosaic Augmentationを含む強力なAugmentation。
* **アーキテクチャ:** 3D ResNet18, 3D ResNet34 + U-Net様Decoder?
* **アルゴリズム:** BCE Loss。AdamW + GradualWarmupScheduler。
* **テクニック:**
    * **データ:** 4 Fold CV (Fragment 2を2分割)。入力256x256、32スライス (16-48層)。
    * **Augmentation:** **Mosaic Augmentation** を適用。さらに多様なAlbumentations（フリップ、回転、輝度、ノイズ、ブラー、歪み、ShiftScaleRotate、CoarseDropout）を適用。
    * **学習:** 50 epochs。FP16。
    * **推論:** ResNet18 (Fold 1, 2a) + ResNet34 (4 Fold) の計6モデルをアンサンブル。タイルサイズ1024x1024、Stride 512。タイルの端に近い予測を除外。閾値0.5。TTA (4回転、フリップ)。後処理でノイズ除去。

**[6位](https://www.kaggle.com/competitions/vesuvius-challenge-ink-detection/discussion/417274)**

* **アプローチ:** EfficientNet (U-Net Decoder) と SegFormerのアンサンブル。IR画像を補助入力として利用。推論時の回転とパーセンタイル閾値。
* **アーキテクチャ:** EfficientNet-B6/B7 + U-Net Decoder、SegFormer-B3。
* **アルゴリズム:** SoftBCEWithLogitsLoss。Adam + Cosine Schedule with Warmup。
* **テクニック:**
    * **特徴量:** CT画像スライスに加え、**赤外線(IR)画像**をチャンネルとして追加（CNNモデル）。SegFormerは3チャンネル入力のため、異なるスライス組み合わせ（中央のみ、中央+IR、中央+平均など）で学習し平均。
    * **データ分割:** 7-Fold または 10-Fold CV。Fold数を増やすとCV/LBともに向上。
    * **推論:** テストデータが回転していると仮定し、**推論時に入力画像を90度回転**させて予測し、結果を逆回転させる。TTA (Horizontal Flip)。
    * **閾値:** パーセンタイル閾値を使用（Public LBでは0.96、Private LBでは0.95が最適だった）。閾値の選択に注意。
    * **アンサンブル:** 複数のEfficientNetモデルとSegFormerモデルの予測値を平均。

**[7位](https://www.kaggle.com/competitions/vesuvius-challenge-ink-detection/discussion/417430)**

* **アプローチ:** nnU-Netフレームワークを利用。3D Encoder + 2D Decoderアーキテクチャ。SEブロックによるスライス重み付け。
* **アーキテクチャ:** nnU-Netベースのカスタムアーキテクチャ。3D Encoder（各ステージ4層Conv）+ 2D Decoder（各ステージ2層Conv）。Skip Connectionに**Squeeze-and-Excitation (SE) ブロック**を適用（チャンネル方向ではなく**深さ(Z)方向**に適用し、スライスごとの重要度を学習）。
* **アルゴリズム:** nnU-Netデフォルト（Dice+CE Loss?）。AdamW?
* **テクニック:**
    * **データ分割:** 各Fragmentを前景領域が均等になるように25分割し、5 Fold CV。
    * **入力:** 32スライスを選択（強度分布の中央付近）。パッチサイズ 32x512x512。FragmentごとのZ-score正規化。強度値を0.5-99.5パーセンタイルでクリッピング。
    * **Augmentation:** nnU-Netデフォルト拡張（回転、スケール、ノイズ、ブラー、輝度/コントラスト、ガンマ、ミラーリング）を使用。回転はXY平面内のみ。
    * **学習:** バッチサイズ2または4。Weight Decay 3e-5または1e-4。
    * **アンサンブル:** 性能の良かった2 Fold x 2モデル（異なるバッチサイズ/WD）の重みをアンサンブル。
    * **TTA:** 全軸ミラーリング + XY平面内90度回転 (計8パターン)。
    * **後処理:** ①閾値を0.6に上げ、さらに連結成分解析で95パーセンタイル値<0.8の成分を除去。②予測結果にさらに2D U-Net (入力2048x2048) を適用してリファイン。①の方がPrivate LBは良かった模様。

**[8位](https://www.kaggle.com/competitions/vesuvius-challenge-ink-detection/discussion/417383)**

* **アプローチ:** 3D ResNet/SEResNet Encoder + U-Net Decoder。複数モデルのアンサンブル。チャンネル方向のTTA。
* **アーキテクチャ:** 3D SEResNet101, 3D ResNet34 Encoder + U-Net Decoder。SegFormer-b3も使用。
* **アルゴリズム:** BCE + Dice Loss。
* **テクニック:**
    * **データ:** 3-Fold CV (Fragment 1, 2, 3) または 4-Fold CV (Fragment 2を上下分割)。
    * **入力:** ランダムに20チャンネル (スライス) を15-40層の間から選択。解像度は192x192, 224x224, 256x256など複数試行。
    * **Augmentation:** 異なる解像度でのリサイズ、Z軸方向の圧縮、3D回転、Channel Dropout、フリップ、回転、Albumentations (輝度コントラスト、歪み、ノイズ、ブラーなど)。
    * **学習:** 3D SEResNet101は2段階学習（低解像度→高解像度）。
    * **推論:** **チャンネルTTA:** 推論時に15-40層を5層ずつスライドさせながら20チャンネルウィンドウで複数回予測し平均。Rotation TTA (90度)。FP16。
    * **後処理:** 推論時にエッジピクセルを除外。閾値は固定 (0.55) と動的 (パーセンタイル3%) の2通りで提出。
    * **アンサンブル:** 5つのモデル（3D SEResNet101, SegFormer x2, 3D ResNet34 x2）の予測を重み付き平均。

**[9位](https://www.kaggle.com/competitions/vesuvius-challenge-ink-detection/discussion/417361)**

* **アプローチ:** 3D CNN + 2D DecoderとTransformerベースモデルのアンサンブル。
* **アーキテクチャ:**
    * ResNet34D + U-Net Decoder (3D CNN + 2D Decoder)。
    * PVTv2-B3 + DAFormer Decoder (Transformerベース)。
* **アルゴリズム:** 不明 (BCE + Diceなどか？)。
* **テクニック:**
    * **データ分割:** 2 Fold CV (Validation: fragment-1, fragment-2aa)。
    * **入力:** ResNet34Dは256x256, 32スライス。PVTv2は384x384, 16スライス。
    * **アンサンブル:** 上記2モデル x 2 Foldの計4モデルをアンサンブル。

**[10位](https://www.kaggle.com/competitions/vesuvius-challenge-ink-detection/discussion/417363)**

* **アプローチ:** 3D Encoder + 2D FPN Decoder。Attention Pooling。インク領域と非インク領域のサンプリング比調整。デノイザー追加。
* **アーキテクチャ:** 3D ResNet Encoder + Attention Pooling (Z軸方向) + 2D FPN Decoder。追加のデノイズ用U-Net。
* **アルゴリズム:** BCE Loss。AdamW。
* **テクニック:**
    * **事前学習:** ink-idのInkClassifier3DCNNで3D Encoderを事前学習（CVが0.03-0.06向上）。
    * **データ分割:** 5 Fold (Fragment 2を3分割)。
    * **入力:** 224x224、16スライス (20-36層)。
    * **サンプリング:** インクを含む領域1に対して、全くインクを含まない領域を8の比率でサンプリング。
    * **Attention Pooling:** Z軸方向に適用するシンプルなAttention Pooling層を自作。
    * **Denoiser:** 最終出力マスクからノイズ成分を予測するU-Net (tu-resnet10t encoder) を追加し、元の予測から差し引く。
    * **Augmentation:** 多様なAlbumentations。
    * **推論:** TTA (3 x rot90 + オリジナル)。閾値はCVに基づいて決定（パーセンタイル93%は不採用）。

