# E2E Transparent Clock 配置分析
## UDPv4 + One-Step 模式設定指南

本文檔分析 linuxptp 專案中如何啟用 **End-to-End Transparent Clock (E2E-TC)** 搭配 **one-step** 同步和 **UDPv4** 網路傳輸。

---

## 1. 配置參數總覽

要啟用 E2E-TC + one-step + UDPv4，需要設定以下三個核心參數：

| 參數名稱 | 設定值 | 功能說明 |
|---------|--------|----------|
| `clock_type` | `E2E_TC` | 指定時鐘類型為 End-to-End Transparent Clock |
| `twoStepFlag` | `0` | 啟用 one-step 模式（0 表示 one-step，1 表示 two-step） |
| `network_transport` | `UDPv4` | 使用 UDP over IPv4 作為網路傳輸層 |

---

## 2. 配置檔案設定

### 2.1 基本 E2E-TC 配置範例

參考 [`configs/E2E-TC.cfg`](file:///d:/prg/arterychip/Davicom/ptp/linuxptp/configs/E2E-TC.cfg)：

```cfg
[global]
priority1        254
free_running     1
freq_est_interval 3
tc_spanning_tree 1
summary_interval 1
clock_type       E2E_TC
network_transport L2
```

> [!IMPORTANT]
> 預設的 E2E-TC 配置使用 `L2` (Layer 2) 傳輸，需要修改為 `UDPv4`。

### 2.2 修改為 E2E-TC + UDPv4 + One-Step

建立新配置檔 `E2E-TC-UDPv4-OneStep.cfg`：

```cfg
[global]
# 時鐘類型設定
clock_type       E2E_TC

# 網路傳輸層設定
network_transport UDPv4

# One-step 模式設定
twoStepFlag      0

# 延遲機制（必須為 E2E）
delay_mechanism  E2E

# 時間戳類型（使用 onestep 硬體時間戳）
time_stamping    onestep

# TC 相關參數
priority1        254
free_running     1
freq_est_interval 3
tc_spanning_tree 1
summary_interval 1

# UDPv4 相關設定
ptp_dst_ipv4     224.0.1.129
udp_ttl          1
```

---

## 3. 程式碼實現分析

### 3.1 Clock Type 定義

在 [`clock.h`](file:///d:/prg/arterychip/Davicom/ptp/linuxptp/clock.h#L42) 中定義：

```c
CLOCK_TYPE_E2E = 0x1000,
```

在 [`config.c`](file:///d:/prg/arterychip/Davicom/ptp/linuxptp/config.c#L172) 中的字串映射：

```c
{ "E2E_TC",  CLOCK_TYPE_E2E },
```

### 3.2 E2E-TC 初始化驗證

在 [`ptp4l.c`](file:///d:/prg/arterychip/Davicom/ptp/linuxptp/ptp4l.c#L237-L245) 中的驗證邏輯：

```c
case CLOCK_TYPE_E2E:
    if (cfg->n_interfaces < 2) {
        fprintf(stderr, "TC needs at least two interfaces\n");
        goto out;
    }
    if (DM_E2E != config_get_int(cfg, NULL, "delay_mechanism")) {
        fprintf(stderr, "E2E_TC needs E2E delay mechanism\n");
        goto out;
    }
    break;
```

> [!WARNING]
> E2E-TC 模式**必須**：
> - 至少配置**兩個網路介面**
> - `delay_mechanism` 必須設定為 `E2E`

### 3.3 One-Step 實現機制

#### 3.3.1 One-Step 判斷函數

在 [`msg.h`](file:///d:/prg/arterychip/Davicom/ptp/linuxptp/msg.h#L473-L478) 中定義：

```c
static inline Boolean one_step(struct ptp_message *m)
{
    if (assume_two_step)
        return 0;
    return !field_is_set(m, 0, TWO_STEP);
}
```

訊息的 `flagField[0]` 中 `TWO_STEP` 位元：
- **清除 (0)** → one-step 模式
- **設定 (1)** → two-step 模式

#### 3.3.2 時間戳類型列舉

在 [`msg.h`](file:///d:/prg/arterychip/Davicom/ptp/linuxptp/msg.h#L76-L82) 中定義：

```c
enum timestamp_type {
    TS_SOFTWARE,
    TS_HARDWARE,
    TS_LEGACY_HW,
    TS_ONESTEP,   // One-step 硬體時間戳
    TS_P2P1STEP,
};
```

#### 3.3.3 TC 中的 One-Step 處理

在 [`tc.c`](file:///d:/prg/arterychip/Davicom/ptp/linuxptp/tc.c#L451-L468) 中，TC 轉發 SYNC 訊息時的處理：

```c
int tc_fwd_sync(struct port *q, struct ptp_message *msg)
{
    struct ptp_message *fup = NULL;
    int err;

    if (one_step(msg)) {
        // 對於 one-step 訊息，TC 建立 FOLLOW_UP
        fup = msg_allocate();
        if (!fup) {
            return -1;
        }
        // 設定 FOLLOW_UP 訊息內容
        fup->header.tsmt = FOLLOW_UP | msg_transport_specific(msg);
        // ... 複製其他欄位 ...

        // 修改原始 SYNC 訊息為 two-step
        msg->header.flagField[0] |= TWO_STEP;
    }

    // 轉發事件訊息
    err = tc_fwd_event(q, msg);
    // ...
}
```

> [!NOTE]
> Transparent Clock 在接收到 one-step SYNC 訊息時：
> 1. 會自動產生對應的 FOLLOW_UP 訊息
> 2. 將原始的 one-step SYNC 轉換為 two-step SYNC
> 3. 在 correction field 中加入 residence time

### 3.4 Network Transport 設定

#### 3.4.1 傳輸類型定義

在 [`transport.h`](file:///d:/prg/arterychip/Davicom/ptp/linuxptp/transport.h#L36-L37) 中定義：

```c
enum transport_type {
    TRANS_IEEE_802_3 = 0,
    TRANS_UDP_IPV4 = 1,
    TRANS_UDP_IPV6,
    TRANS_UDS,
};
```

#### 3.4.2 配置字串映射

在 [`config.c`](file:///d:/prg/arterychip/Davicom/ptp/linuxptp/config.c#L218-L223) 中：

```c
static struct config_enum nw_trans_enu[] = {
    { "L2",    TRANS_IEEE_802_3 },
    { "UDPv4", TRANS_UDP_IPV4   },
    { "UDPv6", TRANS_UDP_IPV6   },
    { NULL, 0 },
};
```

在 [`config.c:321`](file:///d:/prg/arterychip/Davicom/ptp/linuxptp/config.c#L321) 中的預設值：

```c
PORT_ITEM_ENU("network_transport", TRANS_UDP_IPV4, nw_trans_enu),
```

---

## 4. E2E-TC 核心程式碼流程

### 4.1 訊息處理入口

E2E-TC 的訊息處理在 [`e2e_tc.c`](file:///d:/prg/arterychip/Davicom/ptp/linuxptp/e2e_tc.c) 中實現：

```c
enum fsm_event e2e_event(struct port *p, int fd_index)
```

主要處理以下訊息類型：

| 訊息類型 | 處理函數 | 說明 |
|---------|---------|------|
| `SYNC` | `tc_fwd_sync()` | 轉發 SYNC，處理 one-step 轉換 |
| `FOLLOW_UP` | `tc_fwd_folup()` | 轉發 FOLLOW_UP，更新 correction field |
| `DELAY_REQ` | `tc_fwd_request()` | 轉發延遲請求 |
| `DELAY_RESP` | `tc_fwd_response()` | 轉發延遲回應，更新 correction field |
| `ANNOUNCE` | `tc_forward()` | 轉發 announce 訊息 |

### 4.2 Residence Time 計算

在 [`tc.c`](file:///d:/prg/arterychip/Davicom/ptp/linuxptp/tc.c#L268-L313) 的 `tc_fwd_event()` 函數中：

```c
static int tc_fwd_event(struct port *q, struct ptp_message *msg)
{
    tmv_t egress, ingress = msg->hwts.ts, residence;

    // 1. 先發送事件訊息
    for (p = clock_first_port(q->clock); p; p = LIST_NEXT(p, list)) {
        cnt = transport_send(p->trp, &p->fda, TRANS_DEFER_EVENT, msg);
    }

    // 2. 回過頭來收集發送時間戳
    for (p = clock_first_port(q->clock); p; p = LIST_NEXT(p, list)) {
        err = transport_txts(&p->fda, msg);
        egress = msg->hwts.ts;

        // 3. 計算 residence time
        residence = tmv_sub(egress, ingress);

        // 4. 應用 rate ratio 校正
        rr = clock_rate_ratio(q->clock);
        if (rr != 1.0) {
            residence = dbl_tmv(tmv_dbl(residence) * rr);
        }

        // 5. 完成處理並更新 correction field
        tc_complete(q, p, msg, residence);
    }
}
```

### 4.3 Correction Field 更新

在 [`tc.c`](file:///d:/prg/arterychip/Davicom/ptp/linuxptp/tc.c#L179-L238) 中的 `tc_complete_syfup()` 函數：

```c
static void tc_complete_syfup(struct port *q, struct port *p,
                               struct ptp_message *msg, tmv_t residence)
{
    Integer64 c1, c2;

    // 讀取原始 correction field
    c1 = net2host64(fup->header.correction);

    // 加上 residence time
    c2 = c1 + tmv_to_TimeInterval(residence);

    // 加上 peer delay (用於 P2P)
    c2 += tmv_to_TimeInterval(q->peer_delay);

    // 加上非對稱補償
    c2 += q->asymmetry;

    // 寫回
    fup->header.correction = host2net64(c2);
}
```

---

## 5. 命令列啟動方式

### 5.1 使用配置檔

```bash
ptp4l -f E2E-TC-UDPv4-OneStep.cfg -i eth0 -i eth1
```

### 5.2 使用命令列參數

```bash
ptp4l -4 -E -i eth0 -i eth1 \
      --clock_type=E2E_TC \
      --twoStepFlag=0 \
      --time_stamping=onestep
```

參數說明：
- `-4`: 使用 UDPv4 傳輸 (等同於 `--network_transport=UDPv4`)
- `-E`: 使用 E2E 延遲機制 (等同於 `--delay_mechanism=E2E`)
- `-i eth0 -i eth1`: 指定兩個網路介面

---

## 6. 與預設配置的差異對比

### 6.1 預設配置 ([`default.cfg`](file:///d:/prg/arterychip/Davicom/ptp/linuxptp/configs/default.cfg))

```cfg
twoStepFlag      1           # two-step 模式
network_transport UDPv4      # 預設就是 UDPv4 ✓
clock_type       OC          # Ordinary Clock
delay_mechanism  E2E         # E2E 延遲機制 ✓
time_stamping    hardware    # 硬體時間戳
```

### 6.2 E2E-TC 配置 ([`E2E-TC.cfg`](file:///d:/prg/arterychip/Davicom/ptp/linuxptp/configs/E2E-TC.cfg))

```cfg
clock_type       E2E_TC      # Transparent Clock ✓
network_transport L2         # Layer 2 (需改為 UDPv4)
# twoStepFlag 未設定，繼承 default = 1
```

### 6.3 建議的 E2E-TC + UDPv4 + One-Step 配置

```cfg
clock_type       E2E_TC      # ✓ TC 模式
network_transport UDPv4      # ✓ UDPv4 傳輸
twoStepFlag      0           # ✓ One-step
delay_mechanism  E2E         # ✓ E2E 延遲
time_stamping    onestep     # ✓ One-step 硬體時間戳
```

---

## 7. 硬體需求與限制

> [!CAUTION]
> **One-Step 模式硬體需求**
> - 網卡必須支援 **one-step hardware timestamping**
> - 需要 `time_stamping = onestep` 或 `time_stamping = p2p1step`
> - 不是所有網卡都支援此功能

### 7.1 檢查硬體是否支援

使用 `ethtool` 檢查：

```bash
ethtool -T eth0
```

輸出範例：

```
Capabilities:
    hardware-transmit     (SOF_TIMESTAMPING_TX_HARDWARE)
    hardware-receive      (SOF_TIMESTAMPING_RX_HARDWARE)
    hardware-raw-clock    (SOF_TIMESTAMPING_RAW_HARDWARE)
```

### 7.2 時間戳模式選擇

| `time_stamping` 值 | 硬體需求 | 適用場景 |
|-------------------|---------|---------|
| `software` | 無 | 測試用途，精度低 |
| `hardware` | 支援 HW timestamp | 一般 PTP 應用 |
| `legacy` | 舊版 HW timestamp | 相容舊硬體 |
| `onestep` | 支援 one-step HW | **E2E-TC one-step 模式** |
| `p2p1step` | 支援 P2P one-step | P2P-TC one-step 模式 |

---

## 8. 配置驗證

### 8.1 啟動後檢查

啟動 ptp4l 後，檢查 log 輸出：

```
selected /dev/ptp0 as PTP clock
port 1: INITIALIZING to LISTENING on INIT_COMPLETE
port 2: INITIALIZING to LISTENING on INIT_COMPLETE
selected best master clock timed out
clock type: E2E_TC
```

### 8.2 使用 pmc 工具查詢

```bash
pmc -u -b 0 'GET CURRENT_DATA_SET'
```

預期輸出應包含：
- `twoStepFlag 0` (表示 one-step 模式)
- 正確的 network transport 資訊

---

## 9. 常見問題與注意事項

### 9.1 介面數量限制

```c
if (cfg->n_interfaces < 2) {
    fprintf(stderr, "TC needs at least two interfaces\n");
    goto out;
}
```

> [!IMPORTANT]
> Transparent Clock 至少需要**兩個網路介面**，因為它的角色是在不同網路段之間轉發 PTP 訊息。

### 9.2 Delay Mechanism 錯誤

```c
if (DM_E2E != config_get_int(cfg, NULL, "delay_mechanism")) {
    fprintf(stderr, "E2E_TC needs E2E delay mechanism\n");
    goto out;
}
```

E2E-TC 必須搭配 E2E delay mechanism，不能使用 P2P。

### 9.3 twoStepFlag 與 time_stamping 的關係

在 [`config.c`](file:///d:/prg/arterychip/Davicom/ptp/linuxptp/config.c#L1101-L1126) 中有自動調整邏輯：

```c
int two_step_flag = config_get_int(cfg, NULL, "twoStepFlag");

if (timestamping == TS_ONESTEP || timestamping == TS_P2P1STEP) {
    if (two_step_flag) {
        pr_debug("one step mode implies twoStepFlag=0, "
                 "clearing twoStepFlag to match");
        if (config_set_int(cfg, "twoStepFlag", 0)) {
            return -1;
        }
    }
}
```

當 `time_stamping = onestep` 時，系統會自動將 `twoStepFlag` 設為 0。

---

## 10. 總結

### 10.1 必要配置項

```cfg
[global]
clock_type       E2E_TC
network_transport UDPv4
twoStepFlag      0
delay_mechanism  E2E
time_stamping    onestep
```

### 10.2 核心程式碼檔案

| 檔案 | 功能 |
|-----|------|
| [`e2e_tc.c`](file:///d:/prg/arterychip/Davicom/ptp/linuxptp/e2e_tc.c) | E2E-TC 訊息處理邏輯 |
| [`tc.c`](file:///d:/prg/arterychip/Davicom/ptp/linuxptp/tc.c) | TC 轉發與 residence time 計算 |
| [`ptp4l.c`](file:///d:/prg/arterychip/Davicom/ptp/linuxptp/ptp4l.c) | 時鐘類型驗證與初始化 |
| [`config.c`](file:///d:/prg/arterychip/Davicom/ptp/linuxptp/config.c) | 配置參數定義與解析 |
| [`msg.h`](file:///d:/prg/arterychip/Davicom/ptp/linuxptp/msg.h) | one-step 判斷與訊息結構 |

### 10.3 關鍵函數

| 函數 | 位置 | 功能 |
|-----|------|------|
| `e2e_event()` | `e2e_tc.c:81` | E2E-TC 事件處理 |
| `tc_fwd_sync()` | `tc.c:446` | SYNC 訊息轉發與 one-step 處理 |
| `tc_fwd_event()` | `tc.c:268` | 事件訊息轉發與時間戳收集 |
| `tc_complete_syfup()` | `tc.c:179` | SYNC/FOLLOW_UP 配對與 correction 更新 |
| `one_step()` | `msg.h:473` | 判斷訊息是否為 one-step 模式 |

---

## 參考資料

- IEEE 1588-2019: Precision Time Protocol (PTP) 標準
- linuxptp 原始碼: https://github.com/richardcochran/linuxptp
- [`configs/E2E-TC.cfg`](file:///d:/prg/arterychip/Davicom/ptp/linuxptp/configs/E2E-TC.cfg): E2E-TC 範例配置
- [`configs/default.cfg`](file:///d:/prg/arterychip/Davicom/ptp/linuxptp/configs/default.cfg): 預設配置參數
