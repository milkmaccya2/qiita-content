---
title: 銭湯の訪問記録をLeafletで地図に可視化するWebアプリを作った
tags:
  - JavaScript
  - Leaflet
  - GitHubPages
  - ClaudeCode
  - 銭湯
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: true
---

## はじめに

https://qiita.com/milkmaccya2/items/1e1ef669b9b5c5d78367

前回の記事で、13年分のGoogle Mapsタイムラインから銭湯・サウナの訪問履歴を抽出し、ランキングを作りました。76施設・354回という結果が出て満足していたのですが、ランキング表だけだと「どこに行ったか」の空間的な広がりが見えません。

地図にプロットしたら、行動圏の変化や旅先での銭湯訪問が可視化できて面白いのでは？と思い、Leafletを使った静的Webアプリを作ってみました。

![全体像：マーカー表示とサイドバー](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4384587/d3a4b5f5-4075-4e54-8c3c-d7aeb3bc717b.png)
*完成した銭湯マップ。訪問回数に応じてマーカーの大きさが変わる*

## 使った技術

| 技術 | 用途 | 選定理由 |
|------|------|----------|
| [Leaflet](https://leafletjs.com/) | 地図表示 | 無料、軽量、APIキー不要 |
| [Leaflet.heat](https://github.com/Leaflet/Leaflet.heat) | ヒートマップ | Leafletのプラグインとして簡単に導入できる |
| [CARTO Dark Matter](https://carto.com/basemaps/) | 地図タイル | ダークテーマでデータが映える。無料 |
| GitHub Pages | ホスティング | 静的サイトなのでこれで十分。`docs/`配置で設定するだけ |
| Python | データ前処理 | 前回のCSVをWebアプリ用のJSファイルに変換 |

地図ライブラリはGoogle Maps JavaScript APIも候補でしたが、APIキーが必要で読者が試しにくいためLeafletにしました。ビルドツールも使わず、HTML + JS + CSSのシンプルな構成です。

## データの準備

### 前回の出力を結合する

前回の記事で生成した2つのCSVを使います。

- `sento-visits.csv` — 全訪問記録（日付、施設名、緯度経度）
- `sento-ranking-with-urls.csv` — 施設ランキング（施設名、訪問回数、公式サイトURL）

訪問CSVには緯度経度があり、ランキングCSVには公式サイトURLがあるので、施設名をキーにして結合します。出力は `file://` で直接開いてもCORSエラーにならないよう、`const SENTO_DATA = [...]` というJSファイルとして生成しています。

```python
# 施設ごとに訪問日・座標を集約し、ランキング情報と結合してJSONを生成
result.append({
    "name_ja": "○○温泉",
    "lat": 35.xxxxx,
    "lng": 139.xxxxx,
    "total_visits": 43,
    "first_visit": "2018-07-21",
    "last_visit": "2023-02-04",
    "visits_by_year": {"2018": 8, "2019": 12, ...},
    "url": "https://example.com/",
})
```

### 名前の表記揺れへの対処

ここで1つハマりポイントがありました。ランキングCSVと訪問CSVで施設名の表記が微妙にずれているケースがあったのです。

Google Places APIが返す施設名は時期やタイミングによって変わることがあるようで、同じ施設でも引用符の有無や旧名の括弧書きが付くなど、2つのCSV間で名前が一致しないことがありました。

完全一致だと76施設中2施設が欠落したので、引用符を除去した上で前方一致マッチに変更して解決しました。

```python
visit_clean = visit_name.replace('"', '').replace("'", "")
for rname in ranking:
    rname_clean = rname.replace('"', '').replace("'", "")
    if (visit_clean.startswith(rname_clean)
            or rname_clean.startswith(visit_clean)):
        name = rname
        break
```

`placeId` で結合するのが本来は確実ですが、今回はCSV同士の結合だったのでこの方法で対処しました。

## 地図アプリの実装

### マーカー表示

各施設を `L.circleMarker` で表示します。訪問回数に応じて半径を6〜30pxの範囲で変化させています。

```javascript
const radius = 6 + (f.total_visits / maxVisits) * 24;

const marker = L.circleMarker([f.lat, f.lng], {
  radius: radius,
  fillColor: '#f97316',
  fillOpacity: 0.7,
  color: '#fff',
  weight: 1.5,
});
```

77回訪問した施設と1回だけの施設の差がマーカーの大きさで直感的にわかります。

### ポップアップ

マーカーをクリックすると、施設名・訪問回数・期間・年別の内訳バーチャート・公式サイトリンクが表示されます。

![ポップアップ表示](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4384587/e1184bb6-88f0-4254-be27-8dc592645b96.png)
*年別のバーチャートで、いつ頃よく通っていたかがわかる*

### ヒートマップ

Leaflet.heatプラグインで訪問頻度のヒートマップを表示します。ダークテーマの地図タイルとの相性を考えて、明るめのグラデーション（紫→シアン→緑→黄→赤）を設定しました。

```javascript
heatLayer = L.heatLayer(points, {
  radius: 35,
  blur: 25,
  minOpacity: 0.4,
  gradient: {
    0.1: '#6366f1',
    0.3: '#06b6d4',
    0.5: '#22d3ee',
    0.65: '#4ade80',
    0.8: '#facc15',
    0.9: '#f97316',
    1.0: '#ef4444',
  },
});
```

![ヒートマップ表示](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4384587/adc2d411-1a06-4315-99e0-682ed2d296d4.png)
*東京エリアのヒートマップ。豊島区〜北区・台東区あたりが「銭湯ホットスポット」*

### 年別フィルター

年ボタンで絞り込むと、統計・ランキング・地図がすべて連動して更新されます。「2021年はどこに行っていたか」が一目でわかります。

フィルター処理は元データを変換するだけのシンプルな実装です。

```javascript
function getFilteredData() {
  if (!selectedYear) return allData;

  return allData
    .filter(f => f.visits_by_year[selectedYear])
    .map(f => ({
      ...f,
      total_visits: f.visits_by_year[selectedYear],
      visit_dates: f.visit_dates.filter(d => d.startsWith(selectedYear)),
    }))
    .sort((a, b) => b.total_visits - a.total_visits);
}
```

### ランキングサイドバー

左サイドバーに訪問回数順のリストを表示し、クリックすると地図がその施設にフライしてポップアップが開きます。

## データから見えた発見

地図にしてみると、CSVの表やランキングでは見えなかったパターンが浮かび上がりました。

### 2016年から銭湯にハマり始めた

年別フィルターで見ると、2013〜2015年はほぼ訪問がなく（年0〜3回）、**2016年に26回と急増**しています。近所の温泉施設に通い始めたのがきっかけでした。

### 時期ごとに「ホーム銭湯」が変わる

| 時期 | エリア | 傾向 |
|------|--------|------|
| 2016-2017 | 都内東部 | 近所の温泉施設に通い詰め |
| 2018-2019 | 都内北東部 | 人気銭湯を開拓 |
| 2020-2021 | 都内西部 | コロナ禍でも最多ペース |
| 2022-2023 | 埼玉南部 | スーパー銭湯がホームに |
| 2024-2026 | 埼玉南部 | サウナ施設も加わり2拠点体制 |

生活圏の変化がそのまま銭湯の変遷に反映されています。ヒートマップで見ると、光っているエリアが時期ごとに移動していくのが面白いです。

### 旅先でも銭湯に寄る

マーカーを引いて全国表示にすると、沖縄、山形、京都、箱根、鬼怒川など、旅行先でも温泉・銭湯に立ち寄っていることがわかります。1回だけの小さなマーカーが全国に散らばっているのを見ると、旅の記憶が蘇ります。

## GitHub Pagesへのデプロイ

ビルドツールを使っていないので、デプロイは非常にシンプルです。

1. `docs/` ディレクトリに `index.html` と `sento-data.js` を配置
2. GitHubリポジトリの Settings → Pages → Source で `main` ブランチの `/docs` を選択

これだけで公開されます。

## まとめ

- **Leaflet**でAPIキー不要、ビルド不要
- **マーカーサイズ可変**で訪問頻度が直感的にわかる
- **ヒートマップ**で「銭湯ホットスポット」が浮かび上がる
- **年別フィルター**で行動圏の変遷が見える

ランキング表だけでは数字の羅列だったデータが、地図にすることで「どこに住んでいた時期にどの銭湯に通っていたか」というストーリーが見えるようになりました。自分の生活史を銭湯で振り返るのは、なかなか楽しい体験です。

データの抽出方法は[前回の記事](https://qiita.com/milkmaccya2/items/1e1ef669b9b5c5d78367)を参照してください。カフェやラーメン屋など、キーワードを変えれば同じように可視化できます。

## 参考

- [デモサイト（GitHub Pages）](https://milkmaccya2.github.io/sento-map-viewer/)
- [ソースコード（GitHub）](https://github.com/milkmaccya2/sento-map-viewer)
- [データ抽出スクリプト](https://github.com/milkmaccya2/google-maps-timeline-sento)
- [前回の記事：13年分のGoogle Mapsタイムラインから「自分の銭湯ランキング」を作る](https://qiita.com/milkmaccya2/items/1e1ef669b9b5c5d78367)
- [Leaflet公式ドキュメント](https://leafletjs.com/reference.html)
- [Leaflet.heat](https://github.com/Leaflet/Leaflet.heat)
