---
title: 13年分のGoogle Mapsタイムラインから「自分の銭湯ランキング」を作る
tags:
  - Python
  - GoogleMaps
  - GooglePlacesAPI
  - ClaudeCode
private: false
updated_at: '2026-03-22T22:00:31+09:00'
id: 81b2e3d32780ff1aa240
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

ふと「自分、今まで何軒ぐらいの銭湯やサウナに行ったんだろう」と気になりました。

Google Mapsのタイムラインには長年のロケーション履歴が残っているので、ここからうまく抽出すれば過去の訪問履歴を銭湯・サウナだけに絞ってランキング化できるはずです。やってみたところ思った以上にハマりどころがあったので、同じことをやりたい方の参考になればと思い記事にしました。

今回はClaude Codeに手伝ってもらいながら進めています。スクリプトの実装やデータの整理はほぼClaude Code任せで、自分がやったのは「データを手元に持ってくる」部分が中心です。

## 全体の流れ

1. Google Mapsのタイムラインデータをエクスポートする
2. Places APIで施設名を取得する
3. キーワードで銭湯・サウナをフィルタする
4. ノイズを除去してCSVに出力する

順番に見ていきます。

## タイムラインデータのエクスポート

### Google Takeoutでは取れない

最初にGoogle Takeout（takeout.google.com）を試しました。「ロケーション履歴（タイムライン）」にチェックを入れてエクスポートしたのですが、出てきたZIPの中身は`Settings.json`と`Encrypted Backups.txt`だけ。訪問データが1件も入っていませんでした。

これは2024年にGoogleがタイムラインの保存先をクラウドからデバイス上（スマホ本体）に移行した影響です。そういえばそんな話あったなと。移行済みのアカウントでは、Takeoutから訪問履歴が取れなくなっています。

### Androidの設定アプリからエクスポートする

調べ直したところ、[Googleコミュニティのスレッド](https://support.google.com/maps/thread/365988599/google%E3%83%9E%E3%83%83%E3%83%97%E3%81%AE%E3%82%BF%E3%82%A4%E3%83%A0%E3%83%A9%E3%82%A4%E3%83%B3%E3%82%92%E3%82%A8%E3%82%AF%E3%82%B9%E3%83%9D%E3%83%BC%E3%83%88%E3%81%99%E3%82%8B%E6%96%B9%E6%B3%95?hl=ja)が参考になりました。正解はGoogle Mapsアプリではなく、Androidの設定アプリから辿る方法です。

1. Androidの**設定アプリ**（⚙）を開く
2. **「位置情報」** → **「位置情報サービス」** → **「タイムライン」**
3. **「タイムラインをエクスポート」**をタップ

エクスポート先としてGoogle Driveを選択するとJSONファイルが保存されます。自分の場合、13年分で約60MBでした。

## エクスポートしたデータの構造

JSONを開いてみると、こんな構造になっています。

```json
{
  "semanticSegments": [
    {
      "startTime": "2024-01-15T10:00:00.000+09:00",
      "visit": {
        "topCandidate": {
          "placeId": "ChIJxxxxxxxxxxxxxxx",
          "semanticType": "UNKNOWN",
          "placeLocation": {
            "latLng": "35.xxx°, 139.xxx°"
          }
        }
      }
    }
  ]
}
```

ここで気づくのですが、**施設名がありません**。`placeId`（Google Maps内部のID）と緯度経度しか入っていないんですね。施設名を取得するには、この`placeId`をGoogle Places APIに問い合わせる必要があります。

## Places APIで施設名を取得する

### 準備

1. [Google Cloud Console](https://console.cloud.google.com/) でプロジェクトを選択（なければ作成）
2. 「Places API (New)」を有効化
3. APIキーを作成

### コスト

<!-- TODO: 実際のコストを確認して追記する -->

Places API (New) の Place Details で `displayName` を取得する場合、1リクエストあたり約$0.003（Basic SKU）です。自分のデータでは約3,000件のユニークな場所があったので、$10前後の見込みです。詳細は確認でき次第追記します。

### placeId から施設名を取得する

Places API (New) に `placeId` を渡すと、施設名・住所・カテゴリが返ってきます。

```python
url = "https://places.googleapis.com/v1/places/{}".format(place_id)
req = urllib.request.Request(url, headers={
    "X-Goog-Api-Key": api_key,
    "X-Goog-FieldMask": "displayName,types,formattedAddress",
})
with urllib.request.urlopen(req, timeout=10) as resp:
    data = json.loads(resp.read())
    name = data.get("displayName", {}).get("text", "")
    address = data.get("formattedAddress", "")
    types = data.get("types", [])
```

`X-Goog-FieldMask` で取得するフィールドを絞ることで、コストを Basic SKU（$0.003/リクエスト）に抑えられます。

### キーワードとカテゴリでフィルタする

取得した施設名とカテゴリを使って、銭湯・サウナに該当するかを判定します。

```python
KEYWORDS = [
    "銭湯", "サウナ", "温泉", "スパ", "湯", "風呂", "岩盤浴",
    "bath", "sauna", "spa", "onsen", "hot spring",
]
RELEVANT_TYPES = {"spa", "hot_spring", "public_bath", "thermal_bath"}

def matches(place):
    name = place.get("name", "").lower()
    types = set(place.get("types", []))
    if types & RELEVANT_TYPES:
        return True
    return any(kw.lower() in name for kw in KEYWORDS)
```

施設名のキーワードマッチだけだとノイズが出るので、Google Places のカテゴリ（`types`）も併用しています。

### スクリプト全体

上記を組み合わせたスクリプトをGitHubに置いています。標準ライブラリのみで動きます。

https://github.com/milkmaccya2/google-maps-timeline-sento

3,000件のAPI呼び出しには10分程度かかりました。結果は`place-cache.json`にキャッシュされるので、途中で止まっても再実行すればキャッシュ済みの分はスキップされます。

## ノイズの除去と公式サイトの調査

キーワードマッチだけだとノイズが混じります。「spa」に引っかかるスパゲッティ屋やアパレルショップ、「湯」が名前に入るジムなど。これもClaude Codeに全施設リストを見せて除外してもらいました。JAXAの宇宙センターが混じっていたのは笑いました。

各施設の公式サイトのURLもClaude Codeにsub agentsで4並列検索してもらい、70施設以上の公式サイトURL付きCSVが出力されました。

## 結果

13年分のタイムラインから、**70施設以上・数百回の銭湯・サウナ訪問**が抽出できました。

```
============================================================
  銭湯・サウナ訪問履歴  （2013-07-14 〜 2026-03-21）
============================================================
  総訪問回数: 462
  ユニーク施設数: 92

--- 訪問回数ランキング TOP 5 ---
   1. ██████████████████████████████  77回
   2. ██████████████████████████████  43回
   3. ██████████████████████████████  32回
   4. █████████████████████████████   29回
   5. █████████████████               17回
```

地元のスーパー銭湯が圧倒的1位で、次いで都内の銭湯が並びます。旅行先の温泉もちゃんと拾えていました。

公式サイトのURLと一緒にリストを眺めていると、忘れていた施設が出てきて「あーここ行ったな」となります。旅行先でたまたま寄った温泉とか、閉業してしまった施設とか。なかなかよかったです。

## Google Takeoutで他にできそうなこと

今回はロケーション履歴を使いましたが、Google Takeoutには他にも面白いデータがあります。

### YouTube / YouTube Musicの視聴履歴

視聴履歴をJSONでエクスポートして、自分だけの「YouTube Wrapped」を作れます。最も見たチャンネルのランキング、時間帯別の視聴傾向など。[ytmusicstats](https://ytmusicstats.vercel.app/) のような既存ツールもあります。

### Google検索履歴（My Activity）

過去数年分の検索クエリをワードクラウド化したり、月別の関心トピックをクラスタリングしたりできます。検索履歴は「その時の自分が本当に知りたかったこと」の記録なので、日記より正直な自分史になります。

### Chrome閲覧履歴

ドメイン別の滞在傾向や、時間帯別のブラウジングパターンを分析できます。[chrome-takeout-analyzer](https://github.com/OurCodeBase/chrome-takeout-analyzer) でインタラクティブなレポートも作れます。

### Google Fitデータ

歩数・消費カロリー・睡眠データなどのCSVを時系列で可視化できます。今回のロケーション履歴と組み合わせれば「この日はどこを歩き回って何歩だったか」まで紐づけられます。

### Gmail

mbox形式でエクスポートして、送受信頻度の時系列分析やセンチメント分析ができます。

---

どのデータもJSON/CSVで提供されるので、スクリプトとの相性がいいです。複数のデータを組み合わせるともっと面白くなりそうです。

## まとめ

今回やってみて分かったポイントです：

- **Google Takeoutではタイムラインデータが取れません**（2024年以降のデバイス移行済みアカウント）
- **Androidの設定アプリからエクスポート**するのが正解です
- エクスポートデータには**施設名がない**ので、Places APIでの名前解決が必要です
- 実装はClaude Codeに丸投げで問題ありませんでした。自分がやったのはデータのエクスポートとAPIキーの用意ぐらいです

カフェやラーメン屋の訪問回数なんかもキーワードを変えれば同じように出せるので、気になる方はぜひ試してみてください。

## 参考

- [スクリプト（GitHub）](https://github.com/milkmaccya2/google-maps-timeline-sento)
- [Googleマップのタイムラインをエクスポートする方法 - Google コミュニティ](https://support.google.com/maps/thread/365988599/google%E3%83%9E%E3%83%83%E3%83%97%E3%81%AE%E3%82%BF%E3%82%A4%E3%83%A0%E3%83%A9%E3%82%A4%E3%83%B3%E3%82%92%E3%82%A8%E3%82%AF%E3%82%B9%E3%83%9D%E3%83%BC%E3%83%88%E3%81%99%E3%82%8B%E6%96%B9%E6%B3%95?hl=ja)
- [Places API (New) - Google Developers](https://developers.google.com/maps/documentation/places/web-service)
- [Google Takeout](https://takeout.google.com/)
