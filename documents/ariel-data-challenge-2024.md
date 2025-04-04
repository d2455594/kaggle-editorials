---
tags:
  - Kaggle
  - 物理学
startdate: 2024-08-02
enddate: 2024-11-01
---
# NeurIPS - Ariel Data Challenge 2024
[https://www.kaggle.com/competitions/ariel-data-challenge-2024](https://www.kaggle.com/competitions/ariel-data-challenge-2024)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、ESA（欧州宇宙機関）の次期宇宙望遠鏡ミッション「Ariel」が観測するであろう**系外惑星のトランジット分光データ**（シミュレーションによって生成）を解析し、惑星大気を通過した光が吸収されて減少する割合を示す**透過スペクトル**（具体的には惑星半径と主星半径の比の二乗、$(R_p/R_s)^2$）を波長ごとに正確に推定し、さらにその**推定値の不確実性（標準偏差$\sigma$）**も定量的に評価するモデルを開発することです。
* **背景:** Arielミッションは、数百から千個規模の系外惑星の大気を系統的に調査し、その組成や多様性を明らかにすることを目的としています。トランジット分光法は、惑星が主星の手前を通過する際に、星の光が惑星大気を通過することで特定の波長が吸収される現象を利用し、大気成分を特定する主要な手法です。このコンペティションは、将来Arielが得るであろう実データに対する高度な解析アルゴリズムの開発を促進し、系外惑星科学の進展に貢献することを目指しています。
* **課題:** 観測データ（シミュレーションデータ）には、惑星大気のシグナルだけでなく、検出器固有のノイズ、時間や波長によって変動する系統誤差（ゲインドリフトなど）、星自身の活動の影響、前景光や背景光などが重畳されています。特に、惑星大気による吸収シグナルは非常に微弱（典型的には$10^{-4} \sim 10^{-5}$オーダー）であるため、これらのノイズや系統誤差を精密に除去・補正し、真のスペクトルを抽出することが極めて重要です。さらに、単にスペクトルを推定するだけでなく、その推定値の信頼度を示す不確実性（エラーバー、$\sigma$）も正確に評価することが求められます。データは時間、波長、センサー上のピクセル位置の3次元にわたる複雑な構造を持っています。

**データセットの形式 (Dataset Format)**

提供されるデータは、Ariel望遠鏡の観測を模倣したシミュレーションデータと、対応する真の透過スペクトルです。

1.  **シミュレーション観測データ (Training/Test Data):**
    * `train/` および `test/` ディレクトリ内に、各惑星 (`planet_ID`) ごとの観測データが格納されています。
    * 各惑星ディレクトリ内には、複数の観測チャネル（例: `AIRS-CH0`, `FGS1`）に対応する時系列データが `parquet` 形式で保存されています（例: `AIRS-CH0.parquet`）。
    * 各 `parquet` ファイルは、複数の時間ステップ (`t_obs`) におけるセンサーの読み出しデータを格納しており、各時間ステップでセンサー上の各ピクセル (`x`, `y`) における光子数 (`photon_counts`) などが記録されています。これがモデルの主たる入力となります。
    * データには、ホットピクセル、デッドピクセル、ゲインドリフト、PSF（点像分布関数）の変動、前景光といった現実的なノイズや系統誤差がシミュレーションによって付加されています。
2.  **正解ラベルデータ (Training Labels):**
    * `train_labels.csv`: 学習用データに対応する真のスペクトル情報。
        * `planet_ID`: 惑星ID。
        * `wavelength`: 波長 (nm)。
        * `transit_depth`: 真の透過スペクトル（$(R_p/R_s)^2$）。これが予測対象$\mu$の真値$\mu_{true}$。
        * `transit_depth_sigma`: 真のスペクトルの不確実性$\sigma_{true}$。**注意:** これはターゲット変数ではなく、参考情報やモデル構築に利用される可能性があります。ターゲットは$\mu$とその予測誤差$\sigma_{pred}$です。
3.  **サンプル提出ファイル (Sample Submission):**
    * `test_labels_sample.csv`: 提出ファイルのフォーマットを示すサンプル。
        * `planet_ID`, `wavelength`, `transit_depth`, `transit_depth_sigma` のカラムを持ちます。参加者は、`test.csv` に対応する各惑星の各波長について、推定したスペクトル値$\mu_{pred}$を `transit_depth` 列に、その推定値の標準偏差$\sigma_{pred}$を `transit_depth_sigma` 列に記入して提出します。

**評価指標 (Evaluation Metric)**

* **指標:** **Ariel Likelihood Metric (アリエル尤度メトリック)**
* **計算方法:** この指標は、予測されたスペクトル分布（平均$\mu_{pred}$、標準偏差$\sigma_{pred}$の正規分布）が、真のスペクトル値$\mu_{true}$をどれだけよく説明できるかを示す対数尤度に基づいています。具体的には、各惑星の各波長について以下の値を計算し、全惑星・全波長で合計します。
$$\text{Score per point} = - \frac{1}{2} \left( \frac{\mu_{true} - \mu_{pred}}{\sigma_{pred}} \right)^2 - \log(\sqrt{2\pi}\sigma_{pred})$$
$$\text{Final Score} = \sum_{\text{all planets}} \sum_{\text{all wavelengths}} \text{Score per point}$$
* **意味:**
    * 第1項$- \frac{1}{2} \left( \frac{\mu_{true} - \mu_{pred}}{\sigma_{pred}} \right)^2$ は、予測値と真値の差（$\sigma_{pred}$で正規化）が小さいほど0に近づきます（ペナルティが小さい）。
    * 第2項$- \log(\sqrt{2\pi}\sigma_{pred})$は、予測の不確実性$\sigma_{pred}$が小さいほど大きな値（対数なので、より小さい負の値）になります。
    * この指標は、単に予測値$\mu_{pred}$が真値$\mu_{true}$に近いだけでなく、予測の不確実性$\sigma_{pred}$が適切であること（真値が予測分布の信頼区間内に尤もらしく収まること）を同時に要求します。$\sigma_{pred}$が過剰に小さいと第1項のペナルティが大きくなり、過剰に大きいと第2項のペナルティが大きくなります。
    * スコアは**大きい**ほど、より良い予測（正確な$\mu_{pred}$と適切な$\sigma_{pred}$の両立）であることを示します。

要約すると、このコンペティションは、シミュレーションされた系外惑星のトランジット分光データから、惑星大気の透過スペクトルとその不確実性を推定するタスクです。データは時系列のセンサー画像データであり、性能は予測分布の尤度に基づくカスタムメトリック（大きいほど良い）で評価されます。

---

**全体的な傾向**

このコンペティションでは、非常にノイズが多く系統誤差を含む時系列分光データから微弱な信号（透過スペクトル）とその不確実性を精密に抽出する必要があり、多段階の処理パイプラインと多様なモデリングアプローチが用いられました。

1.  **データ前処理:** ノイズ除去と系統誤差補正が極めて重要でした。
    * **空間的集計:** センサーの中央領域のピクセル値を選択・合計して各波長のライトカーブを作成。SNRに基づく重み付けも有効。
    * **前景光除去:** センサー端のピクセル情報を用いて前景光を推定し除去する手法が特に1位チームで効果を発揮。
    * **ピクセル処理:** ホットピクセルは保持し、デッドピクセルは補間する戦略が一般的。
2.  **トランジット区間検出:** ライトカーブからトランジットの開始・中間・終了時刻を正確に特定。微分、畳み込み、曲線フィッティングなどが利用された。
3.  **系統誤差（ドリフト）モデリング:** 時間や波長に依存する変動（ゲインドリフトなど）をモデル化し除去（デトレンド）。
    * **多項式フィッティング:** 最も一般的なアプローチ。時間に対して2～5次の多項式をフィット。波長依存性を考慮したモデル（例:$(1+f(t) \cdot g(\lambda))$）も有効（特に1位）。
    * **ベイジアン推論 / ガウシアンプロセス (GP):** ドリフトを時間・波長の滑らかな関数としてモデル化（2位）。
4.  **スペクトル ($\mu$) 推定:** デトレンド後のライトカーブから各波長のトランジット深さを推定。
    * **直接フィッティング:** ドリフトモデルとトランジットモデル（箱型関数など）を組み合わせ、ライトカーブ全体をフィット (`curve_fit`, OLS, 最適化探索など)。
    * **ベイジアン推論 / GP:** 他のパラメータと同時にスペクトルも確率的に推定（2位）。
5.  **スペクトル ($\mu$) 後処理/平滑化:** ノイズの影響を低減し、物理的に滑らかなスペクトルを得るための処理。
    * **移動平均/中央値、Savitzky-Golayフィルタ:** 基本的な平滑化。
    * **ガウシアンプロセス (GP):** スペクトル形状をGPでモデル化し平滑化（1位、2位）。
    * **次元削減 (PCA, NMF, AutoEncoder):** データ全体からスペクトルの主要成分を抽出し、ノイズを除去しつつ再構成（1位、3位、8位）。物理的な吸収特徴（H₂O, CO₂, CH₄など）に対応する成分が見出されることも。
6.  **不確実性 ($\sigma$) 推定:** 各波長の$\mu$推定値の標準偏差を推定。評価指標上、非常に重要。
    * **フィッティング誤差/共分散:** OLSや最適化のパラメータ誤差から算出（8位）。
    * **ブートストラップ:** データリサンプリングによる推定値のばらつき（1位）。
    * **GPによる予測:** GPモデルの予測分散（1位、2位）。
    * **経験的/ヒューリスティックモデル:**$\mu$の特徴量（標準偏差、深さ、MAEなど）から$\sigma$を回帰予測（多項式、GBDT）またはルールベースで決定（3位、4位、6位、7位、8位、9位）。
7.  **シミュレータコードの活用:** データ生成に使われたExoSim2/TauREx3のコードを解析し、ゲインドリフトの正確な関数形や前景光の存在といった「隠れた仕様」を発見・利用することが、特に1位チームの成功要因となった。
8.  **アンサンブル:** 異なる$\mu$推定手法（GP, AE, NMF, 多項式）や$\sigma$推定手法の結果を組み合わせる。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/ariel-data-challenge-2024/discussion/544317)**

* **アプローチ:** シミュレータコード解析に基づく精密な前処理・モデリング + GP/AE/NMFアンサンブル。
* **アーキテクチャ/アルゴリズム:** 多項式フィット (ゲインドリフト/スペクトル), ブートストラップ, ガウシアンプロセス (GP), AutoEncoder (AE), 非負値行列因子分解 (NMF)。
* **テクニック:**
    * **前処理:** AIRS-CH0のみ使用。ホットピクセル保持。**前景光処理**（センサー端で推定し中央から除去）。
    * **$\mu$推定:** トランジット区間特定後、**ゲインドリフト関数形 ($(1+f(t) \cdot g(\lambda))$)** を組み込んだモデルで直接フィッティング（2段階フィットで安定化）。
    * **$\sigma$推定:** **ブートストラップ**で$\mu$推定のばらつきから算出。
    * **$\mu$後処理:** **GP, AE(隠れ層4次元), NMF(ランク5)** の3モデルで$\mu$を平滑化し、**6:2:2でアンサンブル**。
    * **$\sigma$後処理:** 定数値、惑星ごとの$\mu$の標準偏差、GPによる$\sigma$予測値の3つを加重平均。

**[2位](https://www.kaggle.com/competitions/ariel-data-challenge-2024/discussion/543853)**

* **アプローチ:** **純粋なベイジアン推論 + ガウシアンプロセス (GP)** による統合的モデリング。
* **アーキテクチャ/アルゴリズム:** ガウシアンプロセス (GP), ベイジアン推論, 主成分分析 (PCA), KISS-GP (スパース化)。
* **テクニック:**
    * **統合モデル (Prior):** ノイズ、星のスペクトル、時間ドリフト(1D GP)、時間・波長ドリフト(2D GP)、トランジット形状（時間、幅をフィット）、トランジット深さ（平均値+波長依存GP+FGS用項+PCA成分）を全て確率分布・GPで事前定義。
    * **推論 (Posterior):** 観測データに基づき、ベイズの定理を用いて全パラメータの事後分布を推定。$\mu$と$\sigma$は事後分布から直接得る。非線形性は反復線形化で対応。
    * **GPカーネル:** 複数のスケールを持つRBFカーネルを使用。ハイパーパラメータは訓練データで調整。
    * **PCA:** 粗いフィット結果からPCAでスペクトルの共通形状を抽出し、モデルに組み込む。
    * **事後補正:**$\mu$の平均値に対するスケーリング（1.0064、前景光未処理の影響か）と、$\sigma$に対するスケーリングを適用。

**[3位](https://www.kaggle.com/competitions/ariel-data-challenge-2024/discussion/543944)**

* **アプローチ:** 多項式フィッティング + ヒューリスティック後処理 + PCAによる最終精密化。
* **アーキテクチャ/アルゴリズム:** 多項式フィッティング (`scipy.optimize.curve_fit`), Savitzky-Golayフィルタ, 主成分分析 (PCA)。
* **テクニック:**
    * **前処理:** ホットピクセル保持。高強度ピクセルのみ使用。時間ビニング縮小。SNRに基づく波長重み付け。デッドピクセル位置に基づくペナルティ。
    * **$\mu$抽出:** 時間方向に多項式フィット（次数2,4,5を評価しペナルティ付きRMS誤差で選択）。波長方向に移動平均（N=8または20）した信号を使用。トランジット区間はガウス微分畳み込みで検出。特定波長域で狭い移動平均窓を使用。大域的な波長範囲で求めた多項式を固定し、狭い範囲で再フィット。
    * **$\mu$後処理:** スペクトルのダイナミクス（相関で判定、Taurex生成データも利用）に応じて処理分岐。低ダイナミクスは平均予測線、高ダイナミクスは生スペクトルを調整（SGフィルタ、オフセット補正、クリッピング、高波長域線形補間）。
    * **$\sigma$推定:** 波長依存の経験的構築。$\mu$の深さや平均からの乖離、ダイナミクスに基づきパラメータ変更。
    * **最終精密化:** 全惑星の推定$\mu$に**PCA**を適用し、主要5成分で再構成してノイズ除去。

**[4位](https://www.kaggle.com/competitions/ariel-data-challenge-2024/discussion/544471)**

* **アプローチ:** 多項式フィッティング (Ingress/Egress別) + 2D多項式シグマ推定 + 2D多項式回帰による精密化 + アンサンブル。
* **アーキテクチャ/アルゴリズム:** 多項式フィッティング (カスタムBinary Search, Numba), 2D多項式回帰 (TensorFlow, AdamW)。
* **テクニック:**
    * **トランジット区間検出:** 時間軸で2次多項式x2+線形関数をフィットし、RMSE最小となる接続点p1,p2,p3,p4を探索。
    * **$\mu$抽出 (ベース):** Ingress側(開始～p1, p2～中間)とEgress側(中間～p3, p4～終了)で独立に2次多項式をフィットし、$\mu$を推定 (Sergei手法改良)。Ingress側 (pred1) を重視 (1.9:1)。
    * **$\sigma$推定 (ベース):**$\mu$の特徴量 (std(pred), distance(pred1,pred2)) から**2D多項式**で$\sigma$をフィッティング。
    * **$\mu$後処理 (ベース):**$\mu$の std に応じて波長平滑化窓幅を変更。低std領域で平均値に揺らぎ追加。高波長域を平均値で置換。
    * **$\mu$精密化 (2D多項式回帰):** 時間と波長に対する**2D多項式**でライトカーブ全体をフィットするモデルをTensorFlowで構築・学習（複数惑星並列処理）。2種類の多項式次数設定でモデル作成。
    * **アンサンブル:** ベースモデル、2D多項式モデル1、2D多項式モデル2の$\mu$を等価重みで平均。$\sigma$はベースモデルとモデル1を使用 (0.2:1)。

**[5位](https://www.kaggle.com/competitions/ariel-data-challenge-2024/discussion/543760)**

* **アプローチ:** 多項式フィッティング + ガウス関数特徴量 + 線形回帰 + **データ拡張**。
* **アーキテクチャ/アルゴリズム:** 多項式フィッティング, ガウス関数フィッティング, 線形回帰 (LR), 全結合NN (σ予測)。TauREx3 (データ拡張)。
* **テクニック:**
    * **前処理:** GPU並列化、補間。
    * **$\mu$抽出:** 多項式フィットでデトレンド。スペクトル形状を**5つのガウス関数**でモデル化し、そのパラメータや集計値を特徴量としてLRで最終的な$\mu$を予測。平均スペクトルを引くなどしてモデルを制約。
    * **$\sigma$推定:**$\mu$の標準偏差との相関を利用 + NNモデル。
    * **データ拡張:** **TauREx3**を用いて、訓練データにない9種類の分子(CO, NH3など)の吸収スペクトルをシミュレーションし、訓練データに加算して学習データを拡張。

**[6位](https://www.kaggle.com/competitions/ariel-data-challenge-2024/discussion/543666)**

* **アプローチ:** 多項式フィッティング + ヒューリスティック特徴量 + CNNアンサンブル。
* **アーキテクチャ/アルゴリズム:** Savitzky-Golay (SG)フィルタ, 多項式フィッティング, 1D CNN (2ヘッド)。
* **テクニック:**
    * **前処理:** ピクセルデータの外れ値を周波数軸ガウス平滑化で除去。
    * **トランジット区間検出:** 時間軸SGフィルタ。
    * **$\mu$抽出 / 特徴量:** 多項式フィットで$\mu$の初期値を推定。物理的な吸収帯とは異なるが、GA等で選択した**最適な波長区間**の$\mu$値や統計量を特徴量として利用。
    * **$\sigma$推定:**$\mu$の最大最小差から回帰推定（スペクトル全体で1つの平均シグマ）。
    * **モデル:**
        * ヒューリスティック: 特徴量の重み付き和。$\mu$の差が大きい場合と小さい場合で処理分岐。
        * CNN: 1D CNN（波長軸畳み込み、窓サイズ21）。2ヘッド出力（$\mu$と$\sigma$）。GaussianLoss。
    * **アンサンブル:** ヒューリスティックモデルとCNNの予測をRMSEに基づき混合。

**[7位](https://www.kaggle.com/competitions/ariel-data-challenge-2024/discussion/543679)**

* **アプローチ:** 多項式フィッティング + ヒューリスティック後処理 + 平坦スペクトル予測。
* **アーキテクチャ/アルゴリズム:** 多項式フィッティング (Sergei手法), Gaussianフィルタ。
* **テクニック:**
    * **トランジット区間検出:** 微分+畳み込み。多項式フィットで改善。
    * **$\mu$抽出:** Sergei手法の多項式フィット（次数2～4を試し、曲率が大きい場合は次数4を選択）。波長0～240のみ平均して初期推定。各領域をガウスフィルタで平滑化。
    * **$\mu$後処理:** 波長軸で畳み込み（窓サイズはMAEに応じて変更）。平均値からのシフトとスケーリング。
    * **$\sigma$推定:** フィット残差のMAEとスペクトルの平坦度（の2成分）からスケーリングで推定。
    * **平坦スペクトル処理:** スペクトルが平坦かどうかを予測し、平坦な場合は平均$\mu$を予測値とし、$\sigma$も低く設定。

**[8位](https://www.kaggle.com/competitions/ariel-data-challenge-2024/discussion/543776)**

* **アプローチ:** 多項式フィッティング (OLS) + SVD後処理 + 経験的シグマ推定。
* **アーキテクチャ/アルゴリズム:** 多項式フィッティング (OLS), Quantile Regression, SVD, 指数平滑化, GLS。
* **テクニック:**
    * **前処理:** 高波長域で空間次元削減。
    * **$\mu$抽出:** トランジット区間検出後、線形方程式としてOLSで直接$\mu$を解く。Quantile Regressionで外れ値除去。波長別に独立してフィット (Model2)。全波長平均でもフィット (Model1)。
    * **$\mu$後処理:** Model2の$\mu$にSVDを適用し主成分で再構成（SNR重み付け）。波長間で指数平滑化（SNR重み付け）。Model1とModel2をアンサンブル (0.6 * Model1 + 0.4 * Model2)。両者の平均値比でバイアス補正。Model1と2の差が大きい場合はModel2を優先。
    * **$\sigma$推定:** Model1とModel2の差、Model2のOLS標準誤差、隣接波長間の最大$\mu$差などを用いたヒューリスティックルールで$\sigma$を決定。

**[9位](https://www.kaggle.com/competitions/ariel-data-challenge-2024/discussion/543983)**

* **アプローチ:** 2種類の$\mu$推定法の組み合わせ + 単純化された$\sigma$推定。
* **アーキテクチャ/アルゴリズム:** 多項式フィッティング (Sergei手法), 差分法, Butterworthフィルタ, Hann窓。
* **テクニック:**
    * **前処理:** ホスト提供のもの（ビン化/ホットピクセル処理省略）。
    * **トランジット区間検出:** 1次/2次微分。
    * **$\mu$抽出:** 2手法を加重平均。(1) 多項式フィット（波長全体(μ_1)と47分割チャンク(μ_47)で実施）。(2) トランジット前後の差分法（短時間窓）。時間軸Butterworthフィルタ、波長軸Hann窓（μ_47の場合）で信号を平滑化。
    * **$\sigma$推定:** 4分割チャンクの$\mu$の標準偏差をそのまま$\sigma$とする。
    * **$\mu$精密化:** 推定された$\sigma$に基づき、$\mu_1$と$\mu_{47}$の加重平均の重みを変更（$\sigma$が大きいほど$\mu_{47}$の重みを増やす）。

**[10位](https://www.kaggle.com/competitions/ariel-data-challenge-2024/discussion/544189)**

* **アプローチ:** 多項式フィッティング (カスタム関数) + ランダムサブサンプリング集計 + GBDTによる$\sigma$推定。
* **アーキテクチャ/アルゴリズム:** 多項式フィッティング (カスタム平滑関数), 勾配ブースティング (GBDT)。
* **テクニック:**
    * **トランジット区間検出:** 微分 + 線形フィット。
    * **$\mu$抽出:** トランジット形状を近似する**カスタム平滑関数**$T(t, p1, p2, \tau, d)$を用いて、$\text{Signal}(t) \approx P(t) \times T(t, ...)$の形で多項式$P(t)$と$\mu=d$をフィット。
    * **ランダムサブサンプリング集計:** 波長方向に移動窓を取り、窓内の波長から**ランダムにサブサンプル**（サイズ `k_binn_wl`）して複数回（15回）フィット。さらに各サブサンプルを小グループに分割してフィット。これらの結果を集計して最終的な$\mu$を得る（詳細な集計方法はコード参照）。ノイズの多い高波長域では処理を簡略化。
    * **$\sigma$推定:** GBDTを用い、$\mu$予測に関する特徴量から$\sigma$を予測（良波長域用とノイズ波長域用の2値を予測）。
    * **アンサンブル:** 異なる `k_binn_wl` パラメータで実行した結果をブレンド。
