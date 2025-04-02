---
tags:
  - Kaggle
  - ヘルスケア
  - WeightedLogLoss
startdate: 2023-07-27
enddate: 2023-10-16
---
# RSNA 2023 Abdominal Trauma Detection
[https://www.kaggle.com/competitions/rsna-2023-abdominal-trauma-detection](https://www.kaggle.com/competitions/rsna-2023-abdominal-trauma-detection)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、**腹部外傷**患者の**CT（コンピュータ断層撮影）スキャン画像**を解析し、肝臓、脾臓、腎臓（左右区別なし）、腸といった主要臓器の**損傷の有無および重症度**（腎臓・肝臓・脾臓のみ、healthy/low/highの3段階）、さらに**活動性血管外漏出（腹腔内出血の兆候）の有無**を自動的に検出・分類する機械学習モデルを開発することです。
* **背景:** 腹部への鈍的または鋭的損傷は、救急医療において頻繁に見られ、内出血や臓器損傷は生命を脅かす可能性があります。CTスキャンは腹部外傷診断の標準的な検査法ですが、特に緊急時には迅速かつ正確な読影が求められます。AIを活用してCT画像から自動的に損傷を検出し、重症度を評価することで、診断時間の短縮、放射線科医の負担軽減、診断精度向上、そして最終的には患者の救命率向上に貢献することが期待されます。
* **課題:**
    * **複数ターゲットの予測:** 患者全体に対する5つの損傷有無（肝臓、脾臓、腎臓、腸、血管外漏出）と、3つの臓器（肝臓、脾臓、腎臓）それぞれの3段階の重症度分類、合計11個のターゲットを同時に予測する必要があります。
    * **3Dデータの扱い:** CTデータは多数の2Dスライスから構成される3Dボリュームデータであり、スライス間の連続性や3次元的な損傷の広がりを捉えるモデル構造が必要です。
    * **データの不均衡:** 重度の損傷や特定の臓器の損傷、特に血管外漏出は発生頻度が低く、クラス不均衡の問題があります。
    * **微妙な所見の検出:** 低悪性度の臓器損傷や少量の血管外漏出は、画像上での所見が微妙で見逃しやすい可能性があります。
    * **複数シリーズの存在:** 一人の患者に対して複数のCTシリーズ（例: 造影前、動脈相、門脈相など）が存在する場合があり、それらを適切に扱う必要があります。

**データセットの形式 (Dataset Format)**

提供される主なデータは、腹部CTスキャン画像（DICOM形式）と、それに対応する患者レベルおよび画像レベルの損傷ラベルです。

1.  **CT画像データ:**
    * `train_images/`, `test_images/`: 患者 (`patient_id`) ごとに、1つまたは複数のCTシリーズ (`series_id`) がディレクトリとして格納されています。
    * 各シリーズディレクトリ内には、多数の **DICOMファイル (`.dcm`)** が含まれ、それぞれが軸位断のCTスライス画像を表します。DICOMヘッダーには撮影条件などのメタ情報が含まれます。
2.  **ラベルデータ (Training Labels):**
    * `train.csv`: **患者レベル**の損傷ラベル。各 `patient_id` に対して以下のターゲットカラムがあります。
        * `bowel_injury`: 腸損傷の有無 (0 or 1)。
        * `extravasation_injury`: 活動性血管外漏出の有無 (0 or 1)。
        * `kidney_healthy`, `kidney_low`, `kidney_high`: 腎臓の状態（3クラスのone-hot表現）。
        * `liver_healthy`, `liver_low`, `liver_high`: 肝臓の状態（3クラスのone-hot表現）。
        * `spleen_healthy`, `spleen_low`, `spleen_high`: 脾臓の状態（3クラスのone-hot表現）。
    * `image_level_labels.csv`: **画像（スライス）レベル**の損傷ラベル。各スライス (`patient_id`, `series_id`, `instance_number`) に対して、`Bowel` または `Extravasation` の損傷が存在するかどうかを示します。これは補助的な教師信号として利用できます。
    * `train_series_meta.csv`: 各シリーズの `patient_id` などのメタ情報。
3.  **任意データ:**
    * **セグメンテーションマスク:** 主催者からは提供されませんが、多くのチームがTotalSegmentatorなどのツールや公開データセットを用いて、肝臓、脾臓、腎臓などの臓器マスクを生成し、臓器のクロップ（切り出し）や補助タスクに利用しました。
    * **バウンディングボックス:** 血管外漏出の位置を示すバウンディングボックスデータも一部参加者によって作成・共有され、利用されました。
4.  **`sample_submission.csv`**:
    * 提出ファイルのフォーマット。`patient_id` と、予測対象となる11個のターゲットクラス（`bowel_injury`, `extravasation_injury`, `kidney_healthy`, ... `spleen_high`）それぞれの予測確率の列を持ちます。各臓器のhealthy/low/highの確率は合計1になるように正規化する必要があるかもしれません（要確認）。

**評価指標 (Evaluation Metric)**

* **指標:** **重み付き対数損失 (Weighted Log Loss)**
* **計算方法:** 各患者について、11個のターゲットクラスそれぞれの予測確率 $$ p_{ij} $$ と真のラベル $$ y_{ij} $$ (0 or 1) を用いて対数損失を計算し、クラスごとに定められた重み $$ w_j $$ を掛けて合計します。これを全患者 $$ N $$ で平均します。
    $$ \text{Score} = - \frac{1}{N} \sum_{i=1}^{N} \sum_{j=1}^{11} w_j y_{ij} \log(\max(\min(p_{ij}, 1 - 10^{-15}), 10^{-15})) $$
    * **重み ($$w_j$$):** `bowel_injury`: 1, `extravasation_injury`: 6, `kidney_healthy`: 1, `kidney_low`: 2, `kidney_high`: 4, `liver_healthy`: 1, `liver_low`: 2, `liver_high`: 4, `spleen_healthy`: 1, `spleen_low`: 2, `spleen_high`: 4。合計11クラス。
    * **any_injury:** 提出ファイルには含まれませんが、評価時には患者がいずれかの臓器損傷（healthy以外）または血管外漏出を持つ場合に1となる `any_injury` ラベルが内部的に計算され、重み6でスコアに加算されます。
* **意味:** 各損傷カテゴリの予測確率と真のラベルとの間の乖離度を測る指標です。予測が真のラベルに近く、かつ自信度（確率）が高いほど損失は小さくなります。クラスごとに重みが設定されており、特に**血管外漏出 (`extravasation_injury`) と `any_injury` の重みが6と非常に高く**、これらの予測精度が全体のスコアに大きく影響します。スコアは**小さい**ほど、モデルの性能が高いことを示します。

要約すると、このコンペティションは、腹部CT画像から複数の臓器損傷と血管外漏出の有無・重症度を予測するマルチラベル分類タスクです。データは3D DICOM画像と患者・画像レベルのラベルで構成され、性能は重み付き対数損失（小さいほど良い）によって評価されます。

---

**全体的な傾向**

この腹部外傷検出コンペティションでは、3DのCTボリュームデータから複数の損傷タイプと重症度を予測する必要があり、多くのチームが**2.5Dアプローチ**を採用しました。これは、3Dデータを直接扱う代わりに、連続する複数の2Dスライス情報を組み合わせて利用する手法です。

1.  **アーキテクチャ:** 主流は **2D CNN + シーケンスモデル (RNN/Transformer)** の組み合わせでした。
    * **2D CNN:** EfficientNet (V2Sなど), ConvNeXt (V2 Tinyなど), MaxViT, CoAtNet, ResNe(X)t, Pyramid Vision Transformer (PVT), InternImage など、多様なCNNバックボーンがスライスごとの特徴抽出に用いられました。
    * **シーケンスモデル:** CNNで抽出したスライス特徴量のシーケンスを集約し、患者レベルの予測を行うために **LSTM** や **GRU** (特に双方向) が多く使われました。Transformer Encoderも利用されました。
2.  **3Dセグメンテーションの活用:** 多くのトップチームが、学習や推論の**前処理**として **3Dセグメンテーション** を行い、肝臓・脾臓・腎臓などの関心臓器領域を特定しました。これにより、モデルの入力データを臓器周辺に限定する**クロップ（切り出し）**が可能になり、計算効率と予測精度の向上に貢献しました。MONAI SwinUNETRやResNetベースの3D U-Net、あるいはTotalSegmentatorなどの外部ツールが利用されました。
3.  **マルチタスク学習と補助損失:** 分類タスクと同時に、セグメンテーションマスクの予測や画像レベルでの損傷有無の予測を**補助タスク (Auxiliary Task)** として学習させる手法が有効でした。これにより、モデルがより豊かな表現を獲得し、学習が安定化、精度が向上しました。特に、セグメンテーションマスクに対するDice Lossなどを補助損失として加える戦略が多く見られました。
4.  **データ前処理:**
    * **ウィンドウ処理:** CT値に対するウィンドウ処理（Soft tissue windowなど）は標準的に行われました。複数ウィンドウをチャンネルとして利用するアプローチもありました。
    * **リサンプリング/スライス選択:** 患者ごとに異なるスライス数やスライス間隔を統一するため、リサンプリングや等間隔でのスライス選択が行われました。
    * **解像度:** 384x384や512x512などの比較的高解像度で学習されました。
    * **クロッピング:** 臓器クロップの他に、画像の黒い背景領域を除去するクロップも行われました。
5.  **ソフトラベル/可視性:** 1位のチームは、各スライスにおける臓器の「見え具合（可視性）」をセグメンテーションマスクから算出し、これを患者レベルの損傷ラベルに乗じることで**ソフトラベル**を作成し、学習ターゲットとしました。
6.  **アンサンブル:** 最終的なスコア向上のため、異なるアーキテクチャ、異なるFold、異なる前処理、異なるクロップサイズなどで学習された**多数のモデルの予測をアンサンブル**することが一般的でした。単純平均や加重平均が用いられました。
7.  **後処理:** 予測確率のスケーリング（係数乗算、平方根など）や、スタッキングモデル（MLPなど）による最終調整も行われました。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/rsna-2023-abdominal-trauma-detection/discussion/447449)**

* **アプローチ:** 3Dセグメンテーションによる臓器クロップ + 2.5D CNN+GRU。**ソフトラベル**学習。**補助セグメンテーション損失**。
* **アーキテクチャ/アルゴリズム:**
    * セグメンテーション: 3Dモデル（詳細は不明だがマスク生成用）。
    * 分類: 2D CNN (CoAtLite Medium/Small, EfficientNetV2S) + GRU。
    * 損失: BCE Loss (分類) + Dice Loss (補助セグメンテーション)。
* **テクニック:**
    * **データ準備:** 臓器マスクに基づきStudyレベルでクロップ。96スライスを等間隔抽出し、32x3chにreshape (2.5D)。
    * **ソフトラベル:** 各スライスの臓器マスク面積から臓器可視性を計算し、患者レベルの損傷ラベルに乗じてソフトターゲットを作成。
    * **補助損失:** CNNバックボーンの中間特徴マップからU-Net DecoderやシンプルなCNNヘッドでセグメンテーションマスクを予測し、Dice Lossを追加。
    * **アンサンブル:** 複数Fold、複数アーキテクチャのモデルをスライスレベルでアンサンブル後、Maxプーリングで患者レベル予測。

**[2位](https://www.kaggle.com/competitions/rsna-2023-abdominal-trauma-detection/discussion/447453)**

* **アプローチ:** 2D CNN+RNN と 臓器クロップ+2.5D CNN+RNN のハイブリッド。臓器存在確率の活用。
* **アーキテクチャ/アルゴリズム:**
    * 臓器存在確率推定: EfficientNetV2-t (2D)。
    * 全体分類: MaxViT-tiny (+LSTM)。
    * 臓器別分類(クロップ): 3D ResNet18 (クロップ用) + CoAtNet-1 (+LSTM/Attention/Transformer)。
    * RNNアグリゲーション: カスタムLSTM（臓器別プーリング、Attention）。
* **テクニック:**
    * **データ準備:** 512pxリサイズ->中央384pxクロップ。2スライスごとサンプリング。
    * **2Dモデル:** 臓器存在確率モデルで各スライスでの臓器有無を予測。それを基にフレームサンプリング（臓器があるスライス中心）や特徴量プーリング（臓器領域のみ）を実施。11クラス分類（臓器別3クラス + 腸/血管外漏出）。
    * **クロップモデル:** 3D ResNet18で臓器をクロップ後、2.5D CNN+RNNで臓器別に3クラス分類。
    * **RNN集約:** 2Dモデルの特徴量・確率とクロップモデルの特徴量を入力とし、LSTMで患者レベル予測。臓器ごとに独立したロジットを持つ。
    * **その他:** CutMix、重いAugmentation使用。

**[3位](https://www.kaggle.com/competitions/rsna-2023-abdominal-trauma-detection/discussion/447464)**

* **アプローチ:** 3Dセグメンテーション + 臓器別クロップ (2サイズ) + 2.5D CNN+シーケンスモデル。全ターゲット同時予測。
* **アーキテクチャ/アルゴリズム:**
    * セグメンテーション: 3D ResNet18/50 U-Net (Qishen氏コード)。
    * 分類: 2D CNN (ConvNeXt, SE-ResNeXt, MaxViT, CaFormer, XCiT) + Pooling/LSTM/GRUネック。
* **テクニック:**
    * **クロップ:** セグメンテーションマスクを拡張し、2種類のサイズの臓器キューブを作成。
    * **入力:** 2.5D形式 (例: 15x3x128x128)。
    * **マルチターゲット:** 全11ターゲット（+補助タスク）を同時に予測。
    * **肝臓モデル:** マスクを入力チャンネルに追加しノイズ低減。
    * **全臓器モデル:** 異なる臓器のクロップ（サイズ/枚数調整）を同一バッチで学習するためのカスタムサンプラー。
    * **アンサンブル:** 多数のモデル（バックボーン、ネック、画像サイズ、枚数、クロップサイズ、Augmentation、エポック違い）の加重平均。
    * **後処理:** 予測確率のスケーリング。

**[4位](https://www.kaggle.com/competitions/rsna-2023-abdominal-trauma-detection/discussion/447848)**

* **アプローチ:** 3Dセグメンテーション + 2.5D U-Net (Transformer Encoder)。セグメンテーションと分類のマルチタスク。
* **アーキテクチャ/アルゴリズム:** Unet (Encoder: Pyramid Vision Transformer V2 (PVT), MaxViT)。6つの分類ヘッド。
* **テクニック:**
    * **データ準備:** 3Dマスク (TotalSegmentator + 2D再学習) を使用。DICOMリスケール (1,1,5)。5チャンネルマスク (肝臓, 脾臓, 腎臓, 腸, 体)。
    * **入力:** 14フレームをサンプリング。体マスクで不要領域除去。臓器マスクでZ軸範囲を限定。
    * **マルチタスク:** セグメンテーション (マスク予測) と分類 (6ヘッド) を同時に学習。
    * **損失:** CE Loss (メトリックと同じ重み付け)。
    * **アンサンブル:** 異なるEncoder (PVT-B2/B3/B4, MaxViT-T) のモデルをアンサンブル。後処理でスケーリング。

**[6位](https://www.kaggle.com/competitions/rsna-2023-abdominal-trauma-detection/discussion/448208)**

* **アプローチ:** 3Dセグメンテーション + 臓器別/目的別モデル (Organ, Bowel, Extravasation)。2.5D CNN + シーケンスモデル。
* **アーキテクチャ/アルゴリズム:**
    * セグメンテーション: 3D ResNet18d U-Net。
    * Organ/Bowel: 2.5D CNN (EfficientNetV2S, SE-ResNeXt50) + LSTM。
    * Extravasation: 2段階。1段目: 2.5D CNN (SE-ResNeXt50/EffNetV2S) 特徴抽出 (+セグメンテーション補助損失)。2段目: GRU。
* **テクニック:**
    * **クロップ:** Organモデルは15スライス、Bowelモデルは30スライスをクロップ。Extravasationはクロップなし。
    * **入力:** 2.5D (5チャンネル: [n-2, n-1, n, n+1, n+2])。
    * **Bowelモデル:** 患者ラベルと画像レベルラベル両方使用。
    * **Extravasationモデル:** 補助タスクとして、公開bboxラベルを用いたセグメンテーション損失を追加。

**[7位](https://www.kaggle.com/competitions/rsna-2023-abdominal-trauma-detection/discussion/447549)**

* **アプローチ:** 2.5D パイプライン。InternImage + UnetPlusPlus。カスタムヘッド。補助マスク損失。
* **アーキテクチャ/アルゴリズム:** InternImage (Base) + UnetPlusPlus Neck + カスタムヘッド (Organ Head: Mask2Former風Queryベース, Extravasation Head: Image Level Label補助)。
* **テクニック:**
    * **データ準備:** シーケンスサンプリング (T=24/32/48)。有効ピクセル数でクロップ。リサイズ (256x384)。
    * **入力:** [T * 3, H, W]。
    * **補助損失:** TotalSegmentatorで生成した臓器マスクをGround Truthとしてマスク損失を追加。
    * **損失:** 重み付きCE Loss (分類)、マスク損失。
    * **アンサンブル:** 異なるシーケンス長、異なるFoldのモデルを平均。

**[8位](https://www.kaggle.com/competitions/rsna-2023-abdominal-trauma-detection/discussion/447706)**

* **アプローチ:** 臓器別/目的別モデル (Organ, Bowel, Extravasation)。2D CNN特徴抽出 + Transformer。パッチベースExtravasationモデル。カスタムラベル活用。
* **アーキテクチャ/アルゴリズム:**
    * Organ/Bowel: 2D CNN (ConvNeXt-tiny) + Transformer (3層)。
    * Extravasation: 2段階。1段目: パッチCNN (ConvNeXt-tiny) -> Transformer。2段目: Transformer。
* **テクニック:**
    * **データ準備:** 3ウィンドウをチャンネルとしたPNG変換。Cropモデル (MobileNetV3) で黒領域除去。Organ IDモデル (MobileNetV3) で臓器存在推定。
    * **Organモデル:** 自作のスライスレベル損傷ラベルでCNN学習。Transformerで集約。左右腎臓区別。
    * **Bowelモデル:** 提供画像レベルラベルでCNN+Transformer学習。
    * **Extravasationモデル:** 自作bboxラベルを利用。画像を12パッチに分割しCNN+Transformerで画像レベル特徴抽出。さらにTransformerでシリーズレベル予測。
    * **アンサンブル:** 5-Foldモデル平均。後処理で確率スケーリング（平方根）。

**[9位](https://www.kaggle.com/competitions/rsna-2023-abdominal-trauma-detection/discussion/447506)**

* **アプローチ:** 3Dセグメンテーション + 臓器別/目的別モデル (Organ, Bowel, Extravasation)。2段階学習。
* **アーキテクチャ/アルゴリズム:**
    * セグメンテーション: 3D ResNet18d, UNet。
    * 分類: 2段階。1段目: 2D CNN (SE-ResNeXt)。2段目: LSTM + [1D CNN, Attention]。
* **テクニック:**
    * **クロップ:** 臓器モデルはクロップあり。Bowel/Extravasationはクロップなし。
    * **入力:** 2.5D (4チャンネル: [i-1, i, i+1, mask])。
    * **2段階学習 (Bowel/Extravasation):** 1段目で画像レベルラベルを用いて2D CNNを学習。2段目で1段目CNNの特徴量をLSTM等で集約し、患者レベルラベルで学習（1段目重みはFreeze）。
    * **データ選択:** 患者に複数シリーズある場合、aortic_hu値に基づいて選択（Organ: 低い方、Extravasation: 高い方）。
    * **損失:** 重み付き損失（メトリック準拠）。
    * **後処理:** 予測確率のスケーリング（係数乗算）。

**[10位](https://www.kaggle.com/competitions/rsna-2023-abdominal-trauma-detection/discussion/447450)**

* **アプローチ:** 3Dセグメンテーション + 臓器別モデル + Stacking。
* **アーキテクチャ/アルゴリズム:**
    * セグメンテーション: 3D MONAI SwinUNETR。
    * Organ/Bowel: 2.5D CNN + LSTM。
    * Extravasation (tattaka氏担当): 2段階。1段目: ResNetRS50。2段目: BiGRU + Attention Pooling + Max Pooling。
    * Stacking: 4層MLP。
* **テクニック:**
    * **クロップ:** 肝臓/脾臓/腎臓はクロップ。Bowel/Extravasationは黒領域除去のみ。
    * **腎臓モデル:** 左右腎臓をクロップ後、横連結して入力（水平フリップAug/TTA可能に）。
    * **Bowelモデル:** 患者レベルと画像レベルラベル両方使用。
    * **Extravasationモデル:** 1段目画像レベル学習、2段目シリーズレベル学習。画像特徴量はstride=3でサンプリング。隣接特徴量との差分も入力。
    * **Stacking:** 各モデルの予測値を入力とし、MLPで最終予測。評価メトリック（Any Injury含む）を直接損失関数として学習。
    * **学習:** Positiveサンプルをアップサンプリング。Focal Loss。画像レベルラベルの再学習（Max Logitを新ラベルに）。
