# gamma_turret_pcb

## 役割

砲塔側の制御基板。M5Stack Stamp-P4（ESP32-P4）を載せ、CAN 2 系統、射出 ESC の PWM 2ch、装填サーボの半二重 UART、上位コントローラとの UART、IMU を 1 枚にまとめる。

## 主要部品

| 部品 | 型番 | 用途 |
|---|---|---|
| マイコンモジュール | M5Stack Stamp-P4（ESP32-P4） | 制御。ピンソケットで差し替え可 |
| CAN トランシーバ | MCP2562-E/P × 2 | CAN1（砲塔・ジンバル系）、CAN2（車体系） |
| レギュレータ | M78AR05-1 | 24V → 5V（Stamp-P4 用） |
| IMU | BNO055 | 姿勢。UART 接続 |
| 24V コネクタ | AMASS XT30PW-M × 2、XT30PW-F × 1 | 24V 入力、外部 DCDC への往復 |
| CAN / ESC 信号 | JST PH S2B-PH-K × 4 | CAN1、CAN2、射出 ESC 左右 |
| ESC 電源 | JST XH S2B-XH-A × 2 | 射出 ESC 左右 |
| サーボ / 上位 UART | JST XH S3B-XH-A × 2 | ST3215（半二重）、上位コントローラ |
| CAN 終端抵抗 | 120 Ω × 2 | CAN1 / CAN2 |
| 表示・操作 | LED × 6、タクト SW × 2 | 電源 / デバッグ / アクティビティ、BOOT / RESET |

回路図は [gamma_turret.pdf](./gamma_turret.pdf)、部品表は [bom.csv](./bom.csv)、3D は [gamma_turret.step](./gamma_turret.step)。
設計時の検討は [docs/](./docs/) に、旧 STM32 版の設計は [old_design/](./old_design/) に残している。

## ライセンス

© Rabbits Robotics、[CERN-OHL-P-2.0](./LICENSE)。
