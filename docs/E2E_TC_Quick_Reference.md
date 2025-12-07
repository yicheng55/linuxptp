# E2E-TC One-Step UDPv4 快速參考

## 快速配置

### 最小配置檔案 (e2e_tc_onestep.cfg)

```ini
[global]
clock_type              E2E_TC
twoStepFlag             0
network_transport       UDPv4
delay_mechanism         E2E
time_stamping           hardware
tc_spanning_tree        1
free_running            1
priority1               254

[eth0]
[eth1]
```

### 執行命令

```bash
# 方法 1: 使用配置檔案
sudo ptp4l -f e2e_tc_onestep.cfg -i eth0 -i eth1 -m

# 方法 2: 純命令列
sudo ptp4l -E -4 -H -i eth0 -i eth1 \
    --clock_type E2E_TC \
    --twoStepFlag 0 \
    --tc_spanning_tree 1 \
    -m
```

---

## 配置參數對照表

| 參數 | 值 | 說明 |
|------|-----|------|
| `clock_type` | `E2E_TC` | End-to-End Transparent Clock |
| `twoStepFlag` | `0` | One-Step 模式 |
| `network_transport` | `UDPv4` | IPv4 UDP 傳輸 |
| `delay_mechanism` | `E2E` | E2E 延遲機制 (必須) |
| `time_stamping` | `hardware` | 硬體時間戳 |
| `tc_spanning_tree` | `1` | 啟用生成樹協議 |

---

## 關鍵代碼路徑

### 1. 配置解析流程

```
ptp4l.c:main()
  └─> config_get_int(cfg, NULL, "clock_type")  // 讀取 E2E_TC
      └─> 驗證: n_interfaces >= 2
      └─> 驗證: delay_mechanism == E2E
          └─> clock_create(CLOCK_TYPE_E2E, cfg)
```

**檔案位置**: `ptp4l.c` 第 209-244 行

### 2. 端口初始化

```
port_open()  // port.c:3283
  └─> switch (type) {
      case CLOCK_TYPE_E2E:
          p->dispatch = e2e_dispatch;  // e2e_tc.c:28
          p->event = e2e_event;        // e2e_tc.c:76
      }
```

**檔案位置**: `port.c` 第 3311-3313 行

### 3. 訊息處理流程

```
e2e_event()  // e2e_tc.c:76
  └─> switch (msg_type(msg)) {
      case SYNC:
          tc_fwd_sync(p, msg)      // tc.c:441
            └─> if (one_step(msg))  // msg.h:459
                └─> 創建 Follow_Up
                └─> 設置 TWO_STEP flag
            └─> tc_fwd_event(q, msg)  // tc.c:267
                └─> 轉發到所有端口
                └─> 收集 TX 時間戳
                └─> 計算 residence time
                └─> tc_complete()  // tc.c:238
                    └─> correction += residence

      case FOLLOW_UP:
          tc_fwd_folup(p, msg)  // tc.c:409

      case DELAY_REQ:
          tc_fwd_request(p, msg)  // tc.c:432

      case DELAY_RESP:
          tc_fwd_response(p, msg)  // tc.c:436
      }
```

---

## 核心函數說明

### tc_fwd_sync (tc.c:441)

**功能**: 轉發 Sync 訊息

```c
int tc_fwd_sync(struct port *q, struct ptp_message *msg)
{
    struct ptp_message *fup = NULL;

    // 處理 one-step Sync
    if (one_step(msg)) {
        fup = msg_allocate();
        // 複製 Sync 資訊到 Follow_Up
        fup->header.tsmt = FOLLOW_UP | (msg->header.tsmt & 0xf0);
        fup->follow_up.preciseOriginTimestamp = msg->sync.originTimestamp;
        // 將 Sync 轉為 two-step
        msg->header.flagField[0] |= TWO_STEP;
    }

    tc_fwd_event(q, msg);  // 轉發 Sync
    if (fup) {
        tc_fwd_folup(q, fup);  // 轉發 Follow_Up
    }
    return 0;
}
```

**重點**: One-Step Sync 會被 TC 轉換為 Two-Step 處理

### tc_fwd_event (tc.c:267)

**功能**: 轉發事件訊息並計算駐留時間

```c
static int tc_fwd_event(struct port *q, struct ptp_message *msg)
{
    tmv_t ingress = msg->hwts.ts;  // 入口時間戳

    // 步驟 1: 轉發到所有端口
    for (p = clock_first_port(q->clock); p; p = LIST_NEXT(p, list)) {
        if (!tc_blocked(q, p, msg)) {
            transport_send(p->trp, &p->fda, TRANS_DEFER_EVENT, msg);
        }
    }

    // 步驟 2: 收集 TX 時間戳並計算駐留時間
    for (p = clock_first_port(q->clock); p; p = LIST_NEXT(p, list)) {
        if (!tc_blocked(q, p, msg)) {
            transport_txts(&p->fda, msg);  // 取得 TX 時間戳
            egress = msg->hwts.ts;         // 出口時間戳
            residence = tmv_sub(egress, ingress);  // 駐留時間
            tc_complete(q, p, msg, residence);  // 加入 correction
        }
    }
    return 0;
}
```

**重點**: 駐留時間 = 出口時間戳 - 入口時間戳

### tc_complete_syfup (tc.c:177)

**功能**: 將駐留時間加入 Follow_Up 的 correction field

```c
static void tc_complete_syfup(struct port *q, struct port *p,
                              struct ptp_message *msg, tmv_t residence)
{
    // 匹配 Sync 和 Follow_Up...

    // 更新 correction field
    c1 = net2host64(fup->header.correction);
    c2 = c1 + tmv_to_TimeInterval(residence);  // 加入駐留時間
    c2 += tmv_to_TimeInterval(q->peer_delay);  // 加入對等延遲
    c2 += q->asymmetry;                        // 加入不對稱補償
    fup->header.correction = host2net64(c2);

    // 傳送 Follow_Up
    transport_send(p->trp, &p->fda, TRANS_GENERAL, fup);

    // 恢復原始值供其他端口使用
    fup->header.correction = host2net64(c1);
}
```

**重點**: Correction = 原始值 + 駐留時間 + 對等延遲 + 不對稱補償

### one_step 判斷 (msg.h:459)

```c
static inline Boolean one_step(struct ptp_message *m)
{
    if (assume_two_step)
        return 0;
    return !field_is_set(m, 0, TWO_STEP);
}
```

**重點**: TWO_STEP flag 為 0 時，代表 one-step 訊息

---

## 資料結構

### enum clock_type (clock.h:36)

```c
enum clock_type {
    CLOCK_TYPE_ORDINARY   = 0x8000,
    CLOCK_TYPE_BOUNDARY   = 0x4000,
    CLOCK_TYPE_P2P        = 0x2000,
    CLOCK_TYPE_E2E        = 0x1000,  // E2E_TC
    CLOCK_TYPE_MANAGEMENT = 0x0800,
};
```

### enum transport_type (transport.h:33)

```c
enum transport_type {
    TRANS_UDS = 0,
    TRANS_UDP_IPV4 = 1,  // UDPv4
    TRANS_UDP_IPV6,
    TRANS_IEEE_802_3,
};
```

### enum transport_event (transport.h:48)

```c
enum transport_event {
    TRANS_GENERAL,      // 一般訊息 (Follow_Up, Delay_Resp)
    TRANS_EVENT,        // 事件訊息 (Sync, Delay_Req)
    TRANS_ONESTEP,      // One-Step 訊息
    TRANS_P2P1STEP,     // P2P One-Step 訊息
    TRANS_DEFER_EVENT,  // 延遲的事件訊息 (用於 TC)
};
```

---

## 配置驗證

### config_harmonize_onestep (config.c:1047)

**功能**: 協調 twoStepFlag 與 time_stamping 設定

```c
int config_harmonize_onestep(struct config *cfg)
{
    enum timestamp_type tstype = config_get_int(cfg, NULL, "time_stamping");
    int two_step_flag = config_get_int(cfg, NULL, "twoStepFlag");

    switch (tstype) {
    case TS_HARDWARE:
        if (!two_step_flag) {
            // twoStepFlag=0 → 升級為 onestep
            config_set_int(cfg, "time_stamping", TS_ONESTEP);
        }
        break;
    case TS_ONESTEP:
    case TS_P2P1STEP:
        if (two_step_flag) {
            // onestep → 清除 twoStepFlag
            config_set_int(cfg, "twoStepFlag", 0);
        }
        break;
    }
    return 0;
}
```

**重點**: `twoStepFlag=0` 會自動將 `time_stamping` 設為 `onestep`

---

## 常見問題

### Q1: 為什麼需要至少兩個介面？

A: Transparent Clock 的目的是在不同網路段之間轉發 PTP 訊息。至少需要：
- 1 個入口介面 (ingress)
- 1 個出口介面 (egress)

驗證代碼: `ptp4l.c:233-236`

### Q2: E2E_TC 一定要用 E2E delay_mechanism 嗎？

A: 是的。E2E_TC 與 P2P_TC 的區別在於使用不同的延遲測量機制：
- E2E_TC → E2E (Delay_Req/Delay_Resp)
- P2P_TC → P2P (Pdelay_Req/Pdelay_Resp)

驗證代碼: `ptp4l.c:237-240`

### Q3: One-Step 訊息如何被 TC 處理？

A: TC 會將 one-step Sync 轉換為 two-step：
1. 創建對應的 Follow_Up 訊息
2. 在 Sync 上設置 TWO_STEP flag
3. 在 Follow_Up 中加入駐留時間

代碼位置: `tc.c:447-461`

### Q4: 如何驗證 TC 正常運作？

A: 檢查日誌輸出：
```
ptp4l[xxx]: port 1 (eth0): INITIALIZING to LISTENING
ptp4l[xxx]: port 2 (eth1): INITIALIZING to LISTENING
```

使用 Wireshark 抓包檢查：
- Follow_Up 的 correction field 應該包含駐留時間
- One-Step Sync 應該被轉換為 Two-Step

---

## 測試配置

### 測試環境

```
[Master] ---- [E2E-TC] ---- [Slave]
         eth0          eth1
```

### Master 配置

```ini
[global]
clock_type              OC
twoStepFlag             0
network_transport       UDPv4
delay_mechanism         E2E
time_stamping           hardware
clientOnly              0
serverOnly              1
```

### Slave 配置

```ini
[global]
clock_type              OC
twoStepFlag             1
network_transport       UDPv4
delay_mechanism         E2E
time_stamping           hardware
clientOnly              1
serverOnly              0
```

### 執行測試

```bash
# Master
sudo ptp4l -f master.cfg -i eth0 -m

# E2E-TC
sudo ptp4l -f e2e_tc_onestep.cfg -i eth0 -i eth1 -m

# Slave
sudo ptp4l -f slave.cfg -i eth0 -m
```

---

## 除錯技巧

### 啟用詳細日誌

```ini
logging_level           7
verbose                 1
```

或使用命令列：

```bash
sudo ptp4l -f config.cfg -i eth0 -i eth1 -l 7 -m
```

### 使用 Wireshark

過濾器：
```
ptp
```

檢查項目：
1. Sync 訊息的 TWO_STEP flag
2. Follow_Up 的 correction field
3. Delay_Req/Delay_Resp 對

### 檢查網卡時間戳支援

```bash
ethtool -T eth0
```

應該包含：
```
Capabilities:
    hardware-transmit
    hardware-receive
    hardware-raw-clock
```

---

## 參考檔案

| 檔案 | 主要功能 |
|------|---------|
| `e2e_tc.c` | E2E TC 事件處理 |
| `tc.c` | TC 核心轉發邏輯 |
| `tc.h` | TC 函數宣告 |
| `port.c` | 端口管理與初始化 |
| `clock.c` | 時鐘創建與管理 |
| `config.c` | 配置解析與驗證 |
| `ptp4l.c` | 主程式入口 |
| `msg.h` | PTP 訊息處理 |
| `transport.h` | 傳輸層定義 |

詳細說明請參考: `E2E_TC_ONESTEP_UDPv4_Guide.md`
