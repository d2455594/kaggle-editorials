---
tags:
  - Kaggle
  - LLM
  - 敵対的攻撃
startdate: 2024-02-28
enddate: 2024-04-17
---
# LLM Prompt Recovery
[https://www.kaggle.com/competitions/llm-prompt-recovery](https://www.kaggle.com/competitions/llm-prompt-recovery)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、大規模言語モデル（LLM、このコンペではGemma）によって**書き換えられたテキスト (`rewritten_text`)** とその**元テキスト (`original_text`)** のペアが与えられたときに、その書き換え指示に使われた**プロンプト (`rewrite_prompt`) を可能な限り正確に復元・予測する**ことです。
* **背景:** LLMはプロンプトに基づいてテキストを生成・編集しますが、そのプロセスはブラックボックス的な側面も持ちます。どのようなプロンプトが特定の出力を生成したのかを理解（あるいは推定）することは、LLMの挙動分析、コンテンツ生成の再現性確保、モデルの安全性やアライメントの研究において重要です。
* **課題:** 同じ書き換え結果を生成するプロンプトは一つとは限らず、多様な表現があり得ます。評価指標である Mean Sharpened Cosine Similarity は、`sentence-t5` という特定の埋め込みモデルにおける類似度を測るため、意味的に近くても単語が少し違うだけでスコアが大きく変動する可能性があります。また、このコンペティションでは**学習データが提供されず**、参加者は提供された補助データや外部リソースを用いて、自身で学習データや検証戦略を構築する必要がありました。さらに、評価に使用される `sentence-t5` モデルのTensorFlow実装とHugging Face実装の間で、トークナイザ（特に特殊トークン `</s>` の扱い）に差異があり、これがスコアリングに影響を与えるという技術的な罠が存在しました。

**データセットの形式 (Dataset Format)**

このコンペティションでは、参加者が自身で学習データを生成する必要がある点が最大の特徴です。

1.  **学習データ (Training Data):**
    * **提供されません。**
    * 参加者は、コンペで提供される補助データ (`gemma_7b_rewrites.csv`, `gemma_rewrite_prompts.csv` など) や、公開されているデータセット、あるいは自身で用意したテキストとプロンプトを用いて、Gemmaや他のLLM (GPT, Mistral, Llamaなど) を実行し、`(original_text, rewrite_prompt, rewritten_text)` の形式の学習データを生成する必要がありました。
2.  **テストデータ (Test Data):**
    * `test.csv`: 評価用のデータ。
        * `id`: サンプルの一意なID。
        * `original_text`: 書き換え前の元テキスト。
        * `rewritten_text`: Gemmaによって書き換えられた後のテキスト。
    * この `original_text` と `rewritten_text` のペアに対して `rewrite_prompt` を予測します。
3.  **補助データ (Supplemental Data):**
    * Gemmaモデルによる書き換え例やプロンプト例を含むCSVファイルがいくつか提供され、データ生成や分析の参考に利用されました。
4.  **`sample_submission.csv`**:
    * 提出フォーマットのサンプル。`id` と `rewrite_prompt`（予測したプロンプト文字列）の列を持ちます。

**評価指標 (Evaluation Metric)**

* **指標:** **Mean Sharpened Cosine Similarity (平均強調コサイン類似度)**
* **計算方法:**
    1.  予測プロンプト (`prediction`) と正解プロンプト (`ground_truth`) を、指定された `sentence-t5-base` モデルを用いてエンコードし、768次元の埋め込みベクトルを取得します。
    2.  得られた2つのベクトル間のコサイン類似度 ( $$ \text{sim}(v_{pred}, v_{true}) $$ ) を計算します。
    3.  このコサイン類似度を3乗します ( $$ \text{sim}(v_{pred}, v_{true})^3 $$ )。これを「Sharpened Cosine Similarity (SCS)」と呼びます。類似度が高い（1に近い）ほど値は1に近づき、低いほど0に近づく度合いが強調されます。
    4.  テストセット内の全サンプルについてSCSを計算し、その**平均値**を最終スコアとします。
* **意味:** 予測されたプロンプトが、正解のプロンプトと `sentence-t5` の意味空間上でどれだけ近いかを評価します。類似度を3乗することで、非常に類似した（コサイン類似度が1に近い）予測を高く評価し、少しでも離れていると評価が急激に下がる特性を持ちます。スコアは最大1.0で、**高い**ほど良い予測とされます。

要約すると、このコンペティションは、テキスト書き換えのビフォーアフターから元の指示プロンプトを復元するタスクです。学習データが提供されずデータ生成が鍵となり、性能は`sentence-t5`埋め込みを用いた特殊な類似度指標（Mean Sharpened Cosine Similarity、高いほど良い）によって評価されます。

---

**全体的な傾向**

このコンペティションは、学習データが提供されず、特殊な評価指標が用いられたため、ユニークな解法が多く見られました。

1.  **Mean Prompt / 定数予測の優位性:** 多くのチームが、全テストサンプルに対して同じ、あるいは数種類の**最適化された「平均的」なプロンプト（Mean Prompt）**を出力する戦略を採用し、これが非常に高いスコアを達成しました。これは、評価指標 (SCS) が個々のプロンプトの完全な再現よりも、`sentence-t5` 埋め込み空間での平均的な近さを重視するためと考えられます。Mean Promptの最適化には、ブルートフォース、ビームサーチ、ランダムトークン変更などの探索的手法が用いられました。
2.  **`lucrarea` トークンによる"攻撃":** 評価に使われた `sentence-t5` のTensorFlow版トークナイザの挙動（特殊トークン `</s>` をリテラル文字列として扱う）と、ルーマニア語のトークン `lucrarea` が `</s>` と埋め込み空間上で非常に近いという偶然を利用し、予測プロンプトの末尾に `lucrarea` を追加することでスコアを大幅に向上させる手法が発見され、多くのトップチームが利用しました。これは一種のAdversarial Attackと見なせます。
3.  **LLMによる個別予測の試み:** Mistral-7B, Gemma, H2O-Danube などのLLMをファインチューニングし、個々のサンプルに対してプロンプト（全文、差分、タグなど）を予測するアプローチも取られました。しかし、LLM単体ではMean Prompt戦略を超えるスコアを出すのは困難でした。データ生成には、補助データや外部LLM（Gemini, GPT-3.5など）が活用されました。
4.  **ハイブリッド戦略:** 高スコアを達成したチームの多くは、**Mean Prompt戦略とLLMによる個別予測を組み合わせる**ハイブリッドアプローチを採用しました。例えば、Mean PromptにLLMの予測結果（のユニークな単語）を追加する、LLM予測の信頼度に応じてMean Promptと切り替える（Gateモデル）、埋め込み空間上でMean PromptベクトルとLLM予測ベクトルの中間点を探索しデコードする、などの手法が見られました。
5.  **埋め込み空間での操作:** `sentence-t5` の埋め込みベクトルを直接操作する試みもありました。LLMに埋め込みベクトル自体を予測させたり、目標ベクトル（平均ベクトルやLLM予測ベクトル）に近づくように文字列を最適化したりする手法が用いられました。
6.  **Tokenizerの差異への対応:** 評価に使われるTensorFlow版 `sentence-t5` と開発で使われるHugging Face版 `sentence-t5` のトークナイザ（特に `</s>` の扱い）の違いを認識し、検証や最適化プロセスで考慮することが重要でした。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/llm-prompt-recovery/discussion/494343)**

* **アプローチ:** **Adversarial Attack (`lucrarea`の利用)** + LLM予測。
* **アーキテクチャ/アルゴリズム:** Mistral 7b (Instruct & Base), Gemma-7b-1.1-it。
* **テクニック:**
    * **`lucrarea` Attack:** LLMによる予測プロンプトの末尾に特定の文字列 `" 'it 's ' something Think A Human Plucrarealucrarealucrarealucrarealucrarealucrarealucrarealucrarea"` を追加することで、評価指標のバグ/特性を利用しスコアを大幅に向上 (+0.05)。
    * **LLM予測:** 複数のLLMを使用。予測の多様性を確保するため、各モデルに異なる開始動詞を持つプロンプトを与え、それらの予測を連結して最終的なLLM予測部分を生成。

**[2位](https://www.kaggle.com/competitions/llm-prompt-recovery/discussion/494497)**

* **アプローチ:** Mean Prompt最適化 + 埋め込み予測モデル + LLM予測 + 埋め込み空間最適化文字列 のハイブリッド。
* **アーキテクチャ/アルゴリズム:** H2O-Danube2 / Mistral 7b (埋め込み予測用), Gemma (データ生成用)。 ブルートフォース/貪欲探索 (文字列最適化)。
* **テクニック:**
    * **Mean Prompt最適化:** TF版Tokenizer問題を考慮し、特殊トークンを除外してブルートフォース最適化 (`lucrarea`が有効)。
    * **埋め込み予測:** H2O-Danube2/Mistral 7b を Cosine Similarity Loss で学習し、ターゲットプロンプトのT5埋め込みベクトルを直接予測。
    * **LLM予測:** プロンプトの差分（例: "as a shanty"）を予測。Few-shot予測も追加。
    * **埋め込み空間最適化:** 予測された埋め込みベクトルに近づくように、トークン列を貪欲探索で最適化（20トークン）。
    * **最終提出:** 「Few-shot予測 + LLM差分予測 + Mean Prompt + 埋め込み最適化文字列」 を連結。

**[3位](https://www.kaggle.com/competitions/llm-prompt-recovery/discussion/494621)**

* **アプローチ:** クラスタリングによるMean Prompt選択 + LLM予測 (Full Prompt / Tags) + Gateモデル のハイブリッド。
* **アーキテクチャ/アルゴリズム:** Mistral-7B-Instruct-v0.2 (LoRA FT, Full Prompt予測/Tags予測), Mistral-7B (SequenceClassification, Gateモデル/Cluster分類), KMeans, HDBSCAN (クラスタリング), Beam Search (Mean Prompt最適化)。
* **テクニック:**
    * **Mean Prompt:** LB分布に合わせて選択したサブセットでビームサーチにより最適化。クラスタリングにより複数テンプレートを作成。
    * **LLM予測:** Full Prompt予測モデルとTags予測モデルを学習。データ生成戦略（プロンプト候補生成→バリエーション生成→元テキスト生成→Gemma書き換え生成）が重要。学習データをクラスタでバランス取り。Unslothで効率化。
    * **Gateモデル:** Full Prompt予測が明らかに間違っている場合を検出・フィルタリングするための分類モデル。ハードネガティブマイニング使用。
    * **クラスタリング:** テストサンプルを12個のクラスタに分類し、クラスタ毎に最適化されたMean Promptを選択。Cluster分類モデルとKMeansの両方が一致した場合のみ適用。
    * **最終提出:** 選択されたMean Promptをベースに、Full Prompt予測とTags予測からユニークな単語を挿入（Gateモデル通過時のみ）。

**[4位](https://www.kaggle.com/competitions/llm-prompt-recovery/discussion/494362)**

* **アプローチ:** 最適化されたMean Prompt (`lucrarea`含む) + Mistral 7B Instruct予測。
* **アーキテクチャ/アルゴリズム:** Mistral-7B-Instruct-v0.2。貪欲探索 (Mean Prompt最適化)。
* **テクニック:**
    * **Mean Prompt最適化:** TF版Tokenizer問題と `lucrarea` の有効性を認識。LBスコア分布に合うよう学習データサブセットを選択 (ランダムサンプリング+適合度計算)。トークンレベルの貪欲探索でMean Promptを構築。
    * **LLM予測:** Mistral 7B Instructを使用。シンプルな接頭辞 (`"Modify this text by"`) を付与。
    * **最終提出:** 最適化されたMean Prompt + Mistral予測 を組み合わせる（具体的な組み合わせ方法は不明瞭）。

**[5位](https://www.kaggle.com/competitions/llm-prompt-recovery/discussion/499079)**

* **アプローチ:** 埋め込み予測 + 類似プロンプト検索 + LLM候補生成 + Rerank + Suffix Attack (`lucrarea`)。
* **アーキテクチャ/アルゴリズム:** Roberta-base/large (埋め込み予測), Mistral 7b (候補生成)。コサイン類似度検索、順列組み合わせによるRerank。
* **テクニック:**
    * **埋め込み予測:** Robertaモデルを Sharpened Cosine Similarity (SCS) Loss で学習し、T5埋め込みを予測。
    * **候補生成:** 予測埋め込みとコサイン類似度が高いプロンプトをDBから検索。Mistral 7bでも候補を生成。
    * **Rerank:** 複数の候補プロンプトを連結・順列組み合わせし、予測埋め込みに最も近くなる組み合わせを選択。
    * **Suffix Attack:** TF版Tokenizer問題と `lucrarea` の有効性を発見し、Suffixとして追加。

**[6位](https://www.kaggle.com/competitions/llm-prompt-recovery/discussion/494755)**

* **アプローチ:** Mean Prompt反復最適化 + LLM予測 + 埋め込み空間補間 + 文字列デコード。
* **アーキテクチャ/アルゴリズム:** Mistral-7b, OpenChat 3.5 (LLM予測)。ランダムトークン変更による文字列デコード/最適化。
* **テクニック:**
    * **Mean Prompt最適化:** LB提出スコアからLBターゲットの平均ベクトルを推定。推定平均ベクトルに近づくように、ランダムトークン変更（削除/追加/変更）による反復最適化でMean Promptを構築。
    * **LLM予測:** MistralとOpenChatを使用。出力の定型句問題を緩和するため、複数の開始句で生成しT5埋め込みを平均化。
    * **埋め込み空間補間:** 2つのLLM予測の一致度に基づき、Mean PromptベクトルとLLM予測ベクトル（の平均）の間で目標ベクトルを決定。
    * **文字列デコード:** 決定された目標ベクトルに近づくように、`Mean Prompt + LLM予測` を初期値としてランダムトークン変更で文字列をオンラインで最適化（約200エポック）。
    * **備考:** HF版Tokenizerを使用（`lucrarea`は利用せず）。

**[7位](https://www.kaggle.com/competitions/llm-prompt-recovery/discussion/494650)**

* **アプローチ:** Mean Promptの高度な最適化 + Mistral 7B予測の追加。
* **アーキテクチャ/アルゴリズム:** Mistral 7B。Greedy Search, Beam Search, 挿入, 枝刈り (Mean Prompt最適化)。
* **テクニック:**
    * **Mean Prompt最適化:** トークンレベルで様々な探索手法（Greedy, Beam Search, 挿入, 枝刈り）を組み合わせて最適化。TF版Tokenizerの128トークン長制限も考慮。
    * **データセットバイアス低減:** 過去のLB提出結果とのスコア差を比較し、LBの分布に近い学習データサブセットを反復的に構築。
    * **LLM予測:** Mistral 7Bで `"Improve this text by."` で始まるプロンプトを予測。
    * **最終提出:** 最適化されたMean PromptにLLM予測を追加（具体的な追加方法は不明瞭だがスコア向上に寄与）。

**[10位](https://www.kaggle.com/competitions/llm-prompt-recovery/discussion/494689)**

* **アプローチ:** Mean Prompt + Mistral予測 + ルールベース予測 の組み合わせ。
* **アーキテクチャ/アルゴリズム:** Mistral 7B。Beam Search, 貪欲探索 (Mean Prompt最適化)。ルールベース。
* **テクニック:**
    * **Mean Prompt:** 最適化されたMean Prompt (`piece_1`)。`lucrarea` は使っているが、Suffixではなくプロンプト内に組み込まれている。
    * **Mistral予測:** Few-shotとCoTを用いた予測 (`piece_2`)。スコアを下げるパターンをフィルタリング。
    * **ルールベース:** 書き換え後テキスト内のキーワード（対話形式の `X:`, `Y:` や強調 `*XXX*` など）に基づいてテンプレートから予測 (`piece_3`)。
    * **最終提出:** 上記3つのピースを単純に連結。


