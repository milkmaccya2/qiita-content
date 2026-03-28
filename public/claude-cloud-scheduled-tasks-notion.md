---
title: ClaudeのクラウドスケジュールタスクでNotionの自動要約を実現する
tags:
  - 自動化
  - AI
  - MCP
  - Notion
  - ClaudeCode
private: true
updated_at: '2026-03-28T22:51:48+09:00'
id: cdc48d29ed5345a2377d
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

僕は生活・育児・技術のメモを、すべてNotionのInboxにポイポイ入れています。日々溜まっていくメモは、後から探すときに「このページ何だったっけ？」となりがちです。要約があればパッと中身がわかるのですが、毎回手動で書くのは面倒。

以前はNotion AIの自動要約機能を使っていました。メモを書いたら裏側で勝手に要約してくれる、とても便利な機能です。ただ、これには追加の課金が必要でした。

Claude Code（Pro/Max）を契約している身としては、「これ、Claude Codeで代替できないか？」と考えるのは自然な流れです。いくつかの方法を試した結果、**クラウドスケジュールタスク**という機能に辿り着きました。

この記事では、Notionの自動要約を実現する方法を比較検討し、Cloud Scheduled Tasksでの構築手順を紹介します。

## やりたいこと

要件はシンプルです。

- Notion DBの各レコードの**コンテンツを読み取って、300字以内で要約**する
- 要約カラムが空のレコードだけを対象にする
- **定期的に自動実行**する（PCの起動状態に依存しない）
- できれば**コードを書かずに**済ませたい

## 方法の比較検討

Notionの自動要約を実現する方法は複数あります。それぞれの特徴を整理しました。

### 比較表

| 方法 | 構築の手軽さ | メンテナンス | コスト | 即時性 | 必要スキル |
|------|:-----------:|:----------:|:-----:|:-----:|-----------|
| Notion AI | ◎ 設定のみ | ◎ 不要 | △ 月$20/member | ◎ リアルタイム | 不要 |
| GitHub Actions + Claude API | △ コード開発 | × コード修正 | ◎ API従量課金 | △ cron依存 | 開発力 |
| GAS + AI API | △ コード開発 | × コード修正 | ◎ API従量課金 | △ トリガー依存 | 開発力 |
| Zapier / Make | ○ GUI設定 | ○ GUI再設定 | △ 月$9〜20 | ○ トリガー対応 | 低 |
| **Cloud Scheduled Tasks** | **○ 自然言語** | **◎ プロンプト修正** | **◎ Pro/Max内 ※** | **△ 最小1時間** | **Git基本操作** |

※ Cloud Scheduled TasksはClaude Pro/Maxプランのサブスクリプションに含まれており、追加課金なしで利用できます。筆者はMaxプラン契約者のため、この記事で紹介する構成は追加費用ゼロで実現しています。

### Notion AI

正直に言うと、Notion AIの自動要約は素晴らしいです。メモを書いた瞬間にリアルタイムで要約が生成され、ユーザーの操作は一切不要。イベント駆動で動くため、「書いたら勝手に要約されている」という体験は他の方法では再現できません。正直、この体験を手放すのは怖かったです。

ただし、完全なAI機能（自動要約含む）を使うには[Businessプラン（月$20/member）](https://www.notion.com/ja/product/ai)が必要です。Claude Pro/Maxを既に契約している場合、同じことがほぼ実現できるのに追加で月$20を払い続けるかは悩みどころです。リアルタイム性が必須でなければ、数時間ごとのバッチ処理で十分という判断もあります。

### GitHub Actions + Claude API

GitHub Actionsのcronトリガーで定期実行し、スクリプト内でNotion APIとClaude APIを呼び出す構成です。Claude API（Haiku）なら月$0.16程度と非常に安い。ただし、**スクリプトを書く必要がある**のと、Notion APIやClaude APIに破壊的変更があればコードの修正が必要です。なお、GitHub Actionsのcronは15〜30分程度の遅延が発生することがありますが、要約用途ではほぼ問題にならないでしょう。

### GAS（Google Apps Script）+ AI API

GASのトリガーで定期実行し、Notion APIとAI API（Gemini/Claude）を呼び出す構成。GitHub Actionsと似ていますが、**[単一実行6分の制限](https://developers.google.com/apps-script/guides/services/quotas)**があります。処理対象が多いと時間切れになるリスクがあり、安定運用には工夫が必要です。

### Zapier / Make

ノーコードで構築できます。GUIでNotion→AI要約→Notion書き戻しのワークフローを組めます。ただし、Zapierなら月$19.99〜、Makeなら月$9〜の**別途課金**が発生します。Notion AIの課金をやめたのに別のSaaSに課金するのは本末転倒です。

### Cloud Scheduled Tasks

Claude Code（Pro/Max）に含まれるクラウド定期実行機能です。**自然言語のプロンプトでタスクを定義**し、MCPコネクタ経由でNotionと連携します。Pro/Max契約者なら追加費用ゼロ。自分でAPIを叩くコードを書かないため、APIの変更に自分で追従する必要がありません。メンテナンスが必要になるとすればプロンプトの微修正程度です。ただし、GitHubリポジトリの作成やCLAUDE.mdの記述など、最低限のGit操作は必要です。

### 比較のポイント: メンテナンスコスト

初期構築の手間だけでなく、**運用後のメンテナンスコスト**が重要です。

スクリプト型（GitHub Actions、GAS）は、APIの破壊的変更があればコードを書き換える必要があります。Notion APIもClaude APIもアクティブに更新されているため、これは現実的なリスクです。

一方、Cloud Scheduled Tasksは自然言語でタスクを定義しています。自分でAPIクライアントのコードを書いていないため、APIの仕様変更があっても自分でコードを修正する必要がありません。MCPコネクタやClaude自身のアップデートで対応されるケースが多く、自分の作業としてはプロンプトの微修正で済みます。もちろんMCPコネクタ側に不具合が出る可能性はゼロではありませんが、自前のスクリプトを保守し続けるよりは圧倒的に楽です。

## Cloud Scheduled Tasksとは

Claudeには3つのスケジューリング方法があります（[公式ドキュメント](https://code.claude.com/docs/en/web-scheduled-tasks)）。

| | Cloud Scheduled Tasks | Desktop Scheduled Tasks | /loop |
|---|---|---|---|
| 実行環境 | Anthropicクラウド | ローカルPC | ローカルPC |
| 利用に必要なもの | なし | Desktopアプリ起動中 | CLIセッション維持 |
| 最小間隔 | 1時間 | 1分 | 1分 |

実は最初、Claude Coworkのスケジュール機能（Desktop Scheduled Tasks）を使っていました（[参考](https://support.claude.com/en/articles/13854387-schedule-recurring-tasks-in-cowork)）。これはローカルPCで動くため、PCを閉じている間は実行されません。夜間や外出中に要約が走らないのが不便で、Cloud Scheduled Tasksに移行しました。

Cloud Scheduled TasksはAnthropicのクラウドインフラで実行されるため、PCの状態に一切依存しません。設定したスケジュール通りに、確実に動きます。

なお、Cloud Scheduled Tasksの作成・管理は `claude.ai/code/scheduled` のWeb UIから行えます。claude.aiにはNotion、Slack、Linearなどの[MCPコネクタ](https://developers.notion.com/docs/mcp)が組み込まれており、OAuthで接続するだけでタスクから利用できます。

## セットアップ手順

### 1. リポジトリを用意する

Cloud Scheduled TasksはGitHubリポジトリが必須です。実行のたびにデフォルトブランチからクローンし、その中のCLAUDE.mdをコンテキストとして読み込む仕組みになっています。つまり、リポジトリはコードを置く場所ではなく、タスクの設定ファイル置き場として使います。

今回はコード変更が目的ではないので、最小限のリポジトリを作ります。

```
notion-summary-task/
├── CLAUDE.md
└── README.md
```

### 2. CLAUDE.mdを書く

タスクの実行コンテキストとして、CLAUDE.mdにNotion DBの情報と要約ルールを書いておきます。

```markdown
# Notion 要約タスク

## 概要
Notion Inbox DBの「要約」カラムが空のレコードを検出し、
コンテンツを読み取って300字以内の日本語で要約を生成・書き込むタスク。

## Notion DB 情報
- DB ID: `xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`
- ビューURL: `view://xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`

## プロパティ
| プロパティ | 型 | 説明 |
|-----------|------|------|
| 名前 | title | ページタイトル |
| 日付 | created_time | 作成日 |
| 要約 | text | 要約テキスト（このカラムに書き込む） |
| URL | url | 元のURL |
| タグ | multi_select | カテゴリタグ |

## 要約ルール
- 日本語で300字以内
- ページのコンテンツ（本文）を読み取って要約する
- 要約カラムが空のレコードのみ対象
- 既に要約が入っているレコードはスキップ
```

ポイントは、**プロンプトではなくCLAUDE.mdに詳細を書く**ことです。プロンプトはシンプルに「CLAUDE.mdを参照して実行しろ」とだけ書けば、毎回のクローン時にCLAUDE.mdが読み込まれます。

### 3. GitHubにpushする

```bash
git init
git add CLAUDE.md README.md
git commit -m "initial commit"
git remote add origin https://github.com/yourname/notion-summary-task.git
git push -u origin main
```

### 4. スケジュールタスクを作成する

`claude.ai/code/scheduled` にアクセスし、「新しいスケジュールタスク」を作成します。

<!-- TODO: スクリーンショット1 - claude.ai/code/scheduled のタスク一覧画面 -->
<!-- TODO: スクリーンショット2 - タスク設定画面全体 -->

設定内容は以下の通りです。

**名前**: `notion要約タスク`

**プロンプト**:
```
CLAUDE.mdを参照し、Notion Inbox DBの要約カラムが空のレコードを検出して
コンテンツを読み取り、300字以内の日本語で要約を書き込め。
```

**モデル**: Opus 4.6（1Mコンテキスト）
※ 300字要約だけならSonnet 4.6でも十分です。今回はNotionページの本文が長いケースも想定してOpusを選びましたが、コストを抑えたい場合はSonnetを選択するとよいでしょう。

**リポジトリ**: `yourname/notion-summary-task`

**スケジュール**: カスタムcron `0 */6 * * *`（6時間ごと）

**コネクタ**: Notion

プロンプトがこれだけで済むのは、CLAUDE.mdにすべての詳細を書いているからです。タスクの修正が必要になったときも、CLAUDE.mdを更新してpushするだけで反映されます。

### 5. Notionコネクタの接続

コネクタの「Notion」を追加すると、OAuthの認可フローが走ります。Notionのワークスペースを選択して権限を許可すれば、以降はCloud Scheduled Tasksから自動的にNotionにアクセスできます。

なお、OAuthの認可時にアクセスを許可するページ・データベースの範囲を選択できます。プライベートなメモをクラウドで処理することに懸念がある場合は、対象のDBだけにアクセスを絞るとよいでしょう。

<!-- TODO: スクリーンショット3 - Notionコネクタ接続時のOAuth認可画面 -->

## 動作結果

タスクが実行されると、要約カラムが空だったレコードに要約が書き込まれます。

<!-- TODO: スクリーンショット4 - 要約が埋まったNotionのDB画面（Before/After） -->

実行履歴はタスクの詳細ページから確認できます。各実行はセッションとして記録され、何をしたかを後から確認できます。

<!-- TODO: スクリーンショット5 - タスク実行履歴/セッション画面 -->

## まとめ

Notionの自動要約を実現する方法はいくつかありますが、Claude Code（Pro/Max）契約者にとっては**Cloud Scheduled Tasks**が最もバランスの良い選択肢です。

- **コードを書かない**: 自然言語のプロンプトとCLAUDE.mdだけで定義
- **自分でメンテするコードがない**: API変更への追従はMCPコネクタ側で行われる
- **追加課金なし**: Pro/Maxプランのサブスクリプション内で利用可能

こんな人に向いています:
- Claude Code（Pro/Max）を契約している
- スクリプトを書くより自然言語で済ませたい
- PCを常時起動しておきたくない

逆に、Claude Codeを契約していない場合やAPIコストを極限まで抑えたい場合は、GitHub Actions + Claude API（Haiku）が最もコスパの良い選択肢です。ノーコードで手軽に始めたいならZapierやMakeも検討の価値があります。

自然言語でタスクを定義して、クラウドで自動実行する。スクリプトを書いてcronで回していた時代と比べると、自動化のハードルがかなり下がったと感じます。

## 参考文献

- [Schedule tasks on the web - Claude Code Docs](https://code.claude.com/docs/en/web-scheduled-tasks)
- [Run prompts on a schedule - Claude Code Docs](https://code.claude.com/docs/en/scheduled-tasks)
- [Use Claude Code Desktop](https://code.claude.com/docs/en/desktop)
- [Manage costs effectively - Claude Code Docs](https://code.claude.com/docs/en/costs)
- [Notion AI](https://www.notion.com/ja/product/ai)
- [Notion MCP - Official Docs](https://developers.notion.com/docs/mcp)
- [Schedule recurring tasks in Cowork - Claude Help Center](https://support.claude.com/en/articles/13854387-schedule-recurring-tasks-in-cowork)
- [Google Apps Script Quotas](https://developers.google.com/apps-script/guides/services/quotas)
