---
tags:
  - Kaggle
url: https://www.kaggle.com/competitions/fide-google-efficiency-chess-ai-challenge
startdate: 2024-09-13
enddate: 2024-12-13
---
**全体的な傾向:**

このコンペでは、非常に限られたリソース（サイズ、メモリ、時間）の中で、強いチェスエンジンを開発することが課題です。上位解法では、既存の強力なオープンソースエンジン（Stockfish、Cfish）をベースとし、リソース制約を満たすように軽量化するアプローチが主流です。評価関数（NNUEまたはHCE）、探索アルゴリズムの最適化、パラメータチューニング（SPSA）、バイナリサイズの圧縮などが重要なテクニックとして活用されています。

**各解法の詳細:**

**2位 (Approvers)**

- **アプローチ:** StockfishのC移植版（Cfish）をベースに、サイズ、メモリ、時間制限、オープニングブックに合わせて最適化。ドメイン知識と一般的な開発スキルを組み合わせる。
- **アーキテクチャ:** NNUE（ニューラルネットワーク評価関数）を導入。シンプルなNNUEアーキテクチャ（768入力 -> 64隠れ層 -> 8出力）。
- **アルゴリズム:** 探索アルゴリズム（Iterative Deepening、Quiescence Searchなど）をSTC（Short Time Control）向けに最適化。SPRT（Sequential Probability Ratio Test）とSPSA（Simultaneous Perturbation Stochastic Approximation）によるテストとチューニング。
- **テクニック:**
    - **探索機能:** STC Elo Gainer Optimization、STC Fail-High Handling、Quiescence Search Time Checking、Sudden Death Time Control Optimizationなど、短時間思考向けの最適化を導入。VVLTC（Very Long Long Time Control）向けの最適化も一部組み込み。
    - **評価:** シンプルなNNUEアーキテクチャを採用。3段階の段階的学習とSPSAによる最終調整。ネットワークは8bit/16bitに量子化。
    - **サイズ最適化:** 不要な機能の削除、`gcc` から `clang` への切り替え、コンパイラフラグの調整、`#pragma` ディレクティブ、関数属性（`minsize`, `cold`, `section(".text.small")`）の適用、`libm` と `lpthread` への依存性をカスタム実装で排除し、シングルスレッド化。
    - **テスト:** SPRTを使用して変更の有効性を統計的に評価。SPSAを使用してパラメータをチューニング。OpenBenchによる分散コンピューティングで約20Mゲームを実行。

**3位 ("Fix the bugs?")**

- **アプローチ:** StockfishのC移植版（Cfish）をベースに、メモリとサイズの制約を満たすように最小化。NNUE評価関数を設計・学習。探索アルゴリズムの改善。SPSAによるチューニング。
- **アーキテクチャ:** NNUE評価関数。シンプルなフィードフォワードニューラルネットワーク。
- **アルゴリズム:** 探索アルゴリズム（AlphaBeta pruning、fail-soft/fail-hardの混合、Late Move Reductions Deeperなど）を改善。SPRTとSPSAによるテストとチューニング。
- **テクニック:**
    - **メモリとサイズの最小化:** 不要な機能の徹底的な削除（tablebases, opening books, NNUE実装の初期版, マルチスレッド, etc.）。Kaggleビルド専用のプリプロセッサ定義。エンジン入出力の最小化。`clang` と特定のコンパイラフラグを使用してバイナリサイズを削減。非クリティカルな関数に属性を付与。Pythonラッパーの難読化と圧縮。
    - **エンジン改善:** Pawn Correction History、Non-Pawn Correction History、Minor/Major Correction History、Counter Correction History、Counter Move Pruningの改善、fail-softとfail-hardの混合、LMRの改善、Prior Counter Move Bonus、Altair Fail-Soft Multicutなどのパッチを適用。
    - **NNUE:** Grapheusツールを使用して、Etherealエンジンによる自己対局データで学習。768入力（キングミラーリング、ポーンの非占有マス除去）のシンプルなアーキテクチャ。バッチ勾配降下法とADAMオプティマイザを使用。2段階の学習（50/50 eval/WDL、Pure WDL）。int8量子化と重み転置などの圧縮トリック。

**4位 (Lgeu)**

- **アプローチ:** Stockfish 16をベースに、HCE（手作り評価関数）に小さなニューラルネットワーク（NNUEではない）を追加してEloレーティングを向上。
- **アーキテクチャ:** 3層のMLP（多層パーセプトロン）。
- **アルゴリズム:** Stockfish 16の探索アルゴリズム。
- **テクニック:**
    - **ベースエンジンの選択:** Stockfish 16（HCEをサポートする最後のバージョン）を選択。
    - **RAM使用量:** ポーンハッシュテーブル、Continuation History、Transposition Tableのサイズを削減。不要なモジュールを削除。ルークのMagic BitboardをCFishの古典的なアプローチに置き換え。libstdc++への依存性を排除。
    - **評価関数の改善:** HCEを拡張する3層MLPを追加。入力特徴量は99個。NNUEのClipped ReLUを使用。MSE損失関数で学習。QAT（Quantization Aware Training）を試行。
    - **トレーニング:** `kaggle-environments` で約7万ゲームを実行し、探索中に遭遇した局面を確率的に保存してトレーニングデータとして使用。HCEベース、NNUEベース、実験的な評価関数を持つエージェントを使用。
    - **提出戦略:** HCEのみのエージェントとNN強化エージェントの両方を提出。

**9位 (ymgaq)**

- **アプローチ:** StockfishのC移植版（Cfish）をベースに、NNUEを削除してHCEのみを使用。メモリ最適化、バイナリサイズ圧縮、探索の強化、SPSAによるパラメータチューニング。
- **アーキテクチャ:** HCE（手作り評価関数）のみを使用。
- **アルゴリズム:** Stockfish16のHCE探索アルゴリズムを参考に修正。SPSA（Simultaneous Perturbation Stochastic Approximation）によるパラメータチューニング。
- **テクニック:**
    - **ベースプログラムの選択:** Cfishを選択（glibcによる低メモリ消費、Kaggleのシングルコア環境での探索速度）。
    - **メモリ最適化:** NNUEを削除。Transposition Table、Counter Move History、Material Table、Pawn Hash Tableのサイズを削減。
    - **バイナリサイズ圧縮:** 不要な機能やUCIオプションを削除。コンパイルコマンドの最適化（`-ffunction-sections -fdata-sections -Wl,--gc-sections`）。`strip` コマンドを使用。`upx --lzma` でバイナリを圧縮。
    - **探索の強化:** Cfishの探索をStockfish16のHCE探索メソッドとパラメータに近づけるように修正。O3最適化を使用。
    - **SPSA:** シンプルなカスタムSPSAスクリプト（cutechess-cliを使用）をローカルマシンで実行し、パラメータをチューニング。

**10位 (Niboshi)**

- **アプローチ:** Stockfish 4、Stockfish 16を経て、RAM制約のため最終的にCfishを使用。HCEを無効化し、NNUEのみに依存。バイナリサイズ効率の良いネットワークを探索。
- **アーキテクチャ:** CNNとDenseネットワークを組み合わせたカスタムNNUEネットワーク。
- **アルゴリズム:** NNUE評価関数。
- **テクニック:**
    - **ベースプログラムの選択:** RAM制約のためCfishを使用。
    - **NNUE:** HCEを無効化し、NNUEのみを使用。バイナリサイズ効率の良いネットワークを設計（CNNと隣接位置で重みを共有するDenseネットワークの組み合わせ）。学習データはLeelaとStockfishの生成データの混合。
    - **その他の変更:** Transposition tableとContinuation Historyにハッシュテーブルを使用しサイズを削減。コンパイルオプションの調整（nnue.cは-O3、その他は-Os）。未使用の機能などを削除。