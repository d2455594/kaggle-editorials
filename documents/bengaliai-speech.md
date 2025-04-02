---
tags:
  - Kaggle
startdate: 2023-07-18
enddate: 2023-10-18
---
# Bengali.AI Speech Recognition
[https://www.kaggle.com/competitions/bengaliai-speech](https://www.kaggle.com/competitions/bengaliai-speech)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、提供されたベンガル語の音声データに対して、その内容を正確なテキスト（句読点を含む）に書き起こす自動音声認識 (Automatic Speech Recognition, ASR) システムを開発することです。
* **背景:** Bengali.AI は、ベンガル語の自然言語処理 (NLP) と音声技術の発展を促進するコミュニティ主導のイニシアチブです。ベンガル語は話者数が多い一方で、高品質な音声認識リソースが比較的少ない「低リソース言語」の一つとされています。このコンペティションを通じて、ベンガル語ASR技術の向上と、関連するデータセットやモデルの開発を促進することが期待されています。
* **課題:**
    * 提供された訓練データセットに**多くのノイズ**（不正確な書き起こし、低い音質、短い音声断片など）が含まれており、これを適切に処理・フィルタリングする必要があること。
    * ベンガル語という**低リソース言語**であるため、利用可能な高品質なラベル付き学習データが限られていること。
    * 多様な話者、アクセント、録音環境、発話スタイル（読み上げ、自発話）に対応できる**頑健なモデル**を構築すること。
    * 音声認識だけでなく、文の区切りを示す**句読点（। , ? など）**も正確に予測する必要があること。
    * 訓練データにはない未知のドメインや音響条件（OOD: Out-of-Distribution）のテストデータに対する**汎化性能**を確保すること。
    * 長時間の音声ファイルを処理するための効率的な推論方法。

**データセットの形式 (Dataset Format)**

提供される主なデータは、ベンガル語の音声ファイルとその書き起こしテキストです。

1.  **トレーニングデータ:**
    * `train.csv`: 各音声ファイルに関するメタデータ。
        * `id`: 音声ファイル名（拡張子なし）。
        * `sentence`: **ターゲット変数**となる、書き起こされたベンガル語の文（句読点を含む場合がある）。
        * `split`: データ分割情報 (`train` または `valid`)。
        * その他、話者の性別 (`gender`)、アクセント (`accent`)、ドメインなどの情報。
    * `train_mp3s/`: MP3形式の音声ファイル。ファイル名は `train.csv` の `id` に対応。
    * **外部データ:** OpenSLR (37, 53), MADASR, Shrutilipi, Kathbath, Macro など、公開されている他のベンガル語音声データセットの利用が推奨・許可されています。
2.  **テストデータ:**
    * `test.csv`: テストセットの音声ファイルID (`id`) のリスト。
    * `test_mp3s/`: 予測対象となるMP3形式の音声ファイル。
3.  **`sample_submission.csv`**:
    * 提出フォーマットのサンプル。
        * `id`: テスト音声ファイルID。
        * `sentence`: 予測された書き起こしテキスト（句読点を含む）。

**評価指標 (Evaluation Metric)**

* **指標:** **WER (Word Error Rate)** 単語誤り率。
* **計算方法:** 予測されたテキストと正解のテキストを比較し、単語レベルでの誤り（置換、削除、挿入）の総数を、正解テキストの総単語数で割って計算されます。
    * `WER = (Substitutions + Deletions + Insertions) / Number of Words in Reference`
    * 比較の前に、テキストは通常、小文字化、句読点や特定記号の除去などの正規化処理が行われます（ただし、このコンペでは句読点も評価対象）。
* **意味:** 音声認識システムの精度を評価するための標準的な指標です。予測されたテキストが正解とどれだけ異なっているかを示します。スコアは**低い**ほど（0に近いほど）、認識精度が高いことを意味します。

要約すると、このコンペティションは、ノイズの多いデータや低リソースという課題に対処しながら、ベンガル語の音声ファイルを句読点込みで正確にテキスト化するASRタスクです。データは音声ファイルとテキストラベルで構成され、性能は単語誤り率 (WER、低いほど良い) によって評価されます。

---

**全体的な傾向**

このベンガル語音声認識コンペティションでは、データセットに含まれる**ノイズへの対処**が成功の鍵となりました。多くのチームが、提供されたデータの品質評価（WERやMOSスコアに基づくフィルタリング）や、信頼できる外部データセット (OpenSLR, MADASR, Shrutilipiなど) の活用、疑似ラベリング、TTSによるデータ合成を行いました。

モデルアーキテクチャとしては、**事前学習済みモデルのファインチューニング**が主流で、特に **Whisper (medium)** や **Wav2Vec2 (IndicWav2Vec, XLS-R)** が広く採用されました。これらは大量のデータで事前学習されており、低リソース言語に対しても比較的高い性能を発揮します。

性能向上のための重要なテクニックとして、**音声データ拡張**（ノイズ付加、速度・ピッチ変更、複数音声の連結、SpecAugmentなど）、**言語モデル (LM)** の統合（主にKenLMを用いたn-gramモデル）、そして**句読点予測モデル**の導入が挙げられます。句読点予測は、ASRモデルとは別に、BERT系のモデルを用いたトークン分類タスクとして解かれることが一般的でした。

推論時には、長時間の音声を処理するための**チャンキング**や、デコード精度を高めるための**ビームサーチ**、そして複数のモデル出力を組み合わせる**アンサンブル**が有効でした。

**各解法の詳細**

**[1位](https://www.kaggle.com/competitions/bengaliai-speech/discussion/447961)**

* **アプローチ:** Whisper + 句読点モデル。データクリーニングと疑似ラベル、TTSデータが鍵。
* **アーキテクチャ:** ASR: OpenAI Whisper-medium。句読点: MuRIL-base-cased (TokenClassification)。
* **アルゴリズム:** ASR: Sequence-to-Sequence。句読点: トークン分類。
* **テクニック:** 複数外部データセット使用。GoogleTTSによる音声合成。疑似ラベリング (YouTube動画)。WERに基づくデータフィルタリング。独自Tokenizer学習 (推論高速化)。Augmentation (スペクトログラムマスキング、リサンプリング、速度/ピッチ変更)。句読点モデルは4モデルアンサンブル。推論時ビームサーチ (num_beams=4)、チャンク処理。

**[2位](https://www.kaggle.com/competitions/bengaliai-speech/discussion/447976)**

* **アプローチ:** IndicWav2Vec + LM + 句読点モデル。重度なAugmentationとデータ処理。
* **アーキテクチャ:** ASR: ai4bharat/indicwav2vec_v1_bengali (CTC)。LM: KenLM (6-gram)。句読点: ai4bharat/IndicBERTv2-MLM-Sam-TLM (LSTMヘッド付きTokenClassification)。
* **アルゴリズム:** CTC、n-gram LM、トークン分類。
* **テクニック:** 複数外部データセット使用。データ正規化。Augmentation (audiomentationsライブラリ使用: TimeStretch, RoomSimulator, 背景ノイズ/音楽付加, Gain, SpecAugment)。短い音声を連結するAugmentation。複数サイクル学習 (Cosine scheduler with restarts)。推論時チャンク処理。LMは大規模テキストコーパスで学習。句読点モデルはトークンマスキングAugmentation、3 Foldアンサンブル、ビームサーチ。

**[3位](https://www.kaggle.com/competitions/bengaliai-speech/discussion/447957)**

* **アプローチ:** IndicWav2Vec + LM + 句読点モデル。データフィルタリングとDemucsノイズ除去。
* **アーキテクチャ:** ASR: ai4bharat/indicwav2vec_v1_bengali (CTC)。LM: KenLM (5-gram)。句読点: XLM-RoBERTa-large/base (TokenClassification)。
* **アルゴリズム:** CTC、n-gram LM、トークン分類。
* **テクニック:** データフィルタリング (validデータで学習したモデルでtrainデータを予測し、高WERデータを除外)。複数外部テキストデータでLM学習。推論時: 音声長でソートし動的パディング。Demucsによるノイズ除去 (適用有無を予測結果のトークン数で判断)。句読点モデルは損失重み調整 (PADトークン重み0)、アンサンブル。

**[4位](https://www.kaggle.com/competitions/bengaliai-speech/discussion/447995)**

* **アプローチ:** 大規模Wav2Vec2 + LM。複数ステージ学習とカスタムスケジューラ。
* **アーキテクチャ:** ASR: facebook/wav2vec2-xls-r-1b (CTC)。LM: KenLM (7-gram)。
* **アルゴリズム:** CTC、n-gram LM。
* **テクニック:** データフィルタリング (別モデルでの予測WERに基づく)。3段階学習 (異なる乱数シード)。カスタム学習率スケジューラ (LinearWarmupCosine3LongTail)。Augmentation (TimeStretch, Gain, PitchShift, 背景ノイズ付加)。合成Augmentation (音声分割・結合)。IndicCorp v1/v2でLM学習。後処理 (正規化、Dari適用)。

**[5位](https://www.kaggle.com/competitions/bengaliai-speech/discussion/448006)**

* **アプローチ:** IndicWav2Vec + LM + 句読点モデル。特徴量レベルでのASRモデルアンサンブル。
* **アーキテクチャ:** ASR: ai4bharat/indicwav2vec_v1_bengali (CTC) 複数 + Transformer Encoder + CTC層 (アンサンブル用)。LM: KenLM (5-gram、プルーニング)。句読点: XLM-Roberta-Large。
* **アルゴリズム:** CTC、n-gram LM、トークン分類、Transformer Encoder。
* **テクニック:** データフィルタリング (MOSスコア、複数モデルでのWER)。ASRアンサンブル (複数モデルの最終隠れ状態を結合し、追加のTransformer EncoderとCTC層で学習)。LMのテキストデータ前処理 (文字セット制限、特殊トークン置換)。LMのメモリ管理工夫 (pyctcdecode)。句読点モデルの後処理 (最後の単語後に句読点を強制)。

**[7位](https://www.kaggle.com/competitions/bengaliai-speech/discussion/448074)**

* **アプローチ:** IndicWav2Vec + LM。句読点を語彙に含める。複数モデルの再スコアリングアンサンブル。
* **アーキテクチャ:** ASR: IndicWav2Vec (CTC、カスタム語彙)。LM: KenLM (5-gram)。
* **アルゴリズム:** CTC、n-gram LM。
* **テクニック:** データフィルタリング (文字数/音声長比率、他モデル予測WER)。外部データ使用。句読点をモデル語彙とLMコーパスに含める。アンサンブル (複数モデルの出力をLMで再スコアリングし、最高スコアの文を選択)。

**[10位](https://www.kaggle.com/competitions/bengaliai-speech/discussion/448126)**

* **アプローチ:** 単一Whisperモデル。データクリーニングと推論時の長文対応。
* **アーキテクチャ:** ASR: OpenAI Whisper-medium。
* **アルゴリズム:** Sequence-to-Sequence。
* **テクニック:** データクリーニング (WER/MOSスコアに基づくルール)。複数音声を結合して学習データに使用。Augmentation (SpecAugment, SpecAugment++, CutOut)。推論時: 長時間音声 (>30秒) / 長文 (>448トークン) への対応。ビームサーチ (num_beams=4)。
