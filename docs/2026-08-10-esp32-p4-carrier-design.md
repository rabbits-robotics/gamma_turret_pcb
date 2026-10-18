# ESP32-P4キャリア基板（gamma_turret後継）設計メモ

日付: 2026-08-10
状態: 設計承認待ち（ピン配置・回路方式・BOM確定済み）
KiCadプロジェクト: このフォルダの `gamma_turret.kicad_pro` を新設計で置き換える（旧回路はコピー時の名残。旧ガーバーは gbr_*.zip に保存済み）

## 1. 目的と要件

ロボット競技システムの制御基板刷新。既存STM32F303K8/F407系基板群の後継として、ESP32-P4を核にした1枚基板を設計する。

| 項目 | 要件 |
|---|---|
| コア | M5Stack Stamp-P4（ESP32-P4NRW32・16MB Flash・32MB PSRAM） |
| CAN | 2系統（既存基板群との通信維持） |
| ESC | PWM×2（射出ローラー用） |
| 装填サーボ | ST3215用 半二重UARTバス1系統 |
| 上位通信 | Jetson Orin NanoとUART（3.3V TTL） |
| IMU | 秋月AE-BNO055をUART接続（**SPI非対応と判明しUARTに変更**） |
| 電源 | 24V入力→オンボードで5V生成。3.3VはStamp内蔵DCDCを利用。ST3215用7.4Vは別DCDC（基板はパススルー中継のみ） |
| アナログ入力 | なし（承認済み。電圧監視等はCAN経由で他基板から取得） |
| 無線 | 要件外（必要になればStamp-AddOn C6を後付け。ソケット下クリアランス要実機確認） |

## 2. コアボードの搭載方式

- **Stamp-P4を2.54mmピンソケットで搭載**（NUCLEO方式の差し替え運用）
- DIP用スルーホールは**2列×10穴・ピッチ2.54mm・列間20.32mm（0.8インチ）**の標準グリッド（寸法図PDFのベクトルデータから確定）
- スルーホールに出るのは44GPIO中20ピンのみ（1ピンおき）。使用可能GPIOは14本
- 電源の要: StampはUSB-C入力と外部5V入力（VBUSピン）を内部で理想ダイオードOR → **24V給電したままUSB-Cでの書き込みが安全**

## 3. ピン割り当て（最終版・検証済み）

検証: ESP32-P4データシートv0.5（ストラップ=G34〜38のみ／G0-15はフルIO MUX＋GPIOマトリクス／TWAI・UART1-4・LEDCは全ピン割当可）

### 左列（上=MIPI DSI端）
| # | ピン | 役割 | 備考 |
|---|---|---|---|
| L1 | G15 | UART3 RX ← BNO055 | |
| L2 | G12 | UART3 TX → BNO055 | 10kプルアップ |
| L3 | G10 | UART1 RX ← ST3215 Data直結 | 1kΩ直列（保護） |
| L4 | G8 | UART1 TX —1kΩ→ ST3215 Data | 合流回路 |
| L5 | G1 | PWM2 → ESC2 | 330Ω直列+100kプルダウン |
| L6 | G0 | PWM1 → ESC1 | 同上 |
| L7 | GND | — | |
| L8 | VBUS | 5V入力 | 内部ダイオードOR |
| L9 | G27 | UART2 RX ← Jetson | ファームでUSB機能無効化要 |
| L10 | G26 | UART2 TX → Jetson | 同上・10kプルアップ |

### 右列（上=MIPI DSI端）
| # | ピン | 役割 | 備考 |
|---|---|---|---|
| R1 | G23 | TWAI0 TX → CAN1 | |
| R2 | G21 | TWAI0 RX ← CAN1 | |
| R3 | G19 | TWAI1 TX → CAN2 | |
| R4 | G17 | TWAI1 RX ← CAN2 | |
| R5 | 3V3 | 3.3V出力 | Stamp内蔵DCDC 600mA |
| R6 | G37 ⚠ | 予備1（出力用） | ストラップ。10kプルアップ実装 |
| R7 | G38 ⚠ | 予備2（条件付き） | ストラップ。入力用途は要実機確認（IDF #17644のFTDI受信不能報告） |
| R8 | EN | リセット | タクトSW（モジュール側10kプルアップ済み） |
| R9 | G35 ⚠ | BOOTボタン | ストラップ専用化。GPIO流用しない |
| R10 | GND | — | |

### ピン配置の設計判断の記録
1. **G37/G38は必須信号から排除**（ブートモードストラップ＋G37はROMログ出力＋G38はRX不具合報告）。当初のバッファ+DIR方式をやめて合流回路にしたことでDIRピンが浮き、12信号が非ストラップ12ピンにちょうど収まった
2. ADC1表記ピン（G17-23）のCAN使用、LP系ピン（G0-15）のUART/PWM使用は、データシートのGPIOマトリクス仕様で全て正当（アナログ表記は追加機能の表示にすぎない）
3. G26/27はUSB1.1 PHY1のピン。リセット後デフォルトでUSB機能有効のため、**ファームで無効化してからUART2に使う**（デバッグ用USB-Serial-JTAGはPHY0=G24/25で衝突なし）

## 4. 回路設計

### 4-1. 電源
- 24V IN（XT30）→ **M78AR05-1**（24V→5V・1A・SIP3）→ StampのVBUSピン
  - 逆接保護・TVSは非搭載（2026-08-10ユーザー判断: XT30が物理的に逆挿し不可のため逆接は運用担保。サージはリスク受容。試作でM78AR05が突然死する事象が出たらTVS追加を第一容疑として再検討）
  - 入力C4=100µ/35V電解＋C5=22µ積セラ、出力C3=100µ/35V電解（スイッチングレギュレータの標準構成・省略不可。C3はC4と同一部品に統一 — 2026-08-10、47µのラジアル積セラが秋月に無いため電解化・データシート裏付け済み: 出力容量上限470µF、入力22µはメーカー推奨値と一致、内蔵入力フィルタあり）
- 入力: 100µF/35V電解 + 22µF MLCC、5V側: 47µF
- 7.4V系（2026-08-10最終形: **外部DCDCを基板直結の往復ペア接続に**）: **DCDCout1（XT30PW-F横出し）**で24V＋GNDを外部DCDC（24V→7.4V）へ送り、**DCDCin1（XT30PW-M横出し）**で7.4Vを受けて FirePower×2（XH2P）とSRV1（XH3P）へ分配。GNDはロジックと共通
  - 経緯: 当初は7.4Vを別配線でPH2P受けの予定 → 7.4VラインはESC×2＋サーボの合算電流が流れPHの2A定格では不足の恐れがありXT30化 → さらにDCDC往復を基板コネクタ化してハーネス簡略化
  - オス/メスの使い分け: 電源を出す側（DCDCout）=メス、受ける側（24Vin1・DCDCin）=オスで統一（RC作法）
  - **⚠️運用注意**: XT30が3口並ぶため「バッテリー（メス）をDCDCin（オス）へ誤挿し」が物理的に可能で、その場合+7.5Vレールに24Vが入りESC・サーボ破損リスク。ケーブル長を挿し違え不能に設計するかラベルで対策（ハーネス製作時の必須確認事項）
- 電源LED（5V）

### 4-2. CAN×2
- トランシーバ: **MCP2562FD-E/P**（DIP8）×2 — VCC=5V、**VIO=3.3V**（Stampロジックと整合）
  - ※当初SN65HVD234予定 → 秋月非取扱いのため変更。beta_mainのMCP2562と同族で実績あり。5Vレール駆動になり3.3V収支も改善
- TXD内部プルアップ内蔵（外付け10kはフットプリントのみ置いてDNP）
- STBYピンはGND固定（常時ノーマルモード）
- 終端120Ωはピンヘッダジャンパで切替、各ch PH3P×2（CANH/CANL/GND・デイジー通過）

### 4-3. ST3215バス（合流回路）

**サーボコネクタ運用（2026-08-10確定）**: SRV1はXH 3P（基板側B3B-XH-A）を維持し、**ST3215付属ケーブル（Molex 5264互換）の片端をXHハウジングに付け替えて運用**（案B採用。5267ウエハ直挿し案は秋月外調達となるため見送り）。
- ハーネス材料（秋月）: [XHハウジング3P XHP-3（112256・¥5）](https://akizukidenshi.com/catalog/g/g112256/)＋[XHコンタクト SXH-001T-P0.6 10個入（112264・¥30）](https://akizukidenshi.com/catalog/g/g112264/)
- **組立時作業**: サーボ増設のたびに圧着1回が発生する点に留意（予備ハーネスを事前に数本作っておくと競技現場で楽）
```
G8(TX) ─[1kΩ]─┬─ Data（10kプルアップ→3.3V）→ PH3P×2（7.4V/Data/GND・デイジー）
G10(RX) ─[1kΩ]─┘
```
- 前回装填機構メモの標準接続。送信エコーはファームで読み捨て
- リセット中はバスのプルアップでアイドルHigh維持。サーボは自発送信しないため安全

### 4-4. BNO055（2026-08-10更新: 極性解決・実装方式確定）
- **基板にDIP直載せ**（丸ピンソケット8P推奨。ケーブル接続案から変更 → IMUはこの基板上に載る）
- **極性確定（秋月説明書AE-BNO055-BO_20220413.pdfで裏付け）**: ピン3=SDA/T=モジュール送信→P4のRX（G15）、ピン4=SCL/R=モジュール受信→P4のTX（G12・10kプルアップ）。モジュール内部回路図でCOM0(SDA/TX)→ピン3、COM1(SCL/RX)→ピン4を確認済み
- ピン5(RESET)/6(INT)/8(VOUT=2.8V出力)は未接続
- レベル変換は電源電圧追従（説明書明記）→ **3.3V給電必須**（5V給電だと信号5VになりP4非トレラント違反）
- **組立時作業**: モジュールのUARTモード化（PS1=High。対応ジャンパはJ3/J5、出荷時はJ4/J5接続=I2Cモード。切替は実物説明書のジャンパ表で最終確認）

### 4-5. Jetson UART
- PH3P（TX/RX/GND）、両線220Ω直列。3.3V TTL直結

### 4-6. ESC PWM×2
- 2ピンコネクタ（Signal/GND）×2でGPIO直結（FireRight/FireLeft。2026-08-10実装時に3Pヘッダ中央NC案から変更 — BEC逆給電の可能性が構造的に消えるため改善）
- 330Ω直列・100kプルダウンは非搭載（2026-08-10ユーザー判断: 同構成3世代の実績＋ESCのアーミング防壁＋「ローラー停止中は割り出し禁止」のファーム制約による三重防護で省略。**ESCの銘柄・型番を変更する場合はこの判断を再考すること**）

### 4-7. HMI・予備・LED（2026-08-10 LED構成確定）
- BOOT/RESETタクトSW（高さ非対称: BOOT=低背4.3mm/RESET=標準7mm）
- G37/G38に10kプルアップ（R17/R18、ストラップ安定化＋将来のI2Cバス化に流用可）
- **LED×6（全て吸い込み向き・1kΩ、KiCad Device:LEDは1=K/2=A）**:
  - 電源LED1（+5V）
  - デバッグLED2/3（G37/G38、**吸い込み向き厳守** — 普通の向きだとリセット中にストラップ電圧がLED順方向電圧に張り付き、ダウンロードモード進入を阻害する恐れ。吸い込みならリセット中は消灯・ストラップ無影響）
  - アクティビティLED4/5/6（CAN1_TX・CAN2_TX・SRV_TXのロジック線。通信中のLowパルスで点灯。CANH/CANL・SRV_DATAには接続禁止 — 差動対称性/合流分圧を壊すため）
- G37のLEDはROMブートログで起動時に一瞬点滅する（再起動インジケータとして利用可）。デバッグLED直結後もG37/G38のI2C化とは共存可能（Low期間に+1.3mA負荷のみ）

## 5. 電源収支（計算済み・成立）

3.3Vレール（Stamp内蔵SY8089A・600mA）:
| 負荷 | 最悪 |
|---|---|
| P4本体（360MHz・全ペリフェラルclk・flash/PSRAMアクセス込み） | 〜250mA |
| BNO055 | 12.3mA |
| プル抵抗・LED等 | 〜8mA |
| **合計** | **〜270mA（余裕55%）** |

5Vレール（M78AR05-1・1A）:
| 負荷 | 平常 | 最悪 |
|---|---|---|
| Stampモジュール一式（内蔵DCDC効率90%込み） | 〜60mA | 〜200mA |
| CANトランシーバ×2（5V駆動・両chドミナント同時がピーク） | 〜30mA | 112mA |
| BNO055（Stamp 3.3V経由） | 〜9mA | 9mA |
| LED・プル抵抗類 | 〜5mA | 10mA |
| **合計** | **〜100mA** | **〜330mA（余裕3倍）** |

5Vから重負荷に給電しない設計（サーボ=7.4V別系統パススルー・ESC中央ピンNC・Jetson信号のみ）のため1Aで成立。**将来5V負荷（LEDテープ・USBホスト給電等）を追加する場合は再計算のこと**（C6 AddOn追加は+80mA程度で許容内）。24V入力換算 ≈ 60mA。入力範囲6.5〜32Vのため24V系の変動も許容内。

## 6. BOM（主要部品・調達先リンク）

| 部品 | 数 | 単価 | 調達先 |
|---|---|---|---|
| M5Stack Stamp-P4 | 1 | $12.95 | [M5Stack公式](https://shop.m5stack.com/products/m5stamp-esp32p4-module)（またはスイッチサイエンス取扱確認） |
| 超高効率DCDC 5V1A M78AR05-1 | 1 | ¥850 | [秋月 113536](https://akizukidenshi.com/catalog/g/g113536/) |
| CANトランシーバ MCP2562FD-E/P | 2 | ¥200 | [秋月 114383](https://akizukidenshi.com/catalog/g/g114383/) |
| XT30PW-M（オス横出し・24Vin1/DCDCin1） | 2 | ¥70 | [秋月 117949](https://akizukidenshi.com/catalog/g/g117949/) |
| XT30PW-F（メス横出し・DCDCout1） | 1 | ¥70 | [秋月 117948](https://akizukidenshi.com/catalog/g/g117948/) |
| XHコネクタ横 2P S2B-XH-A（FirePower×2）・3P S3B-XH-A（SRV1・PC1） | 2+2 | ¥15 | [112262](https://akizukidenshi.com/catalog/g/g112262/)/[112263](https://akizukidenshi.com/catalog/g/g112263/) |
| PHコネクタ横 2P S2B-PH-K-S（CAN×2・ESC信号×2） | 4 | ¥10 | [秋月 112633](https://akizukidenshi.com/catalog/g/g112633/) |
| 丸ピンICソケット 8P（BNO055直載せ用） | 1 | 〜¥30 | 秋月（DIP8用。2026-08-10直載せ決定に伴い追加） |
| AE-BNO055-BO | 1 | — | [秋月 116996](https://akizukidenshi.com/catalog/g/g116996/)（購入済み前提） |
| LED 3mm 5色構成: 緑OSG5TA3Z74A（電源・5V枠）／黄OSY5JA3Z74A（デバッグG37）／黄緑OSG8HA3Z74A（デバッグG38）／赤OSR5JA3Z74A×2（CANアクティビティ）／橙OSO5PA3133A-1MA 低消費750mcd@1mA（サーボアクティビティ・10個入・2026-08-10確定） | 6 | 〜¥180 | [緑111635](https://akizukidenshi.com/catalog/g/g111635/)・[黄111639](https://akizukidenshi.com/catalog/g/g111639/)・[黄緑111637](https://akizukidenshi.com/catalog/g/g111637/)・[赤111577](https://akizukidenshi.com/catalog/g/g111577/)・[橙116909](https://akizukidenshi.com/catalog/g/g116909/)。**※3.3V系はVf約2Vの赤・橙・黄・黄緑のみ可。純緑525nm/青/白はVf≈3.1Vで不可（5V系の電源LED枠のみ使用可）** |
| ピンソケット1×10（2本）・タクトSW×2・受動部品 | 一式 | 〜¥500 | 秋月（汎用在庫品） |
| 基板製造（JLCPCB 2層 5枚） | — | 〜¥1,500 | JLCPCB |

## 7. ファームウェア申し送り

1. **当面 `CONFIG_FREERTOS_UNICORE=y`** — v1.3シリコンのデュアルコアハング（[IDF #18899](https://github.com/espressif/esp-idf/issues/18899)・未解決）回避。制御負荷はシングルコアで十分
2. G26/27のUSB機能無効化（UART2使用前）
3. サーボバス送信エコーの読み捨て
4. ST3215は**ステップモード**制御＋カウント端数のBresenham丸め（詳細: docs/superpowers/specs/2026-07-26-bb-feeder-trigger-wheel-design.md 5.1節）
5. コンソールはUSB-Serial-JTAG（StampのUSB-C）。**UART0へのログ出力はsdkconfigで無効化必須** — G37がデバッグLEDと同居するため、放置するとログのたびLEDが薄光りする（2026-08-10 LED構成確定に伴い必須化）
6. TWAIループバック自己試験は使わない（v1.0シリコンでクラッシュ報告 [IDF #18660](https://github.com/espressif/esp-idf/issues/18660)）

## 8. ブリングアップ手順

1. 電源単体（Stamp非搭載）: 24V投入→5V確認、逆接テスト
2. Stamp搭載: USB-C書き込み確認（esptoolでシリコンrev確認も）→ 24V給電のままUSB併用確認
3. CAN疎通（既存基板と、両ch・終端条件を変えて）
4. ST3215単体4項目検証（装填機構メモの宿題: 多回転カウンタ・到達判定・ステップ所要時間・負方向ステップ）
5. BNO055極性・UART疎通
6. Jetson疎通、ESC PWM波形確認

## 9. 未決事項

- 基板外形・取付穴位置（ロボット搭載スペース確定後。約60×50mm想定・M3×4）
- ~~BOOT/RESETボタンの誤押下対策配置~~ → **確定（2026-08-10）: ボタン高さを非対称にする。SW2(BOOT)=高さ4.3mm級の低背タクトSW、SW1(RESET)=高さ7mm級の標準タクトSW**（同一6mm角フットプリントの高さ違い品）。危険な組合せ（BOOT押下+リセット事象=ダウンロードモード起動）のBOOT側だけ物理的に押されにくくする。配置はケーブルルートから遠ざけたうえで2つ隣接（片手操作性）。単独押下はBOOT=無害/EN=数秒停止で、出力系のプル抵抗によりリセット中のグリッチは構造的に発生しない
- ~~BNO055モジュールのUARTピン極性~~ → 解決済み（2026-08-10、秋月説明書で裏付け。4-4節参照）
- Stamp-AddOn C6とソケット脚の物理干渉（無線化する場合のみ）
- スイッチサイエンスでのStamp-P4取扱有無（送料・納期次第でM5Stack直販）

## 10. 主な検証ソース

- ESP32-P4データシートv0.5（ピン仕様・ストラップ・消費電流）
- M5Stack Stamp-P4公式（[Docs](https://docs.m5stack.com/en/core/Stamp-P4)・回路図PDF・寸法図PDF・[KiCadフットプリント](https://github.com/m5stack/M5_Hardware/blob/master/KiCad/Footprints/M5Stack.pretty/Stamp-P4.kicad_mod)）
- ESP-IDF issues: [#18899](https://github.com/espressif/esp-idf/issues/18899)（デュアルコアハング）・[#18344](https://github.com/espressif/esp-idf/issues/18344)（USB書き込み）・[#17644](https://github.com/espressif/esp-idf/issues/17644)（G38 RX）・[#18660](https://github.com/espressif/esp-idf/issues/18660)（TWAIループバック）
- esptool [Boot Mode Selection (P4)](https://docs.espressif.com/projects/esptool/en/latest/esp32p4/advanced-topics/boot-mode-selection.html)
