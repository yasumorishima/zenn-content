---
title: "pybaseballが未対応のBaseball Savantリーダーボードを全部Python化した"
emoji: "⚾"
type: "tech"
topics: ["python", "pypi", "baseball", "mlb", "savant"]
published: true
---

## はじめに

[pybaseball](https://github.com/jldbc/pybaseball) は Baseball Savant のデータを Python で扱う定番ライブラリですが、対応していないリーダーボードが多数あります。

- Pitch Tempo（投球テンポ）
- Arm Strength（送球速度）
- Pitch Movement（球の変化量）
- Catcher Blocking / Throwing / Stance
- Baserunning / Basestealing
- Timer Infractions（ピッチクロック違反）
- など

これらを Python から取得できるようにしたのが **[savant-extras](https://github.com/yasumorishima/savant-extras)** です。

## バージョン履歴

| バージョン | 追加内容 |
|---|---|
| v0.1.0 | bat_tracking（日付範囲指定対応） |
| v0.2.0 | pitch_tempo, arm_strength |
| v0.3.0 | 残り13リーダーボードを一括追加 |
| v0.3.1 | Known Issues ドキュメント追加 |

現在 **16リーダーボード・33関数** に対応しています。

## 対応リーダーボード一覧

| リーダーボード | 関数例 | カテゴリ |
|---|---|---|
| Bat Tracking | `bat_tracking()` | Batting |
| Batted Ball | `batted_ball()` | Batting |
| Home Runs | `home_runs()` | Batting |
| Year-to-Year | `year_to_year()` | Batting |
| Pitch Tempo | `pitch_tempo()` | Pitching |
| Pitch Movement | `pitch_movement()` | Pitching |
| Pitcher Arm Angle | `pitcher_arm_angle()` | Pitching |
| Running Game | `running_game()` | Pitching |
| Timer Infractions | `timer_infractions()` | Pitching |
| Arm Strength | `arm_strength()` | Fielding |
| Catcher Blocking | `catcher_blocking()` | Catching |
| Catcher Throwing | `catcher_throwing()` | Catching |
| Catcher Stance | `catcher_stance()` | Catching |
| Baserunning | `baserunning()` | Baserunning |
| Basestealing | `basestealing()` | Baserunning |
| Swing & Take | `swing_take()` | Batting ※ |

※ Swing & Take は Baseball Savant の CSV エンドポイントが現在空データを返す状態のため、取得できません（Known Issue）。

## インストール

```bash
pip install savant-extras
```

## 使い方

### Batting: Bat Tracking

```python
from savant_extras import bat_tracking

df = bat_tracking("2025-04-01", "2025-09-30", min_swings=100)
print(df[["name", "avg_bat_speed", "attack_angle"]].head(5))
```

### Pitching: Pitch Tempo

```python
from savant_extras import pitch_tempo

df = pitch_tempo(2025)
print(df[["entity_name", "median_seconds_empty"]].head(5))
```

### Pitching: Pitch Movement

```python
from savant_extras import pitch_movement

df = pitch_movement(2025)
# 四シームの変化量上位
ff = df[df["pitch_type"] == "FF"].sort_values("pitcher_break_z", ascending=False)
print(ff[["last_name, first_name", "avg_speed", "pitcher_break_z", "pitcher_break_x"]].head(5))
```

### Fielding: Arm Strength

```python
from savant_extras import arm_strength

df = arm_strength(2025)
print(df[["fielder_name", "primary_position_name", "max_arm_strength", "arm_overall"]].head(5))
```

### Catching: Catcher Throwing

```python
from savant_extras import catcher_throwing

df = catcher_throwing(2025)
print(df[["player_name", "pop_time", "arm_strength", "caught_stealing_above_average"]].head(5))
```

### Baserunning

```python
from savant_extras import baserunning, basestealing

df_run = baserunning(2025)
df_steal = basestealing(2025)
```

### Timer Infractions

```python
from savant_extras import timer_infractions

df = timer_infractions(2025)
print(df[["entity_name", "all_violations", "pitcher_timer", "batter_timer"]])
```

## Kaggle Notebook / Dataset

全リーダーボードを使ったサンプル分析を Kaggle で公開しています。

- **Notebook**: https://www.kaggle.com/code/yasunorim/savant-extras-all-baseball-savant-leaderboards
- **Dataset（15 CSV、2024+2025）**: https://www.kaggle.com/datasets/yasunorim/baseball-savant-leaderboards-2024

## Known Issues

**Swing & Take** リーダーボードは、Baseball Savant の CSV エンドポイントが全年度でヘッダーのみ（データなし）を返す状態になっています。API 側の問題のため、修正されるまで取得できません。他の15リーダーボードは正常に取得できます。

## リンク

- **PyPI**: https://pypi.org/project/savant-extras/
- **GitHub**: https://github.com/yasumorishima/savant-extras
