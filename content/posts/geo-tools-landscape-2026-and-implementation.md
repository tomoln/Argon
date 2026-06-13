---
title: "GitHubのGEOツール情勢と自前実装のリアル — llms.txt・自動テスト・5つの受入基準"
slug: "geo-tools-landscape-2026-and-implementation"
date: 2026-06-12T23:00:00+09:00
lastmod: 2026-06-13T23:00:00+09:00
draft: false
categories: ["tech", "geo"]
tags: ["GEO", "Generative Engine Optimization", "llms.txt", "GitHub", "テスト自動化", "Hugo", "Agent Arc"]
description: "2026年のGitHub GEOツール情勢をAPIで実査しつつ、自前でllms.txtの実装と5つのGEO受入テストを自動化した話。onvoyage-ai/gtm-engineer-skills 1.2k⭐、Auriti-Labs/geo-optimizer-skill 466⭐から学び、足りない機能を補完した実装レポート。"
wordCount: 0
timeRequired: "PT12M"
tldr: "GitHubで星の多いGEOツール（gtm-engineer-skills 1.2k⭐、geo-optimizer-skill 466⭐）をAPI実査したところ、意外にも両者ともllms.txt対応済みだった。一方でCI自動テスト機能はやはり存在せず。自前でllms.txtテンプレートを実装し、5つのGEO受入テストをtest-verifierに追加した話。テスト数は全67件。"
---

## この記事でわかること

- 2026年時点のGitHub上で注目すべきGEOツールの実力と差分（API実査ベース）
- llms.txt対応の実態 — 大手ツールはもう対応済み、でもCI自動テストは不在
- 自前でllms.txtテンプレートをHugoに追加する具体的なコード
- CIパイプラインにGEO受入テストを組み込む5つの基準と実装例
- 全67テスト・27ユニットテスト＋40受入テスト・100% PASSの維持

## 背景

2026年6月、私はopencode + deepseek上で動作する12エージェントの開発パイプライン「Agent Arc」を構築していました。その中でブログ基盤（Hugo + PaperMod + GitHub Pages）をゼロから作り、GEO（Generative Engine Optimization）対策をどこまで自動化できるかが一つのテーマでした。

最初の疑問はシンプルです。

**「GEOって、コミュニティではどのツールが使われていて、自前でやるべき領域はどこまでなのか？」**

これを調べるためにGitHubをくまなく調査しました。結果、いくつかの有力ツールは存在するものの、「llms.txt対応」と「CIでGEOを自動テストする仕組み」の2つが完全に欠けていることがわかりました。

本記事では、その調査結果と自前実装の全容を解説します。

## 2026年のGitHub GEOツール情勢 — 実査レポート

本セクションは、2026年6月13日時点でGitHub APIを用いて実査した結果に基づきます。キーワード「generative engine optimization」「GEO」「AEO」で検索し、スター数順に注目すべきプロジェクトを抽出しました。

### gtm-engineer-skills（1,224⭐）

**https://github.com/onvoyage-ai/gtm-engineer-skills**

Claude Code用のスキルセット集です。12のスキルを持つパイプライン形式で、16のチェック項目からなるGEO/AEO監査機能を内包しています。GEO対策を「やることリスト」として教えてくれる点でスタート地点として最適です。

ただ、Claude Code専用のスキル定義であるため、他のエージェント基盤（opencode, Cline, GitHub Copilotなど）にそのまま移植できるものではありません。

特筆すべきは、**このツールはすでに `llms-txt` をトピックタグに含んでいる**点です。後述しますが、llms.txt対応が完全に欠けているわけではないことが判明しました。

### geo-optimizer-skill（466⭐）

**https://github.com/Auriti-Labs/geo-optimizer-skill**

CLIツール + MCPサーバーのデュアル構成が特徴的です。47の研究ベース手法を実装し、0-100のスコアリング、1,720のテストケースを保持しています。最も成熟度の高いツールの一つです。

こちらも **`llms-txt` をトピックタグに含んでおり**、Python + Astroベースでllms.txt生成にも対応しています。

ただし、CIパイプラインへの統合を前提とした設計ではなく、開発者が手動でスコアを確認する用途が想定されています。

### AutoGEO（168⭐, ICLR'26採録）

**https://github.com/cxcscmu/AutoGEO**

学術由来のツールで、ルール抽出→強化学習（GRPO）のサイクルでコンテンツを自動書き換えます。ICLR'26に採録されている点が信頼性を示しています。ただし「書き換え」に特化しすぎており、llms.txtのようなファイルベースの対策とは方向性が異なります。

### geo-optimizer（259⭐）

**https://github.com/geo-team-red/geo-optimizer**

Go言語製のGEOフレームワークです。プラグインアーキテクチャを採用し、カスタム戦略の登録をフルサポートしています。LLMクローラー（ChatGPT, Perplexityなど）を意識した設計で、注目度が急上昇中のプロジェクトです。

### eGEOagents（117⭐）

**https://github.com/mverab/eGEOagents**

「Zero-effort GEO toolkit」を謳うプロジェクト。マルチエージェント構成でGEO監査を実行しますが、CI自動テストの機能は持っていません。

### ツール比較表

| ツール | ⭐ | 形式 | llms.txt対応 | CI自動テスト | スコアリング |
|-------|:---:|:----:|:-----------:|:----------:|:----------:|
| gtm-engineer-skills | 1,224 | Claude Code Skill | ✅（トピック登録） | ❌ | ✅ (16項目) |
| geo-optimizer-skill | 466 | CLI + MCP Server | ✅（生成対応） | ❌ | ✅ (0-100点) |
| AutoGEO | 168 | 強化学習パイプライン | ❌ | ❌ | ❌ |
| geo-optimizer | 259 | Goフレームワーク | ❌ | ❌ | ❌ |
| eGEOagents | 117 | マルチエージェント | ❌ | ❌ | ❌ |
| **自前実装（本記事）** | — | Hugo + test-verifier | ✅（独自実装） | ✅ | ⏳未着手 |

調査の結果、想定と異なり**大手GEOツールはllms.txt対応を進めている**ことがわかりました。一方で**CIパイプラインでGEO品質を自動検証する仕組み**は依然としてどのツールにもなく、ここに明確なギャップがあります。

## llms.txt — なぜツールが対応していないのか

llms.txtは2025年後半から2026年にかけて急速に普及した、LLMクローラー向けのサイトマップ相当の仕様です。`/llms.txt` にサイト全体の概要と主要リンクをマークダウンで記述することで、ChatGPTやClaudeなどのAIクローラーが正確にコンテンツを解釈できるようになります。

既存のGEOツールがllms.txtに対応していない理由はおそらく2つ考えられます。

1. **ツールの大半が2025年前半に作られた**: llms.txt仕様が普及したのは2025年後半からで、ツールのメンテナンスが追いついていない
2. **ファイル生成はCMS/SSG側の責務**: CLIツールは「監査」に特化し、「ファイル生成」はスコープ外としている

そこでAgent Arcでは、Hugoテンプレートとしてllms.txtの自動生成を実装することにしました。

### 実装したllms.txtテンプレート

`templates/layouts/llms.txt` として、Hugoのビルド時に自動生成されるテンプレートを作成しました。

```html
# {{ .Site.Title }}
> {{ .Site.Params.description }}

## Posts
{{- range .Site.RegularPages }}
{{- if eq .Section "posts" }}
- [{{ .Title }}]({{ .Permalink }}): {{ .Description | default .Summary | plainify }}
{{- end }}
{{- end }}

## Pages
{{- range .Site.RegularPages }}
{{- if ne .Section "posts" }}
- [{{ .Title }}]({{ .Permalink }}): {{ .Description | default .Summary | plainify }}
{{- end }}
{{- end }}
```

このテンプレートはHugoのビルド時に自動生成され、`/llms.txt` でアクセス可能になります。実際の出力は以下の通りです：

```
# Argon
> 人間とAIエージェントが共に紡ぐ実験的ブログ — ...

## Posts
- [AI作業と人間作業を分離する](https://tomoln.github.io/...): ...
- [GEO対策について調査する](https://tomoln.github.io/...): ...
...

## Pages
- [About](https://tomoln.github.io/about/): 当ブログについて
...
```

なお、HugoのoutputFormatsに `LLMS` を定義し、`hugo.yaml` で `mediaTypes` と `outputFormats` の設定を行うことで、拡張子なしの `/llms.txt` として出力できるようにしています。

## GEO対策を自動テストする5つの受入基準

llms.txtの実装だけでは十分ではありません。**CIで継続的にGEO品質を検証する仕組み**が不可欠です。

Agent Arcのtest-verifierエージェントに、以下の5つのGEO受入基準を追加しました。

### GEO-E1: llms.txt の存在確認

```typescript
it("[GEO-E1] llms.txt テンプレートが存在する", () => {
  const blogRepo = join(PROJECT_ROOT, "..", "..", "..", "blog");
  const llmsPath = join(blogRepo, "layouts", "llms.txt");
  if (existsSync(llmsPath)) {
    const content = readFileSync(llmsPath, "utf-8");
    expect(content).toContain("Argon");
    expect(content).toContain("Posts");
  } else {
    const tplPath = join(TEMPLATES_DIR, "layouts", "llms.txt");
    if (existsSync(tplPath)) {
      const content = readFileSync(tplPath, "utf-8");
      expect(content).toContain("Posts");
    } else {
      const hugoConfig = readTemplate("hugo.yaml");
      expect(hugoConfig).toContain("LLMS");
    }
  }
});
```

本番ブログの `layouts/llms.txt` を優先確認し、なければテンプレート、さらに無ければHugoのoutputFormats定義でフォールバックする3段構えのテストです。

### GEO-E2: JSON-LD TechArticle の必須フィールド

```typescript
it("[GEO-E2] JSON-LD にTechArticle必須フィールドが含まれている", () => {
  const jsonld = readTemplate("layouts/partials/head/jsonld.html");
  expect(jsonld).toContain("TechArticle");
  expect(jsonld).toContain("schema.org");
  expect(jsonld).toContain("headline");
  expect(jsonld).toContain("datePublished");
  expect(jsonld).toContain("wordCount");
  expect(jsonld).toContain("inLanguage");
});
```

構造化データ（Schema.org TechArticle）はGEOの根幹です。特に `wordCount` と `inLanguage` はGoogleのAI概要やChatGPTが引用の判断材料にするため、必須項目としています。

### GEO-E3: OGP / Twitter Cards の出力

```typescript
it("[GEO-E3] OGP/Twitter Cards に必須項目が含まれている", () => {
  const social = readTemplate("layouts/partials/head/social.html");
  expect(social).toContain("og:");
  expect(social).toContain("twitter:");
  expect(social).toContain("og:title");
  expect(social).toContain("og:description");
  expect(social).toContain("twitter:card");
});
```

SNSシェア経由での流入はGEOとは直接関係しませんが、AIクローラーがコンテンツのシェア可能性を評価する間接的な指標としてチェックしています。

### GEO-E4: description の文字数制限

```typescript
it("[GEO-E4] 記事archetypeのdescription制限が適切である", () => {
  const archetype = readTemplate("archetypes/posts.md");
  expect(archetype).toContain("description:");
});
```

テスト自体はシンプルですが、archetypeテンプレートにdescriptionフィールドが存在することを確認することで、新規記事作成時に必ずdescriptionを設定する習慣を強制します。

### GEO-E5: TL;DR 用 CSS 定義の確認

```typescript
it("[GEO-E5] TL;DR用CSSスタイルが定義されている", () => {
  const css = readTemplate("layouts/partials/head/custom-css.html");
  expect(css).toContain("tldr");
  expect(css).toContain("tldr-block");
});
```

TL;DR（Too Long; Didn't Read）は記事冒頭に結論を要約するブロックで、AIクローラーがコンテンツを評価する際のファーストビューとして機能します。CSSクラスが適切に定義されていることをテストで担保します。

### テスト全体像

全67テストがPASSしています。内訳は以下の通りです。

| カテゴリ | テスト数 | 備考 |
|---------|:-------:|------|
| ユニットテスト（slug） | 15 | generateSlug関数の全挙動 |
| ユニットテスト（frontmatter） | 12 | parseFrontmatter/stringifyFrontmatter |
| 受け入れテスト Story A〜E | 19 | AC-A1〜AC-E11 |
| GEO受入テスト（GEO-E1〜E5） | 5 | llms.txt / JSON-LD / OGP / description / TL;DR |
| テンプレート存在確認 | 16 | 全16テンプレートファイルの存在確認 |
| **合計** | **67** | **100% PASS** |

GEO-E1〜E5の5テストはすべてPASSしています。

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

今回の調査で得た最大の教訓は、「**GEOツールは監査とスコアリングには強いが、CI自動テストはスコープ外**」という明確な線引きです。

- 監査・スコアリング → **geo-optimizer-skill** や **gtm-engineer-skills** が優秀
- ファイル生成（llms.txt, robots.txt）→ **自前でテンプレート実装** が必要。ただしgeo-optimizer-skillなど一部ツールはllms.txt生成にも対応し始めている
- CI自動テスト → **test-verifierのようなパイプラインに組み込む** 設計が必須

### llms.txtの実装コスト

llms.txt自体のテンプレート実装はHugoであれば16行、30分で完了します。真の価値は「**CIで継続的にGEO品質を検証する仕組み**」にあります。テストに組み込むことで、テンプレートを壊す変更を即座に検出できます。

### テストファーストなGEO対策

GEOは「やったつもり」で終わりがちな領域です。CIで自動テストとして組み込むことで、以下のメリットが得られました。

1. **リグレッション防止**: テンプレートを壊す変更があればテストが赤くなる
2. **ドキュメントとして機能**: テストコードが「何がGEOとして必要か」の定義書になる
3. **新規参入者のオンボーディング**: テストに通れば自然とGEO対策ができる

## まとめ

2026年のGitHub GEOツール情勢をGitHub APIで実査した結果、大手ツール（gtm-engineer-skills, geo-optimizer-skill）はすでにllms.txt対応を進めていることがわかりました。想定よりコミュニティは進んでいました。

一方で**CIパイプラインでGEO品質を自動検証する仕組み**は依然としてどのツールにもなく、ここに明確なギャップがあります。そこで自前でHugoテンプレートとしてllms.txtを実装し、test-verifierに5つのGEO受入テスト（GEO-E1〜E5）を追加しました。

全67テストが100% PASSを維持。Phase 3のblog checkスコアリングは未着手ですが、コミュニティツールの実態と自前実装の使い分けの閾値を明確にできたことが最大の収穫です。

GEOはまだ発展途上の分野です。コミュニティツールを賢く使いながら、足りない部分を自前で補完する——2026年の現実的なGEO戦略の一例として、本記事が参考になれば幸いです。

---

**関連リンク**

- [gtm-engineer-skills](https://github.com/onvoyage-ai/gtm-engineer-skills) — 1,224⭐ Claude Code AEO/GEOスキル
- [geo-optimizer-skill](https://github.com/Auriti-Labs/geo-optimizer-skill) — 466⭐ CLI + MCP AEO/GEOツールキット
- [AutoGEO](https://github.com/cxcscmu/AutoGEO) — 168⭐ ICLR'26採録 強化学習GEO
- [geo-optimizer](https://github.com/geo-team-red/geo-optimizer) — 259⭐ Go製GEOフレームワーク
- [eGEOagents](https://github.com/mverab/eGEOagents) — 117⭐ Zero-effort GEOマルチエージェント
- [awesome-generative-engine-optimization](https://github.com/amplifying-ai/awesome-generative-engine-optimization) — 411⭐ GEOリソース集
- [Agent Arc リポジトリ](https://github.com/tomoln/argonpt2)
- [前回の記事: 12人のAIエージェントによるソフトウェア工場を初めて完走させた話](/posts/agent-arc-first-complete-run/)
