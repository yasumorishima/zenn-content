---
title: "ホルムズ海峡の船舶データを可視化したら、封鎖の影響が数字に表れていた"
emoji: "🚢"
type: "tech"
topics: ["python", "raspberrypi", "docker", "fastapi", "ais"]
published: false
---

:::message alert
**データの範囲について**: 本記事のデータは[aisstream.io](https://aisstream.io/)の**陸上AIS受信局**に基づくものです。海峡中央部など沖合のカバレッジは限られており、衛星AISを含めた全体像とは異なる可能性があります。記事中の数値は全て**2026年3月中旬時点**の観測値であり、状況は日々変化しています。
:::

## この記事について

2026年3月、ホルムズ海峡の通航が大幅に制限される状況が報じられています。世界の石油輸送の約20%が通過するこのチョークポイントで何が起きているのか——Raspberry Pi 5と無料のAISデータを使って、リアルタイムモニタリングシステムを構築し、実際のデータを観測しています。

本記事では、構築した仕組みと、そこから見えてきたデータの特徴を紹介します。

**リポジトリ**: [yasumorishima/hormuz-ship-tracker](https://github.com/yasumorishima/hormuz-ship-tracker)

![ペルシャ湾全域の船舶分布（3月中旬時点） — UAE沿岸に集中し、海峡中央部はほぼ空白](https://raw.githubusercontent.com/yasumorishima/hormuz-ship-tracker/7188e89/docs/snapshot_latest.png)
*ペルシャ湾全域のスナップショット（6時間ごと自動生成）。ゲートラインの位置、transit IN/OUT統計、船種別の分布が表示されている。UAE沿岸に船舶が集中し、海峡中央部がほぼ空白であることが確認できる。*

## AISデータとは

AIS（Automatic Identification System）は、船舶が位置・速度・針路・船名・船種などをVHF帯で自動送信する国際安全システムです。300総トン以上の国際航海船舶に搭載が義務付けられています。

[aisstream.io](https://aisstream.io/)が世界中の陸上AIS受信局から収集したデータをWebSocket APIで無料配信しており、今回のデータソースとして利用しています。

## システム構成

```
aisstream.io (WebSocket)
  → Collector (AIS受信 + 陸地フィルタ + SQLite保存)
  → Analytics Engine (ゲートライン通過検知 + 船舶状態分類)
  → FastAPI + Leaflet.js + Chart.js (ダッシュボード)
  → matplotlib (6時間ごとスナップショット → GitHub自動push)
```

Raspberry Pi 5上のDockerで2コンテナ（collector + snapshot-cron）を24時間稼働させています。

## データから見えること

### 停泊率 67%（3月中旬時点）

観測している約290隻のうち、約67%が停泊状態（速度0.5ノット未満）でした。通常の港湾エリアでは30〜40%程度と考えられるため、顕著に高い値です。

### 待機船団 — 35隻が6時間以上停泊（3月中旬時点）

6時間以上移動していない船舶を「待機船団」として集計すると、約35隻が該当しました。24時間以上動いていない船舶も11隻確認されています。

待機船団の国旗（MMSI MIDから推定）：

| 国旗 | 隻数 |
|---|---|
| パナマ | 9 |
| マーシャル諸島 | 3 |
| UAE | 3 |
| クウェート | 2 |
| その他（リベリア、マレーシア、韓国等） | 各1 |

パナマやマーシャル諸島は便宜置籍国（open registry）であり、大型商船やタンカーが多く登録されています。待機船団の中にタンカーが7隻含まれていた点も特徴的です。

### 海峡通過 — 陸上AISでの検出はほぼゼロ（3月中旬時点）

ホルムズ海峡の最狭部にゲートライン（仮想通過線）を設定し、船舶の通過を自動検知する仕組みを実装しました。24時間で検出されたのは1隻のみでした。

**ただし、これはaisstream.ioの陸上AIS受信局で捉えられた範囲に限られます。** 海峡中央部は沿岸受信局からの距離があり、カバレッジが限定的です。衛星AISであれば異なる結果が得られる可能性があります。実際に、報道ではトルコ船やインド船など限定的な通過が報じられていますが、陸上AISでは捕捉できていない可能性があります。

「データがないこと」＝「船がいないこと」ではない——この点は本記事の全データに当てはまる重要な前提です。

### UAE沿岸に集中する船舶（3月中旬時点）

データの大部分がDubai / Jebel Ali / Fujairah周辺に集中しています。この地域に3つのゲートラインを設置して分析しました。

| ゲート | INBOUND | OUTBOUND |
|---|---|---|
| Dubai / Jebel Ali Approach | 20 | 9 |
| Fujairah Approach | 0 | 7 |
| Strait of Hormuz | 0 | 1 |

Dubai港への入港が出港を大きく上回っている点、Fujairahは出港のみ（バンカリング＝燃料補給後の出航と推測される）という非対称パターンが観測されています。

## 技術的な実装

### ゲートライン通過検知

海峡やポートの入口に仮想ゲートライン（2点を結ぶ線分）を定義し、船舶の連続する位置レポートがこの線分を横切ったかどうかを計算幾何で判定しています。

```python
def segments_intersect(p1, p2, p3, p4):
    """線分p1-p2と線分p3-p4が交差するか判定"""
    d1 = cross_product(p3, p4, p1)
    d2 = cross_product(p3, p4, p2)
    d3 = cross_product(p1, p2, p3)
    d4 = cross_product(p1, p2, p4)
    if ((d1 > 0 and d2 < 0) or (d1 < 0 and d2 > 0)) and \
       ((d3 > 0 and d4 < 0) or (d3 < 0 and d4 > 0)):
        return True
    return False
```

通過方向（INBOUND/OUTBOUND）はゲートベクトルに対する外積の符号で判定し、同一船舶の6時間以内の重複検知を除外しています。

### MMSI → 国旗マッピング

AISのMetaDataにcountry_codeフィールドが存在しないケースがあったため、MMSI番号の上位3桁（MID: Maritime Identification Digits）から国旗を推定する方式に切り替えました。100カ国以上のMIDに対応しています。

### データ駆動の状況判定

ダッシュボードの表示テキストは全てデータから自動生成されます。海峡通過数・停泊率・待機船団数に基づいて状況レベル（normal / elevated / high / critical）を判定し、UIの色やメッセージが自動的に変化します。

```python
if strait_transits == 0 and anchored_pct > 40:
    return {"level": "critical", "title": "Strait Transit Suspended", ...}
elif 0 < strait_transits <= 5:
    return {"level": "elevated", "title": "Limited Strait Transit", ...}
else:
    return {"level": "normal", "title": "Monitoring Active", ...}
```

この設計により、危機が解消されれば表示も自動的に通常モードに戻ります。

### 陸地フィルタ

AISデータにはGPS精度の問題や建物設置のAIS中継器による陸上位置データが混入します。Natural Earth 10mの陸地ポリゴンとShapelyのprepared geometryを使い、陸上の位置を除外しています。

### AIS destination正規化

AISの目的地フィールドは自由入力のため、同じ港が多数の表記で送信されます（DUBAI / AE DXB / AEDXB / DMC DUBAI 等）。40以上のバリアントを20の正規名にマッピングしています。

## 制約と注意点

- **陸上AISの限界**: 無料のaisstream.ioは陸上受信局ベース。海峡中央部や沖合のカバレッジは限られる
- **AIS速度102.3ノット**: AIS仕様で「速度利用不可」を示すセンチネル値（0x3FF）。異常値ではなくフィルタが必要
- **2分間隔のスロットリング**: 同一船舶のデータを2分間隔に間引いているため、高速通過の検出精度に影響する可能性がある
- **データ収集期間**: 現時点で数日分のデータ。長期的なトレンド分析には蓄積が必要

## まとめ

aisstream.ioの無料APIとRaspberry Pi 5を使って、ペルシャ湾全域の船舶をリアルタイムに収集・分析するシステムを構築しました。停泊率の高さ、待機船団の存在、海峡通過数の少なさなど、現在の海上交通の状況がデータとして観測されています。

データは蓄積を続けており、今後は時系列での変化を追跡できる見込みです。

データソース: [aisstream.io](https://aisstream.io/) / 陸地ポリゴン: [Natural Earth](https://www.naturalearthdata.com/)
