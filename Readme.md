# 水上ビル商店街 Visitor Flow Intelligence
## プロジェクトサマリー

---

## 🗺 概要

BLEセンサーで収集した来訪者データをリアルタイムで可視化するWebマップシステム。
商店街の各拠点間の人の流れをパーティクルアニメーションで表現する。

**公開URL:** `https://[username].github.io/[repo-name]/`

---

## 🏗 システム構成

```
M5NanoC6（BLEスキャン）
    ↓ ESP-NOW
M5Stamp S3（中継・集約）
    ↓ WiFi
Google Apps Script（WebApp）
    ↓
BigQuery（[BQ_PROJECT_ID]）
    ↓
index.html（GitHub Pages）
    ↓
Leaflet.js + p5.js（ブラウザで表示）
```

---

## 📁 ファイル構成

```
mizunoue-flow-map/
├── index.html          ← メインマップ（2D・p5.js）
├── index3d.html        ← 3Dアークマップ（deck.gl）※別途
└── instagram-bot/
    ├── post.js         ← スクショ→Instagram投稿スクリプト
    ├── package.json
    └── .github/
        └── workflows/
            └── daily-post.yml  ← GitHub Actions（毎朝8時JST）
```

---

## 🔧 BigQuery テーブル・ビュー

| 名前 | 種類 | 内容 |
|------|------|------|
| `ble_logs.raw_logs` | テーブル | BLE生ログ（mac_hash, loc_id, ts） |
| `ble_logs.device_locations` | テーブル | 拠点座標（loc_id, lat, lng） |
| `ble_logs.visitor_flow` | ビュー | 動線データ（from_loc, to_loc, hour_jst, distance_meters, visitor_count） |
| `ble_logs.visitor_presence` | ビュー | 拠点別来訪者数（loc_id, hour_jst, visitor_count, avg_dwell_minutes） |

### ⚠️ タイムスタンプの注意点
`ts` は **JST で保存**されているため、BigQueryでは補正不要。
```sql
-- ✅ 正しい
WHERE DATE(ts) = '2026-03-15'

-- ❌ 二重補正になる
WHERE DATE(DATETIME(ts, 'Asia/Tokyo')) = '2026-03-15'
```

---

## 🌐 GAS WebApp API

**エンドポイント:** `https://script.google.com/macros/s/...`

### リクエスト
```javascript
// GET（ブラウザ直接・デバッグ用）
?date=2026-03-15

// POST（マップのGOボタン・キャッシュ回避）
body: JSON.stringify({ date: '2026-03-15' })
```

### レスポンス
```json
{
  "date": "2026-03-15",
  "locations": [{ "loc_id": "A棟", "lat": "...", "lng": "..." }],
  "flows": [{ "from_loc": "C棟", "to_loc": "D棟", "hour_jst": "12", "distance": "93", "count": "26" }],
  "presence": [{ "loc_id": "C棟", "hour_jst": "12", "count": "150", "avg_dwell": "7" }],
  "total": 3727,
  "generated": "2026-03-15T10:00:00.000Z"
}
```

### SQL の特徴
- `flowSQL`: `visitor_flow` ビューに `device_locations` をJOINして距離を動的計算（ST_DISTANCE）
- `totalSQL`: 15分未満滞在のユニークMAC数をカウント（staffを除外）
- `hour_jst` を含めて時間帯別フィルタリングに対応

---

## 🎨 index.html の機能

### ビジュアル
| 要素 | 意味 |
|------|------|
| パーティクルの量 | 来訪者数（300人基準の絶対値） |
| パーティクルの大きさ | その動線の相対的な多さ |
| Stayingの色（暖色） | 天気で変化（☀️黄橙 / 🌧️深紅 / 🌨️ピンク 等） |
| Passingの色（寒色） | 気温で変化（青白→シアン→ティール→緑→黄緑） |
| Stayingの速さ | 遅い（ゆっくり漂う） |
| Passingの速さ | 速い（すばやく駆け抜ける） |
| ノードサイズ | 来訪者数（presence）に比例 |
| ノードリング | 在場強度でパルス |

### データソース
- **天気・気温**: Open-Meteo API（無料・APIキー不要）
  - 今日 → forecast API
  - 過去 → archive API（正午の値）
- **座標**: 豊橋市水上ビル商店街周辺（lat: [LAT], lng: [LNG]）

### UI機能
| 機能 | 説明 |
|------|------|
| ヘッダー KPI | Total Visitors / Active Flows / Avg Dwell / Staying Rate / Peak Node |
| Live Ticker | 左上パネルに4指標をカウントアップアニメーション |
| Top Route | 最多動線を常時表示 |
| Flow Ranking | サイドバーに動線ランキング（金銀銅バッジ付き） |
| Visitor Split | Staying/Passingの来訪者数バー |
| ノード数値ラベル | p5.jsで各拠点の右下に来訪者数・平均滞在時間を描画 |
| 天気バッジ | ヘッダーに現在の天気と気温 |
| トグルボタン | All / Passing / Staying でフィルタ |
| 日付ピッカー + GOボタン | 任意の日付のデータを取得 |
| **時間スライダー** | 0〜23時の時間帯別表示 |
| **▶ 再生ボタン** | データがある時間帯を1.2秒ごとにタイムラプス再生 |
| レスポンシブ対応 | スマホではサイドバーが下からスライドするドロワーに変換 |
| 地図タイル | CartoDB light（英語ラベル）+ ダークフィルター |
| キャッシュ回避 | POST リクエストでGoogleのCDNキャッシュをバイパス |

---

## 📸 Instagram 自動投稿（準備中）

**フロー:**
```
GitHub Actions（毎朝 8:00 JST）
  → Puppeteer でスクリーンショット（1080×1080px）
  → Imgur に画像アップロード
  → Instagram Graph API で投稿
```

**必要なGitHub Secrets:**
| Secret名 | 内容 |
|----------|------|
| `SITE_URL` | GitHub PagesのURL |
| `IMGUR_CLIENT_ID` | Imgur クライアントID |
| `IG_USER_ID` | Instagram ユーザーID（数字） |
| `IG_ACCESS_TOKEN` | Instagram アクセストークン（60日有効） |

**投稿キャプション例:**
```
📡 Visitor Flow · 2026/03/15 Sun

Daily pedestrian data from Mizunoue Building Shopping District.
Warm particles = visitors who stayed. Cool = passing through.
Every dot is a real person. Every day is different.

@[your_instagram_account]

#mizunoue #datacenter #visitorflow #urbandata
#smartcity #datavisualization #pedestrian #shoppingdistrict
#toyohashi #aichi #japan
```

---

## 🔑 重要な設定値

```javascript
// 拠点分類の距離閾値
const isStaying = d => d >= 150;  // 150m以上 = Staying

// パーティクル生成（絶対値基準）
const spawnRate = Math.min(flow.count / 300, 1) * 0.28;
// → 300人でフル密度

// パーティクルサイズ
this.sz = 1 + r * 3;  // 1〜4px（相対値）

// パーティクル速度
Staying: s.random(0.003, 0.007)  // 遅い
Passing: s.random(0.009, 0.018)  // 速い

// 気温グラデーション基準
// -5°C=氷青白 → 10°C=シアン → 20°C=ティール → 28°C=緑 → 35°C=黄緑
const t = Math.max(0, Math.min(1, (weather.temp + 5) / 40));
```

---

## 🚧 今後の展望

- [ ] Instagram 自動投稿の完成（トークン取得が残っている）
- [ ] 動画録画機能（CCapture.js）
- [ ] タイムラプス動画のインスタ自動投稿
- [ ] 動線を道路に沿わせるルーティング
- [ ] 駅前エリアのメッシュ可視化
- [ ] 来訪者データをMIDI音楽に変換
- [ ] LED リボン制御（GAS API連携）
- [ ] まちなか広場の2子ノードリレー展開

---

## 📚 使用技術・ライブラリ

| 技術 | 用途 |
|------|------|
| Leaflet.js 1.9.4 | 地図表示 |
| p5.js 1.9.4 | パーティクルアニメーション |
| CartoDB Basemaps | 英語地図タイル |
| Open-Meteo API | 天気・気温データ（無料） |
| Google BigQuery | データウェアハウス |
| Google Apps Script | API サーバー |
| GitHub Pages | 無料ホスティング |
| GitHub Actions | 自動投稿スケジューラ |
| Puppeteer | ヘッドレスブラウザ（スクショ） |
| Imgur API | 画像ホスティング |
| Instagram Graph API | 自動投稿 |

---

## 📋 更新履歴

### v3.0 — イベントレイヤー統合
- **街のイベントオーバーレイ追加**
  - Googleスプレッドシート（イベントリスト）をGASで読み込み
  - 当日のイベントを画面上部中央にバッジとして表示
  - イベント種別ごとにアイコン・色を自動割り当て（朝市🌅・アンティークマーケット🏺・夏至祭🌞）
  - BLE拠点と紐付けた `locations` カラムで該当拠点のリングをゴールドにハイライト
  - BLE拠点外の場所も `locations` に記述可能
- **スプレッドシート列構成**
  - A列: date / B列: title / C列: locations（カンマ区切り）/ D列: note

---

### v2.5 — タイムゾーン修正
- `ts` がJSTで保存されていることが判明
- `DATE(DATETIME(ts, 'Asia/Tokyo'))` → `DATE(ts)` に修正（二重補正を解消）
- `visitor_presence` / `visitor_flow` 両ビューを修正
- GASの `totalSQL` も同様に修正
- これによりLookerとWebの数値が一致するようになった

---

### v2.4 — 時間スライダー・タイムラプス
- **時間帯別表示スライダー追加**（0〜23時・全日モード）
- **▶ 再生ボタン**でデータがある時間帯を1.2秒ごとに自動進行
- GASのSQLに `hour_jst` を追加（flow・presence両方）
- `applyHourFilter()` 関数でクライアント側フィルタリング

---

### v2.3 — リアルタイム更新対応
- GETからPOSTリクエストに切り替え（GoogleのCDNキャッシュ回避）
- GASに `doPost()` 追加・`doGet()` も併存（デバッグ用）
- GOボタン押下ごとに毎回BigQueryから最新データを取得

---

### v2.2 — 天気・気温連動カラー
- **Open-Meteo API**（無料・APIキー不要）で豊橋の天気・気温を自動取得
- Staying（立ち寄り）→ **天気で色が変わる**（暖色系のみ）
  - ☀️ Clear=黄橙 / ☁️ Cloudy=赤橙 / 🌧️ Rainy=深紅 / 🌨️ Snowy=ピンク / ⛈️ Stormy=バーガンディ
- Passing（素通り）→ **気温で色が変わる**（寒色系のみ）
  - −5°C=氷青白 → 10°C=シアン → 20°C=ティール → 28°C=緑 → 35°C=黄緑
- 過去の日付を選択した場合はarchive APIで正午の値を取得
- ヘッダーに天気バッジ表示
- 凡例の色もリアルタイムで更新

---

### v2.1 — パーティクル密度の絶対値化
- パーティクル生成を相対値→**絶対値ベース**に変更
- 300人基準でフル密度（イベント時の最大人数）
- 1〜5人ならほぼ数粒だけ表示
- 普段（30〜50人）は控えめ、イベント時（300人）は最大密度

---

### v2.0 — 全文英語化・数値ダッシュボード強化
- UI全文英語化（場所特定回避のため）
- ヘッダーに5つのKPI追加（Total Visitors / Active Flows / Avg Dwell / Staying Rate / Peak Node）
- 左上にLive Tickerパネル（カウントアップアニメーション付き）
- 左下にTop Routeバッジ
- 各ノードに来訪者数・平均滞在時間ラベルを右下に表示
- サイドバーにVisitor Splitバー
- 地図タイルをCartoDB（英語ラベル）に変更

---

### v1.5 — パーティクル速度・方向の意味付け
- Staying → **遅い**（0.003〜0.007）ゆっくり漂う
- Passing → **速い**（0.009〜0.018）すばやく駆け抜ける

---

### v1.4 — レスポンシブ対応
- 768px以下でサイドバーが下からスライドするドロワーに変換
- ヘッダーKPI帯はモバイルで非表示
- 「≡ Flows」ボタンでドロワー開閉

---

### v1.3 — p5.jsパーティクルエンジン
- Leaflet地図上にp5.jsキャンバスをオーバーレイ
- ベジェ曲線でパーティクルが流れる
- フェードイン・アウト・ホワイトコア輝点・グローエフェクト
- ノード拡散リング（在場強度でパルス）
- ビネット（周辺光量落ち）

---

### v1.0 — 初期リリース
- Leaflet.js + SVGによる動線マップ
- GAS → BigQuery データパイプライン
- 来訪者数に比例したマーカーサイズ・色
- サイドバー動線ランキング
- 日付ピッカー・GOボタン
