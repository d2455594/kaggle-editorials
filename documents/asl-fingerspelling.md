---
tags:
  - Kaggle
startdate: 2023-05-11
enddate: 2023-08-25
---
# Google - American Sign Language Fingerspelling Recognition
[https://www.kaggle.com/competitions/asl-fingerspelling](https://www.kaggle.com/competitions/asl-fingerspelling)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、アメリカ手話 (ASL) の**指文字 (Fingerspelling)** を行っている動画から抽出された手の動きのデータ（ランドマーク）をもとに、綴られている**文字列を正確に認識する**モデルを開発することです。
* **背景:** GoogleのAI for Social Goodプログラムの一環として開催されました。指文字は、手話の語彙にない固有名詞などを表現するために不可欠な要素であり、手話認識・翻訳技術の精度向上において重要な課題です。前年に開催されたASL Signsコンペ（個々の手話単語認識）に続く、より連続的で複雑な認識タスクとなります。
* **課題:**
    * **連続動作の認識とセグメンテーション:** 指文字は一連の流れるような手の動きであり、文字と文字の間の明確な区切りがないため、時間的な連続性を捉え、個々の文字を分離・認識する必要があります。
    * **話者による多様性:** 指文字の速度、明瞭さ、手の形や動きの大きさは話者によって大きく異なります。
    * **視点とオクルージョン:** 記録時のカメラアングルや手の位置によってランドマークの見え方が変化し、指の一部が隠れる（オクルージョン）こともあります。
    * **データの品質:** Mediapipeによって抽出されたランドマークデータにはノイズや欠損が含まれる可能性があります。また、データセットには非常に短いシーケンスや、手の情報がほとんどないシーケンスも含まれます。
    * **TFLiteモデルの制約:** 提出モデルはTensorFlow Lite形式である必要があり、モデルサイズ (40MB) と使用できる演算子 (Ops) に厳しい制限があります。これにより、複雑なモデルアーキテクチャやデコードアルゴリズムの実装が困難になります。
    * **推論時間制限:** 5時間という推論時間制限内で、テストデータ全体の予測を完了する必要があります。

**データセットの形式 (Dataset Format)**

提供されるデータは、主に指文字動画から抽出されたランドマークの時系列データと、対応する正解文字列です。

1.  **トレーニングデータ:**
    * `train.csv`: 各訓練シーケンスのメタデータ。
        * `path`: ランドマークデータファイル (`.parquet`) への相対パス。
        * `sequence_id`: 各シーケンスの一意なID。
        * `phrase`: **ターゲット変数**。指文字で綴られた正解の文字列。
        * `participant_id`: 指文字を行った話者のID。
    * `supplemental_metadata.csv` & `supplemental_landmarks/`: 追加のランドマークデータ（ラベルなし、または弱いラベル）。
    * `character_to_prediction_index.json`: 文字とモデル出力インデックス（整数）のマッピング定義。
    * `*.parquet` ファイル (各 `path` に対応): Mediapipeで抽出されたランドマークの時系列データ。
        * `frame`: フレーム番号。
        * `row_id`: `frame`-`type`-`landmark_index` 形式のID。
        * `type`: ランドマークの種類 (`face`, `left_hand`, `pose`, `right_hand`)。
        * `landmark_index`: 各タイプ内でのランドマークのインデックス番号。
        * `x`, `y`, `z`: 正規化されたランドマーク座標。zは深度を表すが、信頼性は高くないとされる。
2.  **テストデータ:**
    * `test_landmarks/`: テスト用のランドマークデータファイル (`.parquet`)。
    * (隠しデータ): テストデータに対応する正解の `phrase` 文字列。
3.  **`sample_submission.csv`**:
    * 提出フォーマットのサンプル。
        * `row_id`: テストシーケンスの `sequence_id`。
        * `phrase`: モデルによって予測された指文字の文字列。

**評価指標 (Evaluation Metric)**

* **指標:** **正規化レーベンシュタイン距離 (Normalized Levenshtein Distance)**
* **計算方法:**
    * モデルが予測した文字列 (`prediction`) と、正解の文字列 (`ground_truth`) の間のレーベンシュタイン距離 (編集距離: 文字の挿入、削除、置換の最小回数) を計算します。
    * 計算された編集距離を、**正解文字列の文字数 (`len(ground_truth)`)** で割ります。
    * `NormalizedLevenshteinDistance = LevenshteinDistance(prediction, ground_truth) / len(ground_truth)` (ただし、`len(ground_truth)` が0の場合は1とするなどの処理が必要)。
* **意味:** 予測された文字列が、正解の文字列と文字レベルでどれだけ異なっているかを評価する指標です。文字数で正規化することで、長い文字列と短い文字列の間で公平に比較できます。スコアは**低い**ほど良い（0が完全一致）性能を示します。

要約すると、このコンペティションは、手のランドマーク時系列データから指文字で綴られた文字列を認識するシーケンス・トゥ・シーケンスのタスクです。データはランドマーク座標と正解文字列で構成され、性能は正規化レーベンシュタイン距離（低いほど良い）で評価されます。TFLiteモデルの制約（サイズ、Ops、推論時間）への対応が重要な課題です。

---

**全体的な傾向**

このアメリカ手話指文字認識コンペティションでは、前回のASL Signsコンペの知見を活かしつつ、連続的なシーケンスを扱うための工夫が求められました。上位解法の多くは、音声認識 (ASR) 分野で実績のある**シーケンス・トゥ・シーケンス (Seq2Seq) モデル**、特に **Transformer** ベースのアーキテクチャを採用しました。**Conformer** や **Squeezeformer** といった ASR 向けの強力なエンコーダー構造をランドマークデータに適用する試みが成功を収めています。

**エンコーダー・デコーダー構造**が主流であり、ランドマークの時系列特徴量をエンコーダーで処理し、その出力を Transformer デコーダーや RNN (LSTM/GRU) デコーダーに入力して文字シーケンスを生成する構成が一般的でした。また、フレーム単位で文字確率を出力する **CTC (Connectionist Temporal Classification)** も依然として有効なアプローチであり、特に **CTC と Attention デコーダーを組み合わせたハイブリッドモデル**が高い性能を示しました。

入力としては、**両手 (left_hand, right_hand)** のランドマークに加え、**ポーズ (pose)** や**顔（特に唇 lip）** のランドマークを含めることが精度向上に寄与しました。生の座標 (x, y, z) だけでなく、時間差分（速度・加速度）特徴量や、適切な正規化、欠損値処理が重要でした。3D座標を活用し、回転などの**3次元空間でのデータ拡張**を行うアプローチも有効でした。

**データ拡張 (Augmentation)** は過学習抑制と汎化性能向上のために極めて重要でした。時間軸方向の操作（リサンプリング、シフト、ワーピング、マスキング）、空間的な操作（アフィン変換、左右反転）、特定の部位のドロップアウト（指、顔、ポーズ）、そして **CutMix**（時間軸上で異なるサンプルを混合）などが効果を発揮しました。

学習テクニックとしては、**長いシーケンスの扱**い（パディングとマスキング、リサイズ）、**補助損失**（例: 逆順シーケンス予測）、**AWP (Adversarial Weight Perturbation)**、**EMA/SWA**、**LayerDrop** などが採用されました。

デコード（予測文字列生成）においては、Greedy Search に加えて **Beam Search** を用いることでスコアが向上しましたが、TFLite の制約下で効率的に実装する必要がありました。また、信頼度の低い予測や入力データが極端に短い場合に、**固定の文字列（例: "2 a-e -aroe"）で置き換える**後処理が多くのチームで有効でした。

モデルの **TFLite 変換**は必須であり、使用可能な演算子（Ops）の制限内でモデルを設計・実装する必要がありました。モデルサイズの制約 (40MB) を満たすために、**fp16 量子化**も一般的に行われました。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/asl-fingerspelling/discussion/434485)**

* **アプローチ:** Squeezeformer Encoder + Transformer Decoder。信頼度予測と後処理。
* **アーキテクチャ:** 修正Squeezeformer (時間U-Netなし、RoPE使用、事前LayerNorm代替) + 2層Transformer Decoder。
* **アルゴリズム:** Seq2Seq (Attentionベース)。
* **テクニック:** 入力130点(手x2, ポーズ, 顔)。複数特徴抽出モジュール(全体+部位別)。Augmentation (CutMix, **FingerDropout**, TimeStretch, Affine, Masking等多数)。逆順シーケンス補助損失。**信頼度スコア予測**。後処理 (低信頼度/短シーケンスを定型句置換)。PyTorch学習→TF変換→TFLite (fp16)。

**[2位](https://www.kaggle.com/competitions/asl-fingerspelling/discussion/434588)**

* **アプローチ:** Conformer/Transformer Encoder + **CTC/Attention Joint Training & Decoding**。
* **アーキテクチャ:** Stacked Conv1D + Transformer Encoder (17層), 1層GRU (CTC用), 1層Transformer Decoder (Attention用)。
* **アルゴリズム:** Seq2Seq (CTC + Attention ハイブリッド)。
* **テクニック:** 入力214点(手x2, ポーズ, 唇, 鼻, 目)。差分特徴量。Augmentation (Flip, Resample, Affine, Masking)。Decoder入力トークン置換Augmentation。ラベルスムージング。**CTC Prefix Scoreを用いたBeam Search**。AWP。400エポック学習。

**[3位](https://www.kaggle.com/competitions/asl-fingerspelling/discussion/434393)**

* **アプローチ:** Squeezeformer Encoder + CTC。時間次元削減とRoPE。
* **アーキテクチャ:** Squeezeformer (NEMOベース、17層) + CTC Decoder。Encoder内で時間次元削減 (320->160)。
* **アルゴリズム:** CTCベース。
* **テクニック:** 入力 手+唇+目+鼻+ポーズ (x,y,z) + 正規化 + 絶対位置。入力フレーム数320。Augmentation (Time Scale, Time Mask(50%フレーム), Affine)。**RoPE (Rotary Positional Embedding)**。**Stochastic Depth (InstDropout)**。Dropoutは最終層のみ。CTC後処理(ブランク処理ルール)。補助データ併用学習+ファインチューニング。AWP。PyTorch学習→NobucoでKeras変換→TFLite。

**[4位](https://www.kaggle.com/competitions/asl-fingerspelling/discussion/434983)**

* **アプローチ:** Conformer Encoder + Transformer Decoderアンサンブル。MWER学習とBeam Search。
* **アーキテクチャ:** Conformer Encoder (12層) + 2層Transformer Decoder。
* **アルゴリズム:** Seq2Seq (Attentionベース)。
* **テクニック:** 入力 手x2+ポーズ+唇+鼻。差分特徴量。Augmentation (Flip, Resample, Affine, Spatial Cutout, Frame Masking)。**ターゲットトークンAugmentation** (ランダム置換)。**Minimum Word Error Rate (MWER) Training**。**Beam Search** (k=5/6、TFLite互換実装、長さペナルティ)。後処理 (低情報量サンプルを定型句置換)。

**[5位](https://www.kaggle.com/competitions/asl-fingerspelling/discussion/434415)**

* **アプローチ:** Vanilla Transformer + Data2vec事前学習 + CTC + CutMix + KD。
* **アーキテクチャ:** Vanilla Transformer Encoder (24層, 256次元) + Conv Stem + CTC Decoder。
* **アルゴリズム:** CTCベース。
* **テクニック:** **Data2vec 2.0**による事前学習。入力 手+ポーズ+唇 (3D)。アスペクト比補正 + **3D空間Augmentation** (Rotation, Shear, Scale, Shift)。**CTC SegmentationによるCutMix**。**知識蒸留** (Largeモデル → Smallモデル)。LayerDrop。RoPE。補助データ併用学習。

**[9位](https://www.kaggle.com/competitions/asl-fingerspelling/discussion/434871)**

* **アプローチ:** Conv1D+Transformer Encoder + CTCアンサンブル。
* **アーキテクチャ:** Stacked Conv1D + Transformer Blocks (6ブロック) + CTC Decoder。
* **アルゴリズム:** CTCベース。
* **テクニック:** 入力 手+唇+ポーズ+目。Max Frame 356。非手フレームのダウンサンプリング。Augmentation (Flip, Resample, Rotate, Masking, Noise, Scale)。補助データ/外部データ併用学習。複数エポック重み平均 (SWA風)。**モデルサイズ拡張** (学習済み最終層をコピーして層追加)。後処理 (短フレーム/短予測を定型句置換)。
