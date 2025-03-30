---
tags:
  - Kaggle
startdate: 2024-04-04
enddate: 2024-06-11
---
# # BirdCLEF 2024
https://www.kaggle.com/competitions/birdclef-2024


**全体的な傾向**

上位解法では、まずデータ前処理とラベル調整に重点を置いており、train_audio のほか、pseudo labeling によって生成された unlabeled_soundscapes を効果的に活用することで、ターゲットドメイン（実際の音風景）の特性に適応する戦略が見受けられました。音声を 5 秒や 10 秒単位に分割し、隣接チャンクとの統合や、Mel スペクトログラム生成のパラメータ（n_fft、hop_length、n_mels、fmin、fmax など）の最適化を行うことで、鳥の鳴き声の特徴をより正確に捉える工夫が施されています。

また、各解法は EfficientNet、EfficientViT、RegNet、seresnext など、画像認識で実績のある CNN/SED モデルをベースに、Raw signal を直接入力する手法や、複数の入力（スペクトログラムと生波形）を融合するハイブリッドモデルを組み合わせるなど、多様なアーキテクチャを採用しています。さらに、pseudo labeling、チェックポイントスープ（モデルの重み平均化）、知識蒸留といった段階的な学習戦略や、複数モデルのアンサンブルによって予測の安定性と精度向上を実現している点も特徴的です。

加えて、データ拡張（Mixup、Cutmix、XY masking、時間・周波数軸マスキングなど）や、推論時の平滑化処理、OpenVINO や INT8 量子化を活用した高速化手法など、学習・推論の双方での最適化施策が盛り込まれ、全体として精度向上と効率化の両立を目指したアプローチが全体の傾向として挙げられます。

**各解法の詳細**

**1位** 

- **アプローチ・前処理**  
    ・2024年の train_audio と pseudo labeling による unlabeled_soundscapes を利用。  
    ・Google 鳥鳴き声分類器で音質フィルタリング、重複除去、一次・二次ラベルの調整を実施。
    
- **モデル・入力**  
    ・10秒チャンク（2×5秒）を入力とし、Mel スペクトログラム（n_fft=1024、hop_length=500、n_mels=128 など）を作成。  
    ・EfficientNet_b0 や RegNetY_008（ImageNet 事前学習済み）をバックボーンに採用。
    
- **学習・最適化**  
    ・CrossEntropyLoss、AdamW、CosineAnnealingLR で学習（7〜12エポック、バッチサイズ96）。  
    ・データ拡張（ランダムチャンク抽出、XY masking、horizontal cutmix）を実施。
    
- **推論と後処理**  
    ・Sigmoid による推論、チャンクの平均化と「min()」による不確実性低減。  
    ・OpenVINO を用いた推論高速化とメモリ内キャッシュなどで全体推論時間の短縮。  
    

---

**2位**

- **アプローチ・前処理**  
    ・初めは5秒のみを利用し、追加データや外部クラスは使わずに始動。  
    ・Google の分類器でフィルタリングし、pseudo label の活用で段階的に性能向上。
    
- **モデル・入力**  
    ・EfficientNet B0（tf_efficientnet_b0_ns）を中心とし、256×256 の 3 チャンネル Mel スペクトログラムを使用。
    
- **学習・最適化**  
    ・CosineAnnealingLR（ウォームアップ含む）、BCE と FocalLoss の平均を損失関数に採用。  
    ・ドロップアウト層の追加、チェックポイントスープでモデルの安定化を図る。
    
- **Pseudo Labeling**  
    ・ターゲットドメインの unlabeled_soundscapes を活用し、段階的に擬似ラベルで学習データを拡充。
    
- **推論・後処理**  
    ・各チャンクの予測を近隣ウィンドウと平均し、最終的な ensemble で LB スコアを向上。  
    

---

**3位**

- **アプローチ・前処理**  
    ・訓練データに加え、Xeno-Canto や過去コンペのデータを補完。  
    ・各レコードからランダムな 5 秒クリップを抽出、必要に応じてパディングを実施。
    
- **モデル・入力**  
    ・画像入力として log Mel スペクトログラム（224×224 または 288×288）を利用。  
    ・EfficientViT、EfficientNet、MobileNet、TinyNet、MNASNet など多様な CNN ベースを組み合わせ。
    
- **学習・最適化**  
    ・Pseudo Labeling によるデータ拡充と知識蒸留を積極的に活用。  
    ・大きなバッチサイズや複数エポック（第一段階と第二段階で異なるエポック数）で学習。
    
- **後処理**  
    ・音声全体の情報を活かすため、ウィンドウごとに隣接チャンクと融合する手法を採用。  
    

---

**4位**

- **アプローチ・前処理**  
    ・Melspec モデルと Raw Signal モデルの両面からアプローチ。  
    ・TTA（Test Time Augmentation）を用い、複数の予測結果を統合。
    
- **モデル・入力**  
    ・Model A：2021–2023 の事前学習済みモデル（seresnext26ts、rexnet_150 など）で Melspec を生成。  
    ・Model B：シンプルな Melspec CNN（inception-next-nano を中心）で、xeno-canto の追加データも活用。  
    ・Model C：Raw signal を直接入力し、tf_efficientnet_b0_ns で SED ヘッドを付加。
    
- **学習・最適化**  
    ・BCE Loss とサンプルウェイト、クラス不均衡対策を実施。
    
- **推論・後処理**  
    ・Weighted Mean、Geometric Mean、平滑化、Cut-off、隣接ウィンドウの最大値選択など多彩な手法で最終予測を調整。  
    

---

**5位**

- **アプローチ・前処理**  
    ・生データ（Raw signal）とスペクトラム双方のモデルを組み合わせるハイブリッド手法。  
    ・音声波形の正規化や、5秒のランダムサンプルを訓練に採用。
    
- **モデル・入力**  
    ・Raw signal モデル（hgnetb0 ベース）とスペクトラムモデル（efficientnet_b0）を個別に構築。  
    ・両者の特徴量を連結し、融合層で統合。
    
- **学習・最適化**  
    ・Mixup（確率1）やXY Cut Out を含むデータ拡張を実施。  
    ・BCEWithLogitsLoss を用い、OpenVINO による推論高速化を実現。  
    

---

**6位**

- **アプローチ・前処理**  
    ・2024年データのみでの学習に加え、Pseudo Labeling と追加データの導入でスコア向上。
    
- **モデル・入力**  
    ・ResNet 系（resnet18d, resnet34d, efficientnet_v2s など）を採用し、5秒単位の入力で学習。
    
- **学習・最適化**  
    ・Pseudo Label の損失計算を、ファイル内の複数チャンクごとに個別に実施し、ロスの計算方法を工夫。  
    ・移動平均を用いた平滑化で、最終予測を改善。
    
- **その他**  
    ・サンプリング方法に RMS を利用し、音量変動に対応。  
    

---

**7位**

- **アプローチ・前処理**  
    ・SED モデルと CNN モデルの組み合わせ。  
    ・Birdnet や鳥鳴き声分類器を用いて音声中の鳥の区間を抽出。
    
- **モデル・入力**  
    ・SED（tf_efficientnet_b2_s_in21k など）や CNN（resnet34d）を用いて全種または希少種に特化したサブセットモデルを構築。
    
- **学習・最適化**  
    ・知識蒸留や Sumixup（Mixup に類似した手法）を導入。  
    ・Int8 量子化とモデル分割（希少種のサブセット化）により、ターゲットドメインへの適応を強化。
    
- **推論・後処理**  
    ・Logit や Rank 平均を組み合わせた ensemble 戦略。  
    

---

**8位**

- **アプローチ・前処理**  
    ・大規模な事前学習と pseudo labeling を組み合わせ、まずは他年のデータで学習後、2024年データにファインチューニング。
    
- **モデル・入力**  
    ・SED：eca_nfnet_l0 や tf_efficientnet_b0.ns_jft_in1k を中心に、シンプルな 2D CNN（eca_nfnet_l0）も併用。  
    ・Melスペクトログラムのパラメータ設定がシンプルかつ効果的。
    
- **学習・最適化**  
    ・FocalLossBCE を採用し、secondary label にも一定の重みを与える。  
    ・Mixup、Sumix といったデータ拡張とシンプルな TTA を実施。
    
- **推論・後処理**  
    ・OpenVINO を利用して推論時間を短縮（約90分程度）。  

---

**9位**

- **アプローチ・前処理**  
    ・クロスバリデーションは行わず、public LB に直接最適化。  
    ・Mixup や Cutmix を中心とした拡張で、オーバーフィッティングを抑制。
    
- **モデル・入力**  
    ・主に eca_nfnet_l0 ベースのモデルを採用。  
    ・ボトルネック層や CAD（Contrastive Adversarial Domain）を導入して特徴圧縮とドメイン適応を狙う。
    
- **学習・最適化**  
    ・SupCon loss（コントラスト学習）を用いた損失関数の工夫で、BCE Loss と組み合わせた学習。
    
- **後処理**  
    ・隣接ウィンドウとの平均化などで平滑化を実施し、最終的な予測のブレを低減。  
    

---

**10位**

- **アプローチ・前処理**  
    ・2021～2024年の全データを活用し、段階的に2024年と unlabeled data でファインチューニング。
    
- **モデル・入力**  
    ・tf_efficientnet_v2 系（b0, b3）を使用し、単一チャネルのスペクトログラムを入力。
    
- **学習・最適化**  
    ・複数段階の学習ステージを通じ、最終的に pseudo labeling を活用して unlabeled data を取り入れることで精度向上を実現。
    
- **推論・後処理**  
    ・各段階の出力を組み合わせた ensemble 戦略を採用し、最終スコアを向上。  
    ※（10位の詳細は一部省略されているが、他順位と同様にデータ拡張、pseudo labeling、ensemble 戦略がキーとなっている）  



