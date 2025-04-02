---
tags:
  - Kaggle
  - ヘルスケア
  - UNet
startdate: 2023-11-08
enddate: 2024-02-07
---
# SenNet + HOA - Hacking the Human Vasculature in 3D
[https://www.kaggle.com/competitions/blood-vessel-segmentation](https://www.kaggle.com/competitions/blood-vessel-segmentation)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、高解像度の3D顕微鏡画像データを用いて、人間の腎臓内部の**複雑な血管網（特に糸球体などの微小血管を含む）を正確にセグメンテーション（領域分割）する**機械学習モデルを開発することです。
* **背景:** 人体組織の微細な3次元構造、特に血管系の構造を理解することは、老化プロセスを研究するSenescence Network (SenNet)プログラムや、人体の器官を詳細にマッピングするHuman Organ Atlas (HOA)プロジェクトにとって極めて重要です。血管構造の正確なマッピングは、疾患の理解や診断、治療法の開発に貢献します。しかし、手作業による3D血管セグメンテーションは膨大な時間と労力を要するため、自動化技術の開発が強く望まれています。
* **課題:** 提供されるデータは非常に高解像度（マイクロメートルスケール）の3次元画像であり、血管構造は極めて複雑で微細です。特に毛細血管のような細い構造や分岐部を正確に捉えることが技術的な課題となります。また、トレーニングデータにはアノテーション（正解ラベル）が密に付与されたデータセット（dense）と、疎らにしか付与されていないデータセット（sparse）が含まれており、このラベルの不完全性に頑健なモデルを構築する必要がありました。さらに、PublicテストセットとPrivateテストセットで画像の解像度が異なる（50um/voxel vs 63um/voxel）という点も考慮が必要でした。

**データセットの形式 (Dataset Format)**

提供される主なデータは、腎臓の3D画像スライスと、対応する血管のセグメンテーションマスクです。

1.  **トレーニングデータ:**
    * 3つの腎臓サンプル（`kidney_1_dense`, `kidney_2_sparse`, `kidney_3_dense`, `kidney_3_sparse`）の高解像度3D画像データ。
    * 各サンプルは、連続する2D画像スライス（TIFF形式）の集まりとして提供されます (`train/{dataset_id}/images/`)。これは実質的に3Dボリュームデータを表します。
    * 各画像スライスに対応する血管のセグメンテーションマスク（正解ラベル）もTIFF形式（2値画像）で提供されます (`train/{dataset_id}/labels/`)。`dense`ラベルは血管領域が密にマークされており、`sparse`ラベルは一部の血管のみがマークされています。
    * `kidney_1_voi`は血管以外の関心領域データで、使用は任意でした。
    * `HuBMAP-20-background_controls`フォルダには背景コントロール画像が含まれます。
2.  **テストデータ:**
    * トレーニングデータと同様の形式で、2つの腎臓サンプル（`kidney_5`, `kidney_6`）の3D画像データ（TIFFスライス）が提供されます (`test/{dataset_id}/images/`)。
    * 正解ラベル（マスク）は提供されません。参加者はこれらの画像データに対して血管領域を予測します。
    * **注意点:** テストデータ`kidney_5`（Public LB用）と`kidney_6`（Private LB用）では、ボクセルあたりの解像度がトレーニングデータ（50um/voxel）と異なり、`kidney_6`は63um/voxelでした。
3.  **外部データ:**
    * Human Organ Atlas (HOA) プロジェクト ([https://human-organ-atlas.esrf.eu/](https://human-organ-atlas.esrf.eu/)) から、追加の腎臓データや他の臓器（脾臓など）のデータが利用可能でした。一部のチームはこれらのデータを疑似ラベル生成などに活用しました。
4.  **`sample_submission.csv`**:
    * 提出フォーマットのサンプル。`id`（データセットID、スライス番号、ピクセル位置などを含む識別子）と`rle`（予測された血管領域のランレングスエンコーディング）の列を持ちます。

**評価指標 (Evaluation Metric)**

* **指標:** **Dice係数 (Dice Coefficient / Sorensen-Dice Index)**
* **計算方法:** 予測された血管領域（ピクセル集合）と実際の血管領域（正解ラベル）の間のオーバーラップを測定します。
    * Dice = (2 * |予測領域 ∩ 正解領域|) / (|予測領域| + |正解領域|)
    * 値は0から1の範囲を取り、完全に一致する場合に1となります。
* **意味:** セグメンテーションの精度を測る標準的な指標です。予測と正解の一致度が高いほど値が大きくなります。特に、このコンペでは血管のような細長く分岐する構造を対象としており、ピクセル単位での一致度が重要となります。コンペの説明では**Surface Dice**（表面での一致度を測る指標で、細い構造の評価に適している）の重要性も言及されていましたが、最終的な評価指標は標準的なDice係数でした。スコアは**高い**ほど良い評価となります。

---

**全体的な傾向**

この高解像度3D血管セグメンテーションタスクでは、**U-Netベースのアーキテクチャ**が圧倒的な主流でした。特に、**2.5Dアプローチ**（複数の隣接スライスをチャンネルとして2D U-Netに入力する）が、完全な3D U-Netよりも計算コストと性能のバランスから多くの上位チームに採用されました。エンコーダー（Backbone）としては、**ConvNeXt**、**EfficientNet**、**MaxViT**などの強力な画像認識モデルが好まれました。

データ処理戦略として、**マルチビュー推論**が鍵となりました。入力となる3DボリュームをX, Y, Zの3軸方向からスライスし、それぞれの方向で学習・推論を行い、最終的に予測結果を統合することで、3次元的な構造情報を効果的に捉え、精度を向上させました。

高解像度データへの対応として、画像全体ではなく、一定サイズの**パッチ（タイル）に分割**して処理するか、**画像をリサイズ**して入力するアプローチが取られました。入力解像度は512x512から1536x1536まで様々でした。

**疎ラベル（sparse labels）の活用**も重要なテーマでした。多くのチームは、まず密ラベル（dense labels）データでモデルを学習させ、そのモデルを使って疎ラベルデータや外部データに**疑似ラベル（Pseudo labels）を生成**し、それらを学習データに追加してモデルを再学習する（**Refining from Sparse to Dense**）戦略を採用しました。疑似ラベルはソフトラベル（確率値）のまま使うことが効果的でした。

PublicテストセットとPrivateテストセットの**解像度の違い（50um vs 63um）** に対処することも、Private LBでの成功に不可欠でした。推論時にPrivateテストデータ（63um）をトレーニングデータの解像度（50um）に合わせて**3D補間（リサイズ）**し、予測後に元の解像度に戻す手法が極めて有効でした。

損失関数としては、標準的な**Dice Loss**、**Focal Loss**、**BCE Loss**の組み合わせに加え、境界領域の学習を強化する**Boundary Loss**や、評価指標であるSurface Diceを意識した**カスタム損失関数**も試されました。

強力な**データ拡張（Augmentation）** も広く用いられ、幾何学的変換（フリップ、回転、アフィン変換、Elastic Transformなど）や、輝度・コントラスト調整、ノイズ付加に加え、**CutMix**や、3Dボリュームをランダムに回転させてからスライスを切り出す**3D回転拡張**も効果的でした。

推論時には、マルチビュー推論、**TTA（Test Time Augmentation）**、閾値最適化が行われました。一部の3Dモデルベースの解法では、**後処理**として小さな連結成分（偽陽性の塊）を除去する手法も有効でした。最終提出は、複数のモデルや異なる設定（Fold、Augmentation、入力解像度など）からの予測を組み合わせた**アンサンブル**が一般的でした。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/blood-vessel-segmentation/discussion/475522)**

* **アプローチ:** 2つの2.5D U-Netモデル（ConvNeXt-tiny backbone）のアンサンブル。マルチビュー（x, y, z軸スライス）入力。
* **アーキテクチャ:** SMP U-Net + ConvNeXt-tiny backbone。BatchNormをGroupNormに、ReLUをGELUに変更。入力層に畳み込みステムを追加。入力3ch（隣接3スライス）。
* **アルゴリズム:** AdamW + CosineAnnealingLR。損失関数: Focal Loss + Dice Loss + Boundary Loss + カスタムロス（Surface Dice模倣）。
* **テクニック:**
    * **データ:** 全トレーニングデータ使用。正規化は `/65535.0` のみ。入力サイズ1536x1536。
    * **Augmentation:** 多様な幾何学的・輝度変換に加え、**3D回転拡張**（一方のモデルのみ、エポック数も増加）。
    * **学習:** バッチサイズ8 x 勾配蓄積4。通常モデル20 epochs、3D回転拡張モデル30 epochs。
    * **推論:** 3軸推論 + 8xTTA。高解像度推論（3072x3072 or 動的スケール）。閾値0.4。`torch.compile()` で高速化。
    * **備考:** アンサンブルより3D回転拡張ありのシングルモデルの方がPrivate LBスコアが高かった (0.835)。

**[2位](https://www.kaggle.com/competitions/blood-vessel-segmentation/discussion/475657)**

* **アプローチ:** 単一の3D U-Netモデル。後処理によるノイズ除去が特徴。
* **アーキテクチャ:** 3D U-Net (入力パッチサイズ 128x128x32)。
* **アルゴリズム:** AdamW + CosineDecayLR。損失関数: Binary Focal Loss。
* **テクニック:**
    * **データ:** 全データを3D Numpy配列化し、スケール調整。正規化・クリッピングは不使用。ランダムな位置・回転でパッチを生成（データ拡張）。
    * **学習:** マルチプロセスでデータ生成を高速化。
    * **推論:** タイリング（パッチ分割）で予測。
    * **閾値調整:** 血管の体積比率がある程度一定であると仮定し、予測される血管体積の割合に基づいて閾値を調整。
    * **後処理:** 予測マスクに対して連結成分解析を行い、非常に小さい（接続されていない）血管セグメントを偽陽性として除去する。この後処理がスコアに大きく貢献。
    * **備考:** Public LBは低かったが (0.43)、後処理によりPrivate LBで2位に浮上。大きな塊のみを残す後処理はPublic LBが0.00になった。

**[3位](https://www.kaggle.com/competitions/blood-vessel-segmentation/discussion/475074)**

* **アプローチ:** 2D U-Netモデルのみを使用。疎ラベルから密ラベルへの段階的改良（Refining from Sparse to Dense）と、Privateテストセットの解像度を模倣した学習が特徴。
* **アーキテクチャ:** SMP U-Net (MaxViT-Large, EfficientNetV2-s, SeResNext101 backbone), UNet++ (EfficientNetV2-l backbone)。
* **アルゴリズム:** AdamW? 損失関数: ? (Diceなど標準的なものか)
* **テクニック:**
    * **ラベル精緻化:** Denseデータで学習→Sparseデータ(kidney 3)に疑似ラベル生成→全データ(k1, k3 dense+sparse+pseudo)で再学習→Sparseデータ(kidney 2)に疑似ラベル生成→全データ+k2疑似ラベルで最終学習。疑似ラベル生成時の閾値は公式の注釈割合を参考に選択。
    * **解像度模倣:** Privateテストセットの解像度(63um/voxel)が低いことを考慮し、学習時のスケール拡張の中心を0.8に設定（縮小方向を重視）。
    * **入力解像度:** 主に512x512で学習・推論。高解像度推論のリスク（特にXY軸方向）を考慮。
    * **全データ学習:** モデルの収束が安定していたため、CVを行わず全腎臓データ+疑似ラベルで最終モデルを学習。
    * **強度Augmentation:** 腎臓間の強度差に対応するため、輝度/コントラスト/ガンマのAugmentationを強く適用。
    * **閾値安定性:** モデル（特にMaxViT）が閾値に対して比較的安定していたため、最終モデルでも過去の検証で見つけた閾値を適用可能だった。
    * **アンサンブル/最終提出:** MaxViT単体と4モデルアンサンブルの両方を提出し、偶然どちらもPrivate LB 0.727を記録。

**[4位](https://www.kaggle.com/competitions/blood-vessel-segmentation/discussion/475052)**

* **アプローチ:** 2Dモデルと3Dモデルのアンサンブル。Boundary DoU Lossを使用。マルチビューTTA。
* **アーキテクチャ:**
    * 2D: Unet++ (EfficientNet-b5, b6, b7, mit_b5 backbone) + SCSE Attention。
    * 3D: DynUnet (MONAIライブラリ、nnUnet風アーキテクチャ)。
* **アルゴリズム:**
    * 2D: AdamW + Cosine LR。
    * 3D: SGD + CosineAnnealingLR。
    * 損失関数: **Boundary DoU Loss** (論文ベース、両モデルで使用)。
* **テクニック:**
    * **データ:** 全トレーニングデータ + HOAの腎臓データ (`50um_LADAF-2020-31_kidney`) の疑似ラベルを使用。スタックワイズのパーセンタイル正規化。
    * **疑似ラベル:** 腎臓データのみ使用（他のHOAデータは使用せず）。2Dモデルアンサンブルで生成。
    * **学習:** 2-Fold CV (kidney_2, kidney_3_denseをValidation)。重み付きサンプリング（Denseは1, Sparseはその疎密度に応じた重み）。Augmentationに**CutMix** (同じ臓器・同じ軸方向からクロップ) を使用。
    * **推論 (2D):** マルチビュー推論 (3軸)。Sliding Window (Crop 800, Overlap 0.25)。D4 TTA。
    * **推論 (3D):** Sliding Window (Crop 256, Overlap 0.25)。D4 TTA。
    * **後処理:** 3Dモデルの予測に対して、2Dモデルの予測から作成したROIマスクを適用し、腎臓外部の偽陽性を除去（Public/Privateでスコア向上）。Canny Filterによる腎臓セグメンテーションマスクはPublicは向上したがPrivateで失敗。
    * **アンサンブル:** 2Dモデルと後処理済み3Dモデルの予測を重み1:1で平均。

**[5位](https://www.kaggle.com/competitions/blood-vessel-segmentation/discussion/475288)**

* **アプローチ:** 2.5D U-Netモデルアンサンブル。段階的な疑似ラベル生成と外部データ（脾臓含む）活用。入力画像のアップスケールが特徴。
* **アーキテクチャ:** SMP U-Net (EfficientNet-v2-s/m, MaxViT-base, DPN68 backbone)。カスタム`UnetUpscale`モジュール。
* **アルゴリズム:** AdamW? 損失関数: Boundary強調CE Loss + Dice Loss + Focal Loss (一部モデルはTwersky Lossも)。
* **テクニック:**
    * **検証:** Kidney 1とKidney 3を交互に学習/検証データとして使用。
    * **データ/疑似ラベル:** 4段階の疑似ラベル生成プロセス。k1→k2疑似→k1+k2→腎臓外部データ疑似→k1+k2+腎臓外部→脾臓外部データ疑似→全データで最終学習。ソフトラベルを使用。
    * **入力処理 (UnetUpscale):** モデル内部で入力をバイリニア補間で2倍にアップスケールし、U-Netで処理後、再度ダウンスケールして出力。これにより、高解像度情報を効果的に利用。主に入力512x512を内部1024x1024で処理。
    * **損失関数:** 境界付近のピクセルに重みを置いたカスタムCE Lossを使用。
    * **推論:** マルチビュー推論 (3軸)。Sliding Window (Overlap 50%)。
    * **解像度対応:** Private LB用に推論時に**3D補間**を実施。入力画像を63um→50umにリサイズし予測、予測結果を50um→63umにリサイズして戻す。これによりPrivateスコアが大幅に向上 (0.634→0.670)。
    * **アンサンブル:** 異なるバックボーン、異なるFoldのモデルをアンサンブル。

**[6位](https://www.kaggle.com/competitions/blood-vessel-segmentation/discussion/475252)**

* **アプローチ:** 2.5Dモデルと3Dモデルの混合。疑似ラベル活用。リサイズによる解像度統一（50um/voxel）での推論も試行（Private LBで有効）。
* **アーキテクチャ:**
    * 2.5D: U-Net++ (EfficientNet-B3/B5/B7, se_resnext50 backbone)。入力5スライス。
    * 3D: DynUNet (MONAI)。
* **アルゴリズム:** 2.5D: Adam + CosineAnnealingLR。3D: SGD + CosineAnnealingLR。損失関数: Boundary Loss (+ 0.5 Focal Symmetric Loss がベスト)。
* **テクニック:**
    * **検証:** Kidney 2とKidney 3 denseをValidationに使用したが、スコアの相関は低かった。
    * **データ:** 全トレーニングデータ + HOA腎臓データの疑似ラベル使用。パーセンタイル正規化。
    * **学習 (2.5D):** マルチビュー入力。入力512x512クロップ。Non Empty確率0.5。標準Augmentation + CutMix。重み付きサンプリング（疑似ラベルデータはリピート）。
    * **学習 (3D):** MONAIのDynUNet使用。入力128x128x128? 標準Augmentation（Zoomは効果なし）。
    * **推論:** Sliding Window (Overlap 0.5)。Flip TTA。マルチビューTTA (2.5D)。
    * **後処理/解像度対応:** 腎臓マスク生成（セグメンテーションモデル or 強度ベースアルゴリズム）による後処理を試行。アルゴリズム版はPublic向上/Private悪化。**推論時に画像を50um/voxel相当にリサイズし、予測後に元に戻す**アプローチがPrivate LBで有効だった (0.753, 0.745)。
    * **アンサンブル/最終提出:** 2.5Dモデルと後処理ありの2.5Dモデルを提出。

**[7位](https://www.kaggle.com/competitions/blood-vessel-segmentation/discussion/475964)**

* **アプローチ:** 2D U-Netフレームワーク内に3D畳み込み層を組み込んだハイブリッド2.5Dモデル。3D回転拡張と疑似ラベルを活用。
* **アーキテクチャ:** 2D Unetベース + 3D畳み込み層。入力3ch（隣接3スライス）、出力3ch（中央予測を主に使用）。
* **アルゴリズム:** AdamW + CosineAnnealingLR。損失関数: BCEWithLogitsLoss + DiceLoss + FocalLoss（3chそれぞれ計算し中央chの重みを0.9に）。
* **テクニック:**
    * **データ準備:** 全腎臓データ使用。**3D回転拡張**により、任意の角度からのスライスを生成。疑似ラベル生成プロセスあり（k1+k3回転拡張で学習→k2疑似ラベル生成→k2回転拡張）。
    * **正規化:** 各腎臓の標準偏差に基づいて正規化。
    * **入力/解像度:** 入力3ch。クロップサイズ1024x1024。
    * **推論:** シングルモデル。マルチビュー推論（3軸平均）。TTAは水平フリップのみ。元の画像解像度で推論。
    * **後処理:** 閾値0.2適用後、3D Closing処理（形態学的演算）。

**[9位](https://www.kaggle.com/competitions/blood-vessel-segmentation/discussion/475080)**

* **アプローチ:** 2.5D U-Netモデル (MaxViT-tiny backbone)。マルチビュー推論。強力な正則化。
* **アーキテクチャ:** カスタムU-Net (SMP UnetDecoder + timm MaxViT-tiny backbone + カスタムConv Stem)。入力1ch (中央スライスのみ?)。
* **アルゴリズム:** AdamW (Weight Decay 1e-2) + ?。損失関数: Dice Loss (smooth_factor=0.1)。
* **テクニック:**
    * **データ:** Kidney 1とKidney 3のみ使用。
    * **正則化重視:** 不安定なCV/LBを考慮し、正則化を強化。EMA、長めの学習(50 epochs)、高Weight Decay、CutMix/MixUp (学習前半)、強力なAugmentation。
    * **Augmentation:** フリップ、輝度、ノイズ/ブラー、歪み (GridDistortion/OpticalDistortion)、ShiftScaleRotate、CoarseDropout。
    * **入力解像度:** 512x512クロップ。
    * **推論:** マルチビュー推論 (xy, xz, yz軸)。Sliding Window (Crop 512, Stride 256)。
    * **後処理:** 確率値に対する**単純な閾値処理**（パーセンタイル閾値は不安定と判断し不使用）。
    * **動作しなかったこと:** より大きなモデル、高解像度推論(1024)、Rotate90拡張、他の腎臓データでの事前学習。
