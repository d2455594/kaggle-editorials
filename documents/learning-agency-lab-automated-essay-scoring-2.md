---
tags:
  - Kaggle
startdate: 2024-04-04
enddate: 2024-07-03
---
# Learning Agency Lab - Automated Essay Scoring 2.0
https://www.kaggle.com/competitions/learning-agency-lab-automated-essay-scoring-2

**全体的な傾向**

上位解法では、Transformerベースの言語モデル（特にDeBERTa）のアンサンブルが主流でした。データセットの特性（Persuade Corpus 2.0とKaggle-onlyデータ）を考慮した学習戦略や、疑似ラベリング、閾値最適化などが重要なテクニックとして用いられました。

**各解法の詳細**

**1位**

- **アプローチ:** CV（交差検証）とLB（リーダーボード）を信頼し、データ分析と実験に基づいて新しいデータ採点パターンを発見。それらのスコアを反映したDeBERTaアンサンブルを訓練。2段階の疑似ラベリングを実施し、「古い」データを「新しい」スコアで再学習。浮動小数点予測を閾値処理により整数に変換。オーバーフィットを避けたアンサンブル。
- **アーキテクチャ:** DeBERTa large、DeBERTa baseのアンサンブル。
- **アルゴリズム:** MSE損失、Binary Cross-Entropy損失、AdamWオプティマイザ、コサイン減衰スケジューラ。Nelder-Mead法、Powell法（閾値探索）。
- **テクニック:**
    - **CV-LB相関の獲得:** 新旧データの採点基準の違いを分析し、2段階学習（古いデータで事前学習、新しいデータでファインチューニング）を実施。
    - **DeBERTaの訓練:** 異なるプーリング方法、損失関数、コンテキスト長などを試行し、多様なモデルを生成。3シード平均化による安定化。
    - **疑似ラベリング:** 新しいデータでファインチューニングしたアンサンブルを用いて古いデータを疑似ラベル化（2ラウンド）。
    - **閾値処理:** 浮動小数点予測を整数スコアに変換するための閾値を探索（`minimize` 関数とPowell法を使用）。アンサンブルの重みにも閾値を適用。
    - **アンサンブル:** 単純平均、Nelder-Mead法、Hill-climbing法。

**2位**

- **アプローチ:** 2段階学習（Kaggle-Persuadeで事前学習後、Kaggle-Onlyでファインチューニング）。MLM（Masked Language Modeling）による事前学習。最適な閾値を探索し、モデルをアンサンブル。
- **アーキテクチャ:** DeBERTa-v3-largeを主に使用。
- **アルゴリズム:** BCEWithLogitsLoss（順序回帰）、MSE損失。
- **テクニック:**
    - **2段階学習:** テストセットと類似性の高いKaggle-Onlyデータでファインチューニング。
    - **MLM:** 全学習データで10エポックのMLM事前学習。
    - **CV:** 4分割SKF。Kaggle-PersuadeとKaggle-Onlyの両データで検証。
    - **閾値探索:** OptimizedRounderを使用し、最終閾値を調整。Kaggle-Onlyデータのみで閾値を探索する実験も実施。
    - **モデル:** 順序回帰、Petモデル（prompt tuning）、文埋め込みの平均プーリングなど、多様なモデルを試行。
    - **アンサンブル:** 投票アンサンブル。

**3位**

- **アプローチ:** 競技データとPersuade Corpus 2.0の利用を重視。2段階学習（全データで事前学習後、競技データに焦点を当ててファインチューニング）。ソフトラベリング、MLM事前学習、レイヤー凍結。
- **アーキテクチャ:** DeBERTa-v3-base、DeBERTa-v3-largeを主に使用。
- **アルゴリズム:** BCE損失、AdamWオプティマイザ。
- **テクニック:**
    - **データ利用:** 競技データとPersuade Corpus 2.0を区別して分析し、競技データに焦点を当てた学習戦略を採用。
    - **データ前処理:** 改行文字を特殊トークンに変換。
    - **CV戦略:** Prompt name/scoreに基づいたMultilabelstratifiedkfold。Data AとData Bを別々に分割。
    - **モデルと学習:** MLM事前学習とレイヤー凍結（9層または6層）。回帰とBCE損失を使用。2段階学習でソフトラベリングを活用。
    - **アンサンブル:** Nelder-Mead法を用いた単純なアンサンブル。

**4位**

- **アプローチ:** Persuadeデータと非Persuadeデータの非互換性を考慮。テストサンプルは全て非Persuadeデータであると仮定。データソースのタグ付け、データソース分類ヘッドの追加、非Persuadeデータのスコアに基づく早期停止。
- **アーキテクチャ:** DeBERTa-large、DeBERTa-v3-large、Qwen2-1.5B-Instructのアンサンブル。
- **アルゴリズム:** AdamWオプティマイザ。
- **テクニック:**
    - **データソースの区別:** 入力にデータソースのタグを追加。データソース分類ヘッドを追加。一部モデルではデータソースごとに異なる分類ヘッドを使用。
    - **早期停止:** 非Persuadeデータのスコアに基づいて早期停止。
    - **提出:** 全てのサンプルを非Persuadeデータとして扱う。
    - **その他:** 疑似ラベリングと蒸留のためのタグのスワップを試行。動的マイクロバッチ照合、高速なDeBERTa実装。

**5位**

- **アプローチ:** Kaggle-onlyデータとPersuade 2.0データを用いた2段階ファインチューニング。DeBERTaモデルとLGBMのブレンド。
- **アーキテクチャ:** DeBERTa (small/base/large)、LightGBM。
- **アルゴリズム:** LightGBM。
- **テクニック:**
    - **データ分割:** Kaggle-onlyデータをStratifiedKFoldで分割。
    - **ファインチューニング:** Persuade 2.0データとKaggle-onlyデータでDeBERTaモデルをファインチューニング。その後、Kaggle-onlyデータのみで再学習。
    - **ブレンド:** DeBERTaモデルの予測とLGBMの予測を重み付け平均（0.9:0.1）。

**6位**

- **アプローチ:** LBプロービングとデータ分析に基づいた検証戦略。`persuade` フラグの追加、テストセットのみで学習したVectorizerの利用、DeBERTaの最大推論長の増加、DeBERTa OOFのCDF変換、外部特徴量の追加、リラベリング。
- **アーキテクチャ:** DeBERTa-v3-large、LightGBM。
- **アルゴリズム:** TfidfVectorizer、CountVectorizer。
- **テクニック:**
    - **検証:** テストセットのトピック分布を推定し、それに基づいて検証データの重みを調整。`persuade` フラグを特徴量として追加。
    - **特徴量エンジニアリング:** テストセットのみでVectorizerを学習。DeBERTa OOFをCDFに変換。texstatとspaCyベースの特徴量を追加。
    - **リラベリング:** DeBERTaの予測と実際のラベルの差が大きい場合にラベルを修正。

**7位**

- **アプローチ:** 強力な公開ノートブックをベースに改善。Persuade 2.0の理解、Kaggle-onlyデータの重複、ハイパーパラメータ調整。
- **アーキテクチャ:** DeBERTa-v3-largeベースのLGBM。
- **アルゴリズム:** LGBM。
- **テクニック:**
    - **データ理解:** Persuade 2.0とKaggle-onlyデータの特性を分析。
    - **データ重複:** Kaggle-onlyデータを重複させて学習データを強調。ただし、リークを防ぐためにCVで工夫。
    - **ハイパーパラメータ調整:** LGBMのハイパーパラメータを探索的に調整。学習時のQuadratic Weighted Kappaのa, b値を調整。推論時のa値も調整。