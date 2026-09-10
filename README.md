# seo_japanese

日本語のAI生成文を、意味・事実を維持したまま「機械的で不自然なAI文」から脱却させるための独立プロジェクト。

このリポジトリは、2026-09-10に実施した調査レポート **「AI生成文を『不自然なAI文』から脱却させるための設計・評価・運用戦略」** と、GitHub上の既存実装調査をもとに、Codexへそのまま引き継げる設計資料を格納する。

## North Star

目的はAI検出器の回避ではない。

- 日本語母語話者が読んで自然である
- 具体性、文脈適合性、語り口、リズムが機械的でない
- 元情報の意味・事実・固有名詞・数値・日付・引用を壊さない
- 生成器自身の自己評価だけに頼らず、独立したchecker / criticで検証する
- 運用で得た編集差分を将来の改善データに変える

## V1

V1はYAGNIを徹底し、fine-tuningや大規模MLOpsを先に作らない。

1. `blader/humanizer` を主要reference implementationとして調査・参照
2. 日本語固有のAI文パターンを追加
3. rewrite pipelineを実装
4. claim / factuality preservation checkerを独立実装
5. naturalness criticを独立実装
6. rewriteは最大1回
7. golden regressionで品質を固定
8. CLI / Python API / agent skillとして他PJから呼べる形にする

## Documents

- `AGENTS.md` — Codex向けの作業ルールと優先順位
- `docs/HANDOFF.md` — 調査から実装までのハンドオフ
- `docs/DESIGN.md` — V1アーキテクチャ・API・受入条件
- `docs/UPSTREAM.md` — `blader/humanizer` の採用理由と取り込み方針
- `docs/RESEARCH_NOTES.md` — 元PDFから採用した設計原則
- `docs/CODEX_GOAL.md` — Codexにそのまま渡せる実装指示
- `docs/source/README.md` — 調査元PDFの出典・保管方針

元PDFの実装に必要な内容は上記資料へ抽出済み。GitHub接続にはバイナリファイルを直接アップロードするactionがないため、元PDF本体は今回の自動ハンドオフでは未コミット。PDF本体を後から置く場合は `docs/source/` に保管する。

## Intended integration

このPJは記事生成PJそのものではなく、文章の自然化・品質保証を担当する独立コンポーネントとする。

将来的には、記事生成側から次のように利用する想定。

```python
result = humanize(
    text=article,
    facts=source_material,
    language="ja",
    content_type="article",
)
```

記事生成と自然化を疎結合に保つことで、SEO記事以外の説明文、メール、サイト内解説などにも再利用できるようにする。
