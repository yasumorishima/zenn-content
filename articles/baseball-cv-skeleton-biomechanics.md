---
title: "Driveline C3Dデータで投球・打撃の3D骨格検知をやってみた"
emoji: "🦴"
type: "tech"
topics: ["python", "baseball", "データ分析", "機械学習", "oss"]
published: true
---

:::message
本記事で使用しているモーションキャプチャデータは [Driveline OpenBiomechanics Project](https://www.openbiomechanics.org/) のものです。ライセンスは **CC BY-NC-SA 4.0** です。

**引用**: Wasserberger KW, Brady AC, Besky DM, Jones BR, Boddy KJ. *The OpenBiomechanics Project: The open source initiative for anonymized, elite-level athletic motion capture data.* (2022).
ライセンス: https://creativecommons.org/licenses/by-nc-sa/4.0/

本記事の派生物（グラフ・GIF等）も同ライセンスに従います。
:::

## はじめに

野球選手のフォームを3Dで可視化し、関節の動きと球速の関係を探ってみました。

使ったのは以下の3つです。

- **Driveline OpenBiomechanics Project (OBP)** — プロレベルのモーションキャプチャC3Dデータ（100投手＋98打者）
- **ezc3d** — C3Dファイルの読み書きライブラリ（MITライセンス、[GitHub](https://github.com/pyomeca/ezc3d)）
- **matplotlib** — 3D可視化・アニメーション

→ **GitHub**: https://github.com/yasumorishima/baseball-cv

### ezc3dとの関わり

実はezc3dには[PR #384](https://github.com/pyomeca/ezc3d/pull/384)でバグ修正を貢献しています。自分がOSSに参加したライブラリを使って実際に分析するという流れになりました。

---

## Step 1: C3Dデータから3D骨格を可視化

C3Dファイルにはモーションキャプチャで取得した各マーカー（体の各部位）の3D座標が記録されています。

- **投球データ**: 45マーカー、360Hz、約726フレーム
- **打撃データ**: 55マーカー（体45＋バット10）、360Hz、約804フレーム

ezc3dで読み込んでmatplotlibで描画します。

```python
import ezc3d

c3d = ezc3d.c3d("pitching_sample.c3d")
points = c3d["data"]["points"]  # shape: (4, n_markers, n_frames)
labels = c3d["parameters"]["POINT"]["LABELS"]["value"]
```

### 投球フォームの骨格アニメーション

![投球骨格アニメーション](https://raw.githubusercontent.com/yasumorishima/baseball-cv/master/data/output/skeleton_pitching_anim.gif)

45個のマーカーを結んで骨格として描画しています。ワインドアップからリリースまでの一連の動きが見えます。

### 打撃フォームの骨格

![打撃骨格](https://raw.githubusercontent.com/yasumorishima/baseball-cv/master/data/output/skeleton_hitting_anim.gif)

打撃側は55マーカーで、バットの10マーカーは赤色で表示しています。

---

## Step 2: 一般動画でも骨格検知できる（MediaPipe）

モーションキャプチャ設備がなくても、スマホで撮った普通の動画から骨格を検知できます。GoogleのMediaPipe Poseを使うと、1本の動画から関節点33箇所をリアルタイムで自動検出できます。

```python
import mediapipe as mp

mp_pose = mp.solutions.pose
pose = mp_pose.Pose(
    static_image_mode=False,
    min_detection_confidence=0.5,
    min_tracking_confidence=0.5
)
```

本記事の分析はDriveline OBPのC3Dデータ（専用センサー）を使っていますが、手軽に試したい場合はMediaPipeが出発点として使えます。

---

## Step 3: 関節角度・角速度の抽出

各関節が「どれだけ動くか（角度）」と「どれだけ速く動くか（角速度）」を、投球全体のフレームを通じて計算しました。

### 投球（Pitching）

| 関節角度 | 最小値 | 最大値 |
|---|---|---|
| 肘屈曲 (Elbow Flexion) | 50.5° | 156.7° |
| 肩外転 (Shoulder Abduction) | 4.6° | 117.7° |
| 体幹回旋 (Trunk Rotation) | 0° | 58° |
| 膝屈曲 (Knee Flexion) | 99.1° | 163.8° |

例えば肘の屈曲は50〜157°の範囲で動いており、投球中に大きく曲げ伸ばしされています。体幹回旋は0〜58°と、腕に比べてそれほど大きくないように見えますが、後の分析で球速との相関が最も強い指標として浮かび上がります。

### 角速度の時系列（投球）

![関節角速度の時系列](/images/baseball-cv-kinematic-pitching.png)

各関節が1秒間に何度回転しているか（角速度）をフレームごとにプロットしたものです。投球のどの瞬間に、どの関節が特に速く動いているかが一目で分かります。

---

## Step 4: 骨格の動き × 球速の相関分析

Driveline OBPのC3Dデータには、ファイル名に球速情報が含まれています（例: `..._809.c3d` → 80.9 mph）。

16投手分のデータを使い、骨格の動きから取り出した指標（角度・速度・回旋量など）と球速の相関を調べました。

### 相関結果

![相関散布図](/images/baseball-cv-scatter.png)

![相関行列](/images/baseball-cv-correlation.png)

| 指標 | 相関（r） | p値 |
|---|---|---|
| 体幹の最大角速度 (Peak Trunk Angular Velocity) | 0.119 | 0.673 |
| 肘の最大角速度 (Peak Elbow Angular Velocity) | 0.094 | 0.739 |
| 肩の最大外転角 (Peak Shoulder Abduction) | 0.180 | 0.520 |
| **体幹の回旋量 (Trunk Rotation Range)** | **0.425** | **0.114** |

相関係数（r）は−1〜+1の値をとり、1に近いほど「一方が大きいとき、もう一方も大きい」という正の関係を示します。p値は0.05を下回ると統計的に確実とされますが、16人では少なすぎて断言はできません。それでも**体幹の回旋量が球速と最も強い正の相関**（r=0.425）を示しました。「体幹をどれだけ大きく回せるか」が球速に関係している可能性を示しています。サンプル数を増やせばより明確な結果が期待できます。

---

## 技術スタック

| 技術 | 用途 |
|---|---|
| ezc3d | C3Dファイル読み込み |
| MediaPipe Pose | 動画からの骨格検知 |
| matplotlib | 3D可視化・GIFアニメーション |
| numpy / scipy | 角度計算・線形補間 |

---

## まとめ

- Driveline OBPのC3Dデータからezc3dで3D骨格を可視化できた
- 関節角度・角速度の時系列を抽出し、投球フェーズごとの特徴を観察できた
- 体幹回旋のレンジが球速と最も高い相関（r=0.425）を示した
- MediaPipeを使えば一般動画からも骨格検知が可能（本記事の分析はモーションキャプチャC3Dを使用）

モーションキャプチャデータとStatcastの球質データを組み合わせた分析は、選手のパフォーマンス改善に役立つ可能性があります。

→ **GitHub**: https://github.com/yasumorishima/baseball-cv

**データ**: [Driveline OpenBiomechanics Project](https://www.openbiomechanics.org/)（CC BY-NC-SA 4.0）
**ezc3d**: [pyomeca/ezc3d](https://github.com/pyomeca/ezc3d)（MIT License）
