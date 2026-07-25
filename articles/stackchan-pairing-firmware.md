---
title: "スタックちゃんがアプリとペアリングできない原因は出荷時ファームだった（USB書き込みで解決）"
emoji: "🤖"
type: "tech"
topics: ["m5stack", "esp32", "iot", "bluetooth", "ble"]
published: false
---

## 結論

M5Stack の公式スタックちゃん（`M5STACK-K151`）を買って開封し、スマートフォンアプリ「StackChan World」で初期設定をしようとしたら、デバイスを選んだ後に `No devices found` と出て設定が完了しませんでした。

原因は **Bluetooth の権限でもアプリの不具合でもなく、出荷時ファームウェアが古かったこと**でした。USB 経由で公式の最新ファームウェアを書き込んだら、一度で最後まで通りました。

しかもこの問題は**放っておくと自力で抜け出せない構造**になっています。

- 新しいファームウェアは OTA で入る
- OTA には Wi-Fi 接続が必要
- Wi-Fi の設定にはアプリのペアリングが必要
- **そのペアリングが古いファームウェアで失敗する**

つまり **USB からの書き込み以外に出口がありません**。

## 環境と時期

| | |
|---|---|
| 製品 | M5StackChan AI デスクトップロボット（ESP32-S3 搭載） / `M5STACK-K151`（公式完成品） |
| 入手 | 2026-07-24（スイッチサイエンス） |
| 作業日 | 2026-07-25 |
| 出荷時ファーム | **1.2.4**（2026-04-20 公開） |
| 作業日時点の公式最新 | **1.4.4**（2026-07-13 公開） |
| 母艦 | Windows 11 |
| スマートフォン | Android |

**時期がかなり重要な話です。** 本製品は 2026-05 発売の新製品で、ファームウェアもアプリも更新が続いています。**入手時期によって出荷時のファーム版数は変わる**ので、この記事の症状に当たるかどうかも時期によって変わります。逆に言えば「発売から数ヶ月経った在庫を買った」場合は、同じ罠を踏む可能性があります。

## 症状

初期設定の流れは次のとおりです。

1. 本体を起動してサーボテストを通す
2. 本体画面に 12 桁の ID が表示される
3. アプリで「Add a new StackChan」→ 近くのデバイスをスキャン
4. 本体画面と一致する ID を選ぶ
5. デバイス名 → AI エージェント → Wi-Fi を設定

**4 で ID を選んだ直後に `No devices found`** となり、5 に進めませんでした。

## 効かなかった対処

「デバイスが見つからない」という文言なので、当然スキャン周りを疑いました。以下は全部やりましたが、症状は一切変わりませんでした。

- アプリへの Bluetooth 権限・位置情報権限の付与（OS の設定アプリ側からも確認）
- 位置情報サービスの有効化（Android の BLE スキャンは位置情報が絡む）
- アプリの強制終了・再起動、スマートフォンの再起動
- 本体の再起動、ID 表示画面に戻ってからのスキャン
- Bluetooth の OFF / ON

ここで一点、**やってはいけないこと**も分かりました。**OS の Bluetooth 設定画面からペア設定してはいけません。** 確認のつもりで OS 側からペアリングすると、ボンディング済みデバイスとして登録され、**アプリ側の「新しいデバイスを探す」一覧から外れます**。私は一度これをやってしまい、状況をさらに分かりにくくしました。登録してしまった場合は削除してください。

## シリアルログを取ったら話が全然違った

権限を疑うのをやめて、USB シリアルのログを取りました。本体を PC に USB 接続すると、CoreS3（ESP32-S3）のネイティブ USB がシリアルポートとして見えます（`VID_303A` / `PID_1001`、USB-Serial/JTAG）。

その状態でアプリからペアリングを試したときのログがこれです。

```
I (111140) NimBLE: connection established; status=0
I (111160) NimBLE:  peer_ota_addr_type=1 peer_ota_addr=
I (111160) NimBLE: 63:a2:04:c7:fa:5d
[info] [WifiConfigServer] app Connected
I (112260) NimBLE: Stack-Chan characteristic write; conn_handle=1 attr_handle=22
I (112260) NimBLE: Config data received (42 bytes): {"cmd":"handshake","data":"1784964126892"}
I (112420) NimBLE: GATT procedure initiated: notify;
I (112430) NimBLE: notify_tx event; conn_handle=1 attr_handle=22 status=0 is_indication=0
I (112440) NimBLE: Config notification sent
```

読めることは次のとおりです。

- **BLE 接続は成立している**（`connection established; status=0`）
- **アプリは本体に接続できている**（`app Connected`）
- **アプリはハンドシェイクを送っている**（`{"cmd":"handshake"}`）
- **本体は応答も返している**（`Config notification sent`）
- そして**この直後、アプリからは何も来ません**。切断ログすら出ません

つまり `No devices found` という表示は実態と違っていて、**止まっていたのは「探索」ではなく「本体が返した応答の検証」**でした。表示に引っ張られて権限を疑い続けたのが遠回りの原因です。

念のためアプリ側の実装も確認しました。スタックちゃんのファームウェア・アプリ・サーバーは [m5stack/StackChan](https://github.com/m5stack/StackChan) で公開されていて、アプリ（Flutter）のソースも同じリポジトリに入っています。デバイス選択画面の実装には接続 30 秒・検証 20 秒のタイムアウトがあり、失敗時のメッセージとして `Device did not return encryption data.` や `Device verification decryption failed.` が用意されています。検証を通過してからサーバーにデバイスを登録する流れです。

## 原因はファーム版数、そして詰みの構造

シリアルの起動ログに版数が出ます。

```
I (690) app_init: Project name:     stack-chan
I (696) app_init: App version:      1.2.4
I (700) app_init: Compile time:     Apr 15 2026 09:18:33
```

**1.2.4**。作業日時点の公式最新は **1.4.4** で、9 バージョン分の開きがありました。

ここで冒頭に書いた循環にはまります。OTA には Wi-Fi、Wi-Fi にはペアリング、ペアリングは古いファームで失敗。**USB で書き込む以外に更新経路がありません。**

なお **自分でソースからビルドしても解決しません**。公開リポジトリのハンドシェイク実装（`firmware/main/hal/utils/secret_logic/secret_logic.cpp`）はスタブで、トークンを返す関数が固定文字列を返すだけです。weak シンボルとして宣言されていて、公式ビルド時に本物の実装が差し込まれる構造になっています。したがって**アプリの検証を通せるのは公式バイナリだけ**です。

## 解決: M5Burner を使わず CLI で公式ファームを書き込む

公式の書き込みツールは M5Burner（GUI）ですが、**M5Burner が使っているファームウェア配信 API は公開されている**ので、CLI だけで完結できます。Python も不要です。

### 1. ファーム一覧を取得する

```bash
curl -s "https://m5burner-api.m5stack.com/api/firmware" \
  | jq -r '.[] | select(.name=="StackChan-UserDemo") | .versions[] | "\(.version)  \(.published_at)  \(.file)"'
```

出荷時ファームウェアの名前は **`StackChan-UserDemo`**（author: M5Stack）です。出力はこうなります。

```
V1.2.4   2026-04-20  3c8ffe6be0ca26375d836e10e06e3609.bin
V1.4.3   2026-07-02  fb75fa818e63b7ee6b0d35eba308f386.bin
V1.4.4   2026-07-13  790e3fcde496020aa7f188153b23e6f0.bin
```

旧バージョンも残っているので、**書き戻し（ロールバック）も可能**です。これが分かっていると気楽に更新できます。

### 2. バイナリを取得する

```bash
curl -sL "https://m5burner-cdn.m5stack.com/firmware/790e3fcde496020aa7f188153b23e6f0.bin" \
  -o stackchan-v1.4.4.bin
```

先頭バイトが `e9`（ESP イメージのマジックナンバー）であれば、オフセット `0x0` から書くマージ済みのフルイメージです。

```bash
od -An -tx1 -N 1 stackchan-v1.4.4.bin
#  e9
```

### 3. esptool を用意する（Python 不要）

Espressif が単体実行ファイルを配布しています。

```bash
gh api repos/espressif/esptool/releases/latest \
  --jq '.assets[] | select(.name|test("windows-amd64")) | .browser_download_url'
```

展開すると `esptool-windows-amd64/esptool.exe` が入っています。

### 4. まず読み取りだけで通信を確認する

```powershell
.\esptool.exe --port COM3 flash-id
```

```
Detecting chip type... ESP32-S3
Chip type:          ESP32-S3 (QFN56) (revision v0.2)
USB mode:           USB-Serial/JTAG
Detected flash size: 16MB
```

**ダウンロードモードへの手動操作は不要でした**（USB-Serial/JTAG 経由で esptool が自動リセットします）。接続できないときだけ、リセットボタンを約 2 秒長押しし、内部の緑色 LED が点灯したら離してダウンロードモードに入れます。

### 5. 書き込む

```powershell
.\esptool.exe --port COM3 write-flash 0x0 stackchan-v1.4.4.bin
```

```
Wrote 12783792 bytes (3185393 compressed) at 0x00000000 in 47.7 seconds
Verifying written data...
Hash of data verified.
```

12.8MB 弱で 50 秒程度でした。

### 6. 版数を確認する

シリアルの起動ログで `app_init: App version: 1.4.4` を確認できれば完了です。この状態でアプリからペアリングをやり直したら、**一度で通り**、AI エージェント設定 → Wi-Fi 設定 → 音声での会話まで完走しました。

## ほかに踏んだ罠

### シリアルポートを開くと本体が再起動する

ログを取ろうとポートを開いた瞬間、本体が再起動します。

```
rst:0x15 (USB_UART_CHIP_RESET),boot:0x2b (SPI_FAST_FLASH_BOOT)
```

**ログ取得と本体操作は同時にできません。** 初期設定の途中でログを取ろうとすると、ウィザードをやり直すことになります（私はこれで一度巻き戻しました）。逆に「起動ログを頭から取りたい」ときは、ポートを開いたままリセットを掛ければ取れます。

### インタラクティブなシリアルコンソールは無い

`help` などを送っても応答はなく、書き込み自体がブロックします。シリアルは読み取り専用として扱うのが正解です。

### 2 つの USB-C ポートに同時給電しない

本体には CoreS3 側とかかと（ベース）側の 2 つの USB-C ポートがあります。CoreS3 側から入れた 5V は M-BUS の `BUS_5V` を通ってベース側へ回るため、**両方に電源を挿すと同一レールで 2 つの電源がぶつかります**。公式ドキュメントの「双方から給電を試す」は、片方ずつ試すという意味です。給電と書き込みはかかと側のポートが推奨されています。

### サーボのゼロ点はフラッシュ全書き換えでも消えない

書き込み前に NVS のバックアップを取っていたのですが、書き込み後も同じゼロ点が読み出されました（`[ScsServo] id: 1 get zero pos: 460 from settings`）。**この値はサーボ本体側に保持されている**ようで、フラッシュを全書き換えしても失われませんでした。

## 充電については別の注意がある

ファームウェアは起動時に AXP2101（電源管理 IC）の充電電流を **700mA** に設定します。

```cpp
auto ret = setChargerConstantCurr(XPOWERS_AXP2101_CHG_CUR_700MA);
```

ESP32-S3 は USB 2.0 デバイスとして動作するため、**PC の USB ポートから取れるのは 500mA が上限**です。本体の消費（液晶・カメラ・サーボ・無線）と合わせると入力上限を超えるので、**PC 給電では充電が始まりません**。給電には 5V で余裕のある USB 充電器を使ってください。

サーボが現在位置を返さない場合（`[ScsServo] ignore invalid current pos: -1`）も、電力不足のサインとして読めます。

## まとめ

- アプリの `No devices found` は**実態と違う表示**でした。BLE 接続もハンドシェイクも成功していて、止まっていたのは応答の検証段階でした
- 原因は**出荷時ファームウェアが古いこと**。USB で公式最新を書き込んだら一度で通りました
- OTA ↔ Wi-Fi ↔ ペアリングが循環しているので、**USB 書き込み以外に出口がありません**
- **表示されるエラー文言を信じすぎず、ログを取る。** これが一番の教訓でした。権限を何度も見直した時間は、ログを 1 回取れば不要だったものです

手順とスクリプトはリポジトリにまとめてあります。

- https://github.com/yasumorishima/stackchan-lab

### 参考

- 公式ドキュメント: https://docs.m5stack.com/ja/StackChan
- Arduino での開発（ダウンロードモードの手順あり）: https://docs.m5stack.com/ja/arduino/stackchan/program
- ファームウェア・アプリ・サーバーのソース: https://github.com/m5stack/StackChan
