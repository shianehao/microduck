# 移植：換 Rockchip 主板、換 Feetech 伺服

Status: idea · Date: 2026-11-04

這是一份**還沒成為設計**的整理：評估把主板換成更強的 Rockchip SoC（RK3576／RK3588S），以及把
XL330 伺服換成 Feetech HD-1910-C001，各要動到哪些地方、卡在哪裡、該先驗證什麼。裡面沒有任何決定。
真的要做時，主板的部分落在 [`robotd-design.md`](../design/robotd-design.md) 的範圍聲明與
`robotd-params/src/board.rs`，伺服的部分落在 `robotd-design.md` §1.1／§2 — 那兩處擁有機制，本頁只指路。

順帶評估過、結論是**不建議**的兩個選項，理由記在 [§4](#4-評估過但不建議的選項)：Raspberry Pi 5、Arduino UNO Q。

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

## 4. 評估過但不建議的選項

- **Raspberry Pi 5**：沒有硬體 H.264 編碼（`mediad` 要改軟體編碼，和 50 Hz 迴圈搶 CPU）、沒有 NPU、相機
  堆疊要改 libcamera、要新 OS 映像；85×56 mm、5 V／5 A、要主動散熱，塞不進 25 cm／800 g 的機身。
  專案本來就刻意不支援 Pi（`robotd-design.md` 開頭）。
- **Arduino UNO Q**（QRB2210，4×A53）：算力和 RK3566 同級，換了沒有收益；失去 MPP／RKNN；68.58×53.34 mm；
  Vin 下限 7 V 低於 2S 負載下的 6.6 V，實際仍要穩壓。內建 STM32 理論上能專管伺服匯流排，但 `robotd` 在一般
  Linux 上已能穩定 50 Hz，不值得為此重構。

---

## 5. 相關事實

### 5.1 機構與授權

- 本 repo 沒有 CAD。MJCF＋STL 在 `microduck_rl`（由 Onshape 以 onshape-to-robot 匯出，Onshape 原檔未公開）；
  **STL 授權為 CC BY-NC-SA 4.0，不可商用**。依報導，機構與電路設計不開源。
- 改尺寸會連動：`microduck_rl` 的 MJCF、本 repo 嵌入的 `kinematics/assets/alpha/robot_walk.xml` 與其測試
  fixture、`odometry/src/anchors.rs`（產生檔）、`robotctl/assets/duck.bin`（`scripts/bake-duck-mesh.py`）、
  感測器 mount（`SflpDecoder`、相機角度），以及全部策略。
- 等比例放大 *k* 倍：質量 ∝ *k*³、抗重力扭力 ∝ *k*⁴、慣量 ∝ *k*⁵。放大很快就超出 XL330 能力 — 這也是
  Feetech 伺服被考慮的理由之一，但兩件事一起改，風險疊加。

### 5.2 在本地訓練（RTX 3050 Laptop，4 GB，38 W）

- Ampere 架構、驅動 595／CUDA 13.2，MuJoCo Warp 可以跑。
- VRAM 主要吃在模擬資料，與環境數成正比；從 `--env.scene.num-envs 1024` 開始，用 `nvidia-smi` 看用量，
  有餘裕再試 2048。先練 `-Flat-` 任務。
- 預估比 README 的「4096 環境 1–2 小時」慢上數倍（未實測）。要能上機的策略，用 `--hf-jobs` 或租雲端 GPU。

---

## 6. 未決問題

- HD-1910 的原廠協定與暫存器表；是否真的與 Microduck 機構相容。
- `imu_to_dxl` 韌體是否公開、能否改講 Feetech 協定。
- 目標板的 IMX219 驅動／overlay、UART 1 Mbps 實測、I²S codec 支援。
- XL330 的 BAM 辨識資料能否向上游取得，作為新伺服辨識的對照。
- `microduck_yocto` 是否有計畫支援 RK3576／RK3588S 的 BSP。
