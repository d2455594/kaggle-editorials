---
tags:
  - Kaggle
  - 病理学
startdate: 2023-10-07
enddate: 2024-01-04
---
# UBC Ovarian Cancer Subtype Classification and Outlier Detection (UBC-OCEAN)
https://www.kaggle.com/competitions/UBC-OCEAN

**全体的な傾向:**

このコンペでは、卵巣がんのホールスライド画像（WSI）および組織マイクロアレイ（TMA）画像から、5つの主要な組織学的サブタイプ（HGSC, LGSC, EC, CC, MC）を分類し、さらにそれらに当てはまらない「Other」（正常組織や希少なサブタイプなど）を外れ値として検出することが課題です。データセットには解像度の異なるWSI（低解像度、広範囲）とTMA（高解像度、狭範囲）が含まれ、特に巨大なWSI画像を効率的に処理する必要がありました。

上位解法は、主に**Multiple Instance Learning (MIL)** のアプローチを採用しています。これは、画像を多数のタイル（パッチ）に分割し、各タイルの特徴量を抽出した後、それらを統合して画像全体の分類を行う手法です。特徴抽出には、**事前学習済みモデル**（特に病理画像用に訓練されたPhikon, Lunit-DINO, CTransPathなど）が広く利用されました。MILの集約モデルとしては、**Chowder, CLAM, ABMIL, DSMIL, TransMIL, Perceiver** などが使われています。

WSIとTMAの**解像度の違いに対処する工夫**（TMAのダウンサンプリング、個別のアプローチ）や、**効率的なタイリング**（組織領域の検出、PyVipsなどのライブラリ活用、カスタムコード）も重要でした。また、**外部データの活用**（特にTMAデータや"Other"クラスに相当するデータ）が精度向上に寄与した解法が多く見られます。**外れ値検出**には、予測確率の閾値処理、予測エントロピー、補助ラベルの利用、ArcFaceによる距離測定などが用いられました。**モデルのアンサンブル**は、安定性と精度向上のために広く採用されています。データ拡張（特にStain Augmentation）、クラス不均衡への対処（重み付きサンプリング、損失関数の重み付け）、疑似ラベリングなどのテクニックも活用されました。

**各解法の詳細:**

**1位**

- **アプローチ:** Owkinの病理学用基盤モデルPhikonで抽出したタイル特徴量上で、MILモデルChowderのアンサンブルを訓練。予測エントロピーで外れ値検出。外部データ不使用。
- **アーキテクチャ:** Phikon (ViT-Base), Chowder (MIL)。
- **アルゴリズム:** Otsu thresholding (組織検出), AdamW, CrossEntropyLoss, WeightedRandomSampler, Prediction Entropy (外れ値検出)。
- **テクニック:**
    - **データ準備:** 組織検出(Otsu)、WSI/TMAの解像度統一（TMAを448pxでタイリング後224pxにリサイズ）、TMA検出(ResNet18+LR)、効率的なタイリング(カスタムCコード+libpng)、並列処理(Ray)。
    - **モデリング:** Phikonで特徴抽出、Chowderアンサンブル(N=50)、クラス不均衡対策(重み付きサンプリング、クラス重み付きCE損失)。
    - **推論/後処理:** 複数Fold/Seedから選択した65モデルの平均予測、LRによるキャリブレーション、モデルフィルタリング、予測エントロピー閾値による外れ値検出(LBで閾値調整)。成功した別アイデア：Phikonファインチューニング、予測分散による外れ値検出。

**2位**

- **アプローチ:** 標準的なMILパイプライン。複数の特徴抽出器(Lunit-DINO)とMILモデル(ABMIL, DSMIL, TransMIL)を試行・組み合わせ。
- **アーキテクチャ:** Lunit-DINO (ViT-small patch16/patch8), ABMIL, DSMIL, TransMIL。
- **アルゴリズム:** (詳細な言及なし)。
- **テクニック:**
    - **データ準備:** WSI/TMAごとにDataset作成(pyvips使用)、WSIサムネイルから組織領域を推定しタイル座標リスト作成、WSIサイズに応じてタイルサンプリング率調整(R_ratio)。
    - **モデリング:** 2種の特徴抽出器と3種のMILモデルの組み合わせ。
    - **推論/後処理:** (詳細な言及なし)。外部データ使用(効果薄と判断)。

**3位**

- **アプローチ:** 外部データの積極的な活用と、提供されたセグメンテーションマスクを用いた合成TMA画像生成。Lunit-DINO特徴抽出器とCLAMモデル(AttentionベースMIL)を使用。
- **アーキテクチャ:** Lunit-DINO, CLAM。
- **アルゴリズム:** Attention mechanism (CLAM), Instance-level loss (CLAM)。
- **テクニック:**
    - **データ準備:** 複数ソースから外部データ収集、合成TMA画像生成(癌組織タイル + 健常/間質タイルを"Other"として)、PyVipsと非同期ローディングでタイル処理。
    - **モデリング:** Lunit-DINOで特徴抽出(16bit精度)、CLAMモデル訓練("Other"ラベル用にインスタンスレベル損失を修正)。
    - **推論/後処理:** (詳細な言及なし)。5-Fold CV、患者単位でのFold分割、特定データセットを除外した検証。

**4位**

- **アプローチ:** セグメンテーションモデルによるタイル選択と、タイル分類モデルによるスコア予測の2段階構成。WSIとTMAで一部処理を分ける。
- **アーキテクチャ:** ConvNeXt, Hornet, EfficientNetV1/V2 (分類器バックボーン), Segmentation model。
- **アルゴリズム:** Binary CrossEntropy (BCE), Sigmoid activation。
- **テクニック:**
    - **データ準備:** TMAは中央クロップ＆リサイズ(768x768)。WSIはセグメンテーションモデルで癌確率が高い領域をクロップ＆リサイズ(1536x1536 -> 768x768)。問題のあるサムネイルは再生成。
    - **モデリング:** 16個の分類モデルアンサンブル(5クラス+非癌クラスの6クラス分類)、データ拡張(Stain augmentation含む)。
    - **推論/後処理:** 予測スコアの中央値でアンサンブル、低スコア(<0.05)または下位%タイルを"Other"と判定。GPU分散。外部データ不使用。

**5位**

- **アプローチ:** WSI用のタイル選択セグメンテーションモデルと、タイル分類モデルを用いた2段階推論。外部データとデータマイニング(疑似ラベリング)を活用。
- **アーキテクチャ:** ConvNeXt-base/large, EVA (分類器バックボーン), SEResNeXt101 UNet (セグメンテーション)。
- **アルゴリズム:** RandAugment, Stain Normalization (StainNorm), Pseudo-labeling。
- **テクニック:**
    - **データ準備:** ランダムクロップ(1536x1536)、TMAは中央クロップ＆リサイズ。
    - **モデリング:** タイル選択モデル(補助モデル予測を組み合わせたヒートマップ教師ラベルでUNet訓練)、分類モデル(WSIラベルをタイルラベルに使用、外部データを"Other"として利用、データマイニングで疑似ラベル生成し2段階訓練)、データ拡張(StainNorm含む)。
    - **推論/後処理:** WSIは選択されたTop5タイルの予測平均、TMAは中央クロップタイルの予測、TTAとしてStainNorm使用。

**6位**

- **アプローチ:** iBOT事前学習済みViT-Base特徴抽出器とChowderモデル(MIL)のアンサンブル。予測エントロピーによる外れ値検出。
- **アーキテクチャ:** iBOT-ViT-Base, Chowder。
- **アルゴリズム:** Prediction Entropy (外れ値検出)。
- **テクニック:**
    - **データ準備:** WSI/TMAからランダムに1000パッチ選択。
    - **モデリング:** iBOT-ViT-Baseで特徴抽出、Chowderモデル訓練。
    - **推論/後処理:** 7つのChowderモデルのアンサンブル、予測の平均エントロピーで"Other"検出（閾値調整）。

**7位**

- **アプローチ:** MILベース。CTransPath/LunitDINO特徴抽出器とDSMIL/Perceiver分類器の組み合わせ。Sigmoid出力の閾値処理または確率エントロピーで外れ値検出。
- **アーキテクチャ:** CTransPath, LunitDINO, DSMIL, Perceiver, pyvips。
- **アルゴリズム:** CrossEntropyLoss (DSMIL), BCEWithLogitsLoss (Perceiver), Mixup, Label Smoothing, Prediction Entropy (外れ値検出)。
- **テクニック:**
    - **データ準備:** pyvips使用、x10解像度にダウンサンプリング、WSIの重複領域除去、タイリング(256x256)。
    - **モデリング:** 2種の特徴抽出器と2種のMIL分類器の組み合わせ。PerceiverでMixupとLabel Smoothing使用。
    - **推論/後処理:** 4モデルのアンサンブル。外れ値検出(最大予測確率<0.4 or 予測確率エントロピー>閾値)。外部データ不使用。

**8位**

- **アプローチ:** WSIとTMAを完全に分離。TMAはパッチ分類+ArcFace検索、WSIはMIL(DTFD-MIL)。TMA用モデルをWSIの特徴抽出器として活用。
- **アーキテクチャ:** EfficientNetV2 (s/l), ConvNeXt (small/large), ArcFace head, DTFD-MIL。
- **アルゴリズム:** ArcFace retrieval, DTFD-MIL。
- **テクニック:**
    - **データ準備:** TMAは公式マスクでタイリング、WSIはタイル抽出(複数解像度)。TMAをダウンサンプリングしWSIとスケール統一。
    - **モデリング(TMA):** パッチ分類器訓練(5クラス+"Other")、疑似ラベル生成、ArcFaceモデル訓練。
    - **モデリング(WSI):** TMA用モデルを特徴抽出器として使用、"Other"パッチプール作成、DTFD-MIL訓練時に"Other" WSIを動的合成。
    - **推論(TMA):** ArcFace検索 -> 6クラス分類アンサンブル。
    - **推論(WSI):** 複数解像度の特徴抽出器+DTFD-MILアンサンブル。マルチスレッド処理。

**9位**

- **アプローチ:** WSI/TMA識別と個別タイル化戦略。タイルベース分類。補助ラベルを用いた3段階訓練。
- **アーキテクチャ:** EfficientNetB4, EfficientNetV2s, MaxViT-tiny。
- **アルゴリズム:** Binary CrossEntropy (BCE)。
- **テクニック:**
    - **データ準備:** WSI/TMA識別(サイズと黒ピクセル率)、WSIタイル化(0.33倍リサイズ後512x512、不良ピクセル率で段階的サンプリング)、TMAタイル化(中央クロップ後512x512リサイズ)。
    - **モデリング:** WSIタイルのみ使用、バッチ内でタイルランダム選択、3段階訓練(通常訓練→補助ラベル生成＆再訓練→Fine-tuning)。
    - **推論/後処理:** WSIはタイル予測アンサンブル、TMAはリサイズ後予測、外れ値検出(補助ラベル予測平均<0.5)、Votingアンサンブル。外部データ不使用。

**10位**

- **アプローチ:** 外部TMAデータと強力なラベルを活用したタイルベース分類。WSI用にセグメンテーションモデルを利用。
- **アーキテクチャ:** EfficientNetV2-small, MaxViT-Tiny FPN (セグメンテーション)。
- **アルゴリズム:** CrossEntropyLoss (重み付き), AdamW, CosineAnnealingLR。
- **テクニック:**
    - **データ準備:** WSI用にセグメンテーションモデル訓練、マスクに基づきTop16タイル選択(1024x1024)。TMAは白領域除去。外部データ多数活用。
    - **モデリング:** EfficientNetV2-small訓練、豊富なデータ拡張。
    - **推論/後処理:** 5-Foldアンサンブル、3x TTA、WSIは16タイルの予測平均。
