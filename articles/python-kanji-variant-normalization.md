---
title: "「宮﨑」と「宮崎」は別人扱い — Pythonで日本語スポーツデータの表記揺れに向き合った"
emoji: "⚾"
type: "tech"
topics: [python, データ分析, 野球, pandas]
published: false
---

## きっかけ

NPB選手成績予測ダッシュボードに「選手名で検索するとwRC+の複数年推移グラフが表示される」機能を実装しました。

テスト中、**「宮崎」で検索すると複数選手がヒットするのに、グラフが表示されるのは1選手だけ**という問題に気づきました。

原因を調べると、Pythonの文字列比較では **同じ字に見えて別の文字コード**のケースが混在していました。

---

## CJK互換漢字とは

Unicodeには、印刷上は同じ字形でも**異なるコードポイントが割り当てられている漢字**が存在します。

| 表示 | コードポイント | 区分 |
|---|---|---|
| 崎 | U+5D0E | CJK統合漢字（標準字） |
| 﨑 | U+FA11 | CJK互換漢字（互換字） |
| 高 | U+9AD8 | CJK統合漢字（標準字） |
| 髙 | U+9AD9 | CJK統合漢字 |

NPBの公式データでは、選手の本名表記に忠実なためか、年度によって使用する文字コードが異なることがあります。

```python
a = "宮﨑 敏郎"  # 﨑 はU+FA11
b = "宮崎 敏郎"  # 崎 はU+5D0E

print(a == b)       # False
print(a[1] == b[1]) # False
print(hex(ord(a[1])), hex(ord(b[1])))  # 0xfa11 0x5d0e
```

Pythonの `==` はコードポイントで比較するため、**目で見て同じ字でも別文字として扱われます**。

---

## データを確認する

NPBの打者データ（2015〜2025年）を確認すると：

```python
import pandas as pd

df = pd.read_csv("npb_batting_detailed_2015_2025.csv")
miyazaki = df[df["player"].str.contains("宮")]
print(miyazaki["player"].value_counts())
```

```
宮﨑 敏郎    10   ← 互換漢字（2016〜2025）
宮崎 敏郎     1   ← 標準字（2015）
*宮崎 竜成    1   ← 先頭にアスタリスクまで混入
宮崎 一樹     2
```

同一選手が**2種類の名前で登録されていた**のです。

`sabermetrics.py` でwRC+を計算する際、正規化せずに保存していたため、CSVにも2つの別エントリとして出力されていました。

---

## 解決策: str.maketrans で一括変換

Pythonの `str.maketrans()` と `str.translate()` を使えば、異体字の一括変換を効率よく実装できます。

```python
# 互換漢字 → 標準字の変換テーブル
VARIANT_MAP = str.maketrans("﨑髙濵澤邊齋齊國島嶋櫻", "崎高浜沢辺斎斉国島島桜")

def normalize_player_name(name: str) -> str:
    """
    選手名を正規化する
    - CJK互換漢字 → 標準字
    - データソース由来の先頭 * を除去
    - 全角スペース → 半角スペース
    """
    return (
        str(name)
        .replace("\u3000", " ")  # 全角スペース → 半角
        .strip()
        .lstrip("*")             # 先頭のアスタリスクを除去
        .strip()
        .translate(VARIANT_MAP)  # 異体字を統一
    )
```

データ読み込み直後に適用します：

```python
df = pd.read_csv("npb_batting_detailed_2015_2025.csv")
df["player"] = df["player"].apply(normalize_player_name)
```

---

## 効果

正規化してCSVを再生成すると：

```python
miyazaki = df[df["player"].str.contains("宮")]
print(miyazaki["player"].value_counts())
```

```
宮崎 敏郎    11   ← 2015〜2025の11年分に統合
宮崎 一樹     2
宮崎 竜成     1   ← * も除去された
宮崎 祐樹     1
```

宮崎敏郎選手のwRC+推移グラフが、1年分から**11年分**として正しく表示されるようになりました。

---

## 検索側でも正規化する

CSVの正規化だけでなく、検索クエリ側でも同じ変換を適用しておくと、ユーザーがどちらの文字で入力しても一致します。

```python
def _fuzzy(s: str) -> str:
    """スペース除去 + 異体字を統一（検索クエリ用）"""
    return s.replace(" ", "").replace("\u3000", "").translate(VARIANT_MAP)

def search_player(df: pd.DataFrame, name: str) -> pd.DataFrame:
    q = _fuzzy(name)
    return df[df["player"].apply(lambda p: q in _fuzzy(p))]
```

---

## 同様に注意が必要な文字

今回対応したのは野球データでよく見かけるものですが、日本語データ全般で起きうるケースです：

| 互換字 | 標準字 | よく見る選手名 |
|---|---|---|
| 﨑 (U+FA11) | 崎 | 宮﨑、髙﨑 など |
| 髙 (U+9AD9) | 高 | 髙橋、髙山 など |
| 濵 | 浜 | 濵田 など |
| 澤 | 沢 | 澤村、澤本 など |

---

## まとめ

- 日本語データの選手名には**目で見て同じでも別コードポイントの漢字**が混在することがある
- Pythonの `==` はコードポイント比較なので、正規化しないと**集計・グラフ描画で欠落**が起きる
- `str.maketrans` + `str.translate` でデータパイプラインに一行で対策できる
- **データ保存前**と**検索クエリ**の両方で正規化するのがベスト

データの見た目だけ信じると静かにバグります。Unicodeは奥が深いです。

---

今回の実装は [npb-prediction](https://github.com/yasunorim/npb-prediction) で公開しています。
実際のダッシュボードは [npb-prediction.streamlit.app](https://npb-prediction.streamlit.app/) で動いています。
