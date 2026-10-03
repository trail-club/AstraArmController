# AstraArmController

AhaRobot のアーム 1 本ぶんを制御する ESP32 ファームウェア。上位機 (Ubuntu +
ROS 2) と UART で繋がり、Feetech STS3215 サーボを 12 個 (対向駆動 2 関節 × 4 個
+ 単発関節 4 個) 束ねて位置・速度・負荷を制御する。

このリポジトリは
[AstraFirmwares](https://github.com/trail-club/AstraFirmwares) から submodule
として使うことを想定しているが、単独クローンでもビルドできる。

## ハードウェア

| 項目 | 値 |
| --- | --- |
| 制御基板 | Waveshare "Servo Driver with ESP32" (ESP32-D0WD-V3, 4MB flash) |
| サーボ | Feetech STS3215 (model 777) × 12 個 |
| ホストとの通信 | UART0 (USB経由), 921600 bps |
| サーボバス | UART1 (GPIO18=RXD / GPIO19=TXD), 1 Mbps 半二重 |

サーボ ID の対応:

| 系統 | ID |
| --- | --- |
| joint0 (対向 4) | 4, 5, 6, 7 |
| joint1 (対向 4) | 8, 9, 10, 11 |
| 単発関節 | 12, 13, 14, 15 (15 はグリッパ) |

## リポジトリ構成

```
AstraArmController/
├── platformio.ini         PlatformIO 設定 (ボード種別・依存ライブラリ)
├── include/               ヘッダ (宣言と定数)
│   ├── main.h             グローバル定数・型
│   ├── comm.h             ホストとの通信プロトコル
│   ├── config.h           EEPROM に保存する設定
│   ├── dualMotor.h        対向駆動関節の制御
│   ├── trapTraj.hpp       台形速度プロファイル
│   └── EIShaper.h         Input Shaper (振動抑制フィルタ)
├── src/                   実装本体
│   ├── main.cpp           setup() / loop() エントリポイント
│   ├── comm.cpp           UART でホストと packet 送受信
│   ├── config.cpp         EEPROM のロード/セーブ
│   ├── dualMotor.cpp      対向 4 個の同期制御・位置平均化
│   ├── trapTraj.cpp       軌道生成
│   └── EIShaper.cpp       Input Shaper 実装
├── lib/SCServo/           Feetech 公式 SDK (STS/SCS プロトコル)
├── scripts/               ホスト側の Python (ログ解析・プロット)
└── test/                  unit test 置き場
```

### 起動時の挙動 (`src/main.cpp` `setup()` → タイマ割り込み)

1. `Serial.begin(921600)` — ホストとの UART
2. `Serial1.begin(1000000, ..., 18, 19)` — サーボバス
3. EEPROM から config をロード (`Config read` が Serial に出る)
4. タイマ登録後、~66 Hz で:
   - `read_pos()` … 全サーボの位置を読み、対向 4 個は符号を合わせて平均を取り関節位置とする
   - 軌道生成 → PID → PWM 出力を計算
   - 対向 4 個へ **符号付き** で個別指令 (符号は `JOINT_SERVO_SIGN[]` テーブル)
   - 単発関節にも指令 (欠番スロットは飛ばす)
5. `comm.cpp` が並行して、ホストからの目標指令を受信・現在状態を送出

## ビルドと焼き込み

### 必要なもの

- Python 3.9 以上
- USB-シリアル (Waveshare 基板が繋がった状態)

### PlatformIO の導入

`pipx` で入れるのが一番手軽で衝突しない。無ければ `python3 -m pip install --user pipx` で入れる。

```bash
pipx install platformio
```

これで `pio` コマンドが使える。バージョン確認:

```bash
pio --version   # PlatformIO Core, version 6.x
```

alternative: グローバルを汚したくない場合は仮想環境でも同じ。

```bash
python3 -m venv .venv
. .venv/bin/activate
pip install platformio
```

### ビルド

```bash
pio run
```

初回は ESP32 プラットフォーム (espressif32) と依存ライブラリ (Adafruit
SSD1306) をダウンロードするので数分かかる。成功すると
`.pio/build/esp32dev/firmware.bin` などが生成される。

### 焼き込み

`platformio.ini` は `upload_port` を固定していないので、PlatformIO が自動検出する。単一の USB シリアルデバイスが繋がっていれば追加指定なしで通る。

```bash
pio run -t upload
```

自動検出に失敗する / 複数デバイスがある場合は明示指定:

```bash
# Linux
pio run -t upload --upload-port /dev/ttyUSB0
# macOS
pio run -t upload --upload-port /dev/cu.usbserial-0001
# 環境変数でも可
PLATFORMIO_UPLOAD_PORT=/dev/cu.usbserial-0001 pio run -t upload
```

焼き込みで書き替えられる領域は 4 つ:

| アドレス | 内容 |
| --- | --- |
| `0x1000` | bootloader |
| `0x8000` | partitions table |
| `0xe000` | boot_app0 (OTA スイッチ) |
| `0x10000` | AstraArmController アプリ本体 |

サーボ自身の EEPROM (中点オフセット等) には触らない。事前にキャリブレーションで
書き込んだ中点はそのまま保たれる。

### 起動確認

```bash
pio device monitor -b 921600
```

`Config read` に続いて 9 個の設定値 (EIShaper, joint_vel_max, ...) がテキストで
出て、その後は状態パケットのバイナリが流れる (これは通常動作)。`Error reading
#X, checkout your wire connection` が出たら該当 ID のサーボと配線を確認。

## trail-club フォーク固有の変更点

以下は upstream の `waveshare/AstraArmController` からの差分。ハードコーディング
を実機に合わせて可変にした。

### `JOINT_SERVO_SIGN[]` (対向取付の符号)

対向 4 個の取付向きは組み付け規約によって変わる。upstream は
`{+1, -1, +1, -1}` を全関節に決め打ちしていたが、trail-club の実機は両関節とも
`{+1, -1, -1, +1}` (3・4 番目が逆)。**符号が違うと 4 個のうち 2 個が残り 2 個を
PWM 800/1000 で押し返して関節が固着する**。

`src/dualMotor.cpp`:

```c
const int JOINT_SERVO_SIGN[4 * JOINT_NUM] = {
  +1, -1, -1, +1,   // joint0: ID4, ID5, ID6, ID7
  +1, -1, -1, +1,   // joint1: ID8, ID9, ID10, ID11
};
```

機体が変わったら、AhaRobot 側の `tools/servo/teach_calibrate.py --verify` で
実測してこのテーブルを更新する。

### `NONE_JOINT_ID[]` (単発関節の欠番対応)

upstream は単発関節を `12, 13, 14, 15` の連番で決め打ちしていたが、機体によっては
サーボが欠けることがある。表にした:

```c
const int NONE_JOINT_ID[NONE_JOINT_NUM] = { 12, 13, 14, 15 };
```

**`-1` を入れるとその関節は読み書きを飛ばし、常に 2048 (=0 rad) を返す。**
自由度 (`NONE_JOINT_NUM = 4`) は変えないので、ホスト側の通信フォーマット・
グリッパの `arr[-1]` 添字・`JOINT_MIN/MAX` はそのまま。

## ホストとの接続前に把握しておくこと

`arm_controller.py` (ROS 側) の `do_init=True` が呼ぶ `setupTorque(128)` は
以下の EEPROM 書き替えを行う。**一度実行すると純正ファームに戻しても設定は残る。**

- ID4-11 の動作モードを PWM モードに書き換える
- グリッパ ID15 の中点を offset 1448 で強制的に書き換える
- バックラッシュ中点 `config.init_pos[]` を測定して ESP32 の EEPROM に保存

キャリブレーションで書き込んだ中点オフセットとは別の書き込みなので、初回のみ
実施し、以降は `do_init=False` で起動する運用が良い。

## 純正ファームに戻したいとき

AhaRobot 側リポジトリの `tools/firmware/backup/` に、焼き替え前に取得した
Waveshare 純正デモの 4MB 全域 dump が保存してある。戻し方は同ディレクトリの
README を参照。

## macOS で使うときの注意

`/dev/tty.usbserial-*` を開くと Silicon Labs (CP2102) ドライバが DTR/RTS を
アサートし、基板の自動リセット回路経由で ESP32 が EN=Low で保持されて起動
しない現象がある。以下で回避:

- 通信するときは `/dev/cu.usbserial-*` を使う (`/dev/tty.*` は DCD を待つので
  ハンドシェイクで詰まる)
- `screen` を使うと DTR/RTS を握るので使わない。代わりに `pio device monitor` か
  `python3 -m serial.tools.miniterm --dtr 0 --rts 0` を使う
