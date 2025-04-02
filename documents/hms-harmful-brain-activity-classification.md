---
tags:
  - Kaggle
  - 脳波
  - EfficientNet
  - SwinTransformer
  - ダブルバナナモンタージュ
  - ヘルスケア
startdate: 2024-01-10
enddate: 2024-04-09
---
# HMS - Harmful Brain Activity Classification
[https://www.kaggle.com/competitions/hms-harmful-brain-activity-classification](https://www.kaggle.com/competitions/hms-harmful-brain-activity-classification)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、集中治療室 (ICU) などで記録された脳波 (EEG) データを用いて、発作 (Seizure) を含む有害な脳活動パターンを自動的に分類する機械学習モデルを開発することです。
* **背景:** 脳波モニタリングは、特に意識障害のある患者において、非けいれん性発作のような検出が難しい神経学的イベントを発見するために重要です。しかし、専門家によるリアルタイムでのEEG解釈はリソース的に困難であり、自動検出システムの開発が求められています。
* **課題:** EEG信号はノイズが多く、患者間・記録間のばらつきも大きいです。様々な種類の有害な脳活動パターン (Seizure, LPD, GPD, LRDA, GRDA, Other) を、専門家の多数決に基づく曖昧なラベルから高精度に分類することが求められます。これは、時系列データに対するマルチラベル分類タスクと捉えることができます。

**データセットの形式 (Dataset Format)**

提供される主なデータは、EEG信号データ、事前に計算されたスペクトログラム、およびそれらに対応する専門家の投票ラベルです。

1.  **トレーニングデータ:**
    * `train_eegs/`: 各EEG記録のデータ（`.parquet`形式）。中央の50秒間のEEG信号が含まれます。多数の電極からの時系列信号データです。
    * `train_spectrograms/`: 各EEG記録に対応する事前に計算されたスペクトログラム（`.parquet`形式）。約10分間のデータから計算されています。
    * `train.csv`: メタデータファイル。
        * `eeg_id`: EEG記録の一意なID。
        * `spectrogram_id`: スペクトログラムの一意なID。
        * `patient_id`: 患者の一意なID。
        * `eeg_sub_id`, `spectrogram_sub_id`: 同じ`eeg_id`内で異なる時間オフセットを持つサブサンプルを示すID。
        * `eeg_label_offset_seconds`: `train_eegs`内の50秒クリップの開始時間オフセット。
        * `expert_consensus`: 各EEG記録に対する専門家の分類結果（Seizure, LPD, GPD, LRDA, GRDA, Other）。
        * 各クラス (`seizure_vote`, `lpd_vote`, `gpd_vote`, `lrda_vote`, `grda_vote`, `other_vote`) に対する投票数。
        * **ターゲット変数:** これら投票数を合計投票数で正規化した確率分布。
2.  **テストデータ:**
    * `test_eegs/`, `test_spectrograms/`: トレーニングデータと同様の形式でEEGデータとスペクトログラムが提供されます。
    * `test.csv`: テストデータに対応するメタデータ (`eeg_id`, `spectrogram_id`, `patient_id`)。
    * 参加者は、このテストデータに対して各クラスの確率を予測します。
3.  **`sample_submission.csv`**:
    * 提出フォーマットのサンプル。`eeg_id` と各クラス (`seizure_vote`など) の予測確率の列を持ちます。

**評価指標 (Evaluation Metric)**

* **指標:** **Kullback-Leibler (KL) Divergence** (カルバック・ライブラー情報量)
* **計算方法:** 予測された確率分布 P と、専門家の投票に基づく真の確率分布 Q との間の平均KLダイバージェンスを計算します。
    * KL(Q || P) = Σ Q(i) * log(Q(i) / P(i)) （i は各クラス）
    * 評価指標 = (1 / N) * Σ KL(Q\_j || P\_j) （j は各EEG記録、NはEEG記録の総数）
    * 予測確率 P(i) がゼロにならないように、小さな値（例: 1e-6）でクリッピングされます。
* **意味:** モデルが予測した脳活動パターンの確率分布が、専門家の意見に基づく真の確率分布にどれだけ近いかを測ります。KLダイバージェンスは、2つの確率分布間の「距離」や「差異」を表す指標であり、**低い**ほどモデルの予測が真の分布に近いことを示します。したがって、このコンペティションではスコアが**低い**ほど良いモデルと評価されます。

要約すると、このコンペティションは、EEG信号やスペクトログラムから有害な脳活動パターンを分類するマルチラベル分類タスクです。データはEEG時系列、スペクトログラム、および専門家の投票ラベルで構成され、性能は予測確率分布と真の確率分布との間のKLダイバージェンス（低いほど良い）によって評価されます。

---

**全体的な傾向**

このコンペティションでは、EEG信号をどのようにモデルに入力するかが重要なポイントでした。主に以下の3つのアプローチが見られました。

1.  **2Dスペクトログラム + 2D CNN/Transformer:** EEG信号を時間周波数表現であるスペクトログラム（特にメルスペクトログラムやSTFT）に変換し、画像認識で実績のある2D CNN（EfficientNet, ConvNeXt, ResNetなど）やVision Transformer（ViT, Swin Transformer, MaxViTなど）に入力する。Kaggle提供のスペクトログラムと、参加者が独自に生成したスペクトログラムの両方が使われました。複数の時間窓（例: 50秒全体と中央10秒）の情報を組み合わせることも有効でした。
2.  **1D生波形 + 1D CNN/RNN/Transformer:** 生のEEG時系列データを直接1D CNN（WaveNet, EEGNet, Squeezeformerベースなど）、RNN（LSTM, GRU）、あるいはTransformerベースのモデルで処理する。
3.  **マルチモーダルアプローチ:** 上記1と2、あるいは複数の異なる特徴（例: スペクトログラムと生波形の両方、異なるパラメータで生成したスペクトログラム）を組み合わせ、複数のバックボーンを持つモデルや、特徴量を融合するモデルを構築する。

**データの扱いとCV戦略**も重要でした。

* **ターゲットラベル:** 専門家の投票数を正規化した確率分布をターゲットとする。投票数が少ないデータ（例: `vote < 10`）の扱い（学習に使うか、重み付けするか、疑似ラベルを使うか）が工夫されました。
* **CV:** 患者ID (`patient_id`) に基づくGroupKFoldが標準的に用いられました。評価時には、専門家の意見がより一致しているデータ（例: `vote >= 10`）のみを用いることで、Public/Private LBとの相関を高める試みが多く見られました。
* **前処理:** バンドパスフィルター（Butterworthなど）、ノッチフィルター（60Hzなど）、クリッピング、信号のモンタージュ（特にDouble Banana Montage）が広く利用されました。
* **Augmentation:** Mixup、CutMix、スペクトログラムマスキング（XYMasking）、波形の反転・シフト・ノイズ注入、チャンネルシャッフル/マスキング、脳の左右対称性を利用したフリップなどが有効でした。

**学習とアンサンブル:**

* **学習:** AdamWオプティマイザ、Cosine Annealing学習率スケジューラ、EMA (Exponential Moving Average)、Label Smoothing、2段階学習（全データでの事前学習後、高品質データでのファインチューニング）などが多く採用されました。
* **アンサンブル:** 異なるモデルアーキテクチャ、異なる入力形式（スペクトログラム、生波形）、異なるFoldの予測結果を加重平均するアンサンブルがスコア向上に不可欠でした。重みは手動調整、Nelder-Mead法、L-BFGS-B法、あるいは簡単なニューラルネットワークで学習されました。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/hms-harmful-brain-activity-classification/discussion/492560)**

* **アプローチ:** 4人のメンバーによるモデルのアンサンブル。2Dスペクトログラムモデルと1D波形モデルの両方を使用。
* **アーキテクチャ:**
    * 2D (スペクトログラム): EfficientNet, MaxViT, ConvNeXt, Swin Transformerなど。入力は独自生成のメルスペクトログラム（複数チャンネルを結合）。
    * 1D (生波形): WaveNetベースのカスタムアーキテクチャ、LSTMなど。
* **アルゴリズム:** KL Divergence Loss。AdamW。Cosine Annealing。
* **テクニック:**
    * **CV:** GroupKFold (`patient_id`)。評価は `vote >= 10` データ。
    * **データ:** Double Banana Montage。独自生成メルスペクトログラム (torchaudio)。複数時間窓 (50s/10s) の利用。高品質データ (`vote >= 10`) でのファインチューニング。疑似ラベルの活用。
    * **Augmentation:** Mixup, CutMix, スペクトログラムマスキング, 波形反転/シフト。
    * **アンサンブル:** 各メンバーのモデル出力を加重平均。

**[2位](https://www.kaggle.com/competitions/hms-harmful-brain-activity-classification/discussion/492254)**

* **アプローチ:** スペクトログラム用3D CNNと生波形用2D CNNの組み合わせ、およびそれらの個別モデルのアンサンブル。
* **アーキテクチャ:**
    * スペクトログラム: X3D-L (3D CNN)。16チャンネルのスペクトログラムを時間軸として入力。EfficientNet-B5 (2D CNN) も使用 (チャンネルを連結して画像化)。
    * 生波形: EfficientNet-B5, HGNet-B5 (2D CNN)。波形を(16*10)x1000の画像にreshapeして入力。
    * 複合: X3D-LとEfficientNet-B5の特徴量を結合。
* **アルゴリズム:** KL Divergence Loss。AdamW。Cosine Schedule。
* **テクニック:**
    * **前処理:** Double Banana Montage。0.5-20Hzバンドパスフィルター (MNE/Scipy)。+/-1024でクリップ。
    * **スペクトログラム:** STFT (torchaudio)。50秒全体と中央10秒の情報を結合。
    * **学習:** 2段階学習 (最初は全データ、次に `vote >= 6` データ)。オフセットを用いたターゲット平均化。
    * **Augmentation:** 波形の左右反転 (ミラーリング)。
    * **CV/Validation:** 10 Fold。評価は `vote >= 6` データ。
    * **アンサンブル:** 6モデル (スペクトログラムx2, 生波形x3, 複合x1) の加重平均。

**[3位](https://www.kaggle.com/competitions/hms-harmful-brain-activity-classification/discussion/492471)**

* **アプローチ:** 多様な2つのアプローチ（2D CNN + スペクトログラム、1D CNN + Squeezeformer + 生波形）によるモデルのアンサンブル。
* **アーキテクチャ:**
    * 2D (スペクトログラム): MixNet-L/XL。16チャンネルのメルスペクトログラムを連結。中央10秒を拡大（ズーム）する工夫。
    * 1D (生波形): カスタム1D CNN (Grouped Convolution主体) + Squeezeformerブロック。
* **アルゴリズム:** KL Divergence Loss。AdamW。Cosine Decay。
* **テクニック:**
    * **CV/データフィルタリング:** Patient IDベースの4-Fold。評価と学習データとして `vote > 9` のデータ (約6350行) を主に利用。冗長な拡張データを除去。
    * **前処理:** Double Banana Montage。Butterworthバンドパスフィルター (GPU on-the-fly)。信号クリップ `/ 32` による正規化。チャンネル順序の最適化。
    * **スペクトログラム:** メルスペクトログラム (torchaudio)。
    * **Augmentation:** チャンネルマスキング、バンドパス範囲変更、時間窓シフト、波形時間反転、左右脳反転 (1Dモデル)、中央10秒シフト (2Dモデル)。
    * **学習:** 1Dモデルは低投票数データも段階的に利用（損失重み減衰）。Drop Path。
    * **アンサンブル:** 23モデル (2D x 18, 1D x 5) の予測結果を、小さなNNで学習した重みとバイアス項を用いて結合。

**[4位](https://www.kaggle.com/competitions/hms-harmful-brain-activity-classification/discussion/492240)**

* **アプローチ:** 4人のメンバーによる1Dモデルと2Dモデル、計8モデルのアンサンブル。
* **アーキテクチャ:**
    * 2D (スペクトログラム): Swin Transformer v2 Large, TinyViT, CAFormer-S18, ConvNeXt-Large。Chrisの手法に基づくスペクトログラム、Kaggleスペクトログラムとの結合など。
    * 1D (生波形): 左右対称性を意識したカスタム1D CNN + Transformer/LSTM、MobileNetV2ベースの1D CNN（チャンネルミキサー付き）。
* **アルゴリズム:** KL Divergence Loss。AdamW。Cosine Schedule。EMA。
* **テクニック:**
    * **CV/データ:** GroupKFold。評価は `vote >= 10` データ。`vote >= 10` または `vote > 7` データで学習/ファインチューニング。
    * **前処理:** Butterworthバンドパスフィルター (0.5-30Hz or 0.5-20Hz)。チャンネル品質係数(CQF)の導入。
    * **スペクトログラム:** Chrisの手法。複数秒数 (10/30/40/50s) の試行。
    * **Augmentation:** Mixup (EEG波形レベル), XYMasking, 波形反転 (hflip), チャンネルシャッフル, ランダム中央クロップ (波形)。
    * **学習:** 2段階学習。疑似ラベルの利用。
    * **アンサンブル:** 8モデルの予測結果を、線形レイヤー (nn.Linear) を用いて全学習データにオーバーフィットさせて重みを学習。Seizure/LRDA/GRDAクラスのアップサンプリング。

**[5位](https://www.kaggle.com/competitions/hms-harmful-brain-activity-classification/discussion/492652)**

* **アプローチ:** 5人のメンバーによるモデルのアンサンブル。1Dと2Dの両アプローチを含む。
* **アーキテクチャ:** (各メンバーの詳細に依存)
    * 2D (スペクトログラム): EfficientNet, ConvNeXt, MaxViT, Vision Transformerなど。独自生成メルスペクトログラム、CWTスペクトログラム。
    * 1D (生波形): WaveNet, LSTM, Transformerベースなど。
* **アルゴリズム:** KL Divergence Loss。AdamW。Cosine Schedule。
* **テクニック:**
    * **CV/データ:** GroupKFold。高品質データ (`vote >= 10` or `vote > 7`) での学習/評価。
    * **前処理:** Double Banana Montage, Butterworthフィルター。
    * **スペクトログラム:** メルスペクトログラム、CWT。
    * **Augmentation:** Mixup, CutMix, SpecAugment, 時間反転、チャンネルマスキング。
    * **学習:** 2段階学習、疑似ラベル。
    * **アンサンブル:** 各メンバーのモデル予測の加重平均。

**[6位](https://www.kaggle.com/competitions/hms-harmful-brain-activity-classification/discussion/492619)**

* **アプローチ:** 4つの入力 (Kaggleスペクトログラム, EEGスペクトログラム, 生波形50s, 生波形10s) を持つマルチモーダルな単一Vision Transformerモデルとそのアンサンブル。
* **アーキテクチャ:** DinoV2 (ViT family) を各入力のバックボーンとして使用。最終層の特徴量を結合して分類。
* **アルゴリズム:** KL Divergence Loss。Adan optimizer。One Cycle LR Scheduler。EMA。
* **テクニック:**
    * **CV:** GroupKFold (`patient_id`)。
    * **前処理:** NaN率が高いeeg_idを除去。Kaggleスペクトログラムはクリップ＆対数変換。EEGスペクトログラムは `signal.spectrogram` で生成し4x4に連結。生波形はButterworth(0.5-40Hz)、クリップ、reshapeして画像化 (200x(4*4*50)など)。
    * **入力:** 4種類の入力にそれぞれViTバックボーンを適用。
    * **Augmentation:** Mixup, XYMasking (スペクトログラム), ランダムNaN挿入 (生波形)。
    * **学習:** 2段階学習 (`vote < 10` → `vote >= 10`)。Gradient Checkpointing。各eeg_idからランダムなオフセットを選択。
    * **TTA:** 中央10秒の生波形入力に対し、時間シフト (左右2秒) を含む3x TTA。
    * **アンサンブル:** 異なる設定 (スペクトログラム生成法、ViTサイズ) の単一モデル5つのアンサンブル。

**[7位](https://www.kaggle.com/competitions/hms-harmful-brain-activity-classification/discussion/492281)**

* **アプローチ:** 2D生波形モデルと2Dスペクトログラムモデル (STFT, Multitaper) のアンサンブル。Kaggleスペクトログラムは不使用。
* **アーキテクチャ:** ConvNeXt-Base, MaxViT-Tiny。
* **アルゴリズム:** KL Divergence Loss。
* **テクニック:**
    * **前処理:** 18個のバイポーラ信号差分を使用。
    * **生波形入力:** 各信号を(20, 500)にreshapeし、18個を縦に結合して(360, 500)の2D画像とする。50秒全体と中央10秒の情報も利用。
    * **スペクトログラム入力:**
        * STFT: `cusignal`で各信号のスペクトログラムを生成し、縦に結合して(512, 512)の画像とする。複数時間窓 (50s, 50/10s, 50/30/10s, 30/10s) のモデルを学習。
        * Multitaper: Pythonライブラリで生成し、結合とパディングで(500, 372)の画像とする。50秒全体と中央10秒の情報も利用。
    * **アンサンブル:** 3種類のデータ形式 (Raw, STFT, Multitaper) からのモデル予測を重み (0.2, 0.65, 0.15) でブレンド。

**[8位](https://www.kaggle.com/competitions/hms-harmful-brain-activity-classification/discussion/492482)**

* **アプローチ:** 3つの異なる単一モデル (2Dスペクトログラム, 2D波形プロット, 1D生波形) を事前学習し、それらを結合した「メガモデル」を構築。疑似ラベルも活用。
* **アーキテクチャ:**
    * 2Dスペクトログラム: TinyViT-21M-512。
    * 2D波形プロット: TinyViT-21M-224。
    * 1D生波形: EEGNet-1D。
    * メガモデル: 上記3つのバックボーンの特徴量を結合。
* **アルゴリズム:** KL Divergence Loss。
* **テクニック:**
    * **データ/CV:** `vote >= 10` データ (専門家の意見) を重視。`vote < 10` データには疑似ラベルを使用 (`new_label = 0.1*old + 0.9*model_preds`)。CVは `vote >= 10` で評価。
    * **前処理:** MNEライブラリでノッチフィルター(60Hz), バンドパスフィルター(0.5-40Hz)。クリッピング(-500, 500)。
    * **スペクトログラム:** `torchaudio` でメルスペクトログラム生成 (50秒全体と中央10秒)。
    * **Augmentation:** 波形レベルで適用 (左右反転, 上下反転, クロップ・リサイズ, 周波数ドロップアウトなど)。
    * **学習:** 3つの単一モデルを `vote >= 10` データで学習 → `vote < 10` データに疑似ラベル付け → メガモデルを `vote >= 10` (2コピー) + 疑似ラベル付き `vote < 10` (1コピー) データで学習。
    * **最終提出:** メガモデル単体 (14位) と、他のモデルとのアンサンブル (8位)。

**[9位](https://www.kaggle.com/competitions/hms-harmful-brain-activity-classification/discussion/492429)**

* **アプローチ:** 3人のメンバーによる2Dモデルのアンサンブル。生波形、EEGスペクトログラム、Kaggleスペクトログラムを入力とするマルチエンコーダーモデルが特徴。
* **アーキテクチャ:**
    * マルチエンコーダー: 生波形 (GRU)、EEGスペクトログラム、Kaggleスペクトログラムそれぞれにバックボーン (EfficientNetV2-S/M, ResNet34d) を適用し、Attentionで特徴量を集約。あるいは、1D LSTM/WaveNet (生波形)、2D ResNet18d (スペクトログラム) を組み合わせた後、EfficientNetで処理。
* **アルゴリズム:** KL Divergence Loss。コントラスティブ損失 (生波形 vs EEGスペクトログラム)。補助損失 (NaN予測)。L-BFGS-B (アンサンブル重み最適化)。
* **テクニック:**
    * **CV/データ:** StratifiedGroupKFold (5 or 10 fold)。各eeg_idから代表的なサブサンプルを選択。`vote >= 4` で学習後、`vote >= 10` でファインチューニング。疑似ラベルの利用 (3段階目学習)。
    * **前処理:** 50チャンネルまでの多様なモンタージュ。EKGチャンネルの利用。ノッチフィルター(60Hz)、ハイパスフィルター(0.4Hz)。Robust Scaler。
    * **生波形特徴:** 異なる時間窓 (50s/10s, 40s/10sなど) の波形をGRUで処理。
    * **Augmentation:** 波形反転、Mixup (波形レベル)、チャンネルスワッピング/カットアウト、ラベルAugmentation (投票数変更)。
    * **アンサンブル:** L-BFGS-Bを用いて多数 (約150) のモデルから重要度の高いモデル (約20-30) を選択し、重みを最適化。

**[10位](https://www.kaggle.com/competitions/hms-harmful-brain-activity-classification/discussion/492207)**

* **アプローチ:** 豊富な情報を含む単一の入力画像を生成し、2D Vision Transformerで処理する単一モデルアプローチとそのアンサンブル。
* **アーキテクチャ:** EfficientViT-B2, MixNet-XL。
* **アルゴリズム:** KL Divergence Loss。Adan optimizer。
* **テクニック:**
    * **入力画像生成:** 19チャンネル (Double Banana Montage) のスペクトログラム、中央10秒のスペクトログラム、生波形データを連結して1つの大きな画像を作成。
    * **CV/データ:** GroupKFold。評価は全サブIDの平均。学習は `vote >= 10` データを重視 (重み付け)。`vote >= 10` と `< 10` で異なる線形ヘッドを使用。
    * **前処理:** 19チャンネルの差分信号。
    * **スペクトログラム:** `scipy.spectrogram` (torchaudioより良好だった)。
    * **学習:** 1段階学習 (`vote >= 10` 重視)。各eeg_idからランダムなサブID (高投票数優先) を選択。
    * **アンサンブル:** 約20モデルのアンサンブル (多様性は低い)。Optunaで重みを最適化。
    * **単一モデル:** 最良の単一モデルでLB 0.233, PB 0.287を達成。
