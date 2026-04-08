---
name: criteo-tag-checker
description: >
  Criteoタグの設置状況を確認するスキル。サイトURLを入力すると、TOPページからリンクを辿ってクロールし、
  各ページのCriteoイベント（e=vh, e=vl, e=vp, e=vb, e=tc, e=vpg, e=vs）の設置状況を一覧表示する。
  GTMコンテナJS内のCriteoタグも解析対象。ユーザーが「Criteoタグを確認して」「Criteoの設置状況を見て」
  「このサイトにCriteoが入っているか調べて」「タグの設置確認」などと言ったときに使用する。
  Criteo、タグ設置、タグ確認、OneTag、広告タグに関する質問にはこのスキルを使うこと。
---

# Criteoタグ設置状況確認スキル

サイトURLを受け取り、Criteoタグ（OneTag）の設置状況をページ別・イベント別に確認して報告する。

## 確認対象のCriteoイベント一覧

| コード | イベント名 | 説明 |
|--------|-----------|------|
| e=vh | viewHome | トップページ閲覧 |
| e=vl | viewList | 商品一覧ページ閲覧 |
| e=vp | viewProduct | 商品詳細ページ閲覧 |
| e=vb | viewBasket | カートページ閲覧 |
| e=tc | trackTransaction | 購入完了ページ |
| e=vpg | viewPage | 汎用ページ閲覧 |
| e=vs | viewSearch | 検索結果ページ |

## ワークフロー

### Step 1: サイトURLの確認

ユーザーからサイトURLを受け取る。URLが指定されていない場合は聞く。

### Step 2: TOPページの取得とGTM解析

1. WebFetchでTOPページのHTMLを取得する
2. HTMLからGTMコンテナID（`GTM-XXXXXXX`）を探す
3. GTMコンテナIDが見つかったら、`https://www.googletagmanager.com/gtm.js?id=GTM-XXXXXXX` をWebFetchで取得し、Criteo関連のコードが含まれているか確認する

**GTMコンテナ内のCriteo検出キーワード:**
- `criteo`, `static.criteo.net`, `ld/ld.js`, `criteo_q`, `cto_tms`, `cto_bundle`

**GTMコンテナ内のイベント検出キーワード:**
- viewHome / view_home / `"vh"` → e=vh
- viewList / view_list / viewCategory → e=vl
- viewProduct / view_product / viewItem → e=vp
- viewBasket / view_basket / viewCart → e=vb
- trackTransaction / track_transaction / `"tc"` → e=tc
- viewPage / view_page / `"vpg"` → e=vpg
- viewSearch / view_search / `"vs"` → e=vs

### Step 3: ページのクロールとイベント検出

TOPページのリンクを抽出し、同一ドメイン内の主要ページ（15〜20ページ程度）をWebFetchで取得する。

**リンク抽出時の注意:**
- 同一ドメインのみ対象
- 画像・CSS・JS・PDF等のリソースファイルは除外（.jpg, .png, .css, .js, .pdf 等）
- クエリパラメータは除去してURL重複を減らす
- ナビゲーションやメインコンテンツのリンクを優先

**各ページのCriteoイベント検出方法:**

そのページのHTMLソースから、ページ固有のCriteoイベントを検出する。全ページに同じイベントを割り当てないこと。

1. **直接埋め込みの検出**: ページHTML内の `criteo_q.push(...)` や inline script から、そのページで発火するイベントを特定する
2. **dataLayerの検出**: `dataLayer.push({...})` 内にCriteo関連のイベント名がないか確認する
3. **GTMとの組み合わせ**: GTMにCriteoがあり、ページにGTMが読み込まれていても、そのページ固有のイベントデータがない場合は「不明」とする（全イベントを割り当てない）

### Step 4: 結果の集約とURLパターン分析

検出されたイベントごとに、該当URLの共通パターンを見つける。

**URLパターンの作り方:**
- URLの第1パスセグメントでグルーピングする
- 例: `https://example.co.jp/products/123` と `https://example.co.jp/products/456` → `https://example.co.jp/products/ 配下`
- TOPページ（パスなし）はそのままURLを表示

### Step 5: 結果の出力

以下のフォーマットで結果を出力する:

```
## Criteoタグ設置状況: [サイト名/ドメイン]

確認日: YYYY-MM-DD
確認ページ数: XX ページ

| イベント | 説明 | 設置URLパターン | 設置状況 |
|---------|------|----------------|---------|
| e=vh (viewHome) | トップページ閲覧 | https://example.co.jp/ | 設置あり |
| e=vl (viewList) | 商品一覧ページ閲覧 | https://example.co.jp/category/ 配下 | 設置あり |
| e=vp (viewProduct) | 商品詳細ページ閲覧 | https://example.co.jp/products/ 配下 | 設置あり |
| e=vb (viewBasket) | カートページ閲覧 | — | 設置なし |
| e=tc (trackTransaction) | 購入完了ページ | — | 設置なし |
| e=vpg (viewPage) | 汎用ページ閲覧 | https://example.co.jp/about/ 配下 | 設置あり |
| e=vs (viewSearch) | 検索結果ページ | — | 設置なし |

### 検出方法
- GTMコンテナ: GTM-XXXXXXX（Criteo設定あり）
- 検出方式: [GTM経由 / 直接埋め込み / 両方]

### 注意事項
- 静的HTML解析のため、JavaScript実行後に動的に生成されるタグは検出できない場合があります
- カート・購入完了ページはログインが必要な場合が多く、クロールでは到達できないことがあります
```

**重要なルール:**
- Criteoタグが検出されなかったイベントは「設置なし」と明記する
- 各ページにはそのページ固有のイベントのみを割り当てる（GTMにCriteoがあるからといって全ページに全イベントを割り当てない）
- URLパターンは第1パスセグメントで簡潔にまとめる

## Webツール版

ブラウザで使えるWebツール版も公開されている:
https://ikedakazuki-creator.github.io/criteo-tag-checker/

チームメンバーにはこのURLを共有すれば、誰でもブラウザから利用可能。
