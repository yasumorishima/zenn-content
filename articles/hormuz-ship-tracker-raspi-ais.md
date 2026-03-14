---
title: "Raspberry Pi 5でホルムズ海峡のリアルタイム船舶追跡システムを構築した"
emoji: "🚢"
type: "tech"
topics: ["python", "raspberrypi", "docker", "fastapi", "ais"]
published: false
---

## はじめに

ホルムズ海峡は世界の石油輸送の約20%が通過する、地政学的に最も重要な海上チョークポイントの一つです。この海峡を通過する船舶をリアルタイムで追跡するシステムを、手持ちのRaspberry Pi 5上にDocker環境で構築しました。

本記事は、構築過程の自分用振り返りノートとして書いています。

## AIS（自動船舶識別装置）とは

AIS（Automatic Identification System）は、船舶が自動的に位置・速度・針路・船名・船種などの情報をVHF帯で送信する国際的な安全システムです。300総トン以上の国際航海船舶には搭載が義務付けられています。

AISデータには主に2種類のメッセージがあります。

- **PositionReport**: 位置・速度・針路・船首方位（数秒〜数分間隔で送信）
- **ShipStaticData**: 船名・船種・サイズ・目的地・喫水（数分間隔で送信）

## データソース: aisstream.io

[aisstream.io](https://aisstream.io/) は、世界中のAIS受信局からリアルタイムデータをWebSocketで配信する無料サービスです。GitHubアカウントで登録すればAPIキーが発行されます。

WebSocket接続時にバウンディングボックスとメッセージタイプを指定すると、その範囲内の船舶データだけがストリーミングされます。

## アーキテクチャ

```
aisstream.io (WebSocket)
    |
    v
Raspberry Pi 5
  +-------------------+     +-------------------+
  | ais-collector      |     | snapshot-cron      |
  | - WebSocket受信    |     | - matplotlib画像   |
  | - SQLite書き込み   |     | - SHA256比較       |
  | - FastAPI配信      |     | - git push (6h毎) |
  +-------------------+     +-------------------+
    |       |                       |
    v       v                       v
  SQLite  Leaflet.js地図        GitHub README
  (data/ais.db)  (port 8002)    (スナップショット)
```

このRaspberry Pi 5は、既にNPB成績予測API（port 8000）、MLB勝利確率API（port 8001）、Streamlit keepalive（Playwright）の3つのDockerコンテナが稼働しています。今回のship trackerは4つ目と5つ目のコンテナとして追加しました。

## 実装の詳細

### 1. WebSocketコレクター（collector.py）

aisstream.ioへのWebSocket接続、AISメッセージのパース、SQLiteへの保存を行います。

```python
# ホルムズ海峡のバウンディングボックス
BBOX = [[23.5, 54.0], [27.5, 58.5]]

subscribe_msg = {
    "APIKey": API_KEY,
    "BoundingBoxes": [BBOX],
    "FilterMessageTypes": ["PositionReport", "ShipStaticData"],
}
```

ShipStaticDataメッセージは船名・船種・目的地などの静的情報を含みます。これをインメモリキャッシュに保持し、PositionReportと紐づけてSQLiteに保存します。

```python
# ShipStaticDataをインメモリキャッシュに保持
static_cache: dict[int, dict] = {}

if msg_type == "ShipStaticData":
    meta = msg.get("Message", {}).get("ShipStaticData", {})
    mmsi = msg.get("MetaData", {}).get("MMSI")
    if mmsi:
        static_cache[mmsi] = {
            "ship_name": meta.get("Name", "").strip(),
            "ship_type": meta.get("Type"),
            "destination": meta.get("Destination", "").strip(),
            "draught": meta.get("MaximumStaticDraught"),
            "length": meta.get("Dimension", {}).get("A", 0)
                    + meta.get("Dimension", {}).get("B", 0),
            "width": meta.get("Dimension", {}).get("C", 0)
                    + meta.get("Dimension", {}).get("D", 0),
        }
```

船のサイズは、AIS仕様上A/B/C/Dの4つの距離値として送信されます。A+Bが全長、C+Dが全幅になります。

接続切断時は自動再接続します。

```python
except (websockets.exceptions.ConnectionClosed, OSError) as e:
    logger.warning("Connection lost: %s -- reconnecting in 10s", e)
    await asyncio.sleep(10)
```

### 2. FastAPI（api.py）

3つのAPIエンドポイントを提供します。

```python
app = FastAPI(title="Hormuz Ship Tracker")

# 直近30分の各船舶の最新位置
@app.get("/api/latest")
async def latest_positions():
    async with aiosqlite.connect(DB_PATH) as db:
        rows = await db.execute_fetchall("""
            SELECT mmsi, latitude, longitude, speed, course, heading,
                   ship_name, ship_type, destination, flag, timestamp,
                   length, width
            FROM positions
            WHERE id IN (
                SELECT MAX(id) FROM positions
                WHERE received_at > datetime('now', '-30 minutes')
                GROUP BY mmsi
            )
        """)
    # ...

# 特定船舶の航跡（デフォルト6時間）
@app.get("/api/tracks/{mmsi}")
async def vessel_track(mmsi: int, hours: int = 6):
    # ...

# 統計情報（船種別隻数など）
@app.get("/api/stats")
async def stats():
    # ...
```

AIS船種コードは数値で送信されるため、人間が読めるラベルに変換しています。

```python
SHIP_TYPE_LABELS = {
    range(70, 80): "Cargo",
    range(80, 90): "Tanker",
    range(60, 70): "Passenger",
    range(30, 36): "Fishing/Towing/Dredging",
    range(36, 40): "Military/Sailing/Pleasure",
    range(40, 50): "HSC",  # High Speed Craft
}
```

### 3. Leaflet.jsリアルタイム地図（map.html）

CARTO darkタイルを使ったダークテーマの地図です。船種ごとに色分けしたドットで船舶を表示し、30秒ごとに自動更新します。

```javascript
// ダークテーマのタイル
L.tileLayer('https://{s}.basemaps.cartocdn.com/dark_all/{z}/{x}/{y}{r}.png', {
    attribution: '...',
    maxZoom: 19,
}).addTo(map);

// 30秒ごとに自動更新
loadVessels();
setInterval(loadVessels, 30000);
```

船舶をクリックするとポップアップで詳細情報（船名、速度、針路、国旗、目的地、サイズ）が表示されます。「Show Track (6h)」ボタンで過去6時間の航跡をポリラインで描画します。

船種ごとの色分け:
- Tanker: オレンジ
- Cargo: 青
- Passenger: 緑
- Fishing: 紫
- Military: 赤
- HSC: シアン
- Unknown: グレー

![地図のスクリーンショット](https://raw.githubusercontent.com/yasumorishima/hormuz-ship-tracker/master/docs/screenshot.png)

### 4. matplotlibスナップショット（snapshot.py）

6時間ごとにSQLiteから最新データを読み出し、ダークテーマの静的マップ画像を生成します。海岸線の近似ポリゴンも描画して地理的コンテキストを与えています。

```python
fig, ax = plt.subplots(figsize=(14, 9), facecolor="#0a0a1a")
ax.set_facecolor("#0d1b2a")

# 海岸線の近似描画
for segment in COASTLINE_SEGMENTS:
    lats, lons = zip(*segment)
    ax.plot(lons, lats, color="#2a3a4a", linewidth=1.2, zorder=2)
    ax.fill(lons, lats, color="#111822", alpha=0.6, zorder=1)

# 船種ごとに色分けしてプロット
for ship_type, group in sorted(by_type.items()):
    color = get_color(ship_type)
    lats = [v["lat"] for v in group]
    lons = [v["lon"] for v in group]
    size = 30 if "Tanker" in ship_type else 22
    ax.scatter(
        lons, lats,
        s=size, c=color, edgecolors="white", linewidths=0.4,
        alpha=0.85, zorder=5, label=f"{ship_type} ({len(group)})",
    )
```

### 5. 自動push（auto_push.sh）

スナップショット画像のSHA256ハッシュを前回と比較し、変化がある場合のみgit commit & pushします。

```bash
# SHA256比較で変化検出
NEW_HASH=$(sha256sum "$SNAPSHOT" | cut -d' ' -f1)
OLD_HASH=$(sha256sum "$DEST_IMG" | cut -d' ' -f1)

if [ "$NEW_HASH" = "$OLD_HASH" ]; then
    echo "No change in snapshot -- skipping push"
    exit 0
fi

# コミットメッセージに隻数とタイムスタンプを含める
VESSEL_COUNT=$(grep -oP 'Active vessels.*?: \K[0-9]+' "$STATS" || echo "?")
git commit -m "snapshot: ${VESSEL_COUNT} vessels at ${TIMESTAMP}"
git push origin HEAD
```

## Docker構成

2つのコンテナで構成しています。

```yaml
services:
  ais-collector:
    build: .
    container_name: hormuz-tracker
    restart: unless-stopped
    ports:
      - "8002:8002"
    env_file:
      - .env
    volumes:
      - ./data:/app/data
      - .:/repo

  snapshot-cron:
    build: .
    container_name: hormuz-snapshot
    restart: unless-stopped
    entrypoint: /bin/bash
    command:
      - -c
      - |
        apt-get update -qq && apt-get install -y -qq git cron >/dev/null 2>&1
        echo "0 0,6,12,18 * * * /bin/bash /app/src/auto_push.sh >> /var/log/snapshot.log 2>&1" | crontab -
        /bin/bash /app/src/auto_push.sh >> /var/log/snapshot.log 2>&1 || true
        cron -f
    env_file:
      - .env
    environment:
      - GITHUB_TOKEN=${GITHUB_TOKEN}
      - GITHUB_REPO=${GITHUB_REPO:-yasumorishima/hormuz-ship-tracker}
    volumes:
      - ./data:/app/data
      - .:/repo
    depends_on:
      - ais-collector
```

`ais-collector`がWebSocket受信 + FastAPIを担当し、`snapshot-cron`がcronでスナップショット生成 + git pushを担当します。SQLiteファイル（`data/ais.db`）はボリュームマウントで共有しています。

## 起動方法

```bash
# .envファイルにAPIキーとGitHubトークンを設定
echo "AISSTREAM_API_KEY=your-api-key" > .env
echo "GITHUB_TOKEN=your-github-token" >> .env
echo "GITHUB_REPO=your-username/hormuz-ship-tracker" >> .env

# 起動
docker compose up -d

# ログ確認
docker logs -f hormuz-tracker
```

ブラウザで `http://<ラズパイのIP>:8002` にアクセスすると地図が表示されます。

## 設計判断のメモ

- **なぜSQLite**: 単一ファイルで管理が簡単。Raspberry Piのリソース制約上、PostgreSQLは不要。aiosqliteで非同期アクセス可能
- **なぜインメモリキャッシュ**: ShipStaticDataとPositionReportは別メッセージで到着する。SQLiteをJOINで引くよりdict参照のほうが高速
- **なぜSHA256比較**: 船舶の位置が変わっていない時間帯（夜間など）に無駄なgit pushを避ける
- **なぜコレクターとAPIを同一プロセスで実行**: asyncio.gatherで並行実行すれば1プロセスで済む。コンテナを分けるほどの規模ではない
- **なぜ海岸線を近似ポリゴンで描画**: スナップショットにshapefileライブラリの依存を入れたくなかった。視覚的な位置把握ができれば十分

## 今後の課題

- SQLiteの定期パージ（古いレコードの削除）
- 航路密度のヒートマップ可視化
- 特定船舶のアラート通知

## まとめ

aisstream.ioのWebSocket APIを使えば、特定海域の船舶をリアルタイムに収集・可視化するシステムを比較的少ないコードで構築できます。Raspberry Pi 5は既に他のAPIコンテナが稼働していますが、AISデータの収集程度であればリソース的に問題ありませんでした。

データソース: [aisstream.io](https://aisstream.io/)
