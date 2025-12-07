# E2E Transparent Clock One-Step UDPv4 配置與代碼分析

## 概述

本文檔詳細說明如何在 linuxptp-4.2 專案中啟用 **E2E Transparent Clock (E2E-TC)** 的 **one-step** 模式，並使用 **UDPv4** 作為網路傳輸協定。

---

## 一、配置參數設定

要啟用 E2E-TC one-step UDPv4 模式，需要在配置檔中設定以下三個關鍵參數：

### 1.1 clock_type = E2E_TC

**配置項：**
```ini
clock_type  E2E_TC
```

**代碼定義位置：** `config.c` 第 152-157 行
```c
static struct config_enum clock_type_enu[] = {
	{ "OC",      CLOCK_TYPE_ORDINARY },
	{ "BC",      CLOCK_TYPE_BOUNDARY },
	{ "P2P_TC",  CLOCK_TYPE_P2P      },
	{ "E2E_TC",  CLOCK_TYPE_E2E      },
	{ NULL, 0 },
};
```

**clock.h 中的枚舉定義：** 第 38-43 行
```c
enum clock_type {
	CLOCK_TYPE_ORDINARY   = 0x8000,
	CLOCK_TYPE_BOUNDARY   = 0x4000,
	CLOCK_TYPE_P2P        = 0x2000,
	CLOCK_TYPE_E2E        = 0x1000,
	CLOCK_TYPE_MANAGEMENT = 0x0800,
};
```

**說明：**
- `CLOCK_TYPE_E2E` (0x1000) 表示這是一個 End-to-End Transparent Clock
- 與 P2P_TC 不同，E2E_TC 處理 Delay_Req/Delay_Resp 消息

### 1.2 time_stamping = TS_ONESTEP

**配置項：**
```ini
time_stamping  hardware
```
或直接在代碼中設定為 `TS_ONESTEP`

**代碼定義位置：** `config.c` 第 341 行
```c
GLOB_ITEM_ENU("time_stamping", TS_ONESTEP, timestamping_enu),
```

**msg.h 中的枚舉定義：** 第 75-80 行
```c
enum timestamp_type {
	TS_SOFTWARE,
	TS_HARDWARE,
	TS_LEGACY_HW,
	TS_ONESTEP,
	TS_P2P1STEP,
};
```

**說明：**
- `TS_ONESTEP` 模式表示硬體會在 Sync 消息發送時自動填入時間戳
- 硬體會將 Sync 和 Follow_Up 合併處理，但實際上會設定 TWO_STEP flag 並發送 Follow_Up

### 1.3 network_transport = UDPv4

**配置項：**
```ini
network_transport  UDPv4
```

**代碼定義位置：** `config.c` 第 298 行
```c
PORT_ITEM_ENU("network_transport", TRANS_UDP_IPV4, nw_trans_enu),
```

**transport.h 中的枚舉定義：** 第 34-42 行
```c
enum transport_type {
	TRANS_UDS = 0,
	TRANS_UDP_IPV4 = 1,
	TRANS_UDP_IPV6,
	TRANS_IEEE_802_3,
	TRANS_DEVICENET,
	TRANS_CONTROLNET,
	TRANS_PROFINET,
};
```

**說明：**
- `TRANS_UDP_IPV4` (值為 1) 表示使用 UDP over IPv4 傳輸
- 預設配置檔 `configs/E2E-TC.cfg` 使用 L2 (TRANS_IEEE_802_3)，需要修改為 UDPv4

---

## 二、核心代碼流程分析

### 2.1 E2E-TC 事件處理 (e2e_tc.c)

#### 消息接收與分發
**檔案：** `e2e_tc.c` 第 169-200 行

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
		// 處理延遲響應
	}
	break;
}
```

**說明：**
- E2E-TC 會處理 SYNC、DELAY_REQ、FOLLOW_UP、DELAY_RESP 四種消息
- 每種消息都會被轉發到其他端口，並計算 residence time

### 2.2 One-Step Sync 處理 (tc.c)

#### One-Step 判斷函數
**檔案：** `msg.h` 第 459-462 行

```c
static inline Boolean one_step(struct ptp_message *m)
{
	if (assume_two_step)
		return 0;
	return !field_is_set(m, 0, TWO_STEP);
}
```

**說明：**
- 檢查消息的 flagField[0] 中的 TWO_STEP bit
- 如果 TWO_STEP 未設定，則為 one-step 模式

#### tc_fwd_sync - Sync 消息轉發
**檔案：** `tc.c` 第 439-471 行

```c
int tc_fwd_sync(struct port *q, struct ptp_message *msg)
{
	struct ptp_message *fup = NULL;
	int err;

	if (one_step(msg)) {
		// 為 one-step Sync 創建 Follow_Up 消息
		fup = msg_allocate();
		if (!fup) {
			return -1;
		}
		fup->header.tsmt               = FOLLOW_UP | (msg->header.tsmt & 0xf0);
		fup->header.ver                = msg->header.ver;
		fup->header.messageLength      = htons(sizeof(struct follow_up_msg));
		fup->header.domainNumber       = msg->header.domainNumber;
		fup->header.sourcePortIdentity = msg->header.sourcePortIdentity;
		fup->header.sequenceId         = msg->header.sequenceId;
		fup->header.logMessageInterval = msg->header.logMessageInterval;
		fup->follow_up.preciseOriginTimestamp = msg->sync.originTimestamp;

		// 將原 Sync 消息標記為 TWO_STEP
		msg->header.flagField[0]      |= TWO_STEP;
	}

	// 轉發 Sync 事件消息
	err = tc_fwd_event(q, msg);
	if (err) {
		return err;
	}

	// 如果是 one-step，轉發 Follow_Up
	if (fup) {
		err = tc_fwd_folup(q, fup);
		msg_put(fup);
	}
	return err;
}
```

**關鍵流程：**
1. **檢測 one-step 模式**：使用 `one_step(msg)` 判斷
2. **創建 Follow_Up**：動態分配並填充 Follow_Up 消息
3. **設定 TWO_STEP flag**：將原始 Sync 標記為 two-step
4. **轉發 Sync**：調用 `tc_fwd_event()` 處理事件消息
5. **轉發 Follow_Up**：調用 `tc_fwd_folup()` 處理 Follow_Up

### 2.3 事件消息轉發與時間戳處理 (tc.c)

#### tc_fwd_event - 事件消息轉發
**檔案：** `tc.c` 第 262-311 行

```c
static int tc_fwd_event(struct port *q, struct ptp_message *msg)
{
	tmv_t egress, ingress = msg->hwts.ts, residence;
	struct port *p;
	int cnt, err;
	double rr;

	clock_gettime(CLOCK_MONOTONIC, &msg->ts.host);

	/* 第一步：先發送事件消息 */
	for (p = clock_first_port(q->clock); p; p = LIST_NEXT(p, list)) {
		if (tc_blocked(q, p, msg)) {
			continue;
		}
		// 使用 TRANS_DEFER_EVENT 延遲取得時間戳
		cnt = transport_send(p->trp, &p->fda, TRANS_DEFER_EVENT, msg);
		if (cnt <= 0) {
			pr_err("failed to forward event from %s to %s",
				q->log_name, p->log_name);
			port_dispatch(p, EV_FAULT_DETECTED, 0);
		}
	}

	/* 第二步：回頭收集傳送時間戳 */
	for (p = clock_first_port(q->clock); p; p = LIST_NEXT(p, list)) {
		if (tc_blocked(q, p, msg)) {
			continue;
		}
		// 獲取硬體時間戳
		err = transport_txts(&p->fda, msg);
		if (err || !msg_sots_valid(msg)) {
			pr_err("failed to fetch txts on %s to %s event",
				q->log_name, p->log_name);
			port_dispatch(p, EV_FAULT_DETECTED, 0);
			continue;
		}

		// 計算 residence time
		ts_add(&msg->hwts.ts, p->tx_timestamp_offset);
		egress = msg->hwts.ts;
		residence = tmv_sub(egress, ingress);
		rr = tmv_dbl(residence);

		// 完成 TC 處理
		tc_complete(q, p, msg, residence);
	}
	return 0;
}
```

**關鍵流程：**
1. **遍歷所有端口**：將消息轉發到所有非阻塞端口
2. **TRANS_DEFER_EVENT**：使用延遲模式發送，稍後再取時間戳
3. **收集 TX 時間戳**：調用 `transport_txts()` 獲取硬體時間戳
4. **計算 residence time**：egress_ts - ingress_ts
5. **完成 TC 處理**：調用 `tc_complete()` 更新 correction field

#### tc_complete_syfup - 更新 Correction Field
**檔案：** `tc.c` 第 178-238 行

```c
static void tc_complete_syfup(struct port *q, struct port *p,
			      struct ptp_message *fup, tmv_t residence)
{
	// ... 省略匹配邏輯 ...

	// 更新 correction field
	c1 = net2host64(fup->header.correction);
	c2 = c1 + tmv_to_TimeInterval(residence);
	c2 += tmv_to_TimeInterval(q->peer_delay);
	c2 += q->asymmetry;
	fup->header.correction = host2net64(c2);

	// 發送 Follow_Up
	cnt = transport_send(p->trp, &p->fda, TRANS_GENERAL, fup);
	if (cnt <= 0) {
		pr_err("tc failed to forward follow up on %s", p->log_name);
		port_dispatch(p, EV_FAULT_DETECTED, 0);
	}

	// 恢復原始 correction 值供下一個端口使用
	fup->header.correction = host2net64(c1);
	// ... 省略清理邏輯 ...
}
```

**Correction Field 更新公式：**
```
new_correction = old_correction + residence_time + peer_delay + asymmetry
```

### 2.4 UDP 傳輸層處理 (udp.c)

#### UDP One-Step 發送
**檔案：** `udp.c` 第 243-268 行

```c
switch (event) {
case TRANS_GENERAL:
	fd = fda->fd[FD_GENERAL];
	break;
case TRANS_EVENT:
case TRANS_ONESTEP:
case TRANS_P2P1STEP:
case TRANS_DEFER_EVENT:
	fd = fda->fd[FD_EVENT];
	break;
}

// 設定目標地址和端口
addr->sin.sin_port = htons(event ? EVENT_PORT : GENERAL_PORT);

/*
 * 為 UDP checksum 修正延長 payload 2 bytes
 * 這不是標準的一部分，但是 phyter 的工作方式
 */
if (event == TRANS_ONESTEP)
	len += 2;

cnt = sendto(fd, buf, len, 0, &addr->sa, sizeof(addr->sin));
if (cnt < 1) {
	pr_err("sendto failed: %m");
	return -errno;
}

// 立即取得時間戳
return event == TRANS_EVENT ? sk_receive(fd, junk, len, NULL, hwts, MSG_ERRQUEUE) : cnt;
```

**說明：**
- **TRANS_ONESTEP** 使用 event socket (FD_EVENT)
- Payload 長度會額外增加 2 bytes 用於 UDP checksum 修正
- 對於 TRANS_EVENT 會立即使用 `MSG_ERRQUEUE` 取得時間戳
- 對於 TRANS_ONESTEP 則由硬體處理，不需要軟體取時間戳

#### 硬體時間戳設定 (sk.c)

**檔案：** `sk.c` 第 577-606 行

```c
case TS_ONESTEP:
	flags |= SOF_TIMESTAMPING_TX_HARDWARE;
	// ... 其他設定 ...

	switch (hwts_filter) {
	case HWTS_FILTER_NORMAL:
		ifreq.ifr_data = (void *) &cfg;
		memset(&cfg, 0, sizeof(cfg));
		cfg.tx_type   = HWTSTAMP_TX_ON;
		cfg.rx_filter = HWTSTAMP_FILTER_PTP_V2_EVENT;
		break;
	case HWTS_FILTER_CHECK:
		// ... 檢查模式 ...
		break;
	case HWTS_FILTER_FULL:
		switch (transport) {
		case TRANS_UDP_IPV4:
		case TRANS_UDP_IPV6:
		case TRANS_IEEE_802_3:
			switch (type) {
			case TS_ONESTEP:
				tx_type = HWTSTAMP_TX_ONESTEP_SYNC;
				break;
			case TS_P2P1STEP:
				tx_type = HWTSTAMP_TX_ONESTEP_P2P;
				break;
			default:
				tx_type = HWTSTAMP_TX_ON;
				break;
			}
			// ... 設定 tx_type ...
		}
	}
```

**說明：**
- 當使用 `TS_ONESTEP` 時，會設定 `HWTSTAMP_TX_ONESTEP_SYNC`
- 這告訴硬體自動處理 one-step sync 的時間戳插入

---

## 三、配置檔範例

### 完整的 E2E-TC One-Step UDPv4 配置

創建檔案：`configs/E2E-TC-ONESTEP-UDPv4.cfg`

```ini
[global]
#
# E2E Transparent Clock One-Step UDPv4 配置
#

# Clock 類型：E2E Transparent Clock
clock_type              E2E_TC

# 時間戳模式：One-Step (硬體自動填充時間戳)
time_stamping           hardware

# 網路傳輸：UDP over IPv4
network_transport       UDPv4

# TC 特定設定
priority1               254
priority2               254
free_running            1
freq_est_interval       3
tc_spanning_tree        1

# 日誌設定
summary_interval        1
logging_level           6

# 端口設定
delay_mechanism         E2E
```

### 與預設 E2E-TC.cfg 的差異

原配置 (`configs/E2E-TC.cfg`):
```ini
network_transport       L2      # 使用 Layer 2 Ethernet
```

新配置:
```ini
network_transport       UDPv4   # 使用 UDP over IPv4
time_stamping           hardware  # 需明確指定使用硬體時間戳
```

---

## 四、重要數據結構

### 4.1 PTP 消息結構 (msg.h)

```c
struct ptp_message {
	struct ptp_header header;
	union {
		struct sync_msg          sync;
		struct delay_req_msg     delay_req;
		struct follow_up_msg     follow_up;
		struct delay_resp_msg    delay_resp;
		struct pdelay_req_msg    pdelay_req;
		struct pdelay_resp_msg   pdelay_resp;
		struct pdelay_resp_fup_msg pdelay_resp_fup;
		struct announce_msg      announce;
		struct signaling_msg     signaling;
		struct management_msg    management;
	} PACKED;

	struct hw_timestamp hwts;      // 硬體時間戳
	struct timespec ts.host;       // 主機時間戳
	// ... 其他欄位 ...
};
```

### 4.2 TC 傳輸數據 (tc.c)

```c
struct tc_txd {
	struct ptp_message *msg;       // 原始消息
	tmv_t residence;               // residence time
	int ingress_port;              // 入口端口
	TAILQ_ENTRY(tc_txd) list;
};
```

---

## 五、執行與測試

### 5.1 編譯

```bash
make clean
make
```

### 5.2 執行 E2E-TC

```bash
sudo ./ptp4l -i eth0 -i eth1 -f configs/E2E-TC-ONESTEP-UDPv4.cfg -m
```

**參數說明：**
- `-i eth0 -i eth1`: 指定兩個網路介面作為 TC 端口
- `-f configs/E2E-TC-ONESTEP-UDPv4.cfg`: 使用自定義配置檔
- `-m`: 輸出日誌到標準輸出

### 5.3 驗證 One-Step 模式

使用 Wireshark 抓包驗證：

1. **檢查 Sync 消息**：
   - Flags 欄位應設定 `twoStepFlag = 1` (因為代碼會設定 TWO_STEP)
   - originTimestamp 應為硬體填充的實際發送時間

2. **檢查 Follow_Up 消息**：
   - correctionField 應包含 residence time
   - preciseOriginTimestamp 應與 Sync 的 originTimestamp 一致

3. **檢查 UDPv4**：
   - 傳輸層協定應為 UDP
   - 目標 IP 為 224.0.1.129 (PTP 多播地址)
   - 端口：319 (event) 和 320 (general)

---

## 六、關鍵概念總結

### 6.1 One-Step vs Two-Step

| 特性 | One-Step | Two-Step |
|------|----------|----------|
| Sync 時間戳 | 硬體自動填充 | 軟體通過 Follow_Up 提供 |
| Follow_Up | 需要 | 必須 |
| 硬體要求 | 需支援 one-step | 一般硬體即可 |
| 延遲 | 較低 | 較高 |
| TWO_STEP flag | 0 (接收時) → 1 (TC 轉發後) | 1 |

### 6.2 E2E-TC 處理流程

```
Master              TC                 Slave
  |                 |                   |
  |--- Sync ------->| (ingress_ts)      |
  |                 |--- Sync --------->| (egress_ts)
  |                 | (計算 residence)   |
  |                 |                   |
  |--- Follow_Up -->| (更新 correction) |
  |                 |--- Follow_Up ---->|
  |                 | (correction +=    |
  |                 |  residence_time)  |
```

### 6.3 Correction Field 計算

```c
new_correction = old_correction
               + residence_time      // egress_ts - ingress_ts
               + peer_delay          // 端口間延遲
               + asymmetry           // 非對稱補償
```

---

## 七、故障排除

### 7.1 硬體不支援 One-Step

**症狀：** ptp4l 啟動失敗或無法取得時間戳

**解決方案：**
```bash
# 檢查網卡是否支援 one-step
ethtool -T eth0

# 如果不支援，改用 two-step
time_stamping  hardware  # 使用 TS_HARDWARE 而非 TS_ONESTEP
```

### 7.2 UDPv4 多播問題

**症狀：** 無法接收到 PTP 消息

**解決方案：**
```bash
# 檢查防火牆規則
sudo iptables -A INPUT -p udp --dport 319 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 320 -j ACCEPT

# 檢查多播路由
ip maddr show
netstat -g
```

### 7.3 時間戳偏移

**症狀：** Slave 時鐘偏差過大

**解決方案：**
在配置檔中調整時間戳偏移：
```ini
tx_timestamp_offset     0
rx_timestamp_offset     0
```

---

## 八、參考代碼檔案

| 檔案 | 主要功能 |
|------|----------|
| `e2e_tc.c` | E2E-TC 主要邏輯，消息分發 |
| `tc.c` | TC 轉發、residence time 計算 |
| `tc.h` | TC 介面定義 |
| `udp.c` | UDP 傳輸層實現 |
| `sk.c` | Socket 和硬體時間戳配置 |
| `msg.h` | PTP 消息結構定義 |
| `transport.h` | 傳輸層介面定義 |
| `config.c` | 配置參數解析 |
| `clock.c` | Clock 管理 |

---

## 結論

要啟用 E2E-TC one-step UDPv4 模式，需要：

1. **配置三個關鍵參數**：
   - `clock_type = E2E_TC`
   - `time_stamping = hardware` (實際使用 TS_ONESTEP)
   - `network_transport = UDPv4`

2. **硬體支援**：
   - 網卡必須支援 `HWTSTAMP_TX_ONESTEP_SYNC`
   - 驅動必須實現 one-step 時間戳插入

3. **核心流程**：
   - 接收 Sync → 判斷 one-step → 創建 Follow_Up
   - 轉發 Sync (設定 TWO_STEP flag)
   - 計算 residence time
   - 更新 Follow_Up 的 correction field
   - 轉發 Follow_Up

此配置適用於需要低延遲、高精度時間同步的工業自動化、電信網路等場景。
