# Hallucination Guard

Hallucination Guard は、LLM 出力に含まれるハルシネーションを検知・修正するためのフレームワークです。現在、次の 2 つのパイプラインを提供します。

- **Simple** – ハルシネーションを検知し、ソース全文を用いて出力全文を書き直します。LLM 呼び出し回数が少ないため高速で、短い文書や全文の書き換えが許容されるケースに適します。根拠スパン（evidence spans）は返しません。
- **Position-Entailment** – 主張スパン（claim spans）を検出し、矛盾（contradiction）/根拠なし（unsupported）を付与します。根拠スパン（evidence spans）、重要度（severity）、推奨アクション（rewrite/delete/ignore）を付与したうえで、フラグ付けされた箇所のみを書き換えます。構造/フォーマットを保持しつつ非破壊的に修正したい場合に適します。

両パイプラインは `--lang` パラメータ（`en` / `ja`）で英語・日本語に対応します。

---
## 目次

1. 概要（Overview）
2. パイプライン構成（Pipeline Types）
3. 実行環境・前提条件（Virtual Environment）
4. インストール方法（Installation）
5. 設定方法（Configuration）
6. 基本的な使い方（Basic Usage）
7. 入出力仕様（Input / Output）
8. ログとエラーハンドリング（Logging）
9. 注意事項・補足（Notes）

## 1. 概要（Overview）

Hallucination Guard は、LLM 出力のハルシネーションを検知・修正するためのフレームワークです。現在は **Simple** と **Position-Entailment** の 2 つのパイプラインを提供します。  
両パイプラインは `--lang` パラメータ（`en` / `ja`）で英語・日本語に対応します。

---
## 2. パイプライン構成（Pipeline Types）

- **Simple**
  - ハルシネーションを検知し、ソース全文を用いて出力全文を書き直します。
  - 高速（LLM 呼び出し回数が少ない）で、短い文書や全文の書き換えが許容されるケースに適します。
  - 根拠スパン（evidence spans）は返しません。

- **Position-Entailment**
  - 主張スパン（claim spans）を検出し、矛盾（contradiction）/根拠なし（unsupported）を付与します。
  - 根拠スパン（evidence spans）、重要度（severity）、推奨アクション（rewrite/delete/ignore）を付与します。
  - フラグ付けされた箇所のみを書き換えるため、構造/フォーマットを保持しつつ非破壊的に修正したい場合に適します。

---
## 3. 実行環境・前提条件（Virtual Environment）

高速なインストールと仮想環境管理のために **uv** の利用を推奨します。  
https://docs.astral.sh/uv/getting-started/installation/

インストール/実行の前に、プロジェクトフォルダ内で venv を作成し、有効化してください。

```bash
python -m venv .venv # or uv venv --python 3.11
source .venv/bin/activate

## 4. インストール方法（Installation）

```bash
# Install from the local directory
pip install -e /path/to/hallucination-guard

# Or install with development dependencies
pip install -e "/path/to/hallucination-guard[dev]"

## 5. 設定方法（Configuration）

### 5.1 環境変数（Environment Variables）

Azure OpenAI の環境変数設定例:

```bash
export AZURE_OPENAI_ENDPOINT="https://your-resource.openai.azure.com/"
export AZURE_OPENAI_API_KEY="your-api-key"
export AZURE_OPENAI_API_VERSION="2025-03-01-preview"
export AZURE_OPENAI_DEPLOYMENT_NAME="gpt-4.1"
```

Ollama の環境変数設定例:

```bash
export OLLAMA_BASE_URL="http://localhost:11434"
export OLLAMA_MODEL="qwen3:8b" # we recommend using at least a 30B model for better results
```

### 5.2 手動設定（Manual Configuration）

```python
from hallucination_guard.config import LLMConfig, ModelProvider

# Azure OpenAI configuration
azure_config = LLMConfig(
    provider=ModelProvider.AZURE_OPENAI,
    model="gpt-4.1",
    azure_endpoint="https://your-resource.openai.azure.com/",
    azure_api_key="your-api-key",
    azure_api_version="2025-03-01-preview",
    temperature=0.3
)

# Ollama configuration
ollama_config = LLMConfig(
    provider=ModelProvider.OLLAMA,
    model="qwen3:8b",
    ollama_base_url="http://localhost:11434",
    temperature=0.3
)
```

### 5.3 混在プロバイダ構成（Mixed Provider Configuration）

（利用可能な場合）タスクごとに異なるプロバイダを利用します。

```python
config = HallucinationGuardConfig.create_mixed_config(
    detection_provider=ModelProvider.AZURE_OPENAI,
    correction_provider=ModelProvider.OLLAMA,
    translation_provider=ModelProvider.AZURE_OPENAI
)
```

---

## 6. 基本的な使い方（Basic Usage）

### 6.1 Python での基本実行例

```python
import asyncio
from hallucination_guard import EntailmentDetector, PositionAwareCorrector
from hallucination_guard.config import HallucinationGuardConfig, ModelProvider

async def main():
    # Configure with Azure OpenAI
    config = HallucinationGuardConfig.from_env(ModelProvider.AZURE_OPENAI)

    # Initialize detector and corrector
    detector = EntailmentDetector(config.detection_llm)
    corrector = PositionAwareCorrector(config.correction_llm)

    # Example texts with multiple hallucinations
    source_text = """
    人工知能（AI）は1950年代に研究が始まりました。
    初期のAI研究は、人間の思考プロセスを模倣することに焦点を当てていました。
    1997年、IBMのディープブルーがチェスの世界チャンピオンを破りました。
    """

    output_text = """
    人工知能（AI）は1970年代に研究が始まりました。
    初期のAI研究は、人間の感情を理解することに焦点を当てていました。
    1997年、GoogleのAlphaGoがチェスの世界チャンピオンを破りました。
    """

    # Detect hallucinations
    result = await detector.detect(source_text, output_text)

    # Print detected hallucinations
    for i, hallucination in enumerate(result.hallucinations, 1):
        print(f"Hallucination {i}: {hallucination.contradiction}")
        print(f"  Evidence: {hallucination.evidence}")

    # Correct if hallucinations found
    if result.hallucinations:
        correction = await corrector.correct(
            source_text=source_text,
            output_text=output_text,
            detections=result.hallucinations
        )
        print(f"Corrected: {correction.corrected_text}")

asyncio.run(main())
```

### 6.2 Example Output（出力例）

`examples/basic_usage.py` を実行すると次のような出力になります。

```bash
=== Example 1: Azure OpenAI ===

Detecting hallucinations...
Found 3 hallucinations
Severity: medium

Hallucination 1:
  Contradiction: 人工知能（AI）は1970年代に研究が始まりました。
  Evidence: 人工知能（AI）は1950年代に研究が始まりました。
  Confidence: 0.8%
  Position: 5-31

Hallucination 2:
  Contradiction: 初期のAI研究は、人間の感情を理解することに焦点を当てていました。
  Evidence: 初期のAI研究は、人間の思考プロセスを模倣することに焦点を当てていました。
  Confidence: 0.8%
  Position: 36-69

Correcting hallucinations...
Corrected text:
    人工知能（AI）は1950年代に研究が始まりました。
    初期のAI研究は、人間の思考プロセスを模倣することに焦点を当てていました。
    1997年、IBMのディープブルーがチェスの世界チャンピオンを破りました。

Applied 3 corrections
  - Detected corrections: 3
  - Additional corrections: 0
```

### 6.3 Position Tracking（位置追跡）

フレームワークはハルシネーションと修正の正確な位置を追跡します。

```python
for hallucination in result.hallucinations:
    if hallucination.position:
        print(f"Found at: {hallucination.position.start}-{hallucination.position.end}")
        print(f"Sentence: {hallucination.position.sentence_idx}")
```

### 6.4 Preservation Modes（保持モード）

strict / natural の修正モードを選択できます。

```python
# Strict mode - minimal changes only
correction = await corrector.correct(
    source_text, output_text, detections,
    preservation_mode="strict"
)

# Natural mode - more flowing corrections
correction = await corrector.correct(
    source_text, output_text, detections,
    preservation_mode="natural"
)
```

---

## 7. 入出力仕様（Input / Output）

### 7.1 REST API（FastAPI）

FastAPI サーバを `src/hallucination_guard/api/` 配下に提供します。エンドポイントは次の 2 つです。

- `GET /health` – readiness info (provider, model)
- `POST /guard` – detect + correct

サーバ起動（LLM 認証情報は上記の環境変数を利用）:

```bash
uvicorn hallucination_guard.api.server:app --host 127.0.0.1 --port 8001
```

API は、source と output の組合せに対して複数の比較設定をサポートします。

#### Request body（例）

**Single pair**
```json
{
  "items": [
    {"id": "ex1", "source": "reference text", "output": "llm output"}
  ],
  "pipeline": "position_entailment",
  "language": "en",
  "preservation_mode": "strict",
  "enable_translation": false
}
```

**One source, many outputs (cartesian)**
```json
{
  "items": [
    {
      "id": "multi-outputs",
      "source": "reference text",
      "outputs": ["output_a", "output_b"],
      "pairing": "cartesian"
    }
  ],
  "pipeline": "position_entailment",
  "language": "en"
}
```

**Many sources, one output (cartesian / RAG)**
```json
{
  "items": [
    {
      "id": "multi-sources",
      "sources": ["doc_a", "doc_b"],
      "output": "answer",
      "pairing": "cartesian"
    }
  ],
  "pipeline": "position_entailment",
  "language": "en"
}
```

**Aligned lists (zip)**
```json
{
  "items": [
    {
      "id": "zip",
      "sources": ["doc_a", "doc_b"],
      "outputs": ["out_a", "out_b"],
      "pairing": "zip"
    }
  ],
  "pipeline": "position_entailment",
  "language": "en"
}
```

#### Response format

```json
{
  "pipeline": "...",
  "language": "...",
  "results": [
    {
      "id": "...",
      "pair_results": [
        {
          "source": "...",
          "output": "...",
          "detection": {
            "hallucinations": [...],
            "raw_output": "...",
            "has_hallucinations": true,
            "severity": "high",
            "score": 0.95
          },
          "correction": {
            "corrected_text": "...",
            "corrections": [...],
            "diff_metrics": {...}
          },
          "error": null,
          "processing_time_ms": 1234.5
        }
      ],
      "error": null,
      "processing_time_ms": 1234.5
    }
  ]
}
```

**Note**: API クライアントは `results[*].pair_results` を参照して detections / corrections を取得してください。

### 7.2 Example Python client

`examples/api_client.py` は benchmark JSONL からサンプル payload を実行します。シナリオと言語を選び、必要に応じて出力を保存できます。

```bash
source .venv/bin/activate
# single source/output (English)
uv run python examples/api_client.py --scenario single --language en --data-path /path/to/benchmark_en.jsonl --api-url http://127.0.0.1:8001/guard --output tmp/resp_single.json
# single source/output (Japanese)
uv run python examples/api_client.py --scenario single --language ja --data-path /path/to/benchmark_ja.jsonl --api-url http://127.0.0.1:8001/guard --output tmp/resp_single_ja.json
# one source, many outputs (cartesian)
uv run python examples/api_client.py --scenario multi-outputs --language en --data-path /path/to/benchmark_en.jsonl --api-url http://127.0.0.1:8001/guard --output tmp/resp_multi_outputs.json
# many sources, one output (cartesian)
uv run python examples/api_client.py --scenario multi-sources --language en --data-path /path/to/benchmark_en.jsonl --api-url http://127.0.0.1:8001/guard --output tmp/resp_multi_sources.json
# aligned pairs (zip)
uv run python examples/api_client.py --scenario zip --language en --data-path /path/to/benchmark_en.jsonl --api-url http://127.0.0.1:8001/guard --output tmp/resp_zip.json
```

保存されるファイルは `GuardResponse` を含む JSON です。`results[*].pair_results` を確認して detections / corrections を参照してください。

---

## 8. ログとエラーハンドリング（Logging）

原文 README の範囲では、ログ出力仕様（ログ形式やログ出力場所など）の詳細は記載されていません。  
REST API のレスポンスには `error` と `processing_time_ms` が含まれるため、API 利用時はこれらのフィールドを確認してください。

---

## 9. 注意事項・補足（Notes）

### 9.1 Evaluation & Reports（評価・レポート）

`run_evaluation.py` を使用して JSON + HTML の比較レポートを生成できます。

```bash
source .venv/bin/activate
uv run python run_evaluation.py \
  --dataset ../benchmarks-hallucination/extracted_datasets/unified_hallucination_corrections.jsonl \
  --samples 5 \
  --pipeline position_entailment \
  --compare-with simple \
  --metrics llm,change \
  --enable-llm-judge \
  --html-report tmp_compare_final_5.html \
  --correction-mode natural \
  --output tmp_compare_final_5.json
```

フラグの説明:
- `--pipeline`: 主に評価するパイプライン（例: position_entailment）
- `--compare-with`: 比較対象のパイプライン（例: simple）
- `--metrics`: 指標（`llm`, `change` など）をカンマ区切りで指定
- `--enable-llm-judge`: 指標の一部として LLM judging を有効化
- `--correction-mode`: `strict` または `natural`
- `--samples`: dataset からサンプルする件数
- `--html-report`: HTML 比較レポートの出力先
- `--output`: 生の JSON 結果の出力先

### 9.2 Evaluation Metrics（評価指標）

- **Standard**（ベンチマークの正解データとの重なりを測定）: BLEU, ROUGE, BERTScore
- **Change**（修正結果と元出力の距離を測定）: Edit Distance, Length Ratio, Text Edit Ratio (TER)
- **LLM-as-a-Judge**: source への faithfulness、修正後出力の usability、2つのアプローチの比較






