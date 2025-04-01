---
tags:
  - Kaggle
startdate: 2023-02-14
enddate: 2023-05-16
---
# Stable Diffusion - Image to Prompts
[https://www.kaggle.com/competitions/stable-diffusion-image-to-prompts](https://www.kaggle.com/competitions/stable-diffusion-image-to-prompts)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、**Stable Diffusion 2.0** で生成された画像を入力として、その画像を生成するために使用された**テキストプロンプトを予測する**（あるいは、それに意味的に非常に近いテキストプロンプトを生成する）モデルを開発することです。
* **背景:** 近年、Stable Diffusion、Midjourney、DALL-E 2などのテキスト-画像生成モデルが急速に進化し、広く利用されています。これらのモデルはテキストプロンプトに基づいて高品質な画像を生成できますが、逆に**画像からそれを生成したであろうプロンプトを推定する**「逆問題 (Image-to-Prompt)」も重要になってきています。これは、既存の画像のプロンプトを理解したり、類似画像を生成するためのプロンプトエンジニアリングのヒントを得たりするのに役立ちます。
* **課題:**
    * **データセットの欠如:** このタスクのための大規模な「画像-正解プロンプト」ペアのデータセットは公開されていません。参加者は**自身でトレーニングデータを生成**する必要があります。
    * **プロンプトの多様性:** Stable Diffusionのプロンプトは非常に多様で、写実的な描写から芸術的なスタイル、抽象的な概念まで様々です。単語の組み合わせや特定のキーワード（例: "trending on artstation"）が画像に大きな影響を与えます。
    * **多対多の関係:** 同じプロンプトでもノイズ（seed）によって異なる画像が生成され、逆に異なるプロンプトから似たような画像が生成される可能性があります。
    * **評価指標:** 予測されたプロンプトと正解プロンプトの**意味的な類似性**を評価する必要があり、単純な単語の一致では不十分です。
    * **計算資源:** 大量の画像生成とモデル学習には相応の計算資源が必要です。

**データセットの形式 (Dataset Format)**

このコンペティションの最大の特徴は、主催者から**トレーニングデータが提供されない**ことです。参加者は自身でトレーニングデータセットを構築する必要があります。

1.  **データ生成プロセス (参加者実施):**
    * **プロンプト収集:** 様々なソースからテキストプロンプトを収集します。一般的なソースとして以下が挙げられます。
        * **DiffusionDB:** Stable Diffusionユーザーが実際に使用したプロンプトの大規模データベース。
        * **LAION:** 大規模な画像-テキストペアデータセット。テキスト部分をプロンプトとして利用。
        * **画像キャプションデータセット:** COCO Captions, Flickr30k, Conceptual Captions, VizWizなど。これらのキャプションをプロンプトとして利用。
        * **LLM生成:** ChatGPTなどの大規模言語モデルにプロンプトを生成させる。
        * **カスタム生成器:** 特定の構造（例: [主題] [スタイル] [修飾子]）に基づいてプロンプトを自動生成する。
    * **画像生成:** 収集したプロンプトを入力として、**Stable Diffusion 2.0** を使用して画像を生成します。通常、ノイズ除去ステップ数やguidance scaleなどのパラメータを設定します。多くの参加者が、異なる`seed`を用いて同じプロンプトから複数の画像を生成し、データの多様性を高めています。
    * **データクリーニング:** 収集したプロンプトや生成した画像ペアから、低品質なものや重複を除去します（例: テキストの類似度計算、画像とテキストのCLIPスコアによるフィルタリング）。
2.  **提供されるデータ:**
    * **`test/images/`:** 評価対象となる画像ファイル（約7,000枚）。これらの画像に対応するプロンプトを予測します。
    * **`sample_submission.csv`:** 提出フォーマットのサンプル。`imgId_eId` (画像IDと埋め込みベクトルのインデックスを組み合わせたもの) と `val` (予測されたプロンプト埋め込みベクトルの各次元の値) の列を持ちます。
    * **`prompts.csv` (補足):** テスト画像生成に使われたプロンプトの一部が提供されていますが、これは主にEDA（探索的データ分析）用であり、テストセットの完全な正解ラベルではありません。

**評価指標 (Evaluation Metric)**

* **指標:** **コサイン類似度 (Cosine Similarity)**
* **計算方法:**
    1.  参加者は、各テスト画像に対して予測されるプロンプトを生成します。
    2.  予測されたプロンプトと、その画像を生成した実際の（隠された）プロンプトの両方を、指定されたSentence Transformerモデル (`sentence-transformers/all-MiniLM-L6-v2`) を用いて384次元の埋め込みベクトルに変換します。
    3.  2つのベクトル間のコサイン類似度を計算します。
        Cosine Similarity = (A · B) / (||A|| ||B||)
    4.  全てのテスト画像に対するコサイン類似度の平均値が最終スコアとなります。
* **意味:** 予測されたプロンプトと実際のプロンプトが、単語レベルではなく**意味的にどれだけ近いか**を評価します。コサイン類似度は、ベクトルの向きがどれだけ似ているかを示す指標です。スコアは **-1から1** の範囲を取り、**1に近い**ほど、予測プロンプトが正解プロンプトと意味的に類似していると評価されます。

要約すると、このコンペティションは、Stable Diffusionで生成された画像から元のプロンプトを予測するImage-to-Promptタスクです。参加者は自身で大規模な画像-プロンプトペアデータセットを生成する必要があり、性能は予測プロンプトと正解プロンプトの埋め込みベクトル間のコサイン類似度（1に近いほど良い）によって評価されます。

---

**全体的な傾向**

このコンペティションでは、画像からプロンプトを直接生成するのではなく、**画像からプロンプトの埋め込みベクトル（Sentence Transformer `all-MiniLM-L6-v2` で得られる384次元ベクトル）を直接予測する**アプローチが支配的でした。これは、評価指標が埋め込みベクトル間のコサイン類似度であることに直接対応した戦略です。

そのためのモデルとして、**強力な画像認識モデルのFine-tuning**が主流でした。特に、**CLIP (Contrastive Language–Image Pre-training)** のVision Transformer (ViT) やConvNeXtベースの画像エンコーダ、あるいはBLIP-2のVision Model、EVAシリーズなどが広く用いられました。これらのモデルは、画像とテキストの意味的な関連性を捉える事前学習がされているため、本タスクに適していました。

**データセットの構築**が勝敗を分ける重要な要素であり、多くのチームが数百万〜1千万枚規模の画像を自前で生成しました。データソースとしては、**DiffusionDB**、**LAION**、**COCO Captions**などの既存データセットのテキスト部分を利用するほか、**ChatGPT**などのLLMにプロンプトを生成させる方法も取られました。生成されたデータの**質と多様性**を高めるためのフィルタリングやクリーニングも重要でした。画像生成の高速化（xformers, fp16, 低解像度/ステップ数削減）も多くのチームが取り組んだ課題です。

モデルの学習においては、**大規模データセットでの事前学習**（例: COYO-700M）後に、**高品質なデータセットでファインチューニング**する2段階学習が有効でした。CLIPモデルのFine-tuningにおいては、元の知識を保持しつつタスクに適応させるためのテクニック（**LP-FT: Linear Probe and then Fine Tune**, **Layer-wise Learning Rate Decay**, **LORA**）が効果を発揮しました。FC層を追加するなどのアーキテクチャ変更も試されました。

画像**Augmentation**は限定的にしか使われなかった、あるいは効果が薄かったという報告が多く、一方で**同一プロンプトに対して異なるSeedで画像を複数生成**し、学習時にランダムに選択することが有効でした。

最終的な提出は、異なるバックボーンや学習設定を持つ複数のモデルの**埋め込みベクトルをアンサンブル**（重み付き平均など）するアプローチが一般的でした。一部では、kNNのようなテキスト検索ベースの手法や、画像キャプション生成モデルを補助的に使うアプローチも見られました。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/stable-diffusion-image-to-prompts/discussion/412936)**

* **アプローチ:** 画像からプロンプト埋め込みを直接予測。大規模データ生成とモデルFine-tuning。
* **アーキテクチャ:** ViT-Large, ConvNeXt-XXLarge, BLIP-2 Vision Model (LLM部除去)。追加FC層が効果的。
* **アルゴリズム:** Cosine Embedding Loss (?)。(Optimizer/Schedulerは詳細不明だが、Warmup付きCosine Scheduleが一般的)。EMA。
* **テクニック:**
    * **データ生成:** 約1060万枚生成 (DiffusionDB, COCO, VizWiz等 + ChatGPT生成 + COYO-700M事前学習)。重複除去、類似度フィルタリング。高速生成 (xformers, fp16, 512x512, 25 steps)。
    * **学習:** 2段階学習 (LQデータ事前学習 -> HQデータFine-tuning)。LORA (下位層に適用)。入力解像度336x336。Augmentationなし(CLIP)/デフォルト(BLIP2)。
    * **アンサンブル:** 3モデルの埋め込みを重み付き平均。

**[2位](https://www.kaggle.com/competitions/stable-diffusion-image-to-prompts/discussion/412819)**

* **アプローチ:** 画像からプロンプト埋め込みを直接予測。CLIPモデルのFine-tuning。
* **アーキテクチャ:** CLIP Vision Model (ConvNeXt-xxlarge, BLIP-2 Vision, EVA02-L, EVA02-e)。一部モデルにQ-Former追加。
* **アルゴリズム:** (損失不明、Cosine Embedding Lossが有力)。(Optimizer/Scheduler不明)。
* **テクニック:**
    * **データ生成:** 約880万枚生成 (DiffusionDB, COCO, OpenImages+BLIP2キャプション, ChatGPT生成)。高速生成 (DPMSolver++, 16 steps, xformers)。異なるseed、異なるguidance scale (7.5/9.0)で生成。OpenImagesキャプションはChain生成で複雑化。
    * **学習:** LP-FT (線形プロービング→Fine-tuning)。高解像度学習 (224 -> 336/448)。
    * **Q-Former:** Vision Modelの特徴量をトークン埋め込みに変換し、Sentence Transformerのトークン埋め込みと比較（ハンガリアン法でマッチング）。効果は限定的。
    * **アンサンブル:** 4モデル (+Q-Former付き) の重み付き平均 (Adamで重み最適化)。

**[3位](https://www.kaggle.com/competitions/stable-diffusion-image-to-prompts/discussion/412887)**

* **アプローチ:** 画像からプロンプト埋め込みを直接予測。CLIPモデルのFine-tuning。
* **アーキテクチャ:** CLIP Vision Model (ViT-Base, ViT-Large, ViT-Huge, ConvNeXt-Large/XXLarge)。
* **アルゴリズム:** (損失不明、Cosine Embedding Lossが有力)。(Optimizer/Scheduler不明)。
* **テクニック:**
    * **データ生成:** 約40万枚生成 (Vizwizが鍵、DiffusionDB, COCO, Lexica)。DiffusionDBはプロンプトフィルタリングとハードサンプリングで選定。
    * **学習:** CLIP Fine-tuning戦略 (LP-FT, EMA+Layer-wise LR Decay)。
    * **Augmentation:** 弱めのAugmentation (Crop, RandomErase, RandAug(一部除外))。同一プロンプト別シード生成 (10-20%)。Invisible Watermark Augmentation (効果小)。
    * **アンサンブル:** 複数モデル (ViT-L, ViT-H, ConvNext-L/XXLなど) の平均。

**[4位](https://www.kaggle.com/competitions/stable-diffusion-image-to-prompts/discussion/412863)**

* **アプローチ:** テキスト検索ベースとViTエンコーダ予測のアンサンブル。
* **アーキテクチャ:**
    * **テキスト検索:** CLIP-bigG, CLIP-H14 (Embedding検索用)。CLIP Interrogator風 (コンポーネント検索)。CoCa (キャプション生成)。
    * **ViTエンコーダ:** CLIP Vision Model (ViT-L, H, G, bigG)。
* **アルゴリズム (ViT):** Cosine Embedding Loss。(Optimizer/Scheduler不明)。
* **テクニック:**
    * **データ:** 多様なソースからプロンプト収集 (DDB, Gustavosta, Krea, Discord, Midjourney, GCC, nocaps, coco, textcaps, WIT, MagicPrompt, etc.)。画像生成 (約700万枚)。テキスト検索用に別データセット (YFCC, Laion CoCo, Datacomp) も使用。
    * **テキスト前処理:** クリーニング、重複除去。
    * **テキスト検索:** 5600万埋め込みDBから Exhaustive Search (高速化: データ/次元削減(PCA), fp16, チャンク化, torch.mm)。上位60候補をCLIP-H14で再ランク付け。
    * **Interrogator:** 14種のテンプレートに基づき、160万コンポーネントDBから検索・結合。
    * **ViT学習:** 2段階学習 (DDB -> 自前生成データ)。Layer Freezing。
    * **TTA (ViT):** 異なる画像正規化統計 (事前学習時 vs データセット計算値) で2回推論し平均。
    * **アンサンブル:** テキスト検索結果とViTエンコーダ予測 (4モデル平均) を重み付き平均 (CV最適/LB最適の2種提出)。

**[5位](https://www.kaggle.com/competitions/stable-diffusion-image-to-prompts/discussion/412965)**

* **アプローチ:** 画像からプロンプト埋め込みを直接予測。複数モデルのアンサンブル。
* **アーキテクチャ:** eva02_large, convnext_xxlarge, convnext_large, vit_large。シンプルなBackbone + FC層(biasなし)。
* **アルゴリズム:** Cosine Embedding Loss。MADGRAD Optimizer。Warmup + Cosine Schedule。
* **テクニック:**
    * **データ生成:** 約500万枚生成 (cc3m, lexica, diffusiondb, mscoco)。同一プロンプトから3枚生成。重複プロンプト除去 (類似度>0.9)。
    * **CV戦略:** プロンプト埋め込みの類似度に基づきGroupKFold (10 Fold、Fold0を検証用)。
    * **学習:** 2段階学習 (初期データセット -> 全データセットFine-tuning)。高解像度入力 (336, 384, 448)。軽量なAugmentation (RandomResizedCrop, ColorJitter)。
    * **アンサンブル:** 4モデル x 複数Fold (計18 Fold) の重み付き平均。重みはモデルごと、または次元ごと (384x4) で学習。

**[6位](https://www.kaggle.com/competitions/stable-diffusion-image-to-prompts/discussion/412680)**

* **アプローチ:** 画像からプロンプト埋め込みを直接予測。最新Backboneの活用。
* **アーキテクチャ:** eva_large (336), convnext_large (384), swin_large (384)。
* **アルゴリズム:** Cosine Embedding Loss (+ 0.01 * MSELoss)。(Optimizer/Scheduler不明)。
* **テクニック:**
    * **データ生成:** 約887万枚生成 (多様な画像/ビデオキャプションDB: COCO, DiffusionDB, Flickr, Nocaps, SBU, Textcaps, VizWiz, WebVidなど)。重複除去 (faiss)、テキスト前処理 (非英語/短すぎる/長すぎるキャプション除去)。深度効果のため低解像度データや同一プロンプト別シードデータ追加。
    * **高速生成:** torch.compile, xformers。
    * **学習:** バッチサイズ128, fp16。Horizontal Flipのみ有効。
    * **アンサンブル:** 14モデル (3 Backbone x 複数学習設定/データ) の平均。TTA (解像度変更: 460, 512)。

**[7位](https://www.kaggle.com/competitions/stable-diffusion-image-to-prompts/discussion/412694)**

* **アプローチ:** CLIP Vision Tower + キャプションモデルのアンサンブル。カスタムプロンプト生成。
* **アーキテクチャ:** CLIP Vision Model (convnext_xxlarge, eva02_largeなど + 小型ViT/ConvNeXt)。キャプションモデル (GIT)。
* **アルゴリズム:** (損失不明、Cosine Embedding Lossが有力)。(Optimizer/Scheduler不明)。Layer-wise LR Decay。WiSE-FT (アンサンブルには不使用)。
* **テクニック:**
    * **データ生成:** 多様なソース (DiffusionDB, CC, COCO, Flickr, ChatGPT) + カスタムプロンプト生成器 (主題、メディア、形容詞、動詞の組み合わせ)。重複除去 (類似度<0.7)。
    * **ヘッド事前学習:** カスタムプロンプトからCLIPテキスト埋め込みと競技用埋め込みを生成し、両者をマッピングする線形層を事前学習。これをモデルの最終層の初期値として利用。
    * **学習:** 勾配累積で大バッチサイズ模擬。軽量Augmentation (RandomResizedCrop)。
    * **アンサンブル:** 複数CLIPモデル + GITキャプションモデルの重み付き平均。


