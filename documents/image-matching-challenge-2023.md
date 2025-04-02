---
tags:
  - Kaggle
  - mAA
  - SPSG
startdate: 2023-08-12
enddate: 2023-06-13
---
# Image Matching Challenge 2023
[https://www.kaggle.com/competitions/image-matching-challenge-2023](https://www.kaggle.com/competitions/image-matching-challenge-2023)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、様々な条件下で撮影された複数枚の画像間で、正確かつ頑健な対応点（キーポイントマッチング）を見つけ出し、それらの情報を用いて画像間の幾何学的な関係性（相対的なカメラの位置・姿勢）を高精度に推定するアルゴリズムを開発することです。
* **背景:** 画像マッチングはコンピュータビジョンの基礎技術であり、3Dシーン再構成 (Structure from Motion - SfM)、拡張現実 (AR)、ロボットナビゲーション、画像検索など、多岐にわたるアプリケーションで不可欠です。しかし、視点変化、照明変化、オクルージョン（隠れ）、画像の回転やスケール変化、テクスチャの少ない領域などがあると、正確なマッチングは困難になります。
* **課題:** 与えられた画像セットに対して、できるだけ多くの正確な対応点を見つけ出し、それらを用いてカメラの外部パラメータ（回転と並進）を推定することです。推定されたカメラパラメータの精度が評価されます。時間制限と計算リソース制限（2CPUコア、1GPU）の中で、効率的かつ高精度なマッチングとSfMパイプラインを構築する必要があります。

**データセットの形式 (Dataset Format)**

提供される主なデータは、複数のシーンに属する画像群です。

1.  **トレーニングデータ & テストデータ:**
    * `train/` および `test/` ディレクトリ以下に、シーンごとにサブディレクトリが配置されます（例: `heritage/cyprus`, `haiper/bike`, `urban/kyiv-puppet-theater`）。
    * 各シーンディレクトリには、そのシーンを構成する複数枚の画像（`.jpg`など）が含まれます。画像枚数はシーンによって異なります（数十枚～数百枚）。
    * **アノテーション:** トレーニングデータセットに対しては、真のカメラパラメータ（内部・外部）が提供される場合がありますが、コンペティションの形式によっては、評価は提出されたカメラパラメータと隠された真値との比較によって行われます。
    * `sample_submission.csv`: 提出フォーマットのサンプル。通常、画像ファイル名と、推定されたカメラの回転（四元数または回転行列）および並進ベクトルが含まれます。

**評価指標 (Evaluation Metric)**

* **指標:** **Mean Average Accuracy (mAA)**
* **計算方法:**
    1.  参加者は、各テストシーン内の画像に対するカメラの外部パラメータ（回転 R と並進 T）を推定し提出します。
    2.  提出されたカメラパラメータと、運営側が保持する真のカメラパラメータとの間で、回転誤差と並進誤差を計算します。
    3.  複数の誤差閾値ペア（例: (回転誤差 1°, 並進誤差 0.2m), (2°, 0.5m), (3°, 1m), (5°, 2m), (10°, 5m)など）を設定します。
    4.  各閾値ペアに対して、回転誤差と並進誤差の両方が閾値内に収まっている画像の割合（Accuracy）を計算します。
    5.  全ての閾値ペアにおけるAccuracyの平均値を計算し、これをAverage Accuracy (AA) とします。
    6.  最終的なスコア (mAA) は、全てのテストシーンにおけるAAの平均値となります。
* **意味:** 推定されたカメラの位置と姿勢が、どれだけ正確であるかを評価します。複数の厳しい閾値から緩い閾値までの精度を平均するため、全体的に頑健で高精度な姿勢推定が求められます。スコアは**高い**ほど、より正確なカメラパラメータ推定ができていると評価されます。

要約すると、このコンペティションは、複数の画像セットから画像間の対応点を見つけ出し、それを用いてカメラの外部パラメータ（位置・姿勢）を高精度に推定するタスクです。データはシーンごとの画像群で構成され、性能は推定されたカメラパラメータと真値との間の精度（Mean Average Accuracy - mAA、高いほど良い）によって評価されます。限られた計算リソースと時間の中で効率的な処理パイプラインを構築することも重要です。

---

**全体的な傾向**

このコンペティションは、画像マッチングとStructure from Motion (SfM) の組み合わせが中心でした。上位解法は、効率的な画像ペア選択、頑健な特徴点抽出・マッチング、そして高精度なSfMパイプラインの構築に注力していました。

* **パイプライン:**
    1.  **画像ペア選択 (Shortlisting):** 全ペアのマッチングを避けるため、画像検索 (NetVLAD, CosPlace) や軽量なマッチング (低解像度SPSG) で関連性の高いペア候補を絞り込む。全ペアを使用する戦略もあった。
    2.  **特徴点抽出 & マッチング:**
        * **スパース:** SuperPoint+SuperGlue (SPSG) が依然として強力だが、ライセンスの制約から KeyNetAffNetHardNet, DISK, ALIKED, SIFT なども代替として広く用いられた。LightGlue マッチャーも登場。
        * **密 (Detector-Free):** LoFTR, DKM, MatchFormer なども使われたが、複数視点間での一貫性 (Repeatability) の低さが課題。
        * **アンサンブル:** 複数の特徴抽出・マッチング手法の結果を組み合わせることで頑健性と精度が向上。
    3.  **幾何検証:** RANSAC (USAC\_MAGSACなど) を用いて、マッチング結果から外れ値を除去し、基礎行列 (Fundamental Matrix) や基本行列 (Essential Matrix) を推定。COLMAP内部の幾何検証を利用、あるいは事前に実行して結果をDBに書き込む戦略も。
    4.  **Structure from Motion (SfM):** **COLMAP** がデファクトスタンダードとして広く利用された。Pycolmapも活用された。
* **回転不変性の確保:**
    * 多くの学習ベースマッチャー（特にSuperGlue）は回転に弱いため、画像を複数角度 (0, 90, 180, 270度) に回転させてマッチングを行う、あるいは回転推定モデル (`check_orientation`) で補正する、SIFTのような回転不変な特徴量を使うなどの対策が重要だった。
* **効率化と頑健性:**
    * **並列処理:** GPU処理（マッチング）とCPU処理（SfM）を並列化。
    * **キャッシュ:** 特徴量やマッチング結果のキャッシュ。
    * **COLMAPのランダム性対策:** バンドル調整 (BA) などのランダム性を避けるため、複数回実行して最良の結果を選択する、シングルスレッドで実行する、初期画像ペアを明示的に指定するなどの工夫が見られた。
    * **高解像度入力:** マッチング精度向上のため、入力画像を比較的高解像度 (例: 1280, 1600) で処理。
* **後処理/リファインメント:**
    * **Pixel-Perfect SfM (PixSfM):** より高精度な再構成のための後処理。
    * **未登録画像のローカライズ:** HLocなどのツールを用いて、SfMで登録されなかった画像を追加登録。
    * **信頼度ベースのマージ (1位):** 密マッチング結果などを信頼度に基づいてマージし、一貫性を向上させてからSfMに入力。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/image-matching-challenge-2023/discussion/417407)**

* **アプローチ:** スパース(SPSG)と密(LoFTR/DKM)マッチングを組み合わせ、密マッチングの複数視点間での不整合問題を解決する Coarse-to-Fine SfM フレームワーク。
* **アーキテクチャ:** SPSG, LoFTR, DKMv3, COLMAP, および独自開発の反復リファインメントモジュール (Attentionベース多視点マッチング + 幾何リファインメント)。
* **アルゴリズム:** 信頼度誘導マージ (Confidence-guided Merge)、反復リファインメント (Iterative Refinement)。
* **テクニック:**
    * **マッチング:** 回転検出 (0/90/180/270度)、重複領域検出 (SPSGで初回マッチング→領域推定→密マッチング)。SPSG+LoFTR/DKM のアンサンブル。
    * **Coarse SfM:** マッチング結果を信頼度ベースでNMSマージし、一貫性を向上させた上でCOLMAPで粗いSfMモデルを構築。PBA (Parallel BA) を利用（初期段階はLM法）。
    * **Iterative Refinement:** 粗いSfMモデルを入力とし、独自Attentionベース多視点マッチングで特徴トラックを精密化し、幾何リファインメント (幾何BA + トラック位相調整) を行う処理を反復。
    * **その他:** NetVLADによるペア選択。COLMAPのランダム性排除のためシングルスレッド実行。

**[2位](https://www.kaggle.com/competitions/image-matching-challenge-2023/discussion/416873)**

* **アプローチ:** SuperPoint/SuperGlue (SPSG) を中心に、複数解像度でのアンサンブル (TTA) と、COLMAPのランダム性に対処する工夫を導入。
* **アーキテクチャ:** SuperPoint, SuperGlue, COLMAP。回転検出モデル (`check_orientation`)。
* **アルゴリズム:** Exhaustive Matching (全ペア候補)、複数回実行と投票によるマッチングフィルタリング、複数回実行によるSfMモデル選択。
* **テクニック:**
    * **ペア選択:** 画像検索モデルは使用せず、全てのユニークなペア候補を作成し、SPSGマッチ数が閾値 (100) 未満のペアを除外。
    * **マッチング:** SPSG (FP16、Keypoint Threshold=0.005, Match Threshold=0.2)。キーポイント/ディスクリプタ/マッチをキャッシュ。
    * **TTA:** 複数解像度 ([1088, 1280, 1376]) でSPSGを実行し、マッチを結合。
    * **回転補正:** 回転検出モデルで画像の向きを補正。
    * **COLMAP初期化:** 最もペア数/マッチ数が多い画像を初期画像として明示的に指定。
    * **ランダム性対策:**
        * `match_exhaustive`を複数回 (10回) 実行し、8回以上出現したマッチのみを採用。
        * `incremental_mapping`を異なる閾値 ([100, 125, 75, 100]) で複数回実行し、登録画像数/3D点数で最良のモデルを選択（小規模シーンのみ）。

**[3位](https://www.kaggle.com/competitions/image-matching-challenge-2023/discussion/416918)**

* **アプローチ:** SuperPoint/SuperGlue (SPSG) をベースに、効率的なスクリーニング、画像分割によるマッチ数増加、DKMとのアンサンブル、並列処理を導入。
* **アーキテクチャ:** SuperPoint, SuperGlue, DKM v3, COLMAP。
* **アルゴリズム:** DBSCANベースのクラスタリング（2022年解法参照、ただし今回は不使用か？）。並列処理 (Queue)。
* **テクニック:**
    * **スクリーニング:** 軽量SPSG (longside=1200) でマッチングし、マッチ数が閾値 (30) 未満のペアを以降の処理から除外。回転補正もこの段階で実施。
    * **画像分割:** 各画像を4分割し、全組み合わせ (4x4=16ペア) でSPSG (longside=1400) を実行。TTAより効果的かつ高速。
    * **アンサンブル:** SPSGとDKMv3の結果をアンサンブル。互いに補完的なマッチングを提供。
    * **並列処理:** GPUを使うマッチング処理とCPUを使うCOLMAPマッピング処理をキューで並列化し、時間効率を改善 (20-30%向上)。
    * **COLMAP設定:** スレッド数を1に設定し、OOMを回避。

**[4位](https://www.kaggle.com/competitions/image-matching-challenge-2023/discussion/416816)**

* **アプローチ:** 軽量SPSGを用いたkNNベースのペア選択、回転補正、並列処理、2022年優勝解法のクロップ+マルチスケールアンサンブルを応用。
* **アーキテクチャ:** SuperPoint, SuperGlue, MatchFormer, COLMAP。
* **アルゴリズム:** kNN Shortlisting、DBSCANベースのクラスタリング+マルチスケールアンサンブル (2022年解法)。
* **テクニック:**
    * **ペア選択 (kNN Shortlist + Completion):** 軽量SPSG (KP=512) で全ペアをマッチングし、RANSAC後のインライア数に基づいて各画像のk近傍ペアを選択。どの画像も最低kペアを持つように、マッチキーポイント数で補完。
    * **回転補正:** 軽量SPSGで4方向回転マッチングを行い、最適な回転を検出・補正。
    * **マッチング:** SPSG + MatchFormer のマルチスケール・クロップアンサンブル (詳細は2022年解法参照)。
    * **並列処理:** マッチング (GPU) と COLMAP (CPU) を別スレッドで並列実行。
    * **効果がなかったこと:** TTA、Pixel-Perfect SfM (環境構築問題)、TensorRT、FP16、他のマッチャー、OpenGLUE、Mapperパラメータ調整 (OOM問題)。

**[5位](https://www.kaggle.com/competitions/image-matching-challenge-2023/discussion/417045)**

* **アプローチ:** KeyNetAffNetHardNet + AdaLAM をベースに、複数の類似検出器とのアンサンブル、およびCOLMAPとの連携を最適化。
* **アーキテクチャ:** KeyNetAffNetHardNet (および類似の非学習ベース検出器 GFTTAffNetHardNet など), HardNet8, AdaLAM, COLMAP。
* **アルゴリズム:** USAC_MAGSAC (幾何検証)。
* **テクニック:**
    * **特徴抽出:** 4種類のKeyNetAffNetHardNet系検出器を使用 (KP=8000, longside=1600)。OriNet有効化 (回転頑健性)。
    * **マッチング:** AdaLAMで全ペアをマッチング。平均マッチング距離 < 0.5 のペアのみ保持。`force_seed_mnn=True`, `ransac_iters=256`。
    * **アンサンブル:** 4つの検出器からのマッチを結合。
    * **幾何検証:** OpenCVのUSAC_MAGSACで基本行列を計算し、インライアのみ保持。結果をtwo-view geometryとしてCOLMAPデータベースに直接書き込み、`match_exhaustive`をスキップ（決定論的結果のため）。
    * **焦点距離:** EXIFから焦点距離を抽出し、COLMAPに`prior_focal_length=True`として設定。
    * **並列処理:** シーンごとにマッチング (DB生成) と再構成 (COLMAP) を行い、異なるシーンの再構成と次のシーンのマッチングを並列化。シーンは画像数が多い順に処理。

**[7位](https://www.kaggle.com/competitions/image-matching-challenge-2023/discussion/427143)**

* **アプローチ:** SuperPointの代替としてDISK, ALIKED, SIFTを使用し、新しいマッチャーLightGlueと組み合わせる。PixSfMによるリファインメント。ライセンスフリーな解法を目指す。
* **アーキテクチャ:** ALIKED, DISK, SIFT, SuperPoint (比較用)。LightGlue, NN-Ratio (SIFT用)。PixSfM, COLMAP。
* **アルゴリズム:** 画像検索 (NetVLAD)、アンサンブル。
* **テクニック:**
    * **ペア選択:** NetVLAD (k=20, 30 or 50)。
    * **特徴抽出:** ALIKED (KP=2048), DISK (KP=5000), SIFT, SuperPoint。画像サイズ (longside=1600)。
    * **マッチング:** SIFTはNN-Ratio。他はLightGlue (MegaDepthで学習済み)。
    * **アンサンブル:** 複数の特徴抽出・マッチングの組み合わせ (例: ALIKED2K+DISK+SIFT with LG+LG+NN)。
    * **SfM:** COLMAP。PixSfMによるリファインメント（回転画像がない場合のみ）。シーン内の全画像でカメラパラメータを共有する設定（画像形状が同じ場合）。
    * **後処理:** HLocを用いて未登録画像を追加ローカライズ。
    * **LightGlue:** SuperGlueより高速かつ高精度な新しいマッチャー。APACHEライセンス。

**[9位](https://www.kaggle.com/competitions/image-matching-challenge-2023/discussion/416842)**

* **アプローチ:** 回転補正、NetVLADによる高度なペア選択、SPSG+MagSac、COLMAPを組み合わせたパイプライン。
* **アーキテクチャ:** 回転検出モデル (`check_orientation`)、NetVLAD、SuperPoint, SuperGlue、COLMAP。
* **アルゴリズム:** Binary Search (ペア数決定)。MAGSAC。
* **テクニック:**
    * **回転補正:** 回転検出モデルを使用。自己アンサンブル（複数回転バージョンで繰り返し予測）と閾値調整で精度向上。
    * **ペア選択 (Retrieval):** NetVLADを使用。水平反転画像とのディスクリプタ加算（対称性対策）。Re-ranking（Top1ペアで再検索しスコア合算）。ペア数をBinary Searchで決定（グラフ連結性をチェック）。小規模シーンでは全ペア使用。
    * **マッチング:** SuperPoint + SuperGlue。
    * **幾何検証:** MAGSAC。
    * **COLMAP:** インクリメンタルマッピング。
    * **決定論:** シングルスレッド実行でCOLMAPのランダム性を排除。
    * **注意点:** OpenCV/PILの自動回転挙動に注意。
    * **代替案:** GlueStick + SuperPointも有望（商用利用可能）。
    * **効果がなかったこと:** Pixel-Perfect-SFM、セマンティックマスク、照明補正、PnPローカライズ、他のマッチャー、他の検索手法、グリッドサンプリングなど。
