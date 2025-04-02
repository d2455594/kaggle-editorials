---
tags:
  - Kaggle
startdate: 2023-05-08
enddate: 2023-05-25
---
# BirdCLEF 2023 - Bird Sound Classification
[https://www.kaggle.com/competitions/birdclef-2023](https://www.kaggle.com/competitions/birdclef-2023)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、野外で録音された連続的な音声データ（サウンドスケープ）から、**どの種類の鳥が鳴いているかを特定する**機械学習モデルを開発することです。生物多様性のモニタリングと保全を支援するための音響認識技術の向上を目指します。
* **背景:** BirdCLEFは、鳥類の音響データを活用した分類・検出チャレンジであり、過去数年にわたり開催されています。Cornell Lab of OrnithologyやLifeCLEFなどが関与し、Xeno-cantoのような市民科学プロジェクトによって収集された膨大な鳥の鳴き声データが活用されます。音響モニタリングは、広範囲の生物多様性を低コストで継続的に調査する有望な手段です。
* **課題:**
    * **音響環境の複雑性:** 対象となる鳥の声が小さかったり、遠かったりする一方で、他の鳥の声、虫、風、雨、人間の活動音など、様々な背景音やノイズが混在している。
    * **種間の類似性と種内の多様性:** 鳴き声が似ている異なる種類の鳥を区別する必要がある一方、同じ種類の鳥でも地域や状況によって鳴き声が変化する。
    * **データセットの不均衡:** 頻繁に記録される一般的な種と、めったに記録されない希少な種が存在する。
    * **ラベルの曖昧さ (Weak Labels):** 訓練データには主たる種 (primary_label) に加え、同時に鳴いている可能性のある他の種 (secondary_labels) が付与されているが、これが完全ではない場合がある。また、鳥が全く鳴いていない区間 (nocall) の扱いも重要。
    * **ドメインシフト:** 訓練データ（主にXeno-cantoから収集された、比較的クリアな個別の録音が多い）と、テストデータ（フィールドで連続録音されたサウンドスケープ）の間には、音響特性（例: 残響、背景ノイズレベル）の違いがある。
    * **計算リソース制限:** 推論はKaggle Notebook環境で実行され、制限時間（2時間）内に完了する必要があるため、モデルの効率性も求められる。

**データセットの形式 (Dataset Format)**

提供される主なデータは、鳥の鳴き声を含む音声ファイルと関連するメタデータです。

1.  **トレーニングデータ:**
    * `train_metadata.csv`: 訓練音声ファイルに関する情報。
        * `filename`: 音声ファイル名 (例: `species_code/XC123456.ogg`)。
        * `primary_label`: その録音の主たる鳥の種類を示すコード (例: `amered`)。
        * `secondary_labels`: 同時に録音されている可能性のある他の鳥の種類のリスト (例: `['redhea', 'grhowl']`)。空の場合もある。
        * `type`: 鳴き声の種類 (例: `['call', 'song']`)。
        * `latitude`, `longitude`: 録音場所の緯度経度。
        * `rating`: Xeno-cantoでの録音品質評価。
        * `author`: 録音者。
        * `duration`: 音声ファイルの長さ（秒）。
    * `train_audio/`: Ogg Vorbis形式の音声ファイル。サブディレクトリは `primary_label` に対応。
    * **外部データ:** Xeno-canto全体、過去のBirdCLEFコンペデータ、他の鳥類音声データセット (例: Zenodo上のデータ)、ノイズデータセット (例: ESC-50, ff1010bird_nocall) の利用が広く行われました。
2.  **テストデータ:**
    * `test_soundscapes/`: テスト用の長時間（通常数分）のサウンドスケープ音声ファイル (Ogg Vorbis形式)。ファイル名は `[soundscape_id]_[chunk_id].ogg` という形式の場合があります (過去の形式であり、2023年は単一ファイルの場合も)。
    * `test.csv`: 提出ファイルを作成するためのフォーマット定義。各行が特定のサウンドスケープの特定の時間区間（5秒ごと）における特定の鳥の種類の予測に対応します。
        * `row_id`: `[soundscape_id]_[bird_species]_[seconds]` の形式の識別子 (例: `soundscape_123_amered_5`)。 `seconds` は時間区間の終了秒を示します (5, 10, 15, ...)。
        * `target`: この行に対応する鳥が存在するかどうかの真偽値 (提出時には予測確率またはブール値を記述するが、テストデータには含まれない)。
    * `soundscape_labels.csv`: 各サウンドスケープファイルに含まれる可能性のある鳥の種類のリスト（ただし、これは全リストであり、実際にそのファイルに全ての種が含まれるわけではない）。

**評価指標 (Evaluation Metric)**

* **指標:** **パディングされたクラスごとの平均適合率 (Padded Competition Metric)** (過去には row-wise micro averaged F1 score とも説明されたが、cmAPに近い指標として解釈されている)
* **計算方法:**
    * テストデータは5秒間のチャンクに分割されて評価されます。
    * 各 `row_id` (各サウンドスケープ、各鳥、各5秒区間) に対して、モデルがその鳥が存在すると予測したかどうか（または確率）を提出します。
    * 評価システムは、提出された予測と隠された正解ラベルを比較し、各鳥の種類ごとに適合率 (Precision) を計算します。
    * 全ての鳥の種類にわたって、これらの適合率が平均化されます (Micro Average または Macro Average に近い形)。
    * 提出時に各 `row_id` に対して予測がない場合や、予測数が一定数に満たない場合に、ダミーの予測で「パディング」される可能性があります (これが "Padded" の意味するところと考えられます)。
    * 以前のコンペと異なり、予測確率に対する**閾値の設定が不要**になりました。モデルが出力するスコアやランキングに基づいて評価されます。
* **意味:** 各5秒間の区間において、モデルがリストアップした鳥の種類が、実際にその区間に存在したかどうかを、偽陽性を抑えつつ評価します。特に、多くの種類の中から正しく存在する種を上位にランク付けする能力が重要になります。スコアは**高い**ほど良い性能を示します。

要約すると、このコンペティションは、長時間のサウンドスケープ音声から5秒ごとに存在する鳥の種類を特定するマルチラベル分類タスクです。データは音声ファイルとメタデータで構成され、性能は閾値不要の適合率ベースの指標（高いほど良い）で評価されます。推論時間制限も考慮すべき重要な要素です。

---

**全体的な傾向**

BirdCLEF 2023では、過去のコンペと同様に**サウンドイベント検出 (Sound Event Detection, SED)** のアプローチが主流でした。これは、CNNベースのエンコーダー（EfficientNet系, NFNet系, ResNe(X)t系, ConvNeXt系などが人気）で音声スペクトログラムから特徴量を抽出し、Attention Pooling層（時間軸または周波数軸）を通して時間方向の情報を集約し、最終的に分類ヘッドで各鳥種の存在確率を出力する構成です。

データ面では、提供された訓練データに加え、**Xeno-canto全体や過去のBirdCLEFコンペデータなど、利用可能な外部データを大規模に活用**することが一般的でした。特に、不足している種や音質のバリエーションを補うために重要でした。また、訓練データとテストデータの音響特性の違い（ドメインシフト）に対処するため、**多様なデータ拡張 (Data Augmentation)** が不可欠でした。Mixup、背景ノイズ付加、音量・ピッチ・速度の変更、スペクトログラムマスキング (SpecAugment) に加え、特に**残響 (Reverb) 付加**が有効な手法として挙げられました。

ラベル戦略としては、primary labelのみを使う方法と、secondary labelsをソフトラベル（例: 0.3-0.5程度の確率）や重み付けとして活用する方法がありました。no-call（鳥がいない）データを学習に含めることも一般的でした。

モデル学習では、大規模データでの**事前学習 (Pretraining)** と、ターゲットデータでの**ファインチューニング (Finetuning)** の2段階学習が多く見られました。また、強力な既成モデル（例: Google Bird Vocalization Classifier）からの**知識蒸留 (Knowledge Distillation)** も有効な戦略でした。

推論においては、**モデルアンサンブル**がスコア向上の鍵となりました。異なるバックボーン、異なる学習設定、異なるFoldで学習した複数のモデルの予測を平均化（重み付き、ランクベース、温度付きなど）することで、頑健性と精度を高めていました。Kaggle Notebookの**推論時間制限（2時間）**をクリアするため、ONNX、OpenVINO、TorchScript (JIT) などを用いた**モデル最適化**や、ThreadPoolExecutorなどによる**並列処理**が広く利用されました。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/birdclef-2023/discussion/412808)**

* **アプローチ:** SEDモデル3種のアンサンブル。データ品質と量の確保が鍵。
* **アーキテクチャ:** SED (Backbones: eca_nfnet_l0, convnext_small_fb_in22k_ft_in1k_384, convnextv2_tiny_fcmae_ft_in22k_in1k_384)。
* **アルゴリズム:** SED。
* **テクニック:** XC APIのバグ修正によるデータ追加。データクリーニング（手動/自動重複削除）。2段階学習（事前学習 on 822種 + ファインチューニング on 264種）。クラスサンプリング重み付け。Augmentation (Mixup, 背景ノイズ(Zenodo nocall), RandomFiltering, SpecAug)。推論時: 温度付き平均、Attention確率とMax確率の加重平均。ONNX使用。

**[2位](https://www.kaggle.com/competitions/birdclef-2023/discussion/412707)**

* **アプローチ:** SEDモデル (3種) + 2021年2位解法CNNモデル (4種) の計7モデルアンサンブル。
* **アーキテクチャ:** SED (Backbones: tf_efficientnetv2_s_in21k, seresnext26t_32x4d, tf_efficientnet_b3_ns)。CNN (Backbones: tf_efficientnetv2_s_in21k, resnet34d, tf_efficientnet_b3_ns, tf_efficientnet_b0_ns)。
* **アルゴリズム:** SED、CNN。
* **テクニック:** 複数データセット使用 (XC含む)。多様なAugmentation (ノイズ各種、背景ノイズ、ピッチ/時間シフト、マスキング、Mixup(波形/スペクトログラム)、周波数カット、Self Mixup)。重み付きDataLoader。2段階学習+損失関数切り替え(CE→BCE)。一部モデルは30秒クリップでファインチューニング。推論時チャンク処理、TTA。**OpenVINO**による高速化。アンサンブルは重み付き平均とランク平均。

**[3位](https://www.kaggle.com/competitions/birdclef-2023/discussion/414102)**

* **アプローチ:** 周波数帯域へのAttentionを持つ修正SEDモデルのアンサンブル。残響Augmentation。
* **アーキテクチャ:** SED (Backbones: tf_efficientnet_b0_ns, tf_efficientnetv2_s_in21k)。分類ヘッドで周波数帯域Attentionを使用（時間軸Attentionの代わりに）。
* **アルゴリズム:** 修正SED。
* **テクニック:** 複数データセット使用 (XC, 過去コンペノイズデータなど)。多様なAugmentation (ランダムチャンク選択(重み付けあり), フィルタ, Mixup, ノイズ, ピッチ/時間シフト, **Reverb(Impulse Response畳み込み)**, スペクトログラム補間/色Jitter)。全クラス(2021+2023+nocall)で学習。secondary labelはソフトターゲット(0.3)。TorchScriptと入力事前計算で推論高速化。推論時タイマー設定。

**[4位](https://www.kaggle.com/competitions/birdclef-2023/discussion/412808)**

* **アプローチ:** SEDモデル (eca_nfnet_l0) 4種のアンサンブル。**Google Bird Vocalization Classifierからの知識蒸留**。
* **アーキテクチャ:** SED (Backbone: eca_nfnet_l0)。
* **アルゴリズム:** SED、知識蒸留。
* **テクニック:** 複数データセット使用 (XC, ff1010bird_nocall, Zenodo, ESC50など)。知識蒸留 (教師モデルの出力をソフトターゲットとしてKLDivLossで学習)。損失関数はBCE + KD(KLDivLoss)。Augmentation (Mixup(p=1.0), ノイズ, 背景ノイズ, LowPassFilter, PitchShift)。20秒クリップで学習。PyTorch JITとThreadPoolExecutorによる推論高速化。

**[5位](https://www.kaggle.com/competitions/birdclef-2023/discussion/412903)**

* **アプローチ:** SEDモデル (4バックボーン) のアンサンブル (15 Fold/モデル)。
* **アーキテクチャ:** SED (Backbones: tf_efficientnet_b1/b2/b3_ns, tf_efficientnetv2_s_in21k)。
* **アルゴリズム:** SED。
* **テクニック:** 複数データセット使用 (XC, ESC50ノイズ, 2021 nocall)。2段階学習 (事前学習+ファインチューニング)。Augmentation (Mixup, ノイズ各種, スペクトログラムマスキング)。primary/secondaryラベル使用 (ratingに基づく重み付け)。一部モデルは疑似ラベルでファインチューニング。全データでのFull-fitモデルも使用。ONNXとThreadPoolExecutorによる推論高速化。

**[6位](https://www.kaggle.com/competitions/birdclef-2023/discussion/412708)**

* **アプローチ:** 2021年2位解法CNNモデルと**BirdNET埋め込み**を結合したモデルのアンサンブル。
* **アーキテクチャ:** 2021年2位解法CNN (Backbones: eca_nfnet_l1, seresnext26t_32x4d) の出力とBirdNET V2.2の埋め込みベクトルを結合し、線形層で分類。
* **アルゴリズム:** CNN + 特徴量結合。
* **テクニック:** 2段階学習 (事前学習+ファインチューニング)。BirdNET部分は固定。30秒クリップで学習（BirdNET埋め込みは平均化）。Augmentation (Shift, Mixup, ノイズ, フィルタ, Random Power, スペクトログラムマスキング)。secondary labelはソフトターゲット(0.4/0.5)、ラベルスムージング。オーバーサンプリング（クラス数、複数ラベル有無、残響付加）。ONNXとThreadPoolExecutorによる推論高速化。

**[7位](https://www.kaggle.com/competitions/birdclef-2023/discussion/412922)**

* **アプローチ:** CNNモデル (2バックボーン) のアンサンブル (19モデル)。独自Augmentation `sumix`。知識蒸留。
* **アーキテクチャ:** CNN (Backbones: torchvision efficientnet_b2, timm rexnet150)。一部モデルにAttention Head追加。
* **アルゴリズム:** CNN、知識蒸留。
* **テクニック:** 独自Augmentation **sumix/sumup** (波形を加算しラベルも混合、音量も調整可能)。Mixup, CutMixも併用。知識蒸留 (損失関数に双方向KLダイバージェンス追加)。Attention Head追加 (MultiHeadAttentionClassifier)。OpenVINOによる推論高速化。混合精度学習 (bfloat16)。EMA。

**[8位](https://www.kaggle.com/competitions/birdclef-2023/discussion/412871)**

* **アプローチ:** SEDモデル (eca_nfnet_l0) のアンサンブル (6モデル)。マルチモーダルなデータ拡張。
* **アーキテクチャ:** SED (Backbone: eca_nfnet_l0)。
* **アルゴリズム:** SED。
* **テクニック:** **YOLOv8**によるSound Object Detectionを用いたWeak-to-Strong前処理。**地理情報**に基づくGeometric Mixup。**協調フィルタリング風**Resonance Mixup。Inner Mixup。段階的ダウンスケーリング学習 (Global→Local)。複数データセット使用 (ZenodoデータはYOLOv8で使用)。

**[9位](https://www.kaggle.com/competitions/birdclef-2023/discussion/412794)**

* **アプローチ:** CNNモデル (3バックボーン) のアンサンブル (7モデル)。複数ラウンド学習。
* **アーキテクチャ:** CNN (Backbones: efficientnet_b0, eca_nfnet_l0, convnext_tiny)。
* **アルゴリズム:** CNN。
* **テクニック:** 複数ラウンド学習 (データセット、学習タイプ(蒸留/疑似ラベル)を変更)。AWP, SWA, Attention Head。Augmentation (HFlip, Cutout, ノイズ各種, Random Volume, Pixel Dropout)。疑似ラベル活用 (secondary labelとして付与)。OpenVINOと並列DataLoaderによる推論高速化。後処理 (高信頼度予測の増幅、試行のみ)。

**[10位](https://www.kaggle.com/competitions/birdclef-2023/discussion/412713)**

* **アプローチ:** SEDモデル (6バックボーン) のアンサンブル (6-12 Fold/モデル？)。**段階的スペクトログラム事前学習**。
* **アーキテクチャ:** SED (Backbones: efficientnetv2_m, eca_nfnet_l0, efficientnet_v0, mnasnet_100, spnasnet_100, mobilenetv2_100)。
* **アルゴリズム:** SED。
* **テクニック:** 段階的スペクトログラム事前学習 (異なるMelSpecパラメータセットで学習したモデルを初期値として再利用)。ソフトラベル (secondary=0.3)。Augmentation (ノイズ, 背景ノイズ, CutMix, Mixup)。ONNXとThreadPoolExecutorによる推論高速化。

