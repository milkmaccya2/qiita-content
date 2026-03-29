---
title: CLAUDE.mdを置くだけでノーコード自動化サーバーを作る — Cloud Scheduled Tasksのハック的活用法
tags:
  - 自動化
  - AI
  - MCP
  - Notion
  - Claude
private: true
updated_at: '2026-03-29T21:26:00+09:00'
id: cdc48d29ed5345a2377d
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

GitHubリポジトリにソースコードは1行もありません。あるのは `CLAUDE.md` という「自然言語の指示書」が1ファイルだけ。

たったこれだけで、**「6時間ごとにNotionの未要約メモを検出し、内容を読んで300字以内で要約を書き込む」** という定期バッチ処理がAnthropicのクラウド上で動き続けています。

```text
notion-summary-task/
└── CLAUDE.md   ← これだけ（プロンプトが書かれている）
```

今回使ったのは、Claude（Pro/Max）に搭載されている **「Cloud Scheduled Tasks」** という機能です。

本来は「定期的に依存関係をアップデートする」「テストを回す」といった開発ワークフローのための機能ですが、これを**ZapierやMakeのような「ノーコードiPaaS」としてハック**してみたところ、最高に便利でした。

この記事では、**「Claude Pro/Maxに課金しているなら絶対に使い倒すべき、最強の自動化基盤の作り方」** を、Notionの自動要約を例に紹介します。

## 背景：Claudeの元を全力で取る

僕は生活や技術のメモをすべてNotionのInboxに投げており、「溜まったメモをAIに定期的に要約してほしい」と思っていました。

これを実現する手段としてNotion AI（[Businessプラン月$20/member](https://www.notion.com/ja/product/ai)）は非常に優秀です。メモを書いた瞬間にリアルタイムで要約が生成され、ユーザーの操作は一切不要。正直、この体験を手放すのは怖かったです。

しかし、僕はすでにClaude Max（月$100）に課金しています。**「すでに優秀なLLMに課金しているのに、同じような機能のために別のAIオプションや自動化SaaSに二重で課金するのは悔しい」** という思いがありました。

そこで、「Claudeのサブスクに含まれている機能をハックして、追加コストゼロで完全自動化の仕組みを作れないか？」と考えたのが今回のスタートです。

## AI自動化アーキテクチャの比較

定期的なAI処理（今回の例ならNotion要約）を構築するためのアーキテクチャを比較します。

| アーキテクチャ | 構築の手軽さ | 運用・メンテ | 追加コスト | 柔軟性 |
|---|:-:|:-:|:-:|:-:|
| 特定SaaSのAI機能（例: Notion AI） | ◎ 設定のみ | ◎ 不要 | △ 月$20など | × そのSaaS内限定 |
| ノーコード自動化ツール（Dify / Make / n8n等） | ○ GUIベース | ○ 比較的楽 | △ 月$0〜20+API代 | ◎ 高い |
| 自作スクリプト（GAS / Actions） | △ コード開発 | × API変更で壊れる | ◎ ほぼAPI代のみ | ◎ 高い |
| **Cloud Scheduled Tasks（今回のハック）** | **○ 自然言語** | **◎ プロンプト修正** | **◎ Pro/Max内包（追加$0）※** | **○ MCP対応なら何でも** |

※ Cloud Scheduled TasksはClaude Pro/Maxプランのサブスクリプションに含まれており、追加課金なしで利用できます。筆者はMaxプラン契約者のため、この記事で紹介する構成は追加費用ゼロで実現しています。

表の通り、Cloud Scheduled Tasksは **「ノーコードツール並みの手軽さ」を持ちながら、「追加コストゼロ（サブスク内包）」かつ「スクリプト保守の苦労なし」** という、Pro/Max契約者にとって見逃せない選択肢です。

## なぜ「空のリポジトリ」なのか？（今回のハックの仕組み）

Cloud Scheduled TasksはGitHubリポジトリと紐付けて動きます。実行のたびにリポジトリをクローンし、その中のコードに対して処理を行うのが本来の使い方です。

しかし、claude.aiのWeb版には [Connectors](https://support.claude.com/en/articles/11176164-pre-built-web-connectors-using-remote-mcp) と呼ばれるリモートMCP統合機能が組み込まれています。Notion、Slack、Linearなど50以上の外部サービスとOAuth認証で接続でき、Cloud Scheduled Tasksからもそのまま利用できます。

つまり、「リポジトリ内のコードに対する処理」を一切させず、**「MCP経由で外部APIを叩いてデータを処理させる」ことだけに特化させれば、実質的に高度なノーコード自動化サーバーとして機能する**わけです。

これが、コードゼロ・CLAUDE.mdのみの空リポジトリを用意した理由です。自分でAPIクライアントを書かないため、Notion APIの破壊的変更があっても自分がコードを直す必要はありません。MCPコネクタ側が追従してくれます。

## セットアップ手順（Notion自動要約の例）

### 1. リポジトリを用意する

設定ファイル置き場としてのリポジトリを作ります。必要なのは `CLAUDE.md` 1ファイルだけです。

### 2. CLAUDE.mdを書く

タスクの実行コンテキストとして、Notion DBの情報と要約ルールを自然言語で書きます。Zapierでいうところのワークフロー設定です。

```markdown
# Notion 要約タスク

## 概要
Notion Inbox DBの「要約」カラムが空のレコードを検出し、
コンテンツを読み取って300字以内の日本語で要約を生成・書き込む。

## Notion DB 情報
- DB ID: `xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`
- ビューURL: `view://xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`

## プロパティ
| プロパティ | 型 | 説明 |
|-----------|------|------|
| 名前 | title | ページタイトル |
| 日付 | created_time | 作成日 |
| 要約 | text | 要約テキスト（ここに書き込む） |

## 要約ルール
- 日本語で300字以内
- ページの本文を読み取って要約する
- 要約カラムが空のレコードのみ対象（既存のものはスキップ）
```

### 3. GitHubにpushする

```bash
git init
git add CLAUDE.md
git commit -m "initial commit"
git remote add origin https://github.com/yourname/notion-summary-task.git
git push -u origin main
```

### 4. スケジュールタスクを作成する

[claude.ai/code/scheduled](https://code.claude.com/docs/en/web-scheduled-tasks) のWeb UIから「新しいスケジュールタスク」を作成します。

<!-- TODO: スクリーンショット1 - claude.ai/code/scheduled のタスク一覧画面 -->
<!-- TODO: スクリーンショット2 - タスク設定画面全体 -->

設定内容は以下の通りです。

| 項目 | 設定値 |
|------|--------|
| **プロンプト** | CLAUDE.mdを参照し、Notion DBの要約カラムが空のレコードを検出して本文を読み取り、要約を書き込め。 |
| **モデル** | Opus 4.6（用途に応じてSonnetでも可） |
| **リポジトリ** | `yourname/notion-summary-task` |
| **スケジュール** | `0 */6 * * *`（6時間ごと） |
| **コネクタ** | Notionを追加（OAuth認可） |

プロンプトが1行で済むのは、詳細をすべて `CLAUDE.md` に書いているからです。条件を変えたい時は `CLAUDE.md` を更新してpushするだけで反映されます。

なお、Notionコネクタ接続時のOAuth認可では、アクセスを許可するページ・データベースの範囲を選択できます。プライベートなメモをクラウドで処理することに懸念がある場合は、対象のDBだけにアクセスを絞るとよいでしょう。

<!-- TODO: スクリーンショット3 - Notionコネクタ接続時のOAuth認可画面 -->

## 動作結果

スケジュール通りにクラウドで実行され、要約カラムが空だったレコードに要約が書き込まれます。PCを閉じていても、寝ていても確実に動きます。

<!-- TODO: スクリーンショット4 - 要約が埋まったNotionのDB画面（Before/After） -->

実行の過程（セッションログ）はWeb UIから確認可能です。

<!-- TODO: スクリーンショット5 - タスク実行履歴/セッション画面 -->

## 応用アイデア：他に何をさせるか？

今回はNotionの自動要約を例にしましたが、claude.aiのConnectorsが対応しているサービスであれば同じ手法が使えます。以下のような「普段ならDifyやMakeでワークフローを組んだり、専用スクリプトを書いたりする処理」も、自然言語の指示書を置くだけで定期実行できます。

- **Slackのデイリー要約**: 毎日18時に #times や特定プロジェクトのチャンネルを読み取り、重要な決定事項だけを箇条書きにして別のチャンネルに投稿する
- **GitHub Issueの自動トリアージ**: 毎日、未分類のIssueをチェックし、内容を読んで適切なラベル（bug, enhancement など）を自動で付与する
- **Linearのタスク整理**: 放置されている古いタスクを検出し、担当者に状況確認のコメントを自動で残す

ロジックの構築（条件分岐やデータ抽出）をすべてLLMの知能に丸投げできるため、GUIでフィルターやパスを組むよりもシンプルです。

## まとめ

本質は **「Cloud Scheduled Tasksは、自然言語で書けるノーコード自動化基盤として使える」** ということです。

- **究極のノーコード**: ロジックも条件分岐も自然言語（CLAUDE.md）で書ける
- **メンテナンスフリー**: 外部APIの仕様変更はMCPコネクタ側が吸収してくれる
- **追加課金なし**: Pro/Maxプランの範囲内で完結する

「スクリプトを書いてcronで回す」時代から、「空のリポジトリに指示書だけ置いてAIに定期実行させる」時代へ。Claude課金勢の皆様は、ぜひ手元の自動化タスクをこの方法に置き換えてみてください。

## 参考文献

- [Schedule tasks on the web - Claude Code Docs](https://code.claude.com/docs/en/web-scheduled-tasks)
- [Run prompts on a schedule - Claude Code Docs](https://code.claude.com/docs/en/scheduled-tasks)
- [Manage costs effectively - Claude Code Docs](https://code.claude.com/docs/en/costs)
- [Notion AI](https://www.notion.com/ja/product/ai)
- [Notion MCP - Official Docs](https://developers.notion.com/docs/mcp)
- [claude.ai Connectors](https://support.claude.com/en/articles/11176164-pre-built-web-connectors-using-remote-mcp)
- [Google Apps Script Quotas](https://developers.google.com/apps-script/guides/services/quotas)
