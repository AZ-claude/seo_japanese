# UPSTREAM — blader/humanizer

## Reference repository

`https://github.com/blader/humanizer`

調査時点で、AI生成文の「AIっぽさ」を構造的patternとして扱うAgent Skill実装として最も本PJに近い。

## 採用理由

Humanizerは単なる禁止語リストではない。

現在のSKILLではAI文に見られるpatternを25種に整理し、大きく以下に分類している。

- staging instead of stating
- rhythm by rule
- inflation / borrowed authority
- formatting by rule
- chatbot / drafting leftovers

また処理手順として、

1. tellを検出
2. supported claimを保持したままrewrite
3. draftを元文と比較
4. 残存tellを再確認
5. final rewrite

を定義している。

特に本PJと相性がよい原則:

- 元文の段落構造を固定しない
- supported claimを落とさない
- 名前・数値・日付・引用・citation等を勝手に追加しない
- 不足情報を自然さのために捏造しない
- style sampleはoptional
- technical/reference proseとpersonal proseを区別する
- file modeではcode、paths、URLs、metadata等を保護する

## License

調査時点ではMIT License。

実装時には必ず最新版LICENSEを再確認し、必要なcopyright / license noticeを保持する。

## Integration policy

### やらない

upstreamをコピーして英語patternを直接日本語へ書き換え、独自forkだけを育てる構成にはしない。

### やる

Humanizerを「base pattern / reference workflow」として扱い、日本語固有ルールを別layerにする。

概念:

```text
blader/humanizer
       │
       ▼
abstract base patterns
       │
       ├───────────────┐
       ▼               ▼
English realization   Japanese realization
                       │
                       ▼
                 seo_japanese rules
```

例えばHumanizerの `staged run-up before the point` を、単純翻訳ではなく日本語での発現として以下へmappingする。

```text
それでは詳しく見ていきましょう
ここから詳しく解説します
まずは〜について確認していきましょう
〜について解説していきます
```

## Update strategy

実装開始時にCodexは最新版の以下を再取得すること。

- README.md
- SKILL.md
- AGENTS.md
- LICENSE
- scripts/
- release / recent commits when relevant

upstream更新を取り込む際は、日本語ルールへ自動で上書きしない。

変更を次に分類する。

1. abstract pattern change
2. English-only realization change
3. workflow / claim preservation improvement
4. packaging / agent integration change

1と3は本PJへ反映候補。2は参考のみ。4は利用方式に影響する場合だけ取り込む。

## Why not detector-focused humanizers

GitHub上にはAI detector bypassを前面に出したhumanizer実装も多数存在するが、本PJのNorth Starとは異なる。

本PJではdetector scoreをoptimization targetにしない。

評価対象は、

- naturalness
- specificity
- rhythm
- vocabulary fit
- voice consistency
- Japanese cultural / pragmatic fit
- non-template feel
- factuality / meaning preservation

とする。

Detectorは将来追加してもred-team diagnosticに限定する。
