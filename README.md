# Hallucination Guard

Hallucination Guardはフレームワークであり、単体のツールではない。Detector、Corrector、Pipeline、API、評価といった複数の部品を組み合わせて利用することを前提としており、拡張、比較、差し替えが可能な設計となっている。  
本フレームワークは、ハルシネーションである「ソース文書に基づかない主張」を対象として、まず検知を行い、その結果に基づいて必要に応じて修正を行う。  
この検知→修正の順序が、パイプライン設計の軸である。  
対象はLLMが生成した出力テキストであり、入力となるソース文書自体を修正するものではない。  
そのため、本フレームワークはRAG、要約、QA、チャットなどにおける生成結果の出力チェック係として位置づけられる。

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

---

## 1. 概要（Overview）

Hallucination Guardはフレームワークであり、単体のツールではない。Detector、Corrector、Pipeline、API、評価といった複数の部品を組み合わせて利用することを前提としており、拡張、比較、差し替えが可能な設計となっている。  
本フレームワークは、ハルシネーションである「ソース文書に基づかない主張」を対象として、まず検知を行い、その結果に基づいて必要に応じて修正を行う。  
この検知→修正の順序が、パイプライン設計の軸である。  
対象はLLMが生成した出力テキストであり、入力となるソース文書自体を修正するものではない。  
そのため、本フレームワークはRAG、要約、QA、チャットなどにおける生成結果の出力チェック係として位置づけられる。

---

## 2. パイプライン構成（Pipeline Types）

Hallucination Guardは2つの方式を提供する。

- **Simple**  
  ハルシネーションを検知し、問題があった場合に出力全文をソース文書を使って書き直す方式。全文の書き換えが許容される場合に向く。根拠スパンは返さない。

- **Position-Entailment**  
  出力中の主張スパンを検出し、矛盾または根拠なしを判定し、問題箇所に限定して非破壊的に修正する方式。構造・フォーマット保持が必要な場合に向く。

両方式は `--lang` パラメータ（`en` / `ja`）により英語・日本語に対応する。

Position-Entailment方式は、LLM出力中の主張スパンを検出し、それぞれを矛盾（contradiction）または根拠なし（unsupported）として分類する。  
各主張について、ソース文書中の対応する根拠位置（あるいは根拠が存在しないこと）を明示し、重要度および修正方針（rewrite／delete／ignore）を付与する。  
修正は問題のあるスパンのみに限定して行い、全文は書き換えず、文書の構造やレイアウト、順序を保持する。  
これにより、どの主張がなぜ不適切であるかを説明可能な形で示しつつ、最小限の修正を行う非破壊的な修正方式を実現する。  
本方式は、Simple方式と比較して説明可能性および構造保持性に優れ、英語および日本語の両言語に対応する。  
Simple方式は説明可能性や構造保持を重視しない一方、Position-Entailment方式は問題箇所に限定した修正と根拠提示を行う。

---

## 3. 実行環境・前提条件（Virtual Environment）

We recommend using **uv** for faster installation and managing virtual environments:  
https://docs.astral.sh/uv/getting-started/installation/

Create and activate a venv inside the project folder before installing/running:

```bash
python -m venv .venv # or uv venv --python 3.11
source .venv/bin/activate
