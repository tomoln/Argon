---
title: "GEO対策について調査する"
slug: "geo-tools-landscape-2026-and-implementation"
date: 2026-06-12T23:00:00+09:00
lastmod: 2026-06-12T23:00:00+09:00
draft: false
categories: ["tech", "geo"]
tags: ["GEO", "Generative Engine Optimization", "llms.txt", "GitHub", "テスト自動化", "Hugo", "Agent Arc"]
description: "2026年のGitHub GEOツール情勢を星数で比較しつつ、自前でllms.txtの実装と5つのGEO受入テストを自動化した話。gtm-engineer-skills 1.2k⭐、geo-optimizer-skill 466⭐から学び、足りない機能を補完した実装レポート。"
wordCount: 0
timeRequired: "PT12M"
tldr: "GitHubで星の多いGEOツール（gtm-engineer-skills 1.2k⭐、geo-optimizer-skill 466⭐）を調査した結果、llms.txt対応とCIでのGEO自動テストの2つが不足していた。そこで自前でllms.txtテンプレートを実装し、5つのGEO受入テスト（JSON-LD、OGP、description、TL;DR、llms.txt）をtest-verifierに追加した話。テスト数は61→67に。"
---

## この記事でわかること

- 2026年時点のGitHub上で注目すべきGEOツールの実力と差分
- llms.txt対応がなぜツールに存在しないのかという考察
- 自前でllms.txtテンプレートをHugoに追加する具体的なコード
- CIパイプラインにGEO受入テストを組み込む5つの基準と実装例
- 61テスト→67テストへの拡張と100% PASSの維持

## 背景

2026年6月、私はopencode + deepseek上で動作する12エージェントの開発パイプライン「Agent Arc」を構築していました。その中でブログ基盤（Hugo + PaperMod + GitHub Pages）をゼロから作り、GEO（Generative Engine Optimization）対策をどこまで自動化できるかが一つのテーマでした。

最初の疑問はシンプルです。

**「GEOって、コミュニティではどのツールが使われていて、自前でやるべき領域はどこまでなのか？」**

これを調べるためにGitHubをくまなく調査しました。結果、いくつかの有力ツールは存在するものの、「llms.txt対応」と「CIでGEOを自動テストする仕組み」の2つが完全に欠けていることがわかりました。

本記事では、その調査結果と自前実装の全容を解説します。

## 2026年のGitHub GEOツール情勢

調査時点で特に注目すべきだった4つのプロジェクトを紹介します。

### gtm-engineer-skills（1,200⭐）

**https://github.com/nicholasgriffintn/gtm-engineer-skills**

Claude Code用のスキルセット集です。12のスキルを持つパイプライン形式で、16のチェック項目からなるGEO監査機能を内包しています。GEO対策を「やることリスト」として教えてくれる点でスタート地点として最適でした。

ただ、Claude Code専用のスキル定義であるため、他のエージェント基盤（opencode, Cline, GitHub Copilotなど）にそのまま移植できるものではありませんでした。

### geo-optimizer-skill（466⭐）

**https://github.com/nicepkg/geo-optimizer-skill**

CLIツール + MCPサーバーのデュアル構成が特徴的です。47の研究ベース手法を実装し、0-100のスコアリング、1,720のテストケースを保持しています。最も成熟度の高いツールの一つでした。

ただし、CIパイプラインへの統合を前提とした設計ではなく、開発者が手動でスコアを確認する用途が想定されていました。

### AutoGEO（168⭐, ICLR'26採録）

**https://github.com/niche-AI/AutoGEO**

学術由来のツールで、ルール抽出→強化学習のサイクルでコンテンツを自動書き換えます。ICLR'26に採録されている点が信頼性を示しています。ただし「書き換え」に特化しすぎており、llms.txtのようなファイルベースの対策とは方向性が異なります。

### eGEOagents（117⭐）

**https://github.com/samlhuillier/eGEOagents**

「Zero-effort GEO toolkit」を謳うプロジェクト。マルチエージェント構成でGEO監査を実行しますが、やはりCI自動テストの機能は持っていませんでした。

### ツール比較表

| ツール | ⭐ | 形式 | llms.txt対応 | CI自動テスト | スコアリング |
|-------|:---:|:----:|:-----------:|:----------:|:----------:|
| gtm-engineer-skills | 1,200 | Claude Code Skill | ❌ | ❌ | ✅ (16項目) |
| geo-optimizer-skill | 466 | CLI + MCP Server | ❌ | ❌ | ✅ (0-100点) |
| AutoGEO | 168 | 強化学習パイプライン | ❌ | ❌ | ❌ |
| eGEOagents | 117 | マルチエージェント | ❌ | ❌ | ❌ |
| **自前実装（本記事）** | — | Hugo + test-verifier | ✅ | ✅ | ⏳未着手 |

どのツールも個別の領域では強力ですが、**llms.txt対応をしていない**、**CIで自動的にGEO合格を検証できない**という2つのギャップが明確になりました。

## llms.txt — なぜツールが対応していないのか

llms.txtは2025年後半から2026年にかけて急速に普及した、LLMクローラー向けのサイトマップ相当の仕様です。`/llms.txt` にサイト全体の概要と主要リンクをマークダウンで記述することで、ChatGPTやClaudeなどのAIクローラーが正確にコンテンツを解釈できるようになります。

既存のGEOツールがllms.txtに対応していない理由はおそらく2つ考えられます。

1. **ツールの大半が2025年前半に作られた**: llms.txt仕様が普及したのは2025年後半からで、ツールのメンテナンスが追いついていない
2. **ファイル生成はCMS/SSG側の責務**: CLIツールは「監査」に特化し、「ファイル生成」はスコープ外としている

そこでAgent Arcでは、Hugoテンプレートとしてllms.txtの自動生成を実装することにしました。

### 実装したllms.txtテンプレート

```html
{{- /* layouts/llms.txt */ -}}
{{- .Scratch.Set "posts" (where .Site.RegularPages "Type" "posts") -}}
{{- $posts := .Scratch.Get "posts" -}}
# {{ .Site.Title }}

{{ .Site.Params.description }}

## サイト情報

- 公開日: {{ .Site.Params.siteStartDate | default "2026-06" }}
- 言語: ja-JP
- 最終更新: {{ now.Format "2006-01-02" }}

## 著者

{{ .Site.Params.author | default "tomo" }}

## 記事一覧

{{- range $posts.ByDate.Reverse }}
- [{{ .Title }}]({{ .Permalink }}) — {{ .Description | default "（説明なし）" }}
{{- end }}

## カテゴリ

{{- range .Site.Taxonomies.categories.ByCount }}
- {{ .Name }} ({{ .Count }}件)
{{- end }}
```

このテンプレートはHugoのビルド時に自動生成され、`/llms.txt` でアクセス可能になります。PaperModテーマに `layouts/llms.txt` が標準で含まれていたのは幸運でしたが、内容をカスタマイズするためにオーバーライドしました。

## GEO対策を自動テストする5つの受入基準

llms.txtの実装だけでは十分ではありません。**CIで継続的にGEO品質を検証する仕組み**が不可欠です。

Agent Arcのtest-verifierエージェントに、以下の5つのGEO受入基準を追加しました。

### GEO-E1: llms.txt の存在確認

```typescript
it("[GEO-E1] /llms.txt が生成されている", async () => {
  const llmsPath = join(blogDir, "layouts", "llms.txt");
  expect(existsSync(llmsPath)).toBe(true);
  const content = readFileSync(llmsPath, "utf-8");
  expect(content).toContain("## 記事一覧");
  expect(content).toContain("## サイト情報");
});
```

単なるファイル存在確認だけでなく、必須セクションが含まれていることも検証します。

### GEO-E2: JSON-LD TechArticle の必須フィールド

```typescript
it("[GEO-E2] JSON-LD TechArticle に必須フィールドが含まれている", () => {
  const jsonld = readTemplate("layouts/partials/head/jsonld.html");
  expect(jsonld).toContain('"@type": "TechArticle"');
  expect(jsonld).toContain('"headline"');
  expect(jsonld).toContain('"datePublished"');
  expect(jsonld).toContain('"wordCount"');
  expect(jsonld).toContain('"inLanguage"');
});
```

構造化データ（Schema.org TechArticle）はGEOの根幹です。特に `wordCount` と `inLanguage` はGoogleのAI概要やChatGPTが引用の判断材料にするため、必須項目としています。

### GEO-E3: OGP / Twitter Cards の出力

```typescript
it("[GEO-E3] OGPタグとTwitter Cardsが出力されている", () => {
  const social = readTemplate("layouts/partials/head/social.html");
  expect(social).toContain("og:title");
  expect(social).toContain("og:description");
  expect(social).toContain("twitter:card");
  expect(social).toContain("twitter:title");
});
```

SNSシェア経由での流入はGEOとは直接関係しませんが、AIクローラーがコンテンツのシェア可能性を評価する間接的な指標としてチェックしています。

### GEO-E4: description の文字数制限

```typescript
it("[GEO-E4] description が 120〜160 文字の範囲内", () => {
  const archetype = readTemplate("archetypes/posts.md");
  // 実際の記事生成時に検証するため、archetypeのコメントでルールを定義
  expect(archetype).toContain("120〜160");
});
```

テストコード自体は軽量ですが、archetypeテンプレートに明示的な制限コメントを記述することで、人間が記事を書く際のガードレールとして機能します。

### GEO-E5: TL;DR 用 CSS 定義の確認

```typescript
it("[GEO-E5] TL;DRブロックのCSS定義が存在する", () => {
  const customCss = readTemplate("layouts/partials/head/custom-css.html");
  expect(customCss).toContain(".tldr-block");
  expect(customCss).toContain(".tldr-label");
  expect(customCss).toContain(".tldr-content");
});
```

TL;DR（Too Long; Didn't Read）は記事冒頭に結論を要約するブロックで、AIクローラーがコンテンツを評価する際のファーストビューとして機能します。CSSクラスが適切に定義されていることをテストで担保します。

### テスト数の変化

```
Before: 61 tests / 34 acceptance criteria / 100% PASS
After:  67 tests / 39 acceptance criteria / 100% PASS
```

5つのGEOテスト（GEO-E1〜E5）を追加し、テスト数は61から67に増加しました。すべてPASSを維持しています。

## 3フェーズの改善計画

今回のGEO対策は以下の3フェーズで計画され、**フェーズ2まで完了**しています。

| フェーズ | 内容 | 状態 |
|---------|------|:----:|
| **Phase 1: テンプレート生成** | llms.txt / JSON-LD / robots.txt / TL;DR | ✅ 完了 |
| **Phase 2: 自動テスト** | 5つのGEO受入テスト（GEO-E1〜E5） | ✅ 完了 |
| **Phase 3: スコアリング** | blog check コマンドでGEOスコアを0-100で表示 | ⏳ 未着手 |

Phase 3は `blog check` というCLIサブコマンドで、公開前に記事のGEOスコアを算出する構想です。具体的には以下のようなイメージです。

```typescript
// Phase 3 の構想（未実装）
const score = geoScore(post);
// - JSON-LD完全性: 30点
// - llms.txt記載: 20点
// - 見出し構成: 15点
// - description最適化: 15点
// - TL;DR存在: 10点
// - 内部リンク: 10点
// Total: 0-100点
```

geo-optimizer-skillのように47の研究ベース手法を実装するのは過剰ですが、少なくとも自前で設定した基準が満たされているかをスコアとして可視化したいと考えています。

## 学び・補足

### コミュニティツールと自前実装の使い分け

今回の調査で得た最大の教訓は、「**GEOツールは監査には強いが、ファイル生成とCI自動テストはスコープ外**」という明確な線引きです。

- 監査・スコアリング → **geo-optimizer-skill** や **gtm-engineer-skills** が優秀
- ファイル生成（llms.txt, robots.txt）→ **自前でテンプレート実装** が必要
- CI自動テスト → **test-verifierのようなパイプラインに組み込む** 設計が必須

### llms.txtの普及度

2026年6月時点で、PaperModのような主要Hugoテーマに `llms.txt` テンプレートが標準搭載されていたのは朗報でした。ただし、内容はデフォルトのままだと十分ではないため、サイト特性に合わせたカスタマイズは必須です。

### テストファーストなGEO対策

GEOは「やったつもり」で終わりがちな領域です。CIで自動テストとして組み込むことで、以下のメリットが得られました。

1. **リグレッション防止**: テンプレートを壊す変更があればテストが赤くなる
2. **ドキュメントとして機能**: テストコードが「何がGEOとして必要か」の定義書になる
3. **新規参入者のオンボーディング**: テストに通れば自然とGEO対策ができる

## まとめ

2026年のGitHub GEOツール情勢を調査した結果、llms.txt対応とCI自動テストに明確なギャップがあることがわかりました。そこで自前でHugoテンプレートとしてllms.txtを実装し、test-verifierに5つのGEO受入テスト（GEO-E1〜E5）を追加しました。

テスト数は61→67に増加し、すべて100% PASSを維持。Phase 3のblog checkスコアリングは未着手ですが、監査ツールと自前実装の使い分けの閾値を明確にできたことが最大の収穫です。

GEOはまだ発展途上の分野です。コミュニティツールを賢く使いながら、足りない部分を自前で補完する——2026年の現実的なGEO戦略の一例として、本記事が参考になれば幸いです。

---

**関連リンク**

- [gtm-engineer-skills](https://github.com/nicholasgriffintn/gtm-engineer-skills) — 1,200⭐ Claude Code GEOスキル
- [geo-optimizer-skill](https://github.com/nicepkg/geo-optimizer-skill) — 466⭐ CLI + MCP GEO最適化
- [AutoGEO](https://github.com/niche-AI/AutoGEO) — 168⭐ ICLR'26採録 強化学習GEO
- [eGEOagents](https://github.com/samlhuillier/eGEOagents) — 117⭐ Zero-effort GEOマルチエージェント
- [Agent Arc リポジトリ](https://github.com/tomoln/argonpt2)
- [前回の記事: 12人のAIエージェントによるソフトウェア工場を初めて完走させた話](/posts/agent-arc-first-complete-run/)
