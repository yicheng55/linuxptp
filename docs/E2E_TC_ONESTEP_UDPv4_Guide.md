# LinuxPTP E2E Transparent Clock (E2E-TC) 啟用指南

## 目錄
1. [概述](#概述)
2. [配置方法](#配置方法)
3. [配置檔案範例](#配置檔案範例)
4. [關鍵參數說明](#關鍵參數說明)
5. [代碼架構分析](#代碼架構分析)
6. [執行命令](#執行命令)

---

## 概述

### 什麼是 E2E Transparent Clock？

E2E Transparent Clock (E2E-TC) 是 IEEE 1588 PTP 協議中的一種時鐘類型，它不參與主從時鐘的選舉，而是：
- **轉發 PTP 訊息**到所有其他端口
- **計算駐留時間 (Residence Time)** 並將其累加到 correction field
- 使用 **End-to-End (E2E) 延遲機制**，即 Delay_Req/Delay_Resp

### 配置目標

本指南說明如何配置：
- **Clock Type**: E2E_TC (End-to-End Transparent Clock)
- **Timestamping**: One-Step (單步時間戳)
- **Network Transport**: UDPv4 (IPv4 UDP)

---

## 配置方法

### 方式一：使用配置檔案

創建或修改配置檔案，設定以下關鍵參數：

```ini
[global]
# 設定時鐘類型為 E2E Transparent Clock
clock_type              E2E_TC

# 設定為 one-step 模式 (twoStepFlag = 0)
twoStepFlag             0

# 設定網路傳輸為 UDPv4
network_transport       UDPv4

# 設定延遲機制為 E2E (必須與 E2E_TC 匹配)
delay_mechanism         E2E

# 使用硬體時間戳
time_stamping           hardware
```

### 方式二：使用命令列參數

```bash
ptp4l -E -4 -H -i eth0 -i eth1 -f /path/to/config.cfg
```

命令列參數說明：
- `-E`: 設定 delay_mechanism 為 E2E
- `-4`: 設定 network_transport 為 UDPv4
- `-H`: 設定 time_stamping 為 hardware
- `-i eth0 -i eth1`: 指定兩個網路介面 (TC 至少需要兩個介面)

---

## 配置檔案範例

### 完整的 E2E-TC + One-Step + UDPv4 配置

```ini
#
# E2E Transparent Clock with One-Step and UDPv4 Configuration
#
[global]
#
# Clock Type Settings
#
clock_type              E2E_TC

#
# One-Step Configuration
# twoStepFlag = 0 啟用 one-step 模式
# time_stamping = onestep 會自動將 twoStepFlag 設為 0
#
twoStepFlag             0
time_stamping           hardware

#
# Network Transport
#
network_transport       UDPv4
delay_mechanism         E2E

#
# Transparent Clock Options
#
tc_spanning_tree        1
free_running            1

#
# Timing Parameters
#
priority1               254
freq_est_interval       3
summary_interval        1

#
# Other Settings
#
logging_level           6
verbose                 1

#
# Interface Sections (至少需要兩個介面)
#
[eth0]

[eth1]
```

---

## 關鍵參數說明

### 1. clock_type

| 值 | 說明 | 內部常數 |
|---|---|---|
| `OC` | Ordinary Clock (普通時鐘) | `CLOCK_TYPE_ORDINARY = 0x8000` |
| `BC` | Boundary Clock (邊界時鐘) | `CLOCK_TYPE_BOUNDARY = 0x4000` |
| `P2P_TC` | P2P Transparent Clock | `CLOCK_TYPE_P2P = 0x2000` |
| `E2E_TC` | E2E Transparent Clock | `CLOCK_TYPE_E2E = 0x1000` |

**位置**: `config.c` 第 152-157 行
```c
static struct config_enum clock_type_enu[] = {
    { "OC",      CLOCK_TYPE_ORDINARY },
    { "BC",      CLOCK_TYPE_BOUNDARY },
    { "P2P_TC",  CLOCK_TYPE_P2P      },
    { "E2E_TC",  CLOCK_TYPE_E2E      },
    { NULL, 0 },
};
```

### 2. twoStepFlag (One-Step vs Two-Step)

| 值 | 模式 | 說明 |
|---|---|---|
| `0` | One-Step | Sync 訊息直接包含精確時間戳，不需要 Follow_Up |
| `1` | Two-Step | 需要 Follow_Up 訊息傳送精確時間戳 |

**重要**: 設定 `twoStepFlag = 0` 時，系統會自動升級 `time_stamping` 為 `onestep` 模式。

**位置**: `config.c` 第 1045-1082 行 (`config_harmonize_onestep` 函數)

### 3. network_transport

| 值 | 說明 | 內部常數 |
|---|---|---|
| `UDPv4` | IPv4 UDP | `TRANS_UDP_IPV4 = 1` |
| `UDPv6` | IPv6 UDP | `TRANS_UDP_IPV6 = 2` |
| `L2` | IEEE 802.3 (Layer 2) | `TRANS_IEEE_802_3 = 3` |

**位置**: `config.c` 第 201-206 行
```c
static struct config_enum nw_trans_enu[] = {
    { "L2",    TRANS_IEEE_802_3 },
    { "UDPv4", TRANS_UDP_IPV4   },
    { "UDPv6", TRANS_UDP_IPV6   },
    { NULL, 0 },
};
```

### 4. delay_mechanism

| 值 | 說明 |
|---|---|
| `E2E` | End-to-End (使用 Delay_Req/Delay_Resp) |
| `P2P` | Peer-to-Peer (使用 Pdelay_Req/Pdelay_Resp) |
| `Auto` | 自動選擇 |

**注意**: `E2E_TC` 必須使用 `E2E` 延遲機制！

### 5. tc_spanning_tree

| 值 | 說明 |
|---|---|
| `0` | 禁用生成樹協議 |
| `1` | 啟用生成樹協議 (推薦) |

啟用時，TC 會根據 BMCA 狀態決定轉發行為，避免迴圈。

---

## 代碼架構分析

### 1. 主要檔案結構

```
linuxptp-4.2/
├── e2e_tc.c        # E2E TC 事件處理與狀態機
├── tc.c            # Transparent Clock 核心邏輯
├── tc.h            # TC 相關函數宣告
├── port.c          # 端口管理，根據 clock_type 選擇處理函數
├── clock.c         # 時鐘創建與管理
├── config.c        # 配置解析與驗證
└── ptp4l.c         # 主程式入口
```

### 2. 時鐘類型判斷流程

**檔案**: `ptp4l.c` 第 209-244 行

```c
type = config_get_int(cfg, NULL, "clock_type");
switch (type) {
case CLOCK_TYPE_ORDINARY:
    if (cfg->n_interfaces > 1) {
        type = CLOCK_TYPE_BOUNDARY;
    }
    break;
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
// ...
}
```

**重點**:
- E2E_TC 需要**至少兩個網路介面**
- 必須使用 **E2E 延遲機制**

### 3. 端口事件處理函數選擇

**檔案**: `port.c` 第 3300-3315 行

```c
switch (type) {
case CLOCK_TYPE_ORDINARY:
case CLOCK_TYPE_BOUNDARY:
    p->dispatch = bc_dispatch;
    p->event = bc_event;
    break;
case CLOCK_TYPE_P2P:
    p->dispatch = p2p_dispatch;
    p->event = p2p_event;
    break;
case CLOCK_TYPE_E2E:
    p->dispatch = e2e_dispatch;
    p->event = e2e_event;
    break;
}
```

E2E_TC 使用 `e2e_dispatch` 和 `e2e_event` 函數處理 PTP 事件。

### 4. E2E TC 事件處理

**檔案**: `e2e_tc.c`

#### e2e_event 函數 (第 76-229 行)

主要處理接收到的 PTP 訊息：

```c
switch (msg_type(msg)) {
case SYNC:
    if (tc_fwd_sync(p, msg)) {
        event = EV_FAULT_DETECTED;
        break;
    }
    if (dup) {
        process_sync(p, dup);
    }
    break;
case DELAY_REQ:
    if (tc_fwd_request(p, msg)) {
        event = EV_FAULT_DETECTED;
    }
    break;
case FOLLOW_UP:
    if (tc_fwd_folup(p, msg)) {
        event = EV_FAULT_DETECTED;
        break;
    }
    if (dup) {
        process_follow_up(p, dup);
    }
    break;
case DELAY_RESP:
    if (tc_fwd_response(p, msg)) {
        event = EV_FAULT_DETECTED;
    }
    if (dup) {
        process_delay_resp(p, dup);
    }
    break;
case ANNOUNCE:
    if (tc_forward(p, msg)) {
        event = EV_FAULT_DETECTED;
        break;
    }
    if (dup && process_announce(p, dup)) {
        event = EV_STATE_DECISION_EVENT;
    }
    break;
// ...
}
```

### 5. TC 核心轉發邏輯

**檔案**: `tc.c`

#### tc_fwd_sync - Sync 訊息轉發 (第 441-471 行)

```c
int tc_fwd_sync(struct port *q, struct ptp_message *msg)
{
    struct ptp_message *fup = NULL;
    int err;

    // 處理 one-step Sync 訊息：轉換為 two-step
    if (one_step(msg)) {
        fup = msg_allocate();
        if (!fup) {
            return -1;
        }
        // 創建 Follow_Up 訊息
        fup->header.tsmt               = FOLLOW_UP | (msg->header.tsmt & 0xf0);
        fup->header.ver                = msg->header.ver;
        fup->header.messageLength      = htons(sizeof(struct follow_up_msg));
        fup->header.domainNumber       = msg->header.domainNumber;
        fup->header.sourcePortIdentity = msg->header.sourcePortIdentity;
        fup->header.sequenceId         = msg->header.sequenceId;
        fup->header.logMessageInterval = msg->header.logMessageInterval;
        fup->follow_up.preciseOriginTimestamp = msg->sync.originTimestamp;
        // 設定 TWO_STEP 標誌
        msg->header.flagField[0]      |= TWO_STEP;
    }

    err = tc_fwd_event(q, msg);
    if (err) {
        return err;
    }
    if (fup) {
        err = tc_fwd_folup(q, fup);
        msg_put(fup);
    }
    return err;
}
```

**重要**: 當收到 one-step Sync 時，TC 會：
1. 創建對應的 Follow_Up 訊息
2. 將 Sync 訊息的 TWO_STEP 標誌設為 1
3. 在 Follow_Up 中加入駐留時間

#### tc_fwd_event - 事件訊息轉發 (第 267-311 行)

```c
static int tc_fwd_event(struct port *q, struct ptp_message *msg)
{
    tmv_t egress, ingress = msg->hwts.ts, residence;
    struct port *p;
    int cnt, err;
    double rr;

    clock_gettime(CLOCK_MONOTONIC, &msg->ts.host);

    // 第一步：轉發事件訊息到所有其他端口
    for (p = clock_first_port(q->clock); p; p = LIST_NEXT(p, list)) {
        if (tc_blocked(q, p, msg)) {
            continue;
        }
        cnt = transport_send(p->trp, &p->fda, TRANS_DEFER_EVENT, msg);
        if (cnt <= 0) {
            pr_err("failed to forward event from %s to %s",
                q->log_name, p->log_name);
            port_dispatch(p, EV_FAULT_DETECTED, 0);
        }
    }

    // 第二步：收集傳輸時間戳並計算駐留時間
    for (p = clock_first_port(q->clock); p; p = LIST_NEXT(p, list)) {
        if (tc_blocked(q, p, msg)) {
            continue;
        }
        err = transport_txts(&p->fda, msg);
        if (err || !msg_sots_valid(msg)) {
            pr_err("failed to fetch txts on %s to %s event",
                q->log_name, p->log_name);
            port_dispatch(p, EV_FAULT_DETECTED, 0);
            continue;
        }
        ts_add(&msg->hwts.ts, p->tx_timestamp_offset);
        egress = msg->hwts.ts;
        // 計算駐留時間 = 出口時間 - 入口時間
        residence = tmv_sub(egress, ingress);
        // 應用頻率比例補償
        rr = clock_rate_ratio(q->clock);
        if (rr != 1.0) {
            residence = dbl_tmv(tmv_dbl(residence) * rr);
        }
        tc_complete(q, p, msg, residence);
    }

    return 0;
}
```

#### tc_complete_syfup - 完成 Sync/Follow_Up 處理 (第 177-236 行)

```c
static void tc_complete_syfup(struct port *q, struct port *p,
                              struct ptp_message *msg, tmv_t residence)
{
    // ... 匹配 Sync 和 Follow_Up ...

    // 將駐留時間加入 correction field
    c1 = net2host64(fup->header.correction);
    c2 = c1 + tmv_to_TimeInterval(residence);
    c2 += tmv_to_TimeInterval(q->peer_delay);  // 加入對等延遲
    c2 += q->asymmetry;                        // 加入不對稱補償
    fup->header.correction = host2net64(c2);

    cnt = transport_send(p->trp, &p->fda, TRANS_GENERAL, fup);
    // ...

    // 恢復原始 correction 值，供其他出口端口使用
    fup->header.correction = host2net64(c1);
}
```

### 6. One-Step 檢測

**檔案**: `msg.h` 第 459-465 行

```c
static inline Boolean one_step(struct ptp_message *m)
{
    if (assume_two_step)
        return 0;
    return !field_is_set(m, 0, TWO_STEP);
}
```

---

## 執行命令

### 基本執行

```bash
# 使用配置檔案
sudo ptp4l -f e2e_tc_onestep_udpv4.cfg -i eth0 -i eth1 -m

# 使用命令列參數
sudo ptp4l -E -4 -H -i eth0 -i eth1 \
    --clock_type E2E_TC \
    --twoStepFlag 0 \
    --tc_spanning_tree 1 \
    -m
```

### 參數說明

| 參數 | 說明 |
|---|---|
| `-f config.cfg` | 指定配置檔案 |
| `-i eth0 -i eth1` | 指定網路介面 (至少兩個) |
| `-m` | 輸出訊息到 stdout |
| `-E` | 使用 E2E 延遲機制 |
| `-4` | 使用 UDPv4 傳輸 |
| `-H` | 使用硬體時間戳 |

### 驗證執行

成功執行後，應該看到類似輸出：

```
ptp4l[12345.678]: port 1 (eth0): INITIALIZING to LISTENING
ptp4l[12345.678]: port 2 (eth1): INITIALIZING to LISTENING
```

---

## 注意事項

1. **介面數量**: E2E_TC 至少需要兩個網路介面
2. **延遲機制匹配**: E2E_TC 必須使用 E2E delay_mechanism
3. **硬體支援**: One-step 模式需要網卡支援硬體時間戳
4. **權限**: 需要 root 權限執行 ptp4l
5. **時間戳模式**: 設定 `twoStepFlag=0` 時，系統會自動升級為 onestep 時間戳模式

---

## 參考資料

- `configs/E2E-TC.cfg` - 官方 E2E-TC 配置範例
- `configs/default.cfg` - 預設配置參數
- `ptp4l.8` - ptp4l 手冊頁
- IEEE 1588-2019 標準文件
