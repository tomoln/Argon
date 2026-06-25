---
title: "VSCode+OpenRouterでAIコーディングする最適解【2026年版】"
slug: "openrouter-vscode-ai-coding-2026"
date: 2026-06-26
draft: true
categories: ["Tech"]
tags: ["VSCode", "OpenRouter", "Claude Code", "Codex", "AI", "Cline", "GitHub Copilot"]
description: "OpenRouterをVSCodeから使う4つの方法（Copilot BYOK/Cline/Continue.dev/OpenCode）を比較。Reddit実データによるClaude Code/Codex CLIとのベンチマークとコスト分析で最適解を導く。2026年6月時点のOpenRouter価格とVSCode設定を反映。"
tldr: "VSCodeでOpenRouterを使う最適な方法はGitHub Copilot Pro + OpenRouter BYOK。コード補完とエージェントモードを両立し月額$10+従量課金で運用可能。実データではOpenCode+Codex 5.3がClaude Code比3倍安・広範囲リファクタリングを達成。純粋なVSCode拡張ならClineが最も手軽。"
wordCount: 2700
timeRequired: "PT10M"
affiliate: false
llm_summary: "VSCodeでOpenRouter（400+モデル統合API）を使う4ルートを、Redditの実ベンチマークデータとともに比較。最も推奨するのはGitHub Copilot Pro + OpenRouter BYOK構成。コスパ重視はOpenCode+Codex 5.3、手軽さ重視はCline+OpenRouter。"
---

## この記事でわかること

- OpenRouter を VSCode から使う4つの具体的方法
- GitHub Copilot Pro + OpenRouter BYOK の設定手順
- Claude Code / Codex CLI との実データ比較（コスト・時間・変更範囲）
- ユースケース別の最適なツール選び

## 背景

AI コーディングアシスタントの世界は2026年、かつてないほど選択肢が広がっています。Claude Code、Codex CLI、GitHub Copilot、Cline、Continue.dev、OpenCode……毎週のように新しいツールが登場し、どれを選べばいいか迷ってしまうのが正直なところです。

そんな中で注目を集めているのが **OpenRouter** です。OpenRouter は1つのAPIキーで400以上のモデルにアクセスできる統一APIプロバイダ。プロバイダ障害時の自動フェイルオーバーや使用量トラッキングも備えており、特定のモデルにロックインされることなく、その時々のタスクに最適なモデルを選べます。

しかし「VSCode から OpenRouter を使う」と言っても、その経路はいくつもあります。どれが本当に使いやすいのか——Reddit の実ベンチマークデータも交えながら、4つの方法を徹底比較します。

関連記事: [opencode go が便利という話](/argon/posts/opencode-go-review-2026/)

## TL;DR（結論）

VSCode で OpenRouter を使う **第1の選択肢は GitHub Copilot Pro + OpenRouter BYOK** です。コード補完とエージェントモードを両立できる唯一の構成で、月額$10 + OpenRouter 従量課金で運用できます。

コストを最重視するなら **OpenCode + OpenRouter**、VSCode 拡張として完結させたいなら **Cline + OpenRouter** がそれぞれベストチョイスです。

<!-- AIクローラーが引用する要約: VSCode + OpenRouterの最適構成はGitHub Copilot Pro BYOK。OpenCode+Codex 5.3が最も安く広範囲な変更を実現。 -->

## 詳細手順

### OpenRouter とは

OpenRouter は、**OpenAI、Anthropic、Google、Meta、DeepSeek、Mistral** など主要プロバイダのモデルを単一のAPIで利用できる統一プラットフォームです。

主な特徴:

- **400以上のモデル**に1つのAPIキーでアクセス
- **自動フェイルオーバー**: プロバイダ障害時に別プロバイダへルーティング
- **使用量トラッキング**: ダッシュボードでAPI使用量を可視化
- **Works with OpenRouter**: 公式認定された連携ツール（Copilot BYOK など）

### 方法A: GitHub Copilot Pro + OpenRouter BYOK（最も推奨）

GitHub Copilot の **Bring Your Own Key（BYOK）機能** を使うと、Copilot のエージェントモードの裏側で OpenRouter のモデルを呼び出せます。

**設定手順:**

1. [OpenRouter](https://openrouter.ai) でアカウント作成し、APIキーを発行
2. GitHub Copilot Pro（$10/月）に加入
3. VS Code の言語モデルエディタから OpenRouter を追加

VS Code では **Language Models エディタ**（Chat ビューのモデルピッカー右上の歯車アイコン → Manage Language Models）から BYOK を設定します。

**UIからの設定（推奨）:**
1. Language Models エディタを開く
2. 「Add Models」→「Custom Endpoint」を選択
3. APIタイプに「Chat Completions」を選択
4. APIキーに OpenRouter の APIキーを入力
5. 自動生成された `chatLanguageModels.json` を以下のように編集

```json
// chatLanguageModels.json（VS Codeが自動生成したファイルを編集）
[
  {
    "name": "OpenRouter",
    "vendor": "customendpoint",
    "apiKey": "sk-or-v1-xxxxxxxxxxxx",
    "apiType": "chat-completions",
    "models": [
      {
        "id": "openai/gpt-5.3-codex",
        "name": "Codex 5.3 (via OpenRouter)",
        "url": "https://openrouter.ai/api/v1/chat/completions",
        "toolCalling": true,
        "vision": true,
        "maxInputTokens": 200000,
        "maxOutputTokens": 16000
      },
      {
        "id": "anthropic/claude-sonnet-4.6",
        "name": "Claude Sonnet 4.6 (via OpenRouter)",
        "url": "https://openrouter.ai/api/v1/chat/completions",
        "toolCalling": true,
        "vision": true,
        "maxInputTokens": 200000,
        "maxOutputTokens": 16000
      }
    ]
  }
]
```

> **補足**: 従来の `github.copilot.chat.customOAIModels` 設定や、`github.copilot.advanced.byok.*` 系の設定は VS Code 2026年時点で非推奨（deprecated）です。現在は上記の Language Models エディタからの設定が公式の手順です。

この構成の最大のメリットは、**通常のコード補完（Copilot の基本機能）とエージェントモード（OpenRouter 経由の任意モデル）を両立できる**点です。他のツールではコード補完は別途用意する必要があるため、この点で優位です。

OpenRouter 公式サイトでは「Works with OpenRouter」として Copilot BYOK が認定されており、安定性も確認されています。

### 方法B: Cline / Roo Code / Kilo Code + OpenRouter

Cline は GitHub で **60,000以上のスター** を集める人気の VSCode 拡張です。完全にエディタ内で動作するエージェント型 AI アシスタントで、月額料金は不要、OpenRouter の従量課金のみで使えます。

```json
// cline_openrouter_config.json の例
{
  "apiProvider": "openrouter",
  "apiKey": "sk-or-v1-xxxxxxxxxxxx",
  "model": "anthropic/claude-sonnet-4.6",
  "temperature": 0.2,
  "maxTokens": 65536
}
```

**注意点:** Cline はコード補完機能を持たないため、補完が必要な場合は別途 VSCode の組み込み補完や GitHub Copilot Free を使う必要があります。

### 方法C: Continue.dev + OpenRouter

Continue.dev は OSS の VSCode / JetBrains AI アシスタントです。`.continuerc.json` にプロバイダ設定を記述します。

```json
// .continuerc.json
{
  "models": [
    {
      "title": "OpenRouter Sonnet",
      "provider": "openrouter",
      "model": "anthropic/claude-sonnet-4.6",
      "apiKey": "sk-or-v1-xxxxxxxxxxxx",
      "baseUrl": "https://openrouter.ai/api/v1"
    }
  ],
  "tabAutocompleteModel": {
    "title": "Starcoder",
    "provider": "openrouter",
    "model": "deepseek/deepseek-v4-pro",
    "apiKey": "sk-or-v1-xxxxxxxxxxxx"
  }
}
```

Continue.dev はチャット型の対話がメインで、自律エージェント（ファイル編集・コマンド実行）は Cline ほど強力ではありません。

### 方法D: OpenCode（CLI）+ OpenRouter

[OpenCode](https://opencode.ai) は Claude Code にインスパイアされた CLI エージェントです。OpenRouter の API キーを環境変数で直接指定します。

```bash
# OpenRouter の API キーを設定
export OPENROUTER_API_KEY="sk-or-v1-xxxxxxxxxxxx"

# OpenCode を起動（モデル指定）
opencode --model openrouter:anthropic/claude-sonnet-4.6
```

CLI で動作するため VSCode 拡張としての統合度は低いですが、**コマンドラインでの高速な反復作業**に優れています。OpenCode のエージェントループ機構については「[ループエンジニアリングという新しい開発パラダイム](/argon/posts/loop-engineering-paradigm-shift/)」で詳しく解説しています。

### 実データで見るパフォーマンス比較

Reddit の各コミュニティで実際に行われたベンチマークテストの結果を見てみましょう。

#### リファクタリングタスク（10K行 Electron + React アプリ）

| ツール + モデル | 時間 | 費用 | API 呼び出し | 変更ファイル数 |
|---|---|---|---|---|
| Claude Code + Sonnet 4.6 | ~15分 | $3.85 | 136回 | 2ファイル |
| OpenCode + Sonnet 4.6 | ~15分 | $3.18 | 157回 | 8ファイル |
| **OpenCode + Codex 5.3** | **~7分** | **$1.44** | **79回** | **16ファイル** |
| OpenCode + Gemini 2.5 Pro | ~9分 | $1.88 | 92回 | 11ファイル |

出典: [reddit r/opencodeCLI](https://reddit.com/r/opencodeCLI) の実検証（2026年6月）

**注目ポイント:**

- **OpenCode + Codex 5.3** が時間・費用・変更範囲の3冠
- Claude Code は変更範囲が2ファイルに留まり、同じモデル（Sonnet 4.6）でも OpenCode の方が広範囲をカバー
- ツールの設計思想の違いが結果に大きく影響

> **データの信頼性について**: 上記のリファクタリングタスクの数値は reddit r/opencodeCLI のユーザー投稿を、フルスペック実装タスクの数値は reddit r/codex のユーザー投稿をそれぞれ引用しています。Reddit のアクセス制限により第三者検証ができないため、参考値としてご覧ください。実際のパフォーマンスはタスク内容・プロンプト・モデルバージョンにより変動します。

#### フルスペック実装タスク（React + アクセシビリティ + テスト）

| ツール + モデル | 時間 | 仕様遵守 |
|---|---|---|
| Claude Code + Sonnet 4.5 | 6分48秒 | △ 仕様取りこぼしあり |
| Codex CLI + Sonnet 4.5 | 10分14秒 | ◯ 全要件達成 |

出典: [reddit r/codex](https://reddit.com/r/codex) の実検証（2026年6月）

Codex CLI は時間こそかかるものの、仕様を漏らさず堅実に実装する傾向があります。

### モデル価格比較（OpenRouter 2026年6月時点）

| モデル | input/1M tokens | output/1M tokens |
|---|---|---|
| DeepSeek V4 Pro | $0.435 | $0.87 |
| GPT-5.3-Codex | $1.75 | $14.00 |
| Claude Sonnet 4.6 | $3.00 | $15.00 |
| Claude Opus 4.7 | $5.00 | $25.00 |
| Gemini 2.5 Pro | $1.25 | $10.00 |

コスパ重視なら **DeepSeek V4 Pro**（outputで **Codex 5.3 の約16分の1、Sonnet 4.6 の約17分の1のコスト**）、品質重視なら **Sonnet 4.6** か **Codex 5.3** を選ぶとよいでしょう。

関連記事: [Oracle Cloud 無料枠で OpenCode を動かす](/argon/posts/oracle-cloud-free-tier-opencode/)

### 総合比較: Claude Code / Codex CLI / VSCode + OpenRouter

| 観点 | Claude Code | Codex CLI | VSCode + OpenRouter（推奨）|
|---|---|---|---|
| VSCode 統合度 | 拡張あり（別窓感） | CLI のみ | ◎ ネイティブ |
| モデル選択肢 | Claude のみ | OpenRouter 互換 | ◎ 400+ モデル |
| コード補完 | ❌ なし | ❌ なし | ◯ あり（Copilot連携）|
| 自律エージェント | ◎ 強力 | ◎ 強力 | ◯ エージェントモード |
| コスト | △ 高め | ◯ 月額$10〜 | ◎ BYOK で最適化 |
| 仕様遵守 | △ 取りこぼし | ◯ 堅実 | ◯ モデル次第 |
| 変更範囲 | △ 狭い | ◯ 適切 | ◎ 調整可 |

## まとめ

ここまでの比較を総合すると、用途別の推奨は以下のようになります。

| ユースケース | おすすめ構成 |
|---|---|
| コード補完 + エージェントの両立 | **GitHub Copilot Pro + OpenRouter BYOK** |
| コスパ最優先・CLI派 | **OpenCode + OpenRouter（Codex 5.3 or DeepSeek V4 Pro）** |
| VSCode 完結・無料から始めたい | **Cline + OpenRouter** |
| 堅実な仕様遵守・品質重視 | **Codex CLI** |
| チャット型対話が好み | **Continue.dev + OpenRouter** |

OpenRouter の強みは「モデルを自由に選べること」に尽きます。特定のツールに依存せず、タスクごとに最適なモデルを選択できる柔軟性が、2026年の AI コーディング環境では最も重要な要素です。

まずは **GitHub Copilot Pro + OpenRouter BYOK** で始めて、コストが気になり始めたら **OpenCode + DeepSeek V4 Pro** を試す——そんなステップアップが現実的な道筋でしょう。

---

*本記事で引用したベンチマークデータは reddit r/opencodeCLI および r/codex のユーザー投稿に基づきます（2026年6月時点）。価格データは OpenRouter 公式サイト（openrouter.ai/models）の2026年6月26日時点の情報に基づきます。実際のパフォーマンスや価格はタスク内容・モデルバージョン・プロバイダの価格変更により変動します。*
