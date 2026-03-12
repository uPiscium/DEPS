# DEPS Core Features

このドキュメントは、DEPS リポジトリのコア機能を一覧化し、
各機能の **役割（何を担うか）**、**実装方法（どう作られているか）**、**目的（なぜ必要か）** を整理したものです。

## 0. 全体像

DEPS は、Minecraft 系タスクに対して以下を統合する実行基盤です。

- LLM による高レベル計画（何をするか）
- ゴール条件付き方策モデルによる行動生成（どう動くか）
- craft/smelt の手続き制御
- 実行中の再計画とログ可視化

主な実行起点は `main.py` で、Hydra 設定 (`configs/`) を通じて挙動を切り替えます。

---

## 1. 評価オーケストレーション（`main.py` / `Evaluator`）

### 役割

- タスク評価の全体フローを統括する。
- 環境初期化、計画初期化、ステップ実行、成功判定、ログ出力を一元管理する。

### 実装方法

- `main.py` の `main()` が Hydra 設定を受け取り `Evaluator` を生成。
- `Evaluator.__init__()` で環境 (`MineDojoEnv`)、Planner、Mine/Craft エージェントを初期化。
- `single_task_evaluate()` が反復評価（成功率と平均ステップ長集計）を実行。
- タスク情報は `data/task_info.json` から読み込み。

### 目的

- 「1コマンドで評価実行できる」再現可能な評価基盤を提供する。
- 設定変更（Hydra override）だけで比較実験できるようにする。

---

## 2. LLM 計画生成・再計画（`planner.py`）

### 役割

- タスク自然言語から実行プランを生成する。
- 実行失敗時に、状況説明を踏まえた再計画を行う。

### 実装方法

- `Planner.initial_planning()` が初期プランを生成。
- `Planner.replan()` が失敗後の修正プランを生成。
- `Planner.generate_failure_description()` / `generate_explanation()` が対話履歴を拡張。
- プロンプトは `data/task_prompt.txt`、`data/deps_prompt.txt`、`data/parse_prompt.txt` を使用。
- OpenAI API キーは `data/openai_keys.txt` を読み、`update_key()` でローテーション。

### 目的

- 固定手順で詰まりやすい長期タスクに対して、動的な計画修正能力を与える。

---

## 3. ゴール構造化と辞書マッピング（`goal_lib.json` / `goal_mapping.json`）

### 役割

- LLM のテキストプランを実行可能なゴール辞書へ変換する。
- 各ゴールを MineCLIP/CLIP/horizon 用の語彙にマッピングする。

### 実装方法

- `Planner.generate_goal_list()` が計画テキストを 1 行ずつ解析しゴール化。
- `data/goal_lib.json` が `type`, `precondition`, `tool`, `output` を定義。
- `check_object()` で未登録 goal 名でも output object から補完可能。
- `main.py` 側で `data/goal_mapping.json` を読み、埋め込み参照用語へ変換。

### 目的

- 自然言語と制御コードのギャップを埋め、計画を安定して実行段へ落とし込む。

---

## 4. 階層制御ループ（mine / craft / smelt 切替）

### 役割

- ゴールタイプごとに最適な実行器へ処理を分岐する。
- タスク進捗に応じてゴール遷移・再計画を制御する。

### 実装方法

- `Evaluator.eval_step()` で `curr_goal["type"]` を判定して分岐。
- `mine`: `MineAgentWrapper.get_action()` を使用。
- `craft` / `smelt`: `CraftAgent.get_action()` のジェネレータを使用。
- `check_inventory()` / `check_precondition()` / `check_done()` で状態判定。
- 失敗条件（前提欠如・時間超過）で `replan_task()` を発火。

### 目的

- ニューラル方策とルールベース手続きの長所を統合し、複合タスクを安定して完遂する。

---

## 5. Mine 行動ポリシー（`src/models/simple.py`）

### 役割

- 視覚観測 + ゴール + 補助観測から行動分布を予測する中核モデル。

### 実装方法

- `SimpleNetwork` が policy 本体。
- 画像特徴はバックボーン（Impala / GoalImpala / MineCLIP / ResNet）から取得。
- ゴール埋め込みは `nn.Linear` で表現。
- 融合方式は `rgb`, `concat`, `bilinear`, `film` を選択可能。
- 追加観測として `ExtraObsEmbedding`（biome, compass, gps, voxels）を利用可能。
- `use_prev_action` で前行動埋め込みを追加可能。
- 時系列は GRU か Transformer (`transformers.GPT2Model`) を選択可能。
- `use_horizon`/`use_pred_horizon` で horizon 予測ヘッドを有効化可能。

### 目的

- 目標条件付きで時間的文脈を持つ行動選択を実現し、採掘行動の性能を高める。

---

## 6. 視覚バックボーンとゴール埋め込み（`controller.py` / `src/utils/vision.py`）

### 役割

- テキストゴールを埋め込み化し、視覚認識系と結合してゴール条件付き知覚を作る。

### 実装方法

- `controller.py` の `accquire_goal_embeddings()` が MineCLIP で goal text をエンコード。
- `src/utils/vision.py` の `create_backbone()` が設定名でモデルを生成。
- `resize_image()` で入力解像度をモデル要件に合わせる。
- `MineCLIPWrapper`, `ImpalaCNNWrapper`, `GoalImpalaCNNWrapper`, `ResNetWrapper` を提供。

### 目的

- 何を達成すべきか（goal semantics）を視覚行動モデルへ直接注入し、目的志向性を高める。

---

## 7. Craft/Smelt スクリプトエージェント（`controller.py` / `CraftAgent`）

### 役割

- crafting table や furnace を伴う手続きを確実に実行する。
- 採掘系モデルが不得意な離散手順を補完する。

### 実装方法

- `CraftAgent` が action generator 群を提供（`craft_w_table`, `smelt_w_furnace`, `equip` など）。
- `get_action()` が内部 iterator の状態を保持し、段階的に行動を返す。
- 在庫確認 (`index_slot`) と視点制御 (`look_to`) を組み合わせて操作。

### 目的

- 手順依存の高いクラフト処理を安定化し、全体成功率を底上げする。

---

## 8. 進行管理と再計画トリガ（`Evaluator`）

### 役割

- 現在ゴールの達成可否を監視し、次ゴールへ遷移させる。
- 行き詰まり時に再計画を自動発火する。

### 実装方法

- `update_goal()` がインベントリ達成時にゴールを前進。
- `replan_task()` が失敗説明 + 在庫説明 + explanation を経由して新計画を生成。
- ゴール種別ごとに時間閾値（craft/smelt）を超えると再計画。
- 再計画回数の上限を超えた場合は打ち切り。

### 目的

- 1 回の計画失敗で全体失敗とならない実行ロバスト性を確保する。

---

## 9. ログ記録と GIF 可視化（`main.py`）

### 役割

- 実行時の計画・ゴール・対話・結果を追跡可能にする。
- 成功/失敗の事後解析を容易にする。

### 実装方法

- `Evaluator.logging()` でステップごとの状態を `self.logs` に保存。
- `logs/<timestamp>_<task>.json` に逐次書き込み。
- `record.frames=True` の場合、RGB フレームを蓄積し GIF 化。
- `recordings/<timestamp>/<task>.gif` と対応 JSON を保存。

### 目的

- 再現性・分析性・デバッグ性を高め、実験サイクルを短縮する。

---

## 10. Hydra 設定駆動の実験管理（`configs/`）

### 役割

- データ条件・モデル条件・評価条件を宣言的に管理する。
- CLI から高速に条件切替できるようにする。

### 実装方法

- `configs/defaults.yaml` が `data/model/eval/goal_model` の既定を合成。
- `configs/goal_model/*.yaml` で goal_model のモードを切替。
- 実行時に `python main.py key=value` で override。

### 目的

- コード変更なしで比較実験やアブレーションを実行できる研究運用性を実現する。

---

## 11. データ処理ユーティリティ（`src/utils`）

### 役割

- 学習/評価データの整理、変換、統計、可視化を補助する。

### 実装方法

- `group.py`: accomplishments ベースで trajectory をゴール別ディレクトリに分割。
- `json2lmdb.py`: JSON trajectory を LMDB へ並列変換（`ReaderWorker`）。
- `statistic.py`: データ件数統計を算出。
- `visualize_trajs.py`: trajectory を MP4 で可視化。

### 目的

- データ基盤整備を自動化し、学習前処理と分析を効率化する。

---

## 12. 未実装インターフェース（`selector.py`）

### 役割

- 将来的な候補ゴール選択ロジックの差し込み点を提供する。

### 実装方法

- `Selector` クラスに `check_precondition`, `generate_candidate_goal_list`, `horizon_select` を定義。
- 現状はすべて `NotImplementedError`。

### 目的

- 現在の計画実行器に対して、将来の高度な goal selector を拡張しやすくする。

---

## 機能間の実行フロー

1. `main.py` が Hydra 設定を読み、`Evaluator` を初期化。
2. `Planner` がタスク文から初期計画を生成し、ゴール列へ構造化。
3. `Evaluator.eval_step()` がゴール種別に応じて mine/craft/smelt を切替実行。
4. 在庫・前提条件を監視し、必要時に再計画。
5. 結果を JSON/GIF に保存し、成功率とエピソード長を集計。

---

## この設計の狙い（要約）

- LLM による柔軟な計画能力と、方策モデル/スクリプト制御の実行確実性を組み合わせる。
- 実行中に再計画可能なループを持たせ、オープンワールド環境での失敗耐性を上げる。
- Hydra とログ基盤で研究開発に必要な再現性・比較可能性を担保する。
