---
tags:
  - Kaggle
  - mAA
  - 画像マッチング
startdate: 2024-03-26
enddate: 2024-06-04
---
# Image Matching Challenge 2024 - Hexathlon
[https://www.kaggle.com/competitions/image-matching-challenge-2024](https://www.kaggle.com/competitions/image-matching-challenge-2024)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、6つの異なるカテゴリ（Hexathlon）に分類される多様な画像セットに対して、汎用的かつ高精度な画像マッチングと3D再構成（カメラ姿勢推定）のアルゴリズムを開発することです。特に、従来のマッチング手法が苦手とする困難なシナリオ（例: 透明物体、繰り返し構造、大規模な時間変化、空中写真）への対応が求められます。
* **背景:** Image Matching Challenge (IMC) シリーズの一環として、コンピュータビジョン分野における画像マッチングとStructure from Motion (SfM) 技術の限界を押し広げることを目指しています。現実世界の多様なデータに対するアルゴリズムの頑健性と精度を評価します。
* **課題:** カテゴリごとに異なる特性を持つ画像セットに対し、単一または適応的なアルゴリズムで高精度なカメラの外部パラメータ（回転と並進）を推定することです。特に、テクスチャが乏しく反射が多い「透明物体」カテゴリや、視点変化が大きい「空中写真」カテゴリ、時間経過による変化が大きい「時間変化」カテゴリなどが挑戦的な課題となります。限られた計算リソース（GPU T4x2, 16GB RAM, 2 CPUコア）と実行時間（9時間）内で処理を完了させる必要もあります。

**データセットの形式 (Dataset Format)**

提供される主なデータは、6つのカテゴリに分類された複数のシーンの画像群です。

1.  **トレーニングデータ & テストデータ:**
    * `train/` および `test/` ディレクトリ以下に、シーンごとにサブディレクトリが配置されます。
    * 各シーンディレクトリには、そのシーンを構成する複数枚の画像（`.jpg`など）が含まれます。
    * `categories.csv`: 各テストシーンが属するカテゴリ（例: `aerial`, `heritage`, `historical_preservation`, `symmetries_and_repeats`, `temporal`, `transparent_objects`）を示すファイル。
    * **アノテーション:** IMC2023と同様、真のカメラパラメータは公開されず、提出されたパラメータと隠された真値との比較によって評価が行われます。
2.  **`sample_submission.csv`**:
    * 提出フォーマットのサンプル。`image_path`, `dataset`, `scene`, `rotation_matrix` (9次元ベクトル), `translation_vector` (3次元ベクトル) の列を持ちます。

**評価指標 (Evaluation Metric)**

* **指標:** **Mean Average Accuracy (mAA)**
* **計算方法:** IMC2023と同様です。
    1.  提出されたカメラパラメータと真のパラメータとの間で、回転誤差と並進誤差を計算。
    2.  複数の誤差閾値ペアを設定（例: (1°, 0.2m), (2°, 0.5m), ..., (10°, 5m)）。
    3.  各閾値ペアに対して、誤差が閾値内に収まっている画像の割合（Accuracy）を計算。
    4.  全閾値ペアにおけるAccuracyの平均値 (Average Accuracy - AA) を計算。
    5.  最終スコア (mAA) は、全テストシーンにおけるAAの平均値。
* **意味:** 推定されたカメラの位置と姿勢の全体的な精度を評価します。スコアは**高い**ほど、より正確な推定ができていると評価されます。

要約すると、このコンペティションは、6つの異なるカテゴリの画像セットに対して頑健かつ高精度なカメラ姿勢推定を行うタスクです。データはシーンごとの画像群で構成され、性能は推定されたカメラパラメータと真値との間の精度（mAA、高いほど良い）によって評価されます。カテゴリに応じた戦略の切り替えや、リソース制限内での効率的な処理が求められます。

---

**全体的な傾向**

IMC2024は、6つの異なるカテゴリ（Hexathlon）に対応する必要があり、特に「透明物体」カテゴリが大きな挑戦となりました。上位解法の多くは、シーンのカテゴリを判定し、**カテゴリに応じた戦略を切り替える**アプローチを採用しました。

* **基本パイプライン (非透明シーン):**
    * **画像ペア選択:** 画像検索 (NetVLAD, カスタムグローバル特徴量など) や軽量マッチングによるペア候補絞り込み、あるいは全ペアマッチング。
    * **特徴抽出 & マッチング:** **ALIKED + LightGlue** の組み合わせが、性能とライセンス（Apache）の両面から非常に有力で、多くのチームが採用しました。DISK, SIFT, DeDoDe v2 などもアンサンブルに使用されました。SuperPoint+SuperGlue/LightGlueも依然として高性能ですが、ライセンスが考慮されました。複数手法のアンサンブルが一般的。
    * **幾何検証 & SfM:** RANSAC (MAGSACなど) + **COLMAP / pycolmap** が標準。
* **透明物体シーンへの対策:**
    * **仮定と戦略:** カメラが対象物の周りを円状に撮影したと仮定し、画像間の類似度から撮影順序を推定し、円周上にカメラを配置する戦略が主流。
    * **順序推定:** 画像間の類似度指標（ピクセル差分, SSIM, 光学フロー, グローバル特徴量類似度, 軽量マッチングのマッチ数など）を用いて距離行列を作成し、TSP (巡回セールスマン問題) アルゴリズムで最適な順序を決定。あるいはkNN的な手法で隣接画像を探索。
    * **前景セグメンテーション:** 背景のノイズを除去するため、セグメンテーションモデル (MobileSAM, DINOv2 Segmenter, TokenCutなど) で前景（透明物体）を抽出し、その領域内でのみ特徴点抽出・マッチングを行う。
    * **高解像度処理:** 透明物体の微細な特徴を捉えるため、元画像に近い解像度で処理。
* **回転不変性:** 回転推定モデル (`check_orientation`) や複数角度でのマッチング、回転不変特徴量 (SIFT, AffNet) の利用が継続的に重要。
* **効率化:** 並列処理 (GPU/CPU, Multi-GPU)、キャッシュ、Utility Scriptによるパッケージ事前インストール。
* **SfMの改善:** COLMAPの複数回実行、初期ペア指定、カメラパラメータ共有、未登録画像の再登録/ローカライズ (HLoc, VGGSfM)、Pixel-Perfect SfM (PixSfM) や DFSfM によるリファインメント。
* **新しい技術:** VGGSfM (ニューラルSfM)、MST-Aided SfM (最小全域木を利用したCoarse-to-Fine SfM)。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/image-matching-challenge-2024/discussion/510084)**

* **アプローチ:** 非透明シーンはCOLMAPベースの3D再構成 (I3DR)、透明シーンは直接姿勢推定 (DIP) を行う2段階パイプライン。ALIKED+LightGlueを高解像度で使用。
* **アーキテクチャ:** ALIKED (n16), LightGlue, COLMAP, TSPソルバー, 光学フロー (RAFT), SSIM。
* **アルゴリズム:** DBSCAN (クロップ領域決定)、RANSAC (幾何検証)、TSP (透明シーン順序推定)。
* **テクニック:**
    * **I3DR (非透明):**
        * ペア選択: マッチ数による閾値処理 (ALIKED+LG: >=30, アンサンブル: >=100)。
        * マッチング: ALIKED+LGを高解像度 (1280, 2048) で実行。複数解像度/クロップ領域でのマッチング結果をアンサンブル。回転処理 (0/90/180/270度)。
        * 新クロップ法: 全ペアのマッチ情報を元に画像ごとに代表領域をDBSCANでクラスタリングしクロップ。
        * SfM: RANSACで幾何検証後COLMAP。低登録数シーンでは閾値緩和し再実行。複数モデルのマージ。
    * **DIP (透明):**
        * 仮定: 円周上撮影。
        * 順序推定: 複数手法 (光学フロー, ピクセル差分, SSIM, ALIKED+LGマッチ数) でペア類似度を計算し、距離行列を作成してTSPで最適順序を決定。最終的にはSSIM + "Matching Flow" (詳細不明) のアンサンブル。
    * **効率化:** キーポイント/ディスクリプタのキャッシュ。マルチGPU並列処理。

**[2位](https://www.kaggle.com/competitions/image-matching-challenge-2024/discussion/510499)**

* **アプローチ:** 非透明シーンはMST (最小全域木) を利用したCoarse-to-Fine SfM、透明シーンは撮影順序推定。IMC2023の7位解法をベースに拡張。
* **アーキテクチャ:** DeDoDe v2 + Dual Softmax, DISK + LightGlue, SIFT + NN (マッチング)。カスタムグローバル特徴量 (ALIKED+DINO)。COLMAP, PixSfM, HLoc。TSPソルバー。
* **アルゴリズム:** MST (最小全域木)、TSP。
* **テクニック:**
    * **前処理:** 回転検出、カテゴリ判定 (透明/非透明)、カメラパラメータ共有 (画像サイズ同一時)。
    * **グローバル特徴量:** ALIKED (点) と DINO (パッチ) を対応付け、クラスタリング+VLADで独自グローバル特徴量を生成。
    * **ペア選択:** 上記グローバル特徴量で検索。
    * **マッチング:** DeDoDe v2 + DS, DISK + LG, SIFT + NN のアンサンブル。
    * **MST-Aided SfM (非透明):**
        * Stage 1: 画像間の類似度グラフでMSTを計算し、最も簡潔なデータ関連付けのみでCOLMAPを実行 (粗モデル)。
        * Stage 2: 全てのデータ関連付けとStage1の粗モデルを初期値として幾何検証を行い、最終的なCOLMAPモデルを構築。
    * **後処理:** PixSfMによるリファインメント、HLocによる未登録画像の再ローカライズ。
    * **透明シーン:** グローバル特徴量の類似度グラフからTSPで撮影順序を推定し、円周上にカメラを配置。

**[3位](https://www.kaggle.com/competitions/image-matching-challenge-2024/discussion/510338)**

* **アプローチ:** ニューラルSfM手法「VGGSfM」を既存のpycolmapパイプラインに統合。複数段階でVGGSfMを利用。
* **アーキテクチャ:** VGGSfM (カメラ予測器、トラック予測器)、ALIKED+LightGlue, SP+LightGlue (ベースライン用)、NetVLAD/DINOv2 (ペア選択)、SuperPoint (クエリ点選択)、pycolmap。
* **アルゴリズム:** バンドル調整 (BA)、umeyamaアルゴリズム (姿勢合わせ)。
* **テクニック:**
    * **回転補正:** `check_orientation` モデルを使用し、予測スコア>0.5なら複数回転でマッチ数最大を選択。
    * **追加トラック:** 画像ごとに近傍Nフレームを検索。SuperPointでクエリ点を選択し、VGGSfMトラック予測器で対応点を推定。これをpycolmapに追加マッチとして入力。
    * **SfMトラックリファインメント:** pycolmap再構成から得られた3D点に対応する2Dトラックを抽出し、VGGSfMファイントラック予測器で精密化。更新された2D点で再度グローバルBA。
    * **未登録画像の再配置:** 未登録画像とその近傍の登録済み画像 (計6枚) でVGGSfMを実行。得られた相対姿勢を、umeyamaアルゴリズムで計算した変換行列を用いて元の座標系に合わせる。
    * **マッチング:** ALIKED+LG と SP+LG の結果を併用。
    * **透明シーン:** DBSCANでキーポイントから関心領域を抽出し、再度キーポイント検出。

**[4位](https://www.kaggle.com/competitions/image-matching-challenge-2024/discussion/510611)**

* **アプローチ:** ALIKED+LightGlueをベースに、透明シーンへの対応と効率化・頑健化策を導入。
* **アーキテクチャ:** ALIKED (n16), LightGlue, DINOv2 Segmenter, COLMAP。
* **アルゴリズム:** DBSCAN (透明シーンの前景抽出補助)。
* **テクニック:**
    * **非透明シーン:**
        * マッチング: ALIKED+LG。画像を90度ずつ回転させてキーポイントをキャッシュ・保持。ペア間で4方向のマッチングを試し、マッチ数最大の回転を採用。
        * SfM: マッチ数閾値 (100, 125) でCOLMAPを複数回実行し、最大モデルを選択。
    * **透明シーン:**
        * 前景セグメンテーション: DINOv2 Segmenter ("bottle"クラスを利用) で前景マスクを生成。
        * キーポイント抽出: 元解像度のまま、前景領域内のみでグリッド分割してALIKEDを実行。
        * マッチング: 対応するグリッド間でのみLightGlueを実行 (探索範囲削減)。
    * **ペア選択:** 全ペアマッチング (画像検索不使用)。
    * **全画像利用:** submission.csvにない画像も含む最大100枚の画像セットを再構成に使用。
    * **効率化:** マルチプロセス/マルチGPUによる並列処理。Utility Scriptによるパッケージ事前インストール。

**[5位](https://www.kaggle.com/competitions/image-matching-challenge-2024/discussion/510603)**

* **アプローチ:** シーンカテゴリごとにカスタマイズされたSfMパイプライン。透明物体にはオブジェクト検出とシーケンス推定。
* **アーキテクチャ:** グローバル特徴量 (EVA-CLIP+ConvNeXt+DINOv2)、ALIKED+LightGlue (ベース)、MobileSAM, YoloWorld, RAFT (透明物体用)、COLMAP。
* **アルゴリズム:** DBSCAN (クロップ領域決定、1位解法と同様)。TSPソルバー。
* **テクニック:**
    * **ペア選択:** カスタムグローバル特徴量 + カテゴリ別類似度閾値。
    * **マッチング (非透明):** ALIKED+LightGlue。回転補正 (`check_orientation`)。
    * **透明物体:**
        * 前景検出: MobileSAMでマスク候補生成→キーポイント(ALIKED/SP)密度が高い最小マスクを選択→BBox化。
        * ペア選択/順序推定: 全ペアでマッチングを行い、マッチ数に基づいて各画像のTop2ペアを選択。あるいは光学フロー (RAFT) やグローバル特徴量類似度から距離行列を作成しTSPで順序推定。
    * **回転補正:** `check_orientation`モデルを利用。
    * **効果がなかったこと:** SIFTアンサンブル、暗所画像補正 (CLAHE)、対称性対策 (Doppelgangers)。

**[6位](https://www.kaggle.com/competitions/image-matching-challenge-2024/discussion/511291)**

* **アプローチ:** IMC2023の1位解法「DFSfM (Detector-Free SfM)」をベースに、密マッチングをDKM/RoMaに変更。透明シーンは円周軌道仮定+順序回復。
* **アーキテクチャ:** SuperPoint+SuperGlue (スパース)、DKMv3/RoMa (ViT-B版, 密)、TokenCut (セグメンテーション)、NetVLAD (検索/順序回復)、COLMAP、DFSfMリファインメントモジュール。
* **アルゴリズム:** Confidence-guided Merge、反復リファインメント (DFSfM)、TSP (透明シーン順序回復)。
* **テクニック:**
    * **一般シーン (DFSfM):**
        * ペア選択: NetVLAD。
        * マッチング: 回転検出、重複領域検出の後、SPSG + (DKMv3 または 自前再学習RoMa) を実行。
        * Coarse SfM: マッチ結果を信頼度ベースでマージし、COLMAPを2回実行 (2回目はパラメータ緩和) しベストモデル選択。
        * Iterative Refinement: DFSfMのリファインメントモジュールを適用。
    * **透明シーン:**
        * カテゴリ判定: TokenCutで前景セグメンテーションし、全画像で前景領域が一定なら透明シーンと判定。
        * 順序回復: 画像類似度 (NetVLAD or 軽量マッチング) で距離行列を作成し、TSPで最適順序を決定。
        * 姿勢推定: 円周軌道上に推定された順序でカメラを配置。
    * **RoMa再学習:** 時間のかかるRoMaのバックボーンをViT-Bに変更し再学習、DKMv3並みの速度とより良い性能を達成。

**[8位](https://www.kaggle.com/competitions/image-matching-challenge-2024/discussion/509902)**

* **アプローチ:** ALIKED+LightGlueをベースとし、回転への対応を強化。COLMAP設定の工夫。
* **アーキテクチャ:** ALIKED, LightGlue, pycolmap。
* **アルゴリズム:** ホモグラフィによるアフィン変換 (回転補正)。
* **テクニック:**
    * **マッチング:**
        * 1st step: 画像1を固定し、画像2を0/90/180/270度回転させてALIKED+LGでマッチング。
        * 2nd step: 1st stepで得られたキーポイントを用い、ホモグラフィ行列でアフィン変換を行い、画像の向きを補正してから再度マッチング。
    * **SfM:** pycolmapの`incremental_mapping`を使用。カメラモデル "simple-radial" (2回) と "simple-pinhole" (1回) で実行し、最大のモデルを選択（ランダム性低減）。
    * **効率化:** マルチスレッド（キーポイント抽出用）とマルチプロセス（COLMAP用）。マルチGPU利用。

**[10位](https://www.kaggle.com/competitions/image-matching-challenge-2024/discussion/515089)**

* **アプローチ:** ALIKED+LightGlue と AffNet+HardNet+AdaLAM の2つのマッチングパイプラインのアンサンブル。透明シーンへの対応。
* **アーキテクチャ:** ALIKED, LightGlue, AffNet, HardNet8, AdaLAM, COLMAP。
* **アルゴリズム:** (特筆すべきものなし)
* **テクニック:**
    * **ペア選択:** 全ペアマッチング。
    * **マッチング:**
        * パイプライン1: ALIKED + LightGlue。
        * パイプライン2: AffNet + HardNet8 + AdaLAM (回転不変性を期待)。
    * **アンサンブル:** 上記2つのパイプラインのマッチング結果を融合。
    * **回転対策:** AffNetの使用。回転検出モデルも検討したが最終的に不使用か？
    * **透明シーン:** 画像を中央クロップしてからキーポイント抽出・マッチングを行うことで改善。
    * **効率化:** 並列処理。
    * **COLMAP:** ほぼデフォルト設定 (`min_model_size=3`)。
