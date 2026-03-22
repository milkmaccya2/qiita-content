---
title: 13年分のGoogle Mapsタイムラインから「自分の銭湯ランキング」を作る
tags:
  - Python
  - GoogleMaps
  - GooglePlacesAPI
  - ClaudeCode
private: false
updated_at: '2026-03-22T22:00:31+09:00'
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

ふと「自分、今まで何軒ぐらいの銭湯やサウナに行ったんだろう」と気になりました。

Google Mapsのタイムラインには長年のロケーション履歴が残っているので、ここからうまく抽出すれば過去の訪問履歴を銭湯・サウナだけに絞ってランキング化できるはずです。やってみたところ思った以上にハマりどころがあったので、同じことをやりたい方の参考になればと思い記事にしました。

スクリプトの実装はAI（Claude Code）に任せたので、この記事ではデータの取得方法やハマりどころを中心に書いています。

## 全体の流れ

1. Google Mapsのタイムラインデータをエクスポートする
2. Places APIで施設名を取得する
3. キーワードで銭湯・サウナをフィルタする
4. ノイズを除去してCSVに出力する

順番に見ていきます。

## タイムラインデータのエクスポート

### Google Takeoutでは取れない

最初にGoogle Takeout（takeout.google.com）を試しました。「ロケーション履歴（タイムライン）」にチェックを入れてエクスポートしたのですが、出てきたZIPの中身は`Settings.json`と`Encrypted Backups.txt`だけ。訪問データが1件も入っていませんでした。

これは[2023年12月にGoogleが発表](https://blog.google/products-and-platforms/products/maps/updates-to-location-history-and-new-controls-coming-soon-to-maps/)した、タイムラインの保存先をクラウドからデバイス上（スマホ本体）に移行する変更の影響です。そういえばそんな話あったなと。2024年中に段階的に移行が進み、移行済みのアカウントではTakeoutから訪問履歴が取れなくなっています（[サポートページ](https://support.google.com/maps/answer/14169818)）。

### デバイスからエクスポートする

調べ直したところ、[Google Maps公式ヘルプ](https://support.google.com/maps/answer/6258979)にエクスポート手順が載っていました。

**Androidの場合**

1. **設定アプリ**（⚙）を開く
2. **「位置情報」** → **「位置情報サービス」** → **「タイムライン」**
3. **「タイムラインをエクスポート」**をタップ

**iPhoneの場合**

[Google Maps公式ヘルプ（iPhone向け）](https://support.google.com/maps/answer/6258979?co=GENIE.Platform%3DiOS)を参照してください。

エクスポート先としてGoogle Driveを選択するとJSONファイルが保存されます。自分の場合（Android）、13年分で約60MBでした。

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

Places API (New) では、リクエスト時に取得するフィールドによって料金が変わります。今回は `displayName`、`types`、`formattedAddress` だけ取得するので、最も安い料金帯が適用されます（[$5.00 / 1,000リクエスト](https://developers.google.com/maps/documentation/places/web-service/usage-and-billing)）。

Google Maps Platform には**月額 $200 の無料クレジット**があるので、**月 40,000リクエストまでは無料**です。自分のデータでは約3,000件のユニークな場所があったので、無料枠に余裕で収まりました。

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

Places API (New) では、取得するフィールドが Basic（名前・住所など）/ Contact（電話番号など）/ Atmosphere（レビュー・営業時間など）のカテゴリに分かれていて、カテゴリが上がるほど料金も上がります。`X-Goog-FieldMask` で Basic カテゴリのフィールドだけを指定することで、最も安い料金帯に収められます。

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

キーワードマッチだけだとノイズが混じります。「spa」に引っかかるスパゲッティ屋やアパレルショップ、「湯」が名前に入るジムなど。これもClaude Codeに全施設リストを見せて除外してもらいました。

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

施設名は伏せていますが、地元のスーパー銭湯が圧倒的1位で、次いで都内の銭湯が並びます。旅行先の温泉もちゃんと拾えていました。

公式サイトのURLと一緒にリストを眺めていると、忘れていた施設が出てきて「あーここ行ったな」となります。旅行先でたまたま寄った温泉とか、閉業してしまった施設とか。晩酌のつまみになりますね。

## Google Takeoutで他にできそうなこと

今回はロケーション履歴を使いましたが、Google Takeoutでは他にもYouTubeの視聴履歴、Google検索履歴、Chrome閲覧履歴、Google Fitの歩数データ、Gmailなど、さまざまなデータをエクスポートできます。どれもJSON/CSVで提供されるので、同じようにスクリプトで分析できます。

## まとめ

今回やってみて分かったポイントです。

- **Google Takeoutではタイムラインデータが取れません**（2024年以降のデバイス移行済みアカウント）
- **デバイスからエクスポート**するのが正解です（Androidは設定アプリ、iPhoneはGoogle Mapsアプリから）
- エクスポートデータには**施設名がない**ので、Places APIでの名前解決が必要です
- Places APIの無料枠（月40,000リクエスト）で十分まかなえます

カフェやラーメン屋の訪問回数なんかもキーワードを変えれば同じように出せるので、気になる方はぜひ試してみてください。

## 参考

- [スクリプト（GitHub）](https://github.com/milkmaccya2/google-maps-timeline-sento)
- [Updates to Location History and new notification controls - Google Blog](https://blog.google/products-and-platforms/products/maps/updates-to-location-history-and-new-controls-coming-soon-to-maps/)
- [タイムラインについて - Google Maps ヘルプ](https://support.google.com/maps/answer/14169818)
- [Google マップ タイムラインを管理する - Google Maps ヘルプ](https://support.google.com/maps/answer/6258979)
- [Places API 料金 - Google Developers](https://developers.google.com/maps/documentation/places/web-service/usage-and-billing)
