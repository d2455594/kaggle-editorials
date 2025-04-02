---
tags:
  - Kaggle
  - MAE
  - Transformer
  - 計算生物学
startdate: 2023-09-08
enddate: 2023-12-08
---
# Stanford Ribonanza RNA Folding
[https://www.kaggle.com/competitions/stanford-ribonanza-rna-folding](https://www.kaggle.com/competitions/stanford-ribonanza-rna-folding)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、与えられたRNA分子の塩基配列と、その一部分に関する実験的な構造プローブデータ（DMSおよび2A3実験による反応性データ）に基づいて、配列全体の各塩基における**DMSおよび2A3の反応性（Reactivity）を精密に予測する**機械学習モデルを開発することです。
* **背景:** RNA分子は、遺伝情報の担い手としてだけでなく、触媒作用や遺伝子発現制御など、細胞内で多様な機能を持っています。これらの機能はRNAの複雑な3次元構造と密接に関連しており、その構造は塩基配列によって決定されます。RNA構造を正確に予測することは、生命現象の基本的な理解を深め、RNA標的薬の開発など応用面でも非常に重要です。Ribonanzaプロジェクトは、大規模なRNA構造・反応性データを生成し、AIによる予測精度の向上を目指す取り組みの一環です。
* **課題:** RNAの構造と反応性を配列情報のみから予測することは、計算生物学における長年の難題です。特に、長い配列やシュードノットなどの複雑な高次構造を持つRNAの予測は困難です。提供される実験データには測定ノイズが含まれており（低シグナルノイズ比、SN比）、全塩基が測定されているわけではありません（欠損値が存在）。さらに、テストデータにはトレーニングデータよりも長い配列が含まれることが想定されており、モデルが未知の長さの配列に対しても**頑健に予測できる汎化性能**が求められました。

**データセットの形式 (Dataset Format)**

提供される主なデータは、RNA配列情報と、それに対応する実験的な反応性データです。

1.  **トレーニングデータ:**
    * `train_data.csv`: トレーニングに含まれるRNA配列の情報。
        * `sequence_id`: 各RNA配列を一意に識別するID。
        * `sequence`: RNAの塩基配列（A, C, G, Uの文字列）。
        * `experiment_type`: 実験の種類（このコンペでは主に`DMS_MaP`と`2A3_MaP`に関連）。
        * `dataset_name`: データセット名。
        * `SN_filter`: シグナルノイズ比に基づくフィルタリングフラグ（1: 高SN比、0: 低SN比）。
    * `.parquet` ファイル群: 各`sequence_id`に対応するパーケ形式のファイル。各ファイルには塩基ごとの詳細なデータが含まれます。
        * `sequence`: 配列内の塩基インデックス。
        * `reactivity_DMS_MaP`, `reactivity_error_DMS_MaP`: DMSプローブに対する反応性の測定値と推定誤差。
        * `reactivity_2A3_MaP`, `reactivity_error_2A3_MaP`: 2A3プローブに対する反応性の測定値と推定誤差。
        * `reactivity`, `reactivity_error`: 上記2つのどちらか、または両方が欠損している場合がある。多くの塩基でNaN値が含まれます。
2.  **テストデータ:**
    * `test_sequences.csv`: 評価対象となるRNA配列のID (`id`) と塩基配列 (`sequence`)。Publicテストセット（約3000配列）とPrivateテストセット（約100万配列）の両方の情報が含まれます。Privateテストセットにはトレーニングデータよりも長い配列（最大457塩基）が含まれる可能性があります。
    * 反応性データは提供されません。参加者はこれらの配列に対して反応性を予測します。
3.  **外部データ/ツール:**
    * **BPPM (Base Pairing Probability Matrix):** EternaFold, ViennaRNA, CONTRAfold, RNAsoftなどの外部ツールを用いて、各塩基対が形成される確率を行列として計算し、これを特徴量として利用することが一般的でした。EternaFoldによるBPPMが特に有効とされました。
    * **二次構造/Loop Type:** 同様のツールで予測される二次構造（ドットブラケット記法）やLoop Type（例: CapRで計算）も補助的な特徴量として利用されました。
    * **公開データベース:** RMDB (RNA Mapping Database) などの公開データも参照されました。
4.  **`sample_submission.csv`**:
    * 提出フォーマットのサンプル。`id`（テストデータの行ID、`sequence_id`と`sequence`インデックスから構成）と、予測する2つの反応性 (`reactivity_DMS_MaP`, `reactivity_2A3_MaP`) の列を持ちます。

**評価指標 (Evaluation Metric)**

* **指標:** **平均絶対誤差 (Mean Absolute Error, MAE)**
* **計算方法:** 予測対象となる全ての塩基位置（PublicテストセットおよびPrivateテストセット）について、DMS反応性と2A3反応性の両方に対して、予測値と真の値（隠されている）の絶対誤差を計算し、それらの平均を取ります。
    * MAE = (1 / (評価対象塩基数 * 2)) * Σ (|予測DMS - 真DMS| + |予測2A3 - 真2A3|)
* **意味:** モデルの予測値が平均してどれだけ真の値からずれているかを示す指標です。値が小さいほど予測精度が高いことを意味します。スコアは**低い**ほど良い評価となります。

---

**全体的な傾向**

このRNA反応性予測タスクでは、塩基配列というシーケンスデータを入力として各位置の連続値を予測するため、**Transformerベースのアーキテクチャ**が圧倒的に優勢でした。特に、**Transformer Encoder** を用いたモデルが広く採用されました。一方で、1D ConvolutionやRNN (LSTM, GRU) を組み合わせたハイブリッドモデル（**Squeezeformer** や **RNAdegformer** の改良版など）も高い性能を示しました。AlphaFoldに触発された**Twin Towerアーキテクチャ**も試みられました。

入力特徴量としては、塩基配列そのもの（トークン化してEmbedding）に加え、外部ツールで計算された**BPPM（塩基対合確率行列）** が極めて重要な役割を果たしました。BPPMをモデルに組み込む方法として、**2D CNNで処理**して得た特徴量をTransformer層に渡す、あるいはSelf-Attention計算時に**Attention Biasとして加算**する、Attention Weightとして利用する（**GNN風アプローチ**）などの工夫が見られました。

テストセットにトレーニングセットよりも長い配列が含まれるため、モデルの**長さに対する汎化性能**が非常に重要でした。これに対処するため、絶対位置エンコーディングではなく、**相対位置エンコーディング**（**Dynamic Positional Bias**, **ALiBi**, **Rotary Embedding (RoPE)** など）が広く採用されました。

トレーニングデータに含まれる**低シグナルノイズ比（低SN比）のデータ**をどう扱うかも鍵となりました。戦略としては、SN比に基づいて**損失関数に重みを付ける**、あるいは**サンプリング確率を調整する**、SN比が低いデータを除外するフィルタリング、SN比が高いデータのみで**ファインチューニング**を行う、高SN比データで学習したモデルで低SN比データに**疑似ラベルを生成**して学習に利用する、などが用いられました。

学習においては、**MAE損失**（またはそれに類するL1損失）が評価指標に合わせて用いられ、しばしばSN比や反応性誤差に基づいて重み付けされました。**AdamWオプティマイザ**と**Cosine Annealing**などの学習率スケジューラが一般的でした。

最終的なスコア向上のためには、異なるアーキテクチャ、異なるFold、異なる特徴量セット、異なるシード、あるいは異なる学習データ（疑似ラベル使用有無など）で学習された**複数モデルの予測値を平均化するアンサンブル**が不可欠でした。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/stanford-ribonanza-rna-folding/discussion/460121)**

* **アプローチ:** Transformer Encoderベース。EternaFoldによるBPPMをCNNで処理し、Attentionスコアに加算。動的位置バイアス (Dynamic Positional Bias) を使用。
* **アーキテクチャ:** 12層Transformer Encoder。Self-Attentionブロック内で、BPPMを処理するConvolutional Block (2D Conv + BatchNorm + Activation + スケーリング) の出力をAttentionスコアに加算。一部モデルではConv BlockにSqueeze-and-Excitation (SE) 層を追加。
* **アルゴリズム:** AdamW + OneCycleLR。損失関数はMAE? SN比で重み付けサンプリング。
* **テクニック:**
    * **特徴量:** 塩基配列Embedding、SN比Embedding、EternaFold BPPM。
    * **位置エンコーディング:** Dynamic Positional Bias (学習可能、シーケンス長依存)。
    * **学習:** ほぼ全データで学習 (270 epochs)。SN比で重み付けサンプリング。最後にSGDで追加学習 (約15 epochs)。DMS予測値から2A3を予測するモデルも作成し、最終アンサンブルに利用。
    * **CV戦略:** シーケンス類似度に基づくクラスタリングでの分割を試したが、最終的にはシンプルなKFoldを使用。
    * **アンサンブル:** SEありモデル(15個)、SEなしモデル(10個)、他2モデル、DMS->2A3モデルを含む大規模アンサンブル。
    * **その他:** Publicテストのリーク（学習データと同一配列）を考慮し、提出時に該当予測をゼロ化。

**[2位](https://www.kaggle.com/competitions/stanford-ribonanza-rna-folding/discussion/460316)**

* **アプローチ:** Squeezeformer (Conv1D + Transformer) + GRUヘッド。BPPMをCNNで処理し、Attention Biasとして利用。ALiBi位置エンコーディング。
* **アーキテクチャ:** Squeezeformer Encoder (12層, dim=192, head=4, kernel=17) + 単層GRUヘッド。Self-Attention内で、BPPMを2層の浅い2D CNNで処理した結果をAttention Biasとして加算。
* **アルゴリズム:** AdamW + Cosine Decay LR。損失関数: 重み付きMAE (重み=log1p(SN比).clip(0,10))。
* **テクニック:**
    * **特徴量:** 塩基配列、EternaFold BPPM。補助的にCapR looptype, eternafold mfe, predicted looptype, bpp集約特徴量も使用（効果は限定的）。
    * **位置エンコーディング:** ALiBi (Attention with Linear Biases)。
    * **学習:** 200 epochs。バッチサイズ256。SN比で損失を重み付け。
    * **アンサンブル:** 異なるシードで学習したモデルをアンサンブル。
    * **動作しなかったこと:** SSL、大規模モデル、Augmentation、疑似ラベル。

**[3位](https://www.kaggle.com/competitions/stanford-ribonanza-rna-folding/discussion/460403)**

* **アプローチ:** 2つの独立したモデルのブレンド。1) 改良Squeezeformer、2) AlphaFold風Twin-Towerモデル。低SN比データの改良。
* **アーキテクチャ:**
    * Squeezeformer改: 14層。Relative MultiHeadSelfAttention + Convolution + FeedForward。BPPMを2D Convで処理しAttentionスコアに加算。
    * Twin-Tower: AlphaFold風。MSAスタック（配列のみ使用）とPairスタック表現。MSA側はSqueezeformer Attention、Pair側はTriangular Multiplicative Update。Outer Product MeanとPair Biasで情報交換。相対2D位置エンコーディング（Pair）。
* **アルゴリズム:** AdamW + Cosine Scheduling。損失関数はMAE?
* **テクニック:**
    * **データ:** Twin-Towerモデルは高SN比データのみで学習。その予測信頼度と実験エラーを用いて低SN比データを「修正」し、合成データを作成。Squeezeformerモデルはこの合成データ＋高SN比データで学習。
    * **特徴量:** 塩基配列、EternaFold BPPM。
    * **位置エンコーディング:** 相対位置エンコーディング（Squeezeformer）、相対2D位置エンコーディング（Twin-Tower Pair）。
    * **学習:** Twin-Towerは60 epochs、Squeezeformerは200 epochs。
    * **アンサンブル:** Twin-Towerモデル（配列のみ入力）とSqueezeformerモデル（配列+BPPM+合成データ）のブレンド。

**[4位](https://www.kaggle.com/competitions/stanford-ribonanza-rna-folding/discussion/460203)**

* **アプローチ:** 3種類のモデルアーキテクチャのアンサンブル。1) RNAdegformer改良版、2) 1D Conv + Residual BPP Attention、3) Transformer + BPP Attention Bias。疑似ラベル使用。
* **アーキテクチャ:**
    * RNAdegformer改: TransformerとConv1Dのハイブリッド。ALiBi位置エンコーディング。BPPM Attention Bias。
    * 1D Conv + ResBPPAttn: 1D Conv層と、BPPMをAttention Weightとして用いるResidual BPP Attention層を交互に配置。Bi-LSTMヘッド。
    * Transformer + BPPA Bias: 標準Transformerに近いが、位置エンコーディングの代わりにFFN内にConv1Dを使用。BPPM Attention Bias。
* **アルゴリズム:** AdamW + Cosine Scheduler with Warmup。損失関数はMAE?
* **テクニック:**
    * **特徴量:** 塩基配列、EternaFold BPPM (一部CONTRAfoldも)。
    * **学習:** 2段階学習（高SN比データで学習後、さらにSN比>1.0データでファインチューニング）。
    * **疑似ラベル:** エラー予測モデルも学習し、予測エラーが低いテストデータの予測を疑似ラベルとして使用（SN比でフィルタリング）。疑似ラベルを加えてモデルを再学習。
    * **アンサンブル:** 上記3種類のアーキテクチャ、異なるFold、疑似ラベル使用有無などのモデルをアンサンブル。

**[5位](https://www.kaggle.com/competitions/stanford-ribonanza-rna-folding/discussion/460250)**

* **アプローチ:** Transformer Encoderベース。RMSNorm、ResiDual Norm、相対位置バイアス、GRU層などを導入。多段階疑似ラベル学習。
* **アーキテクチャ:** Transformer Encoder (12層)。 LayerNormをRMSNormに置換。 Pre-normレイアウトをResiDual Normレイアウトに置換。 QKV/FFNのバイアス除去。Headサイズを48に増加。各Encoder層の後に単層BiGRUを追加。一部モデルではGRUの代わりにMulti-head RNN (GRU+LSTM) やBPP Attention Blockを使用。
* **アルゴリズム:** AdamW (weight decay 0.1) + Inverse Square Root LRスケジュール。損失はMAE?
* **テクニック:**
    * **データサンプリング:** SN比に基づいて重み付けサンプリング。SNバイアス項をスケジュールで変化させ、徐々に低SN比データも利用。
    * **Augmentation:** Flip TTAが有効。BPPにガウスノイズを加える。
    * **位置エンコーディング:** 相対位置バイアス。
    * **BPPM組み込み:** Attention Biasとして加算（初期6層のみ）、またはResidual BPP Attentionブロックを使用。
    * **疑似ラベル:**
        * 2段階パイプライン: 5 Foldモデル (Flip TTAあり) で予測 → 不確かさ(stdev)が高い予測を除外 → その疑似ラベルで単一モデルを学習 → そのモデルを事前学習済みとして、元のデータで再度5 Fold学習。
        * 3段階パイプライン: Trainのみ → Train + 疑似ラベル → Trainのみ。LRを段階的に下げる。疑似ラベルは同様にフィルタリング。

**[7位](https://www.kaggle.com/competitions/stanford-ribonanza-rna-folding/discussion/460190)**

* **アプローチ:** Transformerベース。MLPの代わりにMasked Conv1Dを使用。BPPMをAttentionに組み込む複数の方法（Injection, Mixing, Boosting）を試し、アンサンブル。
* **アーキテクチャ:** Transformer Encoder (24層, dim=384, head=6)。FFN (MLP) をMasked 1D Conv (kernel 5) に置換。Rotary Embedding (RoPE) 使用。
* **アルゴリズム:** AdamW + Cosine Annealing with Warmup。損失関数はMAE? エラー値に基づいて重み付け。
* **テクニック:**
    * **BPPM組み込み:**
        * Injection: BPPMを対数化しAttentionスコアに直接加算（最終層に向けて減衰）。
        * Mixing: BPPMに基づいてトークンを混合する行列乗算モジュールを追加。
        * Matrix Mixing: BPPMをConv層で処理しAttention Biasを生成（別チームメンバー）。
        * Dual Stream (Attention Boosting): Attention状態を射影し、非線形変換後にAttentionスコアに加算してブースト。
    * **データ:** EternaFold BPPMが主。他のBPPMや外部データ（RMDB, EX data）も試行（EX dataはPrivate LBに寄与せず）。
    * **学習:** エラー値で損失を重み付け (`w = 1/sqrt(1/6 + err.clip(100))`)。SN比フィルタリングは不使用。Flip Augmentation（BPPMも反転）。Binベースの補助損失。
    * **疑似ラベル:** 複数段階の疑似ラベル生成とファインチューニング。高SN比データでのFTも実施。
    * **アンサンブル:** 上記の異なるBPPM組み込み方、Fold、疑似ラベル有無など約20モデルをアンサンブル。

**[8位](https://www.kaggle.com/competitions/stanford-ribonanza-rna-folding/discussion/460222)**

* **アプローチ:** CNNとGNNを組み合わせたアーキテクチャ。複数のグラフ特徴量（BPPM, Structureなど）を使用。グループ化された畳み込み。
* **アーキテクチャ:** LegNetとOpenVaccine 6位解法に触発。複数のCNN+GNNブロック（BPPM, structure, chunk, segmentごとに独立）。Group Convolutionとeinsumで効率化。予測ヘッドは100次元ベクトルを出力し、重み付き和で最終値を得る（LegNet風）。
* **アルゴリズム:** AdamW? 損失関数: MAE + MSE（SN比で重み付け）。
* **テクニック:**
    * **特徴量:** 塩基配列Embedding、外部ツール(arnie)で計算した二次構造情報（structure, loop_type）、BPPM、およびbpRNAで定義される 'chunk', 'segment'。
    * **CV戦略:** 配列類似度に基づくクラスタリングIDを用いたGroupKFold。長さ206の配列はFold 0に集約。
    * **学習:** RMDBデータセットと交互に学習。疑似ラベルも使用。
    * **アンサンブル:** チームメンバーのモデルとブレンド。

**[10位](https://www.kaggle.com/competitions/stanford-ribonanza-rna-folding/discussion/463352)**

* **アプローチ:** 1D Conv + Transformerモデル。位置エンコーディング不使用。カスタム損失関数。
* **アーキテクチャ:** 1D Conv層 + Transformer Encoder層。位置エンコーディングなし（Conv層が相対位置情報を処理）。
* **アルゴリズム:** AdamW? 損失関数: カスタム重み付きMAE。
* **テクニック:**
    * **特徴量:** 塩基配列（One-hot）、CapR、BPPMのsum/max（モデル2のみ）。BPPMのAttentionへの直接注入は不採用（不信感）。
    * **予測クリッピング:** Sigmoid関数で予測値をクリップ（0-1範囲外への発散防止と学習安定化）。
    * **カスタム損失関数:** MAE損失に、測定誤差(`reactivity_error`)を標準偏差とする正規分布を用いて計算した「ポテンシャル」項を乗算。誤差が大きい測定値の影響を適切に調整（高誤差でも遠い予測には大きなペナルティ、近い予測には小さなペナルティ）。誤差自体にノイズがあることを考慮し、誤差に乗数をかけたり定数を加えたりしてランダム化する一種のラベルスムージングも試行。
    * **アンサンブル:** 配列のみモデルと、+BPPM&CapRモデルをアンサンブル。R1138v1の予測可視化に基づき重みを決定（最終提出は5:3）。
