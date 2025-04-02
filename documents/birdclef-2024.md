---
tags:
  - Kaggle
startdate: 2024-04-04
enddate: 2024-06-11
---
# BirdCLEF 2024 - Bird Sound Classification
[https://www.kaggle.com/competitions/birdclef-2024](https://www.kaggle.com/competitions/birdclef-2024)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、インド西部ガーツ山脈で録音された連続的な音声データ（サウンドスケープ）から、特定の時間枠（5秒ごと）にどの種類の鳥が存在するかを特定するモデルを開発することです。2023年版と同様に、生物多様性のモニタリングと保全活動への応用を目指しています。
* **背景:** BirdCLEFチャレンジの一環として、鳥類の音響モニタリング技術の向上を目的としています。Xeno-cantoなどのプラットフォームから提供されるラベル付きデータに加え、今回は大量のラベルなしサウンドスケープデータが提供され、これらをどう活用するかが焦点となりました。
* **課題:**
    * **ドメインシフト:** ラベル付きの訓練データ（主にXeno-canto由来）と、ラベルなし及びテスト用のサウンドスケープデータの間には、録音環境やノイズ特性の違い（ドメインギャップ）が存在します。
    * **ラベルなしデータの活用:** 大量のラベルなしサウンドスケープデータを効果的に利用し、モデルの汎化性能を向上させる必要があります（例: 疑似ラベリング、半教師あり学習）。
    * **音響環境の複雑性:** 背景ノイズ、他の動物の声、天候音などが混在する中で、目的の鳥の鳴き声を正確に識別する必要があります。
    * **クラス不均衡と希少種:** データセットには非常にサンプル数の少ない種が含まれており、これらの種を精度良く検出することが困難です。
    * **評価指標 (AUC):** 2023年の適合率ベースの指標からAUCに変更され、クラス間の識別能力がより重視されるようになりました。
    * **推論時間制限:** 従来通り、Kaggle Notebook環境での2時間の推論時間制限があり、効率的なモデルと推論パイプラインが求められます。

**データセットの形式 (Dataset Format)**

提供される主なデータは、ラベル付きの短い音声クリップ、ラベルなしの長いサウンドスケープ音声、およびメタデータです。

1.  **トレーニングデータ:**
    * `train_metadata.csv`: ラベル付き音声ファイルに関するメタデータ。
        * `filename`: 音声ファイル名。
        * `primary_label`: 主たる鳥の種類コード。
        * `secondary_labels`: 同時に存在する可能性のある他の鳥の種類リスト。
        * `latitude`, `longitude`, `rating`, `author` など。
    * `train_audio/`: ラベル付き音声ファイル（Ogg Vorbis形式）。
    * `unlabeled_soundscapes/`: **ラベルなし**のサウンドスケープ音声ファイル（Ogg Vorbis形式）。疑似ラベリングや半教師あり学習などに利用可能。
2.  **テストデータ:**
    * `test_soundscapes/`: 予測対象となるサウンドスケープ音声ファイル（Ogg Vorbis形式、約4分間）。
    * `sample_submission.csv`: 提出フォーマットのサンプル。
        * `row_id`: `[soundscape_filename_without_extension]_[bird_species]_[seconds]` の形式（例: `soundscape_XYZ_amered_5`）。
        * `target`: **ターゲット変数**。その鳥がその5秒区間に存在するかどうかの予測確率 (0から1)。

**評価指標 (Evaluation Metric)**

* **指標:** **クラスごとのAUCスコアの重み付き平均 (Weighted Average of Class-wise ROC AUC)**
* **計算方法:**
    * 各テストサウンドスケープは5秒ごとのチャンクに分割されます。
    * 参加者は、各 `row_id` (各サウンドスケープ、各鳥、各5秒区間) に対して、その鳥が存在する確率を予測します。
    * 各鳥の種類ごとに、予測確率と正解ラベル（内部的に保持）を用いてROC曲線下面積 (AUC) が計算されます。AUCは、モデルがその鳥が存在するケースと存在しないケースをどれだけうまく識別できるかを示します (0.5はランダム、1.0は完全識別)。
    * 最終スコアは、全182種の鳥のAUCスコアを、各クラスの正例の数（テストセット全体での出現頻度）に基づいて**重み付け**して平均した値となります。
* **意味:** モデルの全体的な識別性能を評価する指標であり、特にサンプル数の多い（=重みが大きい）鳥の分類精度が最終スコアに強く影響します。閾値に依存しないため、モデルが出力する確率値のランキング精度が重要になります。スコアは**高い**ほど良い性能を示します。

要約すると、このコンペティションは、ラベル付きデータと大量のラベルなしサウンドスケープデータを用いて、音声中の鳥の種類を5秒ごとに予測するマルチラベル分類タスクです。性能はクラスごとの重み付き平均AUC（高いほど良い）で評価され、ドメイン適応と推論効率が重要な課題となります。

---

**全体的な傾向**

BirdCLEF 2024では、前年と同様にCNNベースのアーキテクチャが主流でしたが、評価指標がAUCに変更されたこと、そして大量のラベルなしサウンドスケープデータが提供されたことから、新たな戦略が求められました。

最大のポイントは**ラベルなしデータの活用**であり、ほとんどの上位チームが**疑似ラベリング (Pseudo Labeling)** を導入しました。強力な外部モデル（Google Bird Classifierなど）や、自分たちで訓練したモデル（特にアンサンブル）を用いてラベルなしデータにラベルを付け、これを訓練データに追加することで、ドメイン適応と性能向上を図りました。

モデルアーキテクチャは多様化し、EfficientNet系、RegNetY、ResNe(X)t系、NFNet系、EfficientViT、InceptionNeXtなどが採用されました。SED（Sound Event Detection）構造だけでなく、単純なCNNバックボーンに分類ヘッドをつけた構成も上位に入りました。また、メルスペクトログラムを入力とするモデルに加え、**生の音声波形 (Raw Signal)** を直接入力とするモデルも有効性が示されました。

訓練データについては、提供された `train_audio` の**フィルタリング**（低品質データやラベル不一致の除去）や、**入力チャンクの工夫**（5秒だけでなく10秒入力で時間的文脈を取り込むなど）が行われました。AugmentationはMixupやCutMix、ノイズ付与などが使われましたが、前年ほど複雑化せずシンプルな構成のチームも見られました。

**知識蒸留 (Knowledge Distillation)** も引き続き有効なテクニックとして用いられました。また、希少種の性能を向上させるために**種をサブセットに分割して学習**するアプローチや、ドメイン適応のための**Contrastive Adversarial Domain (CAD) bottleneck** といった新しい手法も試されました。

後処理では、時間的に**隣接するチャンクの予測を用いてスムージング**する手法が多くのチームで採用され、スコア向上に貢献しました。

**アンサンブル**は依然として重要で、異なるモデルアーキテクチャ、異なる入力形式（メルスペクトログラム/生波形）、異なる学習データ（疑似ラベルの有無や種類）、異なるFoldのモデルを組み合わせることで、最終的な精度と頑健性を高めていました。平均化だけでなく、最小値 (`min()`) や幾何平均を用いたアンサンブルも効果を発揮しました。

**推論効率化**のため、**OpenVINO**への変換が広く採用され、ONNXよりも高速であるとの報告が多くありました。INT8量子化も有効な手段として活用されました。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/birdclef-2024/discussion/512197)**

* **アプローチ:** EfficientNet/RegNetYアンサンブル。疑似ラベル活用。独自のデータ分割と後処理。
* **アーキテクチャ:** EfficientNet_b0, RegNetY_008。
* **アルゴリズム:** 画像分類 (CNN)。
* **テクニック:** データ: 2024 train + pseudo labeled unlabeled。データフィルタリング (Google分類器)。疑似ラベル生成 (Google分類器+自モデルアンサンブル)。Fold分割の工夫 (統計量に基づく)。10秒入力 (中心5秒+前後2.5秒)。CE Loss (BCEより優位)。Augmentation (XYマスキング, CutMix)。後処理: sigmoid + 隣接チャンク平均 + **min()によるアンサンブル**。OpenVINO + 並列処理で高速化。

**[2位](https://www.kaggle.com/competitions/birdclef-2024/discussion/512340)**

* **アプローチ:** EfficientNet-b0アンサンブル。疑似ラベル活用 (複数イテレーション)。
* **アーキテクチャ:** tf_efficientnet_b0_ns。モデルヘッドに複数Dropout層追加。
* **アルゴリズム:** 画像分類 (CNN)。
* **テクニック:** データ: 2024 train + pseudo labeled unlabeled (複数回更新)。**疑似ラベル**活用が鍵。5秒入力。多様性確保のためMelパラメータ、画像サイズ、疑似ラベル追加確率などを変更したモデルをアンサンブル。Augmentation (Mixup, Flip, CoarseDropout)。Checkpoint Soup (複数エポックの重み平均)。後処理: 隣接チャンクスムージング。

**[3位](https://www.kaggle.com/competitions/birdclef-2024/discussion/511905)**

* **アプローチ:** 2段階学習 (疑似ラベル生成→最終モデル学習)。EfficientViTなど高速モデル活用。
* **アーキテクチャ:** Level 0: EfficientViT, EfficientNet, MobileNet, TinyNet, MixNet, Avesなど多様。Level 1: EfficientViT-b0, MnasNet-100。
* **アルゴリズム:** 画像分類 (CNN/ViT)、音声波形分類 (Aves)。
* **テクニック:** データ: 2024 train + 過去データ + XC追加データ + pseudo labeled unlabeled。データ数キャップ (種ごと500)。希少種アップサンプリング。疑似ラベルデータの追加方法に工夫 (別バッチ/同バッチ)。secondary labelは損失マスク。後処理: Soundscapeレベル最大値に基づく調整、隣接チャンクスムージング。OpenVINO使用 (精度低下懸念あり)。

**[4位](https://www.kaggle.com/competitions/birdclef-2024/discussion/511845)**

* **アプローチ:** メルスペクトログラムCNNと**生波形CNN**のアンサンブル。TTA、量子化。
* **アーキテクチャ:** Melspec: rexnet_150, seresnext26ts, inception-next-nano (カスタム)。Raw Signal: tf_efficientnet_b0_ns + SED Head。
* **アルゴリズム:** 画像分類 (CNN)、1D信号分類 (CNN)。
* **テクニック:** データ: 2024 train + 過去データ (一部事前学習)。Raw Signalモデルは1Dデータを2D画像形式に変形して入力。TTA (2.5秒シフト)。後処理: スムージング、Cut-off。OpenVINO + **INT8 Post Training Quantization** (学習データでキャリブレーション)。合成検証データセット構築。

**[5位](https://www.kaggle.com/competitions/birdclef-2024/discussion/511535)**

* **アプローチ:** **生波形モデル** + スペクトログラムモデル + Mixモデルのアンサンブル。
* **アーキテクチャ:** Raw Signal: HGNet-b0。Spectrum: EfficientNet-b0。Mix: HGNet-b0 + EfficientNet-b0 の特徴量結合。
* **アルゴリズム:** 1D信号分類、画像分類、特徴量結合。
* **テクニック:** データ: 2024 trainのみ。Raw Signalモデルは1Dデータを2D画像形式に変形して入力。Augmentation (CutMix, Mixup)。OpenVINO使用。

**[6位](https://www.kaggle.com/competitions/birdclef-2024/discussion/511527)**

* **アプローチ:** SED/CNNモデルアンサンブル。疑似ラベル活用。後処理。
* **アーキテクチャ:** ResNet18d, ResNet34d, EfficientNetV2s。
* **アルゴリズム:** SED、画像分類 (CNN)。
* **テクニック:** データ: 2024 train + pseudo labeled unlabeled。疑似ラベル学習時の損失計算工夫。後処理: **移動平均スムージング** + 全体平均加算。RMSによる音声サンプリング。

**[7位](https://www.kaggle.com/competitions/birdclef-2024/discussion/511540)**

* **アプローチ:** SED/CNNモデルアンサンブル。疑似ラベル生成に外部モデル活用。知識蒸留。sumixup。**種サブセット学習**。
* **アーキテクチャ:** SED (tf_efficientnetv2_s_in21k, seresnext26t_32x4d), CNN (resnet34d)。
* **アルゴリズム:** SED、CNN、知識蒸留。
* **テクニック:** データ: 2024 train + XC追加データ + pseudo labeled unlabeled (BirdNet/Google分類器/自モデルで抽出)。**知識蒸留** (2023 4位手法)。**sumixup** Augmentation (2023 7位手法)。**種サブセット分割学習** (希少66種モデルをアンサンブルに加える)。INT8量子化 (NNCF)。

**[8位](https://www.kaggle.com/competitions/birdclef-2024/discussion/511528)**

* **アプローチ:** SED/CNNモデルアンサンブル。疑似ラベル活用。sumix Augmentation。
* **アーキテクチャ:** SED (eca_nfnet_l0, tf_efficientnet_b0), Simple 2D CNN (eca_nfnet_l0)。
* **アルゴリズム:** SED、CNN。
* **テクニック:** データ: 2024 train + 過去データ + ff1010bird + pseudo labeled unlabeled。疑似ラベル学習。**sumix** Augmentation。データフィルタリング (Google分類器)。OpenVINO使用。

**[9位](https://www.kaggle.com/competitions/birdclef-2024/discussion/511510)**

* **アプローチ:** SEDモデルアンサンブル。**Contrastive Adversarial Domain (CAD)** によるドメイン適応。
* **アーキテクチャ:** SED (eca_nfnet_l0, tf_efficientnetv2_b0, rexnet_100)。Bottleneck層追加。
* **アルゴリズム:** SED、Contrastive Learning、Domain Adaptation。
* **テクニック:** データ: 2024 train + unlabeled (CAD用)。**CAD bottleneck**: ドメインラベル (train/unlabeled) を用いてSupCon lossを追加し、ドメイン不変な特徴量学習を目指す。Augmentation (CutMix, Mixup)。後処理: 隣接チャンク平均。Public LBのみでチューニング。

**[10位](https://www.kaggle.com/competitions/birdclef-2024/discussion/511596)**

* **アプローチ:** CNNモデルアンサンブル。3段階学習。疑似ラベル活用。
* **アーキテクチャ:** tf_efficientnetv2_b0, b3。
* **アルゴリズム:** 画像分類 (CNN)。
* **テクニック:** 3段階学習 (過去データ→2024データ→2024+pseudo labeled unlabeled)。疑似ラベル活用の工夫 (ノイズとして加算、低エントロピーサンプル選択、ソフトラベル化)。
