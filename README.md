# qiita-content

[Qiita CLI](https://github.com/increments/qiita-cli) で管理している Qiita 記事のリポジトリです。

`main` ブランチに push すると GitHub Actions 経由で Qiita に自動投稿・更新されます。

## 記事一覧

| ファイル | タイトル |
|---------|--------|
| [sento-ranking.md](public/sento-ranking.md) | 13年分のGoogle Mapsタイムラインから「自分の銭湯ランキング」を作る |

## セットアップ

```bash
npm install
npx qiita login
```

## 記事のプレビュー

```bash
npx qiita preview
```

http://localhost:8888 でプレビューできます。

## 新しい記事の作成

```bash
npx qiita new <記事名>
```

`public/<記事名>.md` が生成されます。
