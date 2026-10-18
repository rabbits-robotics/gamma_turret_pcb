# ESP32-P4キャリア基板 KiCad回路図 実装計画

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 設計メモ（2026-08-10-esp32-p4-carrier-design.md）の回路を `gamma_turret.kicad_sch` としてスクリプト生成し、ERC 0エラー＋期待ネットリスト完全一致まで検証する。

**Architecture:** 回路図はPythonスクリプト `gen_schematic.py` でS式を直接生成（全シンボルは矩形＋サイドピンの自製ミニマルシンボルとして埋め込み、外部ライブラリ依存ゼロ）。検証は `kicad-cli sch erc`（0エラー）と `check_nets.py`（期待ネット表 `expected_nets.csv` とネットリストXMLの照合）の2段構え。見た目の整形は生成後にユーザーがeeschemaで実施可能（再生成すると上書きされる点に注意。整形はネット検証が全部通った後に）。

**Tech Stack:** KiCad 8.0（kicad-cli）、Python 3（標準ライブラリのみ）

**前提確認済み:** kicad-cli 8.0動作、既存sch形式=20231120（KiCad 8）、旧設計はコピーの名残（置き換え承認済み・旧ガーバーはgbr_*.zipに保存済み）

---

## ファイル構成

| ファイル | 役割 |
|---|---|
| `gen_schematic.py` | 回路図生成スクリプト（シンボル工場＋ブロック定義＋S式出力） |
| `expected_nets.csv` | 期待ネット表（net,ref,pin）— 設計メモから起こした正解データ |
| `check_nets.py` | ネットリストXMLと期待表の照合スクリプト |
| `gamma_turret.kicad_sch` | 生成物（上書き） |
| `old_design/` | 旧回路図・基板の退避先 |

---

### Task 1: 準備とテストハーネス（期待ネット表を先に書く）

**Files:**
- Create: `old_design/`（退避フォルダ）
- Create: `expected_nets.csv`
- Create: `check_nets.py`

- [ ] **Step 1: 旧設計の退避**

```powershell
cd C:\Users\kota0\Desktop\Rabbits\board_ws\gamma_turret
New-Item -ItemType Directory -Force old_design
Copy-Item gamma_turret.kicad_sch,gamma_turret.kicad_pcb,gamma_turret.net old_design\
```

- [ ] **Step 2: git初期化（推奨・任意）**

```powershell
git init; git add -A; git commit -m "chore: gamma_turret旧設計の保存とESP32-P4キャリア設計メモ"
```
※gitを使わない場合は各タスク末尾のcommitをスキップし、`*-backups`フォルダ運用のままでよい。

- [ ] **Step 3: 期待ネット表 `expected_nets.csv` を作成（これがテストの正解データ）**

```csv
net,ref,pin
+BATT,J1,1
+BATT,Q1,3
GND,J1,2
GND,R1,2
GND,D1,2
GND,C1,2
GND,C2,2
GND,U2,2
GND,C3,2
GND,LED1,2
GND,C4,2
GND,C5,2
GND,U1,7
GND,U1,20
GND,SW1,2
GND,SW2,2
GND,U3,2
GND,U3,8
GND,U4,2
GND,U4,8
GND,J2,2
GND,J3,3
GND,J4,3
GND,J5,3
GND,J6,3
GND,J7,3
GND,J8,3
GND,J9,2
GND,J10,3
GND,J11,3
GND,J12,3
GND,J13,4
GND,R15,2
GND,R16,2
GND,#FLG02,1
+24V,Q1,2
+24V,D2,1
+24V,D1,1
+24V,C1,1
+24V,C2,1
+24V,U2,1
+24V,#FLG01,1
GATE,Q1,1
GATE,R1,1
GATE,D2,2
+5V,U2,3
+5V,C3,1
+5V,R12,1
+5V,U1,8
+5V,U3,3
+5V,U4,3
+5V,C4,1
+5V,C5,1
+5V,#FLG03,1
LED_K,R12,2
LED_K,LED1,1
+3V3,U1,15
+3V3,U3,5
+3V3,U4,5
+3V3,R8,1
+3V3,R9,1
+3V3,R10,1
+3V3,R11,1
+3V3,J9,1
+3V3,J13,3
+3V3,#FLG04,1
+7V4,J2,1
+7V4,J7,1
+7V4,J8,1
+7V4,#FLG05,1
CAN1_TX,U1,11
CAN1_TX,U3,1
CAN1_RX,U1,12
CAN1_RX,U3,4
CAN1_H,U3,7
CAN1_H,J3,1
CAN1_H,J4,1
CAN1_H,JP1,1
TERM1,JP1,2
TERM1,R13,1
CAN1_L,U3,6
CAN1_L,J3,2
CAN1_L,J4,2
CAN1_L,R13,2
CAN2_TX,U1,13
CAN2_TX,U4,1
CAN2_RX,U1,14
CAN2_RX,U4,4
CAN2_H,U4,7
CAN2_H,J5,1
CAN2_H,J6,1
CAN2_H,JP2,1
TERM2,JP2,2
TERM2,R14,1
CAN2_L,U4,6
CAN2_L,J5,2
CAN2_L,J6,2
CAN2_L,R14,2
SRV_TX,U1,4
SRV_TX,R2,1
SRV_RX,U1,3
SRV_RX,R3,1
SRV_DATA,R2,2
SRV_DATA,R3,2
SRV_DATA,R8,2
SRV_DATA,J7,2
SRV_DATA,J8,2
JET_TX,U1,10
JET_TX,R4,1
JET_TX,R10,2
JET_TXO,R4,2
JET_TXO,J10,1
JET_RX,U1,9
JET_RX,R5,2
JET_RXI,R5,1
JET_RXI,J10,2
IMU_TX,U1,2
IMU_TX,R11,2
IMU_TX,J9,3
IMU_RX,U1,1
IMU_RX,J9,4
PWM1,U1,6
PWM1,R6,1
PWM1_OUT,R6,2
PWM1_OUT,R15,1
PWM1_OUT,J11,1
PWM2,U1,5
PWM2,R7,1
PWM2_OUT,R7,2
PWM2_OUT,R16,1
PWM2_OUT,J12,1
EN_N,U1,18
EN_N,SW1,1
BOOT_N,U1,19
BOOT_N,SW2,1
SPARE1,U1,16
SPARE1,R9,2
SPARE1,J13,1
SPARE2,U1,17
SPARE2,J13,2
```

- [ ] **Step 4: `check_nets.py` を作成**

```python
# check_nets.py — kicad-cliのXMLネットリストとexpected_nets.csvを照合
# 使い方: python check_nets.py [--subset]  (--subset: CSV側に載る接続だけ検査。ブロック追加途中用)
import csv, sys, subprocess, xml.etree.ElementTree as ET
from collections import defaultdict

SCH = "gamma_turret.kicad_sch"
subprocess.run(["kicad-cli", "sch", "export", "netlist",
                "--format", "kicadxml", "-o", "netlist.xml", SCH], check=True)

tree = ET.parse("netlist.xml")
actual = defaultdict(set)   # net名 -> {(ref,pin)}
node_of = {}                # (ref,pin) -> net名
for net in tree.find("nets"):
    name = net.get("name")
    for node in net.findall("node"):
        key = (node.get("ref"), node.get("pin"))
        actual[name].add(key)
        node_of[key] = name

expected = defaultdict(set)
with open("expected_nets.csv", newline="", encoding="utf-8") as f:
    for row in csv.DictReader(f):
        expected[row["net"]].add((row["ref"], row["pin"]))

subset = "--subset" in sys.argv
ng = 0
# 1) 期待した接続が同一ネットにまとまっているか（ref,pin単位で照合）
for net, pins in expected.items():
    names = {node_of.get(p) for p in pins}
    missing = [p for p in pins if p not in node_of]
    if missing:
        print(f"NG [{net}] 未配置ピン: {missing}"); ng += 1; continue
    if len(names) != 1:
        print(f"NG [{net}] 接続が分裂: {names}"); ng += 1; continue
    got = names.pop()
    if got != net:
        print(f"NG [{net}] ネット名不一致: {got}"); ng += 1; continue
    if not subset and actual[got] - pins:
        print(f"NG [{net}] 余計な接続: {actual[got] - pins}"); ng += 1
print("OK: all nets match" if ng == 0 else f"FAILED: {ng} problems")
sys.exit(1 if ng else 0)
```

- [ ] **Step 5: 現時点で実行し「未配置ピン」で失敗することを確認（テストが機能している証拠）**

```powershell
python check_nets.py
```
期待: `NG [...] 未配置ピン:` が多数出て exit 1（旧回路図にはU1等が無いため）

- [ ] **Step 6: Commit**

```powershell
git add expected_nets.csv check_nets.py; git commit -m "test: 期待ネット表と照合スクリプト"
```

---

### Task 2: 回路図生成スクリプトの骨格（シンボル工場＋空シート）

**Files:**
- Create: `gen_schematic.py`

- [ ] **Step 1: `gen_schematic.py` を作成（完全コード）**

```python
# gen_schematic.py — gamma_turret.kicad_sch を生成する（KiCad 8 / 20231120形式）
# 方針: 全シンボルは「矩形＋サイドピン」の自製ミニマル定義を埋め込み。
#       各ピンの先端にグローバルラベルを直置きして結線（ワイヤ描画なし）。
#       NCピンは no_connect マーカーを置く。
import uuid

U = lambda: str(uuid.uuid4())
ROOT_UUID = "aaaaaaaa-0000-4000-8000-000000000001"  # 再生成でも安定させる固定値

# ---------- シンボル定義（ライブラリ部） ----------
def sym_def(name, left, right, w=12.7, pin_len=2.54):
    """left/right: [(番号, 名前, 電気タイプ)] 上から順。戻り値: lib_symbols用S式"""
    n = max(len(left), len(right))
    h = max((n + 1) * 2.54, 5.08)
    top = h / 2
    body = (f'  (symbol "{name}" (pin_names (offset 0.762)) (exclude_from_sim no) '
            f'(in_bom yes) (on_board yes)\n'
            f'   (property "Reference" "U" (at 0 {top + 2.54:.2f} 0) '
            f'(effects (font (size 1.27 1.27))))\n'
            f'   (property "Value" "{name}" (at 0 {-top - 2.54:.2f} 0) '
            f'(effects (font (size 1.27 1.27))))\n'
            f'   (property "Footprint" "" (at 0 0 0) (effects (font (size 1.27 1.27)) (hide yes)))\n'
            f'   (property "Datasheet" "~" (at 0 0 0) (effects (font (size 1.27 1.27)) (hide yes)))\n'
            f'   (symbol "{name}_0_1"\n'
            f'    (rectangle (start {-w/2:.2f} {top:.2f}) (end {w/2:.2f} {-top:.2f}) '
            f'(stroke (width 0.254) (type default)) (fill (type background)))\n')
    pins = []
    for i, (num, pname, etype) in enumerate(left):
        y = top - (i + 1) * 2.54
        pins.append(_pin(num, pname, etype, -w/2 - pin_len, y, 0, pin_len))
    for i, (num, pname, etype) in enumerate(right):
        y = top - (i + 1) * 2.54
        pins.append(_pin(num, pname, etype, w/2 + pin_len, y, 180, pin_len))
    return body + "".join(pins) + "   )\n  )\n"

def _pin(num, name, etype, x, y, angle, length):
    return (f'    (pin {etype} line (at {x:.2f} {y:.2f} {angle}) (length {length}) '
            f'(name "{name}" (effects (font (size 1.02 1.02)))) '
            f'(number "{num}" (effects (font (size 1.02 1.02)))))\n')

def pin_positions(left_n, right_n, w=12.7, pin_len=2.54):
    """シンボル原点基準のピン先端座標 {番号: (dx, dy, 向きdeg)} を返す（sym_defと同じ計算）"""
    n = max(left_n, right_n)
    top = max((n + 1) * 2.54, 5.08) / 2
    pos = {}
    for i in range(left_n):
        pos[str(i + 1)] = (-w/2 - pin_len, top - (i + 1) * 2.54, 0)
    for i in range(right_n):
        pos[str(left_n + i + 1)] = (w/2 + pin_len, top - (i + 1) * 2.54, 180)
    return pos

# ---------- シンボルカタログ（名前: (左ピン, 右ピン, 幅)） ----------
P = "passive"; I = "input"; O = "output"; B = "bidirectional"
PWI = "power_in"; PWO = "power_out"
CATALOG = {
    # Stamp-P4 ソケット（設計メモ3節の最終ピン配置）
    "STAMP_P4": (
        [("1","G15_IMU_RX",B),("2","G12_IMU_TX",B),("3","G10_SRV_RX",B),
         ("4","G8_SRV_TX",B),("5","G1_PWM2",B),("6","G0_PWM1",B),
         ("7","GND",PWI),("8","VBUS_5V",PWI),("9","G27_JET_RX",B),("10","G26_JET_TX",B)],
        [("11","G23_CAN1TX",B),("12","G21_CAN1RX",B),("13","G19_CAN2TX",B),
         ("14","G17_CAN2RX",B),("15","3V3_OUT",PWO),("16","G37_SPARE1",B),
         ("17","G38_SPARE2",B),("18","EN",I),("19","G35_BOOT",I),("20","GND",PWI)], 20.32),
    "MCP2562FD": (
        [("1","TXD",I),("2","VSS",PWI),("3","VDD",PWI),("4","RXD",O)],
        [("5","VIO",PWI),("6","CANL",B),("7","CANH",B),("8","STBY",I)], 12.7),
    "M78AR05":  ([("1","VIN",PWI),("2","GND",PWI)], [("3","VOUT",PWO)], 12.7),
    "PMOS_GSD": ([("1","G",I),("2","S",P)], [("3","D",P)], 7.62),
    "R":        ([("1","~",P)], [("2","~",P)], 5.08),
    "C":        ([("1","~",P)], [("2","~",P)], 5.08),
    "CP":       ([("1","+",P)], [("2","-",P)], 5.08),
    "LED":      ([("1","A",P)], [("2","K",P)], 5.08),
    "D_TVS":    ([("1","A1",P)], [("2","A2",P)], 5.08),
    "D_ZEN":    ([("1","K",P)], [("2","A",P)], 5.08),
    "SW":       ([("1","1",P)], [("2","2",P)], 5.08),
    "CONN2":    ([("1","1",P),("2","2",P)], [], 5.08),
    "CONN3":    ([("1","1",P),("2","2",P),("3","3",P)], [], 5.08),
    "CONN4":    ([("1","1",P),("2","2",P),("3","3",P),("4","4",P)], [], 5.08),
    "PWR_FLAG": ([("1","pwr",PWO)], [], 5.08),
}

# ---------- 配置ブロック（Task 3以降でここに追記していく） ----------
# 形式: (ref, シンボル名, Value表示, (X, Y), {ピン番号: ネット名 or "NC"})
BLOCKS = [
]

# ---------- S式の組み立て ----------
def emit():
    used = sorted({b[1] for b in BLOCKS})
    lib = "".join(
        sym_def(f"gen:{s}", CATALOG[s][0], CATALOG[s][1], CATALOG[s][2]).replace(
            f'"{f"gen:{s}"}_0_1"', f'"{s}_0_1"').replace(f'(symbol "gen:{s}"', f'(symbol "gen:{s}"', 1)
        for s in used)
    items = []
    for ref, sname, value, (X, Y), nets in BLOCKS:
        Lp, Rp, w = CATALOG[sname]
        pos = pin_positions(len(Lp), len(Rp), w)
        pins_sexpr = "".join(f'  (pin "{n}" (uuid "{U()}"))\n' for n, _, _ in Lp + Rp)
        items.append(
            f'(symbol (lib_id "gen:{sname}") (at {X} {Y} 0) (unit 1)\n'
            f' (exclude_from_sim no) (in_bom yes) (on_board yes) (dnp no) (uuid "{U()}")\n'
            f' (property "Reference" "{ref}" (at {X} {Y - 10} 0) (effects (font (size 1.27 1.27))))\n'
            f' (property "Value" "{value}" (at {X} {Y + 10} 0) (effects (font (size 1.27 1.27))))\n'
            f' (property "Footprint" "" (at {X} {Y} 0) (effects (font (size 1.27 1.27)) (hide yes)))\n'
            f' (property "Datasheet" "~" (at {X} {Y} 0) (effects (font (size 1.27 1.27)) (hide yes)))\n'
            + pins_sexpr +
            f' (instances (project "gamma_turret" (path "/{ROOT_UUID}" (reference "{ref}") (unit 1))))\n)')
        for num, net in nets.items():
            dx, dy, ang = pos[num]
            x, y = X + dx, Y - dy   # 回路図座標はY下向き
            if net == "NC":
                items.append(f'(no_connect (at {x:.2f} {y:.2f}) (uuid "{U()}"))')
            else:
                rot = {0: 180, 180: 0}[ang]
                items.append(
                    f'(global_label "{net}" (shape passive) (at {x:.2f} {y:.2f} {rot}) '
                    f'(effects (font (size 1.27 1.27)) (justify {"right" if rot == 180 else "left"})) '
                    f'(uuid "{U()}"))')
    return (f'(kicad_sch (version 20231120) (generator "gen_schematic")\n'
            f' (uuid "{ROOT_UUID}") (paper "A2")\n'
            f' (lib_symbols\n{lib} )\n' + "\n".join(items) +
            f'\n (sheet_instances (path "/" (page "1")))\n)\n')

if __name__ == "__main__":
    with open("gamma_turret.kicad_sch", "w", encoding="utf-8") as f:
        f.write(emit())
    print(f"generated: {len(BLOCKS)} symbols")
```

- [ ] **Step 2: 空のまま生成し、KiCadが読める形式であることを確認**

```powershell
python gen_schematic.py
kicad-cli sch erc --exit-code-violations -o erc.rpt gamma_turret.kicad_sch; type erc.rpt
```
期待: 生成成功・ERCがパースエラーなく完走（violations 0）

- [ ] **Step 3: Commit**

```powershell
git add gen_schematic.py; git commit -m "feat: 回路図生成スクリプト骨格"
```

---

### Task 3: 電源ブロック（24V入力・逆接保護・5V生成）

**Files:**
- Modify: `gen_schematic.py`（BLOCKSに追記）

- [ ] **Step 1: BLOCKSに電源ブロックを追加**

```python
    # --- 電源（設計メモ4-1） ---
    ("J1",  "CONN2", "XT30PW-M_24V_IN", (40, 40),  {"1": "+BATT", "2": "GND"}),
    ("Q1",  "PMOS_GSD", "Pch_逆接保護",  (60, 40),  {"1": "GATE", "2": "+24V", "3": "+BATT"}),
    ("R1",  "R",   "100k",             (60, 60),  {"1": "GATE", "2": "GND"}),
    ("D2",  "D_ZEN", "12V_Zener",      (80, 60),  {"1": "+24V", "2": "GATE"}),
    ("D1",  "D_TVS", "SMBJ33A",        (100, 60), {"1": "+24V", "2": "GND"}),
    ("C1",  "CP",  "100u/35V",         (120, 60), {"1": "+24V", "2": "GND"}),
    ("C2",  "C",   "22u",              (140, 60), {"1": "+24V", "2": "GND"}),
    ("U2",  "M78AR05", "M78AR05-1",    (100, 40), {"1": "+24V", "2": "GND", "3": "+5V"}),
    ("C3",  "C",   "47u",              (160, 60), {"1": "+5V", "2": "GND"}),
    ("R12", "R",   "1k",               (180, 60), {"1": "+5V", "2": "LED_K"}),
    ("LED1","LED", "PWR_LED",          (200, 60), {"1": "LED_K", "2": "GND"}),
    ("#FLG01", "PWR_FLAG", "PWR_FLAG", (40, 80),  {"1": "+24V"}),
    ("#FLG02", "PWR_FLAG", "PWR_FLAG", (50, 80),  {"1": "GND"}),
    ("#FLG03", "PWR_FLAG", "PWR_FLAG", (60, 80),  {"1": "+5V"}),
```

- [ ] **Step 2: 生成→ERC→部分ネット照合**

```powershell
python gen_schematic.py
kicad-cli sch erc --exit-code-violations -o erc.rpt gamma_turret.kicad_sch
python check_nets.py --subset
```
期待: ERC 0 violations。check_netsは電源系ネット（+BATT/GND/+24V/GATE/+5V/LED_K）がOKになり、未配置はU1/CAN系など残り分のみ

- [ ] **Step 3: Commit**

```powershell
git add gen_schematic.py; git commit -m "feat: 電源ブロック（XT30・逆接PMOS・M78AR05）"
```

---

### Task 4: Stampソケット・BOOT/RESET・予備ヘッダ

**Files:**
- Modify: `gen_schematic.py`

- [ ] **Step 1: BLOCKSに追加**

```python
    # --- Stamp-P4ソケット＋HMI＋予備（設計メモ3節・4-7） ---
    ("U1",  "STAMP_P4", "M5Stack_Stamp-P4", (140, 130), {
        "1": "IMU_RX", "2": "IMU_TX", "3": "SRV_RX", "4": "SRV_TX",
        "5": "PWM2", "6": "PWM1", "7": "GND", "8": "+5V",
        "9": "JET_RX", "10": "JET_TX",
        "11": "CAN1_TX", "12": "CAN1_RX", "13": "CAN2_TX", "14": "CAN2_RX",
        "15": "+3V3", "16": "SPARE1", "17": "SPARE2",
        "18": "EN_N", "19": "BOOT_N", "20": "GND"}),
    ("SW1", "SW", "RESET",            (200, 100), {"1": "EN_N", "2": "GND"}),
    ("SW2", "SW", "BOOT",             (200, 115), {"1": "BOOT_N", "2": "GND"}),
    ("R9",  "R", "10k",               (200, 130), {"1": "+3V3", "2": "SPARE1"}),
    ("J13", "CONN4", "SPARE_HDR",     (220, 130), {"1": "SPARE1", "2": "SPARE2", "3": "+3V3", "4": "GND"}),
    ("#FLG04", "PWR_FLAG", "PWR_FLAG", (70, 80),  {"1": "+3V3"}),
```

- [ ] **Step 2: 生成→ERC→部分照合（Task 3と同じコマンド）**

```powershell
python gen_schematic.py; kicad-cli sch erc --exit-code-violations -o erc.rpt gamma_turret.kicad_sch; python check_nets.py --subset
```
期待: ERC 0。EN_N/BOOT_N/SPARE1/SPARE2系がOKに変わる

- [ ] **Step 3: Commit**

```powershell
git add gen_schematic.py; git commit -m "feat: Stampソケット・BOOT/RESET・予備ヘッダ"
```

---

### Task 5: CAN×2

**Files:**
- Modify: `gen_schematic.py`

- [ ] **Step 1: BLOCKSに追加**

```python
    # --- CAN×2（設計メモ4-2） MCP2562FD: VDD=5V, VIO=3.3V, STBY=GND ---
    ("U3",  "MCP2562FD", "MCP2562FD-E/P", (260, 60), {
        "1": "CAN1_TX", "2": "GND", "3": "+5V", "4": "CAN1_RX",
        "5": "+3V3", "6": "CAN1_L", "7": "CAN1_H", "8": "GND"}),
    ("C4",  "C", "100n",              (260, 85),  {"1": "+5V", "2": "GND"}),
    ("J3",  "CONN3", "PH3_CAN1_A",    (300, 50),  {"1": "CAN1_H", "2": "CAN1_L", "3": "GND"}),
    ("J4",  "CONN3", "PH3_CAN1_B",    (300, 70),  {"1": "CAN1_H", "2": "CAN1_L", "3": "GND"}),
    ("JP1", "CONN2", "TERM_JP1",      (320, 50),  {"1": "CAN1_H", "2": "TERM1"}),
    ("R13", "R", "120",               (320, 70),  {"1": "TERM1", "2": "CAN1_L"}),
    ("U4",  "MCP2562FD", "MCP2562FD-E/P", (260, 130), {
        "1": "CAN2_TX", "2": "GND", "3": "+5V", "4": "CAN2_RX",
        "5": "+3V3", "6": "CAN2_L", "7": "CAN2_H", "8": "GND"}),
    ("C5",  "C", "100n",              (260, 155), {"1": "+5V", "2": "GND"}),
    ("J5",  "CONN3", "PH3_CAN2_A",    (300, 120), {"1": "CAN2_H", "2": "CAN2_L", "3": "GND"}),
    ("J6",  "CONN3", "PH3_CAN2_B",    (300, 140), {"1": "CAN2_H", "2": "CAN2_L", "3": "GND"}),
    ("JP2", "CONN2", "TERM_JP2",      (320, 120), {"1": "CAN2_H", "2": "TERM2"}),
    ("R14", "R", "120",               (320, 140), {"1": "TERM2", "2": "CAN2_L"}),
```

- [ ] **Step 2: 生成→ERC→部分照合（同コマンド）** 期待: CAN系ネット全OK

- [ ] **Step 3: Commit** `git commit -am "feat: CAN 2系統（MCP2562FD・終端ジャンパ）"`

---

### Task 6: サーボバス・BNO055・Jetson・ESC

**Files:**
- Modify: `gen_schematic.py`

- [ ] **Step 1: BLOCKSに追加**

```python
    # --- ST3215合流回路（設計メモ4-3） ---
    ("R2",  "R", "1k",                (60, 180),  {"1": "SRV_TX", "2": "SRV_DATA"}),
    ("R3",  "R", "1k",                (60, 195),  {"1": "SRV_RX", "2": "SRV_DATA"}),
    ("R8",  "R", "10k",               (60, 210),  {"1": "+3V3", "2": "SRV_DATA"}),
    ("J2",  "CONN2", "PH2_7V4_IN",    (90, 165),  {"1": "+7V4", "2": "GND"}),
    ("J7",  "CONN3", "PH3_SERVO_A",   (90, 180),  {"1": "+7V4", "2": "SRV_DATA", "3": "GND"}),
    ("J8",  "CONN3", "PH3_SERVO_B",   (90, 195),  {"1": "+7V4", "2": "SRV_DATA", "3": "GND"}),
    ("#FLG05", "PWR_FLAG", "PWR_FLAG", (80, 80),  {"1": "+7V4"}),
    # --- BNO055（4-4）: J9 = 3V3/GND/TX/RX ---
    ("R11", "R", "10k",               (130, 180), {"1": "+3V3", "2": "IMU_TX"}),
    ("J9",  "CONN4", "PH4_BNO055",    (130, 195), {"1": "+3V3", "2": "GND", "3": "IMU_TX", "4": "IMU_RX"}),
    # --- Jetson UART（4-5）: 220Ω直列 ---
    ("R4",  "R", "220",               (170, 180), {"1": "JET_TX", "2": "JET_TXO"}),
    ("R5",  "R", "220",               (170, 195), {"1": "JET_RXI", "2": "JET_RX"}),
    ("R10", "R", "10k",               (170, 210), {"1": "+3V3", "2": "JET_TX"}),
    ("J10", "CONN3", "PH3_JETSON",    (200, 188), {"1": "JET_TXO", "2": "JET_RXI", "3": "GND"}),
    # --- ESC PWM×2（4-6）: 中央ピンNC ---
    ("R6",  "R", "330",               (240, 180), {"1": "PWM1", "2": "PWM1_OUT"}),
    ("R15", "R", "100k",              (240, 195), {"1": "PWM1_OUT", "2": "GND"}),
    ("J11", "CONN3", "ESC1_HDR",      (270, 180), {"1": "PWM1_OUT", "2": "NC", "3": "GND"}),
    ("R7",  "R", "330",               (240, 215), {"1": "PWM2", "2": "PWM2_OUT"}),
    ("R16", "R", "100k",              (240, 230), {"1": "PWM2_OUT", "2": "GND"}),
    ("J12", "CONN3", "ESC2_HDR",      (270, 215), {"1": "PWM2_OUT", "2": "NC", "3": "GND"}),
```

- [ ] **Step 2: 生成→ERC→部分照合（同コマンド）** 期待: 残りネット全OK

- [ ] **Step 3: Commit** `git commit -am "feat: サーボバス・BNO055・Jetson・ESC"`

---

### Task 7: 全体検証とレビュー出力

- [ ] **Step 1: 完全照合（--subsetなし＝余計な接続もゼロであること）**

```powershell
python check_nets.py
```
期待: `OK: all nets match` exit 0

- [ ] **Step 2: ERC最終確認**

```powershell
kicad-cli sch erc --exit-code-violations -o erc.rpt gamma_turret.kicad_sch; type erc.rpt
```
期待: 0 violations（警告が出た場合は内容を記録し、実害の有無を判断してから対処）

- [ ] **Step 3: レビュー用PDF出力してユーザーに提示**

```powershell
kicad-cli sch export pdf -o gamma_turret_sch_review.pdf gamma_turret.kicad_sch
```

- [ ] **Step 4: ユーザーレビュー（チェックポイント）**

eeschemaで開いて確認してもらう。指摘があればBLOCKSを修正→再生成→再検証。
**注意: GUIでの手動整形は、ネット検証完了後かつ再生成が不要になってから行うこと**（再生成で上書きされる）

- [ ] **Step 5: Commit** `git commit -am "feat: 回路図完成（ERC 0・ネット照合一致）"`

---

### Task 8: フットプリント割当（レイアウト準備）

- [ ] **Step 1: BLOCKSの各エントリにフットプリントを追加**

gen_schematic.pyのemit()でFootprintプロパティに以下を設定（BLOCKS各行に6番目の要素として追加し、emit側で `(property "Footprint" ...)` に反映）:

| 部品 | フットプリント |
|---|---|
| U1 | 自作 `gamma_p4:Stamp_P4_Socket_2x10_P2.54_RS20.32`（Task後続のレイアウト計画で作成。2列×10・列間20.32mm） |
| U2 | `Package_TO_SOT_THT:TO-220-3_Vertical`相当のSIP3（M78AR05-1は2.54ピッチSIP3） |
| U3/U4 | `Package_DIP:DIP-8_W7.62mm` |
| J1 | `Connector_AMASS:AMASS_XT30PW-M_1x02_P2.50mm_Horizontal` |
| J2〜J10 | `Connector_JST:JST_PH_B{2,3,4}B-PH-K_1x0{2,3,4}_P2.00mm_Vertical` |
| J11-J13, JP1/2 | `Connector_PinHeader_2.54mm:PinHeader_1x0{2,3,4}_P2.54mm_Vertical` |
| R/C/LED | 手はんだ前提: `R_Axial_DIN0207`・`C_Disc_D5.0mm`・`CP_Radial_D6.3mm`・`LED_D3.0mm`、D1 `D_SMB`、D2 `D_DO-35`、Q1 `TO-252-2`、SW `SW_PUSH_6mm` |

- [ ] **Step 2: 再生成→全検証（Task 7のStep 1-2を再実行）→ Commit**

```powershell
python gen_schematic.py; kicad-cli sch erc --exit-code-violations -o erc.rpt gamma_turret.kicad_sch; python check_nets.py
git commit -am "feat: フットプリント割当"
```

- [ ] **Step 3: 次フェーズへ**

レイアウト（基板外形・Stampソケット自作フットプリント・配置・配線・ガーバー）は別計画として作成する。

---

## セルフレビュー済みの注意点

1. **MCP2562FDのピン配列**はMCP2562-E/Pと同一（1:TXD 2:VSS 3:VDD 4:RXD 5:VIO 6:CANL 7:CANH 8:STBY）を前提。実装時にデータシートで再確認すること
2. **M78AR05-1のピン配列**（1:IN 2:GND 3:OUT、7805互換）も秋月データシートで再確認
3. Q1（PMOS）は D=バッテリ側/S=負荷側/G=100k→GND＋12Vツェナー（G-S間）。ERCでpassiveピン扱いのため向き間違いは検出されない — レビューで目視確認する
4. 生成スクリプトのS式はKiCad 8の必須フィールドを最小構成で出している。ERCがパース警告を出したらerc.rptの行を見て`emit()`を修正（Task 2 Step 2で早期に洗い出す設計）
5. J9（BNO055）の3/4ピン極性は設計メモ9節の未決事項 — ケーブル側で入替可能なためこの段階では確定配線とする
