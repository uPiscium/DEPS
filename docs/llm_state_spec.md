# LLMへ与えるエージェント状態(State)の実装仕様

このドキュメントは、現行実装で「エージェントの現在状態」をLLMへ渡す形式を、コード実装どおりに抽出した仕様書です。

対象コード:

- `planner.py`
- `main.py`
- `data/task_prompt.txt`
- `data/deps_prompt.txt`
- `data/parse_prompt.txt`

## 1. 結論: StateはJSONスキーマではなく、対話テキストとして渡している

計画生成・再計画・説明生成でLLMに渡す状態は、`Planner.dialogue` に蓄積された**プレーンテキスト対話履歴**です。

- 送信API: `openai.Completion.create(...)`
- モデル:
  - 計画/再計画/説明: `code-davinci-002` (`query_codex`)
  - 行パース: `text-davinci-003` (`query_gpt3`)
- 入力フィールド: いずれも `prompt=<string>`

つまり、計画系の状態入力において、JSON Schemaバリデーション付きの構造化入出力は使っていません。

## 1.1 システムが想定するI/Oプロトコル（実装準拠）

本システムのLLM連携は、チャットAPIではなく **Completions APIの単発リクエスト/レスポンス** を前提にしています。

- 通信単位: 1回の `openai.Completion.create(...)` 呼び出し
- 入力チャネル: 単一文字列 `prompt`
- 出力チャネル: `response["choices"][0]["text"]`
- 文脈保持: サーバ側セッションではなく、クライアント側で `Planner.dialogue` を毎回再送

言い換えると、プロトコルは「`prompt` に履歴全文を詰めて送る -> `choices[0].text` を次状態にappendする」の反復です。

## 1.2 APIレベルの入力パラメータ仕様

### 計画/説明/再計画 (`query_codex`)

```json
{
  "model": "code-davinci-002",
  "prompt": "<dialogue text>",
  "temperature": 0.7,
  "max_tokens": 1024,
  "top_p": 1,
  "frequency_penalty": 0,
  "presence_penalty": 0,
  "stop": ["Human:"]
}
```

### 行パース (`query_gpt3`)

```json
{
  "model": "text-davinci-003",
  "prompt": "<parse few-shot + input line>",
  "temperature": 0,
  "max_tokens": 256,
  "top_p": 1,
  "frequency_penalty": 0,
  "presence_penalty": 0
}
```

備考:

- 行パース側のみ `prompt_text[-4000:]` で入力を後方切り詰め
- どちらも例外時は最大10回までリトライ
- 各リクエスト前に `update_key()` でAPIキーをローテーション

## 1.3 アプリケーション層プロトコル（対話ターン規約）

LLMに渡す文字列は、`Human:` / `AI:` プレフィックス付きの対話として運用されます。

- few-shot例 (`data/deps_prompt.txt`) は `Human:` と `AI:` の交互ターン
- 実行時イベントは `Human:` 行として追記
- LLM返答は `AI:` プレフィックスを含む/含まない両方があり得るが、実装は生テキストをそのまま受理

最小プロトコル（擬似BNF）:

```text
dialogue := seed_context task_query ai_plan event*
seed_context := <task_prompt> <deps_prompt>
task_query := "Human: " <task_question> "\n"
ai_plan := <free_text_from_llm>
event := success | failure | inventory | explanation | replan_request | replan_result
```

`event` のうち `explanation` と `replan_result` は、LLM応答テキストそのものです。

## 2. 計画LLMに渡すStateテキスト仕様

### 2.1 初期計画時の入力

`Planner.initial_planning(group, task_question)` は次を連結して送信します。

1. `data/task_prompt.txt` の全文
2. `data/deps_prompt.txt` の全文
3. `Human: {task_question}\n`

形式:

```text
<task_prompt.txt>
<deps_prompt.txt>
Human: <task_question>\n
```

この文字列が `query_codex(prompt_text)` にそのまま渡されます。

### 2.2 実行中に状態として蓄積される対話

初期計画後、`Planner.dialogue` は以下で初期化されます。

```text
<task_prompt.txt><deps_prompt.txt>Human: <task_question>\n<initial_plan_text>
```

以降、イベントごとに追記され、次回LLM呼び出し時の現在状態となります。

追記規則（実装どおり）:

- 成功報告:
  - `Human: I succeed on step {step}.\n`
- 失敗報告:
  - `Human: I fail on step {step}`
  - 注: 実装上は末尾改行なし
- 在庫報告:
  - `Human: My inventory now has {q1} {name1}, {q2} {name2}, ... , \n`
  - `diamond_axe` と `air` は除外
- 再計画要求:
  - `Human: Please fix above errors and replan the task '{task_question}'.\n`

### 2.3 失敗時のLLM呼び出しシーケンス

`Evaluator.replan_task(...)` では、以下順で状態テキストが更新/送信されます。

1. `generate_failure_description(step)`
   - `Human: I fail on step {step}` を追記
   - その時点の `dialogue` を `query_codex(...)` に送信
   - 返答テキスト（失敗理由説明）を追記
2. `generate_inventory_description(inventory)`
   - 在庫文を追記（送信はしない）
3. `generate_explanation()`
   - 更新後 `dialogue` を `query_codex(...)` に送信
   - 返答テキストを追記
4. `replan(task_question)`
   - 再計画要求文を追記
   - 更新後 `dialogue` を `query_codex(...)` に送信
   - 返答テキスト（新計画）を追記

したがって「現在状態(State)」は、上記履歴を順次appendした単一文字列です。

## 3. 行パースLLMに渡す仕様（計画テキスト->goal辞書）

`generate_goal_list(plan)` は、`#` を含む計画行ごとに `online_parser(...)` を呼びます。

`online_parser(text)` のLLM入力:

1. `data/parse_prompt.txt` のfew-shot全文
2. `text`（呼び出し側で `input: {line}` を渡す）

最終入力:

```text
<parse_prompt.txt>
input: <plan_line>
```

`query_gpt3(...)` では送信前に `prompt_text = prompt_text[-4000:]` を適用します。

## 4. パーサが期待する出力フォーマット（擬似スキーマ）

`online_parser` は出力を行単位で処理し、空白除去後に以下キーだけを使用します。

- `action:`
- `name:`
- `object:`
- `rank:`

`tool:` や `materials:` が出てきても無視されます。

期待フォーマット（行ベース）:

```text
name: <string>
action: <string>
object: <python_dict_literal>
rank: <int>
```

型変換仕様:

- `object` は `eval(line[7:])` でPython辞書として評価
- `rank` は `int(line[5:])`

## 5. 実際に内部で使うgoal構造（LLM出力の変換先）

パース結果は `goal_lib.json` を参照して次の辞書に正規化されます。

```json
{
  "name": "<goal_name>",
  "type": "mine|craft|smelt",
  "object": {"<item>": <int>},
  "precondition": {"<item>": <int>},
  "ranking": <int>
}
```

`precondition` は `goal_lib[name]["precondition"]` と `goal_lib[name]["tool"]` のマージ結果です。

## 6. 参考: Conceptual JSON Schema（実装上は未使用）

以下は現行テキスト運用を機械可読化した場合の**対応スキーマ**であり、実コードで直接送ってはいません。

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "PlannerDialogueState",
  "type": "object",
  "required": ["dialogue"],
  "properties": {
    "dialogue": {
      "type": "string",
      "description": "task/deps prompt + Human/AI turns concatenated as plain text"
    },
    "latest_event": {
      "type": "string",
      "enum": ["initial_planning", "success", "failure", "inventory", "explanation", "replan"]
    }
  },
  "additionalProperties": false
}
```

## 7. 実装上の注意点（仕様として観測される挙動）

- 失敗文は末尾改行なしのため、次トークンと連結される可能性がある
- 在庫文は末尾に `", "` が残る形式で送られる
- `query_codex` は `stop=["Human:"]` を指定
- `query_gpt3` は温度0、`max_tokens=256`
- `query_codex` は温度0.7、`max_tokens=1024`
