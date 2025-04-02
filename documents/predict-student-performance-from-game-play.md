---
tags:
  - Kaggle
  - 教育
  - MacroF1Score
startdate: 2023-02-07
enddate: 2023-06-29
---
# Predict Student Performance from Game Play
[https://www.kaggle.com/competitions/predict-student-performance-from-game-play](https://www.kaggle.com/competitions/predict-student-performance-from-game-play)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、教育ゲーム「Jo Wilder and the Capitol Case」のプレイヤーの行動ログデータを用いて、ゲーム内の**評価問題（全18問）に対する正誤を予測する**機械学習モデルを開発することです。
* **背景:** このゲームは、特に読解スキルを向上させる目的で開発されました。プレイヤーのゲーム内での行動（クリック、移動、テキスト閲覧、オブジェクト操作など）を分析することで、学習の進捗状況や理解度を評価し、より効果的な教育介入につなげることを目指しています。ゲームベースの評価は、従来のテスト形式に代わる、より自然で魅力的な評価方法となる可能性があります。
* **課題:** 提供されるデータは、各プレイヤー（セッション）ごとの詳細な**時系列イベントログ**です。この大量のログデータから、18個の評価問題の正誤に関連する**有効な特徴量を抽出・設計する**ことが最大の課題となります。データにはセッションごとに長さが大きく異なる、イベントの種類が多い、ノイズが含まれるといった特徴があります。また、コンペ期間中に**データ漏洩の問題**が発覚し、データセットが更新されるという出来事もありました。

**データセットの形式 (Dataset Format)**

提供される主なデータは、ゲームのプレイログと正解ラベルです。

1.  **トレーニングデータ:**
    * `train.csv`: 各プレイヤー（`session_id`）のゲームプレイ中のイベントログ。
        * `session_id`: プレイヤー（セッション）を一意に識別するID。
        * `index`: セッション内のイベント発生順序（0から始まる）。
        * `elapsed_time`: セッション開始からの経過時間（ミリ秒）。
        * `event_name`: イベントの種類（例: `Maps_click`, `person_click`, `cutscene_click`）。
        * `name`: イベントの詳細（例: `basic`, `undefined`, `close`）。
        * `level`: ゲーム内のレベル（0〜22）。
        * `page`: ノートブックのページ番号（NaNの場合あり）。
        * `room_coor_x`, `room_coor_y`: 部屋の中での座標（NaNの場合あり）。
        * `screen_coor_x`, `screen_coor_y`: 画面上の座標（NaNの場合あり）。
        * `hover_duration`: ホバー時間（NaNの場合あり）。
        * `fqid`: 操作対象の完全修飾ID（Full Qualified ID）（例: `door`, `notebook`, `trophy`）。
        * `room_fqid`: 現在いる部屋の完全修飾ID（例: `tostacks`, `tobasement`）。
        * `text_fqid`: 表示されたテキストの完全修飾ID（例: `tunic.historicalsociety.closet_dirty.trigger_scarf`）。
        * `text`: 表示されたテキスト内容（NaNの場合あり）。
        * `level_group`: 質問が出題されるレベルのグループ（`0-4`, `5-12`, `13-22`）。
    * `train_labels.csv`: トレーニングデータの各セッションにおける、18個の質問に対する正解ラベル。
        * `session_id`: セッションIDと質問番号（q1〜q18）を組み合わせたもの（例: `2009031243127320_q1`）。
        * `correct`: その質問に正解したかどうか（1: 正解, 0: 不正解）。
2.  **テストデータ:**
    * コンペ期間中は、隠されたテストセットに対して予測を行うための**API**が提供されました。参加者は、APIから逐次送られてくるセッションのログデータを受け取り、18個の質問に対する正解確率を予測して返す必要がありました。
    * ログデータの形式は`train.csv`と同様です。
3.  **外部データ:**
    * Field Day LabのOpen Game Data Portal ([https://fielddaylab.wisc.edu/opengamedata/](https://fielddaylab.wisc.edu/opengamedata/)) から、追加のゲームプレイログデータが利用可能でした。多くの参加者がこのデータを利用してモデルの性能を向上させました。
4.  **`sample_submission.csv`**:
    * 提出フォーマットのサンプル。`session_id`（例: `2009031243127320_q1`）と`correct`（予測確率）の列を持ちます。

**評価指標 (Evaluation Metric)**

* **指標:** **Macro F1 Score**
* **計算方法:**
    1.  18個の各質問（q1〜q18）について、予測確率をある閾値（例: 0.5）で0か1に変換し、それぞれのF1スコアを計算します。
        * Precision = TP / (TP + FP)
        * Recall = TP / (TP + FN)
        * F1 = 2 * (Precision * Recall) / (Precision + Recall) （TP: 真陽性、FP: 偽陽性、FN: 偽陰性）
    2.  18個の質問のF1スコアの単純平均を計算します。
        * Macro F1 = (F1\_q1 + F1\_q2 + ... + F1\_q18) / 18
* **意味:** モデルが各質問に対してどれだけバランス良く正解・不正解を予測できているかを評価します。各質問の重要度を均等に扱い、全体の予測性能を示します。スコアは**高い**ほど良い評価となります。閾値の選択がスコアに影響を与える可能性があります。

---

**全体的な傾向**

このコンペティションは、教育ゲームのログデータから生徒の成績を予測するもので、本質的には**時系列データを用いた多ラベル分類（または18個の二値分類）タスク**と捉えることができます。上位解法の多くは、**勾配ブースティング木（GBDT: XGBoost, LightGBM, CatBoost）** と**ニューラルネットワーク（NN: Transformer, Conv1D, LSTM/GRU）** の**アンサンブル**に基づいていました。

**特徴量エンジニアリング**が勝敗を分ける最も重要な要素であり、多くのチームが膨大な時間を費やしました。特に**時間に関する特徴量**（イベント間の経過時間 `elapsed_time_diff`、ホバー時間 `hover_duration`、特定のタスクにかかった時間など）や、**イベントの発生回数**（特定のイベント `event_name` や `fqid` のカウント、特定のエリア `room_fqid` での行動回数など）が重要視されました。ゲームのドメイン知識（どのイベントが重要か、どのシーケンスが学習状況を示すかなど）を活用した特徴量や、座標情報（移動距離など）、テキスト情報、さらには過去の質問の予測確率なども有効な特徴量として利用されました。

データセットに関しては、主催者から提供されたデータに加え、**公開されている外部データ**を活用することが一般的でした。データの前処理（欠損値処理、ログのソート順（`index` vs `elapsed_time`））や、信頼性の高い**交差検証（CV）戦略**（GroupKFoldなど）の構築も重要でした。特にコンペ途中でデータ漏洩の問題とデータ更新があったため、LB（リーダーボード）スコアだけでなく、安定したCVスコアに基づいてモデルや特徴量を選択するアプローチが求められました。

モデリング戦略としては、18個の質問それぞれにモデルを構築するアプローチ、レベルグループ（`0-4`, `5-12`, `13-22`）ごとにモデルを構築するアプローチ、全質問を単一モデルで予測するアプローチなど、多様な方法が試されました。

推論フェーズでは、APIの仕様に合わせた効率的な処理が求められ、**Polars**や**Numba**といった高速化ライブラリの活用や、NNモデルの軽量化（**TF Lite**）、GBDTモデルの高速化（**Treelite**, **lleaves**）なども見られました。最終的な予測ラベルを決定するための**閾値最適化**もスコア向上に貢献しました。

---

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/predict-student-performance-from-game-play/discussion/420217)**

* **アプローチ:** XGBoostとNN（カスタムTimeEmbedding + Conv1Dベース）のアンサンブル。時間（duration）特徴量を重視。
* **アーキテクチャ:**
    * GBDT: XGBoost。
    * NN: カスタムTimeEmbedding（Conv1Dベース）で時間情報をエンコードし、カテゴリカル特徴量（`text_fqid`, `room_fqid`, `fqid`, `event_name`+`name`）のEmbeddingと結合。WaveNetや時間認識イベントの論文に触発された構造。トレーニングは2段階（バックボーン事前学習＋全体ファインチューニング）。
* **アルゴリズム:**
    * GBDT: XGBoost。
    * NN: BCE Loss。AdamW。
    * アンサンブル: GBDTとNNの単純平均（50/50）。
* **テクニック:**
    * **データ:** Field Day Labの公開データを活用し、ほぼ完全なトレーニングデータを再構築。セッション完了データのみ使用（約37kセッション）。データ漏洩を発見し報告。
    * **特徴量 (GBDT):** 時間（duration）とカウントが中心。レベル、部屋、テキスト閲覧、イベントタイプごとの集約。過去の全イベントを使用。座標情報は限定的に利用。
    * **特徴量 (NN):** duration, text\_fqid, room\_fqid, fqid, event\_name+name の5つ。DurationはカスタムTimeEmbeddingで処理。Time-aware events: `duration * (event_1 + event_2 + ...)` の形で時間とイベントを結合。
    * **検証:** 厳密なCV戦略。GBDTは10Bag x 5Foldの平均でノイズ以上の改善のみ採用。NNは1Bag x 5Foldで3〜4Fold改善で採用。
    * **学習:** NNはバックボーン（レベルグループごと）を事前学習し、その後全体をEnd-to-Endで学習（バックボーンはフリーズ）。
    * **推論:** APIシミュレータを構築。NNモデルはTF Liteで高速化。単一閾値（0.625）を使用。多数のモデル（2x4x10 fold GBDT + 3x4x5 fold NN）をアンサンブル。

**[2位](https://www.kaggle.com/competitions/predict-student-performance-from-game-play/discussion/424329)**

* **アプローチ:** 単一のLightGBMモデル。全質問を一つのモデルで予測。効率性（推論速度）を重視。
* **アーキテクチャ:** LightGBM。
* **アルゴリズム:** LightGBM。
* **テクニック:**
    * **データ:** 全データで最終モデルをトレーニング（CVは開発中のみ使用）。
    * **特徴量:** 時間ベースの特徴量が重要。特にレベル1でレポートを閲覧した時間（`LG0_L1_first_report_open_duration`）が効果的。特徴量は1296個。
    * **効率化:** 特徴量生成コードにNumbaとC言語を多用し高速化。推論時間の大部分はLightGBMの予測ステージが占めるように最適化。
    * **推論:** 単一閾値（0.63）を使用。

**[3位](https://www.kaggle.com/competitions/predict-student-performance-from-game-play/discussion/420235)**

* **アプローチ:** GBDT（CatBoost, XGBoost）とNN（Transformer+LSTM）のアンサンブル。複数のモデリング戦略（質問ごと、レベルグループごと、全質問一括）を組み合わせ。
* **アーキテクチャ:**
    * GBDT: CatBoost, XGBoost。
    * NN: Transformer（Conformerライク、Last Query Attentionなど3種） + BiLSTM + LSTM。Poolingはsum, std, max, lastをconcat。
* **アルゴリズム:**
    * GBDT: CatBoost, XGBoost。
    * NN: Multi-label BCE Loss。
    * アンサンブル: GBDTとNNのOOF予測をロジスティック回帰でスタッキング後、重み付き平均（GBT:NN = 6:4）。
* **テクニック:**
    * **データ:** Field Day Labの生データから追加データセット（11kセッション）を生成し活用（特にNNで効果あり）。
    * **モデリング戦略:** GBDTではレベルグループごと（2b）や全質問一括（3b）、NNではMulti-label（2a, 3a）を採用。
    * **特徴量 (GBDT):** レベルグループ間の経過時間、フラグイベント間の時間/indexカウント、過去の質問の予測確率、座標（部屋/スクリーン距離）、特定タスクの時間（例: ノート発見時間）、不要な行動（経験者判定用）の時間/回数など。
    * **特徴量 (NN):** `elapsed_time_diff` (log1p), `event_comb`, `room_fqid`, `page`, `text_fqid`, `level`。
    * **検証:** GroupKFold (session\_id)。追加データは検証セットに含めず。
    * **特徴量選択 (GBDT):** Null Importancesに基づきCatBoostで実施後、`gp_minimize`で最適な特徴量数を探索。
    * **学習 (NN):** レベルグループごと、または全レベルグループ共通で学習。追加データ使用。
    * **推論:** 閾値は共通。

**[4位](https://www.kaggle.com/competitions/predict-student-performance-from-game-play/discussion/420349)**

* **アプローチ:** Transformer、XGBoost、CatBoostのアンサンブル。各モデル3シードx5Fold。線形回帰のメタモデルで最終予測。
* **アーキテクチャ:**
    * NN: 軽量TransformerEncoder (1層, 64次元)。連続値とカテゴリ値（インデックス）を入力。全質問共通モデル。
    * GBDT: XGBoost, CatBoost。レベルグループごとにモデル構築（質問番号を入力特徴量に）。
    * メタモデル: 線形回帰（各質問ごと）。
* **アルゴリズム:**
    * NN: BCE Loss (multi-label)。
    * GBDT: XGBoost, CatBoost。
    * アンサンブル: 線形回帰。
* **テクニック:**
    * **データ:** Field Day Labの生データの大部分（約58kセッション）を学習に使用。検証はKaggleデータのみ。前処理として`hover`行を削除。
    * **特徴量 (NN):** ゲーム内の特定ポイント（`event_name`等を結合して一意化）を定義し、頻出ポイントを除外。ポイント間の時間差、index差、移動距離差、座標、カテゴリ（ポイントID）を使用。
    * **特徴量 (GBDT):** NNの入力特徴量（数値5列）を平坦化してそのまま使用。その他、公開カーネルにあるようなカテゴリごとの統計量（時間差のmean, maxなど）。
    * **検証:** 5Fold CV。
    * **アンサンブル:** 各ベースモデル（Transformer, XGBoost, CatBoost）の3シードx5FoldのOOF予測確率を線形回帰メタモデルへの入力とする。メタモデルは過去と未来の質問の予測確率も入力に使用（例: q7予測にq1-13の予測確率を使用）。
    * **推論:** 閾値の影響が大きいことを認識し、最終提出ではCV最適閾値（0.62）とLB最適閾値（0.60）、および0.64の3つを提出。

**[7位](https://www.kaggle.com/competitions/predict-student-performance-from-game-play/discussion/420119)**

* **アプローチ:** LightGBMモデルをレベルグループごとに3つ構築。効率性（推論速度）を重視し、効率性トラックで1位獲得。
* **アーキテクチャ:** LightGBM。
* **アルゴリズム:** LightGBM。
* **テクニック:**
    * **データ:** Field Day Labの生データを追加学習に使用（CV+0.002）。セッションごとの最大到達レベルを特徴量に追加。
    * **特徴量:** 連続するアクション間の時間差（`elapsed_time_diff`）に重点。6つのカテゴリ変数（`level`, `name`, `event_name`, `room_fqid`, `fqid`, `text`）を連結したキーで集約した時間差の合計や出現回数を特徴量に。`notification_click`イベント間の時間差も使用。PandasではなくPythonリスト操作で効率的に計算。
    * **特徴量選択:** 頻度の低いキーを除外後、学習率大（0.1）で一度学習し、Gain Importanceに基づき500〜700個の特徴量を選択。
    * **学習:** 選択された特徴量で学習率小（0.02）で再学習。最後のレベルグループの学習時には、2番目のレベルグループのラベル情報をAugmentationとして利用（過学習抑制）。
    * **検証:** 4Fold CVを3回（異なるシードで）。
    * **推論:** 単一閾値（0.625）を使用。CVモデルは使わず、全データで再学習したモデルを使用。推論時間約3分。

**[8位](https://www.kaggle.com/competitions/predict-student-performance-from-game-play/discussion/420528)**

* **アプローチ:** XGBoostとTransformer+GRUのアンサンブル。チーム内で役割分担してモデル開発。
* **アーキテクチャ:**
    * GBDT: XGBoost（質問ごとにモデル構築、計90モデル）。
    * NN: GRU (3層) + TransformerEncoder。RIIIDコンペの解法に触発。
* **アルゴリズム:**
    * GBDT: XGBoost。
    * NN: BCE Loss (multi-task learning)。
    * アンサンブル: XGBoostとNNの予測をアンサンブル。
* **テクニック:**
    * **データ:** Field Day Labの公開外部データを活用（CV+0.0005, LB+0.002）。異常なインデックスを持つセッション（約250件）を特定し、並び順を保持/修正する処理を実装。
    * **特徴量 (GBDT):** セッション長、オブジェクトクリックベースの特徴量（Instance features）、Magic bingo特徴量（公開カーネル参考）、標準的なカウント/集約特徴量（`fqid`, `text_fqid`, `room_fqid`, `level`, `event_comb`）、indexのビニング、`elapsed_time_diff`のfirst/sum、ホバー時間の集約、レベルグループ間のトップ特徴量、過去の質問の予測確率（Meta features, CV+0.001）。
    * **特徴量 (NN):** `index`, `time_diff`, `room_coor_x/y`, `screen_coor_x/y`, `hover_duration` (数値) と `level`, `event_name`, `name`, `text`, `fqid`, `room_fqid`, `text_fqid` (カテゴリ)。
    * **特徴量選択 (GBDT):** Gain/SHAPで重要度0のものを削除。重複特徴量、Nullが多い特徴量を除去。
    * **学習 (NN):** レベルグループごとにモデル構築。過去レベルのシーケンスも入力。Multi-task learning（主目的レベルの質問＋他レベルの質問を補助タスクとして予測、CV+0.002）。NNのEmbeddingを抽出し、集約特徴量と結合してGBDTで再学習するアプローチも試行。
    * **推論:** 単一閾値。

**[9位](https://www.kaggle.com/competitions/predict-student-performance-from-game-play/discussion/420046)**

* **アプローチ:** LightGBMとCatBoostのアンサンブル。各質問ごとにモデル構築。CVスコアを重視。
* **アーキテクチャ:** LightGBM, CatBoost。
* **アルゴリズム:** LightGBM, CatBoost。
* **アンサンブル:** LightGBMとCatBoostの単純平均。
* **テクニック:**
    * **データ:** 外部データは不使用？
    * **特徴量:** 「ある地点から別の地点までどれだけ時間がかかったか」を重視。チェックポイント特徴量（ほぼ全ユーザーが通過するイベント間の時間/クリック数）を作成。テキストA→B、fqid A→B、部屋A→Bなどのパターンで多数作成。レベルごとの経過時間、平均座標なども使用。特徴量数はレベルグループが上がるにつれて増加（最終的に18k超）。
    * **検証:** 10-Fold StratifiedKFold（ユーザーの正解数で層化）。
    * **特徴量選択:** 各Foldで学習したモデルのImportanceに基づき、Top500を選択して再学習（Leakage防止）。
    * **学習:** 過去の質問の予測確率を特徴量として使用（CV+0.001）。
    * **推論:** 単一閾値。LightGBMの推論は`lleaves`ライブラリで高速化。

**[10位](https://www.kaggle.com/competitions/predict-student-performance-from-game-play/discussion/420132)**

* **アプローチ:** 複数のXGBoost、LightGBM、NN（LSTM+Transformer）によるStage 1モデルと、それらをスタッキングするStage 2モデル（MLP、ロジスティック回帰）、最後に平均化と閾値最適化を行うStage 3構成。
* **アーキテクチャ:**
    * Stage 1: XGBoost (4種), LightGBM (1種), NN (LSTM+Transformer, 1種)。
    * Stage 2: MLP, Logistic Regression (各質問ごと)。
* **アルゴリズム:** XGBoost, LightGBM, NN, MLP, Logistic Regression。
* **アンサンブル:** スタッキング + 平均化。
* **テクニック:**
    * **データ:** 外部データは不使用？
    * **特徴量 (XGBoost):** `elapsed_time_diff`と`hover_duration`の集約特徴量がベース。過去レベルグループの特徴量や予測確率も使用。Numpy/Numbaで特徴量生成を高速化。
    * **検証:** 5-Fold StratifiedGroupKFold。
    * **スタッキング:** Stage 1モデルのOOF予測をStage 2モデル（シンプルなMLPとロジスティック回帰）への入力とする。過学習を避けるためシンプルな構造を採用。
    * **閾値最適化:** Stage 2の予測の平均値に対し、質問ごとに閾値を最適化（`scipy.optimize.minimize`のPowell法を使用）。CV+0.008の大幅改善。
    * **効率化:** 特徴量生成にNumbaを使用し、Polars使用時（2時間）から13分に短縮。
