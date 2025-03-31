---
tags:
  - Kaggle
startdate: 2023-08-12
enddate: 2023-06-13
---
# Image Matching Challenge 2023
https://www.kaggle.com/competitions/image-matching-challenge-2023

**概要 (Overview)**

- **目的:** このコンペティションの目的は、異なる視点から撮影された**複数の画像群**が与えられたときに、それらの画像間の**幾何学的な対応関係（相対的なカメラ位置・姿勢）を高精度に推定する**アルゴリズム（コンピュータビジョンモデル）を開発することです。具体的には、画像ペア間の**基礎行列 (Fundamental Matrix)** を推定します。
- **背景:** これはコンピュータビジョンの基本的な問題であり、「Structure from Motion (SfM)」や3D再構成として知られています。撮影された複数の2次元画像から、シーンの3次元構造や各画像のカメラパラメータ（位置・姿勢）を復元する技術です。拡張現実（AR）、ロボットナビゲーション、文化遺産のデジタル化、ドローンマッピングなど、多くの応用分野で不可欠な技術となっています。このチャレンジは、コンピュータビジョン分野の主要な学会（例: CVPR）で開催されるワークショップの一部として行われることが多く、最新技術の性能を競います。
- **課題:** 同じシーンを撮影した画像間でも、視点、照明条件、スケール（拡大率）、遮蔽（オクルージョン）などが大きく異なる場合があります。このような変動がある中で、画像間で対応する点（キーポイント）を正確に見つけ出し（マッチング）、その対応点に基づいてロバスト（頑健）に基礎行列を推定することが求められます。特に、誤った対応点（アウトライア）の影響を除去する技術（例: RANSAC）が重要になります。データセットには、複雑な実世界のシーンが含まれることが多く、高い頑健性が要求されます。

**データセットの形式 (Dataset Format)**

提供される主なデータは、複数のシーン（データセット）と、各シーンに属する画像群、そして基礎行列を推定すべき画像ペアのリストです。

1. **画像データ:**
    
    - データは複数の**シーン (dataset)** に分かれています（例: `church`, `dioscuri`, `kyiv-lavra` など）。
    - 各シーンのフォルダ (`train/{dataset}/images/` や `test/{dataset}/images/`) には、そのシーンを様々な角度から撮影した複数の画像ファイル（例: JPG形式）が含まれます。
2. **推定対象ペアリスト (`test.csv` など):**
    
    - どの画像ペアについて基礎行列を推定・提出する必要があるかを定義するCSVファイル。
    - 通常、以下の列を含みます:
        - `sample_id`: 提出時に使用する、各画像ペアの一意な識別子。
        - `dataset`: 画像ペアが属するシーン（データセット）名。
        - `image1_id`, `image2_id`: ペアを構成する2つの画像のファイル名（またはID）。
3. **キャリブレーション情報 (任意):**
    
    - いくつかのデータセットでは、カメラの内部パラメータ（焦点距離、主点など）が提供される場合がありますが、多くの場合、参加者が推定するか、一般的な値を使用する必要があります。
4. **トレーニングデータについて:**
    
    - このコンペティションでは、典型的な教師あり学習のように「トレーニング用の正解基礎行列」が直接提供されるわけではありません。参加者は、公開されている画像マッチングやロバスト推定に関する既存のアルゴリズム（SIFT, SuperPoint, LoFTR, RANSAC, MAGSAC++ など）を利用・改良したり、独自の手法を開発したりします。トレーニングは、これらのアルゴリズム自体の学習や、パイプライン全体のパラメータ調整を意味することが多いです。
5. **`sample_submission.csv`**:
    
    - 提出フォーマットのサンプル。`sample_id` と `fundamental_matrix` の列を持ちます。`fundamental_matrix` 列には、推定された3x3の基礎行列を**平坦化（flatten）**し、9つの数値をスペース区切りで並べた文字列を格納します。

**評価指標 (Evaluation Metric)**

- **指標:** **Mean Average Accuracy (mAA)**
- **計算方法:**
    1. **各画像ペアの評価:** 提出された基礎行列ごとに、主催者側が持つ正解のカメラポーズ（相対的な回転と並進）と比較します。予測された基礎行列から導かれるエピポーラ幾何と、正解の幾何との間の最大角度誤差を計算します。この誤差が特定のしきい値（例: 10度）以下であれば、そのペアの推定は「正確 (accurate)」と判定されます。
    2. **各シーンの評価:** シーンごとに「Average Accuracy (AA)」を計算します。AA = (そのシーン内で正確に推定されたペア数) / (そのシーンの総ペア数)。
    3. **最終スコア:** 全てのシーンのAAの**平均値**を計算したものが mAA となります。mAA = (全シーンのAAの合計) / (シーン数)。
- **意味:** この指標は、モデルが推定した画像ペア間の相対的なカメラジオメトリ（基礎行列によって表現される）が、実際のカメラポーズとどれだけ整合しているかを評価します。各シーンでの精度を平均することで、特定の（簡単な、または難しい）シーンに対する性能の偏りをならし、全体的な幾何学的推定精度を測ります。スコアは**高い**ほど（最大1.0）、多様なシーンにおいて、より多くの画像ペアの幾何学的関係を正確に推定できていることを示します。

要約すると、このコンペティションは、複数の画像からカメラの相対ポーズを推定する fundamental matrix 推定タスクです。データはシーンごとの画像群と評価対象ペアリストで構成され、性能は実際のカメラポーズとの幾何学的な整合性を測る Mean Average Accuracy (mAA、高いほど良い) によって評価されます。

---


**全体的な傾向**

上位解法では、局所特徴量マッチング（Local Feature Matching）とStructure from Motion (SfM) を組み合わせたパイプラインが主流でした。特徴量抽出とマッチングには、SuperPoint/SuperGlue (SPSG) のような学習ベースのSparse（疎）特徴量と、LoFTRやDKMのような学習ベースのDense（密）特徴量を組み合わせる、あるいはSIFTのような古典的特徴量と学習ベース特徴量を組み合わせるハイブリッドアプローチが多く見られました。KeyNetAffNetHardNetのような古典的手法に基づく検出器+学習ベース記述子の組み合わせや、新しいマッチャーであるLightGlueも利用されました。

SfM処理には主にCOLMAPが用いられましたが、その前段でのペア選択（Image Retrievalや軽量モデルでのスクリーニング、kNN shortlist）による効率化と精度向上が重要な戦略でした。また、画像間の回転ずれへの対応（複数角度でのマッチング、回転補正モデルの利用）は、多くの解法で必須のテクニックとされました。さらに、マッチング精度向上のためにマルチスケールTTAや画像分割マッチングなども用いられました。COLMAPの実行においては、初期画像選択の工夫、複数回再構築による最良結果の選択、PixSfMなどを用いた後段のリファインメント、並列処理による高速化、シングルスレッド実行による再現性確保といったテクニックが見られました。

**各解法の詳細**

**1位**

- **アプローチ:** Sparse (SPSG) + Dense (LoFTR/DKM) マッチング結果を信頼度に基づいてマージし、Coarse-to-fine SfMフレームワークで精度を向上。
- **アーキテクチャ/モデル:** Sparse: SuperPoint/SuperGlue (SPSG)。 Dense: LoFTR, DKMv3。 Retrieval: NetVLAD (推定)。 SfM: COLMAP。 Refinement: TransformerベースMulti-view Matching (独自)。
- **アルゴリズム:** Confidence-guided Merge (NMS)、Iterative Refinement (Multi-view Matching + Geometry Refinement: Geometric-BA + Track Topology Adjustment)。
- **テクニック:**
    - **ペア選択:** 画像検索 (NetVLAD) でkペア選択。
    - **マッチング:** 回転検出 (4角度マッチング)、Overlap Detection (SPSGで領域推定→Denseマッチング)。SPSG + LoFTR/DKMのアンサンブル。
    - **Coarse SfM:** マッチング結果を信頼度ベースでNMSマージ (上位10k点) → COLMAPで再構築 (Geometry Verificationスキップ、PBAは途中から有効化)。
    - **Iterative Refinement:** 独自AttentionベースMulti-view Matchingで特徴トラックを精密化し、COLMAPでジオメトリを再最適化、を繰り返す。
    - **安定化:** RANSACシード固定、COLMAPシングルスレッド実行で決定性を確保。

**2位**

- **アプローチ:** SuperPoint/SuperGlue (SPSG) を軸に、マルチスケールTTA、回転補正、COLMAPの複数回実行による安定化・精度向上策を多用。
- **アーキテクチャ/モデル:** SuperPoint/SuperGlue (SPSG)。 Rotation Detector (check_orientation)。 SfM: COLMAP。
- **アルゴリズム:** TTA (マルチスケール)、Exhaustive Matching複数回実行＋フィルタリング、複数回再構築（最良選択）。
- **テクニック:**
    - **ペア選択:** 全ペアを使用し、マッチ数が閾値(100)未満のペアを棄却。
    - **マッチング:** SPSG (kp閾値0.005, match閾値0.2, sinkhorn 20回)。ハーフプレシジョン(FP16)利用。キーポイント/ディスクリプタ/マッチのキャッシュ。
    - **TTA:** 複数画像サイズ([1088, 1280, 1376])でマッチングし結果を結合。
    - **回転補正:** check_orientationモデルで画像の向きを補正。
    - **COLMAP:** 初期画像を明示的に選択（ペア数・マッチ数最大）。Exhaustive Matchingを複数回(10回)実行し、8回以上出現したマッチのみ使用。シーン画像数に応じて複数回(4回、異なるマッチ数閾値)再構築を行い、登録画像数・3D点数で最良を選択。

**3位**

- **アプローチ:** SuperPoint/SuperGlue (SPSG) を中心に、軽量SPSGによる初期スクリーニング、画像分割マッチング、DKMv3とのアンサンブルで精度と効率を両立。
- **アーキテクチャ/モデル:** SuperPoint/SuperGlue (SPSG)。 DKM v3。 SfM: COLMAP。
- **アルゴリズム:** 画像分割マッチング、並列処理。
- **テクニック:**
    - **ペア選択/スクリーニング:** 軽量SPSG (longside=1200) で全ペアをマッチングし、マッチ数が閾値(30)未満のペアを除外。
    - **回転補正:** スクリーニング時に回転処理を組み込む。
    - **マッチング:** 画像を4分割し、全組み合わせ(16ペア)でSPSG (longside=1400) マッチングを実行。SPSGとDKMv3の結果をアンサンブル。
    - **効率化:** マッチング処理とCOLMAPマッピング処理を別プロセス/スレッドで並列実行（キュー利用）。COLMAPはシングルスレッド実行。

**4位**

- **アプローチ:** 軽量SPSGを用いたkNN Shortlist作成と補完による効率的なペア選択。回転補正と並列処理も実施。
- **アーキテクチャ/モデル:** SuperPoint/SuperGlue (SPSG)。 MatchFormer。 SfM: COLMAP。
- **アルゴリズム:** kNN Shortlist + 補完、DBSCAN（クロップアンサンブル用）。
- **テクニック:**
    - **ペア選択:** 軽量SPSG (kp=512) で全ペアをマッチングし、Inlier数に基づいて各画像のk最近傍ペアを選択。さらに、どのペアにも選ばれなかった画像に対し、マッチ数ベースで最低kペアを補完的に追加。
    - **回転補正:** 軽量SPSGで4角度のマッチングを行い補正。全画像を同サイズ(840x840)にリサイズしバッチ処理。
    - **マッチング:** SPSGとMatchFormerを使用。2022年解法同様のクロップ&マルチスケールアンサンブル（DBSCAN利用）。
    - **効率化:** マッチングとCOLMAPバンドル調整を別スレッドで並列実行。

**5位**

- **アプローチ:** 古典的なKeyNetAffNetHardNet8検出器とAdaLAMマッチャーを軸に、複数検出器のアンサンブルとCOLMAP連携の工夫で精度向上。
- **アーキテクチャ/モデル:** Keypoint Detector: KeyNetAffNetHardNet8系 (GFTTAffNetHardNetなど4種)。 Matcher: AdaLAM。 SfM: COLMAP。 Geometry Verification: USAC_MAGSAC (OpenCV)。
- **アルゴリズム:** AdaLAMマッチング、USAC_MAGSAC。
- **テクニック:**
    - **特徴抽出/マッチング:** 4種類のKeyNetAffNetHardNet系検出器で特徴点(max 8k)を抽出（最大辺1600）。全ペアをAdaLAMでマッチング（平均距離<0.5のペア保持、force_seed_mnn=True, ransac_iters=256）。OriNet有効化（Upright無効化）で回転頑健性向上。
    - **アンサンブル:** 4つの検出器からのマッチを結合。
    - **SfM連携:** USAC_MAGSACでペアごとに基本行列を計算しInlierのみ抽出。結果をtwo-view geometryとしてDBに書き込み、COLMAPの`match_exhaustive`をバイパスして`incremental_mapping`を実行。
    - **焦点距離:** EXIFから焦点距離を抽出し、`prior_focal_length=True`でCOLMAPに設定。
    - **効率化:** マルチプロセッシング利用（複数検出器の並列処理、マッチングと再構築の並列化）。シーンを画像数でソートし処理。

**7位**

- **アプローチ:** 新しいマッチャーLightGlueを軸に、多様な特徴抽出器（ALIKED, DISK, SIFT）と組み合わせたアンサンブル。PixSfMによるリファインメント。
- **アーキテクチャ/モデル:** Feature Extractor: ALIKED, DISK, SIFT (OpenCV), SuperPoint。 Matcher: LightGlue, NN-ratio (SIFT用)。 Retrieval: NetVLAD。 SfM: COLMAP + PixSfM。
- **アルゴリズム:** LightGlueマッチング、NetVLAD検索、PixSfMリファインメント。
- **テクニック:**
    - **ペア選択:** NetVLADで上位kペア(k=20, 30, 50)を選択。
    - **特徴抽出/マッチング:** ALIKED(kp=2k), DISK(kp=5k), SIFTをLightGlue (ALIKED, DISK用) またはNN-ratio (SIFT用) でマッチング。SuperPointも比較対象。
    - **アンサンブル:** 複数特徴抽出器＋マッチャーの組み合わせによるマッチ結果を結合。特にSIFTは回転対応と高速性から有効。
    - **SfM/リファインメント:** COLMAPで再構築後、PixSfMでリファインメント（回転画像がないシーンのみ）。一部シーンで共有カメラパラメータを強制。
    - **未登録画像のローカライズ:** hlocツールボックスを用いて後処理的に実行。

**9位**

- **アプローチ:** COLMAPベースラインに回転補正と改善されたペア選択（NetVLAD+Re-ranking, ペア数決定アルゴリズム）を導入。SPSG+MAGSACを使用。
- **アーキテクチャ/モデル:** Orientation Model (check_orientationベース)。 Retrieval: NetVLAD。 Feature Extractor/Matcher: SuperPoint/SuperGlue。 Geometry Verification: MAGSAC。 SfM: COLMAP。
- **アルゴリズム:** NetVLAD + Re-ranking、ペア数決定アルゴリズム（完全グラフ＋連結性チェック＋二分探索）。
- **テクニック:**
    - **回転補正:** check_orientationモデルに変更を加え、複数回転画像での予測と閾値処理によるSelf-Ensemblingで精度向上。
    - **ペア選択 (Retrieval):** NetVLADを使用。水平反転画像を加算したディスクリプタ、TTA（画像サイズ）、Re-ranking（Top1ペアで再クエリ）で性能向上。
    - **ペア選択 (数決定):** 完全グラフ上で各画像が同程度のペア数を持つように、連結性を保ちつつ二分探索でペア数を決定（小シーンでは全ペア）。
    - **マッチング/SfM:** SPSG + MAGSACフィルタリング。
    - **安定化:** COLMAPシングルスレッド実行で決定性確保。
    - **その他:** GlueStickも有望だったが高スコアには至らず。PixSfMは効果あったが最終提出には含めず。OpenCV/PILの自動回転挙動に注意。

