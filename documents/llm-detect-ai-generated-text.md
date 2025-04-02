---
tags:
  - Kaggle
  - LLM
  - DeBERTa
  - AUC
startdate: 2023-11-01
enddate: 2024-01-23
---
# LLM - Detect AI Generated Text
[https://www.kaggle.com/competitions/llm-detect-ai-generated-text](https://www.kaggle.com/competitions/llm-detect-ai-generated-text)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、与えられたエッセイテキストが、**人間によって書かれたものか、大規模言語モデル（LLM）によって生成されたものかを判定する**機械学習モデルを開発することです。
* **背景:** 近年、LLMの性能は飛躍的に向上し、人間が書いた文章と見分けがつかないほど自然なテキストを生成できるようになりました。これは様々な応用を可能にする一方で、教育現場でのエッセイ課題における不正利用や、フェイクニュース・プロパガンダの拡散など、悪用の懸念も高まっています。そのため、AIによって生成されたテキストを正確に検出する技術が求められています。
* **課題:** 多様なLLM（GPT-3.5/4, Claude, PaLM, Llama, Mistralなど、商用・オープンソース問わず）によって生成されたテキストを識別する必要があります。さらに、AI生成テキストであることを隠すための**難読化（typoの挿入、言い換え、文字の置換など）**が行われている可能性も考慮しなければなりません。コンペティションで提供される学習データは限られており、特にAI生成テキストのサンプルが少ないため、参加者自身が**多様なAI生成テキストを含む大規模な学習データセットを構築・収集する**ことが成功の鍵となります。未知のLLMや将来登場するLLMに対しても頑健な検出モデルを構築することが最終的な目標です。

**データセットの形式 (Dataset Format)**

提供されるデータは主にエッセイテキストとそのラベル（人間作成かAI生成か）ですが、AI生成テキストの学習データが不足している点が特徴です。

1.  **トレーニングデータ:**
    * `train_essays.csv`: トレーニング用のエッセイデータ。
        * `id`: エッセイの一意なID。
        * `prompt_id`: エッセイがどの課題（プロンプト）に基づいて書かれたかを示すID。
        * `text`: エッセイの全文。
        * `generated`: **ターゲット変数**。1ならLLM生成、0なら人間が作成したことを示す。 **注意:** 提供されるデータでは `generated = 1` のサンプルが極端に少ない（または存在しない可能性もある）。
    * `train_prompts.csv`: トレーニング用エッセイの課題（プロンプト）情報。
        * `prompt_id`: プロンプトID。
        * `prompt_name`: プロンプトのタイトル。
        * `instructions`: プロンプトの指示内容。
        * `source_text`: プロンプトに関連する参照テキスト（HTML形式の場合あり）。
    * **外部データ/自己生成データ（重要）:** 上記の公式データだけでは不十分なため、多くの参加者が以下のようなデータを活用・生成しています。
        * 公開データセット: DAIGT V2/V4, SlimPajama, The Pile, Persuade Corpus 2.0 など。
        * LLMによる生成: GPT-3.5/4, Claude, Gemini, PaLM, Llama, Mistral, Falcon など多様なLLMを用いて、様々なプロンプト、パラメータ設定（temperature, top-k, top-pなど）、生成手法（単純生成、書き換え、穴埋めなど）でAIエッセイを大量に生成。
2.  **テストデータ:**
    * `test_essays.csv`: テスト用のエッセイデータ。`id`, `prompt_id`, `text` カラムを持つ。`generated` カラムは含まれない。
3.  **`sample_submission.csv`**:
    * 提出フォーマットのサンプル。`id`（エッセイID）と `generated`（エッセイがAIによって生成された確率、0から1の値）の列を持ちます。

**評価指標 (Evaluation Metric)**

* **指標:** **AUC (Area Under the ROC Curve; ROC曲線下面積)**
* **計算方法:** ROC曲線は、分類器の閾値を様々に変化させたときの真陽性率（True Positive Rate, Recall）を縦軸に、偽陽性率（False Positive Rate）を横軸にプロットしたグラフです。AUCは、このROC曲線の下側の面積を表します。
* **意味:** モデルがAI生成テキスト（陽性）と人間が書いたテキスト（陰性）をどれだけうまく区別できるかを総合的に評価する指標です。値は0から1の範囲を取り、1に近いほどモデルの識別性能が高いことを示します。ランダムな予測ではAUCは約0.5になります。AUCは特定の閾値に依存しないため、モデルの全体的なランキング能力や識別能力を評価するのに適しています。

要約すると、このコンペティションは、与えられたエッセイテキストがAI生成か人間作成かを判定する二値分類タスクです。学習データが不足しているためデータ構築が鍵となり、性能はAUC（高いほど良い）によって評価されます。

---

**全体的な傾向**

このコンペティションの成否を分けた最大の要因は、**大規模で多様な学習データセット（データミックス）の構築**でした。コンペ主催者から提供されるAI生成テキストが非常に限られていたため、参加者は様々なLLM（商用、OSS、ファインチューンモデル）と多様な生成手法（異なるプロンプト、パラメータ、難読化技術）を用いて、数十万から百万を超える規模のAI生成テキストと人間によるテキストのデータセットを作成しました。

モデルとしては、**DeBERTa-v3-large** が非常に強力であり、多くのトップチームで使用されました。Mistral-7BなどのLLM自体を分類タスク用にファインチューニングするアプローチも有効でした。一方で、伝統的な**TF-IDF + 分類器（LightGBM, CatBoost, ロジスティック回帰, SVMなど）** のパイプラインも、特に公開リーダーボード（Public LB）では高い性能を示しましたが、最終的な評価（Private LB）ではTransformerベースのモデルに劣る傾向が見られました。ただし、これらをアンサンブルに加えることは依然として有効でした。

**アンサンブル**はスコア向上のために広く用いられ、異なるモデルアーキテクチャ（DeBERTa, Mamba, RoBERTa, TF-IDF系など）や、異なる学習データ・戦略で学習されたモデルの予測を組み合わせることで、単一モデルよりも頑健で高性能な予測を目指しました。予測値の単純平均、加重平均、ランキングベースの平均化などが採用されました。

難読化（obfuscation）や未知のLLMへの対応として、**データ拡張**（typo挿入、文字置換、スペル修正など）や、**ドメイン適応**（テストデータを用いた疑似ラベリングや短期間のファインチューニング）、**後処理**（クラスタリングやルールベースの調整）なども重要なテクニックとなりました。信頼できる交差検証（CV）の構築は困難でしたが、多様なデータとモデルを組み合わせることで汎化性能を高める試みがなされました。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/llm-detect-ai-generated-text/discussion/473295)**

* **アプローチ:** **データミックス構築**に最注力 + 多様なモデル戦略 + ランキングベースアンサンブル。
* **アーキテクチャ/アルゴリズム:** Mistral-7B ((Q)LoRA Finetuning), DeBERTa-v3-large (分類/ランキング), DeBERTa-v3-small (カスタムTokenizer+MLM+疑似ラベル), Ghostbuster (Llama 7B / TinyLlama 1.1B のトークン確率特徴量 + SVM/RandomForest), Ahmet's Unsupervised Approachの変種。
* **テクニック:**
    * **データミックス:** Persuadeコーパス全体、多様な外部人間テキスト、多種多様なLLM（商用/OSS/ファインチューン）による生成、多様な生成設定（Contrastive Search, 高temperature, fill-in-the-blankなど）、既存データセット（T5生成, DAIGT V2, OUTFOX, Ghostbuster dataなど）を活用。データ拡張（スペル修正、文字操作、同義語置換、難読化、逆翻訳など）。テキストの正規化（特殊文字除去、小文字化）も試行。最終的に160kサンプル（うち人間40k）で学習。
    * **アンサンブル:** 各モデルの予測値を**ランキングに変換**し、そのランキングを（重み付き）平均して最終予測とする。Mistralモデル群に最も高い重み付け。

**[2位](https://www.kaggle.com/competitions/llm-detect-ai-generated-text/discussion/470395)**

* **アプローチ:** 大規模データでの**事前学習** + Persuadeドメインへの**適応ファインチューニング**。
* **アーキテクチャ/アルゴリズム:** DeBERTa-v3-large, DeBERTa-large。アンサンブル。
* **テクニック:**
    * **事前学習:** SlimPajamaデータセットから約50万の人間/AIテキストペアを生成し、DeBERTa-v3-largeをファインチューニング（汎用分類器化）。
    * **ドメイン適応:**
        1.  PersuadeコーパスでLLMをLMファインチューニング（h2o-llmstudio使用）し、学生の文体を模倣したAIエッセイを生成。
        2.  事前学習済みDeBERTaを、このドメイン特化AIエッセイとDAIGT-V4データでさらにファインチューニング。
    * **アンサンブル:** DeBERTa-v3-large x2 と DeBERTa-large x1 を組み合わせ。DAIGT-V4で追加学習したモデルもアンサンブルに追加。

**[3位](https://www.kaggle.com/competitions/llm-detect-ai-generated-text/discussion/470333)**

* **アプローチ:** **TF-IDFパイプライン** + **DeBERTa-v3-largeアンサンブル (12モデル)** の段階的加重平均。
* **アーキテクチャ/アルゴリズム:** TF-IDF + VotingClassifier (CatBoost, LightGBMなど), DeBERTa-v3-large。
* **テクニック:**
    * **データセット:**
        * DeBERTa用1: 外部データ＋カスタム生成/リフレーズデータから反復的に選択した11kサンプル。
        * DeBERTa用2: Pile/SlimPajamaの続きを多様なOSSモデル(約35種)とパラメータで生成した大規模データ(500k〜1.2M)。
        * TF-IDF用: DAIGT-V2 + テストセットからの高信頼度疑似ラベル(1k)。
    * **前処理:** 限定的な難読化解除（エラー15個以上のみ）、文字正規化。
    * **アンサンブル:** 段階的加重平均。まずTF-IDFと11kデータモデルを高信頼度領域で重み付け。次にその結果と大規模データモデルを重み付け平均。
    * **後処理:** プロンプト毎にTF-IDF+UMAPでクラスタリングし、最近傍の人間/AIサンプルとの距離比で予測値をスケーリング。

**[4位](https://www.kaggle.com/competitions/llm-detect-ai-generated-text/discussion/471934)**

* **アプローチ:** 複数の異なるアプローチ（古典ML, LLM特徴量, Transformer）を組み合わせる**「Combined Arms」戦略**。
* **アーキテクチャ/アルゴリズム:** TF-IDF + 線形/GBTモデル, Mistral-7B + 線形/GBTモデル, Longformer, DeBERTa-v3-large。重み付きアンサンブル。
* **テクニック:**
    * **古典ML:** 公開TF-IDFノートブックを改良（ngram範囲拡大、特徴量制限、前処理追加）。**人工的なtypo**をランダムに追加。
    * **LLM特徴量:** Mistral-7Bからトークン確率、logprob特徴量、perplexityなどを抽出し、特徴量空間を作成。TF-IDFモデルでテストセットに**疑似ラベル**を付け、そのラベルを用いてMistral特徴量から線形/GBTモデルを学習。
    * **Transformer-1 (Longformer):** テキストの**データソース**（どのデータセット由来か）を予測するタスクで学習し、その予測値を特徴量として利用。
    * **Transformer-2 (DeBERTa):** 大規模公開データ(約700k)で学習。正則化およびTF-IDFが不確かな場合の補完目的。
    * **アンサンブル:** 各モデルの予測値を重み付き平均。重みはLBスコアではなく、バランスを考慮して手動設定。

**[5位](https://www.kaggle.com/competitions/llm-detect-ai-generated-text/discussion/470093)**

* **アプローチ:** **超大規模データセット(1.7M+)**での学習 + **ドメイン適応 (Teacher-Student)**。
* **アーキテクチャ/アルゴリズム:** DeBERTa-v3-large, **Mamba-790m**。Teacher-Studentフレームワーク。
* **テクニック:**
    * **データセット:** PERSUADE, Uncopyrighted Pile completions, SlimPajama completions, Tricky Crawl (人間が書いたのにAIと誤分類されやすいテキスト) の混合。**Pileデータ重視** (62-99%)。vLLMで効率的に生成。多様なサンプリングパラメータ使用。
    * **ドメイン適応:**
        1.  **Teacher:** DeBERTaとMamba (長コンテキスト1024) でテストデータにソフトラベル付け (DeBERTa 90%, Mamba 10%)。
        2.  **Student:** 事前学習済みの短コンテキストDeBERTa (128/256文字) を、Teacherの予測を模倣するようにテストデータチャンクでファインチューニング。
        3.  最終予測はStudentモデルのチャンク予測の加重平均。
    * **データ拡張:** バグのあるスペルチェック(jamspell使用)、禁止文字除去、typo追加（typoライブラリ、大文字小文字反転）。
    * **学習:** Dropout無効化。線形ウォームアップ+線形減衰LRスケジュール。
    * **備考:** Mambaの使用はプライベートLBスコアを低下させた。DeBERTa単体や、Mambaなしのドメイン適応の方が良い結果だった。

**[6位](https://www.kaggle.com/competitions/llm-detect-ai-generated-text/discussion/471831)**

* **アプローチ:** **エントロピーベース特徴量** + **One-Class SVM**。
* **アーキテクチャ/アルゴリズム:** 事前学習済みLLM (**Phi-2**が最良), One-Class SVM。
* **テクニック:**
    * **特徴量:** Phi-2を用いてテキストのエントロピーベース特徴量を計算。
    * **学習:** **人間が書いたエッセイのみ**（主催者提供データ）を学習データとして使用し、One-Class SVMを訓練。
    * **特徴量選択:** DAIGT-V4データセットを用いて最適な特徴量を選択。

**[7位](https://www.kaggle.com/competitions/llm-detect-ai-generated-text/discussion/470643)**

* **アプローチ:** **非Instruction-Tunedモデルのみ**でデータ生成 + DeBERTa学習 + TF-IDFブレンド。
* **アーキテクチャ/アルゴリズム:** DeBERTa-v3-large, TF-IDF + SGD。
* **テクニック:**
    * **データ生成:** **Instruction-Tunedでない**LLM (Falcon-7B, Mistral-7B, Llama2-7B) を使用。SlimPajama(400k) + Persuade(25k)。多様な温度/top-p/頻度ペナルティ。カスタムプロンプトエンジニアリング。類似度/長さ/繰り返しでフィルタリング。
    * **学習:** DeBERTa-v3-largeを**512トークン長で学習**、1024トークン長で推論（1024学習はスコア低下）。
    * **アンサンブル/後処理:** DeBERTaの予測確率が**40-60パーセンタイルの範囲にある場合、TF-IDF+SGDの予測値で置き換え**。

**[8位](https://www.kaggle.com/competitions/llm-detect-ai-generated-text/discussion/470224)**

* **アプローチ:** **言語的特徴量 (PPL & GLTR)** を利用 + VotingClassifier。
* **アーキテクチャ/アルゴリズム:** GPT-2 (small/medium/large), VotingClassifier (SVM, RandomForestなど)。
* **テクニック:**
    * **特徴量:**
        * **Perplexity (PPL):** GPT-2を用いてテキスト全体および文レベルのPPLを計算。AI生成テキストはPPLが低い傾向。
        * **GLTR (Test-2):** GPT-2の予測確率分布におけるTop-10/100/1000/1000+ランクのトークン数を特徴量として使用。AI生成テキストは高確率トークン（低ランク）を多用する傾向。
    * **学習:** 上記特徴量を組み合わせ、VotingClassifierを学習。大規模データセット(800k)の使用がスコア向上に不可欠だった。
    * **その他:** 高速推論（約10分）。

**[9位](https://www.kaggle.com/competitions/llm-detect-ai-generated-text/discussion/470255)**

* **アプローチ:** TF-IDFパイプライン + BERTパイプライン のアンサンブル。
* **アーキテクチャ/アルゴリズム:** TF-IDF + 複数分類器, DeBERTa-v3-large x2, RoBERTa x1。
* **テクニック:**
    * **データセット:** 多様なプロンプトとモデル（7B-70B, OSS/商用）で生成した200k-700k規模のデータセットを複数バリアント作成。エッセイ生成、リフレーズ、テキスト補完プロンプト。人間テキストは学生エッセイとウェブテキスト。
    * **TF-IDF改良:** **統計的/リバースエンジニアリングによる難読化解除**パイプラインを適用。
    * **後処理:** **クラスタリング**に基づき、TF-IDF特徴量の類似度が高いテキストペアは両方ともLLM生成と判断する調整。
    * **アンサンブル:** TF-IDF系とBERT系の予測を組み合わせる。
