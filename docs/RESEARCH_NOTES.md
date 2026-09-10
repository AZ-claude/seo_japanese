# RESEARCH NOTES — 調査PDFから採用する設計原則

Source: `AI生成文を「不自然なAI文」から脱却させるための設計・評価・運用戦略.pdf`

この文書は元PDFの全文転記ではなく、`seo_japanese` 実装へ直接関係する論点をハンドオフ用に整理したもの。

## Goal definition

最適化対象を「AI detectorを通過すること」にしない。

目標は、良質な人間文が持つ具体性・文脈適応・文化的自然さ・多様なリズムを再現しながら、事実性・意味保存・指示遵守・安全性・法務要件を守ること。

自然さはhard constraintsの内側で改善する。

## Primary evaluation

最終的なNorth Starはhuman evaluation。

自動評価は回帰監視・候補絞り込みに使い、人間らしさの最終合否をLLM judgeやdetectorだけに任せない。

評価trait候補:

- 自然さ
- 具体性
- リズム
- 語彙
- 一貫した声
- 文化適合
- 非テンプレ感
- 内容品質

## Two-pass generation

推奨構造:

### Pass A — content / fact planning

- 事実
- 論点
- 読者
- 順序
- 根拠

を確定する。

### Pass B — surface realization / editorial rewrite

Pass Aの事実を変えずに、

- 文長
- 接続
- 段落
- 語調
- 具体性
- 冗長さ

を編集する。

本PJでは記事生成自体を必須責務にせず、既存記事やdraftを受け取るrewrite componentとして開始してよい。ただし「内容と表現を分離する」思想は維持する。

## Prompt design

「AIっぽくない文章にして」という曖昧指示だけに依存しない。

入力として扱える要素:

- 目的
- 読者
- 書き手と読者の関係性
- 文体profile
- 必須事実
- 推測禁止事項
- 引用 / 固有名詞
- optional reference examples

ただし本PJでは「自分の文章サンプルを必須」にしない。voice matchingはoptional featureとする。

## Decoder

Temperature / top-pに普遍的な正解値はない。

モデル・用途ごとにA/Bする。V1では設定可能にしておき、複雑な自動探索は不要。

## SFT / DPO

初手でfine-tuningしない。

まずPrompt / Examples / two-pass / criticで測定可能なbaselineを作る。

将来的には、自然な方を選ぶpairwise preferenceとDPOの相性がよい。

特に価値の高いtraining dataは、単なる大量のWeb人間文ではなく、

`AI draft → Human edited final`

の編集履歴。

## Rewrite loop

criticが満足するまで繰り返さない。

production rewriteは通常0〜1回、多くても2回を上限とする考え方を採用する。

V1は最大1回固定。

## Existing ecosystem noted by research

元PDFでは以下の技術も整理されている。

- DetectGPT
- Fast-DetectGPT
- Binoculars
- Ghostbuster
- MAUVE
- Hugging Face TRL
- PEFT
- vLLM
- Langfuse

本PJでの扱い:

- detectors: 将来のred-teamのみ
- MAUVE: 将来のcorpus-level regression候補
- TRL + PEFT: SFT / DPOフェーズ候補
- vLLM: self-hostが必要になった場合の候補
- Langfuse: 実験・production observabilityが必要になった場合の候補

V1に全部入れない。

## Recommended improvement order for this repository

```text
1. taxonomy / evaluation schema
2. Humanizer-inspired rewrite
3. factuality / claim preservation
4. independent critic
5. golden regression
6. real usage and edit-diff collection
7. human blind evaluation
8. only then consider SFT / DPO
```

## Production learning loop

将来的な改善loop:

```text
Generation
   ↓
seo_japanese rewrite
   ↓
Human editor
   ↓
Published text
   ↓
Edit-diff dataset
   ↓
Failure taxonomy / preference pairs
   ↓
Candidate prompt/model/rules
   ↓
Blind evaluation
   ↓
Canary / production
```

「thumbs up/down」だけより、人間が実際にどこを書き換えたかを重要なfeedbackとして保存する。

## Guardrail

特定作家や実在個人本人に見せかけることを目標にしない。

House Styleは複数の良い文章から抽象化した特性として扱う。
