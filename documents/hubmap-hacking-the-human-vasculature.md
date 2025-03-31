---
tags:
  - Kaggle
startdate: 2023-05-23
enddate: 2023-08-01
---
# HuBMAP - Hacking the Human Vasculature
https://www.kaggle.com/competitions/hubmap-hacking-the-human-vasculature

**概要 (Overview)**

- **目的:** このコンペティションの目的は、人体の様々な臓器（腎臓、大腸、肺、脾臓、前立腺）から得られた**高解像度の組織学（顕微鏡）画像**において、**血管（Vasculature）** の領域を自動的に特定し、セグメンテーション（領域分割）するモデルを開発することです。
- **背景:** Human BioMolecular Atlas Program (HuBMAP) は、人体の個々の細胞レベルでの高解像度マップを作成し、健康と病気への理解を深めることを目指す大規模プロジェクトです。血管網は、組織への栄養供給や老廃物除去に不可欠であり、多くの疾患（例：腎臓病）の理解や診断において重要な構造です。膨大な量の顕微鏡画像から血管のような特定の構造を自動的に抽出する技術は、HuBMAPのようなプロジェクトで生成されるデータを効率的に解析するために不可欠です。
- **課題:** 組織学画像はしばしばギガピクセル単位の非常に大きなサイズであり、効率的な処理が必要です。また、組織の染色状態、組織の種類、個体差などにより、画像の見た目や血管の形態は大きく異なります。さらに、毛細血管のような微細な構造を他の組織と区別することは困難な場合があります。モデルはこれらの課題に対処し、様々な臓器の画像に対して頑健に血管領域を特定する必要があります。これは**医用画像セグメンテーション**のタスクです。

**データセットの形式 (Dataset Format)**

提供される主なデータは、巨大な組織学画像ファイルと、それに対応する血管領域の正解アノテーション（注釈）です。

1. **トレーニングデータ:**
    
    - `train.csv`: トレーニング画像のメタデータ。各画像の `id`、画像の提供元となった臓器の種類 (`organ`)、画像のサイズ、ピクセルあたりの解像度などの情報が含まれます。
    - `train_annotations.json`: **ターゲット変数**となる血管領域の正解アノテーション。各画像 (`id`) に対して、血管領域の境界線を示す**ポリゴン（多角形）** の座標リストとしてJSON形式で提供されます。
    - `train_images/`: 実際の組織学画像ファイル。通常、非常に大きな**TIFF形式**（しばしばマルチ解像度のPyramidal TIFF）で提供されます。
2. **テストデータ:**
    
    - `test_images/`: 血管領域を予測する必要があるテスト用の組織学画像ファイル（TIFF形式）。
    - `test.csv`: テスト画像のメタデータ。
    - 正解のアノテーション（JSONファイル）は提供されません。
3. **`sample_submission.csv`**:
    
    - 提出フォーマットのサンプル。通常、`id`（画像ID）と `rle`（予測結果）の列を持ちます。`rle` 列には、モデルが予測した**全ての血管領域**を結合した単一のセグメンテーションマスクを、**ランレングスエンコーディング (RLE)** 形式の文字列として格納します。

**評価指標 (Evaluation Metric)**

- **指標:** **Dice Coefficient (ダイス係数)**
- **計算方法:** ダイス係数は、予測されたセグメンテーションマスクと正解のセグメンテーションマスクの重複度を測る指標です。
    - Dice = (2 * |予測マスク ∩ 正解マスク|) / (|予測マスク| + |正解マスク|)
    - ここで、`|予測マスク ∩ 正解マスク|` は両方のマスクで血管と判定されたピクセル数（True Positives）、`|予測マスク|` は予測マスクで血管と判定された全ピクセル数、`|正解マスク|` は正解マスクで血管と判定された全ピクセル数です。
    - 評価は通常、**画像ごと**にDice係数を計算し、テストデータセット全体の**画像の平均Dice係数**が最終的なスコアとなります。
- **意味:** 医用画像セグメンテーションの評価で標準的に用いられる指標であり、予測された血管領域が実際の血管領域とどれだけ正確に一致しているかを示します。値は0から1の範囲を取り、1に近いほど予測精度が高いことを意味します。このコンペティションでは、Dice係数が**高い**ほど、モデルが様々な臓器の画像から血管をより正確にセグメンテーションできていると評価されます。

要約すると、このコンペティションは、人体の様々な臓器の高解像度組織学画像から血管領域を特定するセマンティックセグメンテーションタスクです。データは巨大なTIFF画像とポリゴン形式のアノテーションで構成され、性能は予測マスクと正解マスクの重複度を示すDice係数（画像ごとの平均値、高いほど良い）によって評価されます。

---

**全体的な傾向**

上位解法では、インスタンスセグメンテーションタスク（特に血管構造の検出）に対して、物体検出とセグメンテーションを組み合わせたモデルアーキテクチャ（Mask R-CNN、Cascade R-CNN、HTCなど）や、物体検出モデルにセグメンテーションヘッドを追加したモデル（YOLOv7/v8-seg、RTMDetなど）が多く用いられました。特にMMDetectionライブラリをベースとした実装が多く見られます。データの不均衡性（Dataset1とDataset2の特性の違い、WSI間のばらつき）に対応するため、信頼性の高い交差検証（CV）戦略の構築や、Dataset2のノイズを考慮した学習戦略（マルチステージ学習、疑似ラベル生成、Dataset1のみでのファインチューニング）が重要視されました。データ拡張（Augmentation）は、幾何変換（回転、反転、スケール変更など）やStain Augmentation（染色スタイルの変換）が広く利用されました。モデルの性能向上には、Exponential Moving Average (EMA) や Stochastic Weight Averaging (SWA)、高解像度での学習/推論、Test Time Augmentation (TTA)、Weighted Box Fusion (WBF) によるアンサンブルなどが有効でした。後処理として、小さなマスク領域の除去や、糸球体領域との重複に基づくフィルタリング、マスクスコアを用いたインスタンススコアの調整なども行われました。一部の解法では、ラベルのDilation（膨張）がPublic LBスコアを大きく改善する現象が報告されましたが、CVスコアとの相関が低いため、最終提出ではDilationあり/なしの両方を提出する戦略も取られました。

**各解法の詳細**

**1位**

- **アプローチ:** 物体検出モデル(RTMDet)をベースとし、マスク予測精度よりもバウンディングボックス(bbox)精度に注力。マスク情報を補助的に利用し、EMAを活用して安定性と精度を向上。
- **アーキテクチャ:** RTMDet-x (ベースライン)。 Mask R-CNN、YOLOX-xもアンサンブルに使用。
- **アルゴリズム:** EMA (Exponential Moving Average)、WBF (Weighted Box Fusion)。
- **テクニック:**
    - **EMA:** 固定学習率と組み合わせることで学習の安定化と精度向上に大きく寄与。複数のEMAモデルを保持。
    - **データセット/CV:** Dataset1とDataset2の両方を使用。Dataset1を'i'座標で分割してCV。
    - **マスク情報の活用:** ①RandomRotateScaleCrop Augmentationでbboxを再計算、②モデルにマスクヘッドを追加して補助損失として利用、の両方が有効。追加したマスクヘッドの予測精度も良好。
    - **Augmentation:** 強力な幾何学的Augmentation (RandomRotateScaleCropなど)。
    - **学習:** Dataset1とDataset2の画像を特定の比率(3:5)で混合したバッチで学習。
    - **アンサンブル:** bbox予測は複数モデル(RTMDet x3, YOLOX-x, Mask R-CNN、各2fold分)のWBF。マスク予測はMask R-CNN (入力サイズ1440)の結果を使用。TTA不使用。
    - **Dilation:** 最終日まで適用せず。Publicスコアを見て適用を検討したが、最終提出はDilationなし（結果的に奏功）。

**2位**

- **アプローチ:** 信頼性の高いCV戦略を構築し、多様なモデルのアンサンブルで安定性を確保。Dilationの有無で最終提出を分ける。
- **アーキテクチャ:** Cascade Mask R-CNN (Swin-T, CoAt-Small, ConvNeXt-T, ConvNeXt-S)。
- **アルゴリズム:** SWA (Stochastic Weight Averaging)、WBF (推定)、NMS。
- **テクニック:**
    - **CV戦略:** Dataset1の各WSIを左右に分割し4 Fold作成。訓練データ中の同WSIタイルは`staintools`で別WSIスタイルに変換して使用（Leakage抑制）。
    - **入力データ:** ①元タイル、②隣接タイルでパディングしたタイル、の2種類を使用。
    - **Augmentation:** 強力なAugmentation (Stain Augmentation, 幾何変換, AutoAugmentなど)。
    - **学習:** 2段階学習（Stage1: Dataset1(3 folds)+Dataset2で学習、Stage2: Dataset1(3 folds)のみでFine-tuning）。Blood Vesselクラスのみ学習。複数チェックポイントでSWA。
    - **アンサンブル:** FoldモデルとFull dataモデルを組み合わせ。
    - **後処理:** 複数フィルタリング（Glomeruli重複、低信頼度、アンサンブル画素数カウント、アンサンブルインスタンス数カウント(NMS利用)、小マスク）。
    - **Dilation:** CVでは効果がなかったため信頼せず。最終提出はDilationあり/なしの両方を提出（なしの方がPrivateスコア良好）。

**3位**

- **アプローチ:** マルチステージ学習でDataset2のノイズに対応。ViT-Adapterなど新しいアーキテクチャを積極的に採用。Pseudo Labelも一部利用。
- **アーキテクチャ:** ViT-Adapter-L (x2), CBNetV2 Base, DetectoRS ResNeXt-101-32x4d, DetectoRS ResNet50。
- **アルゴリズム:** WBF (Weighted Box Fusion)。Erosion + Dilation。
- **テクニック:**
    - **マルチステージ学習:** Stage1: Dataset2で事前学習（高LR、低エポック、軽Augmentation）。Stage2: Dataset1でFine-tuning（低LR、高エポック、重Augmentation、高解像度）。
    - **疑似ラベル:** Dataset3に対して疑似ラベルを生成し、一部モデルの学習に使用（LBスコア向上は見られず）。
    - **モデル:** ViT-Adapter, CBNetV2, DetectoRS (HTCベース) など多様なアーキテクチャを採用。
    - **損失関数:** マスクヘッドの損失重みを増加（HTCベースモデル）。
    - **学習:** 高解像度学習 (1200x1200 ~ 2048x2048)。Cosine Scheduler w/ warmup。SGDまたはAdamW。
    - **推論:** Multi Scale Inference (ViT-Adapter)。FlipベースTTA。
    - **アンサンブル:** WBFを使用。CNNとTransformerベースのモデルを組み合わせ多様性を確保。
    - **後処理:** Erosion後にDilationを1回適用（マスク形状の平滑化、+0.005 LB向上）。

**4位 (SDSRV.AI - GoN team)**

- **アプローチ:** 信頼できるCV（Dataset1の一部をValidation）に基づき、多様なモデル（2段階モデル、単段階モデル）をアンサンブル。後処理でスコアを調整。
- **アーキテクチャ:** CascadeRCNN (ResNeXt, RegNet), MaskRCNN (SwinT), HybridTaskCascade (HTC, Re2Net), YOLOv6m。
- **アルゴリズム:** AdamW。Multi scale training。WBF。Mean Average Mask Ensemble。スコア調整式。
- **テクニック:**
    - **CV戦略:** Dataset1の20%をValidationとし、残りのDataset1とDataset2で学習。
    - **前処理:** 重複アノテーション除去。
    - **学習:** "blood_vessel"と"unsure"クラスのみで学習（Glomerulusは除外）。2段階学習（Stage1: 全訓練データ、Stage2: Dataset1データのみでFine-tuning）。AugmentationはMulti scale trainingのみ。
    - **モデル選択:** 精度と多様性を考慮し、CascadeRCNN, MaskRCNN, HTCに加え、高MARを示すYOLOv6mを採用。
    - **アンサンブル:** bboxはWBF、マスクはMean Averageでアンサンブル。
    - **後処理:** 小面積インスタンス除去(<80px)。マスクスコアを用いてインスタンススコアを再計算 (`score_instance = score_bbox * mean(mask[mask > 0.5])`)。
    - **Dilation:** CVスコアとの相関を信頼し、適用せず。

**5位**

- **アプローチ:** Dataset1のみを使用し、高解像度学習とTTA、Cluster-NMSで精度を追求。単一モデル（HTC-DB-B）で高スコア達成。
- **アーキテクチャ:** HTC-DB-B (Hybrid Task Cascade with Deformable BackBone - Swin-B)。DetectoRS-R101も試行。
- **アルゴリズム:** Cluster-NMS with DIoU。SWA (Stochastic Weight Averaging)。
- **テクニック:**
    - **データ:** Dataset1のみ使用（5 folds CV）。COCO事前学習重み利用。
    - **学習:** MMDetection V2。デフォルト1xスケジュール + 1x SWA。高解像度マルチスケール学習 ((512,512)-(1536,1536))。
    - **Augmentation:** RandomFlip, Rotate90のみ。
    - **推論:** 大規模TTA（複数スケール x Flip x Rotate）。R-CNNのNMSをWeighted Cluster-NMS (DIoU) に置き換え。
    - **アンサンブル:** 複数FoldのアンサンブルはCV/Publicを低下させたため単一Foldモデルを提出（ただしPrivateスコアはアンサンブルの方が若干良かった）。
    - **その他:** より大きなモデル(HTC-DB-L)はCV/Publicは低かったがPrivateは高かった。Copy-paste, MixUp, HSV, CutOutなどのAugmentationは効果なし。Dataset2/3利用（事前学習、疑似ラベル）も効果なし。

**7位**

- **アプローチ:** Mask R-CNN (Swin Transformer) をベースとし、Dataset2/3の疑似ラベルを用いて学習データを拡張。アンサンブルとDilationでスコア向上。
- **アーキテクチャ:** Mask R-CNN (Swin Transformer backbone, HTC RoI head)。
- **アルゴリズム:** 不明（MMDetectionベースと推定）。Ensemble Detection Model (Sartoriusコンペ解法参照)。
- **テクニック:**
    - **疑似ラベル:** Dataset1で学習したモデル（Foldごと）を用いてDataset2, 3の疑似ラベルを作成。
    - **学習:** Dataset1 + Dataset2 (元ラベル+疑似ラベル) + Dataset3 (疑似ラベル) でモデルを学習 (5 folds)。
    - **Augmentation:** Random resize (768-1536), flip, Rot90, RandomBrightnessContrast, HueSaturationValue。
    - **TTA:** Resize (1024, 1536), hvflip。
    - **アンサンブル:** RPNとRoI Headの両方でアンサンブル（Sartoriusコンペ解法参照）。
    - **後処理:** Dilation、小マスク除去、Glomerulus領域重複マスク除去。
    - **Dilation:** Dataset1のみ学習ではDilationなしの方がスコアが良かった経験から、Dataset2のノイズが原因と推測。疑似ラベル等でノイズの影響を低減し、Dilationあり/なしの差を縮小（最終的にはDilationありがLB/Private共に良好）。

**8位**

- **アプローチ:** YOLOv8x-seg 単一モデル。強力なAugmentationとマスク閾値の調整が鍵。
- **アーキテクチャ:** YOLOv8x-seg。
- **アルゴリズム:** AdamW、Cosine LR。
- **テクニック:**
    - **CV戦略:** 特殊な2-Fold CV (WSI 1,3 vs 2,4)。マスク閾値の最適値がFold間で大きく異なることを発見。
    - **学習:** 全訓練データを使用。3クラス（blood_vessel, glomerulus, unsure）で学習。強力なAugmentation (mosaic, mixup, copy_paste, 色・幾何変換多数)。
    - **推論:** 学習時より大きい画像サイズ(768)を使用。異なるマスク閾値(0.5と0.2)で2つ提出。閾値0.5がPrivateで高スコア、0.2がPublicで高スコア。
    - **Augmentation:** yolov8デフォルトに加え、hsv, degrees, translate, scale, shear, perspective, flipud, fliplr, mosaic, mixup, copy_pasteなど多数設定。

**9位**

- **アプローチ:** 2段階パイプライン（物体検出→セマンティックセグメンテーション）。Dataset2のアノテーションをモデル予測に基づいて更新。YOLO系モデルのアンサンブルとWBF。
- **アーキテクチャ:** 検出: YOLOv5x6, YOLOv7x, YOLOv8l, YOLOv8x。セグメンテーション: EfficientNetB1-UNet, EfficientNetB2-UNet。
- **アルゴリズム:** WBF。
- **テクニック:**
    - **Dataset2アノテーション更新:** Dataset1で学習したモデルでDataset2を予測し、特定の条件（IoU>0.4など）を満たす予測bboxで元アノテーションを置換（主に拡大方向）。更新後データで学習するとDilationの影響が減少。
    - **CV戦略:** 特殊な2-Fold CV。
    - **検出モデル:** 2クラス（blood_vessel, glomerulus）で学習（unsure無視）。YOLOv5/v7/v8コードを改変し90度回転Augmentation追加。
    - **セグメンテーションモデル:** 元アノテーションのbboxをマスクとして学習。3クラス（unsure含む）で学習。
    - **推論/アンサンブル:** 検出モデルは大規模アンサンブル（10モデル x 16 TTA）。WBFでbboxを統合。セグメンテーションモデルはTTAなしでアンサンブル。
    - **後処理(Dilation):** 最終マスクのDilationではなく、検出bboxを3%拡大する手法を採用（LB +0.005）。

**10位**

- **アプローチ:** YOLOv7ベース。Dataset1でのファインチューニングを重視（Dilation回避のため）。疑似ラベル活用。
- **アーキテクチャ:** YOLOv7 (セグメンテーションヘッドの解像度変更？)。
- **アルゴリズム:** 不明（YOLOv7デフォルトベースと推定）。
- **テクニック:**
    - **学習:** ベースは公開ノートブック。疑似ラベル（ds1で学習したモデルでds2, ds3を予測）を追加して学習後、最後にds1のみでファインチューニング。
    - **入力解像度:** セグメンテーションヘッドの解像度に合わせて入力解像度を変更？ (640->160)。
    - **推論解像度:** 学習時(640)より高い解像度(800)で推論したモデルが最高スコア。
    - **Dilation:** Dataset1でのファインチューニングにより不要と考え、使用せず。
    - **その他:** 疑似ラベルの2回適用、YOLOv8、mmdetection、U-NetによるBBox内セグメンテーション、WBFは効果がなかったか、実装できなかった。

