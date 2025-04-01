---
tags:
  - Kaggle
startdate: 2023-02-24
enddate: 2023-05-02
---
# Google - Isolated Sign Language Recognition
[https://www.kaggle.com/competitions/asl-signs](https://www.kaggle.com/competitions/asl-signs)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、**孤立した（単語単位の）アメリカ手話（ASL）**のサインを示すビデオから抽出された**骨格ランドマークデータ**を用いて、どの**手話単語**が表現されているかを認識する機械学習モデルを開発することです。
* **背景:** 手話は聴覚障碍者の主要なコミュニケーション手段ですが、手話話者と非話者の間のコミュニケーションには障壁があります。手話認識技術は、リアルタイム翻訳などを通じてこの障壁を取り除き、インクルーシブな社会を実現するために重要です。このコンペでは、特に単語単位の手話認識に焦点を当てています。Googleは、聴覚障碍を持つ子供たちがASLを学ぶのを支援するアプリ「PopSign」の開発にこの成果を活用することを目指しています。
* **課題:**
    * **表現の多様性:** 同じ手話単語でも、表現者、速度、スタイルによって動きが異なります。
    * **データの欠損・ノイズ:** ランドマーク抽出の精度限界により、データには欠損（NaN）やノイズが含まれます。特に手の動きが速い場合や隠れている場合に顕著です。
    * **3次元情報の活用:** 提供されるデータは3次元座標ですが、その深さ情報（Z軸）を効果的に活用することが課題です。
    * **クラス数の多さ:** 認識対象となる手話単語は250種類あります。
    * **類似したサイン:** 動きが似ている異なるサインを区別する必要があります。
    * **シーケンス長の変化:** 手話の表現時間は単語によって、また同じ単語でも変動します。
    * **計算資源の制約:** TensorFlow Liteでの実装が想定されており、モデルサイズや推論速度に制約があります。

**データセットの形式 (Dataset Format)**

提供される主なデータは、手話ビデオからMediaPipeで抽出されたランドマークの時系列データです。

1.  **トレーニングデータ (`train_landmark_files/`):**
    * 各ファイルは `.parquet` 形式で、1つの手話サインのランドマークデータを含みます。
    * 各行が1フレームに相当し、以下の列が含まれます:
        * `frame`: フレーム番号。
        * `row_id`: フレームとランドマークタイプ・インデックスを組み合わせた識別子。
        * `type`: ランドマークの種類 (`face`, `left_hand`, `pose`, `right_hand`)。
        * `landmark_index`: 各タイプ内でのランドマークのインデックス。
        * `x`, `y`, `z`: 正規化されたランドマークの3次元座標。**これらがモデルの主要な入力特徴量となります。**
    * **`train.csv`:** 各 `.parquet` ファイルへのパス (`path`) と、対応する手話サイン (`sign`)、参加者ID (`participant_id`)、シーケンスID (`sequence_id`) を含みます。`sign` が**ターゲット変数**です。
2.  **`sign_to_prediction_index_map.json`:**
    * 手話サイン（文字列）を予測インデックス（整数）にマッピングする辞書ファイル。
3.  **テストデータ:**
    * テストデータは隠されており、提出されたモデル（TFLite形式）がサーバー上で実行され評価されます。フォーマットはトレーニングデータと同様と想定されます。
4.  **`sample_submission.csv`:**
    * 提出フォーマットのサンプル。`row_id` (`sequence_id` - `participant_id`) と `sign` (予測された手話サイン) の列を持ちます。

**評価指標 (Evaluation Metric)**

* **指標:** **正規化平均レーベンシュタイン距離 (Normalized Mean Levenshtein Distance)**
* **計算方法:**
    * このコンペでは、各シーケンスに対して単一のサインを予測します。
    * 評価は実質的に **Accuracy (正答率)** で行われます。予測されたサインと正解のサインが完全に一致すればレーベンシュタイン距離は0、異なれば1となります。
    * 全てのテストサンプルに対するレーベンシュタイン距離の平均が最終スコアとなります。
* **意味:** モデルが各手話サインをどれだけ正しく分類できているかを測ります。スコアは **0から1** の範囲を取り、**0に近い**ほど性能が良いことを示します（完全一致ならスコアは0）。

要約すると、このコンペティションは、ビデオから抽出された手の動きや顔の表情などの3Dランドマーク時系列データを用いて、250種類の孤立したASLサインを分類するタスクです。データはランドマーク座標のシーケンスで構成され、性能は正答率（正規化平均レーベンシュタイン距離、0に近いほど良い）によって評価されます。最終的なモデルはTFLite形式で提出する必要があります。

---

**全体的な傾向**

ASLサイン認識タスクでは、ランドマークの時系列データを扱うため、**Transformer** や **1D CNN**、またはその両方を組み合わせたアーキテクチャが上位解法で広く採用されました。特に、フレーム間の相関が強いという仮説から1D CNNを重視する解法や、Transformerの強力な系列モデリング能力を活用する解法が見られました。一部では、音声認識や画像認識のアプローチを応用し、ランドマークデータを**スペクトログラム風の2D画像**として扱い、2D CNN (EfficientNetなど) を適用する試みも成功しています。

**データ前処理**が非常に重要であり、使用するランドマークの選択（手、唇、ポーズ、目など）、**正規化**の方法（鼻基準、眉間基準、肩/腰/唇/目基準、フレームごと、シーケンス全体、Min-Max、Mean/Std）、**欠損値(NaN)の処理**（ゼロ埋め、補間、削除）、**特徴エンジニアリング**（差分(速度/加速度)、点間距離、角度）などが工夫されました。左右の手の扱い（片手のみ使用、利き手への反転統一など）もポイントでした。

**Augmentation**もスコア向上に不可欠で、**アフィン変換**（回転、シフト、スケール、せん断）、**水平フリップ**、時間軸方向の操作（**リサンプリング**、**時間的マスキング/Cutout**、**フレームドロップ**、開始/終了部分のカット）、**パーツ置換**（同じラベルの別サンプルの手や唇を組み合わせる）、**Mixup**（キーポイント空間、埋め込み空間）などが効果的でした。

学習においては、**強力な正則化**（高ドロップアウト率、Drop Path (Stochastic Depth)、ラベルスムージング、**AWP (Adversarial Weight Perturbation)**、OUSM (上位k個のロスが大きいサンプルを除外)）が過学習を防ぐために重要でした。**知識蒸留**や**Mean Teacher**、**Model Soup**といったテクニックも用いられました。

計算資源の制約から、**モデルの軽量化**と**推論速度の最適化**が必須であり、**TensorFlow Lite (TFLite)** への変換が前提とされました。ONNX経由での変換や、TFLiteでの性能を考慮したKeras/TensorFlowでのモデル再実装、Depthwise Convolutionの最適化などが試みられました。最終提出は、複数シードや異なる設定のモデルによる**アンサンブル**が一般的でした。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/asl-signs/discussion/406684)**

* **アプローチ:** 1D CNNとTransformerの混合モデル。TensorFlow + TPU利用。
* **アーキテクチャ:** Conv1Dブロック x3 + Transformerブロック x1 を2回繰り返す構造。1D CNNはDepthwise Conv + Causal Padding。TransformerはBatchNorm + Swish。
* **アルゴリズム:** CCE Loss (Label Smoothingあり/なし)。RAdam + Lookahead Optimizer。Cosine Decayスケジュール。
* **テクニック:**
    * **データ:** 鼻基準正規化。差分特徴量(lag1, lag2)。左手/右手/目/鼻/唇のランドマーク使用。
    * **マスキング:** `tf.keras.layers.Masking` を使用し、可変長入力に対応。因果的パディング。BNやGAPでのマスク適用に注意。
    * **正則化:** Drop Path (p=0.2), 高Dropout (p=0.8, GAP後), AWP (λ=0.2, 15 epoch以降)。
    * **Augmentation:** 時間的(リサンプリング, マスキング), 空間的(フリップ, アフィン変換, Cutout)。
    * **アンサンブル:** 4つの異なるシードで学習したモデルの平均。

**[2位](https://www.kaggle.com/competitions/asl-signs/discussion/406306)**

* **アプローチ:** 2D CNN (EfficientNet-B0) を主軸に、Transformer (BERT, DeBERTa) を補助的に利用したアンサンブル。ランドマークデータをスペクトログラム風画像として扱う。
* **アーキテクチャ:** EfficientNet-B0 (入力160x80x3)。BERT, DeBERTa (Transformer用前処理)。
* **アルゴリズム:** 重み付きCrossEntropyLoss。OneCycle Scheduler。Ranger Optimizer。Label Smoothing。
* **テクニック:**
    * **データ:** 唇(18), ポーズ(20), 手(42)の計80点使用。標準正規化後ゼロ埋め。時間軸を160に補間。Transformer用は別前処理 (61点、手特徴量、距離、角度)。
    * **Augmentation:** アフィン変換 (全体+部位別)、時間的補間、フリップ、指回転 (カスタム)、Mixup、パーツ置換 (同クラス)、時間/点マスキング。
    * **学習:** CNNは8 Fold中1 Foldで学習。Transformerは全データ。Optunaでハイパーパラメータチューニング。
    * **TFLite:** Kerasでモデル書き換え、特にDepthwiseConv2Dをハードコーディングで高速化。`tf.Module` で集約。
    * **アンサンブル:** 3モデル (EffNetB0 Fold0 + BERT Full + DeBERTa Full) の重み付き平均 (Softmaxなし)。

**[3位](https://www.kaggle.com/competitions/asl-signs/discussion/406568)**

* **アプローチ:** 1D CNN (DepthwiseConv1D) とTransformer (公開カーネルベース) のアンサンブル。
* **アーキテクチャ:** カスタム多層Conv1Dモデル (DepthwiseConv1D, MaxPool1D, GlobalAvgPool1D)。Transformer (公開カーネルベース、ユニット数削減、別Embedding方式も試行)。
* **アルゴリズム:** (損失、Optimizer、Schedulerは不明だが、標準的なものと推測)。
* **テクニック:**
    * **データ:** 特定点(顔, 肩, 腰?)基準正規化。深度(Z)未使用。特徴選択 (両手/片手+唇/目/ポーズ上部などモデル毎に変更)。片手化 (利き手でない場合ミラーリング)。32フレーム入力 (一部96フレーム)。
    * **Augmentation:** 大域的アフィン変換、点ごとシフト、パーツ置換 (同ラベルの別サンプルから手/唇)、CutMix (異クラスパーツ混合、ラベル重み付け)。
    * **TTA:** 短いシーケンスへのパディング、手のないフレームの確率的除去。
    * **Transformer:** モーション埋め込み ((dx, dy, dist)シーケンス) とXYZ埋め込みを結合。
    * **アンサンブル:** 6つのConv1Dモデル + 2つのTransformerモデル。

**[4位](https://www.kaggle.com/competitions/asl-signs/discussion/406673)**

* **アプローチ:** 2種類の1D CNN (固定長/可変長) のアンサンブル。手と唇で別々のBackbone。CleanLabによるノイズ除去。
* **アーキテクチャ:**
    * **Backbone (共通):** 手用と唇用で別々に1D CNN。
    * **1DCNN-FixLen:** Conv+MaxPoolで時間次元削減 -> GlobalMaxPooling。
    * **1DCNN-VariableLen:** PoolingなしConv -> マスク付きGlobalMaxPooling。
    * **Head (共通):** 手と唇の特徴量を加算 -> FC層 (Deep版は重み共有) -> 分類層。
* **アルゴリズム:** Label Smoothing Loss (ε=0.5)。AdamW。ArcMarginProduct (m=0、実質Norm正規化のみ)。SWA (15 epoch以降)。
* **テクニック:**
    * **データ:** XY座標のみ。眉間基準正規化。片手化 (フレーム数の多い方を右手として反転)。右手(21)と唇(40)の計122次元。手がないフレームは削除。
    * **入力:** FixLenは96に補間。VariableLenはバッチ内最大長に0パディング。
    * **正則化:** Stochastic Depth (0.1 or 0.5)。
    * **Augmentation:** フレームランダムドロップ (p=0.3)、手のアフィン変換、シーケンス長変更 (FixLen用、64-128)。
    * **CleanLab:** 21-Foldで事後確率計算しノイズ除去（約5000サンプル）。アンサンブルの一部モデルに適用。
    * **アンサンブル:** 6モデル (FixLen x4 + VariableLen x2、CleanLab有無、Head深さ、Seed違い) の平均。

**[5位](https://www.kaggle.com/competitions/asl-signs/discussion/406491)**

* **アプローチ:** Transformerモデル (公開カーネルベース)。特徴エンジニアリングとAugmentation。
* **アーキテクチャ:** Transformer (公開カーネルベース、3層、埋め込みサイズ480)。
* **アルゴリズム:** (損失、Optimizer、Scheduler不明)。
* **テクニック:**
    * **データ:** 約106点使用。単一シーケンスの平均/標準偏差で正規化。
    * **特徴量:** 点間距離 (手/鼻/目など)。
    * **Augmentation:** フリップ、連結など。
    * **正則化:** AWP, Random Mask of frames, EMA。
    * **アンサンブル:** (明記されていないが、複数モデルの可能性あり)。

**[6位](https://www.kaggle.com/competitions/asl-signs/discussion/406537)**

* **アプローチ:** MLPエンコーダ付きTransformer (DeBERTa風)。Mean Teacher + 知識蒸留。Model Soup。
* **アーキテクチャ:** ランドマーク埋め込み -> 特徴抽出MLP (全体+部位別) -> Transformer (DeBERTa風カスタムAttention、複数層)。
* **アルゴリズム:** Cross Entropy Loss (Label Smoothing ε=0.3)。(Optimizer不明、Linear Schedule)。OUSM (Top3ロス除外)。
* **テクニック:**
    * **データ:** フレームストライドでシーケンス長調整 (max_len=25, 80)。全体で平均0、標準偏差1に正規化。部位MLP前に追加センタリング。NaNは0埋め。指がないフレームは削除。
    * **埋め込み:** 1D Conv x2 + ランドマークID/タイプ埋め込み。
    * **Augmentation:** フリップ、全体回転、リサイズ(+歪み)、時間クロップ、補間、Manifold Mixup (スケジュール適用)、手コピー(片手欠損時)、顔CutMix (同ラベル別サンプル)。
    * **学習:** Mean Teacher + 知識蒸留。Model Soup (最終10 epoch)。
    * **TFLite:** Nobucoライブラリ使用。
    * **アンサンブル:** 2つの蒸留済みモデル (異なるmax_len) の平均。

**[8位](https://www.kaggle.com/competitions/asl-signs/discussion/406411)**

* **アプローチ:** Transformer (2層)。シーケンスカットアウトAugmentationが鍵。特徴エンジニアリング。
* **アーキテクチャ:** FFNエンコーダ -> Transformer (2層、384/512隠れ層)。
* **アルゴリズム:** (損失不明、Label Smoothing 0.1)。(Optimizer不明、Cosine Schedule)。
* **テクニック:**
    * **データ:** 手、唇、ポーズ(上半身)使用。一部モデルはポーズ全体+唇サブセット。最大長96に線形補間。部位ごとMin-Max正規化 (一部Mean/Std)。NaNは前方/後方補間。
    * **特徴量:** 時間差分特徴 (shift=[1,2,3,4,6,8,12,16])、角度特徴 (口角、手、腕)、点間距離 (効果薄)。
    * **Augmentation:** シーケンスカットアウト (部位ごと、p=0.4、ランダムな5区間x15%長をNaN化)、ミラーフリップ、ランダム回転。
    * **TFLite:** PyTorch->ONNX->TF経由で変換。`model.half().float()` で高速化。
    * **アンサンブル:** 3つのTransformerモデルの平均。

**[9位](https://www.kaggle.com/competitions/asl-signs/discussion/406343)**

* **アプローチ:** Transformer (公開カーネルベース)。特徴エンジニアリング。Mixup Augmentation。
* **アーキテクチャ:** Transformer (公開カーネルベース、1-2層)。位置エンコーディング (絶対学習可能 or 畳み込み)。
* **アルゴリズム:** (損失不明)。(Optimizer/Scheduler不明、ただしMixupにより長期学習+Cosine Schedulerが可能に)。
* **テクニック:**
    * **データ:** (ランドマーク選択は不明)。
    * **特徴量:** モーション、距離(手/唇/目)、絶対手座標、加速度。
    * **Augmentation:** フリップ、ランダムFPSドロップ (::2)、3Dアフィン変換 (背骨周り回転)、Mixup (キーポイント空間/埋め込み空間、高確率p=~0.7)、時間カット、補間。
    * **TFLite:** Nobucoライブラリ使用。
    * **アンサンブル:** 4モデル (1層x3 + 2層x1、異なる位置エンコーディング) の平均。

**[10位](https://www.kaggle.com/competitions/asl-signs/discussion/406434)**

* **アプローチ:** Transformer + STGCN (手)。パーツごとに異なる埋め込み/特徴抽出。
* **アーキテクチャ:** 線形埋め込み (ポーズ、唇、目、手) + STGCN (手) -> 特徴結合 -> Transformer -> 分類ヘッド。
* **アルゴリズム:** (損失不明)。(Optimizer/Scheduler不明)。
* **テクニック:**
    * **データ:** 顔基準正規化。利き手でない場合フリップ。手がないフレームを削除。時間次元を16にリサイズ。
    * **Augmentation:** Mixup, Copy Paste (Cutout風: 別サンプルの区間を貼り付け)。
    * **アンサンブル:** (明記は無いが複数モデルを示唆)。


