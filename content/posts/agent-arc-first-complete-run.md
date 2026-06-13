---
title: "12人のAIエージェントを配置する"
slug: "agent-arc-first-complete-run"
date: 2026-06-12T21:00:00+09:00
lastmod: 2026-06-12T21:00:00+09:00
draft: false
categories: ["tech", "opencode"]
tags: ["Agent Arc", "opencode", "AIエージェント", "CLI", "Hugo", "GitHub Pages"]
description: "opencode + deepseek 上で動作する12人のAIエージェント（Build 7人 + Blog 5人）の開発パイプライン「Agent Arc」を初めて完走させた実践レポート。3つの人間チェックポイント、2回の差し戻し、61のテストを経てブログが公開されるまでの全工程を解説。"
tldr: "opencodeの12人のAIエージェント（Build 7人 + Blog 5人）による開発パイプライン「Agent Arc」を初めて完走させた。Hugo + GitHub Pages のブログ基盤構築からTypeScript製CLIツールの実装、GEO対策までを全自動パイプラインで行い、61テスト・34受入基準すべてをクリアした実運用レポート。"
---

## この記事でわかること

- 12人のAIエージェントによる開発パイプライン「Agent Arc」の全貌
- 最初のワンストップ実行で起きた実問題とその解決プロセス
- Hugo + PaperMod + TypeScript CLI の技術選定判断とその理由
- AIエージェント開発における人間チェックポイントの設計

## 背景

opencode + deepseek 上で動作する12人のAIエージェント開発パイプライン「Agent Arc」を構築して約2週間。エージェント定義やオーケストレーション設定は完了しているものの、まだ一度もパイプラインを最初から最後まで通しで実行したことがありませんでした。

そこで、「2026年のGEO対策、収益化可能、低コスト、ターミナルCRUDで投稿できるブログを構築したい」という架空の依頼をオーケストレーターに与え、Build Pipeline（7エージェント）→ Blog Pipeline（5エージェント）の全12エージェントをワンストップで動かすことに挑戦しました。

本記事はその初完走レポートです。

## TL;DR（結論）

Agent Arcは**実用的に動作した**。3つの人間チェックポイントと2回の差し戻しを経て、Hugo + GitHub Pages のブログが61テスト・34受入基準すべてをクリアして公開された。ただし「完全自動」ではなく「人間が適切に介入するパイプライン」という設計が功を奏した。slugパストラバーサルやHugo v0.163.1の互換性問題など、エッジケースの検出にはテスト自動化とコード検証が不可欠だった。

## 詳細手順

### 1. 依頼から設計まで：code-researcher → story-writer → spec-writer

パイプラインの最初の3エージェントは**設計の品質を決める最重要フェーズ**です。いずれも読み取り専用権限で動作し、出力は人間チェックポイントを通過します。

#### code-researcher：既存コードベース調査

最初に呼ばれるcode-researcherは、プロジェクト全体をスキャンして現状を把握します。

```
調査結果サマリ:
- 12エージェント定義: ✅ 完了（Build 7 + Blog 5）
- オーケストレーション設定: ✅ 完了（opencode.json + SKILL.md）
- ブログ基盤: ❌ 未着手（これから作る）
- Playwright MCP: ✅ 導入済み・動作確認完了
```

この調査結果を踏まえ、「ブログ基盤をゼロから構築する」という方向性が確定しました。

#### story-writer：5つのユーザーストーリー

story-writerは依頼を具体的なユーザーストーリーに分解します。このケースでは5つのストーリーが生成されました。

| ストーリー | テーマ | 主な受け入れ基準 |
|-----------|--------|-----------------|
| A | ブログ基盤 | blog initでHugo+PaperModがセットアップできる |
| B | CLI CRUD | blog new/list/edit/publish/unpublish/delete が使える |
| C | GEO対策 | JSON-LD/TL;DR/結論ファーストarchetype |
| D | 収益化 | AdSense/GA4/Plausibleのワンスイッチ切替 |
| E | 2026潮流 | AIクローラー許可/モバイルファースト |

**→ ここで人間チェックポイント①：ストーリー承認**

#### spec-writer：技術ブリーフ

承認されたストーリーをもとに、spec-writerが技術選定を行います。

```
選定結果:
- ブログ基盤: Hugo + PaperMod
  - 理由: Go製で爆速ビルド、Markdownネイティブ、Git管理と好相性
- ホスティング: Netlify（後にGitHub Pagesに変更）
  - 理由: 完全無料、GitHub Actions連携
- CLI: TypeScript + Commander + Zod
  - 理由: 既存スタック統一、ランタイム型検証
- GEO対策: TechArticle JSON-LD + TL;DR + robots.txt
- 収益化: params.monetization.enabled ワンスイッチ
```

**→ ここで人間チェックポイント②：ブリーフ承認**

### 2. 実装フェーズ：backend-builder + frontend-builder

ブリーフが承認されると、backend-builderとfrontend-builderが並行実装に入ります。

#### backend-builder：27ファイルのCLIツール

TypeScript + Commander + Zod で10のサブコマンドを持つCLI `blog` を実装しました。

```typescript
// blog CLI のエントリポイント（src/index.ts）
const program = new Command();

program
  .name("blog")
  .description("Hugoブログ管理CLI - 記事作成・編集・公開・削除・Git管理")
  .version("0.1.0");

// 10のサブコマンド
program.command("init")     // ブログ新規作成
program.command("new")      // 記事作成
program.command("list")     // 一覧表示
program.command("edit")     // エディタ編集
program.command("publish")  // 公開
program.command("unpublish")// 下書き戻し
program.command("delete")   // 削除
program.command("push")     // Gitプッシュ
program.command("preview")  // プレビューサーバー
```

slugの生成処理は、日本語・特殊文字の除去、連番重複回避などを実装しています。

```typescript
// src/services/slug.ts
export function generateSlug(title: string, existingSlugs: string[] = []): string {
  let slug = title.toLowerCase();
  slug = slug.replace(/[^\w\s-]/g, ""); // 特殊文字除去
  slug = slug.replace(/\s+/g, "-");      // 空白→ハイフン
  slug = slug.replace(/-+/g, "-");       // 連続ハイフン統合
  slug = slug.replace(/^-+|-+$/g, "");   // トリム
  if (!slug) slug = "article";

  // 重複チェックと連番付与
  let counter = 0;
  let candidate = slug;
  while (existingSlugs.includes(candidate)) {
    counter++;
    candidate = `${slug}-${counter}`;
  }
  return candidate;
}
```

#### frontend-builder：15のHugoテンプレート

HugoテーマPaperModを拡張する15のカスタムテンプレートを実装しました。特筆すべきは3点です。

**① TechArticle JSON-LD（GEO対策）**

```html
{{- if and (eq .Type "posts") (eq .Kind "page") -}}
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "TechArticle",
  "headline": {{ .Title | jsonify }},
  "description": {{ .Description | jsonify }},
  "datePublished": {{ .Date.Format "2006-01-02T15:04:05Z07:00" | jsonify }},
  "wordCount": {{ .WordCount }},
  "timeRequired": {{ .Params.timeRequired | jsonify }},
  "inLanguage": "ja-JP"
}
</script>
{{- end -}}
```

**② TL;DRブロック**

```html
{{- if .Params.tldr -}}
<blockquote class="tldr-block">
  <strong class="tldr-label">TL;DR</strong>
  <p class="tldr-content">{{ .Params.tldr | .RenderString }}</p>
</blockquote>
{{- end -}}
```

**③ AIクローラー許可のrobots.txt**

```
User-agent: GPTBot
Allow: /

User-agent: ChatGPT-User
Allow: /

User-agent: Claude-Web
Allow: /

User-agent: Google-Extended
Allow: /

User-agent: PerplexityBot
Allow: /
```

GEO（Generative Engine Optimization）対策として、AIクローラーを積極的に許可する設計は2026年の潮流を反映しています。

### 3. 品質保証：test-verifier + code-validator

#### test-verifier：61テスト・34受入基準・100% PASS

test-verifierは既存の27ユニットテストに加え、34の受け入れテストを新規作成しました。テストは大きく3層で構成されています。

| テスト層 | 内容 | テスト数 |
|---------|------|---------|
| ユニットテスト | slug生成・frontmatter解析 | 27 |
| 受け入れテスト（Story A〜E） | 各受け入れ基準の検証 | 34 |
| テンプレートテスト | 15テンプレートの存在確認 | 15（内数） |

受け入れテストの一部を紹介します。

```typescript
// AC-B5: blog publish で draft = false
it("[AC-B5] blog publish <slug> で draft が false になる", async () => {
  const { createPost } = await import("../../src/services/post.js");
  const post = await createPost("Publish Test", join(tmpDir, "content", "posts"));

  const { publishPost } = await import("../../src/commands/publish.js");
  await publishPost("publish-test", { blogDir: tmpDir, contentDir: join(tmpDir, "content", "posts") });

  const content = readFileSync(post.filePath, "utf-8");
  expect(content).toContain("draft: false");
});
```

```typescript
// AC-E7: robots.txt が GPTBot/ChatGPT-User を許可
it("[AC-E7] robots.txt が GPTBot/ChatGPT-User を許可している", () => {
  const robots = readTemplate("layouts/robots.txt");
  expect(robots).toContain("GPTBot");
  expect(robots).toContain("ChatGPT-User");
  expect(robots).toContain("Allow:");
});
```

#### code-validator：Important 5件の修正

code-validatorが検証した結果は以下の通りでした。

| 重要度 | 件数 | 主な指摘 |
|--------|------|---------|
| Critical | 0 | 問題なし |
| Important | 5 | slugパストラバーサル、排他オプション不足、デッドコード、テスト不足 |
| Minor | 6 | 型アノテーション不足、コメント表記ゆれ |

Importantの5件を迅速に修正。特にslugパストラバーサル対策はセキュリティ上重要です。

```typescript
// src/utils/validate.ts — パストラバーサル対策
export function validateSlug(slug: string, allowEmpty: boolean = false): string | null {
  if (!slug || slug.trim().length === 0) {
    if (allowEmpty) return null;
    return "スラッグを指定してください。";
  }
  if (!/^[a-z0-9-]+$/.test(slug)) {
    return (
      `スラッグ '${slug}' は無効な形式です。\n` +
      "スラッグは英小文字・数字・ハイフンのみ使用できます。"
    );
  }
  return null;
}
```

この`validateSlug`関数は`../../`のようなパストラバーサルを含むスラッグを確実に弾きます。

### 4. デプロイ：GitHub Pagesへの道

実は初回のデプロイ先はNetlifyを予定していました。しかし、Netlify CLIの認証問題に遭遇。調べてみるとNetlify CLIはブラウザ経由のOAuth認証が必要で、HEAdless環境ではハードルが高いことがわかりました。

急遽GitHub Pagesに切り替え。デプロイフローを以下のように設計しました。

1. `blog push` でGitHubリポジトリにプッシュ
2. GitHub Actionsが自動検知してビルド
3. `peaceiris/actions-gh-pages@v4` でgh-pagesブランチにデプロイ
4. GitHub Pagesが自動公開

初回のGitHub Actionsデプロイは失敗しました（謎のタイムアウト）。再実行で成功。以後安定しています。

最終的な公開URL：<https://tomoln.github.io/blog/>

### 5. 技術選定の理由と実際の教訓

#### Hugoを選んだ理由（Go製・爆速・Markdownネイティブ）

静的サイトジェネレーターの選択肢は多くありますが、Hugoを選んだ決め手は3つ。

1. **ビルド速度**: Go製で記事100件でも秒単位のビルド
2. **Markdownネイティブ**: Git管理と相性が良く、CLIとの連携が容易
3. **PaperModテーマ**: 50KBの軽量テーマでCore Web Vitals最適化済み、SEO機能内蔵

実際に使ってみて、`hugo.yaml`の記述がやや複雑（mediaTypesやoutputFormatsの設定が必要）な点は感じましたが、一度設定してしまえば問題ありません。

#### PaperModを選んだ理由（軽量・SEO内蔵）

PaperModは以下の特徴がGEO対策フェーズで大きなアドバンテージになりました。

- 標準でJSON-LD構造化データを出力（ただしTechArticle型はカスタム実装が必要）
- OGPタグ対応
- テーマ容量50KB以下
- アクセシビリティ対応

#### CLIをTypeScriptにした理由（既存スタック統一）

このプロジェクトではTypeScript + Node.jsが既存スタックだったので、CLIもTypeScriptで統一。Commander + Zodの組み合わせが非常に強力でした。

```typescript
// Zodスキーマから型を自動生成
const PostSchema = z.object({
  title: z.string().min(1),
  slug: z.string().regex(/^[a-z0-9-]+$/),
  draft: z.boolean().default(true),
  categories: z.array(z.string()).default([]),
  tags: z.array(z.string()).default([]),
});

type Post = z.infer<typeof PostSchema>;
// ↑ これでランタイム検証と型定義の二重管理が解消
```

このパターンは小さなプロジェクトほど威力を発揮します。ランタイム検証とTypeScriptの型が一致するため、不整合バグが根本的に発生しません。

#### 実際に起きた問題8選

初回実行ならではの問題が多く発生しました。すべて記録に残します。

| # | 問題 | 原因 | 解決 |
|---|------|------|------|
| 1 | Hugo未インストール | 環境前提条件の確認漏れ | `winget install Hugo.Hugo` (v0.163.1) |
| 2 | テンプレートがコピーされない | `fs.cp()`が非再帰 | `copyTemplatesRecursive()` を自作 |
| 3 | Hugo v0.163.1非推奨キー | `twitter→x`, `languageCode→locale` | 設定ファイルを書き換え |
| 4 | 静的ページのfrontmatterエラー | `{{ .Date }}`がHugoビルドエラー | 実際の日付に置換 |
| 5 | slugパストラバーサル脆弱性 | slugに`../../`を許容 | `validate.ts`でバリデーション |
| 6 | code-validatorが5件のImportant | セキュリティ・デッドコード等 | すべて修正 |
| 7 | Netlify CLI認証問題 | CLIがブラウザOAuth必須 | GitHub Pagesに移行 |
| 8 | GitHub Actions初回失敗 | 不明（インフラ起因） | 再実行で成功 |

特にHugo v0.163.1の非推奨キー対応は、テンプレートエンジンのバージョンアップに伴うもので、生成AIが最新バージョンの変更をキャッチアップしきれていない典型例でした。コードブロックのエラーメッセージはこんな感じです。

```
Error: "twitter" is deprecated. Use "x" instead.
Error: "languageCode" is deprecated for this use. Use "locale" instead.
```

生成AIが出力する設定には常に「最新バージョンで動くか」という観点での検証が必要です。

### 6. Agent Arcの実用性評価

初回完走を終えての総評です。

**良かった点**

- 人間チェックポイントが適切に機能（設計の早い段階で軌道修正可能）
- test-verifierとcode-validatorの差し戻しループが品質を担保
- エージェント間の情報連携が想定通り動作
- 61テスト + 34受入基準の検証網は実用的

**課題点**

- エラー発生時のリカバリがまだ手動依存（エージェントが自動リトライできないケースがある）
- Hugoバージョン差分のような外部依存の変化に対応できていない
- パイプラインの完走に約45分（LLM応答待ち時間含む）
- Blog Pipelineのsite-publisherがまだ人間のgit push承認を必要とする

## まとめ

12人のAIエージェントパイプライン「Agent Arc」は初回実行で実用レベルに達していることを確認できました。3つの人間チェックポイント、2回の差し戻し、61のテストを経て、Hugo + GitHub Pagesのブログが無事公開されました。

特に重要な学びは次の3つです。

1. **AIエージェントパイプラインは「完全自動」ではなく「人間の適切な介入点」の設計が鍵**
2. **code-validatorのような最終検証エージェントがセキュリティホールを防ぐ**
3. **実プロジェクトで通し実行することで初めて見える問題がある（Hugo互換性、Netlify認証など）**

今後の展望として、Blog Pipelineの完全自動化（人間承認を省いてGitHub Actionsで自動デプロイ）、エラーハンドリングの強化、そして他のプロジェクトから呼び出せるCLI `npx agent-arc init` の提供を計画しています。

Agent Arcに興味を持たれた方は、ぜひGitHubリポジトリをチェックしてみてください。スターやフィードバックをお待ちしています。

---

**関連リンク**

- 実際のブログ: <https://tomoln.github.io/blog/>
- Agent Arc リポジトリ: <https://github.com/tomoln/argonpt2>
- Hugo: <https://gohugo.io/>
- PaperMod: <https://github.com/adityatelange/hugo-PaperMod>
- opencode: <https://opencode.ai/>
