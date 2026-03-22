---
title: "Grafana × BigQuery で踏んだ落とし穴 5選 — ダッシュボードが表示されない原因と対処法"
emoji: "📉"
type: "tech"
topics: ["Grafana", "BigQuery", "GCP", "データ分析", "ダッシュボード"]
published: false
---

## はじめに

Grafana Cloud + BigQuery データソースで7つのダッシュボード（70パネル超）を構築した際に、「クエリは通るのにパネルが真っ白」「No data と表示される」といった問題に何度もハマりました。

この記事では、実際に遭遇した落とし穴とその対処法をまとめます。Grafana 13 + BigQuery データソースプラグインの組み合わせで確認しています。

## 1. 日本語エイリアスにはバッククォートが必須

### 症状

`Syntax error: Illegal input character` でパネルが表示されない。

### 原因

BigQuery の SQL で日本語のカラムエイリアスを使う場合、バッククォートで囲まないとエラーになります。

```sql
-- NG: エラーになる
SELECT team AS チーム, HR AS 本塁打 FROM ...

-- OK: バッククォートで囲む
SELECT team AS `チーム`, HR AS `本塁打` FROM ...
```

ASCII + 日本語が混在するエイリアス（`K率`、`BB率` など）も同様です。

```sql
-- NG
SELECT ROUND(prev_K_pct,1) AS K率

-- OK
SELECT ROUND(prev_K_pct,1) AS `K率`
```

### GROUP BY / ORDER BY も注意

日本語エイリアスを `GROUP BY` や `ORDER BY` で参照する場合も、バッククォートが必要です。

```sql
-- NG
SELECT LEFT(date,4) AS `年度`, COUNT(*) AS `プレイ数`
FROM ...
GROUP BY 年度 ORDER BY 年度

-- OK: カラム番号を使うか、バッククォートで囲む
GROUP BY `年度` ORDER BY `年度`
-- または
GROUP BY 1 ORDER BY 1
```

## 2. BigQuery データソースで `format: "time_series"` は使えない

### 症状

`error unmarshaling query JSON to the Query Model: invalid format value: time_series`

### 原因

Grafana の BigQuery データソースプラグインは `time_series` フォーマットをサポートしていません。クエリの `format` は常に `"table"` を指定します。

```json
// NG
"format": "time_series"

// OK
"format": "table"
```

時系列データの場合、`time` という名前の列を `TIMESTAMP` 型で返せば、Grafana が自動的に時間フィールドとして認識します。

```sql
SELECT CAST(date AS TIMESTAMP) AS time, value
FROM ...
```

## 3. 過去データの timeseries パネルは「Data outside time range」になる

### 症状

パネルに「Data outside time range」と表示され、データが見えない。

### 原因

timeseries パネルはダッシュボードの時間範囲（右上の "Last 6 hours" 等）でフィルタされます。2015〜2025年のような過去データは、現在時刻を基準とした時間範囲に含まれないため表示されません。

### 対処法

年度別の集計データなど、リアルタイム性のないデータは **barchart パネル**を使い、年を文字列として返します。

```sql
-- timeseries 用（リアルタイム向き）
SELECT CAST(date AS TIMESTAMP) AS time, value FROM ...

-- barchart 用（過去データ向き）
SELECT CAST(year AS STRING) AS year, value FROM ...
```

## 4. barchart の fieldConfig に余計なプロパティを入れると描画されない

### 症状

barchart パネルが完全に空白。クエリを直接実行するとデータは返る。エラーメッセージも出ない。

### 原因

Grafana 13 の barchart パネルで、`fieldConfig.defaults` に `color`、`decimals`、`unit` などのプロパティを入れると、パネルが描画されなくなることがあります。

```json
// NG: 描画されない
"fieldConfig": {
  "defaults": {
    "color": {"fixedColor": "#5470c6", "mode": "fixed"},
    "custom": {"axisLabel": "Players"},
    "decimals": 0,
    "unit": "none"
  }
}

// OK: 最小構成で描画される
"fieldConfig": {
  "defaults": {},
  "overrides": []
}
```

まず最小構成で動作確認してから、必要なプロパティを一つずつ追加するのが安全です。

## 5. 展開済み row の panels 配列にパネルがあると非表示になる

### 症状

パネルがダッシュボード上に見えない。JSON 上は存在している。

### 原因

Grafana の row パネルには2つの状態があります。

- **折りたたみ時（`collapsed: true`）**: 子パネルは row の `panels` 配列に格納
- **展開時（`collapsed: false`）**: 子パネルはトップレベルの `panels` 配列に並ぶ。row の `panels` は空

`collapsed: false` なのに row の `panels` 配列にパネルが入っていると、そのパネルは表示されません。

```json
// NG: collapsed=false なのに panels にパネルがある → 非表示
{
  "type": "row",
  "title": "セクション名",
  "collapsed": false,
  "panels": [
    {"type": "barchart", "title": "見えないパネル", ...}
  ]
}

// OK: panels は空にして、パネルは row の後にトップレベルで並べる
{
  "type": "row",
  "title": "セクション名",
  "collapsed": false,
  "panels": []
},
{"type": "barchart", "title": "見えるパネル", ...}
```

また、パネルの `gridPos.y` が row の `gridPos.y` より小さいと、ページ上で row の上に表示されてしまい、セクションの下に見えなくなります。

## おわりに

Grafana + BigQuery は強力な組み合わせですが、API 経由でダッシュボードを構築する場合、UI で作成する場合には遭遇しない問題に多くハマりました。特に「クエリは正しいのにパネルが空白」というパターンは原因特定が難しいため、この記事が参考になれば幸いです。
