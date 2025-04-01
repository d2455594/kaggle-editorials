---
tags:
  - Kaggle
startdate: 2023-02-07
enddate: 2023-06-29
---
# Predict Student Performance from Game Play
[https://www.kaggle.com/competitions/predict-student-performance-from-game-play](https://www.kaggle.com/competitions/predict-student-performance-from-game-play)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、子供たちがプレイする教育ゲーム「Jo Wilder and the Capitol Case」の**イベントログデータ**（クリック、移動、会話などの記録）を分析し、ゲーム内の特定の**問題（質問）に正解できるかどうかを予測する**ことです。
* **背景:** ゲームベース学習は、学習者のエンゲージメントを高め、効果的な学習体験を提供する可能性があります。しかし、ゲームプレイ中のどの行動が学習成果に結びついているのかを理解することは困難です。このコンペでは、詳細なプレイログデータを用いて、生徒の学習状況（特定の問題を解ける能力）をリアルタイムに近い形で把握し、個別最適化された学習支援を提供するための基盤技術を開発することを目指しています。
* **課題:**
    * **長大なシーケンスデータ:** 各プレイヤーのセッションデータは数千から数万のイベントログで構成され、非常に長大です。
    * **多様なプレイスタイル:** プレイヤーごとにゲームの進め方やインタラクションのパターンは大きく異なります。
    * **ノイズと冗長性:** ログデータには、学習に直接関係しない操作や繰り返し行動などが多く含まれます。
    * **学習状態の潜在性:** プレイヤーの理解度やスキル習得度は、ログデータから間接的に推測する必要があります。
    * **効率的な推論:** 将来的にリアルタイムでの介入を目指すため、効率的な特徴量計算と推論が求められます。
    * **データリーク問題:** コンペ期間中に外部公開データとの重複によるデータリークが発覚し、データセットが更新されるという経緯がありました。

**データセットの形式 (Dataset Format)**

提供される主なデータは、プレイヤーごとのゲームプレイログと、各問題への正解ラベルです。

1.  **イベントログデータ (`train.csv`, `test.csv`):**
    * 各行がプレイヤーの1つのアクション（イベント）に対応する時系列データ。
    * `session_id`: 各プレイセッション（プレイヤーごと）のユニークID。
    * `index`: セッション内でのイベントの順序を示すインデックス。
    * `elapsed_time`: セッション開始からの経過時間（ミリ秒）。
    * `event_name`: イベントの種類（例: `Maps_click`, `person_click`, `cutscene_click`, `observation_click`, `notification_click`, `object_click`, `map_hover`, `notebook_click`, `map_click`, `checkpoint` など）。
    * `name`: イベントの具体的な内容（例: `basic`, `undefined`, `close`, `open`, `prev`, `next` など）。
    * `level`: ゲーム内のレベル (0-22)。
    * `page`: ノートブックのページ番号 (NaNあり)。
    * `room_coor_x`, `room_coor_y`: 部屋座標 (NaNあり)。
    * `screen_coor_x`, `screen_coor_y`: 画面座標 (NaNあり)。
    * `hover_duration`: ホバーイベントの継続時間 (NaNあり)。
    * `text`: 表示されたテキスト内容 (NaNあり)。
    * `fqid`: ゲーム内のオブジェクトやキャラクターなどのユニークID (NaNあり)。
    * `room_fqid`: 部屋のユニークID。
    * `text_fqid`: 表示されたテキストに対応するユニークID (NaNあり)。
    * **これら多数の列から効果的な特徴量を抽出することが重要です。**
2.  **正解ラベルデータ (`train_labels.csv`):**
    * **ターゲット変数**を提供します。
    * `session_id`: セッションID。
    * `level_group`: 問題が出題されるレベルの範囲 (`0-4`, `5-12`, `13-22`)。
    * `q`: 問題番号 (1から18)。
    * `correct`: その問題に正解したかどうか (1: 正解, 0: 不正解)。
    * `session_level` (非公式): `session_id` と `level_group`, `q` を組み合わせたユニークIDとしてしばしば使われます。
3.  **外部データ:**
    * Field Day Lab の Open Game Data Portal ([https://fielddaylab.wisc.edu/opengamedata/](https://fielddaylab.wisc.edu/opengamedata/)) から、より多くのゲームプレイログデータが入手可能でした。多くのトップチームがこのデータを活用しました。
4.  **`sample_submission.csv`:**
    * 提出フォーマットのサンプル。`session_level` (例: `20090312431273200_q1`) と `correct` (予測された正解確率) の列を持ちます。

**評価指標 (Evaluation Metric)**

* **指標:** **平均 F1 スコア (Average F1 Score)**
* **計算方法:**
    1.  各問題（q1からq18まで）について、予測された正解確率に閾値（例: 0.5）を適用して二値分類（正解/不正解）を行います。
    2.  各問題ごとに F1 スコアを計算します。
        F1 = 2 * (Precision * Recall) / (Precision + Recall)
    3.  全18問の F1 スコアを平均して最終スコアとします。
* **意味:** 各問題に対する正解・不正解の分類性能を、適合率（Precision）と再現率（Recall）の調和平均で評価し、それを全体で平均したものです。クラス不均衡（一般的に正解の方が多い）がある場合でも、両方のクラスの予測性能をバランス良く評価します。スコアは **0から1** の範囲を取り、**1に近い**ほど性能が良いことを示します。最適な閾値を見つけることもスコア向上に繋がります。

要約すると、このコンペティションは、教育ゲームの詳細なイベントログ時系列データから、プレイヤーが各問題に正解するかどうかを予測する二値分類タスクです。データはイベントシーケンスとそのメタデータ、正解ラベルで構成され、性能は全問題の平均F1スコア（高いほど良い）によって評価されます。

---

**全体的な傾向**

このコンペティションでは、生徒のゲームプレイログという長大な時系列データから、各問題の正答を予測する必要がありました。上位解法の多くは、**勾配ブースティング木 (GBDT)**、特に **XGBoost**, **CatBoost**, **LightGBM** を中心に構築されました。これは、テーブル形式の特徴量エンジニアリングが効果的であったことを示唆しています。一部では、**ニューラルネットワーク (NN)**、特に **Transformer** や **1D CNN**、**LSTM/GRU** を活用するアプローチも成功を収め、最終的にはGBDTとNNの**アンサンブル**が多くのチームで採用されました。

**特徴量エンジニアリング**が極めて重要であり、勝敗を分ける最大の要因となりました。特に効果的だったのは以下の種類のようです。
* **時間ベースの特徴量:**
    * イベント間の経過時間 (`elapsed_time_diff`) の集計（合計、平均、標準偏差、最大、最小など）。
    * 特定のイベント（例: `checkpoint`, `notification_click`）間の所要時間やイベント数。
    * レベルグループごとの総プレイ時間、各レベルでの滞在時間。
    * ホバー時間 (`hover_duration`) の集計。
* **カウントベースの特徴量:**
    * イベントタイプ (`event_name`, `name`) やオブジェクトID (`fqid`, `room_fqid`, `text_fqid`) の出現回数。
    * 特定のキー（複数列の組み合わせ）でのイベント回数。
    * ユニークなIDの数。
* **シーケンス/順序ベースの特徴量:**
    * イベントインデックス (`index`) の差分集計。
    * 特定イベントの最初/最後のインデックス。
    * 前後のイベントとの関係性を示す特徴量。
* **座標ベースの特徴量:**
    * 部屋 (`room_coor`) や画面 (`screen_coor`) の座標移動距離の集計（効果は限定的との報告もあり）。
* **メタ特徴量:**
    * 前のレベルグループの予測確率を現在のレベルグループの特徴量として利用。
    * 各セッションの最大到達レベル。

**外部データの活用**も多くのチームで行われ、Field Day Lab の Open Game Data Portal から取得した追加のセッションデータがモデル性能向上に寄与しました。

**モデル構築戦略**として、以下の点が特徴的でした。
* **Level Group別 vs 全体 vs 問題別:** 予測モデルをレベルグループごと (0-4, 5-12, 13-22) に構築するか、全18問を1つのモデルで予測するか（問題番号を特徴量とする）、あるいは18問それぞれにモデルを構築するか、様々なアプローチが試されました。アンサンブルのために複数の戦略を組み合わせることもありました。
* **NNアーキテクチャ:** NNでは、Transformer Encoderやカスタム1D CNN（WaveNet風）、LSTM/GRUなどが試されました。時間情報を効果的に埋め込むためのカスタムEmbedding層 (TimeEmbedding) も開発されました。
* **2段階学習/スタッキング:** NNの埋め込み層を事前学習したり、GBDTとNNの予測値をメタ特徴量としてロジスティック回帰やMLPで最終予測を行うスタッキングも有効でした。

**効率化**も重要なテーマであり、特徴量計算に `polars` や `numba`、GBDT推論に `lleaves` や `Treelite`、NN推論に `TensorFlow Lite (TF Lite)` などが活用されました。

**CV戦略**としては、`session_id` に基づく `GroupKFold` が一般的でした。データリーク問題があったため、信頼できるCVの構築と、Public LBに過度に依存しないモデル選択が重要でした。最終的な**閾値**の選択（全体で1つ vs 問題ごとに設定）もスコアに影響を与えました。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/predict-student-performance-from-game-play/discussion/420321)**

* **アプローチ:** XGBoost + NN (カスタムConv1D) の50/50アンサンブル。外部データ活用。堅牢なCVと効率性重視。
* **アーキテクチャ:** XGBoost。NN: カスタムTimeEmbedding + カテゴリEmbedding -> WaveNet風Conv1Dバックボーン -> MLP/Skip接続Head。
* **アルゴリズム:** XGBoost: (デフォルト損失)。NN: BCE Loss。(Optimizer/Scheduler不明)。
* **テクニック:**
    * **特徴量 (GBDT):** 時間とカウントの集計が主。座標情報は限定的。全履歴を使用。
    * **特徴量 (NN):** 5特徴量 (duration, text_fqid, room_fqid, fqid, event_name+name)。時間特徴量をカスタムConv1Dで埋め込み。
    * **学習:** NNは2段階 (バックボーン事前学習 -> 全体End-to-End)。ソート順 (元/インデックス) で複数モデル学習。
    * **CV:** 10 bags x 5 folds (GBDT)、Consensus (NN)。ノイズレベル以上の改善のみ採用。
    * **効率化:** TF Lite (NN)、Treelite (XGBoost)。
    * **アンサンブル:** 複数Fold/Seed/ソート順のモデルを平均。閾値0.625。

**[2位](https://www.kaggle.com/competitions/predict-student-performance-from-game-play/discussion/420319)**

* **アプローチ:** 単一LightGBMモデル。時間ベースの特徴量を重視。効率化に注力。
* **アーキテクチャ:** LightGBM。
* **アルゴリズム:** LightGBM (デフォルト損失)。
* **テクニック:**
    * **特徴量:** **時間関連**が多数（タスク完了時間、レポート閲覧時間など）。基本特徴（問題番号、イベント数）も重要。1296特徴量。
    * **学習:** 全データで単一モデルを学習 (CVは5 Foldで実施)。
    * **効率化:** 特徴量生成コードにnumbaとC言語を多用。推論時間の大部分はLGBM予測。
    * **閾値:** 0.63。

**[3位](https://www.kaggle.com/competitions/predict-student-performance-from-game-play/discussion/420217)**

* **アプローチ:** GBDT (XGBoost, CatBoost) + NN (Transformer+LSTM) のアンサンブル。外部データ活用。多様なモデリング戦略。
* **アーキテクチャ:** XGBoost, CatBoost。NN: GRU/LSTM -> Transformer Encoder (Conformer風/標準) -> BiLSTM+LSTM Head。
* **アルゴリズム:** GBDT: (デフォルト)。NN: BCE Loss。Multi-task学習 (他問題も補助出力)。
* **テクニック:**
    * **モデリング戦略:** Question別GBDT、Level Group別GBDT、全体GBDT、Level Group別NN、全体NNを構築。
    * **特徴量 (GBDT):** 時間/インデックス差分集計、座標移動距離、特定イベント間時間/インデックス ("Flag events", "Checkpoint", 回答時間)、"不要な操作"検出、タスク所要時間。
    * **特徴選択:** Null Importance + gp_minimize。
    * **特徴量 (NN):** 時間差分 (log1p)、カテゴリ特徴 (event_comb, room_fqid, page, text_fqid, level)。
    * **ソート順:** GBDTはインデックス順/時間順の両方を試行。
    * **スタッキング:** GBDT群とNN群のoof予測を別々にLogistic Regressionでアンサンブルし、最終的に6:4で重み付け平均。

**[4位](https://www.kaggle.com/competitions/predict-student-performance-from-game-play/discussion/420264)**

* **アプローチ:** Transformer + XGBoost + CatBoost のアンサンブル。外部データ活用。
* **アーキテクチャ:** 軽量Transformer (1層)。XGBoost。CatBoost。Linear Regression (スタッキング用)。
* **アルゴリズム:** Transformer: (損失不明)。XGBoost/CatBoost: (デフォルト)。Linear Regression。
* **テクニック:**
    * **データ前処理:** インデックス順ソート。ホバーイベント除去。Transformer入力用にイベント種類削減 (0.999以上出現するイベントを除去、max_len=452)。
    * **特徴量 (Transformer):** 6特徴量 (時間差分, インデックス差分, 座標距離差分, 部屋XY, カテゴリ埋め込み)。シーケンス長256にクロップ。
    * **特徴量 (GBDT):** Transformer入力特徴量をフラット化 + 基本的な集計特徴量。
    * **モデリング戦略:** Level Groupごとにモデル構築（問題番号を特徴量として入力）。
    * **スタッキング:** 各モデルの予測確率をメタ特徴量とし、問題ごとにLinear Regressionで最終予測。過去と未来の予測確率も入力。
    * **アンサンブル:** 3モデル x 3シードの平均を入力としてスタッキング。

**[7位](https://www.kaggle.com/competitions/predict-student-performance-from-game-play/discussion/420324)**

* **アプローチ:** 単一LightGBMモデル (Level Group別)。時間差分特徴量中心。効率化。
* **アーキテクチャ:** LightGBM。
* **アルゴリズム:** LightGBM (デフォルト)。
* **テクニック:**
    * **特徴量:** **時間差分集計**が主。集計キーは6変数 (level, name, event_name, room_fqid, fqid, text) の組み合わせ。各キーの出現回数も使用。'notification_click'イベント間時間も重要。
    * **特徴選択:** レアなキー組み合わせを除外後、Gain Importanceに基づきTop 500-700特徴量を選択して再学習。
    * **外部データ:** 活用。セッションの最大到達レベルを特徴量に追加。
    * **ラベル拡張:** 最終Level Groupモデル学習時に、中間Level Groupのラベルも学習データに追加（過学習抑制効果）。
    * **CV:** 4-Fold CVを3回繰り返し（シード変更）。
    * **効率化:** 特徴量計算にPythonリスト処理活用。推論にlleavesライブラリ使用。
    * **閾値:** 0.625。

**[8位](https://www.kaggle.com/competitions/predict-student-performance-from-game-play/discussion/420314)**

* **アプローチ:** XGBoost + Transformer (+GRU/LSTM) のアンサンブル。外部データ活用。
* **アーキテクチャ:** XGBoost (Question別)。Transformer (+GRU/LSTM, RIIIDコンペ風)。
* **アルゴリズム:** XGBoost: (デフォルト)。Transformer: (不明)。
* **テクニック:**
    * **特徴量 (XGBoost):** セッション長、オブジェクトクリック関連("Instance features")、Bingo特徴量、標準的なカウント/時間集計、インデックスビニング、ホバー集計、Level Group上位特徴量、過去予測確率("Meta features")。
    * **特徴選択 (XGBoost):** Gain/SHAP Importance、Null Importance。重複/高欠損率特徴量除去。
    * **特徴量 (NN):** 全特徴量使用。
    * **学習 (NN):** Multi-task学習 (他問題も補助的に予測)。
    * **外部データ:** 活用。
    * **推論順序:** 元のシーケンス順序を保持するように工夫。異常インデックスセッションの再インデックス化。
    * **アンサンブル:** XGBoostとTransformerモデルをアンサンブル。

**[9位](https://www.kaggle.com/competitions/predict-student-performance-from-game-play/discussion/420312)**

* **アプローチ:** LightGBM + CatBoost の平均アンサンブル。Question別モデル。時間ベースの特徴量重視。
* **アーキテクチャ:** LightGBM, CatBoost。
* **アルゴリズム:** LightGBM, CatBoost (デフォルト)。
* **テクニック:**
    * **特徴量:** **時間関連**が重要。"Checkpoint"特徴量 (特定イベント間の時間/クリック数)。その他、レベル経過時間、平均座標など。特徴数はLevel Groupごとに増加 (最大18610)。
    * **CV:** 10-Fold StratifiedKFold (合計正解数で層化)。
    * **特徴選択:** FoldごとにImportance Top 500を選択し、そのFoldのモデルを再学習。
    * **メタ特徴量:** 過去の予測確率を使用。
    * **効率化:** 推論にlleavesライブラリ使用。
    * **アンサンブル:** LightGBMとCatBoostの単純平均。

**[10位](https://www.kaggle.com/competitions/predict-student-performance-from-game-play/discussion/420295)**

* **アプローチ:** 複数GBDT (XGBoost, LightGBM) + NN (LSTM+Transformer) -> MLP/ロジスティック回帰でスタッキング -> 平均 + 閾値最適化。
* **アーキテクチャ:** XGBoost, LightGBM (Level Group別)。NN (LSTM+Transformer)。MLP, Logistic Regression (スタッキング用)。
* **アルゴリズム:** GBDT/NN: (不明)。MLP/LogReg: (標準的)。
* **テクニック:**
    * **特徴量 (GBDT):** 時間差分とホバー時間の集計、数値特徴量。過去Level Groupの特徴量と予測確率を利用。
    * **効率化:** 特徴量計算にnumpy+numba。
    * **CV:** 5-Fold StratifiedGroupKFold。
    * **スタッキング:** Stage1モデル (NNx1, LGBMx1, XGBx4) のoof予測を入力として、問題ごとにMLPとLogRegを学習。
    * **アンサンブル:** Stage2の2モデルの予測を平均。
    * **閾値最適化:** 問題ごとに閾値を最適化 (Powell法)。



