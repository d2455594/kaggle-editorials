---
tags:
  - Kaggle
  - パーキンソン病
  - SMAPE
  - LightGBM
  - Transformer
startdate: 2023-02-17
enddate: 2023-05-19
---
# Parkinson's Freezing of Gait Prediction
[https://www.kaggle.com/competitions/tlvmc-parkinsons-freezing-gait-prediction](https://www.kaggle.com/competitions/tlvmc-parkinsons-freezing-gait-prediction)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、パーキンソン病患者の日常生活中の**ウェアラブルセンサー（主に加速度計）データ**を用いて、歩行中に足が意図せず一時的に止まってしまう**「すくみ足 (Freezing of Gait, FOG)」**の発生を検出・予測する機械学習モデルを開発することです。予測対象となるFOGには、歩き始めのすくみ足 (`StartHesitation`)、方向転換時のすくみ足 (`Turn`)、直線歩行中のすくみ足 (`Walking`) の3種類があります。
* **背景:** すくみ足はパーキンソン病患者によく見られる症状であり、突然発生して数秒から数分間続くことがあります。これは転倒の主な原因となり、患者の移動能力やQOL（生活の質）を著しく低下させます。ウェアラブルセンサーを用いてFOGの発生をリアルタイムに近い形で検出し、患者に警告を発したり、運動療法を促したり、あるいは症状の客観的な評価や治療効果のモニタリングに役立てることが期待されています。
* **課題:** FOGの発生パターンは個人差が大きく、また同じ個人内でも状況によって変動します。センサーデータには、FOG以外の様々な日常動作（歩行、静止、方向転換など）やノイズが含まれており、その中からFOG特有のパターンを正確に識別する必要があります。特に、FOGが始まる直前の予兆を捉えて「予測」することは困難な課題です。また、リアルタイムでの応用を視野に入れると、モデルの計算効率も考慮する必要があります。データセットには異なる条件下で収集された2種類（`tdcsfog`, `defog`）が含まれており、これらのデータの特性の違いに対応することも課題となります。

**データセットの形式 (Dataset Format)**

提供される主なデータは、被験者が装着したウェアラブルセンサー（主に足首や腰に装着された加速度計）の時系列データと、ビデオ分析等によってラベル付けされたFOGイベントの発生時刻です。

1.  **センサーデータ (Training/Test Data):**
    * `train/` および `test/` ディレクトリ内に、データソース（`tdcsfog`, `defog`）ごとのサブディレクトリがあり、その中に個々の試行 (`Id`) に対応する `csv` ファイルが格納されています。
    * 各 `csv` ファイルには、時系列のセンサーデータが含まれます。
        * `Time`: 記録開始からの経過時間（ミリ秒）。
        * `AccV`: 垂直方向の加速度。
        * `AccML`: 左右方向（内側/外側）の加速度。
        * `AccAP`: 前後方向の加速度。
        * サンプリング周波数は `tdcsfog` (128Hz) と `defog` (100Hz) で異なります。
2.  **ラベルデータ (Training Labels):**
    * `train/` 内の各 `csv` ファイルには、各タイムステップ (`Time`) におけるFOGイベントの発生状況を示す**ターゲット変数**のカラムが含まれます。
        * `StartHesitation`: 歩き始めのFOGが発生していれば1、そうでなければ0。
        * `Turn`: 方向転換中のFOGが発生していれば1、そうでなければ0。
        * `Walking`: 直線歩行中のFOGが発生していれば1、そうでなければ0。
    * `defog` データには追加のメタ情報カラムも含まれます。
        * `Valid`: ラベルの信頼性を示すフラグ。
        * `Task`: 被験者が特定のタスクを実行中かを示すフラグ。
        * `Event`: ラベル付けに使用されたイベントの種類を示す（訓練データのみ）。
3.  **メタデータ:**
    * `subjects.csv`: 被験者の属性情報（年齢、性別、パーキンソン病罹病期間、運動機能評価スコア (UPDRS III)、すくみ足に関する自己評価 (NFOG-Q) など）。
    * `tdcsfog_metadata.csv`, `defog_metadata.csv`: 各試行 (`Id`) のメタデータ。
    * `tasks.csv`: 各試行で実施されたタスクの内容説明。
    * `events.csv`: `defog` データのラベル付けイベントの詳細情報。
4.  **未ラベルデータ (`notype`):**
    * `defog/notype` ディレクトリには、FOGラベルが付与されていないセンサーデータが含まれます。これは、自己教師あり学習や擬似ラベリングなどの手法に利用可能です。
5.  **`sample_submission.csv`**:
    * 提出フォーマットのサンプル。`Id`（試行IDとタイムステップを結合した形式、例: `003f117e14_0`）と、`StartHesitation`, `Turn`, `Walking` それぞれの予測確率（0から1の実数）の列を持ちます。

**評価指標 (Evaluation Metric)**

* **指標:** **イベントベースのAverage Precision (AP) at $IoU \in [0.3, 0.8]$ (mAP)**
* **計算方法:**
    1.  モデルは各タイムステップに対して、3種類のFOG (`StartHesitation`, `Turn`, `Walking`) それぞれの発生確率を予測します。
    2.  予測確率にある閾値を適用し、連続する陽性予測を1つのFOGイベント区間（開始時刻、終了時刻）として検出します。
    3.  検出された各予測イベント区間と、正解ラベル中の各真のイベント区間との間で、**Intersection over Union (IoU)** を計算します。IoUは、2つの区間の重なり具合を示す指標です (重なり部分の時間長 / 和集合部分の時間長)。
    4.  複数のIoU閾値（0.3, 0.35, 0.4, ..., 0.75, 0.8 の計11段階）それぞれについて、以下を行います:
        * IoUが閾値以上の予測区間をTrue Positive (TP)、閾値未満または対応する真区間がない予測区間をFalse Positive (FP)、検出されなかった真区間をFalse Negative (FN) とします。
        * 予測確率の閾値を様々に変化させ、適合率 (Precision = TP / (TP + FP)) と再現率 (Recall = TP / (TP + FN)) を計算し、適合率-再現率曲線 (Precision-Recall Curve) を描きます。
        * このPR曲線の下面積が、そのIoU閾値における Average Precision (AP) となります。
    5.  11段階のIoU閾値におけるAPを平均します。
    6.  この平均APを、3種類のFOGイベント (`StartHesitation`, `Turn`, `Walking`) それぞれについて計算します。
    7.  最後に、3種類のイベントの平均APをさらに平均した値 (mean Average Precision, mAP) が最終的なスコアとなります。
* **意味:** この指標は、モデルがFOGイベントの「区間」をどれだけ正確に検出できているかを評価します。単なる時点ごとの分類精度ではなく、イベントの開始・終了タイミングを含めた区間全体の一致度が重要になります。複数のIoU閾値で評価することにより、様々なレベルの一致度に対するモデルの性能を総合的に評価します。スコアは0から1の範囲を取り、**高い**ほど良い性能を示します。

要約すると、このコンペティションは、ウェアラブルセンサーの時系列データから3種類のパーキンソン病すくみ足イベントの発生区間を予測するタスクです。データは加速度計データとイベントラベルで構成され、性能はイベント区間の検出精度を測るmAP（高いほど良い）によって評価されます。

---

**全体的な傾向**

このコンペティションでは、パーキンソン病患者の加速度センサーデータから「すくみ足（FOG）」イベントを検出・予測するため、時系列データを効果的に扱えるモデルアーキテクチャと、データ処理戦略が鍵となりました。

1.  **モデルアーキテクチャ:**
    * **RNN (GRU/LSTM):** 時系列データの長期依存関係を捉える能力から、多くのチームが **GRU** や **LSTM**（特に双方向: BiGRU/BiLSTM）を採用しました。Residual Connectionを追加したカスタムGRU（4位）も有効でした。
    * **Transformer:** Attention機構を用いた **Transformer Encoder** も非常に強力で、1位や3位の解法で中心的な役割を果たしました。パッチ化（時系列データを短い区間に分割）や位置エンコーディングと組み合わせて使用されました。
    * **CNN (1D/2D):** 1D CNNは特徴抽出器としてRNN/Transformerの前段に置かれたり（3位, 5位のWaveNet）、1D ResNetとして単独で用いられたりしました（8位）。また、時間周波数表現（スペクトログラム、ウェーブレット）を入力とする **2D CNN** や **U-Net** 構造も有効でした（6位, 10位の1D U-Net+SE）。
2.  **データ処理と特徴量:**
    * **入力:** 主に3軸加速度データ (`AccV`, `AccML`, `AccAP`) が使用されました。
    * **正規化:** 各試行（Id）ごとにデータを正規化（StandardScaler、RobustScaler、平均・標準偏差正規化など）するのが一般的でした。
    * **シーケンス処理:**
        * **パッチ化/チャンク化:** 長い時系列データを固定長のパッチやチャンクに分割してモデルに入力（1位, 3位, 5位, 6位など）。
        * **シーケンス長:** 訓練時と推論時で異なるシーケンス長を使う戦略（特に2位）が有効でした。訓練時は計算効率のために短く（例: 1000-5000）、推論時はより長いコンテキストを利用するために長く（例: 3000-30000）設定し、予測は中央部分を利用するなどの工夫が見られました。
        * **ダウンサンプリング/リサンプリング:** サンプリング周波数を下げる（4位）または統一する（5位, 6位）処理も行われました。
    * **特徴量エンジニアリング:** 加速度データの差分、累積和、時間に関する特徴量（経過時間、Sin/Cosエンコーディング）などが追加されました（2位, 6位, 10位）。スペクトログラムやウェーブレット変換も特徴抽出の一環と見なせます（6位）。
3.  **データセット戦略:**
    * **tdcsfog vs defog:** 2つのデータソースを別々にモデル化するチーム（1位, 2位, 6位）と、統合して単一モデルで扱うチーム（3位, 4位, 5位, 8位, 10位）に分かれました。
    * **未ラベルデータ (`notype`):** `defog` の未ラベルデータを **擬似ラベリング**（2位, 6位）や**自己教師あり事前学習**（5位）に活用する試みが見られました。
4.  **学習・推論テクニック:**
    * **CV:** 被験者ID (`Subject`) でグループ化し、層化k分割交差検証 (StratifiedGroupKFold) を行うのが標準的でした。
    * **損失関数:** Binary Cross Entropy (BCE) が基本。ラベルの有効性（`Valid`, `Task`）を考慮してマスクする処理（2位, 4位）も行われました。
    * **Augmentation:** 時間伸縮、ノイズ付加、左右反転（AccMLの符号反転）、スケール変更など、様々なデータ拡張が試されました（3位, 6位, 10位）。
    * **アンサンブル:** 複数モデル（異なるアーキテクチャ、Fold、シード、ターゲット重み、スナップショットなど）の予測確率を平均化するアンサンブルが一般的でした。
    * **ターゲット解像度低減:** 1位チームはターゲットラベルの解像度を下げる（最大値プーリング）ことで性能を向上させました。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/tlvmc-parkinsons-freezing-gait-prediction/discussion/416026)**

* **アプローチ:** Transformer Encoder + BiLSTM。**パッチ化**と**ターゲット解像度低減**。tdcsfog/defog別モデル。
* **アーキテクチャ/アルゴリズム:** Transformer Encoder (L=5, H=6, D=320/256) + BiLSTM (L=2)。TensorFlow実装。
* **テクニック:**
    * **入力:** 3軸加速度。各シーケンスを正規化。
    * **パッチ化:** 時系列を固定長パッチ (サイズ18/14) に分割。
    * **ターゲット処理:** ターゲットラベルも同じパッチサイズで分割し、各パッチ内の最大値を取ることで解像度を低減。推論時に `tf.tile` で元の解像度に戻す。
    * **データ:** tdcsfogとdefogで別モデル。メタデータ未使用。
    * **学習:** Adam (カスタムSchedule), BCE Loss (マスク考慮)。位置エンコーディングにランダムロールAugmentation。

**[2位](https://www.kaggle.com/competitions/tlvmc-parkinsons-freezing-gait-prediction/discussion/416057)**

* **アプローチ:** GRUモデル。**訓練/推論時のシーケンス長変更**。**擬似ラベリング**。ターゲット別重み付け学習アンサンブル。tdcsfog/defog別モデル。
* **アーキテクチャ/アルゴリズム:** BiGRU。PyTorch実装。BCEWithLogitsLoss, AdamW, Linear Warmupスケジューラ。
* **テクニック:**
    * **シーケンス長:** 訓練時は短いシーケンス長 (tdcsfog: 1000, defog: 5000)、推論時は長いシーケンス長 (tdcsfog: 3000/5000, defog: 15000/30000) を使用し、中央部分の予測を利用。
    * **データ:** tdcsfogとdefogで別モデル。加速度+差分+累積和特徴量。RobustScaler/StandardScalerで正規化。
    * **擬似ラベリング:** defogの `notype` データに対し、`Event` カラム情報を用いてハードラベルを作成し、学習に利用（複数ラウンド実施）。
    * **アンサンブル:** ターゲットごとに損失の重みを変えて学習したモデル（例: StartHesitationの重みを0.6、他を0.4）を複数作成し、それぞれの得意なターゲット予測をアンサンブル。

**[3位](https://www.kaggle.com/competitions/tlvmc-parkinsons-freezing-gait-prediction/discussion/417717)**

* **アプローチ:** Transformer (DeBERTa/ViT) + RNN (LSTM/GRU)。パッチ化。データ拡張。tdcsfog/defog統合モデル。
* **アーキテクチャ/アルゴリズム:** Transformer Encoder (DeBERTa, ViT, ViT Relative Positional Encoding) + LSTM/GRU (通常1層、双方向かは不明)。
* **テクニック:**
    * **パッチ化:** 時系列データをパッチ (サイズ7-13) に分割。シーケンス長は192-384パッチ。
    * **データ拡張:** ストレッチング、クロッピング、アブレーション（一部データ欠損）、累積ガウスノイズなど、重度のデータ拡張を適用。
    * **データ:** tdcsfog/defog統合。加速度データを使用。

**[4位](https://www.kaggle.com/competitions/tlvmc-parkinsons-freezing-gait-prediction/discussion/416410)**

* **アプローチ:** **Residual BiGRU** (カスタムアーキテクチャ)。tdcsfog/defog統合モデル。ダウンサンプリング。
* **アーキテクチャ/アルゴリズム:** 入力FC + LayerNorm + ReLU + (ResidualBiGRUブロック) x N層 + 出力FC。ResidualBiGRUブロックは BiGRU + FC + LayerNorm + ReLU + FC + LayerNorm + ReLU + Skip Connection で構成。PyTorch実装。CrossEntropyLoss (4クラス: 3種FOG + No-Activity), Rangerオプティマイザ, Cosine Annealingスケジューラ。
* **テクニック:**
    * **データ:** tdcsfog/defog統合。加速度データのみ使用。50Hzにダウンサンプリング。標準化。
    * **学習:** バッチサイズ1で全シーケンスを入力。defogの未ラベル区間 (`Valid`=False or `Task`=False) は入力には含めるが損失計算からは除外。
    * **その他:** メタデータ（年齢など）をGRUの初期隠れ状態に利用するモデルも試行（スコアはやや劣る）。

**[5位](https://www.kaggle.com/competitions/tlvmc-parkinsons-freezing-gait-prediction/discussion/418275)**

* **アプローチ:** **WaveNet + GRU**。**自己教師あり事前学習**。tdcsfog/defog統合モデル。
* **アーキテクチャ/アルゴリズム:** 1D CNN (WaveNetブロック: Dilated Conv) + BiGRU + Linear。PyTorch実装。
* **テクニック:**
    * **データ:** tdcsfog/defog統合。加速度データのみ。tdcsfogデータを100Hzにリサンプリング（ただしバージョン依存性に注意）。
    * **シーケンス処理:** 固定長のウィンドウ (サイズ2000、重複500) で分割して学習。推論時はより長いウィンドウ (16000/20000) を使用。
    * **事前学習:** 未ラベルデータ (`notype` 及びラベル付きデータの一部) を用い、時系列の次の値を予測するタスクでWaveNet部分を事前学習。
    * **アンサンブル:** 事前学習あり/なし、検証データ設定違いなど、複数の学習済みモデルを平均アンサンブル。
    * **推論:** ONNX/OpenVINOに変換しCPUで高速化。

**[6位](https://www.kaggle.com/competitions/tlvmc-parkinsons-freezing-gait-prediction/discussion/415992)**

* **アプローチ:** **スペクトログラム/ウェーブレット変換 + 2D CNN/U-Net + Transformer** および **1D CNN** の多様なモデルアンサンブル。
* **アーキテクチャ/アルゴリズム:**
    * 2D系: STFT/CWT + ResNet18/34 (+ U-Net Decoder) + Transformer Encoder (Attention Bias付き)。
    * 1D系: 1D CNN。
* **テクニック:**
    * **入力:** 加速度3軸 + 時間特徴量 (`pct_time`) + メタデータ (1D CNNのみ)。
    * **スペクトログラム/ウェーブレット:** STFT/CWTで時間周波数表現に変換。時間次元はダウンサンプリングされる（予測は0.5秒ウィンドウ単位）。高周波ビンは破棄。周波数エンコーディングも入力に追加。
    * **Transformer Attention Bias:** 時間的に近い部分に高いAttention重みが向くようにバイアスを追加。
    * **データ拡張:** Audiomentationsライブラリを使用（時間伸縮、ノイズ、ピッチシフトなど）+ スケール変更、時間特徴量シフト。
    * **CV:** ネストしたCV (Outer 4-fold, Inner 4-fold)。
    * **アンサンブル:** 複数モデルの重みをGP最適化 (Bayesian Optimization) で決定。外れ値と思われる被験者 (`2d57c2`) の損失をダウンスケール。

**[8位](https://www.kaggle.com/competitions/tlvmc-parkinsons-freezing-gait-prediction/discussion/416021)**

* **アプローチ:** **1D-ResNet**。公開ノートブックベース。5-Fold CVアンサンブル。
* **アーキテクチャ/アルゴリズム:** 1D-ResNet。PyTorch実装。
* **テクニック:**
    * **データ:** tdcsfog/defog統合。加速度データのみ。
    * **シーケンス処理:** 1000msのカットを使用、未来50msを予測する設定（詳細は元ノートブック参照）。
    * **学習:** ReduceLROnPlateauスケジューラ使用。
    * **アンサンブル:** 5-Fold CVで学習したモデルを平均アンサンブル。

**[10位](https://www.kaggle.com/competitions/tlvmc-parkinsons-freezing-gait-prediction/discussion/416513)**

* **アプローチ:** **1D U-Net + Squeeze-and-Excitation (SE)**。長いコンテキスト。データ拡張。TTA。
* **アーキテクチャ/アルゴリズム:** 1D Conv U-Net (Encoder/Decoder x5) + SEブロック。
* **テクニック:**
    * **入力:** 加速度3軸 + 時間特徴量 (`NormalizedTime`, `SinNormalizedTime`)。正規化は行わない。
    * **シーケンス長:** 非常に長いコンテキストウィンドウ (10240サンプル) を使用。
    * **データ拡張:** ランダムローパスフィルタ、時間ワープ（線形補間）、左右反転（AccML符号反転）、強度ワープ（平均からの差分にガウスノイズ係数を乗算）、時間特徴量ノイズ。
    * **TTA:** 左右反転、オーバーラップウィンドウ（ストライド1/8）で16回の予測を平均。
    * **データ:** tdcsfog/defog統合。リサンプリングや単位変換は行わない。
    * **アンサンブル:** 同じハイパーパラメータ、同じCV Foldで、異なるランダムシードで学習した2モデルをアンサンブル（モデル選択はCVとLBスコアに基づく）。
