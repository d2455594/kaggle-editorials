---
tags:
  - Kaggle
  - ヘルスケア
  - AtCoder
startdate: 2023-05-23
enddate: 2023-08-01
---
# HuBMAP - Hacking the Human Vasculature
[https://www.kaggle.com/competitions/hubmap-hacking-the-human-vasculature](https://www.kaggle.com/competitions/hubmap-hacking-the-human-vasculature)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、高解像度の人体腎臓組織の顕微鏡画像 (Whole Slide Images - WSI) から、機能的な組織単位である血管構造（特に血液血管）を高精度に検出・セグメンテーションする機械学習モデルを開発することです。
* **背景:** HuBMAP (Human BioMolecular Atlas Program) は、人体の様々な組織における細胞レベルでの構造と機能をマッピングする大規模プロジェクトです。血管系の正確なマッピングは、腎臓の生理機能や疾患（腎臓病、高血圧など）の理解に不可欠ですが、手作業でのアノテーションは時間がかかり、主観性も伴います。AIによる自動化は、研究の加速と再現性の向上に貢献します。
* **課題:** WSIは非常に高解像度であり、データ量が膨大です。また、組織の染色状態や切片の厚さ、アーティファクトなどにより、画像の外観には大きなばらつきがあります。血管の形状やサイズも多様であり、他の組織構造との区別が難しい場合もあります。提供されるデータセットには、異なるソースからのデータやノイズの多いアノテーションが含まれており、これらを効果的に扱う必要があります。これは本質的には**インスタンスセグメンテーション**タスクです。

**データセットの形式 (Dataset Format)**

提供される主なデータは、WSIをタイル化した画像と、それに対応する血管などの構造物のポリゴン形式のアノテーションです。

1.  **トレーニングデータ:**
    * `train/`: WSIをタイル化した画像（.tiff形式、例: 512x512ピクセル）。複数のデータセット (dataset1, dataset2など) が存在し、それぞれ由来やアノテーション品質が異なる可能性があります。
    * `polygons.jsonl`: 各タイル画像内の構造物（blood\_vessel, glomerulus, unsure）のアノテーションデータ。各構造物はポリゴン（頂点座標のリスト）で表現されます。
    * `tile_meta.csv`: 各タイル画像に関するメタデータ（ソースWSI、座標など）。
2.  **テストデータ:**
    * `test/`: トレーニングデータと同様の形式のタイル画像（.tiff形式）。
    * アノテーションは提供されません。
    * PublicテストセットとPrivateテストセットで由来するWSIが異なる可能性が示唆されています。
3.  **`sample_submission.csv`**:
    * 提出フォーマットのサンプル。`id`（タイル画像のID）、`height`, `width`（画像の高さ・幅）、`rle`（予測された各血管インスタンスのマスクをランレングスエンコーディング形式で記述）の列を持ちます。

**評価指標 (Evaluation Metric)**

* **指標:** **Average Precision (AP) at IoU threshold 0.6** (一般的に `mAP@0.6` や `segm_mAP60` と呼ばれる)
* **計算方法:**
    1.  予測された各血管マスクと真の血管マスクとの間で Intersection over Union (IoU) を計算します。IoU = (重複領域の面積) / (結合領域の面積)。
    2.  IoUが閾値（このコンペでは0.6）を超えた予測を真陽性 (True Positive - TP) とします。閾値以下の予測や、真のマスクに対応しない予測は偽陽性 (False Positive - FP)。予測されなかった真のマスクは偽陰性 (False Negative - FN) となります。
    3.  異なる信頼度スコアで予測をランク付けし、Precision-Recallカーブを描画します。
    4.  Precision-Recallカーブ下の面積 (Average Precision - AP) を計算します。
    5.  最終的なスコアは、全てのテスト画像におけるAPの平均値（Mean Average Precision - mAP）となります。
* **意味:** 予測された血管マスクの位置と形状が、真のマスクとどれだけ正確に一致しているかを評価します。IoU閾値0.6は、比較的厳密な一致を要求することを示します。スコアは**高い**ほど、より正確に血管を検出・セグメンテーションできていると評価されます。

要約すると、このコンペティションは、腎臓組織のタイル画像から血管を検出・セグメンテーションするインスタンスセグメンテーションタスクです。データはタイル画像とポリゴン形式のアノテーションで構成され、性能はIoU閾値0.6における平均適合率（高いほど良い）によって評価されます。

---

**全体的な傾向**

この腎臓血管のインスタンスセグメンテーションタスクでは、高解像度画像の一部であるタイル画像を扱う必要がありました。上位解法の多くは、**Mask R-CNN**系のアーキテクチャ（Cascade R-CNN, HTC, DetectoRS, ViT-Adapterなど）を採用していました。物体検出モデル（YOLO系, RTMDetなど）を単独で、あるいはセグメンテーションモデルと組み合わせて使うアプローチも見られました。

**データの扱い**が非常に重要でした。

* **データセット:** 提供された複数のデータセット（特に高品質なdataset1とノイズが多い可能性のあるdataset2）をどう活用するかが鍵でした。dataset2で事前学習しdataset1でファインチューニングする**2段階学習**や、dataset1のみを使用する戦略がありました。
* **入力形式:** タイル境界の影響を軽減するため、隣接タイルを用いて**パディングされた大きな入力画像**（例: 768x768以上）を作成する手法が広く採用されました。
* **CV戦略:** Public LBが不安定（特にDilation問題のため）であったことから、**信頼できるCross Validation (CV)** の構築が重視されました。WSI（Whole Slide Image）単位での分割や、WSI内の位置に基づく分割が行われました。

**学習と推論のテクニック**も多様でした。

* **アーキテクチャ:** CNNベース（ResNeXt, ConvNeXtなど）とTransformerベース（Swin, CoAt, ViT-Adapterなど）の両方が使われました。
* **学習:** **EMA (Exponential Moving Average)** や **SWA (Stochastic Weights Averaging)** が有効でした。強力なAugmentation（幾何学的変換、色調変換（Stain Augmentation含む）、AutoAugmentなど）が不可欠でした。
* **アンサンブル:** 複数モデル（異なるアーキテクチャ、Fold、入力）のアンサンブルはスコア向上に必須でした。バウンディングボックスのアンサンブルには**WBF (Weighted Boxes Fusion)** がよく使われました。
* **後処理:** 小さなマスクや低信頼度マスクの除去、糸球体領域との重複に基づくフィルタリング、アンサンブル結果を用いたフィルタリング、スコアのリファイン（bboxスコア x マスクスコア）などが行われました。
* **Dilation問題:** 予測マスクをDilation（膨張）するとPublic LBが向上する現象がありましたが、多くの場合CVやPrivate LBではスコアが悪化するため、Dilationなしの提出が主流となりました。これはdataset2のアノテーション特性に起因する可能性が指摘されました。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/hubmap-hacking-the-human-vasculature/discussion/429060)**

* **アプローチ:** 物体検出モデルRTMDetを中心に、バウンディングボックス (bbox) 精度向上に注力。EMAが非常に有効。
* **アーキテクチャ:** RTMDet-x（ベースライン）、YOLOX-x、Mask R-CNN (HTC RoI head)。RTMDetに簡易的なマスクヘッドを追加。
* **アルゴリズム:** WBF (bboxアンサンブル用)。
* **テクニック:**
    * **学習:** EMAの使用（固定学習率と併用）。Dataset1とDataset2の両方を使用。Dataset1は'i'座標で分割 (Train/Val)。Dataset2のサンプル比率を多くする (5/8)。
    * **マスク教師:** マスクアノテーションをbbox精度向上のために利用（ランダム回転時のbbox再計算、マスクヘッド追加）。
    * **Augmentation:** 強力な幾何学的Augmentation (RandomRotateScaleCrop, Rot90, Flip)。
    * **推論:** 複数モデル（RTMDet x3, YOLOX-x, Mask R-CNN、各2 Fold）のbboxをWBFでアンサンブル。マスクはMask R-CNN (入力サイズ1440) のマスクヘッドから生成。TTA不使用。Dilation不使用。

**[2位](https://www.kaggle.com/competitions/hubmap-hacking-the-human-vasculature/discussion/429240)**

* **アプローチ:** 信頼性の高いCV構築と、多様なモデルによるアンサンブル。Dilationあり/なしの両方を提出。
* **アーキテクチャ:** Cascade Mask R-CNN (Swin-T, CoAt-Small, ConvNeXt-T/S バックボーン)。
* **アルゴリズム:** NMS (インスタンスフィルタリング用)。
* **テクニック:**
    * **CV:** Dataset1をWSIごと、さらに左右に分割して4 Fold作成。同じWSI由来のタイルがTrain/Valに含まれないようにStain Augmentation (`staintools`) でスタイル変換したデータを使用。
    * **入力:** 元のタイル、または隣接タイルで128ピクセルパディングしたタイルを使用。
    * **学習:** 2段階学習（Stage1: Dataset1(3 folds) + Dataset2、Stage2: Dataset1(3 folds) のみでファインチューン）。血栓 (blood vessel) クラスのみで学習。SWA (最終5チェックポイント)。
    * **Augmentation:** Stain Augmentation、強力な幾何学的・色調Augmentation、AutoAugment (DETR風)。
    * **後処理:** 糸球体内マスク除去、低信頼度マスク除去、ピクセル数/インスタンス数に基づくアンサンブルフィルタリング（Quantile Threshold使用）、小マスク除去。
    * **アンサンブル:** FoldモデルとFull dataモデルを組み合わせ。

**[3位](https://www.kaggle.com/competitions/hubmap-hacking-the-human-vasculature/discussion/430242)**

* **アプローチ:** Mmdetベースの最新アーキテクチャ (ViT-Adapter, CBNetV2, DetectoRS) を活用。Dataset2を事前学習に利用する2段階学習。
* **アーキテクチャ:** ViT-Adapter-L (x2), CBNetV2-Base, DetectoRS (ResNeXt-101-32x4d, ResNet50)。
* **アルゴリズム:** WBF (bboxアンサンブル用)、NMS (TTA用)。
* **テクニック:**
    * **学習:** 2段階学習 (Stage1: Dataset2で事前学習、高LR、軽Augmentation → Stage2: Dataset1でファインチューン、低LR、重Augmentation、高解像度)。Pseudo Labelも一部使用（効果は限定的）。
    * **入力:** 高解像度入力 (1200x1200, 1400x1400, 1600x1600, 2048x2048)。
    * **損失関数:** マスクヘッド損失の重みを増加 (HTCベースモデル)。
    * **最適化:** SGDまたはAdamW。Cosine Scheduler + Warmup。
    * **Augmentation:** 軽度 (Flip, AutoAugment(Shear, Translate, Color, Equalize), ShiftScaleRotate) と重度 (上記に加え、輝度/コントラスト/色相変換、PhotoMetricDistortion, MinIoURandomCrop, CutOut, Rotate, GridDistortion, OpticalDistortionなど)。
    * **TTA:** FlipベースTTA、マルチスケール推論 (ViT-Adapter)。
    * **アンサンブル:** 5モデルのWBF。
    * **後処理:** Erosion + Dilationによるマスク平滑化。

**[4位](https://www.kaggle.com/competitions/hubmap-hacking-the-human-vasculature/discussion/428994)**

* **アプローチ:** 複数アーキテクチャ（インスタンスセグメンテーション + 物体検出）のアンサンブル。Dataset1を重視したTrain/Val分割と2段階学習。
* **アーキテクチャ:** Cascade R-CNN (ResNeXt, RegNet), Mask R-CNN (Swin-T), HybridTaskCascade (Res2Net), YOLOv6m (物体検出)。
* **アルゴリズム:** WBF (bboxアンサンブル用)、単純平均 (マスクアンサンブル用)。
* **テクニック:**
    * **CV/データ:** Dataset1の一部 (84枚) をValidationとし、残りのDataset1 (338枚) + Dataset2 (1211枚) をTrainingに使用。重複アノテーションの除去。
    * **学習:** 血栓 (blood_vessel) と 不確定 (unsure) クラスのみで学習（糸球体は不使用）。2段階学習（Stage1: 全Trainデータ → Stage2: Dataset1のみでファインチューン）。
    * **最適化:** AdamW + Warmup。
    * **Augmentation:** マルチスケール学習 [(512^2) - (1024^2)] のみ（デフォルトAugmentationは不安定だったため削減）。
    * **アンサンブル:** 5モデル。RPNボックスとROIボックスそれぞれでWBFを実行。マスクは平均化。
    * **後処理:** 小マスク除去 (面積 < 80)。インスタンススコアのリファイン (`score_bbox * mean(mask[mask > 0.5])`)。

**[5位](https://www.kaggle.com/competitions/hubmap-hacking-the-human-vasculature/discussion/429873)**

* **アプローチ:** Dataset1のみを使用し、高解像度入力とHTCモデルに注力。単一Foldモデルで提出。
* **アーキテクチャ:** HTC-DB-B (HTC + Swin-B + DetectoRS)。HTC-DB-Lも試行。
* **アルゴリズム:** Weighted Cluster-NMS with DIoU (R-CNNのNMS代替)。
* **テクニック:**
    * **CV/データ:** Dataset1のみを使用。5 Fold分割。COCO事前学習重みを使用。
    * **学習:** Mmdet2使用。1x schedule + 1x SWA。
    * **入力:** マルチスケール学習 [(512^2) - (1536^2)]。
    * **Augmentation:** RandomFlip, Rotate90。他のAugmentation（Copy-Paste, Mixup, HSV, Cutoutなど）は効果なし。
    * **TTA:** マルチスケール [(640^2) - (1920^2)] + Flip (horizontal, vertical)。
    * **Dataset2/3の利用:** 事前学習やPseudo Labelは効果なし。
    * **アンサンブル:** FoldアンサンブルはCV/Public LBを低下させたが、Private LBは向上。

**[7位](https://www.kaggle.com/competitions/hubmap-hacking-the-human-vasculature/discussion/428295)**

* **アプローチ:** Mask R-CNN (HTC) モデルをDataset1, 2, 3 (Pseudo Label) で学習。アンサンブルと後処理。
* **アーキテクチャ:** Mask R-CNN (Swin Transformerバックボーン, HTC RoI head)。
* **アルゴリズム:** RPN/RoI headレベルでのアンサンブル。
* **テクニック:**
    * **学習:** 3段階パイプライン（1. Dataset1で学習 → 2. Dataset2, 3にPseudo Label付与 → 3. Dataset1 + Dataset2(元ラベル+Pseudo) + Dataset3(Pseudo)で再学習）。
    * **入力:** マルチスケール学習 (768-1536)。
    * **Augmentation:** RandomResize, Flip, Rot90, RandomBrightnessContrast, HueSaturationValue。
    * **TTA:** マルチスケール (1024, 1536) + Flip (hvflip)。
    * **アンサンブル:** RPN提案とRoI head出力の両方でアンサンブル（Sartoriusコンペの手法参照）。
    * **後処理:** Dilation、小マスク除去、糸球体内マスク除去。
    * **Dilation:** Dataset1のみだとDilationなしの方が良いが、Dataset2/3を含めるとDilationありがLBで良かった（差は縮小）。

**[8位](https://www.kaggle.com/competitions/hubmap-hacking-the-human-vasculature/discussion/429352)**

* **アプローチ:** 単一のYOLOv8x-segモデル。強力なAugmentationとマスク閾値の調整に注力。
* **アーキテクチャ:** YOLOv8x-seg。
* **アルゴリズム:** (YOLOv8デフォルト)
* **テクニック:**
    * **CV/データ:** 全学習データ (Dataset1+2) を使用。WSIに基づく2 Fold分割 (Fold1: WSI1,3 / Fold2: WSI2,4)。全3クラスで学習。
    * **学習:** YOLOv8設定。imgsz=512, batch=64, AdamW, Cosine LR。
    * **Augmentation:** 強力な設定 (HSV, degrees=45, translate=0.1, scale=0.5, shear=15, perspective=0, flipud=0.5, fliplr=0.5, mosaic=1.0, mixup=1/3, copy_paste=1/3)。
    * **マスク閾値:** 最適なマスク二値化閾値がFoldによって大きく異なることを発見 (例: 0.2 vs 0.5)。最終提出では2つの異なる閾値 (0.5と0.2) を使用。
    * **推論:** 入力サイズ768 (512より良好)。
    * **効果がなかったこと:** セマンティックセグメンテーションとの組み合わせ、高解像度学習 (768, 1024)、TTA(Rot90)。

**[9位](https://www.kaggle.com/competitions/hubmap-hacking-the-human-vasculature/discussion/428447)**

* **アプローチ:** 2段階パイプライン（物体検出→セマンティックセグメンテーション）。Dataset2のbboxアノテーションをDataset1モデルで更新。
* **アーキテクチャ:**
    * 検出: YOLOv5x6, YOLOv7x, YOLOv8l, YOLOv8x。
    * セグメンテーション: EfficientNetB1/B2-UNet。
* **アルゴリズム:** WBF (検出アンサンブル用)。
* **テクニック:**
    * **Dataset2更新:** Dataset1で学習したモデルを用い、Dataset2のbboxアノテーションを予測bboxで更新（IoU>0.4などで条件付け）。これによりDilationなしでもLBスコアが向上。
    * **CV:** 特殊な2 Fold分割。
    * **検出学習:** 2クラス (blood_vessel, glomerulus) で学習 ("unsure"は無視)。YOLOコードを修正しRot90 Augmentation追加。
    * **セグメンテーション学習:** 元のbboxを入力マスクとして使用。3クラス ("unsure"も含む) で学習。
    * **推論:** 検出は大規模アンサンブル (5モデル x 2 Fold x 16 TTA)。セグメンテーションはアンサンブルのみ (4モデル)。
    * **TTA (検出):** Flip, Rot90, Multi-scale (8x2=16)。
    * **アンサンブル (検出):** WBF (IoU=0.7)。
    * **後処理:** Bboxサイズを3%拡大 (軽微なDilation)。

**[10位](https://www.kaggle.com/competitions/hubmap-hacking-the-human-vasculature/discussion/428301)**

* **アプローチ:** YOLOv7を用いた単一モデルベースのアンサンブル。Dataset1でのファインチューニングを重視（Dilation回避目的）。
* **アーキテクチャ:** YOLOv7 (セグメンテーションヘッド付き)。
* **アルゴリズム:** (YOLOv7デフォルト + 変更)
* **テクニック:**
    * **入力解像度:** YOLOのセグメンテーションヘッド解像度に合わせて入力解像度を調整 (例: 640 -> 160)。高解像度入力 (800) も試行。
    * **学習:** 全データ (Dataset1+2) + Pseudo Label (Dataset3) で学習後、Dataset1のみでファインチューニング。
    * **アンサンブル:** Foldアンサンブル + Pseudo Labelモデル。
    * **効果がなかった/試せなかったこと:** 複数回のPseudo Labeling, YOLOv8, Mmdetection, UNetによるマスク補正, タイル結合, Stain Augmentation, 外部データ。WSF(Weighted Segments Fusion)はNMSより良くなかった。
