---
title: CLAUDE.mdを置くだけでノーコード自動化サーバーを作る — Cloud Scheduled Tasksのハック的活用法
tags:
  - 自動化
  - AI
  - MCP
  - Notion
  - Claude
  - CloudScheduledTasks
private: true
updated_at: '2026-03-29T21:55:25+09:00'
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
└── CLAUDE.md   ← これだけ（自然言語の指示が書かれている）
```

今回使ったのは、2026年2月にリリースされた **[Cloud Scheduled Tasks](https://code.claude.com/docs/en/web-scheduled-tasks)** という機能です（Pro/Max/Team/Enterpriseで利用可能）。

本来はリポジトリのコードに対して「依存関係をアップデートする」「テストを回す」といった開発ワークフローを定期実行するための機能です。しかし、**コードを一切置かず、CLAUDE.mdという指示書だけのリポジトリでMCPコネクタ経由の外部サービス操作に特化させる**ことで、DifyやMakeのようなノーコード自動化基盤として活用できます。これが今回の「ハック」です。

この記事では、Notionの自動要約を例に、この構成の作り方を紹介します。

## 背景：Claudeの元を全力で取る

僕は生活や技術のメモをすべてNotionのInboxに投げており、「溜まったメモをAIに定期的に要約してほしい」と思っていました。

これを実現する手段としてNotion AI（[Businessプラン月$20/member](https://www.notion.com/ja/product/ai)）は非常に優秀です。メモを書いた瞬間にリアルタイムで要約が生成され、ユーザーの操作は一切不要。正直、この体験を手放すのは怖かったです。

しかし、僕はすでにClaude Maxに課金しています。**「すでに優秀なLLMに課金しているのに、同じような機能のために別のAIオプションや自動化SaaSに二重で課金するのは悔しい」** という思いがありました。

そこで、「Claudeのサブスクに含まれている機能を活用して、追加コストゼロで完全自動化の仕組みを作れないか？」と考えたのが今回のスタートです。

## AI自動化アーキテクチャの比較

定期的なAI処理（今回の例ならNotion要約）を構築するためのアーキテクチャを比較します。

| アーキテクチャ | 構築の手軽さ | 運用・メンテ | 追加コスト | 柔軟性 |
|---|:-:|:-:|:-:|:-:|
| 特定SaaSのAI機能（例: Notion AI） | ◎ 設定のみ | ◎ 不要 | △ 月$20など | × そのSaaS内限定 |
| ノーコード自動化ツール（Dify / Make / n8n等）※1 | ○ GUIベース | ○ 比較的楽 | △ 無料〜月$20+API代 | ◎ 高い |
| 自作スクリプト（GAS / Actions） | △ コード開発 | △ API変更時にコード修正 | ◎ ほぼAPI代のみ | ◎ 高い |
| **Cloud Scheduled Tasks（今回の構成）** | **○ 自然言語** | **○ プロンプト修正 ※2** | **◎ Pro/Max内包（追加$0）※3** | **○ Connectors対応サービス** |

※1 Difyやn8nはセルフホストなら無料。Makeは月$9〜、クラウド版n8nは月€24〜。
※2 MCPコネクタ側の更新で多くの変更は吸収されますが、Notion DB側のスキーマ変更やコネクタ自体の不具合時はプロンプトやCLAUDE.mdの修正が必要になる場合があります。
※3 Cloud Scheduled TasksはPro、Max、Team、Enterpriseの全プランで利用可能です。ただしProプランは使用量の上限がMaxより低いため、高頻度の実行にはMaxの方が適しています。筆者はMaxプラン契約者です。

## なぜ「空のリポジトリ」なのか？

Cloud Scheduled TasksはGitHubリポジトリと紐付けて動きます。実行のたびにリポジトリをクローンし、その中のコードに対して処理を行うのが本来の使い方です。

しかし、claude.aiのWeb版には [Connectors](https://support.claude.com/en/articles/11176164-pre-built-web-connectors-using-remote-mcp) と呼ばれるリモートMCP統合機能が組み込まれています。Notion、Slack、Linear、Asana、Figmaなどの外部サービスとOAuth認証で接続でき、Cloud Scheduled Tasksからもそのまま利用できます（[対応サービス一覧](https://support.claude.com/en/articles/11176164-pre-built-web-connectors-using-remote-mcp)）。

つまり、「リポジトリ内のコードに対する処理」を一切させず、**「Connectors経由で外部サービスのデータを処理させる」ことだけに特化させれば、実質的にノーコード自動化基盤として機能する**わけです。

これが、コードゼロ・CLAUDE.mdのみのリポジトリを用意した理由です。自分でAPIクライアントを書かないため、Notion APIの仕様変更があっても自分がコードを直す必要はありません。Connectors側のアップデートで対応されるケースが多いです（ただしSLAが保証されているわけではないので、Connectors自体に不具合が出る可能性はゼロではありません）。

なお、クローンされたリポジトリ内の `CLAUDE.md` は、Claude Codeのセッション開始時に自動的にコンテキストとして読み込まれます。これはClaude Codeの標準動作で、Cloud Scheduled Tasksでも同様です。だからこそ、プロンプトを1行にして詳細はすべて `CLAUDE.md` に書くという構成が成り立ちます。

## セットアップ手順（Notion自動要約の例）

### 1. リポジトリを用意する

設定ファイル置き場としてのリポジトリを作ります。必要なのは `CLAUDE.md` 1ファイルだけです。

### 2. CLAUDE.mdを書く

タスクの実行コンテキストとして、Notion DBの情報と要約ルールを自然言語で書きます。ノーコード自動化ツールでいうところのワークフロー設定にあたります。

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

:::note info
**DB IDとビューURLの取得方法**

- **DB ID**: Notionでデータベースをブラウザで開き、URLの `notion.so/` 直後の32文字がDB IDです（例: `https://notion.so/2b93611beab880fc8a4cfdb981797149`）。
- **ビューURL**: データベースのビューを切り替えたときのURLパラメータ `?v=` 以降がビューIDです。CLAUDE.mdには `view://` プレフィックスをつけた形式（`view://xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`）で記載します。これはNotion MCPがビューを指定するための記法です。ビューを使わずDB全体を対象にする場合はDB IDだけで構いません。
:::

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

<!-- TODO: スクリーンショット1
  撮影対象: claude.ai/code/scheduled のページ全体（タスク一覧が見える状態）
  目的: 読者に「ここからタスクを作る」という入口を見せる
  撮影手順: claude.ai/code/scheduled にアクセスし、ページ全体をキャプチャ
-->

<!-- TODO: スクリーンショット2
  撮影対象: タスク設定画面（既にお持ちの設定画面スクショが使えます）
  目的: プロンプト・モデル・リポジトリ・スケジュール・コネクタの設定項目を見せる
  撮影手順: notion要約タスクの編集画面を開いてキャプチャ（DB IDなどの個人情報はモザイク）
-->

設定内容は以下の通りです。

| 項目 | 設定値 |
|------|--------|
| **プロンプト** | CLAUDE.mdを参照し、Notion DBの要約カラムが空のレコードを検出して本文を読み取り、要約を書き込め。 |
| **モデル** | Opus 4.6（300字要約程度ならSonnet 4.6でも十分。本文が長いページが多い場合はOpusが安定） |
| **リポジトリ** | `yourname/notion-summary-task` |
| **スケジュール** | `0 */6 * * *`（6時間ごと） |
| **コネクタ** | Notionを追加（OAuth認可） |

プロンプトが1行で済むのは、詳細をすべて `CLAUDE.md` に書いているからです。条件を変えたい時は `CLAUDE.md` を更新してpushするだけで、次回実行から反映されます。

### 5. Notionコネクタの接続

コネクタの「Notion」を追加すると、OAuthの認可フローが走ります。認可画面ではページピッカーが表示され、**アクセスを許可するページ・データベースを個別に選択**できます（[Notion公式: Authorization](https://developers.notion.com/guides/get-started/authorization)）。ワークスペース全体ではなく、対象のDBだけにアクセスを絞ることが可能です。

<!-- TODO: スクリーンショット3
  撮影対象: Notionコネクタ接続時のOAuth認可画面（ページピッカーが表示されている状態）
  目的: 読者にアクセス範囲を選択できることを視覚的に示す
  撮影手順: コネクタ設定でNotionを追加し、認可画面が表示されたらキャプチャ（個人情報はモザイク）
-->

## 動作結果

スケジュール通りにクラウドで実行され、要約カラムが空だったレコードに要約が書き込まれます。PCを閉じていても、寝ていても確実に動きます。

<!-- TODO: スクリーンショット4
  撮影対象: Notionのデータベース画面で、要約カラムが埋まっている状態
  目的: 「本当に動いた」という証拠を見せる。Before/Afterがわかるとベスト
  撮影手順: 要約が書き込まれたNotionのInbox DBをキャプチャ（メモの内容はモザイク推奨）
  GIFにできるなら: 要約カラムが空の状態 → タスク実行後に埋まった状態を連続で見せる
-->

実行の過程はタスク詳細ページからセッションログとして確認できます。Claudeが何をしたか（どのレコードを読み、どんな要約を書いたか）を後から追跡できるので、結果が想定と違った場合のデバッグにも使えます。

<!-- TODO: スクリーンショット5
  撮影対象: タスク詳細ページの実行履歴、またはセッションログ画面
  目的: 実行ログが残ることを示す（透明性・デバッグ可能性のアピール）
  撮影手順: claude.ai/code/scheduled からタスクをクリックし、実行履歴 or セッション画面をキャプチャ
-->

## 応用アイデア：他に何をさせるか？

今回はNotionの自動要約を例にしましたが、claude.aiの[Connectors](https://support.claude.com/en/articles/11176164-pre-built-web-connectors-using-remote-mcp)が対応しているサービスであれば同じ手法が使えます。以下のような「普段ならDifyやMakeでワークフローを組んだり、専用スクリプトを書いたりする処理」も、自然言語の指示書を置くだけで定期実行できます。

- **Slackのデイリー要約**: 毎日18時に特定チャンネルを読み取り、重要な決定事項だけを箇条書きにして別のチャンネルに投稿する
- **GitHub Issueの自動トリアージ**: 毎日、未分類のIssueをチェックし、内容を読んで適切なラベル（bug, enhancement など）を自動で付与する
- **Linearのタスク整理**: 放置されている古いタスクを検出し、担当者に状況確認のコメントを自動で残す

ロジックの構築（条件分岐やデータ抽出）をすべてLLMに委ねられるため、GUIでフィルターやパスを組むよりもシンプルです。

## まとめ

本質は **「Cloud Scheduled Tasksを、自然言語で定義するノーコード自動化基盤として使える」** ということです。

- **コードを書かない**: ロジックも条件分岐も自然言語（CLAUDE.md）で定義
- **自分でメンテするコードがない**: 外部APIの仕様変更はConnectors側のアップデートで対応されることが多い
- **追加課金なし**: Pro/Maxプランの範囲内で完結する（筆者はMaxプラン）

「スクリプトを書いてcronで回す」時代から、「空のリポジトリに指示書だけ置いてAIに定期実行させる」時代へ。Claudeに課金している方は、ぜひ手元の定型作業をこの方法で自動化してみてください。

## 参考文献

- [Schedule tasks on the web - Claude Code Docs](https://code.claude.com/docs/en/web-scheduled-tasks)
- [Run prompts on a schedule - Claude Code Docs](https://code.claude.com/docs/en/scheduled-tasks)
- [Manage costs effectively - Claude Code Docs](https://code.claude.com/docs/en/costs)
- [claude.ai Connectors（Pre-built web connectors）](https://support.claude.com/en/articles/11176164-pre-built-web-connectors-using-remote-mcp)
- [Notion AI](https://www.notion.com/ja/product/ai)
- [Notion MCP - Official Docs](https://developers.notion.com/docs/mcp)
- [Notion Authorization（OAuth）](https://developers.notion.com/guides/get-started/authorization)
- [Google Apps Script Quotas](https://developers.google.com/apps-script/guides/services/quotas)
