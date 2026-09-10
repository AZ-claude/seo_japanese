# DESIGN — seo_japanese V1

## Objective

任意の日本語テキストを受け取り、元の意味・事実を維持したまま、AI生成文にありがちな機械的・テンプレ的な表現を減らす。

AI detector回避は目的にしない。

## Public API

```python
result = humanize(
    text: str,
    facts: str | dict | None = None,
    language: str = "ja",
    content_type: str = "article",
    style_profile: dict | None = None,
) -> HumanizeResult
```

```python
@dataclass
class HumanizeResult:
    text: str
    patterns_before: list[str]
    patterns_after: list[str]
    changes: list[dict]
    claim_preservation: dict
    naturalness_scores: dict
    rewrite_count: int
    warnings: list[str]
```

## CLI

```bash
seo-japanese rewrite article.md
seo-japanese evaluate article.md
seo-japanese rewrite article.md --facts source.json
```

File modeでは本文 proseだけを変更し、次を不変にする。

- code block
- inline code
- commands
- paths
- frontmatter / YAML metadata
- URLs / link targets
- structured data

## Components

### `src/pipeline/`

全体orchestration。

責務:

- input normalization
- analyzer → writer → claim checker → criticの呼び出し
- rewrite回数制御
- result metadata生成

### `src/claims/`

事実保持。

最低限、以下のclaim typeを追跡する。

- named entities
- numbers
- dates
- monetary values
- quotes
- citations
- URLs
- rankings / comparisons
- causality / simultaneity where materially stated

判定:

- preserved
- modified
- invented
- dropped

V1 hard gate:

- invented material fact = 0
- materially modified fact = 0

### `src/evaluation/`

Naturalness critic。

structured schemaで以下を返す。

- naturalness
- specificity
- rhythm
- vocabulary
- voice_consistency
- cultural_fit
- non_template_feel
- remaining_patterns
- needs_rewrite
- reasons

### `rules/base/`

`blader/humanizer` の抽象patternを参照する基礎ルール。

英語の語彙リストをそのまま日本語へ翻訳しない。構造的な癖として抽象化する。

### `rules/ja/`

日本語特有のpattern。

各patternは最低限次のfieldを持つ。

```yaml
id: JA-01
name: 定型イントロ
strength: strong
signals:
  - "について解説していきます"
  - "詳しく見ていきましょう"
problem: "本文より先に説明行為を宣言する"
action: "前置きを削除して最初の具体的claimから始める"
false_positive_guard: "章のナビゲーション自体が読者に必要な場合は残す"
```

## Pipeline

### Pass 0 — Normalize

入力を以下へ分離。

```text
prose
immutable regions
facts/source
content type
optional style profile
```

### Pass 1 — Analyze

base + Japanese taxonomyでpatternを検出。

出力例:

```json
{
  "patterns": [
    {"id": "JA-01", "span": "...", "confidence": 0.95},
    {"id": "BASE-06", "span": "...", "confidence": 0.82}
  ]
}
```

### Pass 2 — Rewrite

ルール:

- supported claimを保持する
- 元の段落構造を聖域にしない
- 不要な文は削除可能
- fact / name / number / date / quote / citationを追加しない
- missing detailを自然さのために捏造しない
- style profileがなければcontent typeに適した自然な日本語を使う

### Pass 3 — Claim verification

source/factsがある場合は最優先でそれと比較。

factsが明示されていない場合でも、original textに存在した重要claimの保持を検査する。

FAIL時だけrewrite候補へ戻す。

### Pass 4 — Critic

writerと別モデルclient / prompt interfaceを使う。

criticは評価のみ行い、文章を直接変更しない。

### Pass 5 — Optional rewrite

以下の場合のみ1回実施。

- factuality gateはPASS
- criticが`needs_rewrite=true`
- remaining strong patternsがある

rewrite後はclaim verificationを再実行する。

V1では`max_rewrites = 1`固定。

### Pass 6 — Final output

最終結果とtelemetry metadataを返す。

## Initial Japanese taxonomy

| ID | Pattern |
|---|---|
| JA-01 | 「〜について解説していきます」型イントロ |
| JA-02 | 「重要です」「ポイントです」の自己宣言 |
| JA-03 | 「〜と言えるでしょう」の過剰使用 |
| JA-04 | 「〜することができます」の反復 |
| JA-05 | 段落末テンプレの反復 |
| JA-06 | また/さらに/一方で の機械的接続 |
| JA-07 | 文長・リズムの均一化 |
| JA-08 | 強制三点列挙 |
| JA-09 | section構造の過剰反復 |
| JA-10 | 読者に不要な一般論 |
| JA-11 | 過剰な名詞化 |
| JA-12 | 冗長敬語 |
| JA-13 | 根拠のない「近年注目」型一般化 |
| JA-14 | 不要な総括・締め |
| JA-15 | heading直後の内容反復 |

初期taxonomyは仮説。golden fixtureと実運用結果から更新する。

## Test strategy

### Unit

各patternについてpositive / negative fixtureを用意する。

false positive guardも必須。

### Regression

最低30件の日本語AI文fixtureから開始する。

各fixtureは可能なら以下を持つ。

```yaml
input: ...
facts: ...
expected_preserved_claims: ...
forbidden_new_claims: ...
expected_patterns_before: ...
max_patterns_after: ...
```

### Acceptance assertions

- rewrite後のstrong pattern数が減る
- new material factが0
- names / numbers / dates / citationsが破壊されない
- code / URL / frontmatterが変更されない
- max rewrite countが1を超えない

## Model abstraction

providerを固定しない。

```python
class ModelClient(Protocol):
    def generate(self, *, system: str, user: str, config: GenerationConfig) -> str:
        ...
```

writer / critic / claim checkerはinterface上分離する。

同じproviderを使ってもよいが、同じpromptに役割を混在させない。

## V1 repository layout

```text
src/
  pipeline/
  claims/
  evaluation/
  adapters/
skills/
  humanizer/
  ja-humanizer/
rules/
  base/
  ja/
evalsets/
  golden/
tests/
  unit/
  regression/
docs/
```

## Future, not V1

- Human blind evaluation
- MAUVE / corpus distribution monitoring
- edit-diff dataset
- SFT / LoRA
- DPO
- vLLM
- Langfuse / OpenTelemetry based observability
- detector red-team

これらはV1の評価データが集まってから判断する。
