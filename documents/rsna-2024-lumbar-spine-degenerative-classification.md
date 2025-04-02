---
tags:
  - Kaggle
  - ヘルスケア
  - YOLOX
  - EfficientNet
  - UNet
startdate: 2024-05-17
enddate: 2024-10-09
---
# RSNA 2024 Lumbar Spine Degenerative Classification
[https://www.kaggle.com/competitions/rsna-2024-lumbar-spine-degenerative-classification](https://www.kaggle.com/competitions/rsna-2024-lumbar-spine-degenerative-classification)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、患者の腰椎MRI画像（矢状断T1, T2/STIR, 軸位断T2）を用いて、複数の椎間板レベル（L1/L2からL5/S1）における3種類の**変性状態（脊柱管狭窄症、神経孔狭窄症（左右）、椎間関節下狭窄症（左右））の重症度**を自動で分類する機械学習モデルを開発することです。
* **背景:** 腰椎の変性疾患は腰痛や下肢の神経症状を引き起こす一般的な原因であり、その正確な診断は適切な治療方針の決定に不可欠です。MRIはこれらの疾患の評価に広く用いられていますが、読影には専門的な知識と時間が必要です。AIによる重症度分類は、放射線科医の診断支援、読影の効率化、および診断の一貫性向上に貢献することが期待されています。
* **課題:** 3種類の異なるMRIシーケンスから得られる3次元的な情報を統合し、5つの椎間板レベル、左右を含む合計25箇所の状態について、それぞれ3段階（正常/軽度、中等度、重度）の重症度を分類するという、**複雑なマルチラベル・マルチクラス分類問題**です。画像データから関連する解剖学的位置（椎間板、神経孔など）を正確に特定し、その領域における微妙な信号変化や形態変化を捉えて重症度を判断する必要があります。データセットにはラベルノイズが含まれている可能性も指摘されており、これに対処することも課題の一つです。

**データセットの形式 (Dataset Format)**

提供される主なデータは、腰椎MRIのDICOM画像と、それに対応する重症度ラベルおよび一部の座標情報です。

1.  **トレーニングデータ:**
    * 患者ごとの腰椎MRIスキャンデータ（DICOM形式）。各患者（`study_id`）には通常、3つのシリーズ（矢状断T1、矢状断T2/STIR、軸位断T2）が含まれます。各シリーズは複数のDICOMファイル（スライス画像）から構成されます。
    * `train.csv`: 各`study_id`について、5つのレベル（`L1/L2`から`L5/S1`）ごとに5つの状態（`spinal_canal_stenosis`, `left_neural_foraminal_narrowing`, `right_neural_foraminal_narrowing`, `left_subarticular_stenosis`, `right_subarticular_stenosis`）の重症度ラベル（`normal_mild`, `moderate`, `severe`）が示されています。1行が1つの評価箇所（患者xレベルx状態）に対応し、該当する重症度クラスの列が1、他が0となります（One-Hot形式に近い）。
    * `train_label_coordinates.csv`: トレーニングデータの一部について、各評価箇所に対応する3D座標（`x`, `y`, `instance_number`（スライス番号に相当））が提供されます。これは、関心領域（ROI）の特定や抽出に利用できます。
    * `series_descriptions.csv`: 各シリーズID（`series_id`）とシリーズの説明（例: "Sagittal T1"）の対応を示します。
    * `instance_numbers.csv`: 各シリーズIDとインスタンス番号（スライス番号）の最大値を示します。
2.  **テストデータ:**
    * トレーニングデータと同様の形式で、患者ごとの腰椎MRIスキャンデータ（DICOM形式）が提供されます。
    * 重症度ラベルや座標情報は提供されません。参加者はこれらの画像データから重症度を予測します。
3.  **外部データ:**
    * [@brendanartley]([https://www.kaggle.com/brendanartley)氏によって共有された座標データセット](https://www.google.com/search?q=https://www.kaggle.com/brendanartley)%E6%B0%8F%E3%81%AB%E3%82%88%E3%81%A3%E3%81%A6%E5%85%B1%E6%9C%89%E3%81%95%E3%82%8C%E3%81%9F%E5%BA%A7%E6%A8%99%E3%83%87%E3%83%BC%E3%82%BF%E3%82%BB%E3%83%83%E3%83%88)（[Coordinate Pretraining Dataset](https://www.kaggle.com/competitions/rsna-2024-lumbar-spine-degenerative-classification/discussion/524500)）などが、モデルの事前学習や座標推定の精度向上に利用されました。
4.  **`sample_submission.csv`**:
    * 提出フォーマットのサンプル。`row_id`（`study_id`、状態、レベルを組み合わせた識別子）と、3つの重症度クラス（`normal_mild`, `moderate`, `severe`）それぞれの予測確率の列を持ちます。各行の3つの確率の合計は1になる必要はありません。

**評価指標 (Evaluation Metric)**

* **指標:** **重み付き対数損失 (Weighted Log Loss)**
* **計算方法:**
    1.  各予測ターゲット（25箇所 × 3クラス = 75個の二値分類問題とみなせる）に対して、対数損失（Log Loss）を計算します。
        * Log Loss = - [ y \* log(p) + (1 - y) \* log(1 - p) ]
        （y: 正解ラベル (0 or 1), p: 予測確率）
    2.  各ターゲットクラス（正常/軽度、中等度、重度）に割り当てられた重み（それぞれ 1, 2, 4）を、対応するLog Lossに乗じます。
    3.  全ターゲット（75個）にわたる重み付きLog Lossの平均値を最終スコアとします。
* **意味:** 予測確率の精度を評価する指標であり、特に臨床的に重要度の高い**中等度および重度の誤分類に対してより大きなペナルティ**を与えるように設計されています。モデルがより重篤な状態を正確に予測することを奨励します。スコアは**低い**ほど良い評価となります。

---

**全体的な傾向**

このコンペティションでは、腰椎MRIの3つの異なるシーケンス（矢状断T1, T2/STIR, 軸位断T2）から、複数のレベル・位置における変性状態の重症度を分類する必要がありました。多くの成功した解法は、**2段階アプローチ**を採用していました。

**Stage 1**では、MRI画像から**関心領域（ROI）を特定または抽出**します。これには、座標やキーポイントを直接予測する**検出モデル**（CenterNet, YOLOX, Heatmapベースなど）や、関連するスライスを選択・割り当てる手法（3D射影、2.5D CNN+LSTMなど）が用いられました。特に軸位断画像に対して、対応する椎間板レベルを推定する処理が重要でした。

**Stage 2**では、Stage 1で得られたROI（多くの場合、特定のレベルや状態に対応する画像パッチ）を入力として、**重症度を分類**します。ここでは、**2D CNN**（ConvNeXt, EfficientNet, ResNet系などが人気）をベースとしたアーキテクチャが主流でした。3次元的な情報を捉えるために、複数スライスを時間軸やチャンネル軸として入力する**2.5Dアプローチ**（CNN + RNN/LSTM/Transformer/Attention）や、Attentionメカニズムを利用した**Multiple Instance Learning (MIL)** が効果的に用いられました。

データ処理においては、3つのMRIシーケンスをどのように組み合わせるか（複数チャンネル入力、個別モデルで特徴量抽出後に統合など）、左右対称性を利用したデータ拡張（軸位断の反転など）、そして指摘されていた**ラベルノイズへの対応**（手動修正、損失に基づくノイズ除去など）が工夫されました。

モデリング戦略としては、各状態（脊柱管狭窄症、神経孔狭窄症、椎間関節下狭窄症）や各レベル（L1/L2〜L5/S1）に特化したモデルを構築するアプローチと、レベルや左右を区別せずに学習データ量を増やし、汎用的な特徴を学習させるアプローチが見られました。

学習においては、重み付き損失関数（評価指標に合わせた重み付けやクラス不均衡対策）、補助損失（Auxiliary Loss）の導入、強力なデータ拡張（特にCutMix）、段階的学習（座標/スライス推定モデルの事前学習、分類モデルのファインチューニング）、TTAなどが有効でした。

最終的なスコア向上には、**多様なモデル（異なるアーキテクチャ、異なるFold、異なる入力データ戦略）のアンサンブル**が不可欠であり、単純平均や重み付き平均、スタッキング（MLP, LightGBMなど）が用いられました。予測確率を調整するための後処理（Temperature Scalingなど）も一部で見られました。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/rsna-2024-lumbar-spine-degenerative-classification/discussion/540091)**

* **アプローチ:** 2段階アプローチ。Stage 1でインスタンス番号（スライス位置）と座標を予測し、Stage 2で重症度を分類。Stage 1はさらにインスタンス番号予測と座標予測に分離。
* **アーキテクチャ:**
    * インスタンス番号予測 (Sagittal): 3D ConvNeXt（回帰タスクと分類タスクで学習）。
    * 座標予測 (Sagittal, Axial): 2D CNN Encoder (ConvNeXt-base, EfficientNet-v2-l) + レベル別ヘッド。
    * 重症度分類 (Stage 2): 2.5D MIL (Attention-based MIL + Bi-LSTM)。EncoderはConvNeXt-small, EfficientNet-v2-s。
* **アルゴリズム:**
    * インスタンス番号予測: L1 Loss (回帰), Cross Entropy Loss (分類)。
    * 座標予測: L1 Loss。
    * 重症度分類: Cross Entropy Loss (重み付き、Auxiliary Lossあり)。
* **テクニック:**
    * **Stage 1:** 3Dモデルで深さ（インスタンス番号）を、2DモデルでXY座標を予測。座標予測モデルは外部データセットで事前学習。軸位断のインスタンス番号は3D射影法を利用。
    * **Stage 2 Preprocessing:** Stage 1の予測座標に基づきROIをクロップ（例: Sag T2 for SCSは座標中心に左右非対称、Axial for SSは左右対称など）。入力は5スライス。
    * **Stage 2 Augmentation:** 座標シフト、インスタンス番号シフト（Stage 1の誤差を考慮）、輝度・コントラスト、ShiftScaleRotateなど。
    * **Stage 2 Model:** Attention-based MILにBi-LSTMを組み込み、時間的特徴を学習。Auxiliary LossとしてAttentionスコア自体も損失計算に使用。
    * **アンサンブル:** Stage 1では回帰と分類モデルの予測を中央値でアンサンブル。Stage 2では複数バックボーン、複数Foldのモデルをアンサンブル。

**[2位](https://www.kaggle.com/competitions/rsna-2024-lumbar-spine-degenerative-classification/discussion/539452)**

* **アプローチ:** 軸位断と矢状断で個別に処理。各ターゲット（SCS, NFN, SS）ごとにモデルを作成。最終的にノイズ除去して再学習。チームメンバーの予測とアンサンブル。
* **アーキテクチャ:**
    * スライス/領域検出: YOLOX。
    * 分類モデル: ConvNeXt Small (2.5D MIL)。
* **アルゴリズム:**
    * 検出: YOLOX。
    * 分類: Cross Entropy Loss? (重み付き?)
* **テクニック:**
    * **軸位断:** スライス分類（Heng氏コード利用）→ YOLOXで領域推定 → ConvNeXt Smallで分類（SCSは推定領域、NFN/SSは画像を左右半分にして左右ラベルを統一）。
    * **矢状断:** スライス分類（2.5D Timmモデル）→ YOLOXでレベルごと領域推定（Brendan氏データ利用）→ レベルごとに水平化してクロップ → 5スライス入力のMILモデル (ConvNeXt Small) で分類。NFNはSCSとSSの中間スライスを使用。一部モデルはT1/T2を別チャンネル入力。
    * **ノイズ除去:** チームのアンサンブルOOF予測と正解ラベルの差が大きいサンプルを除外して再学習（Moderate/Severeクラスに係数適用）。Public/Privateで約1%向上。
    * **アンサンブル/後処理:** 各メンバーの予測を重み付き平均。SCSのSevere予測値が最も高いレベルの値を1.25倍する後処理。

**[3位](https://www.kaggle.com/competitions/rsna-2024-lumbar-spine-degenerative-classification/discussion/539453)**

* **アプローチ:** 2段階パイプライン。Stage 1で椎間板レベル検出とROIクロップ。Stage 2でCenter Classifier (SCS用) とSide Classifier (NFN/SS用) による分類。
* **アーキテクチャ:**
    * キーポイント検出 (Stage 1): CenterNet (EfficientNetB6/B4 + FPN)。
    * 分類 (Stage 2): 2D Encoder (ResNet, MNasNet, EfficientNet, ConvNeXt, MaxViTなど多様) + Attentionメカニズム。
* **アルゴリズム:**
    * 検出: CenterNet Loss?
    * 分類: Cross Entropy Loss (重み付き、Auxiliary Lossあり)。
* **テクニック:**
    * **Stage 1:** Sagittal画像で椎間板レベル検出（外部データ＋擬似ラベル活用、手動ノイズ修正）。Axial画像で脊柱管位置検出。3D座標変換を用いてAxialスライスにレベルを割り当て。Sagittal/Axial画像をクロップ（複数設定で多様性確保）。
    * **Stage 2 Model (Center/Side Classifier):** Sag T1, Sag T2, Axial T2の複数スライス（10-15枚）を各Encoderに入力し、Attentionで統合。レベルや左右を区別せずに学習データ量を増加（Centerは5倍、Sideは10倍）。
    * **学習:** Auxiliary Loss（他状態の重症度、スライスレベル予測）。Severeクラスの損失重みを増加。TTA（軸位断フリップ）。
    * **擬似ラベル:** ラベルなしデータにモデル予測をソフトラベルとして使用（精度への影響は小さいが多様性確保のため）。
    * **アンサンブル:** 30モデル（多様な設定）の単純平均。
    * **後処理:** SCS予測のロジットにTemperature Scaling (T=0.91) を適用。

**[4位](https://www.kaggle.com/competitions/rsna-2024-lumbar-spine-degenerative-classification/discussion/539443)**

* **アプローチ:** 2段階パイプライン。キーポイント検出モデルでROIを特定し、分類モデルで重症度を予測。最後にスタッキングモデルで集約。
* **アーキテクチャ:**
    * キーポイント検出: 2.5D CNN + LSTM (Axialレベル分類), U-Net (Axialキーポイント), 2.5D CNN (Sagittalキーポイント), UNet (Sagittalキーポイント、caformer, convnext, resnetrs, swin backbone)。
    * 分類 (Multi-view): 2D Backbone (caformer, resnetrs, rdnet, maxxvit) + Transformer Encoder + Attention Pooling。
    * 分類 (Single-view): 2.5D CNN (Backbone + RNN?)。
    * スタッキング: MLP (Nelder-Mead最適化併用), LightGBM, XGBoost。
* **アルゴリズム:**
    * 検出: BCELoss + DICELoss (UNet)。
    * 分類: Cross Entropy Loss? (Auxiliary Lossあり)。
    * スタッキング: MLP, LightGBM, XGBoost。
* **テクニック:**
    * **キーポイント検出:** 複数アプローチで実施（2.5D+LSTM, UNetなど）。外部座標データセット利用。
    * **分類 (Multi-view):** Sag T1, Sag T2, Axial T2のクロップ画像（キーポイント周辺、複数スライス）を入力。Transformer EncoderとAttention Poolingで特徴統合。Auxiliary LossとしてAttention Pooling前の特徴量に対する損失も計算。
    * **分類 (Single-view):** 各シーケンス（Sag T1, T2, Axial T2）ごとにクロップ画像を入力し、2.5D CNNで重症度予測。Axial T2はSSとSCSの両方を予測。
    * **スタッキング:** MLPではNelder-Mead最適化結果をSkip Connection的に加算。GBDTではレベルごと、ターゲットタイプごとにモデル構築。

**[5位](https://www.kaggle.com/competitions/rsna-2024-lumbar-spine-degenerative-classification/discussion/539472)**

* **アプローチ:** 2段階。Stage 1でHeatmapベースのキーポイント検出。Stage 2で2.5Dモデル (CNN+RNN) による分類。CutMixによるAugmentationと2段階学習が特徴。
* **アーキテクチャ:**
    * キーポイント検出 (Stage 1): 2D UNet (EfficientNet-B5 backbone) + Sequential Model (Transformer, LSTM)。Heatmapベース。
    * 分類 (Stage 2): 2.5Dモデル (CNN Backbone: ConvNeXt, caformer, pvt_v2など + RNN: LSTM/GRU?)。レベルワイズシーケンスモデリング。
* **アルゴリズム:**
    * 検出: Heatmap Loss (MSE?)。
    * 分類: Cross Entropy Loss (重み付き、Any Lossあり)。
* **テクニック:**
    * **Stage 1:** Heatmapベース検出。ラベルをGaussian Filterで拡張し、Z軸方向にも重み付け（Gaussian Expanding Label）。外部座標データセット利用。
    * **Stage 2 Model:** 典型的な2.5D (CNN+RNN) に加え、クラス間関係をモデル化するモジュールを追加。当初全25クラスをLSTMでモデル化したが、同一レベル内のクラスのみをモデル化する方が効果的だった（Level-wise Sequence Modeling）。
    * **学習 (Two-step Training):** Stage 2の学習を2段階化。Step 1 (Pretraining) でAUC最大化を目指し重みなし損失で学習。Step 2 (Finetuning) でバックボーンを凍結し、重み付き損失でHeadのみ学習し、評価指標を最適化。
    * **Augmentation:** CutMix (p=1.0) が過学習防止とAUC向上に最も効果的だった。
    * **アンサンブル:** 多様なバックボーン、異なるStage 1/2設定のモデルをアンサンブル。TTA風アンサンブル（個々のStage 1予測ごとにStage 2推論を行い、最後に平均化）。

**[6位](https://www.kaggle.com/competitions/rsna-2024-lumbar-spine-degenerative-classification/discussion/541813)**

* **アプローチ:** 2段階。Stage 1でROIクロップ。Stage 2で椎間板レベルごとに分類。複数モデルのアンサンブルとMLPによる集約。
* **アーキテクチャ:**
    * 座標予測 (Model 1): CoAtNet。
    * 分類 (Model 1): CoAtNet + RNN。
    * XY座標予測 (Model 2/3): EfficientNetV2 + Linear Head。SpineNetも利用。
    * Z座標予測 (Model 2): ? + L1 Loss。
    * 分類 (Model 2): EfficientNetV2 + BiRNN。
    * 分類 (Model 3): 3 x 2.5D EfficientNetV2 (3D畳み込み適用)。
    * 集約: MLP。
* **アルゴリズム:**
    * 座標予測: MSE, L1 Loss。
    * 分類: Cross Entropy Loss (重み付き)。
* **テクニック:**
    * **Stage 1 (座標/クロップ):** 複数アプローチ。Model 1はCoAtNetで座標予測しROIクロップ。Model 2/3はEfficientNetV2やSpineNetでXY座標予測、Z座標予測も実施。AxialのZ座標はSagittalからの3D射影を利用。
    * **Stage 2 (分類):**
        * Model 1: Sagittal T1/T2を同時入力するCoAtNet+RNN。複数フレーム入力。左右中央ターゲット用に3つのRNNヘッド。
        * Model 2: Axial T2とSagittal T1/T2で別々のEfficientNetV2+BiRNNモデル。
        * Model 3: Axial, Sag T1, Sag T2の3Dクロップを結合し、3つの2.5D EfficientNetV2に入力して各状態を予測。StudyレベルでのSevere有無も予測。
    * **データ:** 外部座標データ利用。左右反転Augmentation。
    * **アンサンブル/集約:** 複数モデル（Model 1, 2, 3のバリアント）の予測ロジットをMLPに入力し、最終出力を得る。MLPはターゲットごとに独立（レベル間の重みは共有）。

**[7位](https://www.kaggle.com/competitions/rsna-2024-lumbar-spine-degenerative-classification/discussion/539486)**

* **アプローチ:** 1段階解法と2段階解法のアンサンブル。（Heng氏パート + lhwcvパート）
* **アーキテクチャ (lhwcvパート):**
    * キーポイント回帰 (Stage 1): 2D CNN (timmモデル), 3D CNN (timm-3dモデル), 2.5D (CNN+LSTM+Attention)。
    * 分類 (Stage 2 Model 1): 3視点の特徴量融合モデル。AxialとSagittalで別々の2.5D CNN Encoder + Linear Head。
    * 分類 (Stage 2 Model 2): 単一視点3D ROI入力モデル。
* **アルゴリズム:**
    * キーポイント回帰: ?
    * 分類: Cross Entropy Loss?
* **テクニック (lhwcvパート):**
    * **Stage 1:** 2D, 3D, 2.5Dなど多様なモデルでキーポイント（座標）を回帰。
    * **Stage 2 Model 1:** Sagittal (T1/T2) と Axial からそれぞれ2.5D CNNで特徴抽出し、結合して分類。
    * **Stage 2 Model 2:** 3Dキーポイントに基づき3D ROIをクロップし、3D CNN?で分類。
    * **アンサンブル:** Stage 2の2モデル、複数バックボーン（pvt, convnext, densenet）、Flip TTAを組み合わせてアンサンブル。Heng氏のモデルとさらにアンサンブル。

**[8位](https://www.kaggle.com/competitions/rsna-2024-lumbar-spine-degenerative-classification/discussion/539548)**

* **アプローチ:** 2段階。Stage 1 (2D分類器) でAttention風に特徴抽出。Stage 2 (1D分類器) でStage 1出力をスタッキング。
* **アーキテクチャ:**
    * Stage 1: 2D CNN + Detector (補助タスク) + Attention風特徴抽出。
    * Stage 2: 1D Classifier (MLP or GBDT?)。
* **アルゴリズム:**
    * Stage 1: Classification Loss + Detection Loss?
    * Stage 2: Classification Loss?
* **テクニック:**
    * **Attention風特徴抽出:** ROIクロップの代わりに、画像全体を入力しつつ、補助タスクとして学習した検出器のHeatmapをAttention Weightのように利用して特徴量を抽出。画像の全体コンテキストを考慮し、検出の曖昧さに頑健。
    * **SSL:** 自己教師あり学習（Multi-view Similarity学習）を導入し、Public/Privateスコアが微増。
    * **ラベリングツール:** ラベルノイズ対応のため自作ツールを作成（ただしドメイン知識不足で活用は限定的）。

**[9位](https://www.kaggle.com/competitions/rsna-2024-lumbar-spine-degenerative-classification/discussion/539690)**

* **アプローチ:** 2段階。Stage 1でHeatmapベースのキーポイント検出。Stage 2でレベルワイズROI分類。
* **アーキテクチャ:**
    * キーポイント検出 (Stage 1): DeepLabV3Plus (ResNet34 Encoder)。シリーズタイプごとに別モデル。
    * 分類 (Stage 2): 2.5Dモデル (ResNet18, Swin-Tiny, ConvNeXt-Nano Backbone + GRU Head)。
* **アルゴリズム:**
    * 検出: MSE Loss。
    * 分類: Cross Entropy Loss。
* **テクニック:**
    * **Stage 1:** Gaussian HeatmapをターゲットとしてDeepLabV3Plusで学習。座標は予測Heatmapのargmax。入力スライスはシリーズタイプごとに選択方法が異なる（中央、相対位置、Sagittalからの3D射影）。
    * **Stage 2:** 検出された座標中心にROI（50mm x 50mm, 5スライス）を抽出し分類。入力サイズはバックボーン依存。
    * **モデリング戦略:** "Split"モデル（SCS, NFN, SSごとに専用モデル）と "Global"モデル（全5状態を同時予測）の両方を学習。SplitモデルのSSとGlobalモデルでは左右反転を利用。
    * **アンサンブル:** 6モデル（Split/Global x 3バックボーン）の5-Fold平均を単純平均。
