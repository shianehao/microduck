# 移植：換 Rockchip 主板、換 Feetech 伺服

Status: idea · Date: 2026-11-04

這是一份**還沒成為設計**的整理：評估把主板換成更強的 Rockchip SoC（RK3576／RK3588S），以及把
XL330 伺服換成 Feetech HD-1910-C001，各要動到哪些地方、卡在哪裡、該先驗證什麼。裡面沒有任何決定。
真的要做時，主板的部分落在 [`robotd-design.md`](../design/robotd-design.md) 的範圍聲明與
`robotd-params/src/board.rs`，伺服的部分落在 `robotd-design.md` §1.1／§2 — 那兩處擁有機制，本頁只指路。

[§4](#4-腳換成自走輪) 評估把腳換成有動力的輪子（輪腳式），[§5](#5-頭部換成機器人手臂) 評估把頭換成機器人手臂。
順帶評估過、結論是**不建議**的兩個選項，理由記在 [§6](#6-評估過但不建議的選項)：Raspberry Pi 5、Arduino UNO Q。

---

## 1. 現況基準

| 子系統 | 現在用的 | 機制的擁有者 |
|---|---|---|
| 主板 | Radxa Zero 3W（RK3566，4×A55，約 65×30 mm）／beta：Seeed 的 RK3566 板 | `robotd-params/src/board.rs` |
| 伺服 | 15 顆 Dynamixel XL330，`/dev/ttyS2`、1 Mbps、protocol v2 | [`robotd-design.md`](../design/robotd-design.md) §1.1 |
| 身體 IMU | `imu_to_dxl` v2 板（LSM6DSV16X，ID 200），和伺服**同一次** fast `sync_read` | `robotd-design.md` §1.1、`duck-control/src/imu.rs` |
| 影像 | `mpph264enc`（RK VPU）→ `webrtcsink`；相機走 rkisp／rkaiq | [`media-bringup.md`](../project/media-bringup.md) |
| 偵測器 | `.rknn` 跑 NPU，`.onnx` 退回 CPU | [`npu-bringup.md`](../project/npu-bringup.md) |
| 電源 | 2S NP-F550（6.6–8.2 V），伺服直接吃電池 | `robotd-design.md`（battery 段） |
| 策略 | `microduck_rl`：mjlab／MuJoCo Warp＋PPO，伺服以 BAM M6 模型模擬 XL330 | [microduck_rl](https://github.com/pollen-robotics/microduck_rl) |

### 1.1 Microduck 用過、正在用、規劃中的 Rockchip 板

板子是**宣告的，不是偵測的**：`robotd.toml` 的 `[board] version` 決定一切，`Board::detect` 只給
`robotctl health` 比對用（`robotd-params/src/board.rs`）。兩塊板都是 RK3566，所以**同一個 aarch64 release
兩塊都跑**，差異由 daemon 依宣告的板子處理，updater 只看 `hw_rev`
（[`updater-design.md`](../design/updater-design.md) §5.6）。

| | `zero3`（`hw_rev = 1`，預設） | `beta`（`hw_rev = 2`） |
|---|---|---|
| 狀態 | 現役。§5.6 稱它為「暫定 — 手邊最近的板子，不是最終選擇」 | 現役。自製板的**第一版（first spin）** |
| 主板 | Radxa Zero 3W，RK3566 | Seeed 做的 RK3566 主板 |
| device-tree `compatible` | `radxa,zero3w-aic8800ds2`（映像）／`radxa,zero3`（Armbian） | `pollen,microduck-beta`（我們的 DT）／`seeed,microduck`（Seeed 出廠映像） |
| 作業系統 | Armbian 26.2.x、Debian 13 userland、Rockchip BSP kernel 6.1.115 | `microduck_yocto` 映像（A/B slot，RAUC），或 Seeed 出廠映像 |
| 擴充 | prototype 的 HAT | 臉部板＋電源板 |
| 無線 | AIC8800（Wi-Fi＋BT 共用射頻，有配對 workaround） | — |
| 音訊 codec | TLV320AIC3104（HAT，I²C3） | 由映像的 ALSA 設定指定 |
| 相機 | IMX219，安裝角 90° | GC2093，安裝角 0°，Seeed 的 IQ 檔 |
| 頭部 IMU | HAT 上的 BMI088，`tofd` 讀，預設關 | 臉部板的 LSM6DSV16X，`robotd` 讀，預設開 |
| ToF | VL53L5CX／VL53L8CX，HAT 的 I²C3 | VL53L8CX，SPI3 CS0 |
| NFC | 桌上用 CLRC663（USB 序列） | ST25R100，SPI，兩支天線 |
| 臉部 LED | 無 | 12 顆，接 GPIO expander |
| 身體 IMU 安裝 | 電源板直立：`SflpDecoder::DEFAULT_MOUNT` | 電源板平躺：`SflpDecoder::BETA_MOUNT` |
| SoC 序號來源 | device tree | `/proc/cpuinfo` |
| NPU | Armbian 的 DT 預設關，要套 `rk3568-npu-enable` overlay | — |

**規劃中的**：repo 裡沒有寫下任何 RK3566 以外的下一塊板。看得到的方向只有兩點：`beta` 是自製板的
「first spin」，暗示之後還有後續版本；`zero3` 被明說是暫定。§2 的 RK3576／RK3588S 是本頁的評估，
**不是**專案的規劃。

**加板與退役的機制**：新板是 `Board` 加一個變體、`hw_rev` 往上遞增；退役是設定 `Board::last_release`，
之後的 release 就以更高的 `min_hw_rev` 打包，舊板的 updater 會拒絕並說明原因（`updater-design.md` §5.6
「Retiring a board」）。boards 只能由舊到新依序退役。

**換 SoC 時這條「一個 build」還成立嗎**：成立。RK3576／RK3588S 一樣是 aarch64，同一份 Rust release
可以照跑；變的是映像裡的 BSP、MPP、rkaiq、RKNN 執行環境與 overlay — 依 §9.1，這些要由 `hooks/` 帶上板。
（Radxa 的 `rk3588s2-bookworm` 套件池裡的 `gstreamer-rockchip` 和 RK3566 用的是同一版，
見 [`media-bringup.md`](../project/media-bringup.md)，SoC 差異在 MPP 內部。）

---

## 2. 換 Rockchip 主板

### 2.1 為什麼是 Rockchip

整條影像與偵測器路徑都建在 Rockchip 專屬元件上：MPP 硬體 H.264 編碼、rkisp／rkaiq、RKNN。換到同家族的
SoC，這些都能沿用；換到別家（Pi 5、Qualcomm）就得重寫影像管線、失去 NPU。依 [CLAUDE.md](../../CLAUDE.md)，
WebRTC 是相機的預設路徑，所以硬體編碼不是可有可無。

| | RK3566（現行） | RK3576 | RK3588S |
|---|---|---|---|
| CPU | 4×A55 | 4×A72＋4×A53 | 4×A76＋4×A55 |
| NPU | 約 1 TOPS | 6 TOPS | 6 TOPS |
| MPP H.264 編碼 | ✅ | ✅ | ✅ |
| 功耗 | 最低 | 中 | 最高，負載下要散熱片、可能要風扇 |

（RK3566 以外的規格依公開資料整理，採購前對原廠規格書。）

### 2.2 候選板（價格為 2026-11 搜尋所得，時間點不一，下單前再確認）

| 板子 | SoC | 形式／尺寸 | 上市價 | 近期零售觀察 | 適合 |
|---|---|---|---|---|---|
| Radxa CM5 | RK3588S | 模組，CM4 腳位（部分相容），約 55×40 mm | 約 $75（4G/32G） | 歐洲 €131（含稅）；16G/128G 約 $346 | 自製載板裝進機身 |
| Orange Pi CM5 | RK3588S | 模組，類 CM4 | $70（4G/32G 首發） | AliExpress 約 $78 起；官方底板約 $20 | 同上，較便宜 |
| Firefly CAM-3576 | RK3576 | 約 38×38 mm | — | — | 最小，最好塞 |
| Radxa ROCK 4D | RK3576 | Pi 3 大小 | 約 $30（2 GB） | 實測待機約 2.7 W | 桌上開發 |
| Radxa ROCK 5C | RK3588S2 | 約 85×56 mm，板載 Wi-Fi／BT | $49.9（2 GB） | 歐洲 RS €65（含稅） | 桌上開發 |
| Orange Pi 5／5B | RK3588S | 約 100×62 mm | $60 起 | 部分地區已難買 | 桌上開發 |
| Khadas Edge2 | RK3588S | 約 82×58×5.7 mm | — | 約 $489 | 薄，但貴 |

**建議路線**：先買一塊桌上板（ROCK 5C 或 ROCK 4D，4 GB 足夠）驗證軟體；要裝進機身再換 CM5 或 CAM-3576
搭自製載板。8 GB 以上對這個專案沒有明顯幫助。

**採購通路**：AliExpress 官方旗艦店（最便宜，1–3 週）、Radxa 官網列的經銷商、Amazon（較貴）、RS／Newegg；
台灣可搜露天、蝦皮。帶 Wi-Fi／BT 的板子屬射頻器材，個人進口依 NCC 與關務署現行規定；比價時算進關稅、營業稅。

### 2.3 挑板的硬體條件

- 一個能跑 **1 Mbps 的 UART**（伺服匯流排）
- **MIPI CSI**，且廠商有 IMX219（或打算換的感測器）的驅動與 overlay
- **I²C、SPI**（ToF、NFC、頭部 IMU）、**I²S**（codec、麥克風、喇叭）
- 板載 **Wi-Fi／藍牙**，否則要另外加模組
- 電源：多數吃 5 V，2S 電池與主板之間要一顆能撐峰值電流的降壓模組

### 2.4 軟體要動的地方

1. `robotd-params/src/board.rs`：加一個 `Board` 變體 — `label`、`hw_rev`（往上遞增，這是 updater
   `min_hw_rev` 的依據，見 [`updater-design.md`](../design/updater-design.md) §5.6）、`camera_sensor`、
   `camera_mount_degrees`、`detect()` 的 device-tree `compatible` 字串，並更新 `BOARD_LABELS`、`ALL`。
2. `deploy/robotd.toml`：伺服 `port` 換成新板的 UART 裝置；`[head_imu]` 的預設依板而定。
3. `scripts/setup-board.sh`、`setup-gstreamer.sh`、`setup-rkaiq.sh`：新板的 overlay、序列埠 getty 遮蔽、
   MPP／rkaiq 版本。凡是對板子做的事都要能從 `hooks/` 跑到（`updater-design.md` §9.1）。
4. 偵測器：`.rknn` 要針對新 SoC 用 RKNN Toolkit 重新轉換。
5. `microduck_yocto`：換成新 SoC 的 BSP — 最大的一塊，不在本 repo。
6. `scripts/board-test.sh` 對新板的 userland 跑一次。
7. 更新 `robotd-design.md` 開頭的範圍聲明（目前寫「only the Radxa」）— 文件先改，程式才跟著改。

### 2.5 作業系統

**挑核心比挑發行版重要。** `mediad` 的 MPP 硬體編碼（`/dev/mpp_service`）、rkisp／rkaiq 與 RKNN 執行環境都要
Rockchip 原廠的 **BSP 核心（目前 6.1）**。現在 `zero3` 跑 Armbian 26.2.x＋BSP 6.1.115，`beta` 跑
`microduck_yocto`（[`updater-design.md`](../design/updater-design.md) §5.6）。

**主線核心（2026）**：RK3588／RK3576 的 H.264／H.265 硬體**解碼**（VDPU381／VDPU383）已由 Collabora 合併
（報導寫 7.0 或 7.1，不一致）；Microduck 需要的硬體**編碼**、以及 ISP，找不到已合併的資料。所以現在的影像管線
搬不到主線上；等兩者進了主線，`mediad` 還得從 MPP 改成 V4L2 編碼介面，是另一個大工程。

**候選與活躍度**（2026-11-04 從各 repo commit 頁第一頁看的快照，不是完整歷史）：

| 作業系統／元件 | 核心 | 最近 commit | 頻率 | 活躍度 |
|---|---|---|---|---|
| **Armbian**（`armbian/build`） | `vendor`（BSP）＋`current`／`edge`（主線，已跟到 7.3） | 11/4 | 三天 35 個，一天十幾個 | 🟢🟢🟢 |
| Rockchip 原廠核心（`rockchip-linux/kernel` `develop-6.1`） | BSP 6.1 | 7/29 | 7 月下旬一波約每週 25 個，之後無；未見 6.6 vendor 分支 | 🟢🟢 |
| Rockchip MPP（`rockchip-linux/mpp`） | — | 11/3 | 每週約 4 個 | 🟢🟢 |
| **Radxa OS**（`RadxaOS-SDK/rsdk`），Debian 12＋RK 6.1 | BSP 6.1 | 11/2 | 平均兩天一個，10 月中後每週約 5 個 | 🟢🟢 |
| gstreamer-rockchip（`JeffyCN/mirrors`） | — | 10/21 | 10 月一波 15 個，之前每隔幾週一個 | 🟡🟢 |
| RKNN Toolkit2（`airockchip/rknn-toolkit2`） | — | v2.5.0（10/20） | 每幾個月一版 | 🟢 |
| Orange Pi 官方（`orangepi-xunlong/orangepi-build`） | BSP | 9/28 | 2026 年約每月一次，5–9 月斷了近四個月 | 🟡 |
| BredOS（Arch Linux ARM，RK3588） | — | 核心 9/4、映像 2/24 | 上半年後很少更新 | 🟡🔴 |
| ubuntu-rockchip（Joshua Riek） | BSP 6.1 | — | **2026-04-29 封存**；`defcom5-rockchip` 有小型 fork | 🔴 |
| Yocto `meta-rockchip` | — | 未查到（`git.yoctoproject.org` 不在當時 session 的網路允許範圍） | — | ❓ |
| Rockchip Linux SDK（Buildroot／Debian／Yocto）、Khadas／Firefly／Advantech 映像 | BSP 6.1 | — | 透過廠商與代理商取得 | — |

觀察：

- **Armbian 最活躍，且仍在維護 rk35xx vendor 核心**：11/2–11/4 有 PCA953X GPIO expander、TLV320AIC31xx codec
  （和 `zero3` HAT 的 TLV320AIC3104 同系列）、RK3576 NPU 修補等 commit。
- **原廠核心還停在 6.1**，和 Microduck 現在用的一致，短期內沒有升級壓力。
- **影像元件還有人維護**：MPP 每週更新，gstreamer-rockchip 10 月有一波。
- 避開已封存或很少更新的：ubuntu-rockchip、BredOS；Orange Pi 的板子改用 Armbian 映像。

**建議**：

| 階段 | 作業系統 | 原因 |
|---|---|---|
| 換板的桌上驗證 | **Armbian vendor 分支** | 最活躍、和 `zero3` 最接近，`setup-board.sh` 等最容易沿用 |
| Radxa 的板子 | Armbian 為主，Radxa OS 備案 | Radxa OS 是原廠且穩定維護 |
| 正式產品 | **`microduck_yocto`**，換新 SoC 的 BSP layer | 維持專案方向；`meta-rockchip` 活躍度待確認 |

不論哪個作業系統，安裝時對板子做的事都要放進 `hooks/`（`updater-design.md` §9.1）。

---

## 3. 換 Feetech HD-1910-C001 伺服

### 3.1 這顆伺服（賣家資料，未對原廠規格書）

| | XL330（現行） | HD-1910-C001 |
|---|---|---|
| 電壓 | 規格 3.7–6.0 V，實際吃 2S（6.6–8.2 V） | 5–8.4 V（另一家寫 4.8 V，待確認）— 比 XL330 更合 2S |
| 扭力 | 較小 | 標示 10 kg·cm（約 1 N·m），金屬齒輪、空心杯馬達 |
| 協定 | Dynamixel protocol v2 | Feetech TTL 匯流排協定（SCS／STS 格式，接近 Dynamixel v1） |
| 價格／供貨 | 穩定 | 約 $16–35；多家缺貨 |

賣家標示「for Microduck」，但本 repo 完全沒有提到這顆伺服，機構是否相容要實際確認。

### 3.2 五個障礙

1. **身體 IMU 的匯流排**（最根本）。`imu_to_dxl` 講 Dynamixel v2，和伺服同一次讀取。換 Feetech 後要擇一：
   - 同一條線輪流送兩種協定 → 每 tick 兩次交易，時序要重新量；
   - 改 IMU 板韌體講 Feetech 協定（韌體原始碼是否公開未知）；
   - 把 IMU 移到別的介面（beta 臉部板也是 LSM6DSV16X）。
2. **策略要重訓**（最花時間）。`microduck_rl` 以 BAM M6 模擬 XL330；換伺服要先做致動器辨識，再更新 MJCF、
   重訓全部任務。`microduck_rl` 沒有公開 XL330 的辨識資料與流程，BAM 本身（Rhoban）是開源的，可參考。
3. **匯流排層重寫**。`duck-control/src/bus.rs`（839 行）全綁 `rustypot` 的 `Xl330Controller`、fast sync read
   （0x8A，v2 專有）、XL330 的暫存器與單位、新伺服自動接手、EEPROM 檢查、硬體錯誤狀態。好消息是有
   `RobotIo` 這個接縫（`duck-control/src/io.rs`），新寫一個 Feetech 實作即可，上層不動。`rustypot` 1.8.0
   已有 `sts3215`、`scs0009`、`scs0043`，**沒有 HD-1910**；暫存器表若同 STS，加一個型號定義就好。
4. **依賴電流的功能要重調**：被抱起偵測（`duck_control::pickup`）、安全限制。兩家電流單位與精度不同。
5. **機構**：尺寸、孔位、舵盤若不同，腿和脖子要重新設計。

好處：扭力大、金屬齒輪耐摔、電壓範圍涵蓋 2S、可能更便宜。

### 3.3 驗證順序

1. 取得原廠規格書與暫存器表，確認是否同 STS 系列。
2. 台架測一顆：用 `rustypot` 的 `sts3215` 控制器讀寫；量 1 Mbps 下 15 顆一次 sync read 的時間，要塞進
   20 ms（50 Hz）預算。
3. 決定身體 IMU 的接法（§3.2 第 1 點）。
4. 致動器辨識 → 更新模擬 → 在 `microduck_rl` 重訓。
5. 寫 Feetech 版 `RobotIo`，加設定開關（例如 `[bus] servo = "hd1910"`），同步更新 `robotd-design.md`
   （目前寫「only the v2 `imu_to_dxl` board」）。

---

## 4. 腳換成自走輪

### 4.1 和現有 roller 模式的差別

現在的 `policy.mode = "roller"` 是**被動輪**：輪子沒有動力，靠雙腳像溜冰一樣推（`microduck_rl` 的 swizzle、
push／brake、下坡等任務）。它只是一組策略加調校預設（`deploy/robotd.toml`、`robotd-design.md` §4.2）。
自走輪是**有馬達驅動的輪子**，是另一件事。

### 4.2 三種做法

| 做法 | 硬體 | 主動平衡 | 軟體改動 | 難度 |
|---|---|---|---|---|
| **A. 腳踝伺服改驅動輪**（輪腳式） | 左右腳踝的 XL330 改連續旋轉直接帶輪；腳掌改成輪子 | **要**：兩輪並排，前後是倒單擺 | 中：仍是 15 顆，2 顆改速度控制 | ⭐⭐⭐ |
| **B. 腳底另加 2 顆輪子馬達** | 新增 2 顆伺服（新 ID，例如 15、25），同一條匯流排 | 要 | 大：15 → 17 個關節，線路協定要改 | ⭐⭐⭐⭐ |
| **C. 四輪或加萬向輪** | 2 顆驅動輪＋1–2 顆萬向輪 | 不用 | 小：差速驅動，可不靠 RL | ⭐⭐，但不太像鴨子 |

**建議 A**：保留雙腳的姿態能力（蹲、伸、傾斜），不增加伺服，且輪腳式用 RL 訓練平衡是成熟做法。

### 4.3 做法 A 的硬體

- XL330 支援速度控制模式（Operating Mode 1），同一條匯流排、同樣的 ID 即可；Feetech STS 系列也有輪子模式。
- 輪徑與速度：XL330 空載約每分鐘一百轉上下，40 mm 輪約 0.2 m/s（依一般規格估，未實測）。輪越大越快、扭力越小。
- 沒有腳踝俯仰，平衡只靠輪子前後移動加髖、膝的姿態；重心越低越好，可能要調整電池位置。

### 4.4 做法 A 的程式（本 repo）

1. **匯流排層**（`duck-control/src/bus.rs`）：每 tick 從一次寫入變兩次 — 13 顆寫 goal position、2 顆寫 goal
   velocity；開機把那 2 顆設成速度模式，EEPROM 檢查依關節區分；輪子的位置會一直累加，改讀速度。
2. **觀測與動作向量**：現在是 `[gyro(3) | projected_gravity(3) | joint_pos(14) | joint_vel(14) |
   last_action(14) | command(13)]`（`robotd-design.md` §4）。輪子的位置沒有意義，要換成速度；動作裡那兩維從角度
   改成速度。這是新的策略格式，要同步 [`policy-manifest.md`](../policy-manifest.md)（擁有該格式）與
   `microduck_rl` 的 `publish` 形狀檢查（目前寫死 `[1,61] → [1,14]`）。
3. **新模式**：`policy.mode` 加 `wheeled`，和 `roller` 一樣是預設組，一起切換策略、增益、安全參數。
4. **安全**：跌倒時**輪子要立刻停**；現有跌倒處理只管位置控制，倒地後還在轉的輪子會把機器人往前拖。`roller`
   模式「完全不監看跌倒」（`robotd-design.md`，因為模型沒看過輪子）— `wheeled` 不能比照。
5. **線路協定**：若 `robot.state` 要回報輪速、或 `JOINT_NAMES` 對外意義改變，依 [CLAUDE.md](../../CLAUDE.md)
   bump `API_VERSION` 並說明安裝影響。
6. **odometry**：可直接由輪速算位移，比腳的觸地點準，但要另寫一套。

### 4.5 策略訓練（`microduck_rl`）

- MJCF：腳踝關節改輪子關節並加速度致動器；可從 roller 模型的被動輪關節改起。
- 致動器：XL330 在速度模式下的行為（扭力—轉速、延遲）和位置模式不同，要另外辨識。
- 任務：原地平衡、速度追蹤、蹲低、倒地站起（輪腳式怎麼站起來要重新設計）。
- Domain randomization 另加：地面摩擦、輪子打滑、輪徑誤差。

### 4.6 優缺點

- **優點**：平地比走路快、省電；可邊移動邊改變姿態；腳踝伺服直接沿用。
- **缺點**：樓梯、地毯邊、高低差難過；停著也要主動平衡，沒電或當機就倒；走路、踢球、前滾翻等現有技能不能用
  （除非輪子能鎖住當腳）；機構、匯流排、策略格式、訓練、安全都要動。

### 4.7 驗證順序

1. 先在 `microduck_rl` 改出輪腳版 MJCF，訓練原地平衡＋速度追蹤，確認可行再做實體（RTX 3050，平地、1024 環境）。
2. 台架：一顆 XL330 設速度模式，量負載下轉速與扭力，決定輪徑。
3. 原型：只改兩腳末端，其他不動。
4. 程式：兩次寫入、`wheeled` 模式、輪子停轉的安全機制、新策略格式。

---

## 5. 頭部換成機器人手臂

### 5.1 脖子本來就是一隻小手臂

脖子和頭有 4 個關節（`neck_pitch`、`head_pitch`、`head_yaw`、`head_roll`，ID 30–33），加上當夾爪用的嘴巴
（`mouth`，ID 34）。`GroundPick` 就是低頭用嘴巴撿東西（`robotd-design.md`）。所以有兩條路：把脖子改造成手臂，
或拆掉頭另裝一隻多軸手臂。

### 5.2 兩種做法

| | **A. 脖子當手臂、嘴巴當夾爪** | **B. 拆頭，裝獨立手臂** |
|---|---|---|
| 伺服 | 維持 15 顆，ID 30–34 不變 | 增加（例如 4–6 軸＋夾爪），要新 ID |
| 機構 | 加長脖子連桿、嘴巴換夾爪，相機留在「頭」 | 全部重設計，相機與感測器另找位置 |
| 協定與策略格式 | **不用改**：`JOINT_NAMES`、觀測向量格式不變 | 要改：`JOINT_NAMES`、觀測／動作維度、[`policy-manifest.md`](../policy-manifest.md)，bump `API_VERSION` |
| 平衡 | 頭本來就是指令的一部分，策略已學過頭會動 | 手臂重又伸得遠，重心升高，策略要重學抵抗干擾 |
| 難度 | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

**建議 A。**

### 5.3 為什麼 A 划算

1. **策略已考慮頭的動作**：`command(13)` 是 `vel(3) + head(4) + body(6)`，頭部 4 個關節角也在觀測向量裡
   （`robotd-design.md` §4）。只需重訓以適應新的質量與長度。
2. **控制介面現成**：`robot.head` 直接指定 4 個關節角；手把有 `[pad_head]`，手把 IMU 還能直接控頭的姿態。
3. **眼在手上（eye-in-hand）**：相機、ToF、頭部 IMU 都在頭上，正好在夾爪旁；`kinematics/src/head.rs` 已有相機與
   ToF 的順向運動學。

### 5.4 A 的限制

- **扭力**：脖子根部的需求大致 ∝ 重量 × 伸出距離。XL330 大概只能抓幾十公克；要更重就換高扭力伺服，又會碰到
  [§3.2](#32-五個障礙) 的匯流排問題。
- **自由度**：4 軸＋夾爪無法任意對準位置與方向，要靠身體配合（蹲低、轉身）。
- **沒有 IK**：`kinematics` 只有順向運動學；「把夾爪移到某點」要另寫逆向運動學，或直接訓練 RL 抓取策略。
- **連帶影響**：合唱與 theremin 的嘴巴動畫要重調；外型改變，鴨子的個性會變淡。

### 5.5 B 要面對的問題

1. **平衡**：手臂的重量與伸距對 800 g 的機器人是大干擾，要在訓練中隨機擺動手臂、學邊動邊平衡
   （loco-manipulation，研究等級）。
2. **匯流排時序**：每多一顆伺服，`sync_read` 就多一份回應；1 Mbps 下應仍塞得進 20 ms，要實測。
3. **協定**：新增 `robot.arm` 之類的指令、改 `JOINT_NAMES`，依 [CLAUDE.md](../../CLAUDE.md) bump `API_VERSION`；
   版本差異記錄並照常服務，不拒絕。
4. **感測器搬家**：`frames: {camera, tof}` 由頭部關節 FK 算出，位置換了計算也要改。
5. **失去的功能**：鴨子的臉、嘴巴動畫、theremin、頭部互動。
6. **安全**：手臂靠近人時，要有扭力與速度上限。

### 5.6 驗證順序（做法 A）

1. **模擬**：在 `microduck_rl` 的 MJCF 加長脖子、嘴巴改夾爪，重訓走路策略，確認頭部指令持續變化時仍能平衡。
2. **扭力評估**：算出目標伸距下脖子根部所需扭力，對照 XL330，決定是否換伺服。
3. **抓取**：先用「搖桿控頭＋按鍵開合嘴巴」手動抓，不用改軟體、現在就能測；之後再考慮 IK 或 RL 抓取策略。
4. **原型**：只換脖子與頭的機構，伺服、ID、程式都不動。

---

## 6. 評估過但不建議的選項

- **Raspberry Pi 5**：沒有硬體 H.264 編碼（`mediad` 要改軟體編碼，和 50 Hz 迴圈搶 CPU）、沒有 NPU、相機
  堆疊要改 libcamera、要新 OS 映像；85×56 mm、5 V／5 A、要主動散熱，塞不進 25 cm／800 g 的機身。
  專案本來就刻意不支援 Pi（`robotd-design.md` 開頭）。
- **Arduino UNO Q**（QRB2210，4×A53）：算力和 RK3566 同級，換了沒有收益；失去 MPP／RKNN；68.58×53.34 mm；
  Vin 下限 7 V 低於 2S 負載下的 6.6 V，實際仍要穩壓。內建 STM32 理論上能專管伺服匯流排，但 `robotd` 在一般
  Linux 上已能穩定 50 Hz，不值得為此重構。

---

## 7. 相關事實

### 7.1 機構與授權

- 本 repo 沒有 CAD。MJCF＋STL 在 `microduck_rl`（由 Onshape 以 onshape-to-robot 匯出，Onshape 原檔未公開）；
  **STL 授權為 CC BY-NC-SA 4.0，不可商用**。依報導，機構與電路設計不開源。
- 改尺寸會連動：`microduck_rl` 的 MJCF、本 repo 嵌入的 `kinematics/assets/alpha/robot_walk.xml` 與其測試
  fixture、`odometry/src/anchors.rs`（產生檔）、`robotctl/assets/duck.bin`（`scripts/bake-duck-mesh.py`）、
  感測器 mount（`SflpDecoder`、相機角度），以及全部策略。
- 等比例放大 *k* 倍：質量 ∝ *k*³、抗重力扭力 ∝ *k*⁴、慣量 ∝ *k*⁵。放大很快就超出 XL330 能力 — 這也是
  Feetech 伺服被考慮的理由之一，但兩件事一起改，風險疊加。

### 7.2 在本地訓練（RTX 3050 Laptop，4 GB，38 W）

- Ampere 架構、驅動 595／CUDA 13.2，MuJoCo Warp 可以跑。
- VRAM 主要吃在模擬資料，與環境數成正比；從 `--env.scene.num-envs 1024` 開始，用 `nvidia-smi` 看用量，
  有餘裕再試 2048。先練 `-Flat-` 任務。
- 預估比 README 的「4096 環境 1–2 小時」慢上數倍（未實測）。要能上機的策略，用 `--hf-jobs` 或租雲端 GPU。

---

## 8. 未決問題

- HD-1910 的原廠協定與暫存器表；是否真的與 Microduck 機構相容。
- `imu_to_dxl` 韌體是否公開、能否改講 Feetech 協定。
- 目標板的 IMX219 驅動／overlay、UART 1 Mbps 實測、I²S codec 支援。
- XL330 的 BAM 辨識資料能否向上游取得，作為新伺服辨識的對照。
- `microduck_yocto` 是否有計畫支援 RK3576／RK3588S 的 BSP；Yocto `meta-rockchip` 的活躍度。
- 主線核心何時有 RK3588／RK3576 的硬體 H.264 編碼與 ISP；Armbian 2026 年 RK3588 vendor 分支的支援範圍。
- 自走輪：XL330 速度模式在負載下的實際轉速與扭力；輪腳式從倒地站起的動作；輪子能否鎖住以保留走路技能。
- 手臂：目標伸距與負載下脖子根部的扭力需求；夾爪機構；抓取用 IK 還是 RL 策略。
