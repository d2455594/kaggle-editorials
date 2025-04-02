---
tags:
  - Kaggle
startdate: 2023-09-06
enddate: 2023-12-06
---
# Child Mind Institute - Detect Sleep States
[https://www.kaggle.com/competitions/child-mind-institute-detect-sleep-states](https://www.kaggle.com/competitions/child-mind-institute-detect-sleep-states)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、手首に装着した加速度センサー（アクチグラフ）から収集された多日の時系列データを用いて、睡眠状態の変化、具体的には**入眠 (onset) と覚醒 (wakeup) のイベント**が発生する正確なタイミングを検出するモデルを開発することです。
* **背景:** Child Mind Institute は、子供たちのメンタルヘルスと学習障害に焦点を当てた研究・臨床ケアを行う独立非営利組織です。睡眠は子供たちの健康と発達に不可欠であり、客観的かつ正確な睡眠パターンの測定は、様々な状態の診断、モニタリング、治療法の開発に役立ちます。アクチグラフは、自然な環境下で長期間の活動データを収集する比較的安価で非侵襲的な方法ですが、そのデータから睡眠状態を正確に推定することは技術的な課題でした。
* **課題:**
    * **個人差の大きさ:** 被験者の年齢、健康状態、生活習慣などにより、睡眠パターンや活動量データは大きく異なる。
    * **データのノイズと欠損:** センサーデータにはノイズが含まれるほか、被験者がデバイスを外すことによる長期間の欠損（偽データ期間）が存在する。
    * **イベント定義とラベルの曖昧さ:** 入眠や覚醒の正確な瞬間を定義し、ラベル付けすることは本質的に難しく、特に短時間の中途覚醒などはノイズと区別しにくい。
    * **長時間時系列データ:** 各被験者のデータは数日から数週間にわたり、非常に長いシーケンスを効率的に処理する必要がある。
    * **イベントのスパース性:** 入眠・覚醒イベントは、長い記録期間の中で見ると比較的まれにしか発生しない。
    * **評価指標 (Average Precision):** イベント検出のタイミング精度が、複数の時間的許容範囲で厳しく評価される。

**データセットの形式 (Dataset Format)**

提供される主なデータは、加速度センサーの時系列データと、専門家によってラベル付けされた睡眠イベントのタイムスタンプです。

1.  **トレーニングデータ:**
    * `train_series.parquet`: 複数の被験者 (`series_id`) のアクチグラフデータを含むテーブル。各 `series_id` は非常に長い時系列データを持つ。
        * `series_id`: 各記録セッション（被験者ごと）の一意なID。
        * `step`: 記録開始からのタイムステップ（5秒間隔）。
        * `timestamp`: 各ステップに対応するタイムスタンプ（UTC）。
        * `anglez`: デバイスのZ軸（垂直方向）の角度。重力に対するデバイスの向きを示す。
        * `enmo`: 加速度のユークリッドノルムから1g（重力加速度）を引いた値。活動量の指標となる。
    * `train_events.csv`: 専門家によって判定された睡眠イベントのデータ。
        * `series_id`: 対応する `train_series.parquet` のID。
        * `night`: 各 `series_id` 内での睡眠期間（夜）の通し番号。
        * `event`: イベントの種類 (`"onset"`: 入眠、または `"wakeup"`: 覚醒)。
        * `step`: イベントが発生した正確な `step`。**ターゲット変数**。
        * `timestamp`: イベントに対応するタイムスタンプ（参考情報）。
2.  **テストデータ:**
    * `test_series.parquet`: テスト用の被験者のアクチグラフデータ。フォーマットは `train_series.parquet` と同じ。
3.  **`sample_submission.csv`**:
    * 提出フォーマットのサンプル。検出された各イベントに対して行を作成する。
        * `row_id`: 提出する各イベント予測の一意なID（自動生成）。
        * `series_id`: 対象となる `series_id`。
        * `step`: 予測されたイベントが発生した `step`。
        * `event`: 予測されたイベントの種類 (`"onset"` または `"wakeup"`)。
        * `score`: その予測に対する信頼度スコア（0から1）。評価指標の計算に使用される。

**評価指標 (Evaluation Metric)**

* **指標:** **平均適合率 (Average Precision, AP)** をイベント検出用に調整したもの。
* **計算方法:**
    * 予測された各イベント（`step`, `event`, `score`）と、正解のイベント（`step`, `event`）を比較します。
    * マッチングは、イベントタイプ (`onset`/`wakeup`) が一致し、かつ予測 `step` と正解 `step` の差が指定された **許容範囲 (tolerance window)** 内にあるかどうかで行われます。許容範囲は複数設定されます（例: ±1分, ±3分, ..., ±30分）。
    * 各許容範囲について、予測を信頼度スコア (`score`) の降順に並べます。
    * 上位から順に予測を見ていき、正解イベントとマッチすれば真陽性 (True Positive, TP)、マッチしなければ偽陽性 (False Positive, FP) とカウントし、適合率 (Precision) と再現率 (Recall) を計算していきます。
    * これにより、各許容範囲での適合率-再現率曲線が描かれ、その曲線下面積 (Average Precision) が計算されます。
    * 最終スコアは、**全ての許容範囲におけるAPスコアの平均値**となります。
* **意味:** モデルが睡眠/覚醒イベントを正しく検出し、かつそのタイミングがどれだけ正確であるかを、複数の時間スケールで総合的に評価します。信頼度スコアの高い予測がより正確であることが求められ、時間的なずれに対するペナルティが段階的に考慮されます。スコアは**高い**ほど良い性能を示します (最大1.0)。

要約すると、このコンペティションは、手首センサーの時系列データから睡眠の開始と終了のタイミングを高精度で検出するイベント検出タスクです。データは加速度と角度の時系列、およびイベントのタイムスタンプで構成され、性能は複数の時間許容度での平均適合率（高いほど良い）によって評価されます。

---

**全体的な傾向**

この睡眠状態検出コンペティションでは、長大な1次元時系列データからスパースなイベント（入眠・覚醒）を検出するという課題に対し、多様なアプローチが見られました。主流となったのは、**RNN (GRU, LSTM)** や **1D CNN**、そして近年時系列分析で注目される **Transformer** をベースとした深層学習モデルです。特に、セグメンテーションタスクで実績のある **U-Net構造** を1次元データに適用したモデルが多くのチームで採用されました。

入力データとしては、生の `anglez` と `enmo` に加え、**特徴量エンジニアリング**によって生成された特徴量が重要な役割を果たしました。移動平均、標準偏差、差分、フーリエ変換などの統計量や周波数特徴、さらに時間帯（時、分、曜日）やデータの周期性（デバイス取り外しによる偽データ期間の検出など）に関する特徴が広く利用されました。

長い時系列データを扱うため、データを一定の長さ（例: 1日分、数日分）の**チャンクに分割**してモデルに入力し、推論時にはオーバーラップさせながら予測値を結合・平均化する手法が一般的でした。

ターゲット変数の設計も工夫が見られ、イベント発生時刻のみを1とするスパースなターゲットではなく、イベント時刻周辺に**ガウシアン分布や指数減衰関数を用いたソフトターゲット（ヒートマップ）** を作成し、モデルが時間的な近さも学習できるようにするアプローチが有効でした。

モデルの出力から最終的なイベント時刻を決定する**後処理**は、スコア向上に不可欠でした。単純な閾値処理ではなく、**ピーク検出アルゴリズム** (例: `scipy.signal.find_peaks`) や、検出されたピーク候補の中から適切なものを選択・抑制する **NMS (Non-Maximum Suppression)** や **WBF (Weighted Boxes Fusion)** に類似した手法が用いられました。

また、深層学習モデルの出力を特徴量として利用し、**LightGBMなどの勾配ブースティングモデルで再スコアリング**を行う2段階のアプローチも、特に上位チームで見られました。これにより、1日あたりのイベント数の制約（各タイプ最大1回）や、より大域的な特徴を考慮した予測が可能になりました。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/child-mind-institute-detect-sleep-states/discussion/459715)**

* **アプローチ:** 1D CNN-GRU-CNNモデル + 2段階目の再スコアリングモデル群による後処理。
* **アーキテクチャ:** Stage 1: CNN(Down)+Residual GRU+CNN(Up)。Stage 2: LGBM, CatBoost, CNN, RNN, Transformerなど多数。
* **アルゴリズム:** Sequence-to-Sequence, Gradient Boosting, etc.
* **テクニック:** データは日毎+オフセットでチャンク化。ターゲットは距離減衰ソフトターゲット(エポック毎に減衰)。周期性ノイズ検出とフィルタリング/特徴量化。SEモジュール。分特徴を最終層結合。**Stage 2後処理**: 候補点を中心に特徴量抽出し、多様なモデルで再スコアリング。許容誤差毎のAP推定に基づきイベントを貪欲法で選択。

**[2位](https://www.kaggle.com/competitions/child-mind-institute-detect-sleep-states/discussion/459627)**

* **アプローチ:** 1D U-Net + LGBM (2段階/3段階)。
* **アーキテクチャ:** Stage 1: 1D U-Net (CNNベース)。Stage 2/3: LGBM。
* **アルゴリズム:** Sequence-to-Sequence, Gradient Boosting。
* **テクニック:** ターゲットはガウシアンヒートマップ。特徴量エンジニアリング多数。Stage 1で候補検出後、Stage 2 LGBMで再スコアリング (日次イベント数制約考慮)。Stage 3 LGBMで候補時刻を微調整。複数モデル(2種CNN)のアンサンブル (Average + WBF風)。

**[3位](https://www.kaggle.com/competitions/child-mind-institute-detect-sleep-states/discussion/459599)**

* **アプローチ:** GRU + UNET + LGBMのアンサンブル。
* **アーキテクチャ:** GRU, 1D UNET, LGBM。
* **アルゴリズム:** Sequence-to-Sequence, Gradient Boosting。
* **テクニック:** データは日毎、30秒粒度に集約。特徴量少数 (anglez/enmo標準偏差、周期性ノイズフラグ、時間帯頻度エンコーディング)。ターゲット周辺をソフトラベル化(+1/-2ステップを1)。Augmentation (系列反転)。後処理: ピーク検出+距離閾値。

**[4位](https://www.kaggle.com/competitions/child-mind-institute-detect-sleep-states/discussion/459637)** (Nikhilパート)

* **アプローチ:** 修正UNET (LSTM/GRU + Transformer)。WBF風後処理。
* **アーキテクチャ:** Modified UNET (Encoder→Bottleneck→Transformer→GRU/LSTM→Decoder)。
* **アルゴリズム:** Sequence-to-Sequence。
* **テクニック:** ターゲットは正規化ガウシアン。入力シーケンスを**パッチ化**して次元削減。UNETのSkip ConnectionはConcatではなくAdd。後処理: Convolution + WBF風アルゴリズム (適応的ウィンドウサイズ、対数/線形重み付け)。

**[5位](https://www.kaggle.com/competitions/child-mind-institute-detect-sleep-states/discussion/459766)**

* **アプローチ:** ヒューリスティック候補生成 + LGBM (時刻補正+スコアリング)。
* **アーキテクチャ:** LightGBM (2種類)。
* **アルゴリズム:** Heuristics, Gradient Boosting (回帰+分類)。
* **テクニック:** Stage 1: ヒューリスティックルール (非活動期間、偽データ期間) で候補step生成。Stage 2: LGBM回帰でstepを補正。Stage 3: LGBM分類で候補スコアを予測。特徴量エンジニアリング多数 (Window特徴量、時間特徴、日次特徴、候補間特徴など)。後処理: step微調整、近接候補除去、日次スコア正規化。偽データ期間検出ルール。

**[6位](https://www.kaggle.com/competitions/child-mind-institute-detect-sleep-states/discussion/459604)**

* **アプローチ:** BiLSTM-UNet。1分粒度集約。
* **アーキテクチャ:** BiLSTMベースのUNet構造 (Skip ConnectionはAdd)。一部Transformerブロック使用。
* **アルゴリズム:** Sequence-to-Sequence。
* **テクニック:** データは日毎+パディング、1分粒度に集約。特徴量に周期性(minute%15)、**重複データ検出フラグ**。ターゲットは複数距離閾値でのイベント有無 (マルチターゲットBCE Loss)。後処理: ピーク検出+10分間抑制。step微調整(+/-1)。

**[7位](https://www.kaggle.com/competitions/child-mind-institute-detect-sleep-states/discussion/459598)**

* **アプローチ:** Wavenet + LightGBMスタッキング。
* **アーキテクチャ:** Wavenet (カスタムWaveBlock)、LightGBM。
* **アルゴリズム:** 1D CNN (Dilated Conv), Gradient Boosting。
* **テクニック:** データは3日間スライディングウィンドウ、1分粒度。特徴量にボラティリティ、過去日同分差分。ターゲット近傍1分も1、近傍2-3分は損失計算無視。2ヘッド出力 (Pointwise CE + 15分窓BCE)。OHEM。後処理: ピーク検出+抑制ルール、低確率候補サンプリング。LGBMスタッキング。

**[8位](https://www.kaggle.com/competitions/child-mind-institute-detect-sleep-states/discussion/460617)**

* **アプローチ:** 1D U-Net Regressor + 1D U-Net Transformer DensityNet。
* **アーキテクチャ:** Regressor: 1D U-Net (ResNet backbone)。DensityNet: 1D U-Net + Transformer Encoder (ALiBi)。
* **アルゴリズム:** Sequence Regression (Huber Loss), Sequence Classification (Cross Entropy)。
* **テクニック:** **Regressor**はイベント位置までの相対距離を予測。**DensityNet**は2日間入力で中央1日のイベント確率密度を予測。ターゲットはラプラス分布で平滑化。後処理: Regressorのピーク候補をDensityNetの条件付き確率でスコアリング。NMS。Matrix Profileによる偽データ除去。入力はanglez/enmo/timeを分けて学習。

**[9位](https://www.kaggle.com/competitions/child-mind-institute-detect-sleep-states/discussion/460372)**

* **アプローチ:** 複数モデル(GRU, Transformer, LGBMなど)のアンサンブル。チームメンバー各々が異なるアプローチを開発。
* **アーキテクチャ/テクニック:** 各メンバーの個別Write-up参照。後処理(Youriパート)で最終統合。
    * Tomo: GRU+CNN+Attention
    * Shu: Transformer
    * Chris: LightGBM
    * Youri: 後処理（ピーク検出、スコア閾値、NMS、時差補正など）

**[10位](https://www.kaggle.com/competitions/child-mind-institute-detect-sleep-states/discussion/459894)**

* **アプローチ:** 1D-UNet GRUモデルの大規模アンサンブル (120モデル)。
* **アーキテクチャ:** 1D-UNet GRU。
* **アルゴリズム:** Sequence-to-Sequence。
* **テクニック:** 特徴量エンジニアリング (差分絶対値、時間エンコーディング、重複データフラグ)。ターゲットは**大小シグマのガウシアン混合ヒートマップ**。12時間ランダムサンプリング学習。SWA。後処理: ピーク検出+抑制ルール+低確率候補サンプリング。大規模アンサンブル。
