---
tags:
  - Kaggle
startdate: 2023-05-08
enddate: 2023-05-25
---
# BirdCLEF 2023
[https://www.kaggle.com/competitions/birdclef-2023](https://www.kaggle.com/competitions/birdclef-2023)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、**連続的な音響データ（サウンドスケープ）**から、特定の時間帯にどの**野鳥の種類**が鳴いているかを特定する機械学習モデルを開発することです。
* **背景:** 鳥類のモニタリングは生物多様性の保全に不可欠ですが、広範囲かつ継続的な調査には多大な労力が必要です。音響モニタリングは、設置したマイクで自動的に録音するため効率的ですが、録音された膨大なデータから鳥の種類を正確に識別するには高度な技術が求められます。このコンペは、AI技術を用いて音響データからの鳥類識別精度を向上させ、生物多様性研究に貢献することを目指しています。
* **課題:**
    * **多様な環境音:** 録音データには、鳥の鳴き声以外にも風、雨、虫、他の動物の声、人間の活動音など様々なノイズが含まれます。
    * **重複する鳴き声:** 複数の鳥が同時に鳴いている場合や、同じ鳥が異なる鳴き方をする場合があります。
    * **クラス数の多さ:** 対象となる鳥の種類が非常に多い (264種)。
    * **データの不均衡:** 種類によって録音データの量に大きな偏りがあります。
    * **ラベルの質:** 提供されるラベル（特にXeno-Cantoから収集されたデータ）には、ノイズや不確実性が含まれる可能性があります (Weak Label)。
    * **ドメインシフト:** トレーニングデータ（主に個別の鳥の録音）とテストデータ（連続的なサウンドスケープ）の間で音響特性が異なる可能性があります（例: 距離、反響）。

**データセットの形式 (Dataset Format)**

提供されるデータは、主に野鳥の鳴き声を含むオーディオファイルとそのメタデータ、およびテスト用のサウンドスケープファイルです。

1.  **トレーニングデータ (`train_audio/`):**
    * 個別の野鳥の鳴き声を含む短いオーディオファイル（主に `.ogg` 形式）。
    * **`train_metadata.csv`:** 各オーディオファイルに関するメタデータ。
        * `primary_label`: 主に録音されている鳥の種類（ターゲット）。
        * `secondary_labels`: 同時に録音されている可能性のある他の鳥の種類。
        * `latitude`, `longitude`: 録音場所の緯度経度。
        * `date`, `time`: 録音日時。
        * `filename`: オーディオファイル名。
        * `rating`: 録音品質の評価。
        * `common_name`: 鳥の一般名。
        * `type`: 鳴き声の種類（歌、コールなど）。
    * 過去のBirdCLEFコンペティションデータや、外部データセット（Xeno-Canto、Zenodoなど）も利用が推奨・許可されています。多くの参加者がこれらのデータを活用しています。
2.  **テストデータ (`test_soundscapes/`):**
    * コンペティションの評価対象となる、より長い連続録音ファイル（サウンドスケープ）。通常、数分間の録音。
    * ファイル名は `soundscape_{ID}.ogg` の形式。
    * これらのファイルに対して、5秒ごとの区間でどの鳥が存在するかを予測します。
    * **`test.csv`:** テストファイルとその区間情報を提供します。
3.  **`sample_submission.csv`:**
    * 提出フォーマットのサンプル。`row_id` (テストファイル名と終了時間 `seconds` を組み合わせた識別子) と、各鳥類コード (例: `asbfly`) の列を持ちます。各行には、対応する `row_id` の区間（過去5秒間）にその鳥が存在するかどうかのブール値 (True/False) を予測します。
4.  **`eBird_Taxonomy_v2021.csv`:**
    * eBirdの分類法に基づく鳥類のリスト。`primary_label` と `common_name` などのマッピング情報が含まれます。

**評価指標 (Evaluation Metric)**

* **指標:** **パディング付きクラス認識平均適合率 (padded C-MAP: Class-Aware Mean Average Precision)**
* **計算方法:**
    1.  提出された各行 (soundscape\_file + seconds) について、`True` と予測された鳥のリストを作成します。
    2.  各鳥類クラスについて、Average Precision (AP) を計算します。APは適合率-再現率曲線の下の面積で、予測のランキング品質を評価します。
    3.  全クラスのAPを平均して Mean Average Precision (MAP) を算出します。
    4.  **パディング:** 提出ファイルには、実際の予測行に加え、各クラスごとに5つのダミー行（評価には影響しない）が追加されます。これは、クラスごとのサンプル数が少ない場合でもAP計算を安定させるための措置です。
* **意味:** モデルが各鳥類をどれだけ正確に、かつランキング上位で検出できているかを総合的に評価します。MAPは、偽陽性（鳥がいないのにいると予測）と偽陰性（鳥がいるのに見逃す）の両方を考慮し、特に予測の信頼度（ランキング）を重視します。2021年までのコンペと異なり、予測確率に対する**閾値の調整が不要**になった点が特徴です。スコアは **0から1** の範囲を取り、**高い**ほど性能が良いことを示します。

要約すると、このコンペティションは、サウンドスケープ音声データから複数の野鳥の種類を5秒ごとに検出するマルチラベル分類タスクです。データはオーディオファイルとメタデータで構成され、性能はパディング付きのクラス認識平均適合率 (padded C-MAP、高いほど良い) によって評価されます。

---

**全体的な傾向**

BirdCLEF 2023では、音声データから鳥の種類を特定するタスクにおいて、**Sound Event Detection (SED)** モデルが主流となりました。特に、画像分類で実績のあるCNNアーキテクチャ（**EfficientNet系, ConvNeXt系, NFNet系, ResNe(X)t系**など）をバックボーンとし、クラス分類用のヘッド（多くはAttention Poolingを含む）を組み合わせる構成が広く採用されました。

**データ**の量が性能に大きく影響するため、提供されたデータに加え、**過去のBirdCLEFコンペデータ**や**Xeno-Canto**から追加データを収集・利用することが一般的でした。Xeno-Canto APIのバグを修正してより多くのデータを取得したチームもありました。また、**ノイズデータ**（Zenodoのno-call区間、ESC50など）を背景音として利用するAugmentationも有効でした。

**Augmentation**は極めて重要で、**Mixup**（波形レベル、スペクトログラムレベル、カスタムMixup）が特に効果的でした。その他、ノイズ付加（Gaussian, Pink, Background）、ピッチシフト、タイムストレッチ、SpecAugment（時間/周波数マスキング）、フィルタリングなどが用いられました。ドメインシフト（トレーニングデータとテストデータの違い）対策として**Reverb（残響）**を加えるAugmentationも試されました。

学習においては、**複数段階学習**（大規模データで事前学習し、ターゲットデータでファインチューニング）や、**知識蒸留**（強力な外部モデルや事前学習済みモデルからの知識を利用）が有効な戦略でした。損失関数は**BCEWithLogitsLoss**が主流で、Focal LossやKL Divergence Loss（知識蒸留時）も使われました。不均衡データ対策として、クラス頻度に基づく**重み付きサンプリング**や損失計算時の重み付けが行われました。

推論においては、**5秒**またはそれより長いクリップで予測を行いました。計算時間制限（2時間）が厳しいため、**推論の高速化**が重要課題となり、**ONNX**や**OpenVINO**へのモデル変換、**TorchScript**の利用、**量子化**（成功例は少ない）、**並列処理**（ThreadPoolExecutor）などが駆使されました。最終提出は、性能と推論速度のバランスを考慮した**複数モデルのアンサンブル**が一般的でした。評価指標が閾値不要のC-MAPになったことで、後処理の複雑さは軽減されました。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/birdclef-2023/discussion/412808)**

* **アプローチ:** データ中心。Xeno-Canto APIのバグ修正によるデータ拡張。事前学習+ファインチューン。SEDモデルのアンサンブル。
* **アーキテクチャ:** SEDモデル。Backbone: eca\_nfnet\_l0, convnext\_small\_fb\_in22k\_ft\_in1k\_384, convnextv2\_tiny\_fcmae\_ft\_in22k\_in1k\_384。
* **アルゴリズム:** Focal Loss。Adam (Cosine Annealing)。クラス重み付きサンプリング。
* **テクニック:**
    * **データ:** 過去コンペデータ+Xeno-Canto (API修正版)。Zenodoのno-call区間。データクリーニング（重複除去）。
    * **学習:** 2段階（事前学習: 822種、ファインチューン: 264種）。5秒クリップ。
    * **Augmentation:** Mixup (OR Mixup), BackgroundNoise (Zenodo), RandomFiltering (カスタムEQ), SpecAugment。
    * **推論:** 5秒チャンク。推論高速化 (ONNX)。後処理 (温度付き平均: `(pred**2).mean()**0.5`, Attention probsとMax probsの重み付き平均)。
    * **アンサンブル:** 3モデル (異なるBackbone、異なる学習率) の平均。

**[2位](https://www.kaggle.com/competitions/birdclef-2023/discussion/412707)**

* **アプローチ:** SEDモデルとCNNモデル（2021年2位解法ベース）のアンサンブル。過去データ+Xeno-Canto活用。疑似ラベル+手動ラベル（効果薄）。
* **アーキテクチャ:**
    * SED: tf\_efficientnetv2\_s\_in21k, seresnext26t\_32x4d, tf\_efficientnet\_b3\_ns。
    * CNN (2021 2nd): tf\_efficientnetv2\_s\_in21k, resnet34d, tf\_efficientnet\_b3\_ns, tf\_efficientnet\_b0\_ns。
* **アルゴリズム:** CrossEntropyLoss -> BCEWithLogitsLoss。クラス重み付きDataLoader。(Optimizer/Scheduler不明)。
* **テクニック:**
    * **データ:** 過去コンペデータ+Xeno-Canto (前景/背景)。
    * **学習:** 2段階（事前学習: 全データ834種、ファインチューン: 264種）。10秒/15秒/20秒/30秒クリップ。
    * **Augmentation:** 多様（ノイズ系, Gain, Pitch/TimeShift, マスク, Mixup (Wave/Spec), 周波数カット, 背景種self-mixup）。
    * **推論:** SEDは10秒入力→中央5秒で予測。TTA (2s shift)。推論高速化 (OpenVINO)。
    * **アンサンブル:** 7モデル (SEDx3, CNNx4) の重み付き平均 または ランク平均。

**[3位](https://www.kaggle.com/competitions/birdclef-2023/discussion/414102)**

* **アプローチ:** 周波数帯Attentionを持つ修正SEDモデル。ドメインシフト対策としてのReverb Augmentation。
* **アーキテクチャ:** SEDモデル (周波数帯Attention модификация)。Backbone: tf\_efficientnet\_b0\_ns, tf\_efficientnetv2\_s\_in21k。
* **アルゴリズム:** BCEWithLogitsLoss。(Optimizer/Scheduler不明)。
* **テクニック:**
    * **データ:** 過去コンペデータ+Xeno-Canto (拡張)。ノイズデータ (BirdCLEF19, DCASE18など)。
    * **入力:** Log Mel Spectrogram (5秒)。スペクトログラムを90度回転させて周波数帯Attentionを実現。
    * **Augmentation:** 多様（ランダム位置クリップ, 疑似ラベル追加, シフト, フィルタ, Mixup (同種/異種/ノイズ), Gain, Pitch/TimeStretch, ノイズ系, Reverb, スペクトログラム補間, Color Jitter）。
    * **学習:** 全種 (659クラス、背景種含む) で学習、推論時に264クラスにフィルタリング。
    * **推論:** 推論高速化 (TorchScript, 入力スペクトログラム事前計算)。推論時タイマーによるタイムアウト回避。
    * **アンサンブル:** 8モデル (EffNetB0 x5 + EffNetV2s x3) の単純平均。

**[4位](https://www.kaggle.com/competitions/birdclef-2023/discussion/412753)**

* **アプローチ:** 知識蒸留 (Google Bird Vocalization Classifierモデルから)。CNNモデル (2021年2位解法ベース)。
* **アーキテクチャ:** CNNモデル (2021 2nd)。Backbone: eca\_nfnet\_l0 (4モデル全て)。入力形式違い (MelSpec, PCEN, MelSpec(別設定))。
* **アルゴリズム:** Loss = 0.1 * BCEWithLogitsLoss + 0.9 * KLDivLoss (Teacherモデルから)。Softmax温度=20。AdamW (CosineLRScheduler)。
* **テクニック:**
    * **データ:** 過去コンペデータ + ff1010bird\_nocall + Xeno-Canto (追加分) + Zenodo + ESC50。
    * **知識蒸留:** 事前計算したTeacherモデルの予測をターゲットとして利用。アグレッシブMixup (p=1.0)。多数エポック学習 (400)。
    * **学習:** 20秒クリップ。5-Fold CV (実質4 Fold)。Primary labelのみ使用。
    * **Augmentation:** Gain系, Noise系, BackgroundNoise (Zenodo, aicrowd), LowPassFilter, PitchShift。
    * **推論:** 推論高速化 (TorchScript JIT)。
    * **アンサンブル:** 4モデル (eca\_nfnet\_l0だが入力設定違い) の単純平均。

**[5位](https://www.kaggle.com/competitions/birdclef-2023/discussion/412903)**

* **アプローチ:** SEDモデル (2021年4位解法ベース)。2段階学習。疑似ラベル。
* **アーキテクチャ:** SEDモデル。Backbone: tf\_efficientnet\_b1\_ns, tf\_efficientnet\_b2\_ns, tf\_efficientnet\_b3\_ns, tf\_efficientnetv2\_s\_in21k。
* **アルゴリズム:** BCEWithLogitsLoss (Ratingに基づくサンプル重み)。AdamW (CosineLRScheduler)。
* **テクニック:**
    * **データ:** 過去コンペデータ + Xeno-Canto (追加分)。ESC50ノイズ, 2021 no-callノイズ。
    * **学習:** 2段階（事前学習: 2021/2022データ+WhiteNoise、ファインチューン: 2023データ+多様なAugmentation）。5秒クリップ。4-Fold CV + 全データ学習モデル。Primary/Secondaryラベル使用。
    * **Augmentation (Finetune):** Mixup (Wave), ノイズ系 (White, Pink, Brown, Injection, ESC50, No-call), スペクトログラムマスク (Time x2, Freq x1)。
    * **疑似ラベル:** 一部モデルでハード/ソフト疑似ラベルをファインチューンに利用。
    * **推論:** 推論高速化 (ONNX, ThreadPoolExecutor)。
    * **アンサンブル:** 15モデル (4 Backbone x (4 Fold + Full fit)) の平均。

**[6位](https://www.kaggle.com/competitions/birdclef-2023/discussion/412708)**

* **アプローチ:** BirdNET EmbeddingとCNNモデル (2021年2位解法ベース) の特徴量を結合。
* **アーキテクチャ:** CNN (2021 2nd、Backbone: eca\_nfnet\_l1, seresnext26t\_32x4d) の出力 + BirdNET (v2.2) のEmbedding -> 線形層。
* **アルゴリズム:** BCEWithLogitsLoss。AdamW (Warmup + Cosine Annealing)。
* **テクニック:**
    * **データ:** BirdCLEF 2021/2022 (事前学習), 2023 (本学習)。
    * **特徴量:** Mel Spectrogram + BirdNET Embedding (3秒分を平均)。BirdNETは学習せず固定。
    * **学習:** 30秒クリップ。ラベル重み付け (Primary=0.9995, Secondary=0.4/0.5)。Label Smoothing=0.01。全データで学習 (CVは最終Epochがベストだったため)。
    * **Augmentation:** Shift, Mixup, Gaussian Noise, Random Lowpass Filter, Random Power, Freq/Time Masking。
    * **Oversampling:** 少数クラス、複数Secondaryラベルを持つデータ、反響エフェクト追加データなどを追加。
    * **推論:** CNNは5秒、BirdNETは3秒入力。推論高速化 (ONNX, ThreadPoolExecutor)。
    * **アンサンブル:** 2モデル (eca\_nfnet\_l1, seresnext26t) の単純平均。

**[7位](https://www.kaggle.com/competitions/birdclef-2023/discussion/412922)**

* **アプローチ:** カスタムAugmentation `sumix` が中心。知識蒸留。Attention Head付きモデル。
* **アーキテクチャ:** EfficientNet-B2 (torchvision), ReXNet150 (timm)。一部モデルにAttention Head追加。
* **アルゴリズム:** Binary Cross Entropy (Primary+Secondaryラベル)。AdamW (定数LR, WD)。EMA。
* **テクニック:**
    * **データ:** BirdCLEF 2021/2022データのみ (外部データ、ノイズ未使用)。
    * **Augmentation:** `sumix` (波形加算+ラベル調整、p=1)、`concatmix` (p=0.5)、Mixup (Spec, p=1)、CutMix (Spec, p=0.5)。ノイズ系Augmentationは`sumix`に含まれるとして未使用。
    * **学習:** 5秒クリップ。Normalization層のWDオフ。混合精度 (`bfloat16`)。
    * **知識蒸留:** 学習済みモデルをTeacherとして、同じ設定で再学習。Loss = 0.33*BCE + 0.34*KL(Student||Teacher) + 0.33*KL(Teacher||Student)。
    * **推論:** 推論高速化 (OpenVINO)。
    * **アンサンブル:** 19モデル (EffNet-B2, ReXNet150, Attention Head有無、蒸留有無など) の平均。

**[8位](https://www.kaggle.com/competitions/birdclef-2023/discussion/412871)**

* **アプローチ:** マルチモーダルData Augmentation (SOD, レコメンデーション風Mixupなど)。段階的学習。
* **アーキテクチャ:** CNNモデル (2021年2位解法ベース)。Backbone: eca\_nfnet\_l0。
* **アルゴリズム:** (損失、Optimizer、Scheduler不明)。
* **テクニック:**
    * **データ:** BirdCLEF 2023, 2020/2021/2022 (事前学習)。ff1010bird\_nocall。Zenodo。
    * **学習:** 段階的ダウンスケーリング学習（長いスペクトログラム入力から始め、徐々に時間軸解像度を下げる）。
    * **Augmentation (Preprocessing):**
        * YOLOv8によるSound Object Detection (SOD) を利用し、鳥のいる可能性が高い区間を選択、ノイズ領域をマスク。
        * Geometrical Mixup (地理的に近いサンプルをMixup)。
        * Resonance Mixup (協調フィルタリング風に共鳴しやすいペアを選択してMixup)。
        * Inner Mixup (地理的に近く同ラベルのクラスタ内で自由にMixup)。
    * **CV:** Train/Test Split。テストデータは30秒を5秒に分割しブートストラップ。Primaryラベルのみで評価。
    * **推論:** 推論高速化 (ONNX, ThreadPoolExecutor)。
    * **アンサンブル:** 6モデル (eca\_nfnet\_l0、異なるMel設定、異なるEpoch) の平均。

**[9位](https://www.kaggle.com/competitions/birdclef-2023/discussion/412794)**

* **アプローチ:** 複数ラウンド学習 (過去データ、蒸留、疑似ラベル)。CNNモデルのアンサンブル。
* **アーキテクチャ:** EfficientNet-B0, eca\_nfnet\_l0, ConvNeXt-Tiny。一部にAttention Head。
* **アルゴリズム:** (損失不明)。AWP, SWA。(Optimizer/Scheduler不明)。
* **テクニック:**
    * **データ:** 過去コンペデータ利用。
    * **学習:** 複数ラウンドで学習設定やデータを変更（蒸留、疑似ラベルも使用）。5秒クリップ。
    * **Augmentation:** HFlip, Cutout, ノイズ系 (Pink, Gaussian, Injection), Random Volume, Pixel Dropout。
    * **疑似ラベル:** 新たなSecondaryラベルとして使用 (prob=0.5)。
    * **推論:** 推論高速化 (OpenVINO、バッチロード最適化: 8ファイル同時ロード)。
    * **アンサンブル:** 7モデル (EffNet-B0 x2, eca\_nfnet\_l0 x2, ConvNeXt-Tiny x3) の重み付き平均 (Sigmoid後に平均)。

**[10位](https://www.kaggle.com/competitions/birdclef-2023/discussion/412713)**

* **アプローチ:** SEDモデル。段階的スペクトログラムパラメータ変更学習。多様な小型Backbone利用。
* **アーキテクチャ:** SEDモデル。Backbone: efficientnetv2\_m, eca\_nfnet\_l0, efficientnet\_v0, mnasnet\_100, spnasnet\_100, mobilenetv2\_100 (Seresnext50, Resnet34は性能不足で除外)。
* **アルゴリズム:** BCEWithLogitsLoss + BCEFocal2WayLoss。(Optimizer/Scheduler不明)。
* **テクニック:**
    * **データ:** 2023データのみ (一部モデルは2022データで事前学習)。
    * **学習:** 5秒クリップ。ソフトラベル (Primary=1, Secondary=0.3)。
    * **段階的学習:** 異なるMelスペクトログラムパラメータで学習したモデルを読み込み、別のパラメータセットで再学習・ファインチューン。
    * **Augmentation:** GaussianNoise, Background Noise, PinkNoise, Gain調整, CutMix, Mixup。
    * **CV:** 5-Fold CV、各モデルでベスト1-2 Foldを選択してアンサンブル。
    * **推論:** 推論高速化 (ONNX (量子化含む), ThreadPoolExecutor)。
    * **アンサンブル:** 6-7モデル (異なるBackbone、異なるFold) の平均。



