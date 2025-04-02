---
tags:
  - Kaggle
  - UNet
  - Dice係数
startdate: 2023-05-11
enddate: 2023-08-10
---
# Google Research - Identify Contrails to Reduce Global Warming
[https://www.kaggle.com/competitions/google-research-identify-contrails-reduce-global-warming](https://www.kaggle.com/competitions/google-research-identify-contrails-reduce-global-warming)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、衛星画像データを用いて、航空機によって生成される**飛行機雲 (contrails)** を正確に検出し、その領域を特定（セグメンテーション）するモデルを開発することです。
* **背景:** Google Research 主催。飛行機雲は、地球のエネルギー収支に影響を与え、結果として地球温暖化の一因となることが指摘されています。飛行機雲の形成を予測し、それを回避するような飛行経路を選択することで、航空業界が気候変動に与える影響を軽減できる可能性があります。このためには、衛星画像から飛行機雲を正確に検出・識別する技術が不可欠です。
* **課題:**
    * **飛行機雲の検出:** 飛行機雲は細長く、形状や輝度が多様であり、特に薄いものや断片的なものは検出が困難です。また、自然に発生する巻雲など、類似した外観を持つ他の雲との区別も難しい課題です。
    * **ピクセルレベルの精度:** 飛行機雲は細い線状構造であるため、その形状を正確に捉えるためにはピクセルレベルでの高いセグメンテーション精度が求められます。
    * **ラベルの不確実性:** 専門家によるアノテーションには、特に境界部分などで意見の不一致（ノイズ）が含まれる可能性があります。
    * **画像の特性:** 赤外線バンドを含むマルチスペクトル衛星画像データを扱う必要があります。
    * **0.5ピクセルのずれ問題:** コンペ期間中に、提供されたアノテーションマスクが実際の飛行機雲の位置からわずかに（右下へ0.5ピクセル）ずれていることが発見され、これへの対応が精度向上に重要となりました。

**データセットの形式 (Dataset Format)**

提供されるデータは、時系列の衛星画像と、対応する飛行機雲のマスク情報です。

1.  **トレーニングデータ:**
    * `train/[record_id]/`: 各訓練サンプル（記録ID）のディレクトリ。
        * `band_08.npy` から `band_16.npy`: 9つの赤外線バンドの画像データ。各ファイルは8つのタイムフレームを含む (`[T, H, W] = [8, 256, 256]`) NumPy配列。
    * `train_contrail_masks.csv`: **ターゲット変数**。各 `record_id` の4番目のタイムフレーム (`T=3`) における飛行機雲のマスク情報。
        * `record_id`: サンプルID。
        * `encoded_pixels`: 飛行機雲が存在するピクセルの位置を示すランレングスエンコーディング (RLE) 文字列。飛行機雲がない場合は空欄。
    * `train_metadata.csv`: 各 `record_id` のタイムスタンプ情報。
    * `human_pixel_masks.npy`: 各 `record_id` のフレーム4に対する、複数のアノテーターによるピクセル単位のマスクデータ (`[record_id, H, W, num_annotators]`)。ソフトターゲット作成などに利用可能。
    * `human_individual_masks.npy`: 各アノテーターごとの個別のマスクデータ（ブール型）。
2.  **テストデータ:**
    * `test/[record_id]/`: トレーニングデータと同様の形式で、各テストサンプルの複数バンド・複数フレームの画像データ。
3.  **`sample_submission.csv`**:
    * 提出フォーマットのサンプル。
        * `record_id`: テストサンプルID。
        * `encoded_pixels`: 予測された飛行機雲マスクのRLE文字列。

**評価指標 (Evaluation Metric)**

* **指標:** **Dice係数 (Dice Coefficient)**
* **計算方法:** 予測された飛行機雲のピクセル集合 (Prediction) と、正解の飛行機雲ピクセル集合 (Ground Truth) の間の重複度を測る指標です。
    * `Dice = (2 * |Prediction ∩ GroundTruth|) / (|Prediction| + |GroundTruth|)`
    * `|X|` は集合Xのピクセル数を表します。`∩` は両方の集合に含まれるピクセル（共通部分）を示します。
    * スコアは0（全く重複しない）から1（完全に一致）の範囲を取ります。
    * 正解マスクが空（飛行機雲なし）の場合、予測マスクも空であればDiceスコアは1、予測マスクが空でなければ0となります。
    * 最終スコアは、全てのテストサンプルに対するDice係数の平均値です。
* **意味:** 予測された飛行機雲の形状と位置が、実際の飛行機雲とどれだけ正確に一致しているかを評価します。偽陽性（飛行機雲でない領域を予測）と偽陰性（実際の飛行機雲を見逃す）の両方にペナルティを与えるバランスの取れた指標です。スコアは**高い**ほど良い性能を示します。

要約すると、このコンペティションは、時系列衛星画像から飛行機雲の領域を高精度にセグメンテーションするタスクです。データはマルチバンドの画像シーケンスとRLE形式のマスクで構成され、性能はピクセル単位の一致度を測るDice係数（高いほど良い）によって評価されます。

---

**全体的な傾向**

この飛行機雲検出コンペティションでは、**U-Net** およびその派生アーキテクチャを用いた**セグメンテーション**が標準的なアプローチとなりました。エンコーダーには、**EfficientNet**, **MaxViT**, **ConvNeXt**, **CoaT**, **ResNeSt**, **NFNet** など、強力な画像認識モデルの事前学習済み重みが利用されました。

入力データとしては、主催者が提供したノートブックで示された赤外線バンドを組み合わせた**偽カラー画像「Ash Color」** が広く用いられましたが、**全9バンド**を使用したり、バンド間の相関を考慮して**直交化**した特徴量を使用したりする試みも見られました。また、入力画像の**解像度を上げる (アップサンプリング)** ことが精度向上に有効であり、512x512 や 768x768、1024x1024 といった高解像度が採用されました。

時間情報を活用するために、単一フレーム (t=4) のみでなく、前後数フレームを含む**複数フレームを入力**とするアプローチが主流でした。これらは、**2.5D アプローチ**（2Dバックボーンの中間層に3D畳み込みやLSTM/Transformerを挿入）や、**チャネル・空間次元へのスタッキング**（例: 4フレームを2x2のパネルにする）といった形で実装されました。

学習ターゲットとしては、提供されたRLEマスクから生成されるバイナリマスクではなく、複数のアノテーターによるマスクの**平均値を用いたソフトターゲット**を使用することが一般的で、これによりラベルノイズに対する頑健性が向上しました。

コンペ中に発見された**アノテーションマスクの0.5ピクセルずれ**問題への対応が、上位入賞の鍵となりました。主な対策は、(1) マスクを事前に0.5ピクセル補正（シフト）して学習し、推論後に逆シフトまたは補正用モデルを適用する、(2) 入力画像を0.5ピクセルシフトして学習・推論する、の2つでした。この補正により、**フリップや90度回転といった幾何学的なデータ拡張 (Augmentation) や Test Time Augmentation (TTA)** が有効になりました。

損失関数としては、**BCE Loss** がソフトターゲットとの相性が良く主流でしたが、**Dice Loss** や **Lovasz Loss**, **Focal Loss** との組み合わせも用いられました。

最終的な提出は、異なるモデルアーキテクチャ、異なる入力解像度、異なる学習Fold、異なるランダムシード、そしてTTAの結果を組み合わせた**アンサンブル**によって行われました。閾値は固定値またはパーセンタイルで最適化されました。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/google-research-identify-contrails-reduce-global-warming/discussion/430618)**

* **アプローチ:** U-Net + MaxViT。**0.5ピクセルずれ**を発見し、**対称化ソフトラベル**で学習後、小型CNNで補正。
* **アーキテクチャ:** U-Net (MaxViT-tinyエンコーダー)。補正用Conv5x5。
* **アルゴリズム:** セグメンテーション。
* **テクニック:** 入力: Ash Color画像 (1024x1024にアップスケール)。ソフトターゲット使用。対称化ラベル `y_sym` を学習ターゲットとし、推論後に5x5 Convで最終予測 `y` に変換。Augmentation (RandomRotate90, HorizontalFlip, ShiftScaleRotate)。TTA (8パターン: Flip+Rot90)。単一時刻(t=4)モデルと4パネル(t=1-4)入力モデルのアンサンブル。

**[2位](https://www.kaggle.com/competitions/google-research-identify-contrails-reduce-global-warming/discussion/430491)**

* **アプローチ:** カスタムU-Net形状モデルアンサンブル。時間情報混合。高解像度入力。
* **アーキテクチャ:** U-Net型 (エンコーダー: CoaT, NeXtViT, SAM-B, EfficientNetV2s)。デコーダーにPixel Shuffle利用。
* **アルゴリズム:** セグメンテーション。
* **テクニック:** 入力: Ash Color (x2, x4アップスケール)。ソフトターゲット。時間情報混合 (中間特徴量マップにLSTM, Transformer, 1D Conv適用)。損失: BCE + Dice Lovasz Loss。ピクセルレベル精度重視。

**[3位](https://www.kaggle.com/competitions/google-research-identify-contrails-reduce-global-warming/discussion/430685)**

* **アプローチ:** **2.5D U-Net** アンサンブル。疑似ラベル活用。
* **アーキテクチャ:** 2.5D U-Net (多様なエンコーダー)。中間スキップコネクションに3D Conv挿入。
* **アルゴリズム:** 2.5Dセグメンテーション。
* **テクニック:** 入力: Ash Color (512x512)。ソフトターゲット。損失: (-Dice + BCE) / 2。Augmentation (Flip/Rotateは低確率)。**疑似ラベル**による事前学習+ファインチューニング。パーセンタイル閾値。大規模アンサンブル (18モデル + チームメンバーモデル)。

**[4位](https://www.kaggle.com/competitions/google-research-identify-contrails-reduce-global-warming/discussion/432998)**

* **アプローチ:** U-Netアンサンブル (大規模エンコーダー)。疑似ラベル活用。
* **アーキテクチャ:** U-Net (エンコーダー: EfficientNet-V2-L/XL, MaxViT)。
* **アルゴリズム:** セグメンテーション。
* **テクニック:** **疑似ラベル** (全データで2ラウンド生成)。ソフトターゲット。高解像度入力 (512, 768)。複合損失 (CE, Dice, Focal)。EMA + SWA。4TTA (hflip, rot90, rot270)。

**[5位](https://www.kaggle.com/competitions/google-research-identify-contrails-reduce-global-warming/discussion/430549)**

* **アプローチ:** 単一フレーム2D U-Netと複数フレーム3D U-Netのアンサンブル。**ピクセルずれ対応**。
* **アーキテクチャ:** 2D U-Net (EfficientNetV2L), カスタム3D U-Net (EfficientNetV2L + Conv3D/ConvLSTM)。
* **アルゴリズム:** 2D/3Dセグメンテーション。
* **テクニック:** 入力: Ash Color (高解像度)。**ピクセルずれ補正** (x=0.408, y=0.453のオフセット？)。ソフトターゲット+マルチクラス風ターゲット (5チャネル)。損失: 重み付きBCE + Dice Loss。TTA8 (Flip+Rotate)。

**[6位](https://www.kaggle.com/competitions/google-research-identify-contrails-reduce-global-warming/discussion/430581)**

* **アプローチ:** 2段階パイプライン (分類+セグメンテーション)。2.5Dモデルと2Dモデルのアンサンブル。
* **アーキテクチャ:** 分類/セグメンテーション共に: 2.5Dモデル (ResNetRS101, Swin, ConvNeXtなど), 2D U-Net (Res2Net, RegNetZなど)。
* **アルゴリズム:** 分類、セグメンテーション。
* **テクニック:** ソフトターゲット。**2Dモデルは11チャンネル入力** (Ash Color差分+全9バンド)。損失: BCE+Dice。**パーセンタイル閾値**。EMA。

**[7位](https://www.kaggle.com/competitions/google-research-identify-contrails-reduce-global-warming/discussion/430691)**

* **アプローチ:** U-Netアンサンブル。**直交化9チャンネル入力**。
* **アーキテクチャ:** U-Net (EfficientNet-B7/B8)。
* **アルゴリズム:** セグメンテーション。
* **テクニック:** 入力: 512x512。**9バンドをSVDで直交化**し入力として使用。ソフトターゲット。Augmentation (Rot90+Flip+ShiftScaleRotate)。TTA8。BCE/Dice損失。

**[8位](https://www.kaggle.com/competitions/google-research-identify-contrails-reduce-global-warming/discussion/430543)**

* **アプローチ:** **OneFormer** + その他CNNモデルのアンサンブル。ソフト/疑似ラベル。
* **アーキテクチャ:** OneFormer (DiNAT-L), EfficientNet, MaxViT, ResNetRS, NFNetF5。
* **アルゴリズム:** セグメンテーション。
* **テクニック:** ソフトターゲット。疑似ラベル使用。Optunaによるアンサンブル重み最適化。BCE損失。高解像度入力。

**[9位](https://www.kaggle.com/competitions/google-research-identify-contrails-reduce-global-warming/discussion/430479)**

* **アプローチ:** U-Net。**ピクセルずれ問題**を発見し、入力画像シフトで対応。
* **アーキテクチャ:** U-Net (多様なエンコーダー)。
* **アルゴリズム:** セグメンテーション。
* **テクニック:** 入力: False Color (512x512)。**入力画像を+0.5ピクセルシフト**して学習・推論。ソフトターゲット (全個別アノテーション使用)。Augmentation (Flip/Rotate有効化)。TTA8。
