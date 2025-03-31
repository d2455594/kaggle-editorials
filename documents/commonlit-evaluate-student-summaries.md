---
tags:
  - Kaggle
startdate: 2023-07-13
enddate: 2023-10-12
---
# CommonLit - Evaluate Student Summaries
https://www.kaggle.com/competitions/commonlit-evaluate-student-summaries

**概要 (Overview)**

- **目的:** このコンペティションの目的は、生徒（3年生から12年生）が与えられた**課題テキスト（プロンプト）**に基づいて作成した**要約文**の質を、自動的に評価するモデルを開発することです。評価は、特に**内容（Content）**と**表現（Wording）**の2つの側面から行われます。
- **背景:** 生徒の文章、特に要約能力を評価することは、教育者にとって時間のかかる重要な作業です。正確な自動評価システムは、生徒への迅速なフィードバック提供、教員の負担軽減、そして個々の生徒の改善点（内容の網羅性、表現の明瞭さなど）を特定することによる個別学習支援に貢献する可能性があります。CommonLitは読解力向上を支援するNPOであり、このコンペティションは教育現場での応用を強く意識しています。
- **課題:** 文章の質の評価には本質的な主観性が伴いますが、このコンペでは複数の評価者によって付けられたスコアを使用し、より客観的な評価指標を目指します。モデルは、要約の質を構成する複数の側面、すなわち主要なアイデアを正確に捉えているか（内容）と、文章の明瞭さ、文法、語彙、簡潔さ（表現）の両方を評価する必要があります。また、対象となる生徒の学年や能力は幅広いため、要約文の長さ、スタイル、誤りの種類なども多様であることに対応しなければなりません。最終的な目標は、人間の専門家の評価と高い相関を持つモデルを構築することです。これは**回帰（Regression）** タスクであり、入力テキストに基づいて連続値のスコアを予測します。

**データセットの形式 (Dataset Format)**

提供される主なデータは、生徒が読んだ課題テキスト、生徒が作成した要約文、そしてその要約文に対する人間による評価スコアです。

1. **トレーニングデータ:**
    
    - `prompts_train.csv`: 生徒が要約を作成する元となった課題テキスト（プロンプト）の情報。`prompt_id`、`prompt_title`（タイトル）、`prompt_text`（テキスト本文）などの列を含みます。
    - `summaries_train.csv`: 生徒が作成した要約文と、それに対する人間の評価者によるスコア。`student_id`（生徒の識別子）、`prompt_id`（対応する課題テキストへのリンク）、`text`（生徒の要約文本文）、そして**ターゲット変数**となる2つの評価スコア `content`（内容の質に関する数値スコア）と `wording`（表現の質に関する数値スコア）の列を含みます。
2. **テストデータ:**
    
    - `prompts_test.csv`: テスト用の課題テキスト。
    - `summaries_test.csv`: テスト用の生徒の要約文。`student_id`, `prompt_id`, `text` を含みます。
    - テストデータには、ターゲット変数である `content` および `wording` スコアは含まれません。
3. **`sample_submission.csv`**:
    
    - 提出フォーマットのサンプル。通常、`student_id`、`content`、`wording` の列を持ちます。参加者は、テストデータに含まれる各要約文に対して、`content` と `wording` のスコアを予測し、この形式で提出します。

**評価指標 (Evaluation Metric)**

- **指標:** **Mean Columnwise Root Mean Squared Error (MCRMSE)** （列ごとの平均二乗平方根誤差）
- **計算方法:**
    1. 2つのターゲット変数（`content` と `wording`）の**各列について個別に**、予測値と真の値の間の二乗平方根誤差（RMSE）を計算します。
        - RMSE = √[ Σ(予測値 - 真の値)² / データ数 ]
    2. 各列で計算されたRMSEの**平均値**を最終的なスコアとします。
        - MCRMSE = (RMSE_content + RMSE_wording) / 2
- **意味:** RMSEは回帰タスクにおける標準的な評価指標であり、予測誤差の大きさを測ります（大きな誤差ほどペナルティが大きくなります）。MCRMSEでは、内容と表現という2つの異なる評価軸それぞれに対する予測精度を個別にRMSEで評価し、その平均を取ることで、モデルが**両方の側面をバランス良く**評価できているかを測ります。スコアは**低い**ほど（0に近いほど）、モデルの予測が人間の評価者の付けたスコアに近いことを示し、性能が良いと評価されます。

要約すると、このコンペティションは、生徒が書いた要約文を、内容と表現の2つの観点から自動評価する回帰タスクです。データは課題テキスト、要約文、および人間による評価スコアで構成され、性能は予測スコアと真のスコアの誤差を示す MCRMSE（低いほど良い）によって評価されます。

---
**全体的な傾向**

上位解法では、Transformerベースの言語モデル、特にDeBERTa V3 Largeが評価モデルとして広く採用されていました。入力として、生徒の要約（summary text）だけでなく、課題文（prompt question）、プロンプトのタイトル（prompt title）、プロンプト本文（prompt text）など、利用可能なテキスト情報を組み合わせることが一般的でした。モデルの性能を引き出すために、長いシーケンス長（maxlen）の設定、特定のプーリング手法（Attention Pooling、Mean Pooling + Head Maskなど）、学習テクニック（層の凍結、差分学習率、EMA）、補助的な損失（Auxiliary Loss）の導入などが行われました。データの品質と多様性を重視するアプローチも見られ、LLM（大規模言語モデル）を用いたデータ生成や疑似ラベリング（Pseudo Labeling）が試みられました。最終的な提出は、複数のモデルや異なる設定のチェックポイントをアンサンブル（単純平均、重み付き平均など）する手法が一般的でした。LightGBMなどのGBDTモデルを特徴量ベースで学習させ、Transformerモデルとアンサンブルするアプローチも見られました。

**各解法の詳細**

**1位**

- **アプローチ:** データ品質と多様性を最重視。LLMを用いて多様なトピックのテキストとそれに対する様々な品質の要約文を生成し、Meta Pseudo Labelingでラベル付けして学習データを拡張。
- **アーキテクチャ:** DeBERTa V3 Large (4-fold)。
- **アルゴリズム:** Meta Pseudo Labeling (Googleの論文参照、3ラウンド実施)。
- **テクニック:**
    - **データ生成:** LLMに2種類のプロンプト（①トピックテキスト生成用、②異なる品質の要約生成用）を与え、データセットを拡張。
    - **入力形式:** プロンプトテキストを含むように入力形式を変更。
    - **学習戦略:** 2段階学習（Stage1: 疑似ラベルデータのみで2エポック、Stage2: 元の訓練データのみで2-3エポック）。
    - **推論:** 入力テキスト長でソートし推論時間を短縮。
    - **パイプライン:** 公開コード (tsunotsuno氏) を利用。

**2位**

- **アプローチ:** 特殊なプーリング手法（Head Mask）、プロンプト課題文のAugmentation、補助クラス（Feedback 3.0の疑似ラベル）、疑似ラベリングによる段階的な学習。
- **アーキテクチャ:** DeBERTa V3 Large/Base、OpenAssistant Reward Model DeBERTa V3 Large、DeBERTa Large。Pooling層 (Mean Pooling + LSTM Layer/Sequence Pooling, Mean Pooling + Linear)。
- **アルゴリズム:** BCE Loss (主損失)、MSE Loss (補助損失、推定)、AdamW (推定)。
- **テクニック:**
    - **入力形式:** `Think through step by step :` + prompt_question + `[SEP]` + `Pay attention to the content and wording :` + text + `[SEP]` + prompt_text。
    - **Pooling:** Head Maskを使用し、生徒の要約（text）部分のトークンのみを対象にMean Pooling。
    - **Augmentation:** LLMでプロンプト課題文（prompt_question）のバリエーションを生成し、訓練時に使用。
    - **補助クラス:** Feedback 3.0データで学習したモデルを用いて、本コンペデータに疑似ラベル（cohesionなど6クラス）を付与し、補助損失として利用（1/2ステップ毎）。
    - **疑似ラベリング:** 性能の良いモデルで疑似ラベルを作成し、元のラベルと連結して再学習。これにより、より長いmaxlenでの学習やDeBERTa-baseモデルの効果的な学習が可能に。
    - **学習:** Layer-wise Learning Rate Decay、層凍結（下位8層）、TransformerバックボーンのDropoutオフ、HeadのMulti-sample Dropout。

**3位**

- **アプローチ:** シンプルなパイプラインとハイパーパラメータ探索。データ前処理とAugmentationを試行。EMAと差分学習率が重要。
- **アーキテクチャ:** DeBERTa (V3 Largeが主)。Pooling層 (CLSトークンと要約文Mean Poolingの結合)。
- **アルゴリズム:** MSE Loss (推定)。EMA (Exponential Moving Average)。差分学習率。
- **テクニック:**
    - **データ前処理/Aug:** 類似エッセイを類似度（Levenshtein距離など）で検出しマージ。類似単語へのランダム置換（Reverse Autocorrect風）Augmentation (Fold 3で効果確認)。
    - **入力形式:** prompt + question + text。token_type_idsで各部分を区別。
    - **学習:** EMAが安定性に必須。差分学習率が有効。
    - **アンサンブル:** 複数Fold、複数チェックポイント（Foldごとに最良1-3個、合計10個）を平均化。

**4位**

- **アプローチ:** 堅牢性重視。4-FoldではなくFull Trainモデルを複数アンサンブルする戦略。多様な入力形式、Pooling、補助損失を試行。
- **アーキテクチャ:** DeBERTa V3 Large、DeBERTa V3 Large SQuAD2。Pooling層 (CLS, Attention Pooling, Mean Pooling(テキスト部分のみ))。ArcFace Head (補助損失用)。
- **アルゴリズム:** MSE Loss (主損失)、ArcFace Loss (補助損失)。
- **テクニック:**
    - **学習戦略:** 1つのFull Trainモデルに4 Fold分の情報を凝縮するイメージで、複数(7)のFull Trainモデルをアンサンブル。
    - **入力形式:** 基本形（textとpromptペア）、オリジナルプロンプト追加形式、など複数パターン。
    - **Pooling:** CLSトークン、オリジナルプロンプト部分のAttention Pooling、テキスト部分のみのMean Poolingなど複数試行。
    - **補助損失:** 訓練データを38タイプに分類し、ArcFace Lossを補助的に使用。
    - **テキスト前処理:** ピリオド・コンマ後のスペース挿入。
    - **推論:** 学習時より長いmaxlenを設定することで性能向上（一部モデル）。

**5位**

- **アプローチ:** DeBERTa V3 LargeとLightGBMのシンプルなアンサンブル。全テキスト情報の入力と長いmaxlen設定。
- **アーキテクチャ:** DeBERTa V3 Large。LightGBM。
- **アルゴリズム:** MSE Loss (推定)。Nelder-Mead (アンサンブル重み最適化)。
- **テクニック:**
    - **入力形式:** summary + question + title + prompt_text を結合して入力。
    - **DeBERTa:** max_len 1536。Embedding層と初期18層を凍結。Dropoutなし。Content/Wording同時予測。
    - **LightGBM:** 公開ノートブックベースの特徴量を使用。
    - **アンサンブル:** DeBERTaとLGBMの予測値を重み付き平均。重みはNelder-Mead法で最適化。
    - **データ拡張:** LLMによる要約生成と疑似ラベル付与を試したが効果なし。

**7位**

- **アプローチ:** DeBERTaモデルをベースに、異なる入力形式や損失関数を持つモデルを学習しブレンディング。LightGBMの結果も加える。CVスコアを重視。
- **アーキテクチャ:** DeBERTa V3 Large、DeBERTa Large。LightGBM。
- **アルゴリズム:** MSE Loss、Log Loss。
- **テクニック:**
    - **入力形式:** `text + sep + prompt_text + sep + prompt_question` や `text + sep + prompt_title + sep + prompt_question + sep + prompt_text` など、複数のパターンを試行。
    - **学習:** 層凍結。異なる入力形式、損失関数（MSE, Log Loss）、maxlen (768, 1280) で複数のモデルを学習。
    - **アンサンブル:** 複数シードのDeBERTaモデル（異なる入力形式、損失、バックボーン）とLightGBMの結果をブレンディング（重み付き平均）。

**8位**

- **アプローチ:** DeBERTa V3 Largeモデルを複数学習しアンサンブル。入力形式にカスタム特殊トークンを使用。100%データでの再学習モデルを推論に使用。
- **アーキテクチャ:** DeBERTa V3 Large (6モデル)。デフォルトのContextPooler。
- **アルゴリズム:** MCRMSE Loss。
- **テクニック:**
    - **入力形式:** `<question> ... <summary> ... <text> ...` の形式。`<question>`, `<summary>`, `<text>` はカスタム特殊トークン。prompt_textは最後に配置（切り捨て考慮）。
    - **学習:** max_len 1024（訓練）、1536（推論）。勾配蓄積(16ステップ)。Dropout調整（FC層のみオフ）。
    - **推論/アンサンブル:** 推論時間削減のため、Foldモデルではなく100%データで再学習したモデルを使用。上位3実験のモデルを各2シード、計6モデルでアンサンブル（単純平均）。

**9位**

- **アプローチ:** 2つのDeBERTa V3 LargeとLSTMを組み合わせたカスタムモデル。特殊なCV戦略。
- **アーキテクチャ:** DeBERTa V3 Large x2、LSTM (双方向)。Mean Pooling。
- **アルゴリズム:** SmoothL1 Loss、AdamW、Linear Warmup Scheduler、EMA。
- **テクニック:**
    - **入力形式:** text1=summary_text, text2=prompt_question + [SEP] + prompt_text。tokenizerにペアで入力。
    - **モデル構造:** 1つ目のDeBERTaは全体を入力。2つ目のDeBERTa (6層) とLSTMは要約テキスト部分のみを入力。LSTMの出力をMean Poolingし、最終Linear層へ。
    - **CV戦略:** prompt_id 814d6bの挙動が他と異なるため、評価から除外/含める2パターンでCVを実施。最終提出は後者ベース。複数シード(3)のアンサンブルで評価・提出。
    - **学習:** max_len 1680（訓練）、4200（推論）。EMA使用。

