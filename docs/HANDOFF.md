# HANDOFF — 日本語AI生成文の自然化プロジェクト

## 1. 背景

このプロジェクトの目的は、AIが生成した日本語文章から「AIっぽさ」を減らし、人間が普通に書いた文章として読める品質へ近づけること。

ここでいう「AIっぽさ脱却」はAI検出器を騙すことではない。対象読者が読んだときに、自然さ・具体性・文脈適合性・語り口・リズムの面で不自然さを感じにくくしながら、元情報の事実性・意味・安全性を守ることを意味する。

調査元PDFでは、自然さを単独で最大化せず、factuality / meaning preservation / task adherence / safety / legal complianceをハード制約として上位に置く設計が推奨されている。

## 2. 調査で得た主要結論

### AI detectorはNorth Starにしない

DetectGPT、Fast-DetectGPT、Binoculars、Ghostbuster等は診断・red-team用途には使えるが、本番の合否判定や学習報酬にしてはいけない。

Detector scoreを直接最適化すると、「自然な文章」ではなく「特定検出器が苦手な文章」への過適合になる。

### Human-likeは複数traitに分解する

最低でも以下を独立評価する。

- naturalness
- specificity
- rhythm
- vocabulary fit
- voice consistency
- Japanese cultural / pragmatic fit
- non-template feel
- factuality / meaning preservation

### 一発生成ではなく二段階にする

内容・事実を決める工程と、表現を自然化する工程を分離する。

表現改善によって新しい事実を勝手に作ることを防ぐため、rewrite後には独立したclaim checkerを通す。

### rewrite loopは有限にする

criticが満足するまで無限に書き直す構成にはしない。V1は最大1回。将来でも原則0〜1回、多くても2回程度とする。

### Fine-tuningは後

初期改善順は概ね次の通り。

`Prompt / Profile → Examples → Decoder → Two-pass → SFT/LoRA → DPO`

V1ではSFT、DPO、vLLM、Langfuse、MAUVEなどを実装しない。評価可能な最小構成を先に作る。

## 3. GitHub調査

主要reference implementationとして `blader/humanizer` を採用候補とする。

Repository:

- `https://github.com/blader/humanizer`
- MIT License
- Agent Skill形式
- Codex / Claude Code / Cursorなどのagent利用を想定

HumanizerはAI文の癖を単語置換ではなく構造的patternとして扱っている。現在のSKILLでは25 patternに整理されており、大きく次に分類される。

- staging instead of stating
- rhythm by rule
- inflation / borrowed authority
- formatting by rule
- chatbot / drafting leftovers

処理手順も参考になる。

1. AI的なtellを検出
2. 元の段落構造を固定せずrewrite
3. 事実・数値・日付・固有名詞等の追加/欠落を確認
4. 残ったtellを再確認
5. final rewrite

この思想は今回のPDFと整合している。

## 4. 採用方針

`blader/humanizer` をforkして直接大改造することは避ける。

理由:

- upstreamの改善を取り込みやすくする
- 英語固有の表現と抽象patternを分離できる
- 日本語独自taxonomyを自分たちの責任範囲として管理できる

推奨構造:

```text
seo_japanese/
├─ skills/
│  ├─ humanizer/
│  └─ ja-humanizer/
├─ rules/
│  ├─ base/
│  └─ ja/
├─ src/
│  ├─ pipeline/
│  ├─ claims/
│  ├─ evaluation/
│  └─ adapters/
├─ evalsets/
│  └─ golden/
├─ tests/
│  ├─ unit/
│  └─ regression/
└─ docs/
```

upstreamのSKILLをそのままvendorするか、install依存にするかは実装開始時に最新版を確認して決定する。

## 5. V1の処理フロー

```text
Input text + optional facts/source
        │
        ▼
Input normalization
        │
        ▼
AI-pattern analysis
        │
        ▼
Japanese rewrite
        │
        ▼
Claim / factuality checker
        │
   PASS ├──────── FAIL
        │           │
        │      one rewrite
        │           │
        └──────┬────┘
               ▼
      Naturalness critic
               │
               ▼
      Final verification
               │
               ▼
          Final output
```

## 6. 日本語taxonomy 初期案

V1では最低15種を実装する。

1. 「〜について解説していきます」等の定型イントロ
2. 「重要です」「ポイントです」等の重要性自己宣言
3. 「〜と言えるでしょう」の過剰使用
4. 「〜することができます」の反復
5. 段落末の定型句反復
6. 「また」「さらに」「一方で」の機械的接続
7. 文長・テンポの均一化
8. 意味上不要な三点列挙
9. 全sectionで同じ結論→説明→まとめ構造
10. 読者に不要な一般論
11. 過剰な名詞化
12. 冗長敬語
13. 根拠のない「近年注目されています」型一般化
14. 不要な総括文
15. 見出し内容を直後の本文でそのまま反復

必要に応じてHumanizerの抽象patternを日本語へmappingする。

## 7. 事実保持

Naturalnessより優先するhard gate。

特に以下を保護対象とする。

- 人名・会社名・商品名等の固有名詞
- 数値
- 日付
- 価格
- 引用
- URL
- citation
- 比較・順位
- 同時発生 / 因果関係等のclaim

source factsが与えられる場合、sourceとoutput双方からclaimを抽出し、少なくとも以下を判定する。

```json
{
  "preserved": 31,
  "modified": 0,
  "invented": 0,
  "dropped": 0
}
```

`invented > 0` または materially modified claimが存在する場合はFAIL。

## 8. Naturalness critic

writerとは別interfaceにする。

例:

```json
{
  "naturalness": 4.2,
  "specificity": 4.1,
  "rhythm": 3.8,
  "vocabulary": 4.3,
  "voice_consistency": 4.4,
  "cultural_fit": 4.1,
  "non_template_feel": 3.7,
  "remaining_patterns": ["JA-06", "JA-14"],
  "needs_rewrite": true
}
```

criticは文章を書き直さず評価だけを返す。

## 9. 学習ループ

V1完成後、実運用から以下を保存できる構造にする。

```text
AI draft
  ↓
seo_japanese output
  ↓
Human final edit
  ↓
Published text
```

将来的に価値が高いfeedbackはthumbs up/downより編集差分。

`system output → human edited final` の差分を分類して、頻出失敗patternを更新する。データ量と評価系が十分になった場合だけSFT / DPOを検討する。

## 10. V1完了条件

- CLIから日本語文章をrewriteできる
- Python APIから呼べる
- agent skillとして利用できる
- Japanese patternを最低15種検出できる
- source factsがある場合にinvented claimを検出できる
- criticがstructured outputを返す
- rewrite回数は最大1
- golden regressionが存在する
- 他repoからREADMEだけで導入方法が分かる

## 11. 非目標

V1では次を実装しない。

- AI detector score optimization
- SFT / LoRA
- DPO / RLHF
- vLLM serving
- Langfuse導入
- MAUVEによる本格distribution eval
- 大規模human annotation platform
- 特定個人の文体模倣

まずは「自然化 → 事実保持 → 評価 → 回帰テスト」が一貫して動くことを優先する。
