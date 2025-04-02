---
tags:
  - Kaggle
startdate: 2024-09-13
enddate: 2024-12-13
---
# FIDE & Google - Efficient Chess AI Challenge
[https://www.kaggle.com/competitions/fide-google-efficiency-chess-ai-challenge](https://www.kaggle.com/competitions/fide-google-efficiency-chess-ai-challenge)

**概要 (Overview)**

* **目的:** このコンペティションの目的は、国際チェス連盟 (FIDE) と Google の共催により、非常に**限られた計算リソース**（CPU 1コア、メモリ 5MB、バイナリサイズ 64KB、思考時間 1手10秒）の下で動作する、可能な限り強力な**チェスAI（チェスエンジン）** を開発することです。
* **背景:** 近年のチェスAIは人間を遥かに凌駕する強さを達成しましたが、その多くは高性能なハードウェアや大規模なニューラルネットワークに依存しています。このコンペティションは、「効率性 (Efficiency)」に焦点を当て、スマートフォンや低スペックなデバイスなど、リソースが限られた環境でも高性能を発揮できるチェスAI技術の開発を促進することを目的としています。
* **課題:**
    * **極端なリソース制約への対応:** 特に**バイナリサイズ 64KB**という厳しい制限の中で、強力な評価関数と探索アルゴリズムを実装する必要があります。メモリ使用量 (5MB) と CPU コア数 (1) も大きな制約です。
    * **短い思考時間:** 1手あたり平均10秒という短い時間で、可能な限り深い読みを行い、最善手を選択する必要があります。Short Time Control (STC) に特化した探索戦略が求められます。
    * **評価関数と探索のトレードオフ:** 高精度な評価関数（特に NNUE）はサイズと計算コストが大きいため、サイズ制限内で実装可能な評価関数（小型 NNUE、HCE、またはその組み合わせ）を選択し、探索アルゴリズムとのバランスを取る必要があります。
    * **コードサイズ最適化:** 不要な機能の削除、データ構造の圧縮、コンパイラ選択（Clang vs GCC）、コンパイルフラグの最適化、関数属性の利用、外部ライブラリ依存の排除など、あらゆる手段を駆使してバイナリサイズを削減する必要があります。
    * **メモリ使用量の削減:** Transposition Table (置換表) や各種履歴テーブルのサイズ削減、メモリ効率の良いデータ構造の採用などが求められます。

**データセットの形式 (Dataset Format)**

このコンペティションは、機械学習モデルではなく**チェスエンジン（実行可能プログラム）** を提出する形式です。したがって、主催者から提供される標準的な「データセット」はありません。

* **学習データ（参加者準備）:** エンジンの評価関数（特にNNUE）を学習させるために、大量のチェス局面データや棋譜が必要になります。参加者は公開されているデータセット（例: Leela Chess Zero データ、Stockfish自己対局データなど）を利用したり、自身で生成したりしました。
* **オープニングブック:** 対局開始時の定跡を指定するファイル。コンペティションの評価環境で使用される特定のオープニングブックに合わせてエンジンを調整する必要がある場合があります。コンペでは特定の`polyglot.bin`ファイルが使用されました。

**評価指標 (Evaluation Metric)**

* **指標:** **Eloレーティング (Elo Rating)**
* **計算方法:** 提出されたチェスエンジン同士が、指定されたリソース制限と時間制御の下で多数回の対局を行います。その対局結果（勝ち、負け、引き分け）に基づいて、Kaggleの評価システム（Glickoレーティングシステムまたは類似のもの）により、各エンジンの相対的な強さを示すEloレーティングが推定されます。
* **意味:** チェスプレイヤーやエンジンの強さを比較するための標準的な指標です。他の提出物との相対的な強さを示し、数値が**高い**ほど強いエンジンであることを意味します。対局結果に基づくため、対戦相手や組み合わせによる運の要素も含まれます。

要約すると、このコンペティションは、極めて厳しいリソース制限下（特に64KBバイナリサイズ）で最強のチェスエンジンを開発するプログラミングチャレンジです。評価は提出されたエンジン同士の対局結果に基づくEloレーティング（高いほど良い）で行われます。コード最適化、評価関数と探索のバランス、STC戦略が鍵となります。

---

**全体的な傾向**

このコンペティションでは、チェスエンジンの世界でデファクトスタンダードとなっている **Stockfish**、またはそのC言語移植版である **Cfish** をベースにした開発が上位を独占しました。最大の課題である**64KBという極めて厳しいバイナリサイズ制限**と5MBのメモリ制限を満たすため、参加者は徹底的な**コード最適化と機能削減**を行いました。

**最適化技術**としては、以下が広く用いられました。
* **機能削除:** マルチスレッド、テーブルベース対応、NNUE（初期段階や一部チーム）、Polyglotオープニングブックサポート、UCI（Universal Chess Interface）の不要部分、デバッグ用コードなど、コアな探索と評価に必須でない機能の徹底的な削除。
* **データ構造圧縮:** Transposition Table (置換表)、各種履歴テーブル (Continuation History, Correction Historyなど)、Pawn Hash Table などのサイズ削減や、よりメモリ効率の良い実装（ハッシュテーブル化など）への変更。Magic Bitboardの代替実装。
* **コンパイル最適化:** `Clang` コンパイラ（特に version 10）が `GCC` よりも小さいバイナリを生成する傾向があり、多くの上位チームで採用。`-O3` (速度最適化) と `-Os` (サイズ最適化) の使い分け、サイズ削減のための特定フラグ (`-fno-unwind-tables`, `-fvisibility=hidden` など) の利用。
* **コードレベル最適化:** `__attribute__((minsize, cold))` などを用いて、パフォーマンスに影響の少ない関数のコードサイズを優先的に削減。インライン展開の抑制。
* **ライブラリ依存排除:** 標準C++ライブラリ (`libstdc++`) や数学ライブラリ (`libm`)、スレッドライブラリ (`lpthread`) への依存をなくし、必要な関数を自前で実装。
* **バイナリ圧縮:** 最終的な実行ファイルを `xz` (LZMA) や `upx` などで圧縮し、提出用スクリプト内で展開。

**評価関数**については、主に2つのアプローチが見られました。
* **小型NNUE:** Stockfish由来のNNUEアーキテクチャを大幅に小型化（層数、ニューロン数削減）し、量子化 (int8/int16) を施してバイナリサイズ制限内に収める。入力特徴量の削減（Pawn配置やKingの位置に関するミラーリング活用）も重要。
* **HCE + α:** Stockfish 16まで存在したHand-Crafted Evaluation (HCE) をベースとし、非常に小さなMLPなどを追加して評価を補強する。

**探索アルゴリズム**では、1手10秒という**Short Time Control (STC)** に特化した最適化が重要でした。ルート探索での奇数plyスキップ、Fail-High時の即時枝刈り、Quiescence Search中の時間チェック、終盤の時間配分調整などが有効とされました。また、Stockfishコミュニティで開発された探索改善手法（各種Correction History、Counter Move Pruning、Fail-Soft/Hard Mixなど）の導入も行われました。

**パラメータチューニング**には、**SPSA (Simultaneous Perturbation Stochastic Approximation)** が広く用いられ、探索や評価に関する多数の定数を自動で最適化していました。

**テストと検証**には、**SPRT (Sequential Probability Ratio Test)** が用いられ、変更が統計的に有意な改善をもたらすかを確認しながら開発が進められました。**OpenBench** などの分散テスト環境も活用されました。

**各解法の詳細**

**[2位](https://www.kaggle.com/competitions/fide-google-efficiency-chess-ai-challenge/discussion/569891)**

* **アプローチ:** Cfishベース + 小型NNUE。STC特化探索。
* **アーキテクチャ/評価:** NNUE (768(入力)→64(Hidden)→1x8(出力 Buckets))。SCReLU活性化。Piece Count Bucketing。入力704次元 (Pawn/King Mirroring)。量子化(int8/int16)。
* **アルゴリズム:** Stockfish探索 + NNUE評価。
* **テクニック:** 3段階NNUE学習 + SPSAチューニング。STC探索最適化 (ルート奇数plyスキップ, Fail-High処理, QS時間チェック, Sudden Death時間管理)。サイズ最適化 (Clang, 最適化フラグ, 関数属性, libm/lpthread排除)。

**[3位](https://www.kaggle.com/competitions/fide-google-efficiency-chess-ai-challenge/discussion/569874)**

* **アプローチ:** Cfishベース + 小型NNUE。多数の探索改善導入。
* **アーキテクチャ/評価:** NNUE (PSQBB+King/Pawn特徴 -> 8 -> 16 -> 1)。ReLU活性化。Piece Count Bucketing。Dual FT構造。入力特徴量削減 (Pawn/King Mirroring)。
* **アルゴリズム:** Stockfish探索 + NNUE評価。
* **テクニック:** 2段階NNUE学習 (50/50 eval/WDL → Pure WDL)。SPSAチューニング (探索・評価)。探索改善 (Correction History各種, Counter Move Pruning, Fail-Soft/Hard Mix, LMR Deeper, Prior Counter Move Bonusなど)。サイズ最適化 (Clang, 最適化フラグ, 関数属性)。Pondering実装。Pythonラッパー&バイナリ圧縮(xz)。

**[4位](https://www.kaggle.com/competitions/fide-google-efficiency-chess-ai-challenge/discussion/563173)**

* **アプローチ:** Stockfish 16 HCEベース + 小型MLP追加。
* **アーキテクチャ/評価:** HCE + 3層MLP (入力99次元 HCE内部特徴量 -> 14x2次元 -> 32次元 -> 1次元)。Clipped ReLU活性化。
* **アルゴリズム:** Stockfish探索 + HCE/NN評価。
* **テクニック:** NN出力はHCE値に0.5倍して加算。NN学習は自己対局データ、MSE損失。メモリ最適化 (Magic Bitboard代替、libstdc++排除)。サイズ最適化。

**[9位](https://www.kaggle.com/competitions/fide-google-efficiency-chess-ai-challenge/discussion/567106)**

* **アプローチ:** Cfishベース。HCEのみ (NNUE不使用)。探索改善とSPSAチューニング。
* **アーキテクチャ/評価:** HCEのみ。
* **アルゴリズム:** Stockfish探索 + HCE評価。
* **テクニック:** Stockfish 16 HCEの探索手法・パラメータを移植。**SPSAチューニング**。メモリ最適化 (TTサイズ削減, Counter Move History削減, Material/Pawn Tableサイズ削減)。サイズ最適化 (-O3, strip, upx圧縮)。

**[10位](https://www.kaggle.com/competitions/fide-google-efficiency-chess-ai-challenge/discussion/563866)**

* **アプローチ:** Cfishベース。NNUEのみ (HCE無効)。独自NNUEアーキテクチャ。
* **アーキテクチャ/評価:** 独自NNUE (15x15 CNN + 近傍重み共有Dense)。中間層896次元相当？
* **アルゴリズム:** Stockfish探索 + NNUE評価。
* **テクニック:** Leela/StockfishデータでNNUE学習。メモリ最適化 (TTサイズ削減, Continuation Historyハッシュ化)。サイズ最適化 (NNUE部-O3, 他-Os)。
