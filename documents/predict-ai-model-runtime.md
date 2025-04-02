---
tags:
  - Kaggle
  - スピアマンの順位相関係数
  - GNN
  - SageConv
  - GraphTransformer
startdate: 2023-08-30
enddate: 2023-11-18
---
# Google - Fast or Slow? Predict AI Model Runtime
[https://www.kaggle.com/competitions/predict-ai-model-runtime](https://www.kaggle.com/competitions/predict-ai-model-runtime)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、与えられたAIモデルの計算グラフとその設定（コンフィグ）に基づいて、特定のハードウェア（Google TPU v3）上での**実行時間を予測する**モデルを開発することです。
* **背景:** GoogleはTensor Processing Unit (TPU) などのAIアクセラレータを開発・提供しています。AIモデルの実行時間は、モデルの構造、演算子の種類、設定、ハードウェア特性など多くの要因に依存し、事前に正確に見積もることは困難です。実行時間を予測できれば、モデル開発サイクルの効率化（例: 遅いモデル構造の早期発見）、ハードウェアリソースの最適化、コンパイラの最適化戦略決定などに役立ちます。
* **課題:**
    * **グラフ構造の表現:** AIモデルの計算グラフ（ノードとエッジ）を効果的に表現し、モデルの入力とする必要があります。グラフニューラルネットワーク (GNN) が自然な選択肢となりますが、グラフのサイズが大きい場合（特にLayoutデータ）の計算コストが課題です。
    * **多様な設定（コンフィグ）:** 同じグラフ構造でも、ノードの設定（例: 畳み込みのカーネルサイズやストライド、テンソルのレイアウト）によって実行時間が大きく変動します。この設定情報をどうモデルに組み込むかが重要です。
    * **ハードウェア特性の暗黙的な学習:** 特定のハードウェア (TPU v3) 上での実行時間を予測するため、モデルはハードウェアの特性をデータから暗黙的に学習する必要があります。
    * **ランキング評価:** 評価指標はKendall Tau相関係数であり、実行時間の絶対値を予測するだけでなく、異なる設定間での実行時間の相対的な順序（ランキング）を正確に予測することが求められます。
    * **データセットの分割:** データは目的（`layout` または `tile`）とデータソース（`xla` または `nlp`）によって分割されており、それぞれ特性が異なるため、個別のモデル開発やドメイン適応が必要になる場合があります。

**データセットの形式 (Dataset Format)**

データは、AIモデルの計算グラフ構造、ノードの特徴量、設定情報、および対応する実行時間から構成されます。データは5つのコレクション（`layout:xla:random`, `layout:xla:default`, `layout:nlp:random`, `layout:nlp:default`, `tile:xla`）に分かれています。

1.  **ファイル形式:** 各コレクションのデータはNumPyの `.npz` 形式で提供されます。各ファイルには、1つの計算グラフに対する複数の設定とその実行時間データが含まれます。
2.  **データフィールド:** 各 `.npz` ファイルには主に以下のデータが含まれます。
    * `node_feat`: 各ノードの特徴量ベクトル (形状: `[num_nodes, 140]`)。テンソル形状、レイアウト情報などが含まれる。
    * `node_opcode`: 各ノードの操作コード（演算子の種類）を示す整数ID (形状: `[num_nodes]`)。
    * `edge_index`: グラフのエッジ（ノード間の接続）を示すテンソル (形状: `[2, num_edges]`)。
    * `node_config_feat`: 各ノードの設定（コンフィグ）に関する特徴量ベクトル (形状: `[num_configs, num_nodes, 18]`)。設定可能なノード以外はパディングされている。
    * `node_config_ids`: 設定（コンフィグ）を持つノードのインデックス (形状: `[num_config_nodes]`)。
    * `config_runtime`: **ターゲット変数**。各設定に対応する実行時間 (形状: `[num_configs]`)。
    * `config_feats` (Tileデータのみ): タイル分割の設定に関する特徴量 (形状: `[num_configs, 128]`)。
3.  **分割:** 主催者によって、各コレクションごとに `train`, `valid`, `test` のグラフIDリストが提供されます。

**評価指標 (Evaluation Metric)**

* **指標:** **Kendall Tau 相関係数 (Kendall Tau Correlation Coefficient)**。ランキングの一致度を測る指標。
* **計算方法:**
    * 各グラフについて、モデルが予測した実行時間と実際の実行時間に基づいて、設定（コンフィグ）のペア間の順序関係（どちらが速いか、遅いか、同じか）を比較します。
    * 全てのペアについて、予測された順序と実際の順序が一致するペア（Concordant pairs）と、一致しないペア（Discordant pairs）の数をカウントします。
    * Kendall Tau は `(Number of Concordant Pairs - Number of Discordant Pairs) / (Total Number of Pairs)` で計算されます（タイの扱いによって若干のバリエーションあり）。
    * 最終スコアは、**全てのテストグラフにわたるKendall Tauスコアの平均値**となります。
* **意味:** モデルが予測した実行時間のランキングが、実際の実行時間のランキングとどれだけ一致しているかを評価します。実行時間の絶対値の誤差ではなく、**相対的な順序の正しさ**を重視します。スコアは -1 (完全な逆順) から +1 (完全な一致) の範囲を取り、**高い**ほど良い性能を示します (0はランダムな順序)。

要約すると、このコンペティションは、AIモデルの計算グラフと設定情報からTPU上での実行時間を予測するランキングタスクです。データはグラフ構造、ノード特徴、設定、実行時間で構成され、性能はKendall Tau相関係数（高いほど良い）によって評価されます。グラフデータの扱いやランキング学習が鍵となります。

---

**全体的な傾向**

このAIモデル実行時間予測コンペでは、グラフ構造とノード設定情報を入力として実行時間のランキングを予測するという課題に対し、**グラフニューラルネットワーク (GNN)** が中心的な役割を果たしました。特に **GraphSAGE (SAGEConv)** や **Graph Attention Network (GAT, GATv2)**、**Graph Isomorphism Network (GIN)**、**APPNP** といった畳み込み層や、それらを組み合わせたアーキテクチャが広く採用されました。**Transformer** をグラフ構造に適用する試みも見られました。

**入力特徴量の処理**が重要であり、ノード特徴量 (`node_feat`) の正規化 (StandardScaler, Log Transform)、操作コード (`node_opcode`) や設定特徴量 (`node_config_feat`, `layout_minor_to_major_*`) に対する**埋め込み (Embedding)** が一般的に行われました。特に、カテゴリ的な特徴量に対して共有埋め込みを用いるなどの工夫が見られました。

Layoutデータセットのグラフサイズが大きいことが計算上のボトルネックであり、多くのチームが**グラフプルーニング**や**サブグラフサンプリング**によって計算効率を改善しました。設定可能なノード (`configurable nodes`) とその近傍ノードのみを抽出する手法や、Dijkstra法で主要パス上のノードを抽出する手法などが用いられました。また、データ読み込みを効率化するために、NPZファイルを再構成したり、設定情報を圧縮したりする工夫も見られました。

**ランキング学習**に特化した損失関数が有効でした。**ペアワイズ損失 (Pairwise Loss)**、特に **Hinge Loss** や **Margin Ranking Loss** が広く用いられました。**Listwise損失 (ListMLE)** も効果的でしたが、数値安定性の問題に対処する必要がありました。1位チームは独自の**DiffMat Loss** (ペアワイズ差分のMargin Ranking Loss) を提案・使用しました。

モデルアーキテクチャでは、GNN層を複数重ね、**残差接続 (Residual Connection)** や **正規化層 (InstanceNorm, LayerNorm, GraphNorm)** を導入することが一般的でした。また、グラフ全体の情報を集約するために**Global Pooling** (Mean, Sum) が用いられました。チャネル間の相互作用を捉えるための**Self-Channel Attention**や、異なる設定（コンフィグ）間の関係を捉える**Cross-Config Attention**といった独自のAttention機構も提案されました。

**学習戦略**としては、データセット（Layout/Tile, XLA/NLP, Random/Default）ごとに**個別のモデルを学習**し、最後にアンサンブルするアプローチが主流でした。勾配累積や、バッチ内でグラフと設定を階層的に扱うサンプリング戦略も用いられました。

**アンサンブル**は最終的なスコア向上に寄与し、異なるモデルアーキテクチャ、異なるFold、異なるランダムシードで学習したモデルの予測値（ランキングスコアや対数尤度など）を単純平均または重み付き平均する手法が取られました。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/predict-ai-model-runtime/discussion/456343)**

* **アプローチ:** GraphSageベースGNN + Attention機構。グラフプルーニングとデータ圧縮。
* **アーキテクチャ:** Linear → 2x [InstanceNorm → SAGEConv → SelfChannelAttention → CrossConfigAttention → Residual → GELU] → Global Mean Pooling → Linear。
* **アルゴリズム:** GNN、Attention。
* **テクニック:** **グラフプルーニング** (設定ノードとその近傍のみ)。設定情報の**圧縮/オンザフライ解凍**。`node_feat`の-1パディング化と共有Embedding。Pairwise Hinge Loss。**Self-Channel Attention**と**Cross-Config Attention**。TTA (設定順序の置換)。データセット別モデル。

**[2位](https://www.kaggle.com/competitions/predict-ai-model-runtime/discussion/456365)**

* **アプローチ:** GraphSageベースGNN。独自損失 DiffMat Loss。
* **アーキテクチャ:** SageConvベース (Layout-XLA: 6層、Layout-NLP/Tile: 8層)。
* **アルゴリズム:** GNN。
* **テクニック:** データ重複除去(config基準)。破損グラフ除去(Layout-XLA)。NPZ再構成による高速読み込み。**独自損失 DiffMat Loss** (ペアワイズ差分へのMargin Ranking Loss)。ListMLE (Layout-NLP)、MAPE+DiffMat (Layout-XLA)。バッチ階層化(グラフ/設定)。早期融合。データセット別モデル。

**[3位](https://www.kaggle.com/competitions/predict-ai-model-runtime/discussion/456377)**

* **アプローチ:** GPSレイヤー (SAGEConv + Linformer) ベースGNN。グラフプルーニング/マージ。
* **アーキテクチャ:** GPS Layer (SAGEConv + Linformer) ベース。
* **アルゴリズム:** GNN (GraphSAGE, Linformer)。
* **テクニック:** 特徴量エンジニアリング (protobufからの追加特徴、入力形状特徴、次元mod特徴)。RWPE位置エンコーディング。**グラフプルーニング/マージ** (設定ノード/近傍以外を削除またはマージ)。仮想出力ノード追加。Pairwise Hinge Loss。データセット別モデル。

**[4位](https://www.kaggle.com/competitions/predict-ai-model-runtime/discussion/456462)**

* **アプローチ:** シンプルなMLP。グラフ構造を陽に扱わない。
* **アーキテクチャ:** MLP。ノード特徴用MLP、設定特徴用MLP、両者の相互作用項 + 設定平均特徴を入力とする最終MLP。
* **アルゴリズム:** MLP。
* **テクニック:** 特定の特徴量インデックスのみ使用。グラフ構造はエッジ情報を介してノード特徴に集約。ターゲットはランクパーセンタイル。カスタムデータローダー。

**[5位](https://www.kaggle.com/competitions/predict-ai-model-runtime/discussion/456093)**

* **アプローチ:** GraphSageベースGNN + Transformerによる次元特徴埋め込み。
* **アーキテクチャ:** 3層 GraphSage (双方向畳み込み)。**Transformerによる次元特徴埋め込み**レイヤー。
* **アルゴリズム:** GNN (GraphSAGE), Transformer。
* **テクニック:** **次元特徴埋め込み**: (6次元x30特徴量)をTransformerで(6トークンx30次元)として扱い、集約して単一ベクトルに。Pairwise Hinge Loss (グラフ内ペア + バッチ内ペア)。Whole Graph入力。Opcode Embedding共有。DropEdge。Log変換。オーバーサンプリング。Numpy mmapによるRAM節約。

**[6位](https://www.kaggle.com/competitions/predict-ai-model-runtime/discussion/456084)**

* **アプローチ:** GraphSage/GIN/GATベースGNN。5ホップ近傍サブグラフ。
* **アーキテクチャ:** 4層 SAGEConv + Residual または 4層 GINConv (Layout)。4層 GATConv (Tile)。
* **アルゴリズム:** GNN (GraphSAGE, GIN, GAT)。
* **テクニック:** **5ホップ近傍サブグラフ**抽出。**Graph Instance Norm**。勾配累積。Pairwise Ranking Loss (Layout)、ListMLE (Tile)。

**[7位](https://www.kaggle.com/competitions/predict-ai-model-runtime/discussion/456673)**

* **アプローチ:** GST (Graph Substructure Transformer) ベースGNN。設定サンプリング。
* **アーキテクチャ:** GSTベース (公開実装参照)。
* **アルゴリズム:** GNN (Transformerベース？)。
* **テクニック:** 設定サンプリング (各グラフから500-1000)。モデルタイプごとの学習。異なるGNN層(Graph Conv, GAT, Transformer)やハイパーパラメータでの複数モデルアンサンブル。

**[8位](https://www.kaggle.com/competitions/predict-ai-model-runtime/discussion/456645)**

* **アプローチ:** LightGBM。テストデータに類似した訓練データのみを使用。
* **アーキテクチャ:** LightGBM。
* **アルゴリズム:** GBDT。
* **テクニック:** Layout: ノード/エッジ数に基づき、テストデータごとに**最も類似した訓練グラフを特定**し、そのデータのみで学習。Tile: 特徴量エンジニアリング (集約特徴量)。ターゲットをMinMaxScalerで変換。損失関数はXEntropy。Validationなし、固定ラウンド学習。

**[9位](https://www.kaggle.com/competitions/predict-ai-model-runtime/discussion/456206)**

* **アプローチ:** GSTベースGNN。**Dijkstra法によるグラフ圧縮**。
* **アーキテクチャ:** GSTベース (公開実装参照)。TransformerConv, GATConvも試行。
* **アルゴリズム:** GNN (Transformerベース？)。
* **テクニック:** **グラフ圧縮**: Dijkstra法で最大インデックスノードから設定ノードへの最短パス上のノード・エッジを抽出。Log変換入力特徴量。Pairwise Hinge Loss。データセット別/モデルアーキテクチャ別モデル学習。アンサンブル。

**[10位](https://www.kaggle.com/competitions/predict-ai-model-runtime/discussion/456129)**

* **アプローチ:** Graph Transformer。**中間層での設定情報融合 (Intermediate Fusion)**。
* **アーキテクチャ:** Graph Transformer (局所: APPNP/SAGE/GAT/GPRGNN、大域: Linear Attention/Self-Attention)。
* **アルゴリズム:** GNN (Transformerベース)。
* **テクニック:** **Intermediate Fusion**: GNN層の後、ノード表現に設定情報を結合し、さらに層を重ねる。Log変換入力特徴量。ListMLE Loss。データセット別モデル。XLA/NLP分割学習。
