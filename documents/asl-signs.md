---
tags:
  - Kaggle
  - 正規化平均レーベンシュタイン距離
startdate: 2023-02-24
enddate: 2023-05-02
---
# Google - Isolated Sign Language Recognition
[https://www.kaggle.com/competitions/asl-signs](https://www.kaggle.com/competitions/asl-signs)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、アメリカ手話 (ASL) の**個別の単語（サイン）** を示している短い動画から抽出されたランドマークデータに基づき、どの**手話単語が表現されているかを分類する**モデルを開発することです。
* **背景:** GoogleのAI for Social Goodプログラムの一環。手話は世界中の多くの人々にとって主要なコミュニケーション手段ですが、手話認識技術はまだ発展途上です。このコンペティションは、手話の基本的な単位である個々のサイン（単語）の認識精度を向上させ、より高度な手話理解・翻訳システムの開発に貢献することを目的としています。
* **課題:**
    * **話者による変動:** 同じ手話単語でも、実行する人によって手の形、動きの速度、大きさ、位置、顔の表情などが異なります。
    * **視点の変動:** カメラの角度や距離によって、ランドマークの見え方が変化します。
    * **クラス内変動とクラス間類似性:** 同じ単語でも実行ごとに微妙な違いがある一方、異なる単語でも似た動きや手の形を持つ場合があります。
    * **データのノイズ:** Mediapipeで抽出されたランドマークには、ノイズや欠損（特に手が速く動く場合やオクルージョンがある場合）が含まれる可能性があります。
    * **TFLiteモデルの制約:** 提出モデルはTensorFlow Lite形式である必要があり、モデルサイズ（40MB）、使用可能な演算子（Ops）、推論時間（3時間）に制限があります。

**データセットの形式 (Dataset Format)**

提供されるデータは、手話のランドマーク時系列データと、対応する手話単語のラベルです。

1.  **トレーニングデータ:**
    * `train.csv`: 各訓練サンプルのメタデータ。
        * `path`: ランドマークデータファイル (`.parquet`) への相対パス。
        * `sequence_id`: 各サンプルのシーケンスID。
        * `sign`: **ターゲット変数**。手話単語のラベル（文字列）。250クラス存在します。
        * `participant_id`: 手話を行った話者のID。
    * ランドマークデータファイル (`train_landmarks/` ディレクトリ内の `.parquet` ファイル): 各フレームの手、顔、ポーズのランドマーク座標データ。
        * `frame`: フレーム番号。
        * `row_id`: `frame`-`type`-`landmark_index` 形式のID。
        * `type`: ランドマークの種類 (`face`, `left_hand`, `pose`, `right_hand`)。
        * `landmark_index`: 各タイプ内でのランドマークのインデックス番号。
        * `x`, `y`, `z`: 正規化されたランドマーク座標。
    * `sign_to_prediction_index_map.json`: 手話単語ラベルと予測インデックス（整数）のマッピング。
    * `supplemental_landmarks/`, `supplemental_metadata.csv`: 追加データ。
2.  **テストデータ:**
    * `test_landmarks/`: テスト用のランドマークデータファイル (`.parquet`)。
    * (隠しデータ): テストデータに対応する正解の `sign` ラベル。
3.  **`sample_submission.csv`**:
    * 提出フォーマットのサンプル。
        * `row_id`: テストシーケンスの `sequence_id`。
        * `sign`: モデルによって予測された手話単語のラベル。

**評価指標 (Evaluation Metric)**

* **指標:** **正解率 (Accuracy)**
* **計算方法:** モデルが予測した手話単語ラベルが、正解のラベルと一致したサンプルの割合。
    * `Accuracy = (Number of Correct Predictions) / (Total Number of Samples)`
* **意味:** モデルが250種類の手話単語をどれだけ正確に分類できるかを測る、最も基本的な分類指標です。スコアは**高い**ほど良い性能を示します (最大1.0)。

要約すると、このコンペティションは、手や顔などのランドマーク時系列データから、表現されている個別のASLサイン（単語）を分類するタスクです。データはランドマーク座標と単語ラベルで構成され、性能は正解率（高いほど良い）によって評価されます。TFLiteモデルの制約下での高精度化が求められます。

---

**全体的な傾向**

このASLサイン認識コンペティションでは、ランドマークの時系列データから250クラスの手話単語を分類するために、**Transformer** ベースのアーキテクチャと **1D CNN** ベースのアーキテクチャ、およびそれらの**ハイブリッド**が上位解法の中心となりました。特に、1D CNNで局所的な時間特徴を抽出し、その出力をTransformerに入力して大域的な時間依存関係を捉えるハイブリッドアプローチが1位チームによって採用され、高い効果を示しました。

**入力特徴量**としては、両手または利き手 (`left_hand`, `right_hand`) のランドマークに加え、**唇 (lips)**、**ポーズ (pose)**、**目 (REYE, LEYE)**、**鼻 (nose)** など、複数の部位の座標 (x, y, z または x, y) を利用することが一般的でした。生の座標だけでなく、**時間差分 (モーション)**、**点間距離**、**角度**といったハンドクラフト特徴量も補助的に用いられました。データの前処理として、**正規化**（基準点を用いたセンタリングや標準化）、**欠損値処理**（ゼロ埋めなど）、**フレームの間引きやリサイズ/パディング**によるシーケンス長調整が重要でした。

**データ拡張 (Augmentation)** は、モデルの頑健性を高め、過学習を防ぐために不可欠でした。空間的な拡張（アフィン変換、左右反転、指の回転）と時間的な拡張（リサンプリング、マスキング、カットアウト）に加え、異なるサンプルの一部を組み合わせる **Mixup** や **CutMix**、**異なる部位のデータを他のサンプルから持ってくる**拡張などが有効とされました。

**モデルアーキテクチャ**の工夫としては、Depthwise Separable Convolutionを用いた軽量な1D CNNブロック、BatchNormと活性化関数の選択（Swish, ReLUなど）、Transformerブロックの設計（ヘッド数、中間層次元、Positional Encoding）、Pooling層の選択（Global Average/Max Pooling）などがありました。

**学習戦略**では、長いエポック数（数百エポック）の学習、学習率スケジューリング（Cosine Decayなど）、Optimizerの選択（AdamW, RAdam+Lookahead）、**ラベルスムージング**、**AWP (Adversarial Weight Perturbation)**、**SWA/EMA**、**DropPath/Stochastic Depth**、**知識蒸留**などが用いられました。

**TFLite変換**と**推論効率化**も重要な課題でした。PyTorchで学習したモデルをONNX経由またはKerasコードに書き換えてTensorFlow/TFLiteに変換する手法が取られました。TFLiteのDepthwiseConv2D演算子の効率問題が指摘され、カスタム実装で高速化を図る試みもありました。fp16量子化も利用されました。

最終提出は、異なるシードや設定で学習した複数のモデルの**アンサンブル**が一般的でした。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/asl-signs/discussion/406684)**

* **アプローチ:** **1D CNN + Transformer** ハイブリッドモデル。
* **アーキテクチャ:** (1D Conv Block x3 + Transformer Block) x2 + GAP + Dense。Conv1DはDepthwise Separable + Causal Padding。TransformerはBatchNorm+Swish。
* **アルゴリズム:** シーケンス分類。
* **テクニック:** 入力: 手+唇+目+鼻+ポーズ(xy)。正規化(鼻基準)。差分特徴量(lag1, lag2)。Augmentation (Affine, Resample, Masking)。Drop Path, 高Dropout率, AWP。CCE Loss + ラベルスムージング。Optimizer: RAdam+Lookahead。4シードアンサンブル。TF+TPU学習。

**[2位](https://www.kaggle.com/competitions/asl-signs/discussion/406306)**

* **アプローチ:** **EfficientNet-B0** (画像モデル転用) + **Transformer (BERT/DeBERTa)** アンサンブル。
* **アーキテクチャ:** EfficientNet-B0 (入力160x80x3画像化)、BERT/DeBERTa (4層)。
* **アルゴリズム:** 画像分類、シーケンス分類。
* **テクニック:** 入力: 手+唇+ポーズ+目。CNN入力用にリサイズ・3次元化。Transformer入力用にハンドクラフト特徴量(モーション, 距離, 角度)。Augmentation (Affine, 時間補間, Flip, **指回転**, **クラス内別サンプル置換**, 時間/特徴マスキング)。Mixup。Weighted CE Loss。Lookahead+RAdam。知識蒸留(Transformer)。**カスタムDepthwiseConv2D**によるTFLite高速化。

**[3位](https://www.kaggle.com/competitions/asl-signs/discussion/406568)**

* **アプローチ:** **1D CNN**アンサンブル (6モデル) + Transformerアンサンブル (2モデル)。
* **アーキテクチャ:** 1D CNN (DepthwiseConv1D + MaxPool), Transformer (公開ノートブックベース)。
* **アルゴリズム:** シーケンス分類。
* **テクニック:** 入力: 手+唇+ポーズ+目 (モデル毎に組み合わせ変更)。**フレーム数32**に固定。正規化。Augmentation (Flip, Rotate/Shift/Scale, Point Shift, **部位組み合わせ**, CutMix)。TTA (Padding, Frame Drop)。Keras実装。

**[4位](https://www.kaggle.com/competitions/asl-signs/discussion/406673)**

* **アプローチ:** 1D CNNアンサンブル (FixLen/VariableLen)。CleanLabによるノイズ除去。
* **アーキテクチャ:** 1D CNN。FixLen: Poolingあり、96フレーム固定長入力。VariableLen: Poolingなし、可変長入力。Deep/Shallow Head。
* **アルゴリズム:** シーケンス分類。
* **テクニック:** 入力: 手+唇。正規化(眉間基準)。Augmentation (Frame Drop, Affine)。**CleanLab**によるノイズ除去。ArcMarginProduct風出力層。SWA。ラベルスムージング。

**[5位](https://www.kaggle.com/competitions/asl-signs/discussion/406491)**

* **アプローチ:** **Transformer**アンサンブル。特徴量エンジニアリング。
* **アーキテクチャ:** Transformer (3層, 埋込480)。
* **アルゴリズム:** シーケンス分類。
* **テクニック:** 入力: 手+鼻+目+唇。正規化。**ハンドクラフト特徴量** (モーション, 距離, 角度)。Augmentation (Flip, Affine, Frame Masking)。AWP。EMA。

**[6位](https://www.kaggle.com/competitions/asl-signs/discussion/406537)**

* **アプローチ:** MLPエンコーダー + **Transformer (DeBERTa)** アンサンブル (2モデル)。Mean Teacherと知識蒸留。
* **アーキテクチャ:** MLP (部位別+全体) + DeBERTa Transformer (2-3層)。
* **アルゴリズム:** シーケンス分類。
* **テクニック:** 入力: 手+唇+ポーズ+目など。正規化。フレーム削減(stride)。Augmentation (Flip, Rotate/Scale/Shift(歪みあり), Interpolate, **Manifold Mixup**, **顔CutMix**)。**Mean Teacher + 知識蒸留**。OUSM損失。Model Soup。NobucoでTFLite変換。

**[8位](https://www.kaggle.com/competitions/asl-signs/discussion/406411)**

* **アプローチ:** **Transformer**アンサンブル (3モデル)。
* **アーキテクチャ:** Transformer (2層)。FFNエンコーダー。
* **アルゴリズム:** シーケンス分類。
* **テクニック:** 入力: 手+唇+ポーズ(上半身)。正規化(MinMax)。差分特徴量(lag1-16)。角度特徴量。Augmentation (**Sequence Cutout**, Mirror, Rotate)。ラベルスムージング。PyTorch学習→ONNX経由TF変換→TFLite (fp16キャストトリック)。

**[9位](https://www.kaggle.com/competitions/asl-signs/discussion/406343)**

* **アプローチ:** **Transformer**アンサンブル (4モデル)。学習可能1D Conv Positional Encoding。
* **アーキテクチャ:** Transformer (1-2層)。学習可能1D Depthwise Conv Positional Encoding。
* **アルゴリズム:** シーケンス分類。
* **テクニック:** 入力: 手+唇+目+ポーズ。差分/距離/角度/加速度特徴量。Augmentation (Flip, Frame Drop, Affine, **Mixup(高確率)**, カット)。Cosine Scheduler。ArcFace Loss試行。NobucoでTFLite変換。

**[10位](https://www.kaggle.com/competitions/asl-signs/discussion/406434)**

* **アプローチ:** **Transformer** (Linear Embedding + Attention) + **STGCN** (手特徴量) のアンサンブル。
* **アーキテクチャ:** Transformer + STGCN (Spatio-Temporal Graph Convolutional Network)。
* **アルゴリズム:** シーケンス分類、グラフニューラルネットワーク。
* **テクニック:** 入力: 手+唇+目+ポーズ。正規化(顔基準)。**STGCN**で手特徴量抽出。Augmentation (**Mixup**, **CopyPaste**)。ArcFace風ラベル (participant+sign) 試行。
