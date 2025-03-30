---
tags:
  - Kaggle
  - LLM
  - NLP
startdate: 2024-05-03
enddate: 2024-08-13
---
# LMSYS - Chatbot Arena Human Preference Predictions
https://www.kaggle.com/competitions/lmsys-chatbot-arena

**全体的な傾向**

- **大規模言語モデル（LLM）の活用:**  
  多くの解法では、事前学習済みの大規模モデル（例：llama3、qwen2、gemmaシリーズ、RLHFlowのモデルなど）をベースに、ファインチューニングやロース（LoRA/QLoRA）による軽量化・高速化を行っています。

- **データ拡張と擬似ラベリング:**  
  オリジナルの競技データに加え、33kデータや1Mクラスの追加データ、さらにはpseudo label（擬似ラベル）を用いることで、学習データを増強しモデルの汎化性能を向上させています。

- **アンサンブルとTTA（Test Time Augmentation）:**  
  5-fold CVや複数モデルの重み平均、さらに入力のswap（response A/Bの入れ替え）などのテクニックで最終的な予測精度の向上を狙っています。

- **効率的な学習・推論:**  
  高速なトレーニングのためにflash-attn、カスタムコラレータ、GPU並列処理、8bit/INT8量子化など、ハードウェアリソースを最大限に活用する工夫がなされています。

---

**1位：Distill is all you need🔥🔥🔥**

- **アプローチ:**  
  大規模モデルから得られたlogits分布を利用し、より小規模な9Bモデルに蒸留（distillation）することで、推論制約をクリアしつつ高精度な予測を実現。

- **アーキテクチャ:**  
  - 基盤モデルとして、llama3 70b、qwen2 72b、gemma2-9bを採用。  
  - モデルはAutoModelForSequenceClassificationを基本とし、LoRA（r=64, alpha=128）やQLoRAで軽量化。

- **アルゴリズム:**  
  - まず、UTデータセットで1エポックの学習を実施。  
  - その後、5-fold CVでlogits分布を取得し、これを用いて9Bモデルへの蒸留学習を行う。

- **テクニック:**  
  - 複数のモデルをアンサンブル（LoRA層の平均）し、最終的にGPTQによる8bit量子化とTTA（長さ2000の入力）を適用して推論精度を向上。

---

**2位：2nd Place Solution**

- **アプローチ:**  
  - Prompt単位でStratifiedGroupKFoldを用いたクロスバリデーションと、追加データ（deduplicated 33kや21k）を活用。  
  - さらに、responseの順番を入れ替える「full swap」をデータ拡張として利用し、ラベルノイズを低減。

- **アーキテクチャ:**  
  - カスタムヘッドを備えたシーケンス分類モデル（gemma-2-9b、RLHFlow/ArmoRM-Llama3-8B-v0.1など）を採用。  
  - 初期段階では全パラメータを用いたフルチューニングを実施。

- **アルゴリズム:**  
  - 学習時はresponse swapを原データとそのswap版の両方を用いて勾配を累積。  
  - さらに、段階的にpseudo labelingを実施し、最終的に複数モデルの平均アンサンブルで予測。

- **テクニック:**  
  - Full Swapによるデータ拡張、複数ステージ（Stage1～3）のファインチューニング、flash-attnやカスタムcollatorによる高速化、4台のA100 GPUを用いた大規模学習。

---

**3位：Mark’s Writeup & Raja’s Writeup**

- **アプローチ:**  
  - Rewardモデルからスタートし、追加のpseudo labeling（500k件規模）で学習データを大幅に拡張。  
  - 2つのバックボーンモデル（RLHFlow/pair-preference-model-LLaMA3-8B、sfairXC/FsfairX-Gemma2-RM-v0.1）のアンサンブルを実施。

- **アーキテクチャ:**  
  - 基盤にはRLHFlowやGemma2系のモデルを使用。  
  - QLoRAを用いて、全線形層に対して低ランク適応を実施。

- **アルゴリズム:**  
  - カスタムのpromptフォーマットで学習を行い、TTA（response A/Bの入れ替え）で最終的な精度を向上。  
  - 擬似ラベルを利用した多段階学習により、ラベルデータの拡充とノイズ低減を狙う。

- **テクニック:**  
  - vLLMを活用した高速擬似ラベリング、TTAによる精度ブースト、pseudo labelingのラウンドを複数回実施して最適な重みを獲得。

---

**4位：4th Place Solution**

- **アプローチ:**  
  - 独自設計のプロンプトにより、会話が長くなった場合でも各ラウンドの情報（prompt、response A、response B）が一定割合で残るよう工夫。  
  - 2段階の学習（Stage1で公式データ＋33kデータ、Stage2でultrafeedbackから擬似ラベル生成）を実施。

- **アーキテクチャ:**  
  - gemma-2-9b-itをベースとし、LoRAを用いたシーケンス分類モデルを採用。  
  - トークン列のカスタムトークナイズ（トークン数に応じた切り捨てと「......」の挿入）を実装。

- **アルゴリズム:**  
  - 各会話ラウンドごとに動的にトークン数を調整し、max_lengthを超えた場合はトークンを適切に省略。  
  - 学習と推論時でTTA（response swap）を適用し、さらにpost-processingで空欄や同一レスポンスに対する補正を実施。

- **テクニック:**  
  - カスタムコラテータによる効率的なシーケンス結合、トークン数に応じた動的な切り捨て、TTAとpost-processingによる予測値の補正。

---

**5位：Team Danube**

- **アプローチ:**  
  - 公式データと33kデータのみを使用し、外部のpseudo labelingや1Mデータは使用せず、Rewardデータで事前学習（pre-training）した後、fine-tuningを実施。

- **アーキテクチャ:**  
  - gemma-2-9bモデルをベースとし、H2O LLM Studioを用いて構築。  
  - 事前学習済みのRewardデータで得たチェックポイントを初期値として採用。

- **アルゴリズム:**  
  - Binary win/lossのラベルを用いた事前学習の後、同一設定で2回のfine-tuningを実施し、swapによるTTAを組み合わせたアンサンブルで最終予測。

- **テクニック:**  
  - Rewardデータでの事前学習によるスコア向上、INT8量子化とGPU並列化による高速推論、TTA（response順序のswap）を利用した多様性の向上。

---

**9位：9th Place Solution**

- **アプローチ:**  
  - gemma2-9b-itモデルを中心に、チャット1Mの擬似ラベルデータを活用し、llama3-8bモデルとのアンサンブルで最終予測。  
  - 事前に不要な行を削除し、ユニークなpromptごとにランダムに2つのレスポンスをサンプルすることでデータ量を調整。

- **アーキテクチャ:**  
  - gemma2-9b-itとllama3-8bを用い、各モデルに対してLoRA（gemma2はr=32、llama3はr=128など）を適用。  
  - トークナイザーは左側切り捨て（truncation_side="left"）を採用し、学習時と推論時でmax_lenを調整。

- **アルゴリズム:**  
  - シンプルなprompt構造（context, response A, response Bの連結）で学習を行い、複数モデルの予測値をアンサンブル。  
  - 擬似ラベル生成や、バッチサイズをトークン長に応じて動的に変更する工夫が施される。

- **テクニック:**  
  - チャット1Mの擬似ラベルデータによるデータ拡充、8bit量子化での高速化、動的バッチサイズ調整による計算効率の最適化。


