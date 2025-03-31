---
tags:
  - Kaggle
startdate: 2023-09-08
enddate: 2023-12-08
---
# Stanford Ribonanza RNA Folding
https://www.kaggle.com/competitions/stanford-ribonanza-rna-folding

**概要 (Overview)**

* **目的:** このコンペティションの目的は、RNA（リボ核酸）分子の**塩基配列**と**実験データ（例：SHAPE-MaP）**に基づいて、特定の条件下（例：DMS、2A3試薬による化学修飾）における各**ヌクレオチド（塩基）の反応性（Reactivity）を予測する**モデルを開発することです。
* **背景:** RNA分子は、タンパク質合成や遺伝子発現制御など、細胞内で多様かつ重要な役割を担っています。その機能は、一次配列（塩基配列）によって決まる立体構造（折り畳み構造）に大きく依存します。RNAの構造を正確に予測することは、生命現象の理解、mRNAワクチンのようなRNAベースの治療法開発、疾患診断などに不可欠です。SHAPE-MaP、DMS、2A3などの実験技術は、RNAの構造を化学反応性の違いとして間接的に測定しますが、完全な3D構造を直接得るわけではありません。AIモデルは、これらの実験データを解釈したり、塩基配列から直接構造や反応性を予測したりするのに役立ちます。Ribonanzaは、このようなRNA構造解析のためのハイスループットな実験・計算プラットフォームです。
* **課題:** RNAの折り畳みは、塩基対形成、スタッキング相互作用、環境因子などが関わる複雑な物理化学的プロセスであり、構造や反応性の正確な予測は困難です。実験的に得られる反応性データにはノイズが含まれる可能性があります。また、モデルは特定の化学プローブ（DMSや2A3）に対応する反応性をヌクレオチドごとに予測する必要があり、これらのプローブはRNA構造の異なる特徴（例：DMSは対合していないAやC、2A3は対合していないGやUを検出しやすい）を反映します。これは、生物学的配列データに対する**回帰（Regression）** タスクであり、各塩基位置における連続的な反応性値を予測します。

**データセットの形式 (Dataset Format)**

提供される主なデータは、RNAの塩基配列、関連する実験データ、および予測対象となる特定の条件下でのヌクレオチドごとの反応性データです。

1.  **トレーニングデータ:**
    * `train_data.parquet`: 主要なトレーニングデータ。各行がRNA分子またはその一部に対応する可能性があります。
        * `sequence_id`: RNA分子/実験の識別子。
        * `sequence`: RNAの塩基配列（A, C, G, Uの文字列）。
        * `experiment_type`: 実験条件やプローブの種類（例: 'DMS_MaP', '2A3_MaP'）。
        * 入力となる実験データ（例: SHAPE-MaPから得られた反応性データなど。各塩基位置に対応する値やエラーを含む可能性がある）。
        * **ターゲット変数**: 予測対象となる特定条件下（DMS_MaP, 2A3_MaP）での反応性データ。通常、配列の各位置に対する反応性（`reactivity`）と関連するエラー（`reactivity_error`）の値のリストまたはベクトルとして提供されます。

2.  **テストデータ:**
    * `test_sequences.parquet`: テスト用のRNA配列データ。`sequence_id`, `sequence`, `experiment_type`, および予測対象となる配列内の位置範囲 (`id_min`, `id_max`) を含みます。テストセットの入力実験データは直接提供されない場合があり、その場合は配列情報から反応性を予測する必要がありますが、プライベートテストセットの構築方法によっては暗黙的に利用される可能性があります。

3.  **`sample_submission.csv`**:
    * 提出フォーマットのサンプル。通常、予測対象となる**各ヌクレオチド位置**に対応する行を持ちます。`id`（`sequence_id` と塩基位置インデックスを組み合わせた一意な識別子）と、予測された反応性 `reactivity` の列が必要です。

**評価指標 (Evaluation Metric)**

* **指標:** **平均絶対誤差 (Mean Absolute Error - MAE)** （特定のターゲットヌクレオチドにおけるDMSと2A3の反応性について計算し、平均したもの）
* **計算方法:**
    1.  コンペティションでは、主に2種類の化学プローブ（DMSと2A3）に対する反応性の予測が評価されます。
    2.  テストデータセット内の予測対象となる各ヌクレオチド位置（`id_min` から `id_max` の範囲）について、予測された反応性と真の反応性の間の絶対誤差を計算します。
        * 絶対誤差 = | 予測された反応性 - 真の反応性 |
    3.  DMSで修飾されるターゲット位置全体での平均絶対誤差（MAE_DMS）と、2A3で修飾されるターゲット位置全体での平均絶対誤差（MAE_2A3）をそれぞれ計算します。
    4.  最終的なスコアは、これら2つのMAEの**平均値**となります。
        * Score = (MAE_DMS + MAE_2A3) / 2
* **意味:** MAEは回帰タスクにおける標準的な指標で、予測誤差の平均的な大きさを測ります。RMSEと比較して、極端な外れ値の影響を受けにくい特徴があります。DMSと2A3という異なるプローブに対するMAEを平均することで、モデルが両方のタイプの実験データを反映する反応性をバランス良く予測できているかを評価します。スコアは**低い**ほど（0に近いほど）、モデルの予測が実際の実験値に近いことを示し、性能が良いと評価されます。

要約すると、このコンペティションは、RNAの塩基配列と実験データから、特定の化学プローブ（DMS, 2A3）に対するヌクレオチドごとの反応性を予測する回帰タスクです。データは配列、実験データ、ターゲット反応性値で構成され、性能は予測値と実測値の間の平均絶対誤差（MAE、DMSと2A3で平均、低いほど良い）によって評価されます。

---
**全体的な傾向**

上位解法では、Transformerベースのアーキテクチャが主流であり、特にRNA配列情報に加えて塩基対確率行列（BPPM）の特徴量を活用するアプローチが多く見られました。BPPMはEternaFoldなどの外部ツールで計算され、モデルに組み込む際には、Attentionメカニズムへのバイアスとして加算したり、畳み込みニューラルネットワーク（CNN）で処理したりする手法が取られました。長い配列への汎化性能を高めるために、相対位置エンコーディング（ALiBi、Dynamic Positional Bias、Relative Positional Encodingなど）が広く採用されました。データの前処理（特にSN比に基づくフィルタリングや重み付け）、モデルのアンサンブル、疑似ラベリング、損失関数の工夫なども重要なテクニックとして用いられています。

**各解法の詳細**

**1位**

- **アプローチ:** Transformerアーキテクチャをベースとし、EternaFoldで計算したBPPMをCNNで処理して特徴量として活用。長い配列への汎化のためにDynamic Positional Biasを導入。アーキテクチャや学習プロセスがわずかに異なる複数のモデルをアンサンブル。
- **アーキテクチャ:** Transformer Encoder（12層）、CNNブロック（BPPM処理用）、オプションでSqueeze-and-Excitation (SE) レイヤー。
- **アルゴリズム:** EternaFold（BPPM計算）、Dynamic Positional Bias（位置エンコーディング）、Self-Attention（BPPM特徴量をAttention値に加算）、AdamWオプティマイザ、One-cycle学習率スケジュール。
- **テクニック:**
    - **データ前処理:** EternaFoldによるBPPM計算、学習可能な埋め込み層（配列、SNフィルター値）、特殊トークン（<start>, <end>）の追加。
    - **モデル:** Transformer Encoderブロックの修正（Self-AttentionブロックでBPPMと配列特徴量の相互作用）、Convolutional Block（2D畳み込み、BatchNorm、活性化関数）、SE Block（オプション）。
    - **学習:** SN比に基づく重み付きサンプリング、追加のSGDファインチューニング、DMSから2A3を予測する別モデル（dms-to-2a3）の学習。
    - **推論:** SNフィルター値を1に固定。
    - **アンサンブル:** 異なるConvolutional Block構造を持つモデル（SEあり/なし）や異なるデータ分割で学習したモデルを含む合計27モデルの加重平均。dms-to-2a3モデルの予測も加味。
    - **CV戦略:** KFold。シーケンス類似性に基づくクラスタリングや長さベースの分割も検討したが、最終的にはKFoldを使用。
    - **その他:** 公開テストデータのリーク対策（訓練データと同一の配列に対する予測をゼロ化）。capR、他の構造予測ツール（ContraFold, ViennaRNA等）、RNA-FM、SQUARNA、3D構造データ、各種位置エンコーディング（xPos, ALiBi）、データ拡張（Reverse Augmentation）などを試行したが、最終的な改善にはつながらなかった。

**2位**

- **アプローチ:** SqueezeformerアーキテクチャとGRUヘッドを組み合わせ、BPPMをシンプルなConv2DNetで処理しAttentionバイアスとして加算。ALiBi位置エンコーディングを採用し、SN比で重み付けした損失関数を使用。
- **アーキテクチャ:** Squeezeformer（Conv1DとTransformerの混合）、GRUヘッド、浅い2層Conv2DNet（BPPM処理用）。
- **アルゴリズム:** ALiBi（位置エンコーディング）、Attention Bias（Conv2DNetで処理したBPPMを加算）、Weighted MAE Loss（log1p(signal_to_noise)で重み付け）、AdamWオプティマイザ、Cosine Decay学習率スケジュール。
- **テクニック:**
    - **特徴量:** CapR looptype、eternafold mfe、predicted looptype、bpp features (sum, nzero, max) を試したが、効果は限定的。
    - **モデル:** Squeezeformerの修正版（BN層、SwiGLUなど）、GRUヘッドの追加、ALiBiによる位置エンコーディング、BPPMをAttentionバイアスとして直接加算、さらにConv2DNetで処理したBPPMを加算。
    - **学習:** 200エポック、バッチサイズ256、AdamW、重み付きMAE損失。
    - **アンサンブル:** 異なるシードで学習したモデルのアンサンブル。
    - **その他:** 自己教師あり学習（SSL）、大規模モデル、データ拡張、疑似ラベリングなどを試したが、採用には至らず。

**3位**

- **アプローチ:** 2つの独立したモデルのブレンド。(1) Squeezeformerベースのモデル。(2) AlphaFoldスタイルのTwin-towerモデル。特にTwin-towerモデルでは、モデルの信頼度（エラー推定値）を利用してノイズの多い低SNRデータを改善し、それをSqueezeformerモデルの学習に活用。
- **アーキテクチャ:**
    - **Squeezeformerモデル:** Squeezeformerブロック（RelativeMultiHeadSelfAttention、Convolution、FeedForward）、2D CNN（BPPM処理用）。
    - **Twin-towerモデル:** AlphaFold風アーキテクチャ（MSA表現スタックとPair表現スタック）、Outer Product Mean、Pair Representation Bias、Relative Multi-Head Attention、Convolution、Triangular Multiplicative Updates。
- **アルゴリズム:** Relative Positional Encoding、Attention ScoreへのBPPバイアス加算（Squeezeformer）、Relative 2D Positional Encoding（Twin-tower）、AdamWオプティマイザ、Cosine Scheduling。
- **テクニック:**
    - **データ前処理/CV:** 4-Fold CV。BPPMを事前処理・キャッシュ。Twin-towerモデルはCleanデータのみで学習し、予測信頼度を推定。
    - **ノイズデータ改善:** Twin-towerモデルの信頼度と実験エラーを組み合わせて低SNRデータを「修正」。
    - **Squeezeformerモデル:** 相対位置エンコーディング、2D CNNで処理したBPPをAttention Scoreに加算、Cleanデータ＋改善された低SNRデータで学習。
    - **Twin-towerモデル:** MSAの代わりに単一シーケンス入力、Axial Self-Attentionの代わりにRelative Multi-Head AttentionとConvolutionを使用、MSAとPair表現が相互に情報を更新（Outer Product Mean, Pair Representation Bias）、予測信頼度（pLDDT類似）の出力。Cleanデータのみで学習（時間的制約）。
    - **学習:** Twin-towerは60エポック、Squeezeformerは200エポック。AdamW、Cosine Scheduling。
    - **アンサンブル:** SqueezeformerモデルとTwin-towerモデルのブレンド。

**4位**

- **アプローチ:** 3種類の異なるアーキテクチャのモデルをアンサンブル。EternaFold（一部ContraFold）によるBPPMを活用。高SN比データでのファインチューニングと、エラー予測を利用した疑似ラベリングを導入。
- **アーキテクチャ:**
    - **Modified RNAdegformer:** RNAdegformerを改変（Transformer + Conv1D）。
    - **1D Conv & Residual BPP Attention:** 1D CNNとBPP Attentionを組み合わせた構造（OpenVaccine解法インスパイア）、Bi-LSTM。
    - **Transformer with BPP Attention Bias:** オリジナルTransformerに近い構造＋BPP Attention Bias、FFN内にConv1D。
- **アルゴリズム:** ALiBi（Modified RNAdegformer）、Attention WeightとしてのBPP利用（1D Convモデル）、Attention BiasとしてのBPP利用（Transformerモデル）、AdamWオプティマイザ。
- **テクニック:**
    - **データ前処理:** EternaFold/ContraFold BPP。
    - **学習:** KFold(k=5)。2段階学習（SN>0.5で初期学習後、SN>1.0でファインチューニング）。エラー予測を用いた疑似ラベリング（予測SN比でフィルタリングし、追加データとして学習）。
    - **モデル（Modified RNAdegformer）:** GLU系活性化関数、レイヤー順序変更、複数ヘッドへのALiBi適用。
    - **モデル（1D Conv）:** 層の深さに応じてカーネルサイズを変化させるResidual BPP Attention。
    - **モデル（Transformer）:** 位置エンコーディングの代わりにFFN内のConv1D。
    - **アンサンブル:** 上記3タイプのモデルを含む複数モデルのアンサンブル。

**5位**

- **アプローチ:** チームメンバーそれぞれが開発したTransformerベースのモデルをアンサンブル。低SN比サンプルの活用、EternaFold BPPの利用、特定の位置エンコーディングや正規化手法の採用、GRUの追加、多段階疑似ラベル学習などが特徴。
- **アーキテクチャ:** Transformer Encoder、オプションでGRU/LSTM（各エンコーダ層後）、BPP処理用Convolutional Block（BMM利用またはAttention Bias）。
- **アルゴリズム:** Relative Positional Bias（最適と判断）、RMS Norm、ResiDual Norm（Pre-normより安定）、Inverse Square Root学習率スケジュール、AdamW。
- **テクニック:**
    - **データサンプリング/拡張:** SN比による重み付きランダムサンプリング、フリップTTA、BPPへのガウスノイズ付加。
    - **正規化/Attention:** RMS Norm、ResiDual Norm、Relative Positional Bias、BPPをAttention Biasとして追加（最初の6層のみ、1x1 Convで処理）、またはBPP BMM Convolutional Blockとして統合。
    - **RNN:** 各エンコーダ層の後に単層双方向GRUを追加。Multi-head RNN/LSTM（GRU+LSTMの2ヘッド構成が最良）も試行。
    - **疑似ラベリング:** 2段階または3段階パイプライン。予測の標準偏差でフィルタリングした疑似ラベルを使用。高SN比データでのファインチューニング。
    - **その他:** QKV層とFFN層からバイアスを除去、ヘッドサイズ増加（32->48）。

**7位**

- **アプローチ:** Transformerベース。MLPの代わりにMasked Conv1Dを使用。BPPMを多様な方法（Attention Matrixへの注入、Token Mixing、Matrix Mixing、Dual Stream Attention Boosting）で組み込む。
- **アーキテクチャ:** Transformer、Masked Conv1D（MLP代替）、BPPM処理用モジュール（Convolutional Stream、Matrix Multiplicationなど）、オプションでGraph Transformer。
- **アルゴリズム:** Rotary Encoding（位置エンコーディング）、Attention MatrixへのBPPM注入（logit形式、層ごとに減衰）、BPPMベースのToken Mixing、Matrix Mixing（CNN+SE）、Dual Stream Attention Boosting（Attention状態を射影・非線形変換してAttention Matrixに加算）、MLM（事前学習）。
- **テクニック:**
    - **データ前処理/CV:** 複数ツールでBPPM生成（効果限定的）、EXデータでのファインチューニング（限定的効果）、シーケンス類似性に基づくCV分割、エラーに基づく損失重み付け(`w = 1/sqrt(1/6 + err.clip(100))` または `log(1.1 + snr) / 2`)。
    - **モデル:** MLPをMasked Conv1Dに置換、多様なBPPM組み込み戦略（注入、Mixing、Boosting）。
    - **学習:** AdamW、Cosine Annealing、フリップ拡張（BPPMは元の順序で計算）、Binベース補助損失、複数段階ファインチューニング、疑似ラベリング（予測誤差で重み付け）。MLM事前学習（slime）。
    - **アンサンブル:** 約20モデルの組み合わせ。

**8位**

- **アプローチ:** CNNとGraph Neural Network (GNN) のみを組み合わせたアーキテクチャ。LegNetやOpenVaccine解法に触発。複数のグラフ特徴量（bpp, structure, chunk, segment）それぞれに独立したCNN+GNNブロックを使用。GroupKFoldによるCV戦略。
- **アーキテクチャ:** CNN、GNN（グラフ特徴量ごとに独立ブロック）、Group Convolution。
- **アルゴリズム:** MAE + MSE損失（SN比重み付け）、予測ヘッドで100次元ベクトル出力→加重和。
- **テクニック:**
    - **データ前処理/CV:** `arnie`パッケージで特徴量抽出（eternafold, rnasoft, rnastructure）、bpp、structure、loop_type、chunk、segment特徴量。シーケンス編集距離に基づくクラスタリングによるGroupKFold。seqlen=206サンプルを特定のFoldに集約。RMDBデータセットの活用（交互学習）。
    - **モデル:** 複数のグラフ特徴量タイプごとに独立したCNN+GNNブロックをGroup Convolutionで並列処理。予測ヘッドで多次元ベクトルを出力し、加重和で最終予測値を計算。
    - **学習:** MAE+MSE損失（SN比重み付け）。疑似ラベル学習。

**10位**

- **アプローチ:** 1D Conv + Transformerモデル（位置エンコーディングなし）。CapRやBPPの集約特徴量（sum, max）を利用。予測値のクリッピングと独自の重み付き損失関数が特徴。シーケンスのみのモデルと特徴量ありモデルをアンサンブル。
- **アーキテクチャ:** 1D CNN、Transformer（位置エンコーディングなし）。
- **アルゴリズム:** 独自の重み付き損失関数（測定誤差の正規分布を考慮: `MAE * (1 - 2*dist.cdf(reactivity-|reactivity-pred|))` where `dist=normal(mu=reactivity, sigma=reactivity_error)`）、予測値のSigmoidクリッピング。
- **テクニック:**
    - **データ前処理:** One-hotエンコーディング（埋め込み層より有効）、CapR、BPP sum & max特徴量。
    - **モデル:** 位置エンコーディングなし（Conv層が担当）、予測値のSigmoidクリッピング。
    - **学習:** 独自の重み付き損失関数。誤差自体も不確かさを持つ可能性を考慮し、誤差に乗数をかけたり定数を加えたりして複数モデルを学習（一種のラベルスムージング）。
    - **検証戦略:** 中間領域で学習し、両端や長配列で検証する段階を経て、最終的には全領域・全長で学習。
    - **アンサンブル:** シーケンスのみモデルと特徴量（CapR+BPP sum&max）ありモデルのアンサンブル。BPPM自体への不信感から、BPPMを直接Attentionに注入するモデルは最終アンサンブルには含めず。R1138v1の予測結果を参考にアンサンブル比率を決定。


