# LinuxPTP E2E Transparent Clock - One-Step UDPv4 技術指南

本文檔詳細分析 linuxptp-4.2 專案中如何啟用 **End-to-End Transparent Clock (E2E-TC)**、**One-Step 模式** 和 **UDPv4 網路傳輸**。

---

## 目錄

1. [快速配置範例](#1-快速配置範例)
2. [配置選項詳解](#2-配置選項詳解)
3. [核心代碼架構](#3-核心代碼架構)
4. [E2E-TC 運作流程](#4-e2e-tc-運作流程)
5. [One-Step 模式實現](#5-one-step-模式實現)
6. [UDPv4 傳輸層實現](#6-udpv4-傳輸層實現)
7. [命令行啟動方式](#7-命令行啟動方式)

---

## 1. 快速配置範例

### 完整配置檔案 (e2e-tc-onestep-udpv4.cfg)

```ini
[global]
# === Clock Type 設定 ===
clock_type          E2E_TC       # End-to-End Transparent Clock

# === Time Stamping 設定 ===
time_stamping       onestep      # One-step 硬體時間戳記
twoStepFlag         0            # 設為 0 啟用 one-step

# === Network Transport 設定 ===
network_transport   UDPv4        # UDP over IPv4

# === Delay Mechanism 設定 ===
delay_mechanism     E2E          # E2E-TC 必須使用 E2E

# === TC 特定選項 ===
tc_spanning_tree    1            # 啟用 spanning tree 過濾
free_running        1            # TC 不需要 servo 調整本地時鐘

# === 其他建議設定 ===
priority1           254          # 較低優先級 (TC 不參與 BMCA)
summary_interval    1            # 日誌輸出間隔
logging_level       6            # LOG_INFO 級別

# === 介面設定範例 ===
[eth0]
# 第一個網路介面

[eth1]
# 第二個網路介面 (TC 至少需要 2 個介面)
```

---

## 2. 配置選項詳解

### 2.1 clock_type 選項

**位置**: `config.c` (第 151-157 行)

```c
static struct config_enum clock_type_enu[] = {
    { "OC",      CLOCK_TYPE_ORDINARY },   // 0x8000 - Ordinary Clock
    { "BC",      CLOCK_TYPE_BOUNDARY },   // 0x4000 - Boundary Clock
    { "P2P_TC",  CLOCK_TYPE_P2P      },   // 0x2000 - P2P Transparent Clock
    { "E2E_TC",  CLOCK_TYPE_E2E      },   // 0x1000 - E2E Transparent Clock
    { NULL, 0 },
};
```

**類型定義**: `clock.h` (第 37-43 行)

```c
enum clock_type {
    CLOCK_TYPE_ORDINARY   = 0x8000,
    CLOCK_TYPE_BOUNDARY   = 0x4000,
    CLOCK_TYPE_P2P        = 0x2000,
    CLOCK_TYPE_E2E        = 0x1000,
    CLOCK_TYPE_MANAGEMENT = 0x0800,
};
```

| 配置值 | 枚舉值 | 說明 |
|--------|--------|------|
| `OC` | `CLOCK_TYPE_ORDINARY` | Ordinary Clock (單端口) |
| `BC` | `CLOCK_TYPE_BOUNDARY` | Boundary Clock (多端口) |
| `P2P_TC` | `CLOCK_TYPE_P2P` | P2P Transparent Clock |
| `E2E_TC` | `CLOCK_TYPE_E2E` | E2E Transparent Clock |

### 2.2 time_stamping 選項

**位置**: `config.c` (第 207-214 行)

```c
static struct config_enum timestamping_enu[] = {
    { "hardware", TS_HARDWARE  },   // 標準硬體時間戳記
    { "software", TS_SOFTWARE  },   // 軟體時間戳記
    { "legacy",   TS_LEGACY_HW },   // 舊版硬體時間戳記
    { "onestep",  TS_ONESTEP   },   // One-step 硬體時間戳記
    { "p2p1step", TS_P2P1STEP  },   // P2P One-step 時間戳記
    { NULL, 0 },
};
```

**類型定義**: `msg.h` (第 76-82 行)

```c
enum timestamp_type {
    TS_SOFTWARE,   // 軟體時間戳記
    TS_HARDWARE,   // 硬體時間戳記 (two-step)
    TS_LEGACY_HW,  // 舊版硬體時間戳記
    TS_ONESTEP,    // One-step 硬體時間戳記
    TS_P2P1STEP,   // P2P One-step
};
```

### 2.3 network_transport 選項

**位置**: `config.c` (第 200-205 行)

```c
static struct config_enum nw_trans_enu[] = {
    { "L2",    TRANS_IEEE_802_3 },   // IEEE 802.3 (Layer 2)
    { "UDPv4", TRANS_UDP_IPV4   },   // UDP over IPv4
    { "UDPv6", TRANS_UDP_IPV6   },   // UDP over IPv6
    { NULL, 0 },
};
```

**類型定義**: `transport.h` (第 33-41 行)

```c
enum transport_type {
    TRANS_UDS = 0,       // Unix Domain Socket (內部使用)
    TRANS_UDP_IPV4 = 1,  // UDP over IPv4
    TRANS_UDP_IPV6,      // UDP over IPv6
    TRANS_IEEE_802_3,    // IEEE 802.3 (Ethernet)
    TRANS_DEVICENET,
    TRANS_CONTROLNET,
    TRANS_PROFINET,
};
```

### 2.4 delay_mechanism 選項

**位置**: `config.c` (第 171-177 行)

```c
static struct config_enum delay_mech_enu[] = {
    { "Auto", DM_AUTO },          // 自動選擇
    { "E2E",  DM_E2E },           // End-to-End (Delay_Req/Delay_Resp)
    { "P2P",  DM_P2P },           // Peer-to-Peer (Pdelay_Req/Pdelay_Resp)
    { "NONE", DM_NO_MECHANISM },  // 無延遲測量
    { NULL, 0 },
};
```

---

## 3. 核心代碼架構

### 3.1 檔案結構

```
linuxptp-4.2/
├── tc.c / tc.h           # Transparent Clock 核心邏輯
├── e2e_tc.c              # E2E-TC 事件處理
├── p2p_tc.c              # P2P-TC 事件處理
├── port.c / port.h       # 端口管理和消息處理
├── clock.c / clock.h     # 時鐘管理
├── config.c / config.h   # 配置解析
├── transport.c           # 傳輸層抽象
├── udp.c / udp.h         # UDPv4 傳輸實現
├── msg.c / msg.h         # PTP 消息處理
└── ptp4l.c               # 主程序入口
```

### 3.2 E2E-TC 初始化流程

**位置**: `ptp4l.c` (第 220-240 行)

```c
type = config_get_int(cfg, NULL, "clock_type");
switch (type) {
    // ...
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
}
```

### 3.3 端口事件處理器綁定

**位置**: `port.c` (第 3300-3315 行)

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
    p->dispatch = e2e_dispatch;   // E2E-TC 狀態機
    p->event = e2e_event;         // E2E-TC 事件處理
    break;
}
```

---

## 4. E2E-TC 運作流程

### 4.1 消息轉發函數

**位置**: `tc.h` (第 28-92 行)

| 函數 | 用途 |
|------|------|
| `tc_fwd_sync()` | 轉發 Sync 消息，計算駐留時間 |
| `tc_fwd_folup()` | 轉發 Follow_Up，修正 correctionField |
| `tc_fwd_request()` | 轉發 Delay_Request，記錄駐留時間 |
| `tc_fwd_response()` | 轉發 Delay_Response，修正 correctionField |
| `tc_forward()` | 轉發一般消息 (Announce, Signaling, Management) |
| `tc_ignore()` | 判斷是否忽略消息 |
| `tc_prune()` | 清理過期的駐留時間記錄 |

### 4.2 E2E-TC 事件處理

**位置**: `e2e_tc.c` (第 76-217 行)

```c
enum fsm_event e2e_event(struct port *p, int fd_index)
{
    // ... 接收消息 ...

    switch (msg_type(msg)) {
    case SYNC:
        if (tc_fwd_sync(p, msg)) {        // 轉發 Sync
            event = EV_FAULT_DETECTED;
        }
        if (dup) {
            process_sync(p, dup);          // 處理本地副本
        }
        break;

    case DELAY_REQ:
        if (tc_fwd_request(p, msg)) {     // 轉發 Delay_Request
            event = EV_FAULT_DETECTED;
        }
        break;

    case FOLLOW_UP:
        if (tc_fwd_folup(p, msg)) {       // 轉發 Follow_Up
            event = EV_FAULT_DETECTED;
        }
        if (dup) {
            process_follow_up(p, dup);     // 處理本地副本
        }
        break;

    case DELAY_RESP:
        if (tc_fwd_response(p, msg)) {    // 轉發 Delay_Response
            event = EV_FAULT_DETECTED;
        }
        if (dup) {
            process_delay_resp(p, dup);    // 處理本地副本
        }
        break;

    case ANNOUNCE:
        if (tc_forward(p, msg)) {         // 轉發 Announce
            event = EV_FAULT_DETECTED;
        }
        if (dup && process_announce(p, dup)) {
            event = EV_STATE_DECISION_EVENT;
        }
        break;
    }
    // ...
}
```

### 4.3 駐留時間計算

**位置**: `tc.c` (第 249-285 行)

```c
static int tc_fwd_event(struct port *q, struct ptp_message *msg)
{
    tmv_t egress, ingress = msg->hwts.ts, residence;

    // 發送事件消息到所有其他端口
    for (p = clock_first_port(q->clock); p; p = LIST_NEXT(p, list)) {
        if (tc_blocked(q, p, msg)) {
            continue;
        }
        cnt = transport_send(p->trp, &p->fda, TRANS_DEFER_EVENT, msg);
    }

    // 收集發送時間戳並計算駐留時間
    for (p = clock_first_port(q->clock); p; p = LIST_NEXT(p, list)) {
        if (tc_blocked(q, p, msg)) {
            continue;
        }
        err = transport_txts(&p->fda, msg);

        ts_add(&msg->hwts.ts, p->tx_timestamp_offset);
        egress = msg->hwts.ts;

        // 駐留時間 = 發送時間 - 接收時間
        residence = tmv_sub(egress, ingress);

        // 頻率補償
        rr = clock_rate_ratio(q->clock);
        if (rr != 1.0) {
            residence = dbl_tmv(tmv_dbl(residence) * rr);
        }

        tc_complete(q, p, msg, residence);
    }
    return 0;
}
```

### 4.4 CorrectionField 修正

**位置**: `tc.c` (第 192-230 行)

```c
static void tc_complete_syfup(struct port *q, struct port *p,
                              struct ptp_message *msg, tmv_t residence)
{
    // ... 匹配 Sync/Follow_Up 對 ...

    // 修正 correctionField
    c1 = net2host64(fup->header.correction);
    c2 = c1 + tmv_to_TimeInterval(residence);  // 加上駐留時間
    c2 += tmv_to_TimeInterval(q->peer_delay);  // P2P 時加上對等延遲
    c2 += q->asymmetry;                         // 加上路徑不對稱補償
    fup->header.correction = host2net64(c2);

    // 發送修正後的 Follow_Up
    cnt = transport_send(p->trp, &p->fda, TRANS_GENERAL, fup);

    // 恢復原始值供下一個出口端口使用
    fup->header.correction = host2net64(c1);
}
```

---

## 5. One-Step 模式實現

### 5.1 One-Step 判斷函數

**位置**: `msg.h` (第 458-463 行)

```c
static inline Boolean one_step(struct ptp_message *m)
{
    if (assume_two_step)
        return 0;
    return !field_is_set(m, 0, TWO_STEP);  // TWO_STEP = (1<<1)
}
```

### 5.2 配置協調函數

**位置**: `config.c` (第 1045-1081 行)

```c
int config_harmonize_onestep(struct config *cfg)
{
    enum timestamp_type tstype = config_get_int(cfg, NULL, "time_stamping");
    int two_step_flag = config_get_int(cfg, NULL, "twoStepFlag");

    switch (tstype) {
    case TS_SOFTWARE:
    case TS_LEGACY_HW:
        // 軟體時間戳只能用 two-step
        if (!two_step_flag) {
            pr_err("one step is only possible with hardware time stamping");
            return -1;
        }
        break;

    case TS_HARDWARE:
        // 硬體時間戳 + twoStepFlag=0 => 自動升級到 TS_ONESTEP
        if (!two_step_flag) {
            pr_debug("upgrading to one step time stamping");
            if (config_set_int(cfg, "time_stamping", TS_ONESTEP)) {
                return -1;
            }
        }
        break;

    case TS_ONESTEP:
    case TS_P2P1STEP:
        // one-step 模式需要 twoStepFlag=0
        if (two_step_flag) {
            pr_debug("one step mode implies twoStepFlag=0");
            if (config_set_int(cfg, "twoStepFlag", 0)) {
                return -1;
            }
        }
        break;
    }
    return 0;
}
```

### 5.3 TC 對 One-Step Sync 的處理

**位置**: `tc.c` (第 398-433 行)

```c
int tc_fwd_sync(struct port *q, struct ptp_message *msg)
{
    struct ptp_message *fup = NULL;
    int err;

    // 如果收到的是 one-step Sync，自動生成 Follow_Up
    if (one_step(msg)) {
        fup = msg_allocate();
        if (!fup) {
            return -1;
        }
        // 構建 Follow_Up 消息
        fup->header.tsmt               = FOLLOW_UP | (msg->header.tsmt & 0xf0);
        fup->header.ver                = msg->header.ver;
        fup->header.messageLength      = htons(sizeof(struct follow_up_msg));
        fup->header.domainNumber       = msg->header.domainNumber;
        fup->header.sourcePortIdentity = msg->header.sourcePortIdentity;
        fup->header.sequenceId         = msg->header.sequenceId;
        fup->header.logMessageInterval = msg->header.logMessageInterval;

        // 將 Sync 的 originTimestamp 複製到 Follow_Up
        fup->follow_up.preciseOriginTimestamp = msg->sync.originTimestamp;

        // 將 Sync 標記為 two-step (設定 TWO_STEP 旗標)
        msg->header.flagField[0]      |= TWO_STEP;
    }

    // 轉發 Sync (作為事件消息)
    err = tc_fwd_event(q, msg);
    if (err) {
        return err;
    }

    // 如果生成了 Follow_Up，則轉發它
    if (fup) {
        err = tc_fwd_folup(q, fup);
        msg_put(fup);
    }
    return err;
}
```

### 5.4 硬體時間戳初始化

**位置**: `sk.c` (第 565-615 行)

```c
int sk_timestamping_init(int fd, const char *device, enum timestamp_type type,
                         enum transport_type transport, int vclock)
{
    int tx_type = HWTSTAMP_TX_ON;

    switch (type) {
    case TS_SOFTWARE:
        tx_type = HWTSTAMP_TX_OFF;
        break;
    case TS_HARDWARE:
    case TS_LEGACY_HW:
        tx_type = HWTSTAMP_TX_ON;
        break;
    case TS_ONESTEP:
        tx_type = HWTSTAMP_TX_ONESTEP_SYNC;   // One-step 發送 Sync
        break;
    case TS_P2P1STEP:
        tx_type = HWTSTAMP_TX_ONESTEP_P2P;    // P2P One-step
        break;
    }
    // ... 設定硬體時間戳過濾器 ...
}
```

---

## 6. UDPv4 傳輸層實現

### 6.1 網路參數定義

**位置**: `udp.c` (第 39-42 行)

```c
#define EVENT_PORT        319                    // 事件消息端口
#define GENERAL_PORT      320                    // 一般消息端口
#define PTP_PRIMARY_MCAST_IPADDR "224.0.1.129"   // PTP 主多播地址
#define PTP_PDELAY_MCAST_IPADDR  "224.0.0.107"   // P2P 延遲多播地址
```

### 6.2 Socket 初始化

**位置**: `udp.c` (第 83-152 行)

```c
static int open_socket(const char *name, struct in_addr mc_addr[2],
                       short port, int ttl)
{
    struct sockaddr_in addr;
    int fd, index, on = 1;

    // 創建 UDP socket
    fd = socket(PF_INET, SOCK_DGRAM, IPPROTO_UDP);

    // 設定 socket 選項
    setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on));

    // 綁定到任意地址和指定端口
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    addr.sin_port = htons(port);
    bind(fd, (struct sockaddr *) &addr, sizeof(addr));

    // 綁定到指定網路介面
    setsockopt(fd, SOL_SOCKET, SO_BINDTODEVICE, name, strlen(name));

    // 設定多播 TTL
    setsockopt(fd, IPPROTO_IP, IP_MULTICAST_TTL, &ttl, sizeof(ttl));

    // 加入多播群組
    addr.sin_addr = mc_addr[0];  // PTP_PRIMARY_MCAST_IPADDR
    mcast_join(fd, index, &addr);

    addr.sin_addr = mc_addr[1];  // PTP_PDELAY_MCAST_IPADDR
    mcast_join(fd, index, &addr);

    return fd;
}
```

### 6.3 傳輸層創建

**位置**: `transport.c`

```c
struct transport *transport_create(struct config *cfg,
                                   enum transport_type type)
{
    switch (type) {
    case TRANS_UDP_IPV4:
        return udp_transport_create();    // 創建 UDPv4 傳輸
    case TRANS_UDP_IPV6:
        return udp6_transport_create();   // 創建 UDPv6 傳輸
    case TRANS_IEEE_802_3:
        return raw_transport_create();    // 創建 Raw Ethernet 傳輸
    // ...
    }
}
```

---

## 7. 命令行啟動方式

### 7.1 使用配置檔案

```bash
# 建立配置檔案後啟動
sudo ptp4l -f e2e-tc-onestep-udpv4.cfg -i eth0 -i eth1
```

### 7.2 純命令行參數

```bash
# 最小化命令 (使用預設 UDPv4 和 E2E)
sudo ptp4l -i eth0 -i eth1 \
    --clock_type E2E_TC \
    --time_stamping onestep \
    --twoStepFlag 0 \
    --network_transport UDPv4 \
    --delay_mechanism E2E \
    --tc_spanning_tree 1 \
    --free_running 1
```

### 7.3 混合方式

```bash
# 配置檔 + 覆蓋參數
sudo ptp4l -f /etc/linuxptp/default.cfg \
    -i eth0 -i eth1 \
    --clock_type E2E_TC \
    --time_stamping onestep \
    -m  # 輸出到 stdout
```

### 7.4 常用命令行選項對照表

| 選項 | 配置參數 | 說明 |
|------|----------|------|
| `-4` | `network_transport UDPv4` | 使用 UDP/IPv4 |
| `-6` | `network_transport UDPv6` | 使用 UDP/IPv6 |
| `-2` | `network_transport L2` | 使用 IEEE 802.3 |
| `-E` | `delay_mechanism E2E` | E2E 延遲機制 |
| `-P` | `delay_mechanism P2P` | P2P 延遲機制 |
| `-A` | `delay_mechanism Auto` | 自動延遲機制 |
| `-H` | `time_stamping hardware` | 硬體時間戳 |
| `-S` | `time_stamping software` | 軟體時間戳 |
| `-i` | N/A | 指定網路介面 |
| `-f` | N/A | 指定配置檔案 |
| `-m` | `verbose 1` | 輸出到 stdout |
| `-l` | `logging_level` | 設定日誌級別 |

---

## 附錄 A: 配置參數快速參考

### E2E-TC 必要配置

| 參數 | 值 | 必要性 |
|------|-----|--------|
| `clock_type` | `E2E_TC` | **必須** |
| `delay_mechanism` | `E2E` | **必須** |
| 網路介面數量 | ≥ 2 | **必須** |

### One-Step 配置 (擇一)

| 方式 | 配置 |
|------|------|
| 方式 1 | `time_stamping onestep` |
| 方式 2 | `time_stamping hardware` + `twoStepFlag 0` |

### UDPv4 配置

| 參數 | 預設值 | 說明 |
|------|--------|------|
| `network_transport` | `UDPv4` | UDP/IPv4 (預設值) |
| `udp_ttl` | `1` | 多播 TTL |
| `dscp_event` | `0` | 事件消息 DSCP |
| `dscp_general` | `0` | 一般消息 DSCP |

---

## 附錄 B: 消息流程圖

```
      Master                  TC (E2E-TC)               Slave
         |                        |                        |
         |-- Sync (one-step) ---->|                        |
         |                        |-- Sync (two-step) ---->|
         |                        |-- Follow_Up ---------->|
         |                        |   (自動生成,含駐留時間)   |
         |                        |                        |
         |                        |<----- Delay_Req -------|
         |<---- Delay_Req --------|                        |
         |                        |                        |
         |---- Delay_Resp ------->|                        |
         |                        |---- Delay_Resp ------->|
         |                        |   (修正 correctionField) |
         |                        |                        |
```

---

*文件生成日期: 2024*
*適用版本: linuxptp-4.2*
