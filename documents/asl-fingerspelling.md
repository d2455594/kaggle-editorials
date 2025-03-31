---
tags:
  - Kaggle
startdate: 2023-05-11
enddate: 2023-08-25
---
# Google - American Sign Language Fingerspelling Recognition
https://www.kaggle.com/competitions/asl-fingerspelling

**概要 (Overview)**

- **目的:** このコンペティションの目的は、アメリカ手話（American Sign Language - ASL）において、単語や固有名詞などを文字ごとに表現する**指文字（Fingerspelling）のシーケンス**を**ビデオ（またはそこから抽出されたランドマークデータ）から認識し、対応する文字列に書き起こす**モデルを開発することです。
- **背景:** 手話認識技術、特に複雑で高速な動きを含む指文字の認識精度を向上させることは、聴覚障がいのある人々のためのコミュニケーション支援において非常に重要です。リアルタイム翻訳ツール、教育ソフトウェア、ヒューマン・コンピュータ・インタラクションの改善など、多くの応用が期待されます。この種のコンペティションは、アクセシビリティ技術の向上を目指す取り組みの一環として行われることが多いです。
- **課題:** 指文字の認識は、その速度、隣接する文字の影響による形状の変化（Coarticulation）、個人差（手の形、速度、スタイル）、類似した形状の文字の存在、ビデオ品質（照明、角度、背景）など、多くの要因により困難なタスクです。モデルは、提供される（多くの場合ランドマーク形式の）時系列データから、これらの変動に対して頑健に正しい文字シーケンスを推定する必要があります。これは、ビデオ（動き）を入力とし、テキスト（文字列）を出力とする**シーケンス・トゥ・シーケンス（Sequence-to-Sequence）** の課題と捉えられます。

**データセットの形式 (Dataset Format)**

提供される主なデータは、指文字のシーケンスを示すビデオから抽出されたランドマーク（キーポイント）データと、それに対応する正解の文字列（フレーズ）です。

1. **トレーニングデータ:**
    
    - `train.csv`: 各指文字シーケンスのメタデータ。`sequence_id`（シーケンスの一意な識別子）、`participant_id`（手話を行った参加者のID）、そして**ターゲット変数**となる正解の文字列 `phrase` が含まれます。
    - `*.tfrecord` ファイル群: 各 `sequence_id` に対応する**時系列ランドマークデータ**が TensorFlow Record 形式で格納されています。これには通常、ビデオの各フレームにおける手（指の関節など）、顔、ポーズ（体）のキーポイントの座標（x, y, z）が含まれます。生のビデオデータではなく、これらのランドマーク情報が主に入力として用いられます。
    - `supplemental_metadata.csv`: 追加のメタデータが含まれる場合があります。
2. **テストデータ:**
    
    - トレーニングデータと同様の `.tfrecord` 形式で、未知の指文字シーケンスのランドマークデータが提供されます。
    - 正解の文字列 `phrase` は含まれません。参加者はこれらのランドマークシーケンスから `phrase` を予測します。
3. **`sample_submission.csv`**:
    
    - 提出フォーマットのサンプル。`sequence_id` と `phrase` の列を持ちます。参加者は、各 `sequence_id` に対して予測した文字列を `phrase` 列に記入して提出します。

**評価指標 (Evaluation Metric)**

- **指標:** **正規化された編集距離 (Normalized Edit Distance)**（レーベンシュタイン距離に基づく）
- **計算方法:**
    1. モデルが予測した文字列 (`prediction`) と、正解の文字列 (`truth`) の間の**レーベンシュタイン距離**（一方の文字列をもう一方の文字列に変換するために必要な最小の編集操作（挿入、削除、置換）回数）を計算します。
    2. 計算されたレーベンシュタイン距離を、**正解の文字列の長さ (`len(truth)`)** で割って正規化します。
    3. 最終的なスコアは、テストデータセット全体の**正規化された編集距離の平均値**となります。
- **意味:** この指標は、モデルが予測した指文字の書き起こしが、正解の文字列とどれだけ異なっているかを文字レベルの編集回数で評価します。音声認識や機械翻訳など、シーケンス変換タスクの評価に標準的に用いられます。スコアは**低い**ほど（0に近いほど）、予測が正解に近いことを意味し、モデルの性能が良いと評価されます。完全に一致する場合、編集距離は0となり、スコアも0になります。

要約すると、このコンペティションは、ASLの指文字を表すランドマークの時系列データから、元の文字列を書き起こすシーケンス・トゥ・シーケンスのタスクです。データはランドマークデータと正解文字列で構成され、性能は予測文字列と正解文字列間の正規化された編集距離（低いほど良い）によって評価されます。

---

**全体的な傾向**

上位解法では、Mediapipe等で抽出された手指、顔、ポーズなどのランドマーク（キーポイント）の時系列データを入力とし、手話の指文字（Fingerspelling）をテキストに変換するアプローチが主流でした。特に、自動音声認識（ASR）分野で用いられるTransformerベースのアーキテクチャ（Conformer, Squeezeformerなど）やアルゴリズム（CTC Loss, Attentionベースデコーダー、Joint CTC/Attention）を応用したものが多く見られます。モデルの過学習を防ぎ、未知の署名者への汎化性能を高めるために、多種多様なデータ拡張（Augmentation）が極めて重要なテクニックとして用いられました。時間軸方向の操作（タイムストレッチ、マスキング、CutMix）、空間的な操作（アフィン変換、フリップ）、特定のランドマーク（指、顔、ポーズ）を欠落させるDropoutなどが効果的でした。また、コンペティションの制約（TF-Liteモデルサイズ、推論時間）を満たすため、モデルの効率化（Rotary Embeddingsの採用、推論時のマスキング、キャッシュ利用）や、PyTorchで開発・学習したモデルをTensorFlow/TF-Liteに移植する作業も一般的でした。補助的なデータセットの利用や、後処理（信頼度の低い予測を固定文字列に置換するなど）、複数モデルのアンサンブルも有効な戦略でした。

**各解法の詳細**

**1位**

- **アプローチ:** 改良版Squeezeformer EncoderとTransformer Decoderの組み合わせ。多様なAugmentationによる正則化。信頼度予測による後処理。
- **アーキテクチャ:** Encoder: Squeezeformer (改良版、時間次元削減なし、Relative Positional EncodingをRotary Embeddings(Llama Attention)に変更)。 Decoder: Transformer Decoder (2層)。 Feature Extraction: 2D CNNベース (ランドマーク種別ごと+全体)。信頼度予測用Linear層。
- **アルゴリズム:** CrossEntropy Loss。補助損失 (逆順シーケンス予測)。MSE Loss (信頼度予測用、OOF予測に基づくターゲット)。Greedy Decoding。Rotary Embeddings (キャッシュ利用)。
- **テクニック:**
    - **データ前処理:** 130キーポイント使用（両手、腕ポーズ、顔）。正規化、NaNゼロ埋め。最大384フレームにリサイズ/パディング。
    - **Augmentation:** CutMix (同一署名者内)、Finger Dropout (指単位の欠落)、Face/Pose Dropout、TimeStretch、時間/空間マスキング、アフィン変換など多数。Decoder入力マスキングも有効。
    - **学習:** Cosine LRスケジュール、AdamW (推定)、混合精度(FP16)。訓練時マスキング（Padding無視）。
    - **効率化/変換:** Rotary Embeddings (RoPE) 導入で高速化・省メモリ化。Decoder推論時の早期停止とキャッシュ利用。PyTorch学習→TensorFlow移植→TF-Lite (FP16)。
    - **後処理:** 予測された信頼度スコアに基づき、低信頼度(<0.15)または短シーケンス(<15フレーム)の予測を固定文字列('2 a-e -aroe')に置換。
    - **補助データ:** 限定的な利用（エポックごとにグループから1サンプル追加）。
    - **アンサンブル:** 2シードのFullfitモデルのLogits平均。

**2位**

- **アプローチ:** ASRアルゴリズム（CTC, Attention, Transducer）を比較検討し、CTCとAttentionを組み合わせたJoint Training & Decodingを採用。前回のASLコンペ1位解法をベースにモデルを深化。
- **アーキテクチャ:** Encoder: Stacked Conv1DBlock + TransformerBlock (ASL Signコンペ1位ベース、サイズ・層数増加)。 Decoder: CTC Decoder (GRU+Linear) + Attention Decoder (Transformer Decoder 1層)。
- **アルゴリズム:** CTC Loss + CrossEntropy Loss (重み付け: 0.25/0.75)。Joint CTC/Attention Decoding (CTC Prefix Score利用)。Greedy Decoding。AWP (Adversarial Weight Perturbation)。
- **テクニック:**
    - **データ前処理:** 全ランドマーク使用。標準化。左利き署名者を反転。最大768フレーム。
    - **モデル:** EncoderはConv1D主体、Transformerブロック混合は効果薄。CTC/Attention Decoderは浅い層を使用。
    - **Augmentation:** Random Resample, Random Affine, Random Cutout, Decoder入力へのランダムトークン置換。長期間学習で効果を発揮するAugmentationを見逃した可能性。
    - **学習:** 400エポック。Cosine Decay LR、AdamW。AWP適用。
    - **効率化:** Attention Decoder推論時のキャッシュ利用。
    - **アンサンブル:** 3モデルのAttention Decoder出力平均が有効 (+0.009)。最終提出はJoint CTC/Attention。

**3位**

- **アプローチ:** Squeezeformer EncoderとCTC Decoderの組み合わせ。時間次元削減とRotary Positional Encoding (ROPE)で効率化。Stochastic Pathで正則化。
- **アーキテクチャ:** Encoder: Squeezeformer (17層、NEMO実装ベース、1層Downsamplingで時間次元削減)。 Decoder: CTC。
- **アルゴリズム:** CTC Loss。Rotary Positional Encoding (ROPE)。AWP。後処理ルール（Blank Index処理）。
- **テクニック:**
    - **データ前処理:** 全ランドマーク使用(手、唇、目、鼻、ポーズ)。入力特徴量に元座標＋正規化座標＋絶対位置を追加。最大320フレーム。
    - **Augmentation:** Time Scale (interp1d)、Time Dim Mask (シーケンス欠落＋50%フレームランダムマスク)、Affine (フリップ率低め)。
    - **学習:** 訓練データ+補助データを重み付け(0.1)して混合学習(400エポック)後、訓練データのみでファインチューニング(10エポック)。Linear Decay LR、Adam。AWP適用。
    - **モデル:** Stochastic Path (InstDropout付き)をEncoderブロックに適用。Dropoutは最終分類層(0.1)のみ。
    - **効率化/変換:** ROPEで高速化。PyTorch学習→NobucoでKeras変換→TF-Lite。TFRecord入力。

**4位**

- **アプローチ:** Conformer EncoderとTransformer Decoderの組み合わせ。Edit Distance最適化を意識した学習と推論（MWER、Beam Search）。
- **アーキテクチャ:** Encoder: Conformer (12層)。 Decoder: Transformer Decoder (2層、軽量)。
- **アルゴリズム:** CrossEntropy Loss。MWER (Minimum Word Error Rate) Training (文字ベースEdit Distance使用)。Beam Search Decoding (TF-Lite互換実装、キャッシュ利用、Linear Length Penalty)。RAdam optimizer。
- **テクニック:**
    - **データ前処理:** 214入力（両手、ポーズ、鼻、唇のx,y）。正規化、NaNゼロ埋め、差分特徴量(t-1, t-2)。最大500フレーム。
    - **Augmentation:** フリップ、リサンプル、アフィン変換、時間マスキング(最大60%)、空間Cutout。トークンAugmentation（ランダム置換）。
    - **学習:** Cosine LR、RAdam。Label Smoothing、Gaussian Weight Noise。補助データ使用(初期100エポック)。MWER Training (Beam Searchで候補生成→Edit Distanceで重み付け→学習)。
    - **推論:** Beam Search (k=5~6)。キャッシュ利用。長さペナルティ。
    - **後処理:** 低情報量サンプル（短フレーム数、手検出フレーム少）の予測を固定文字列(" a-e -are")に置換。

**5位**

- **アプローチ:** Vanilla Transformer Encoderに事前学習(Data2vec 2.0)を適用。CTC SegmentationとTemporal CutMix Augmentation、Knowledge Distillationが特徴。
- **アーキテクチャ:** Encoder: Vanilla Transformer (24層、Conv Stem、Rotary Position Embedding/RoPE)。 Decoder: CTC (Linear層)。
- **アルゴリズム:** Data2vec 2.0 (事前学習)、CTC Loss、Knowledge Distillation (DeiT風)。
- **テクニック:**
    - **データ前処理:** 3Dランドマーク使用。アスペクト比推定と補正。主要な手のみ使用。使用ランドマーク（手、ポーズ、唇）。
    - **Augmentation:** 時間反転、リサンプル、フレームブロックマスク、フレームノイズ、特徴マスク、アフィン変換（各部位に個別適用）、Temporal CutMix (CTC Segmentationに基づく時間アライメント利用)。
    - **事前学習:** Data2vec 2.0をランドマークデータに適用（複数マスクパターン利用）。
    - **学習:** LayerDrop。CutMix Augmentationが性能向上に大きく寄与。補助データはCutMix併用で効果あり。
    - **Knowledge Distillation:** 事前学習済みTransformer Large (Teacher) からTransformer Small (Student) へ知識蒸留（Hard Label予測）。KD用とCTC用に別々のHeadを使用。
    - **CV戦略:** 単純なHold-out (5%)。GroupKFoldはLBとの相関悪し。

**9位**

- **アプローチ:** ASL Signコンペ1位解法ベースのConv1D+Transformer EncoderとCTC Decoder。長期学習と重み平均化。
- **アーキテクチャ:** Encoder: Conv1DBlock (2 Conv1D) + TransformerBlock (6ブロック構成)。 Decoder: CTC。
- **アルゴリズム:** CTC Loss。重み平均 (Weight Averaging)。
- **テクニック:**
    - **データ前処理:** RHAND, LHAND, LIP, POSE, REYE, LEYEのx, y, z座標。標準化。最大356フレーム。非手フレームのダウンサンプリング。
    - **Augmentation:** フリップ、リサンプリング、回転、時間/空間マスキング、時間クロッピング、ノイズ、スケーリングなど重度Augmentation。
    - **学習:** 長期学習 (500+エポック)。段階的にAugmentation導入。補助データ、外部データ（ChicagoWild/Plus）も利用。
    - **モデルサイズ拡張:** 学習済みモデルの最終ブロックの重みをコピーして層数を増やし、再学習時間を短縮。
    - **後処理:** Fill word置換（<30フレーム かつ 予測<=6文字の場合、固定文字列'+w1- ea-or'に置換）。
    - **重み平均:** 最終数エポックのモデル重みを平均化。

