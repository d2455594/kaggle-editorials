---
tags:
  - Kaggle
  - 病理学
  - ヘルスケア
  - ViT
  - EfficientNet
startdate: 2023-10-07
enddate: 2024-01-04
---
# UBC Ovarian Cancer Subtype Classification and Outlier Detection (UBC-OCEAN)
[https://www.kaggle.com/competitions/UBC-OCEAN](https://www.kaggle.com/competitions/UBC-OCEAN)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、デジタル化された卵巣がんの生検サンプル画像（Whole Slide Images: WSI および Tissue Microarrays: TMA）を用いて、**卵巣がんの5つの主要なサブタイプ（HGSC, LGSC, EC, CC, MC）を分類**し、さらにそれ以外の**「その他」（良性腫瘍や正常組織など）を外れ値として検出**するモデルを開発することです。
* **背景:** 卵巣がんは複数の組織学的サブタイプに分類され、それぞれ予後や治療法が異なります。正確なサブタイプ分類は個別化医療の実現に不可欠ですが、病理診断医による評価は主観性やばらつきを含む可能性があります。AIを用いた客観的かつ高精度な分類モデルは、診断の一貫性向上や効率化に貢献し、患者の治療成績改善につながることが期待されます。
* **課題:** 提供される画像データは、非常に高解像度なWSIと、より小さなTMAの2種類が混在しており、それぞれの**スケールの違い**を考慮する必要があります。WSIは1枚あたり数万ピクセル四方に及ぶため、画像全体を直接モデルに入力することは計算資源的に困難であり、効率的な**タイリング（パッチ化）戦略**が必要です。また、TMAはトレーニングデータセットでは少数派であり、異なる倍率で撮影されている可能性があります。サブタイプ間の不均衡や、定義された5つのサブタイプ以外の症例（**外れ値**）を区別する必要がある点も課題でした。さらに、異なる施設からのデータ（マルチセントリックコホート）が含まれるため、染色やスキャナーの違いに対する**モデルの汎化性能**も重要でした。

**データセットの形式 (Dataset Format)**

提供される主なデータは、WSI/TMA画像、サムネイル、ラベル、および一部のマスクデータです。

1.  **トレーニングデータ:**
    * `train_images/`: 主要なトレーニングデータであるWSIおよびTMAの画像（PNG形式）。ファイル名に `_thumbnail` が含まれないものがフル解像度画像。
    * `train_thumbnails/`: 上記画像のサムネイル版（低解像度画像、PNG形式）。組織領域の特定などに利用可能。
    * `train.csv`: 各画像（`image_id`）に対応する情報。
        * `image_id`: 画像の一意なID。
        * `label`: 卵巣がんのサブタイプ（`HGSC`, `LGSC`, `EC`, `CC`, `MC` のいずれか）。
        * `image_width`, `image_height`: フル解像度画像の幅と高さ。
        * `is_tma`: 画像がTMAであるかを示すフラグ（True/False）。
    * `train_tile_meta.csv`: 一部のWSI（`kidney_1`と`kidney_3`）について、腫瘍領域などを示すマスク情報に関連するタイルレベルのメタデータ（コンペ後半で追加）。
    * `train_additional_source.csv`: 各画像のデータ提供元施設などの追加情報。
    * `other_images/`: 「その他」クラスに分類される可能性のある画像（良性腫瘍など）のフォルダ。
    * `other_labels.csv`: `other_images/` 内の画像のラベル（すべて `Other`）。
    * （コンペ後半に追加）`train_masks/`: 一部のWSI（`kidney_1`と`kidney_3`）に対する腫瘍領域等のセグメンテーションマスク（PNG形式）。タイル選択や偽陽性領域の学習などに利用可能。
2.  **テストデータ:**
    * `test_images/`: 評価対象となるWSIおよびTMAの画像（PNG形式）。
    * `test_thumbnails/`: 上記画像のサムネイル版。
    * `test.csv`: テスト画像のIDリスト。ラベル情報は含まれない。
3.  **外部データ:**
    * The Cancer Imaging Archive (TCIA) の [Ovarian Bevacizumab Response](https://doi.org/10.7937/TCIA.985G-EY35) や [CPTAC-OV](https://doi.org/10.7937/TCIA.ZS4A-JD58)、[Hamarneh Labのデータセット](https://www.medicalimageanalysis.com/data/ovarian-carcinomas-histopathology-dataset)、[Stanford TMA Database](https://tma.im/cgi-bin/home.pl)、[proteinatlas.org](https://www.proteinatlas.org/) など、公開されている卵巣がん関連の病理画像データセットが、追加学習データや外れ値クラスのデータとして利用されました。
4.  **`sample_submission.csv`**:
    * 提出フォーマットのサンプル。`image_id` と、予測ラベル（`HGSC`, `LGSC`, `EC`, `CC`, `MC`, `Other` のいずれか）の列を持ちます。

**評価指標 (Evaluation Metric)**

* **指標:** **バランス精度 (Balanced Accuracy)**
* **計算方法:** 各クラス（6クラス: 5つのサブタイプ + Other）ごとにRecall（再現率、感度）を計算し、それらの単純平均を取ります。
    * Recall_class = (クラスclassと正しく予測されたサンプル数) / (クラスclassの全サンプル数)
    * Balanced Accuracy = (Recall_HGSC + Recall_LGSC + Recall_EC + Recall_CC + Recall_MC + Recall_Other) / 6
* **意味:** 各クラスのサンプル数が不均衡な場合でも、各クラスに対する予測性能を平等に評価するための指標です。少数派クラス（例: `Other` や一部のサブタイプ）の予測精度も全体のスコアに等しく寄与します。スコアは**高い**ほど良い評価となります。

---

**全体的な傾向**

このコンペティションは、高解像度のWSI/TMA画像を用いた卵巣がんサブタイプ分類と外れ値検出のタスクでした。WSIの巨大さから、多くの解法が**パッチベースのアプローチ**と**Multiple Instance Learning (MIL)** を採用しました。

**前処理**として、まず画像から組織領域を検出し（大津の二値化など）、それを多数の小さな**パッチ（タイル）**に分割します。WSIとTMAの**倍率の違い**（例: 20x vs 40x）を考慮し、TMAのパッチサイズを大きく取るか、画像をリサイズしてスケールを合わせる処理が一般的でした。また、計算効率のため、WSIからは全パッチではなく、ランダムサンプリングや腫瘍領域予測に基づいた**サンプリング**により一部のパッチ（数百〜数千）のみが使用されました。

**特徴抽出**には、強力な**事前学習済みモデル**が不可欠でした。特に、組織病理学分野に特化した自己教師あり学習モデルである**Phikon**, **CTransPath**, **Lunit-DINO**, **PLIP** や、一般的な画像認識モデル (ConvNeXt, EfficientNet, Vision Transformer (ViT) など) がエンコーダーとして用いられ、各パッチから特徴ベクトルを抽出しました。

抽出されたパッチ特徴量を集約して画像全体の分類を行う**MILモデル**としては、古典的な**Chowder**, **DeepMIL**, **ABMIL** から、比較的新しい**TransMIL**, **DSMIL**, **DTFD-MIL**, **Perceiver**、さらにはAttentionベースの**CLAM**などが使用されました。これらのMILモデルを複数組み合わせた**アンサンブル**が一般的でした。

**外れ値（`Other`クラス）検出**は、評価指標であるバランス精度を向上させる上で重要でした。分類モデルの出力確率の**エントロピー**や**最大確率値**、あるいは**ArcFace**のような距離学習ベースの手法を用いて、予測の不確実性が高いサンプルや既知のサブタイプから距離が遠いサンプルを `Other` と判定する戦略が取られました。外部データセットから正常組織や良性腫瘍の画像を追加学習データとして利用するアプローチもありました。

**データ拡張**としては、標準的なフリップ、回転、スケール、色調変化などに加え、組織病理画像特有の**染色正規化 (Stain Normalization)** や染色拡張 (Stain Augmentation) も試されましたが、必ずしも効果的ではなかったようです。

**交差検証 (CV)** はデータセットの偏り（特定の施設やサブタイプの偏在）のため難しく、Public/Private LBとの相関が低い場合もありました。層化K分割（Stratified KFold）などが用いられましたが、信頼性の高いCV戦略の構築が課題でした。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/UBC-OCEAN/discussion/466455)**

* **アプローチ:** 病理組織用Foundation Model「Phikon」でパッチ特徴量を抽出し、MILモデル「Chowder」のアンサンブルで分類。予測エントロピーで外れ値を検出。
* **アーキテクチャ:** Backbone: Phikon (ViT-Base)。MIL: Chowder。
* **アルゴリズム:** Cross-Entropy Loss。AdamW。Weighted Sampling（クラス均衡化）。
* **テクニック:**
    * **前処理:** Otsuの二値化（HSV空間）で組織検出。TMAはパッチサイズを倍（448px）にし、224pxにリサイズしてWSI（224pxパッチ）とスケールを合わせる。WSIは200パッチに制限。C言語+libpngで高速タイリング。Rayで並列処理。
    * **特徴抽出:** Phikonを使用。他のバックボーン（CTransPath, LUNIT, DinoV2）より高性能。
    * **分類:** Chowderモデルを異なる初期値で50個学習しアンサンブル（安定性と効率性向上）。特定のFold/Seedのモデルを選択して最終アンサンブル（65モデル）。ロジスティック回帰で予測確率をキャリブレーション後、性能に基づいてモデルを選択。
    * **外れ値検出:** 予測確率のエントロピーに閾値を設け、高エントロピーのサンプルを`Other`と分類。閾値はPublic LBで調整。
    * **その他:** PhikonをiBOTで追加学習（Fine-tuning）したモデルや、予測の分散を用いた外れ値検出も有効だったが最終提出には含まず。染色正規化は効果なし。

**[2位](https://www.kaggle.com/competitions/UBC-OCEAN/discussion/465410)**

* **アプローチ:** パッチ特徴量抽出 + MILモデル (ABMIL, DSMIL, TransMIL) のアンサンブル。外部データを利用。
* **アーキテクチャ:** Backbone: Lunit-DINO (ViT-Small, patch16 & patch8)。MIL: ABMIL, DSMIL, TransMIL。
* **アルゴリズム:** Cross-Entropy Loss?
* **テクニック:**
    * **データ:** TCIA (PTRC-HGSOC) およびポーランドの公開データセットを利用。外部データの効果は限定的と評価。
    * **前処理:** PyVipsで画像読み込み。TMAはそのまま、WSIはサムネイルで組織検出し、パッチサイズ分の領域を切り出し。WSIからはサイズに応じてサンプリング率を変更してパッチを抽出（最大80%〜最小50%）。
    * **特徴抽出:** Lunit-DINOのViT-S/16とViT-S/8を使用。
    * **分類:** ABMIL, DSMIL, TransMILの3種類のMILモデルを学習。
    * **アンサンブル:** 複数のバックボーンとMILモデルの組み合わせでアンサンブル。

**[3位](https://www.kaggle.com/competitions/UBC-OCEAN/discussion/465527)**

* **アプローチ:** 外部データと、提供されたマスク情報から生成した合成TMA画像を活用。Lunit-DINO特徴量 + CLAMモデル。
* **アーキテクチャ:** Backbone: Lunit-DINO。MIL: CLAM (Attentionベース)。
* **アルゴリズム:** CLAM Loss (Bag Loss + Instance Loss)。
* **テクニック:**
    * **データ:** TCIA (Bevacizumab Response, CPTAC-OV) やHamarneh Lab、Stanford TMA Databaseなどの外部データを収集。ラベルマッピングに工夫（例: Papillary Serousは学習済みモデルでHGSC/LGSCに再分類）。提供マスクから腫瘍領域をクロップし合成TMA画像を生成。正常/間質領域から`Other`の合成画像も生成。
    * **前処理:** PyVipsで組織領域をタイリング。16bitで特徴量抽出し高速化。
    * **特徴抽出:** Lunit-DINO (ViT-S/16?)。
    * **分類:** CLAMモデルを使用。Instance Loss部分を`Other`クラスを考慮して変更（`Other`クラスやTMAの場合は負例に対する損失を考慮しないなど）。
    * **検証:** 外部データを除いたデータで5-Fold CV。

**[4位](https://www.kaggle.com/competitions/UBC-OCEAN/discussion/465811)**

* **アプローチ:** セグメンテーションモデルでタイル選択 + 分類モデル。WSIとTMAで異なる処理。
* **アーキテクチャ:**
    * タイル選択 (WSI): セグメンテーションモデル（U-Net系?）。
    * 分類: ConvNeXt, Hornet, EfficientNetV1/V2。
* **アルゴリズム:** 分類: Sigmoid出力 + Binary Cross Entropy Loss。
* **テクニック:**
    * **TMA処理:** 中央クロップ（例: 3000->2500）後、768x768にリサイズ。
    * **WSI処理:** サムネイル画像で学習したセグメンテーションモデルでタイル選択。最も癌らしい確率が高いピクセル位置周辺の1536x1536領域をクロップし、768x768にリサイズして分類モデルに入力。
    * **分類:** 5つのサブタイプ + 非がん (`Other`相当) の6クラス分類として学習（マスクデータ利用）。Sigmoid活性化関数を使用。
    * **Augmentation:** Stain Augmentation、幾何学的変換、色調変化、Channel Shuffleなど。
    * **アンサンブル:** 16個の異なるバックボーンの分類モデル予測を中央値でアンサンブル。
    * **外れ値検出:** 予測確率の最大値が低い (< 0.05) または予測確率が下位5-10%のものを`Other`と分類。
    * **その他:** 問題のあるサムネイル画像（複数のスライスが並んでいる等）を検出し、WSIから再生成。

**[5位](https://www.kaggle.com/competitions/UBC-OCEAN/discussion/466017)**

* **アプローチ:** WSIに対してはタイル選択モデル（セグメンテーションベース）+ 分類モデル、TMAは中央クロップ+分類モデル。外部データを`Other`クラスとして利用。データマイニング（疑似ラベル）実施。
* **アーキテクチャ:**
    * タイル分類補助: ConvNeXt-base。
    * 腫瘍セグメンテーション補助: SEResNeXt101 U-Net。
    * タイル選択（推論用）: SEResNeXt101 U-Net。
    * 分類（推論用）: ConvNeXt-base/large, EVA。
* **アルゴリズム:** 分類: Cross Entropy Loss? データマイニング使用。
* **テクニック:**
    * **WSIタイル選択:** 2段階の補助モデルを経て、最終的なタイル選択用セグメンテーションモデルを学習。1) 全前景タイルで学習した分類モデル予測と、2) マスクデータで学習した腫瘍セグメンテーションモデル予測を組み合わせてヒートマップを作成し、それを教師として最終セグメンテーションモデルを学習。推論時はこのヒートマップの上位5タイルを選択。
    * **TMA処理:** 中央クロップ (3072x3072) 後、1536x1536にリサイズ。
    * **分類モデル学習:** 外部データ (Hubmap, Camelyonなど) を`Other`クラスとして追加。データマイニング（Round 1予測に基づき、低確信度タイルを`Other`疑似ラベル、高確信度タイルをWSIラベル疑似ラベルとしてRound 2学習）。
    * **Augmentation:** RandAugment, StainNormなど。
    * **アンサンブル:** ConvNeXt-base, ConvNeXt-large, EVAモデルのアンサンブル。
    * **TTA:** StainNorm TTA。

**[6位](https://www.kaggle.com/competitions/UBC-OCEAN/discussion/465379)**

* **アプローチ:** iBOT-ViT-Base特徴量 + MILモデル (Chowder)。予測エントロピーによる外れ値検出。
* **アーキテクチャ:** Backbone: iBOT-ViT-Base。MIL: Chowder。
* **アルゴリズム:** Cross Entropy Loss?
* **テクニック:**
    * **前処理:** WSI/TMAからランダムに1000パッチをサンプリング（不足時はコピー）。
    * **特徴抽出:** OwkinのiBOT-ViT-Baseを使用。
    * **分類:** Chowderモデルを使用。
    * **アンサンブル:** 7つの異なる学習済みChowderモデルをアンサンブル。
    * **外れ値検出:** アンサンブル予測確率の平均エントロピーを計算し、閾値で`Other`を判定。
    * **その他:** パッチ選択法の重要性を指摘（ただし本解法ではランダム選択）。

**[7位](https://www.kaggle.com/competitions/UBC-OCEAN/discussion/465697)**

* **アプローチ:** CTransPath/LunitDINO特徴量 + MILモデル (DSMIL, Perceiver) のアンサンブル。BCEベースの外れ値検出。
* **アーキテクチャ:** Backbone: CTransPath, LunitDINO (ViT-S/16?)。MIL: DSMIL, Perceiver。
* **アルゴリズム:** DSMIL: CrossEntropyLoss。Perceiver: BCEWithLogitsLoss (+ Mixup, Label Smoothing)。
* **テクニック:**
    * **前処理:** PyVips使用。WSI/TMAを10x相当にダウンサンプリング。WSIは重複領域を除去。非重複Sliding Windowで256x256パッチ生成。
    * **特徴抽出:** CTransPathとLunitDINOを使用。
    * **分類:** DSMILとPerceiverを使用。
    * **外れ値検出:** BCEWithLogitsLossで学習し、最大確率値が閾値 (0.4) 未満の場合、または予測確率のエントロピーが閾値より大きい場合に`Other`と判定する2つの方法を試し、両方ともPrivate 0.6を達成。
    * **アンサンブル:** 4つの組み合わせ（2 Backbone x 2 MIL）のモデルをアンサンブル。

**[8位](https://www.kaggle.com/competitions/UBC-OCEAN/discussion/465382)**

* **アプローチ:** WSIとTMAを完全に別のアプローチで処理。TMAはArcFace + 分類器アンサンブル、WSIはTMA学習済み特徴抽出器 + DTFD-MIL。合成`Other`データ。
* **アーキテクチャ:**
    * TMA分類: EfficientNetV2-s/l, ConvNeXt-small/large。ArcFaceヘッドも使用。
    * WSI特徴抽出: TMA学習済み EffNetV2-s, ConvNeXt-small。
    * WSI分類: DTFD-MIL。
* **アルゴリズム:** TMA分類: Cross Entropy Loss? ArcFace Loss? WSI分類: DTFD-MIL Loss?
* **テクニック:**
    * **スケール統一:** TMAを2倍ダウンサンプリングし、WSIと物理スケールを合わせる。
    * **TMA処理:** マスクを利用してパッチをタイリング。正常/壊死パッチを`Other`として学習。ArcFaceで既知TMAとの類似度を計算し、近いものはラベルを割り当て、遠いものは外れ値候補に。残りを分類器アンサンブル（6モデル）で予測。
    * **WSI処理:** TMAパイプラインで学習したモデルを特徴抽出器として使用。抽出した特徴量でDTFD-MILを学習。学習中に、高`Other`確率パッチプールから動的に`Other` WSIを合成して追加。異なる解像度（768, 1024）のモデルをアンサンブル。
    * **高速化:** WSI処理を高速化するため、ピクセル数でソートしマルチスレッド処理。推論時はタイルの中心領域のみ使用。

**[9位](https://www.kaggle.com/competitions/UBC-OCEAN/discussion/465815)**

* **アプローチ:** WSIとTMAで異なるスケール処理。シンプルなWSIタイル選択（bg_countに基づく）と分類。補助ラベルを用いた2段階学習。
* **アーキテクチャ:** EfficientNet-b4, EfficientNet-v2s, MaxViT-tiny。
* **アルゴリズム:** Binary Cross Entropy (BCE)。補助ラベル損失を追加。
* **テクニック:**
    * **WSI/TMA分割:** 画像サイズと黒ピクセル比率で判定。
    * **WSI前処理:** 0.33倍にリサイズ後、512x512タイルに分割。タイルの背景ピクセル比率 (`bg_count/area`) に基づき3段階でタイルを選択（低背景比率を優先しつつ、最低枚数を確保）。
    * **TMA前処理:** TMAをクロップ（低標準偏差の境界を除去）後、512x512にリサイズ。
    * **学習:** WSIタイルのみ使用。バッチごとに画像から6タイルをランダム選択。2段階学習：Step 1で通常学習→Step 2でStep 1予測を補助ラベル（高確信度予測を使用）として追加損失に加え、より低いLRで再学習。
    * **推論 (WSI):** 学習済みモデルで全タイル予測。タイル予測値の最大値を持つクラスをWSI予測とする？（`idxmax()`を使用）。
    * **推論 (TMA):** WSI用モデルをそのまま適用。
    * **外れ値検出:** 補助ラベルの平均予測値が閾値未満の場合に`Other`と判定。
    * **アンサンブル:** 3つのバックボーンのモデル予測をVoting。

**[10位](https://www.kaggle.com/competitions/UBC-OCEAN/discussion/465455)**

* **アプローチ:** 外部TMAデータセットを多用。シンプルなEfficientNetV2-small + 分類ヘッド。セグメンテーションモデルによるWSIクロップ選択。
* **アーキテクチャ:** Backbone: EfficientNetV2-small。セグメンテーション: MaxViT-Tiny FPN。
* **アルゴリズム:** CrossEntropyLoss (クラス重み付き)。AdamW + Cosine Annealing LR。
* **テクニック:**
    * **データ:** 多数の外部TMAデータセット（Stanford TMA DB, kztymsrjx9, tissuearray.com, usbiolab.com, proteinatlas.org）を追加。UBCコンペのPublicテストのHGSC確信度が高い画像も追加。
    * **WSI前処理:** サムネイルで学習したセグメンテーションモデルでマスク予測。マスク領域に基づいて上位16個の1024x1024クロップを抽出。
    * **TMA前処理:** 低標準偏差の行/列を除去して白い背景を削除。
    * **学習:** WSIクロップとTMAを混合学習。クラス重み（`n / n_i`）を使用。AMP使用。15 epochs。
    * **Augmentation:** TMAは1024にリサイズ。倍率正規化（WSIを512に縮小後1024に戻す）、幾何学的変換、Color Jitter, Channel Shuffle, Blur, Coarse Dropout。
    * **推論:** 5 Foldモデルの平均。3x TTA（フリップ）。WSIは16クロップの予測を平均。
    * **その他:** 初期のOOFスコアは高かった(0.867)がLBスコアが低かった(0.47)ため、外部データを追加する方向に転換。LBスコアを信頼し、最もLBスコアが高かった提出を選択。
