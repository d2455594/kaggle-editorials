---
tags:
  - Kaggle
  - Transformer
  - DeBERTa
  - 教育
  - NER
startdate: 2024-01-18
enddate: 2024-04-24
---
# The Learning Agency Lab - PII Data Detection
[https://www.kaggle.com/competitions/pii-detection-removal-from-educational-data](https://www.kaggle.com/competitions/pii-detection-removal-from-educational-data)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、学生が作成したエッセイの中から、**個人を特定できる情報（Personally Identifiable Information, PII）** を自動的に検出し、その種類を分類する機械学習モデルを開発することです。
* **背景:** 教育データの活用が進む中で、学生のプライバシー保護は極めて重要です。特に、エッセイのような自由記述形式のテキストには、意図せずPIIが含まれるリスクがあります。従来の手作業によるPIIの検出・匿名化はコストと時間がかかるため、AIによる高精度な自動検出技術が求められています。このコンペは、教育分野における安全なデータ活用を支援することを目的としています。
* **課題:** 与えられたテキスト（エッセイ）をトークンレベルで分析し、PIIに該当するトークンとその範囲、およびPIIの種類（名前、メールアドレス、ID番号など7種類）を特定する**固有表現抽出（Named Entity Recognition, NER）タスク**です。自然言語の多様性や曖昧さ（一般的な単語と同じ文字列が名前として使われる場合など）に対応し、文脈を考慮してPIIを正確に識別する必要があります。また、学習データには存在しないか、ごく少数しか含まれないタイプのPII（例: 電話番号）も検出できる**汎化性能**が求められました。

**データセットの形式 (Dataset Format)**

提供される主なデータは、エッセイテキストとそれに対応するPIIラベルです。

1.  **トレーニングデータ:**
    * `train.json`: 約6,800件のエッセイデータが含まれるJSONファイル。各エッセイは以下のキーを持つ辞書のリストです。
        * `document`: エッセイを一意に識別するID。
        * `full_text`: エッセイの全文。
        * `tokens`: テキストを空白や句読点などで分割したトークンのリスト。
        * `trailing_whitespace`: 各トークンの後に続く空白文字（スペース、改行など）を示す真偽値のリスト。元のテキスト形式の復元に使用されます。
        * `labels`: 各`tokens`に対応するPIIラベルのリスト。**BIO形式** (`B-TYPE`, `I-TYPE`, `O`) で記述されます。
            * `O`: PIIではないトークン。
            * `B-TYPE`: PIIタイプ`TYPE`の開始トークン。
            * `I-TYPE`: PIIタイプ`TYPE`の継続トークン。
            * PIIタイプ (`TYPE`) としては、`NAME_STUDENT`, `EMAIL`, `USERNAME`, `ID_NUM`, `PHONE_NUM`, `URL_PERSONAL`, `STREET_ADDRESS` の7種類があります。
2.  **テストデータ:**
    * `test.json`: トレーニングデータと同様の形式ですが、`labels`キーは含まれません。モデルはこのデータに対してPIIラベルを予測します。
3.  **外部データ / 合成データ:**
    * コンペ提供のトレーニングデータだけではサンプル数が限られており、特に稀なPIIタイプの学習が困難でした。そのため、多くの参加者は**大規模言語モデル（LLM）**（Mistral, Mixtral, Llama 3, Gemini, GPTなど）を用いて、**PIIを含む合成エッセイデータを大量に生成**し、トレーニングデータとして活用しました。
    * Kaggle Datasets上で他の参加者が生成・共有した合成データセット（例: [@nbroad]([https://www.kaggle.com/nbroad)氏](https://www.google.com/search?q=https://www.kaggle.com/nbroad)%E6%B0%8F)、[@mpware]([https://www.kaggle.com/mpware)氏のデータセット]([https://www.google.com/search?q=https://www.kaggle.com/mpware)%E6%B0%8F%E3%81%AE%E3%83%87%E3%83%BC%E3%82%BF%E3%82%BB%E3%83%83%E3%83%88](https://www.google.com/search?q=https://www.kaggle.com/mpware)%E6%B0%8F%E3%81%AE%E3%83%87%E3%83%BC%E3%82%BF%E3%82%BB%E3%83%83%E3%83%88))）も広く利用されました。
4.  **`sample_submission.csv`**:
    * 提出フォーマットのサンプル。`document`, `token` (トークンのインデックス), `label` (予測されたBIO形式のラベル) の列を持ちます。テストデータの全トークンに対して予測ラベルを提出する必要があります。

**評価指標 (Evaluation Metric)**

* **指標:** **エンティティレベル F5スコア (Entity-level Fbeta score with β=5)**
* **計算方法:**
    1.  モデルの予測結果（トークンごとのBIOラベル）から、連続する`B-TYPE`と`I-TYPE`のトークン列をPIIエンティティとして抽出します。
    2.  予測されたPIIエンティティが、真のPIIエンティティと**完全に一致**する場合（ラベルタイプとトークンの範囲の両方が一致）にTrue Positive (TP) とカウントします。
    3.  Precision (適合率 = TP / (TP + FP)) と Recall (再現率 = TP / (TP + FN)) を計算します。FPは誤ってPIIと予測されたエンティティ、FNは見逃されたPIIエンティティです。
    4.  β=5としてFbetaスコアを計算します: F5 = (1 + 5²) * (Precision * Recall) / ((5² * Precision) + Recall) = 26 * (Precision * Recall) / (25 * Precision + Recall)
* **意味:** F5スコアは、Recall（再現率）をPrecision（適合率）よりも**5倍重視**する指標です。これは、PII検出タスクにおいて、**偽陽性（誤検出）を多少許容してでも、偽陰性（見逃し）を可能な限り減らすこと**が重要であるというコンペの目的に合致しています。個人情報保護の観点から、見逃しのリスクを低く抑えることが極めて重視されます。スコアは**高い**ほど良い評価となります。

---

**全体的な傾向**

このコンペティションは、テキストからの固有表現抽出（NER）タスクであり、特にPIIの見逃しを厳しく評価するF5スコアが指標でした。上位解法の多くは、**TransformerベースのEncoderモデル**、中でも**DeBERTa-v3-large**を主軸としていました。

最も重要な成功要因の一つは、**外部データおよび合成データの活用**でした。コンペ提供のデータだけでは不足しており、LLMを用いて生成された大量の合成データセット（数千〜数万サンプル）を学習に組み込むことが一般的でした。これにより、特に学習データ中に少ないPIIタイプの検出能力やモデルの汎化性能が向上しました。

モデルアーキテクチャとしては、DeBERTa-v3-largeがデファクトスタンダードでしたが、多様性を確保するために**BiLSTM/GRUレイヤーの追加**や**Multi-Sample Dropout**などの工夫を施したモデルもアンサンブルに用いられました。**知識蒸留**も有効な手法として利用されました。入力テキストが長いため、**Max Length**を大きく設定し（1024〜4096）、**ストライド（オーバーラップ）**を用いて文書全体を処理する手法が一般的でした。

学習においては、クラス不均衡（特に'O'ラベルが多い）に対処するため、PIIラベルの**損失の重みを上げる**、あるいは'O'ラベルの重みを下げる**重み付き損失関数**（CrossEntropyLossやFocal Loss）が広く使われました。

評価指標F5スコアを最大化するためには、モデルの予測結果に対する**後処理 (Post-processing)** が極めて重要でした。単純な確率閾値だけでなく、PIIタイプごとに**異なる閾値**を設定したり、**ルールベース**での修正（例: NAME_STUDENTが大文字で始まらない場合は除去、STREET_ADDRESS内の改行補完、ID_NUMの桁数チェックなど）を適用したり、同一文書内でのラベル整合性を取る（同じ文字列は同じラベルにする）などの処理が行われました。さらに、モデルが見逃した特定のパターン（URL, EMAILなど）を**正規表現**で補完することも有効でした。

最終的な提出は、異なる設定やアーキテクチャで学習された**複数のモデルの予測をアンサンブル**（加重平均や多数決）し、その結果に後処理を適用する形が一般的でした。アンサンブルの重みや後処理の閾値は、交差検証（CV）の結果に基づいてOptunaなどで最適化されました。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/pii-detection-removal-from-educational-data/discussion/497374)**

* **アプローチ:** DeBERTa-v3-largeの多様なバリアント（Multi-Sample Dropout, BiLSTM Head, Knowledge Distillation）を含むアンサンブル。強力な後処理。
* **アーキテクチャ:** DeBERTa-v3-largeベース。Multi-Sample Dropout追加版、BiLSTMヘッド追加版。
* **アルゴリズム:** Token Classification。CrossEntropyLoss (Oラベルの重み=0.05)。Knowledge Distillation (KLDivLoss)。AdamW?
* **テクニック:**
    * **データ:** nbroad氏、mpware氏の外部データ、および自作データ(2k)を使用。
    * **学習:** Max Length 1600-2048。3-4 epochs。LR 1e-5。Knowledge Distillationで性能向上 (+0.005-0.01)。
    * **後処理:** クラス別閾値。ルールベース修正（大文字でない名前除去、PHONE_NUM→ID_NUM変換、STREET_ADDRESSの改行補完、USERNAME連結、ID/URL/EMAILの長さ/フォーマットチェック）。同一文書内でのNAME_STUDENT統一。Regexによる補完。
    * **アンサンブル:** 7グループ10モデルの重み付きVoting Ensemble（重みと閾値はOptunaで最適化）。

**[2位](https://www.kaggle.com/competitions/pii-detection-removal-from-educational-data/discussion/497352)**

* **アプローチ:** DeBERTa-v3-largeモデルのシンプルなアンサンブル（バギング）。トークナイザの差を吸収する前処理と、ルールベースの後処理。
* **アーキテクチャ:** DeBERTa-v3-large。
* **アルゴリズム:** Token Classification (B-/I-プレフィックス除去、7クラス分類)。CrossEntropyLoss? AdamW + Cosine Annealing LR。
* **テクニック:**
    * **前処理:** HuggingFaceの `is_split_into_words=True` オプションで、提供されたトークンリストを直接利用。空白トークンは無視（デフォルトで'O'予測）。B-/I-プレフィックスを除去。
    * **データ:** nbroad氏の外部データを0.5の重みで混合。
    * **学習:** Max Length 512, 1024, 2048。Stride 32。LR 2e-5。4 epochs。
    * **後処理:**
        * NAME_STUDENT統一: あるトークンがNAME_STUDENTと予測されたら、同一文書内の同じ文字列トークンもNAME_STUDENTにする。長さ1やTitle Caseでない場合は除去。
        * STREET_ADDRESS改行補完: "\n" トークンをSTREET_ADDRESSに変更。
        * 閾値調整: 'O'クラスの確率をスケールダウン（係数0.02-0.03）してからargmax。
    * **アンサンブル:** 6つのモデル（異なるMax Length, Strideで学習）の予測確率を平均。

**[3位](https://www.kaggle.com/competitions/pii-detection-removal-from-educational-data/discussion/497482)**

* **アプローチ:** 単一のDeBERTa-v3-largeバックボーンを複数シードで学習しアンサンブル。2段階学習と後処理。
* **アーキテクチャ:** DeBERTa-v3-large。
* **アルゴリズム:** Token Classification。CrossEntropyLoss? AdamW?
* **テクニック:**
    * **データ:** mpware氏のデータセットで事前学習後、コンペデータ+nbroad氏データセットでファインチューニング。
    * **学習:** LR 1e-5。Batch Size 2, Grad Accum 2。Warmup 0.1。2 epochs。Tokenizerに改行・タブ文字などを追加。
    * **検証:** Stratified 5 folds (PII有無で層化)。訓練データに少ないPIIクラスの性能評価用に別途生成/外部データを使用。
    * **後処理:** 'O'ラベル閾値調整。ルールベースFP除去（短い予測、敬称、一般的講師名など）。同一文書内でのNAME_STUDENTラベル修正。

**[4位](https://www.kaggle.com/competitions/pii-detection-removal-from-educational-data/discussion/497367)**

* **アプローチ:** DeBERTa-v3-largeモデルのアンサンブル。Llama 3 70B Instructによる高品質なデータ生成が鍵。テキスト前処理とBiLSTM/GRUヘッドの追加。
* **アーキテクチャ:** DeBERTa-v3-large + BiLSTM/GRUヘッド。
* **アルゴリズム:** Token Classification。Focal Loss。AdamW + Cosine Scheduler。
* **テクニック:**
    * **データ生成:** **Llama 3 70B Instruct** を使用し、高品質な合成データを生成（これが単体モデル性能を大幅に向上）。nbroad氏のデータも使用。False Positiveとなった教師名などを含むサンプルをパラフレーズして追加。
    * **テキスト前処理:** 空白文字を特殊トークン `[SPACE]` に置換。Unicode正規化 (unidecode)。
    * **学習:** Max Length 1280。3 epochs。LR 1e-5 or 2e-5。PyTorchで実装 (HF Trainer不使用)。
    * **推論:** Max Length 4000, Stride 1024で推論するのが最適だった（訓練時より長い）。
    * **CV戦略:** Stratified 5 fold (PII有無で層化)。
    * **後処理:** 数字を含むNAME_STUDENT除去、非難読化された講師名除去など、最小限のルールを適用。他の複雑な後処理は効果なしと判断。
    * **アンサンブル:** 10個のDeBERTa-v3-largeモデル (5 Fold x 2) をアンサンブル。

**[5位](https://www.kaggle.com/competitions/pii-detection-removal-from-educational-data/discussion/497306)**

* **アプローチ:** 3人のメンバーがそれぞれ異なるMax Length (128, 512, 1536) でDeBERTa-v3-largeモデルを学習し、それらをVotingアンサンブル。
* **アーキテクチャ:** DeBERTa-v3-large。一部LSTMヘッド追加。一部モデルはEmbedding/初期レイヤーを凍結。
* **アルゴリズム:** Token Classification。CrossEntropyLoss (Oラベルの重み0.1 or PIIラベルの重みx5)。AdamW + Cosine Scheduler。
* **テクニック:**
    * **データ:** nbroad氏、mpware氏、pjmathematician氏、valentinwerner氏の外部/合成データを活用。データセットの組み合わせを変えて学習。
    * **学習:** メンバーごとに異なるMax Length (128, 512, 1536) で学習し、多様性を確保。Positional Feature（絶対位置、相対位置）を追加（ryotakパート）。EMA使用（ryotakパート）。
    * **推論:** 長いMax Length (4096) での推論が有効（minfukaパート）。AMPで推論高速化。
    * **後処理:** 各メンバーが個別に行った後、アンサンブル。ルールベースFP除去（空白、プレフィックス整合性、Regex、大文字でない名前、冠詞/前置詞に続く名前など）。
    * **アンサンブル:** 各メンバー内でアンサンブル後、最終的にメンバー間の予測をSimple Voting。

**[6位](https://www.kaggle.com/competitions/pii-detection-removal-from-educational-data/discussion/498304)**

* **アプローチ:** DeBERTa-v3-largeとbaseモデルのアンサンブル。トークナイザ間のマッピングを行うカスタムヘッド。Mistralによるデータ生成。
* **アーキテクチャ:** DeBERTa-v3-large, DeBERTa-v3-base。カスタムヘッド（DeBERTaトークン予測→文字単位予測→Spacyトークン予測）。
* **アルゴリズム:** Token Classification (8クラス、BIOプレフィックスなし)。Focal Loss (Oラベル重み0.1)。カスタムヘッドからのAuxiliary Loss、開始/終了トークン予測のBCE Lossも追加。
* **テクニック:**
    * **データ生成:** Mistral-7B-Instruct-v0.2をファインチューニングし、ランダムなツール名やFaker生成PIIを組み合わせて合成データを生成。公開共有データセットも併用。
    * **カスタムヘッド:** DeBERTaトークナイザと提出形式のSpacyトークナイザの差異を吸収するため、文字レベルの予測を経由してマッピング。`scatter_reduce_` (mean) を使用。
    * **学習:** モデルごとに異なるデータセット組み合わせ、Loss構成で学習。Max Length 4600。
    * **推論:** Max Length 6300。
    * **後処理:** 空白文字を'O'に設定。STREET_ADDRESS内の改行補完。同一文書内NAME_STUDENT統一。
    * **アンサンブル:** DeBERTa-v3-large (2種) と DeBERTa-v3-baseの計12モデル（4 Fold x 3）を重み付き平均。

**[7位](https://www.kaggle.com/competitions/pii-detection-removal-from-educational-data/discussion/497310)**

* **アプローチ:** 多数（14個）の公開モデルのアンサンブル。時間制限に基づき、短いテキストには多くのモデルを、長いテキストには少数のモデルを適用する推論戦略。
* **アーキテクチャ:** 公開されているDeBERTaモデル（複数）。
* **アルゴリズム:** Token Classification。
* **テクニック:**
    * **モデル:** 複数の公開データセットから学習済みモデルを利用。
    * **推論戦略:** 推論時間（約8.5時間）を考慮。最初の3時間は全モデルで全データ予測。残りの5.5時間は、短いテキスト（2/3）に対して残りのモデルで追加予測。テキスト長に応じて適用モデル数を変える。
    * **閾値調整:** ラベルタイプごとに異なる閾値（NAME_STUDENTは低め、他は高め）。
    * **後処理:** NAME_STUDENTは大文字開始+小文字のみ。単一B-トークンで短すぎるもの、フォーマット不正（電話、メール、ID）を除去。STREET_ADDRESS内の改行補完。
    * **アンサンブル:** モデル予測のアンサンブル（詳細は不明、VotingかAverageか）。

**[9位](https://www.kaggle.com/competitions/pii-detection-removal-from-educational-data/discussion/497177)**

* **アプローチ:** DeBERTa-v3-large, DeBERTa-v2-xlargeのアンサンブル。Mixtralによる改良版データセット。文字レベルでのトークナイザ差分吸収。
* **アーキテクチャ:** DeBERTa-v3-large (通常版、LayerDrop+MultiSample Dropout版)、DeBERTa-v2-xlarge (LoRA)。
* **アルゴリズム:** Token Classification。
* **テクニック:**
    * **データ:** Mixtral 8x7Bで自作した改良版データセットを使用（有名人の引用、チームメイト名、個人サイトURLなどのパターンを追加）。コンペデータのラベルエラー約30箇所を修正。
    * **学習:** Max Length 512, Stride 128。4 Fold学習後に全データ学習。毎エポックで名前をランダムに置換するコールバックを使用。
    * **トークナイザ対応:** DeBERTa v2とv3のトークナイザ差異に対応するため、文字レベルにマッピングしてから最終的なSpacyトークンレベルの予測を得る（最大値プーリング）。
    * **後処理:** 名前はTitle Caseで英字/ピリオドのみ。敬称除去。同一文書内での名前ラベル統一。B-/I-プレフィックス整合性チェック。特定URL（Coursera, Wikipedia, .edu）除去。電話番号→ID番号変換ルール。電話番号のRegexチェック。
    * **アンサンブル:** 上記3モデルのアンサンブル。
