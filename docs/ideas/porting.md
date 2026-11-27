# 移植：換 Rockchip 主板、換 Feetech 伺服

Status: idea · Date: 2026-11-04

這是一份**還沒成為設計**的整理：評估把主板換成更強的 Rockchip SoC（RK3576／RK3588S），以及把
XL330 伺服換成 Feetech HD-1910-C001，各要動到哪些地方、卡在哪裡、該先驗證什麼。裡面沒有任何決定。
真的要做時，主板的部分落在 [`robotd-design.md`](../design/robotd-design.md) 的範圍聲明與
`robotd-params/src/board.rs`，伺服的部分落在 `robotd-design.md` §1.1／§2 — 那兩處擁有機制，本頁只指路。

[§4](#4-腳換成自走輪) 評估把腳換成有動力的輪子（輪腳式），[§5](#5-頭部換成機器人手臂) 評估把頭換成機器人手臂，
[§6](#6-rockchip-以外的板子) 比較 Rockchip 以外的主板。評估過、結論是**不建議**的選項，理由記在
[§7](#7-評估過但不建議的選項)：Raspberry Pi 5、Arduino UNO Q、NVIDIA Jetson Orin Nano。

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
`robotctl health` 比對用（`robotd-params/src/board.rs`）。三塊板都是 RK3566，所以**同一個 aarch64 release
三塊都跑**，差異由 daemon 依宣告的板子處理，updater 只看 `hw_rev`
（[`updater-design.md`](../design/updater-design.md) §5.6）。

`face-board-v1` 是 `beta` 的主板配上 **V1.0 臉部板**：臉部板以外的一切都和 `beta` 相同，程式裡凡是由主板決定
的地方（ToF、NFC、頭部 IMU、身體 IMU 安裝、SoC 序號、相機）都經 `Board::is_beta_main_board()` 當成 `beta`
處理。差別只在臉部板本身。

| | `zero3`（`hw_rev = 1`，預設） | `beta`（`hw_rev = 2`） | `face-board-v1`（`hw_rev = 3`） |
|---|---|---|---|
| 狀態 | 現役。§5.6 稱它為「暫定 — 手邊最近的板子，不是最終選擇」 | 現役。自製板的**第一版（first spin）** | 現役。`beta` 主板＋V1.0 臉部板 |
| 主板 | Radxa Zero 3W，RK3566 | Seeed 做的 RK3566 主板 | 同 `beta` |
| device-tree `compatible` | `radxa,zero3w-aic8800ds2`（映像）／`radxa,zero3`（Armbian） | `pollen,microduck-beta`（我們的 DT）／`seeed,microduck`（Seeed 出廠映像） | `pollen,microduck-face-board-v1`，排在 `pollen,microduck-beta` 前面 |
| 作業系統 | Armbian 26.2.x、Debian 13 userland、Rockchip BSP kernel 6.1.115 | `microduck_yocto` 映像（A/B slot，RAUC），或 Seeed 出廠映像 | 同 `beta`（未另外記載） |
| 擴充 | prototype 的 HAT | 臉部板＋電源板 | **V1.0 臉部板**＋電源板 |
| 無線 | AIC8800（Wi-Fi＋BT 共用射頻，有配對 workaround） | — | 同 `beta` |
| 音訊 | TLV320AIC3104（HAT，I²C3） | 映像的 ALSA `default`：RK809 放音、臉部板麥克風收音 | 同 `beta` |
| 相機 | IMX219，安裝角 90° | GC2093，安裝角 0°，Seeed 的 IQ 檔 | 同 `beta` |
| 頭部 IMU | HAT 上的 BMI088，`tofd` 讀，預設關 | 臉部板的 LSM6DSV16X，`robotd` 讀，預設開 | 同 `beta`（`robotd` 讀，預設開） |
| ToF | VL53L5CX／VL53L8CX，HAT 的 I²C3 | VL53L8CX，SPI3 CS0 | 同 `beta` |
| NFC | 桌上用 CLRC663（USB 序列） | ST25R100，SPI（`/dev/spidev1.0`），兩支天線 | 同 `beta` |
| 臉部 LED | 無 | 12 顆，接 GPIO expander | **兩顆 Awinic LED 驅動晶片（AW21009、AW2023）** 取代 expander |
| 按鈕 | 無 | 無 | **有一顆** |
| 身體 IMU 安裝 | 電源板直立：`SflpDecoder::DEFAULT_MOUNT` | 電源板平躺：`SflpDecoder::BETA_MOUNT` | 同 `beta` |
| SoC 序號來源 | device tree | `/proc/cpuinfo` | 同 `beta` |
| NPU | Armbian 的 DT 預設關，要套 `rk3568-npu-enable` overlay | — | — |

`provision-board.sh` 接受 `--board beta` 和 `--board face-board-v1`，但只是宣告板子，並警告
`setup-board.sh` 仍然是在架一塊 Zero 3W。

**規劃中的**：repo 裡沒有寫下任何 RK3566 以外的下一塊板。看得到的方向是：`beta` 是自製板的「first spin」，
而 `face-board-v1` 已經是在同一塊主板上換了下一版臉部板；`zero3` 被明說是暫定。§2 的 RK3576／RK3588S 是
本頁的評估，**不是**專案的規劃。

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
`microduck_yocto`，`face-board-v1` 用同一塊主板（[`updater-design.md`](../design/updater-design.md) §5.6）。

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

### 2.6 目前推薦：RK3576，第一塊買 Radxa ROCK 4D（2026-11）

**選 RK3576，不選 RK3588S。** 兩者 NPU 都是 6 TOPS、都有 MPP 硬體編碼；RK3576 的 CPU（4×A72＋4×A53）已比
RK3566 強很多，而且功耗、發熱、價格都低。這隻鴨子的瓶頸是功耗與散熱，不是算力 — `microduck-replica` 的 A7Z
（§6.5）拿外部風扇吹裸板，CPU 過 60°C 就開始降頻，機身裡沒有散熱空間，RK3588S 多出來的算力多半會被降頻吃掉。
只有要在機器人上跑重視覺模型、本地 LLM 或多路相機時才選 RK3588S。

**第一塊（桌上驗證）買 Radxa ROCK 4D，4 GB、一般 RK3576（非工業級 RK3576J）**：

1. 和 Zero 3W 同廠（Radxa）。Radxa 支援表列 ROCK 4D 有 Debian 12＋RK 6.1 官方映像，Radxa OS SDK 穩定更新。
2. Armbian 在跟 RK3576（11 月初有 RK3576 NPU 修補與 rk35xx vendor 核心 commit），用它最接近 `zero3`（§2.5）。
3. RKNN Toolkit2 v2.x 起正式支援 RK3576，偵測器模型可重新轉換。
4. 便宜、省電（實測待機約 2.7 W），最後不用也不心疼。

要裝進機身時，再換同一顆 RK3576 的小尺寸模組（Radxa 的 RK3576 運算模組、Firefly CAM-3576 約 38×38 mm）。

**買到後先驗證**（從 A7Z 的經驗來）：

1. **伺服匯流排**：1 Mbps UART 接 15 顆 XL330＋IMU，連續 5 分鐘以上 sync read 不超時。A7Z 卡最久的正是 UART
   FIFO 小、偶發讀取失敗。
2. **硬體編碼**：`gst-inspect-1.0 mpph264enc` 存在，且真能硬體編碼 720p。
3. **相機**：IMX219 驅動、overlay，rkaiq 有沒有 RK3576 的 IQ 檔。
4. **溫度**：在模擬機身的通風條件下跑控制迴圈＋相機串流，看會不會降頻。

**前提：先確認真的需要換板。** RK3566 跑 50 Hz 迴圈與硬體編碼都沒問題（`roadmap.md`：約 15000 tick 漏 3 個）。
換板的理由應該是要跑更重的模型，或 1 TOPS NPU 不夠用。

**購買（價格多為 2025 上市資料，2026 RAM 漲價後可能更高，下單前確認）**：

| 通路 | 價格 | 備註 |
|---|---|---|
| Arace Tech（Radxa 官方商店） | 2 GB $30、**4 GB $40**、8 GB $60、16 GB $100 | 有使用者反映網站庫存資訊不準 |
| AliExpress（Radxa 官方店） | 2 GB $31.41、**4 GB $37.12**、8 GB $57.82、16 GB $99.80 | 通常最便宜，台灣約 1–3 週 |
| AllNet | — | 授權經銷，歐洲為主 |
| Amazon | 4 GB 第三方約 $59.99（2026-01），缺貨 | 偏貴 |
| 露天（台灣秋葉原電子企業社），商品 11090705205381 | **8 GB NT$2,900**（標題含稅，另有 16 GB），庫存 5 | 價格設定於 **2025-06-02**，一年多沒更新，可能不照價出貨；照片上看不到 Wi-Fi 模組 |
| 露天（同一賣家），商品 11091202292684 | 6 GB NT$4,200，庫存 2 | 價格 2026-10-22 更新；照片左側有金屬屏蔽模組（像 Wi-Fi／BT）；註明「不支持 eMMC」 |

露天兩筆是同一賣家、同一款板子。第一筆較划算，但下單前先用露露通問：NT$2,900 是否有效、板上有沒有 Wi-Fi／BT
（ROCK 4D 可能分有無 Wi-Fi 兩種，照片無法確認）、「不支持 eMMC」指什麼。只做桌上驗證，沒有 Wi-Fi 也可接網路線；
要接近實機（WebRTC、手把、BLE 配網）則需要 Wi-Fi／BT。帶射頻的板子個人進口依 NCC 與關務署現行規定。

一起準備：USB-C PD 電源、散熱片、microSD 卡、IMX219 排線（先查 ROCK 4D MIPI CSI 接口規格）、USB 轉 TTL 序列線。

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

### 3.4 已有的實例

社群已經有 HD-1910 的控制程式：`kissqy/Microduck-kissqy` 的 `microduck-hd1910/firmware`（有
`duck-control/src/feetech.rs`），`microduck-replica` 的 A7Z 版本就是用它（§6.5）。它讓實機邁出了步，但伺服
匯流排的偶發讀取失敗至今未解 — 對應 §3.3 第 2 點的時序問題。細節見 §6.5。

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

## 6. Rockchip 以外的板子

### 6.1 結論

最值得注意的是 **Allwinner A733** 的板子（Radxa Cubie A7Z、Orange Pi Zero 3W）：尺寸和 Zero 3W **一樣
（65×30 mm）**，CPU 更強，有 NPU 和 H.264 硬體編碼。但**任何非 Rockchip 的板子，影像管線、相機 ISP、NPU
工具鏈都得重寫**，移植量一定比換另一顆 Rockchip 大（§2）。

### 6.2 候選（規格與價格多為上市時資料，2026-11 搜尋所得，購買前再確認）

| 板子 | SoC | 尺寸 | CPU | NPU | H.264 硬體編碼 | 價格 | 適合度 |
|---|---|---|---|---|---|---|---|
| **Radxa Cubie A7Z** | Allwinner A733 | **65×30 mm** | 2×A76＋6×A55 | 3 TOPS | ✅ 4K30 H.264／H.265 | 上市價 1 GB 版 $15 起，最高 16 GB | ⭐⭐⭐ 尺寸最合，軟體成熟度有風險 |
| **Orange Pi Zero 3W** | Allwinner A733 | Pi Zero 尺寸 | 同上 | 3 TOPS | ✅ | 2026-04 發表 | ⭐⭐⭐ 同上 |
| **Radxa Zero 2 Pro** | Amlogic A311D | Zero 尺寸 | 4×A73＋2×A53 | 5 TOPS | ✅（確切規格未查到） | 2024 上市價 $64.99（4 GB／16 GB） | ⭐⭐ 小又有 NPU，晶片較舊，2026 供貨未查到 |
| **NXP i.MX 8M Plus** 系列 | i.MX 8M Plus | 依廠商，多為模組＋載板 | 4×A53 | 2.3 TOPS | ✅ 1080p60 H.264／H.265，另有 ISP | 約 $135–$224 | ⭐⭐ CPU 偏弱，但工業級供貨長、主線支援好，適合正式產品 |
| **BeagleY-AI** | TI AM67A | Pi 尺寸 | 4×A53 @ 1.4 GHz | 4 TOPS | ❓ 未查到 | 上市價 $70，零售約 $127–$135 | ⭐ CPU 弱、尺寸大 |
| **Radxa Dragon Q6A** | Qualcomm QCS6490 | 比 Pi 小一點 | 8 核 Kryo 670 | 12 TOPS | ✅（確切規格未查到） | 4 GB $59.50 起；12 GB 約 €148 | ⭐ 算力最強、三個相機接口，功耗與體積偏大 |

### 6.3 換成非 Rockchip 要重做什麼

| 項目 | 換另一顆 Rockchip | 換非 Rockchip |
|---|---|---|
| Rust daemon | 照用 | 照用（同樣 aarch64） |
| `mediad` 編碼元件 | `mpph264enc` 照用 | **重寫**：Allwinner 用自家 VPU 函式庫，NXP／TI 走 V4L2，Qualcomm 用自家元件 |
| 相機 ISP | rkisp／rkaiq 照用 | **重寫**：每家 ISP 與調校工具不同 |
| NPU 偵測器 | RKNN 重新轉換 | **換工具鏈**，模型轉換與執行環境都不同 |
| OS 映像 | 換 BSP layer | 換廠商與 BSP，`hooks/` 與 setup 腳本多半要重寫 |
| `Board` 變體 | 要加 | 要加 |

### 6.4 建議

1. **尺寸不變、算力更強**：試 **Radxa Cubie A7Z**。桌上驗證先確認三件事：GStreamer 有沒有可用的 H.264
   **硬體編碼**元件；IMX219 驅動與 ISP；BSP 多久更新一次。Allwinner 的軟體支援一向比 Rockchip 弱，這是最大風險。
2. **長期正式產品**：考慮 **i.MX 8M Plus** — CPU 弱，但工業級供貨長、主線核心支援好、ISP 與編碼器文件齊全。
3. **最少移植工作**：留在 Rockchip（RK3576／RK3588S，§2），影像與 NPU 堆疊都能沿用。

### 6.5 實例：`microduck-replica` 的 Radxa Cubie A7Z 移植（2026-11）

`shianehao/microduck-replica`（fork 自 `Liangjin5437/microduck-replica`，原始碼在其 `codex/cubie-a7z-preview`
分支）的 `radxa-cubie-a7z/` 目錄，就是 §3＋§6 一起做的實際嘗試：**Cubie A7Z（Allwinner A733）＋飛特 HD-1910
伺服**。以下依該目錄自己的記錄，狀態都是**預覽**。

**架構**

- 控制程式不在該 repo：`robotd`／`mediad` 來自 `kissqy/Microduck-kissqy` 的 `microduck-hd1910/firmware`，
  該目錄只放補丁、安裝工具與測試。
- 作業系統：Radxa 官方 A733 Debian 11（bullseye），核心 `5.15.147-21-a733`。
- 子目錄：`control/`（UART、分組同步讀補丁、唯讀安裝器、匯流排監測、診斷）、`control/camera/`（IMX219
  WebRTC 預覽）、`studio/`（網頁中控、虛擬手把）、`thermal/`（溫控修正）、`image/`（整機映像）、`provision/`
  （開熱點配網）。
- 已發布 `preview1` 映像（2026-11-09），但它自己註明是修復前的歷史版，**尚未做燒錄後的實機啟動、聯網、動作、
  聲音驗收**。

**做到的**

- 網頁虛擬手把操作下，**實機已能邁步**（2026-11-06）；長時間穩定行走未驗收。
- 15 顆伺服與 IMU 可在 Studio 標定。
- **溫控修正**：板級 device tree 讓 `power_allocator` 在低溫也把 CPU 鎖在 **416 MHz**；改成 `step_wise` 後大核
  可到 2002 MHz，過 60°C 才降頻。
- IMX219 出 960×720、15 fps 的 WebRTC 畫面（僅原始碼，preview1 預設關閉）。

**沒解決的**

- **伺服匯流排偶發讀取失敗（最大問題）**：5 分鐘 15000 次讀取，有一次失敗 788 次（多在 ID 32、33），另一次
  2050 次（ID 33、34），也有一次全數成功。根因未找到。把 15 顆的 sync read 拆成每組 3 顆（避免回應塞爆
  64 bytes 的 UART FIFO）有改善但沒根治。A7Z 走 `uart-ng`、UART0／ttyAS0 別名與 32 MHz 時鐘，Zero 3W 的
  UART 設定腳本不能照搬。
- **相機是軟體編碼**：補丁新增 `x264enc`（`ultrafast`＋`zerolatency`），`mediad` 的 systemd 給 `CPUQuota=400%`
  — 證實 §6.4 擔心的事：這條路上 GStreamer 沒有現成可用的 H.264 硬體編碼元件。
- 實體手把、ToF、持續語音、關節追蹤未啟用；喇叭能否出聲未確認。
- 走一陣子後**左腳踩右腳跌倒**，排查中。

**和本頁預測的對照**

| 本頁的預測 | 實際 |
|---|---|
| §3.3：要先量 15 顆一次 sync read 能否塞進 20 ms | 正是卡住的地方：UART FIFO 太小，要分組讀，仍偶發超時 |
| §6.4：A7Z 最大風險是硬體編碼 | 確認，目前只能軟體編碼 |
| §6.4：Allwinner BSP 支援較弱 | 溫控 device tree 有 bug，UART 要另外調 |
| §3.2：策略要重訓 | 沿用一套 `R17` 動作模型（站立、行走、起坐、撿東西、前滾翻、踢球），來源未記載 |

這也是 §2.6 推薦留在 Rockchip 的實證：換到非 Rockchip，編碼、溫控、UART 都得自己解。

### 6.6 Raspberry Pi 4／5（以及 Zero 2W、CM4）

**為什麼一直問「有沒有硬體 H.264 編碼」。** 原始畫面 1280×720 UYVY 一張約 1.8 MiB，30 fps 約 55 MB/s
（約 440 Mbps），Wi-Fi 傳不動；H.264 壓到約 2–3 Mbps 才能給 WebRTC（手機、瀏覽器、遠端中繼）用 —
依 [CLAUDE.md](../../CLAUDE.md)，WebRTC 是相機的預設路徑。偵測器不經編碼，它從 raw tee 拿原始畫面。
任何板子都能用 `x264enc` 軟體編碼，差別在代價：CPU 要留給 50 Hz 控制迴圈，機身沒有散熱空間。專案踩過一次：
在管線裡加 `videoflip` 讓 MPP 改用軟體轉換，**97 °C、降頻到 408 MHz、30 fps 只剩 8 fps**
（`mediad/src/pipeline.rs` 開頭的說明）。所以 Rockchip 的 `mpph264enc` 是留在 Rockchip 的最大理由之一。

**相機是 Pi 最強的地方。** IMX219 就是 Pi Camera v2；libcamera（官方維護）、`rpicam-apps`、Picamera2、GStreamer
`libcamerasrc` 都成熟，ISP 調校官方做好，不用像 Rockchip 那樣處理 rkaiq 和 IQ 檔。社群也最大。
對照：Arduino UNO Q 的相機支援還不成熟 — USB webcam 大致可用（用 VIN 供電時要強制 USB host 模式），MIPI CSI
要另購 Media Carrier，libcamera 有 `VIDIOC_STREAMON` 失敗的回報，上游 CSI 驅動未完全合併（2026-11）。

| | Pi 4（BCM2711） | Pi 5 | Pi Zero 2W | CM4 |
|---|---|---|---|---|
| CPU | 4×A72，比 RK3566 強 | 4×A76 | 4×A53 @ 1 GHz，比 RK3566 弱 | 同 Pi 4 |
| 硬體 H.264 編碼 | ✅ V4L2 `v4l2h264enc`，約 1080p30（未重新查證） | ❌ 只能 libx264 | ✅ | ✅ 同 Pi 4 |
| NPU | ❌ | ❌ | ❌ | ❌ |
| RAM | 依型號 | 依型號 | 512 MB | 依型號 |
| 尺寸 | 85×56 mm | 85×56 mm | **65×30 mm**（同 Zero 3W） | 約 55×40 mm，要自製載板 |
| 評估 | Pi 家族裡最合適；桌上驗證 | 桌上算力實驗 | 前身 `microduck_runtime` 跑的就是它，50 Hz 也是從它沿用；後來因 RAM、算力、無 NPU 換到 Radxa | 想用 Pi 生態又要裝機時的路線 |

**Pi 5 用 libx264 的評估。** Microduck 出廠是 **720p30**（`mediad/src/pipeline.rs:269`），constrained baseline、
約 2–3 Mbps。Raspberry Pi 官方白皮書（2026-04，「H.264 encoding performance on Raspberry Pi 5」）說 libx264
在 4×A76 上可即時編碼 **1080p30**；CPU 占用的確切數字未確認。720p 的像素約 1080p 的 44%，加上 `ultrafast`＋
`zerolatency`，推估約一個核心左右（**未實測**）。算力夠，問題在**持續負載的發熱**：桌上有散熱沒問題，裝進
機身很可能降頻，連帶拖慢控制迴圈；也更耗電。

**換到 Pi，程式要改什麼**（Pi 4／5 共通，除註明外）：

1. **編碼**：`mediad` 找不到 `mpph264enc` 時本來就退回 `x264enc`（`mediad/src/pipeline.rs:856`），Pi 5 這端
   幾乎不用改；Pi 4／Zero 2W／CM4 要加 `v4l2h264enc` 路徑才能用硬體編碼。
2. **相機來源**：rkisp 改成 `libcamerasrc`。
3. **像素格式**：管線傳 UYVY，但 `x264enc` 不吃 UYVY（`pipeline.rs:844`）。讓 libcamera 直接出 I420 就不用轉，
   但偵測器與快照的取樣（`uyvy` crate）要另寫 I420 版；或加 `videoconvert`，多一筆 CPU，也違反「整條管線不做
   轉換」的設計（`pipeline.rs` 開頭）。
4. **隔離**：限制 `x264enc` 的 thread 數，用 systemd `CPUAffinity` 讓它和 `robotd` 分開核心。
5. **偵測器**：沒有 NPU，`duck_detect.onnx` 改在 CPU 跑，或外接 Coral／Hailo。
6. **板子本身**：`Board` 加變體、Pi 版的 `setup-board.sh`／overlay／`hooks/`、OS 映像；更新 `robotd-design.md`
   的範圍聲明（目前刻意不支援 Pi）。

**實測方法**（Pi 5，接 IMX219，至少跑 10 分鐘，有／無散熱片各一次）：

```bash
gst-launch-1.0 libcamerasrc ! video/x-raw,format=I420,width=1280,height=720,framerate=30/1 ! x264enc tune=zerolatency speed-preset=ultrafast bitrate=2500 threads=2 ! h264parse ! fakesink sync=false
```

```bash
watch -n 2 'vcgencmd measure_temp; vcgencmd measure_clock arm; top -bn1 | head -15'
```

溫度持續上升或 `measure_clock` 掉下來，就代表放進機身會出問題。

**結論**：

| | Pi 5＋libx264 | Pi 4＋硬體編碼 | RK3582／RK3576＋`mpph264enc` |
|---|---|---|---|
| 編碼的 CPU 負擔 | 高（推估約一核） | 低 | 幾乎沒有 |
| 發熱 | 高 | 中 | 低 |
| `mediad` 要改多少 | 相機來源、格式 | 相機來源、格式、`v4l2h264enc` | 幾乎不用 |
| 適合 | 桌上驗證、算力實驗 | 桌上驗證；裝機走 CM4 | 桌上驗證與裝機 |

想用 Pi 生態：Pi 4 桌上驗證、CM4＋自製載板裝機。要最少移植工作，仍是 Rockchip（§2.6）。

---

## 7. 評估過但不建議的選項

- **Raspberry Pi 5**：沒有硬體 H.264 編碼（`mediad` 要改軟體編碼，和 50 Hz 迴圈搶 CPU）、沒有 NPU、相機
  堆疊要改 libcamera、要新 OS 映像；85×56 mm、5 V／5 A、要主動散熱，塞不進 25 cm／800 g 的機身。
  專案本來就刻意不支援 Pi（`robotd-design.md` 開頭）。libx264 算力上跑得動 720p30，但持續負載發熱，裝機不適合；
  Pi 4／CM4 有硬體編碼，是 Pi 家族裡較合適的路線，見 §6.6。
- **Arduino UNO Q**（QRB2210，4×A53）：算力和 RK3566 同級，換了沒有收益；失去 MPP／RKNN；68.58×53.34 mm；
  Vin 下限 7 V 低於 2S 負載下的 6.6 V，實際仍要穩壓。內建 STM32 理論上能專管伺服匯流排，但 `robotd` 在一般
  Linux 上已能穩定 50 Hz，不值得為此重構。相機支援也還不成熟（§6.6）。
- **NVIDIA Jetson Orin Nano**：GPU 很強，但依現有了解 Orin Nano 沒有硬體編碼器（未重新查證）；功耗約 7–15 W，
  體積也大。

---

## 8. 相關事實

### 8.1 機構與授權

- 本 repo 沒有 CAD。MJCF＋STL 在 `microduck_rl`（由 Onshape 以 onshape-to-robot 匯出，Onshape 原檔未公開）；
  **STL 授權為 CC BY-NC-SA 4.0，不可商用**。依報導，機構與電路設計不開源。
- 改尺寸會連動：`microduck_rl` 的 MJCF、本 repo 嵌入的 `kinematics/assets/alpha/robot_walk.xml` 與其測試
  fixture、`odometry/src/anchors.rs`（產生檔）、`robotctl/assets/duck.bin`（`scripts/bake-duck-mesh.py`）、
  感測器 mount（`SflpDecoder`、相機角度），以及全部策略。
- 等比例放大 *k* 倍：質量 ∝ *k*³、抗重力扭力 ∝ *k*⁴、慣量 ∝ *k*⁵。放大很快就超出 XL330 能力 — 這也是
  Feetech 伺服被考慮的理由之一，但兩件事一起改，風險疊加。

### 8.2 在本地訓練（RTX 3050 Laptop，4 GB，38 W）

- Ampere 架構、驅動 595／CUDA 13.2，MuJoCo Warp 可以跑。
- VRAM 主要吃在模擬資料，與環境數成正比；從 `--env.scene.num-envs 1024` 開始，用 `nvidia-smi` 看用量，
  有餘裕再試 2048。先練 `-Flat-` 任務。
- 預估比 README 的「4096 環境 1–2 小時」慢上數倍（未實測）。要能上機的策略，用 `--hf-jobs` 或租雲端 GPU。

---

## 9. 未決問題

- HD-1910 的原廠協定與暫存器表；是否真的與 Microduck 機構相容。
- `imu_to_dxl` 韌體是否公開、能否改講 Feetech 協定。
- 目標板的 IMX219 驅動／overlay、UART 1 Mbps 實測、I²S codec 支援。
- XL330 的 BAM 辨識資料能否向上游取得，作為新伺服辨識的對照。
- `microduck_yocto` 是否有計畫支援 RK3576／RK3588S 的 BSP；Yocto `meta-rockchip` 的活躍度。
- 主線核心何時有 RK3588／RK3576 的硬體 H.264 編碼與 ISP；Armbian 2026 年 RK3588 vendor 分支的支援範圍。
- 自走輪：XL330 速度模式在負載下的實際轉速與扭力；輪腳式從倒地站起的動作；輪子能否鎖住以保留走路技能。
- 手臂：目標伸距與負載下脖子根部的扭力需求；夾爪機構；抓取用 IK 還是 RL 策略。
- 非 Rockchip：A733 的 H.264 硬體編碼（`microduck-replica` 目前只能用 `x264enc` 軟體編碼，§6.5）、BSP 更新頻率；
  A311D 2026 供貨；AM67A 與 QCS6490 的編碼規格。
- `microduck-replica` A7Z 版伺服匯流排偶發讀取失敗的根因；`R17` 動作模型的來源。
- ROCK 4D：1 Mbps UART 長時間 sync read 是否穩定、`mpph264enc` 在 RK3576 上是否可用、rkaiq 的 IMX219 IQ 檔、
  機身內溫度；露天那兩筆的實際價格與有無 Wi-Fi／BT。
- Pi：Pi 5 libx264 在 720p30 的實際 CPU 占用與溫度（§6.6 的實測方法）；Pi 4 `v4l2h264enc` 的實際規格與在
  `mediad` 上的可行性。
