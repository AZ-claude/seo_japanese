# CODEX GOAL — V1 implementation handoff

以下をCodexへそのまま渡して実装開始できるようにする。

```text
新規PJ seo_japanese のV1を実装する。

目的:
AI生成文から機械的・テンプレ的な文章パターンを除去し、意味・事実を維持したまま自然な日本語へ変換する provider-independent な文章編集pipelineを作る。

North Star:
AI detector回避ではない。日本語母語話者にとって自然・具体的・文脈適合した文章を作る。factuality / meaning preservationはnaturalnessより上位のhard gateとする。

参照資料:
- README.md
- AGENTS.md
- docs/HANDOFF.md
- docs/DESIGN.md
- docs/RESEARCH_NOTES.md
- docs/UPSTREAM.md
- docs/source/README.md

元調査PDF:
「AI生成文を『不自然なAI文』から脱却させるための設計・評価・運用戦略」
今回のGitHub自動ハンドオフではバイナリPDF本体は未コミットだが、V1実装に必要な判断事項は上記docsへ抽出済み。PDFが後からdocs/source/に追加された場合は補助原典として読むこと。

最初に必ず:
1. 上記docsを全て読む。
2. blader/humanizer の最新版をGitHubで調査する。
3. README.md / SKILL.md / AGENTS.md / LICENSE / scripts を確認する。
4. upstreamの内容を盲目的にコピーせず、抽象pattern・workflow・fact preservation思想と、日本語固有realizationを分離して設計する。

方針:
- YAGNI。
- V1ではfine-tuning、SFT、DPO、RLHF、vLLM、Langfuse、MAUVEを実装しない。
- AI detector scoreをKPIやrewardにしない。
- writer / critic / claim checkerを論理的に分離する。
- rewriteは最大1回。
- 生成モデルproviderを固定しない。
- 自分の文章サンプルを必須にしない。style_profile / voice sampleはoptional。
- upstreamを直接大改造して保守不能なforkにしない。
- destructive / production-sensitive action以外は実装完了まで自動継続する。

想定project structure:

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

Public Python API:

humanize(
    text,
    facts=None,
    language="ja",
    content_type="article",
    style_profile=None
) -> HumanizeResult

HumanizeResult:
- text
- patterns_before
- patterns_after
- changes
- claim_preservation
- naturalness_scores
- rewrite_count
- warnings

CLI:
- seo-japanese rewrite article.md
- seo-japanese evaluate article.md
- seo-japanese rewrite article.md --facts source.json

Pipeline:
1. normalize input
2. immutable regionを保護
3. base + Japanese taxonomyでAI-writing patternを検出
4. supported claimを維持したままrewrite
5. original/source facts と rewritten text のclaimを比較
6. invented / modified factがあればFAIL
7. independent naturalness criticで評価
8. 必要な場合のみ1回rewrite
9. claim verificationを再実行
10. final output + evaluation metadataを返す

Initial Japanese taxonomy:
JA-01 定型イントロ（〜について解説していきます等）
JA-02 「重要です」「ポイントです」等の自己宣言
JA-03 「〜と言えるでしょう」の過剰使用
JA-04 「〜することができます」の反復
JA-05 段落末テンプレ反復
JA-06 また/さらに/一方で の機械的接続
JA-07 文長・リズムの均一化
JA-08 強制的三点列挙
JA-09 section構造の過剰反復
JA-10 読者に不要な一般論
JA-11 過剰な名詞化
JA-12 冗長敬語
JA-13 根拠のない「近年注目」型一般化
JA-14 不要な総括・締め
JA-15 heading直後の内容反復

各ruleは:
- id
- name
- strength
- signals
- problem
- action
- false_positive_guard
を持てる構造にする。

Claim preservation hard gates:
- invented material factual claim = 0
- materially modified factual claim = 0
- names / numbers / dates / prices / quotes / citations / URLsを保護
- ranking / comparison / causality / simultaneityを意味変更しない
- originalの重要claimを不用意にdropしない

File mode immutable:
- code blocks
- inline code
- commands
- paths
- YAML/frontmatter
- URLs/link targets
- structured data

Naturalness critic structured fields:
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

Tests:
- 各Japanese taxonomyにpositive fixtureとnegative fixtureを作る
- false positive guardをテストする
- 最低30件の日本語AI文章golden fixtureから開始
- meaning/fact preservation regressionを用意
- rewrite後にstrong pattern数が減ること
- material fact追加が0であること
- 固有名詞/数字/日付/citationが維持されること
- immutable regionが変更されないこと
- rewrite_count <= 1

V1 acceptance:
- CLIから日本語文章をrewriteできる
- Python APIから利用できる
- agent skillとして利用可能
- Japanese pattern最低15種
- facts入力時にinvented claimを検出できる
- facts未入力時もoriginal claim preservationを検査できる
- criticがstructured resultを返す
- rewrite最大1回
- golden regressionが全PASS
- READMEだけで他PJから導入できる

実装中に新しい大規模基盤が欲しくなっても、V1 acceptanceに不要なら追加しない。
将来候補はdocsに記録するだけに留める。

実装完了後:
- 実装内容
- テスト結果
- 未解決リスク
- V2へ送った事項
をREADME/docsへ反映する。
```
