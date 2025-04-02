---
tags:
  - Kaggle
  - コサイン類似度
  - ViT
startdate: 2023-02-14
enddate: 2023-05-16
---
# Stable Diffusion - Image to Prompts
[https://www.kaggle.com/competitions/stable-diffusion-image-to-prompts](https://www.kaggle.com/competitions/stable-diffusion-image-to-prompts)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、テキストから画像を生成するモデルであるStable Diffusion v2によって生成された画像を入力として受け取り、その画像を生成するために使用された**テキストプロンプトに対応する埋め込みベクトル (prompt embedding)** を予測するモデルを開発することです。予測対象はプロンプトのテキストそのものではなく、`all-MiniLM-L6-v2` というSentence Transformerモデルによって生成された384次元のベクトルです。
* **背景:** 近年、テキストプロンプトに基づいて高品質な画像を生成するDiffusionモデルが大きな進歩を遂げています。一方で、生成された画像から元のプロンプト（またはその意味的特徴）を推定する「Image-to-Prompt」タスクは、モデルの動作理解、生成コンテンツのモデレーション、創造的なツールの開発など、様々な応用が期待される分野です。
* **課題:** 画像から、それを生成したであろうテキストプロンプトの意味的な特徴（埋め込みベクトル）を捉える必要があります。画像には多様なオブジェクト、スタイル、構図が含まれており、これらを正確にベクトル表現にマッピングすることが求められます。また、Stable Diffusionは同じプロンプトからでも異なる画像を生成しうるため、入力画像と出力ベクトルは決定的な一対一対応ではなく、確率的な関係性を学習する必要があります。コンペティションではトレーニングデータが提供されず、参加者自身が**大規模な画像とプロンプト埋め込みのペアデータセットを構築**する必要がある点も大きな課題でした。

**データセットの形式 (Dataset Format)**

主催者からはテスト画像のみが提供され、トレーニングデータは参加者が各自で準備する必要がありました。

1.  **トレーニングデータ（参加者作成）:**
    * 一般的な作成方法:
        * **プロンプト収集:** DiffusionDB, LAION, COCO Captions, Flickr30k, WebVidなど既存の大規模データセットからテキストプロンプトを収集・フィルタリングする、あるいはChatGPTなどの大規模言語モデル（LLM）を用いてプロンプトを生成する。
        * **画像生成:** 収集・生成したプロンプトをStable Diffusion v2モデルに入力し、画像を生成する。数百万～1千万枚規模の画像生成が上位チームでは行われた。
        * **埋め込みベクトル計算:** 各プロンプトを`all-MiniLM-L6-v2`モデルに入力し、384次元の埋め込みベクトルを計算する。
        * **ペア作成:** 生成された画像と、対応するプロンプトの埋め込みベクトルをペアにしてトレーニングデータとする。
2.  **テストデータ:**
    * `images/`: Stable Diffusion v2で生成された評価用の画像（JPEG形式）が約7,500枚含まれるフォルダ。ファイル名は`{imgId}.jpg`。
    * `prompts.csv`: （実際には提供されず、これが予測対象）各テスト画像に対応する真のプロンプトの埋め込みベクトル（384次元）。
3.  **`sample_submission.csv`**:
    * 提出フォーマットのサンプル。
    * `imgId_promptId`: 画像ID (`imgId`) と、予測する埋め込みベクトルの次元インデックス (`promptId`、0から383）を組み合わせた識別子。
    * `val`: 予測された埋め込みベクトルの各次元の値（浮動小数点数）。各画像に対して384行の予測が必要です。

**評価指標 (Evaluation Metric)**

* **指標:** **コサイン類似度 (Cosine Similarity)**
* **計算方法:** 各テスト画像について、モデルが予測した384次元のプロンプト埋め込みベクトルと、実際にその画像を生成した（隠された）真のプロンプト埋め込みベクトルとの間のコサイン類似度を計算します。全テスト画像について計算されたコサイン類似度の平均値が最終的なスコアとなります。
    * Cosine Similarity(A, B) = (A ⋅ B) / (||A|| * ||B||)
* **意味:** 予測されたベクトルと真のベクトルが、高次元空間内でどれだけ同じ方向を向いているかを測定します。値は-1から1の範囲を取り、ベクトルが完全に同じ方向を指す場合に1となります。このコンペでは、予測されたプロンプト埋め込みが、元のプロンプトの意味や特徴をどれだけ正確に捉えられているかを評価します。スコアは**高い**ほど良い評価となります。

---

**全体的な傾向**

この「Image-to-Prompt Embedding」タスクでは、参加者自身による**大規模なデータセット生成**が成功の鍵となりました。数百万から1千万以上の画像とプロンプト（およびその埋め込みベクトル）のペアが、様々なソース（DiffusionDB, LAION, COCO, ChatGPTなど）から収集・生成されました。

モデリングにおいては、画像を入力として384次元のプロンプト埋め込みベクトルを直接予測する**教師あり学習のアプローチ**が主流でした。特に、**CLIP (Contrastive Language-Image Pre-Training)** モデルの**Vision Tower（画像エンコーダー）** をファインチューニングする手法が非常に効果的でした。**ViT-Large/H/G/bigG** や **ConvNeXt-Large/XXLarge**, **EVA02** といった強力な事前学習済みモデルが広く採用されました。Image Captioningモデル（BLIP, CoCaなど）も試されましたが、直接埋め込みベクトルを予測する方が優れていたようです。

学習戦略としては、非常に大規模なデータセット（COYO-700Mなど）を用いた**事前学習**と、コンペ用に生成した高品質なデータセットを用いた**ファインチューニング**の2段階学習が有効でした。CLIPモデルのファインチューニングにおいては、元の性能を損なわないように、**Linear Probing + Fine-tuning (LP-FT)**、**Layer-wise Learning Rate Decay (LLRD)**、**EMA**といったテクニックが重要視されました。また、**LoRA**による効率的なファインチューニングも一部で用いられました。入力画像の**解像度**を上げることも性能向上に寄与しました。

意外な点として、画像に対する**データ拡張（Augmentation）** はあまり効果がない、あるいは逆効果になるケースが多く、全く使わないか、非常に軽いもの（RandomResizedCrop, ColorJitterなど）に留めるのが一般的でした。

最終的な提出では、複数の異なるモデルや設定で得られた予測埋め込みベクトルを**アンサンブル**（主に重み付き平均）することが標準的なプラクティスでした。検証データセットを用いて最適なアンサンブル重みを学習するアプローチも取られました。一部では、テキスト検索やCLIP Interrogatorのような異なるアプローチの結果と組み合わせる手法も見られました。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/stable-diffusion-image-to-prompts/discussion/411237)**

* **アプローチ:** 大規模データ生成（約10.6M画像）と、ViT-L, ConvNeXt-XXL, BLIP-2 (LLM部除去) の3モデルのアンサンブル。
* **アーキテクチャ:** ViT-Large, ConvNeXt-XXLarge, BLIP-2 (Vision Tower + Q-Formerのみ)。最終層の前に大きなFC層を追加すると効果あり。
* **アルゴリズム:** 教師あり学習 (埋め込みベクトル予測)。損失関数はCosine Embedding Loss? LoRAを使用。
* **テクニック:**
    * **データ生成:** DiffusionDB_2M, COCOなど多様なソースから約2Mの高品質プロンプト("PROMPTS_HQ") + COYO-700Mから約6.6Mの低品質プロンプト("PROMPTS_LQ")を使用。画像生成高速化 (XFormer, FP16, 低解像度/ステップ数削減)。
    * **学習:** PROMPTS_LQで事前学習後、PROMPTS_HQでファインチューニング。LoRAを初期層に適用。EMA使用。AugmentationはBLIP-2のみデフォルト使用。入力解像度336x336。
    * **アンサンブル:** 3モデルの予測埋め込みを重み付き平均。多様な予測結果11個のアンサンブルで最終スコア。

**[2位](https://www.kaggle.com/competitions/stable-diffusion-image-to-prompts/discussion/410606)**

* **アプローチ:** 大規模データ生成（DiffusionDB, COCO, OpenImages+BLIP2, ChatGPT）と、複数のCLIP Vision Towerモデルのアンサンブル。
* **アーキテクチャ:** ConvNeXt-xxlarge, BLIP-2 VisionModel, EVA02-L, EVA02-e (いずれもCLIPのVision Tower)。一部モデルでQ-Formerを追加。
* **アルゴリズム:** 教師あり学習 (埋め込みベクトル予測)。損失関数はCosine Embedding Loss?
* **テクニック:**
    * **データ生成:** 画像生成高速化 (DPMSolver++, 16ステップ, xformers)。多様なシード、複数のガイダンススケール(7.5, 9.0)で生成。OpenImagesではBLIP-2を連鎖的に適用し複雑なプロンプトを生成。ChatGPTプロンプト生成では過去の生成例をプロンプトに含めて多様性を確保。約10M画像を生成。
    * **学習:** Linear-Probing (LP) 後にFine-tuning (FT)。入力解像度を段階的に上げる (224→336/448)。位置エンコーディング補間を利用。
    * **Q-Former:** 凍結したVision Towerの後段にQ-Formerを追加し、トークン埋め込みを予測（補助タスク？）。Hungarianアルゴリズムで損失計算。アンサンブルに貢献。
    * **アンサンブル:** 複数モデルの予測埋め込みを重み付き平均（重みは検証データで最適化）。

**[3位](https://www.kaggle.com/competitions/stable-diffusion-image-to-prompts/discussion/410686)**

* **アプローチ:** 約400kの比較的小規模なデータセットで、CLIPモデルのファインチューニング技術に注力。
* **アーキテクチャ:** CLIPモデル (ViT-Base, ViT-Largeがメイン)。
* **アルゴリズム:** 教師あり学習 (埋め込みベクトル予測)。損失関数はCosine Embedding Loss?
* **テクニック:**
    * **データセット:** Vizwiz Captionデータが重要（記述的で長く多様）。DiffusionDB (SD2画像のみ、ハードサンプリング含む), COCO, Lexica.artも使用。
    * **CLIPファインチューニング:** **LP-FT** (Linear Probe then Fine Tune) と **EMA + Layer-wise Learning Rate Decay (LLRD)** の組み合わせがスコア向上に不可欠 (+0.02)。最適な減衰率はモデルサイズ依存。
    * **Augmentation:** 比較的弱い拡張（Crop, RandomErase, RandAugの一部）が有効 (+0.007)。フリップは性能悪化。
    * **データ利用法:** 同じプロンプトから異なるシードで生成した複数画像を保持し、エポックごとにランダムに1枚選択。
    * **アンサンブル:** 複数のCLIPモデル (ViT-L, ViT-H, ConvNeXt-L/XXL) の予測をアンサンブル。

**[4位](https://www.kaggle.com/competitions/stable-diffusion-image-to-prompts/discussion/410798)**

* **アプローチ:** テキスト検索ベースの手法とViTエンコーダー（CLIP Vision Tower）予測のアンサンブル。
* **アーキテクチャ:**
    * テキスト検索: CLIP-bigG, CLIP-H14 embedding。
    * CLIP Interrogator風: CLIP-bigG embedding。
    * CoCaモデル。
    * ViTエンコーダー: CLIP ViT-L/H/G/bigG。
* **アルゴリズム:** テキスト検索/Interrogatorは類似度計算。CoCaはキャプション生成。ViTエンコーダーは教師あり学習 (埋め込みベクトル予測)。損失はCosine Embedding Loss。
* **テクニック:**
    * **データ生成:** 多様なソース (DDB, Gustavosta, Krea, Discord, Midjourney, Google CC, nocaps, coco, textcaps, WIT, LLM生成など) から約7M画像を生成。
    * **テキスト検索:** 56Mのプロンプト埋め込みデータベースを構築。推論時に画像埋め込み (CLIP-bigG) との類似度でTop60候補を選択し、CLIP-H14で再計算してフィルタリング。高速化のためPCA, メモリマップドnumpy, GPU行列演算を使用。
    * **CLIP Interrogator:** 1.6Mの「プロンプト構成要素」の埋め込みデータベースを構築。14個のテンプレートに基づき、各構成要素のTop8候補を検索し、連結してプロンプト候補を生成。
    * **ViTエンコーダー学習:** 2段階学習 (DDB→自前生成データ)。層ごとに凍結解除しながら学習。
    * **TTA:** 異なる正規化統計量（事前学習時と自前計算）を用いた推論結果を平均。
    * **アンサンブル:** ViTエンコーダー4種の平均と、テキスト候補生成手法3種の結果を重み付き平均。重みはCVとLBで最適化。

**[5位](https://www.kaggle.com/competitions/stable-diffusion-image-to-prompts/discussion/410688)**

* **アプローチ:** 複数のCLIP Vision Towerモデルを学習し、その予測埋め込みをアンサンブル。
* **アーキテクチャ:** eva02_large, convnext_xxlarge, convnext_large, vit_large (いずれもVision Tower)。
* **アルゴリズム:** 教師あり学習 (埋め込みベクトル予測)。損失関数: Cosine Embedding Loss。Optimizer: MADGRAD。
* **テクニック:**
    * **データ生成:** 約5Mの画像・プロンプトペアを生成 (cc3m, lexica, diffusiondb, mscoco)。一部プロンプトには複数シードで画像生成。
    * **検証戦略:** プロンプト埋め込みの類似度に基づきデータをグループ化し、GroupKFoldで分割。
    * **学習:** 10-15 epochs。Warmup期間 (1-3 epochs) はFC層のみ学習。Cosine LRスケジュール。弱いAugmentation (RandomResizedCrop, ColorJitter)。追加データセットで3-5 epochsファインチューニング。
    * **高解像度入力:** モデルごとに異なる高解像度 (336, 384, 448) で学習・推論。ViTモデルの位置エンコーディング補間を利用。
    * **アンサンブル:** 4モデル x 複数Fold (計18) の予測をアンサンブル。重みはFold 0の検証データで最適化。次元ごとに重みを学習する手法も試行（わずかに向上）。
    * **動作しなかった手法:** Mixup, GeM/GGeM Pooling, Triplet Attention, Image Captioningモデル, 損失重み付け, kNN。

**[6位](https://www.kaggle.com/competitions/stable-diffusion-image-to-prompts/discussion/410768)**

* **アプローチ:** 約8.87Mの自前生成データセットで、複数のtimmモデル（CLIP Vision Tower系）を学習しアンサンブル。
* **アーキテクチャ:** eva-large (336), convnext-large (384), swin-large (384)。
* **アルゴリズム:** 教師あり学習 (埋め込みベクトル予測)。損失関数: CosineEmbeddingLoss (margin=-0.5) + 0.01 * MSELoss。
* **テクニック:**
    * **データ生成:** 多様な画像/ビデオキャプションデータセット (COCO, DiffusionDB, Flickr, Iaprtc12, Nocaps, SBU, Senticap, Textcaps, VizWiz, VTT, WebVidなど) のプロンプトを使用。類似度0.9以上のプロンプトを除去。テキスト前処理（非英語、短すぎる/長すぎる文を除去）。DiffusionDBデータ（224x224リサイズ版含む）を追加し深度効果を狙う。画像生成高速化 (torch.compile, xformers)。
    * **学習:** 各モデル5 epochs。バッチサイズ128。FP16。Horizontal Flipのみ有効。
    * **アンサンブル:** 3バックボーン x 複数チェックポイント (計14モデル) の予測をアンサンブル。異なる入力解像度 (460x460, 512x512) でのTTAを行い、等価重みで平均。
    * **動作しなかった手法:** Hard Prompts Made Easy, Stable unCLIP, SD VAE特徴量の利用, LLMによるキャプション再生成/要約。

**[7位](https://www.kaggle.com/competitions/stable-diffusion-image-to-prompts/discussion/410618)**

* **アプローチ:** 複数のCLIP Vision Towerモデルと、ゼロショット画像キャプションモデルのアンサンブル。カスタムプロンプト生成器によるデータ拡張。
* **アーキテクチャ:** CLIP Vision Towers (convnext_xxlarge, eva02_largeが主力。ViT, convnextの小型モデルも使用)。Image Captioningモデル (GIT)。
* **アルゴリズム:** 教師あり学習 (CLIPモデル)、ゼロショット推論 (Captioningモデル)。損失関数はCosine Embedding Loss?
* **テクニック:**
    * **データ生成:** 多様なソース (DiffusionDB, CC, COCO, Flickr, ChatGPTプロンプト) から収集。類似度0.7以下でフィルタリング。**カスタムプロンプト生成器**（人/場所/物、画材、形容詞、動詞をランダムに組み合わせ）を活用。
    * **検証:** 多様なソースからのプロンプトを含み、学習データとの相関が低い検証セットを構築。
    * **Head事前学習:** CLIP Text Towerの出力と`all-MiniLM-L6-v2`の出力のマッピングを学習する線形層を事前に学習し、Vision Towerモデルの最終層の重み初期値として利用。
    * **学習:** 2 epochs。Layer-wise learning rate decay。弱い画像Augmentation (RandomResizedCrop)。勾配蓄積による疑似大バッチサイズ。WiSE-FTは単体モデルには有効だがアンサンブルには寄与せず。
    * **アンサンブル:** 複数のCLIPモデルの予測と、GITキャプションモデルの予測（の埋め込みベクトル）を重み付き平均。

