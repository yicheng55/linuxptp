# E2E Transparent Clock One-Step UDPv4 使用說明（linuxptp-4.2）

> 本文件說明如何在 linuxptp-4.2 中啟用 **E2E Transparent Clock (E2E_TC)**、**one-step**，以及 **network_transport = UDPv4**，並對相關程式碼做簡要說明。
>
> 注意：此檔案為全新文件，未參考任何既有的 `.md` 檔內容。

---

## 1. 目標與條件總結

要在 linuxptp-4.2 中啟用 E2E TC one-step UDPv4，關鍵設定為：

- **clockType**：`E2E_TC`（對應 `CLOCK_TYPE_E2E`）
- **delay_mechanism**：`E2E`（對應 `DM_E2E`）
- **network_transport**：`UDPv4`（對應 `TRANS_UDP_IPV4`）
- **time_stamping / twoStepFlag**：設定為 **one-step** 模式（`TS_ONESTEP` 搭配 `twoStepFlag = 0`），且必須符合硬體支援條件

這些條件會在 `ptp4l` 啟動時計算並檢查，若不符合會直接輸出錯誤訊息並退出。

---

## 2. 設定檔關鍵參數

以下參數通常放在 `ptp4l` 設定檔的 `[global]` 區段與各 `[port]` 區段（此處只列出與本主題直接相關的項目）：

### 2.1 clockType: E2E_TC

- 參數名稱：`clockType`
- 目標值：`E2E_TC`
- 對應程式碼：
  - `config.c` 中的 `clock_type_enu`：
    - `"E2E_TC" -> CLOCK_TYPE_E2E`
  - `clock.c` 建立 clock 時會依據 `clockType` 決定 `c->type`。
  - `ptp4l.c` 會在建立 clock 前根據 `CLOCK_TYPE_E2E` 做額外檢查（參考 §3.1）。

### 2.2 delay_mechanism: E2E

- 參數名稱：`delay_mechanism`
- 目標值：`E2E`
- 對應程式碼：
  - `config.c` 的 `delay_mech_enu`：
    - `"E2E" -> DM_E2E`
  - `ptp4l.c` 在 `CLOCK_TYPE_E2E` 分支中檢查 `delay_mechanism` 是否為 `DM_E2E`（不符合則輸出 `E2E_TC needs E2E delay mechanism`）。
  - `port.c` 在 port 初始化時也檢查 `type == CLOCK_TYPE_E2E` 且每個 port 的 `p->delayMechanism == DM_E2E`（否則輸出 `E2E TC needs E2E ports`）。

### 2.3 network_transport: UDPv4

- 參數名稱：`network_transport`
- 目標值：`UDPv4`
- 對應程式碼：
  - `config.c` 的 `nw_trans_enu`：
    - `"UDPv4" -> TRANS_UDP_IPV4`
  - `config.c` 的 `PORT_ITEM_ENU("network_transport", TRANS_UDP_IPV4, nw_trans_enu)`：
    - 表示 **預設值就是 UDPv4**。
  - `ptp4l.c`、`pmc.c` 中有設定 `network_transport` 的 CLI 選項與切換邏輯，但若你只想使用 UDPv4，一般只需在 config 保持預設或顯式設定為 `UDPv4` 即可。
  - 底層傳輸實作則在 `udp.c` 與 `transport.c` 中（參考 §3.4）。

### 2.4 time_stamping / twoStepFlag（one-step）

- 參數名稱：
  - `time_stamping`
  - `twoStepFlag`
- 推薦設定：
  - `time_stamping`：`TS_ONESTEP`（或硬體支援 one-step 時，由程式自動從 `TS_HARDWARE` 升級）
  - `twoStepFlag`：`0`（one-step 模式需 twoStepFlag = 0）
- 對應程式碼：
  - `config.c` 中：
    - `GLOB_ITEM_ENU("time_stamping", TS_ONESTEP, timestamping_enu)` 代表預設為 `TS_ONESTEP`。
    - `GLOB_ITEM_INT("twoStepFlag", 0, 0, 1)` 預設 `twoStepFlag = 0`。
  - `config.c` 的 `config_harmonize_onestep(struct config *cfg)`：
    - 取得 `time_stamping` 與 `twoStepFlag` 後，強制兩者一致：
      - 若 `time_stamping` 為 `TS_SOFTWARE` / `TS_LEGACY_HW` 且 `twoStepFlag = 0`，則報錯：_"one step is only possible with hardware time stamping"_。
      - 若 `time_stamping` 為 `TS_HARDWARE` 且 `twoStepFlag = 0`，則調整為 `TS_ONESTEP`，並輸出 debug 訊息說明升級為 one-step。
      - 若 `time_stamping` 為 `TS_ONESTEP` 或 `TS_P2P1STEP` 且 `twoStepFlag = 1`，則把 `twoStepFlag` 改為 `0`。
  - `clock.c` 中在建立 clock 時會呼叫 `config_harmonize_onestep(config)`：
    - 若 harmonize 失敗（回傳非 0）則建立 clock 失敗，`ptp4l` 啟動中止。

---

## 3. 關聯程式碼行為說明

本節只做邏輯層面的說明，方便你對照原始碼查詢。

### 3.1 ptp4l.c：建立 E2E TC clock 前的檢查

在 `ptp4l.c` 中，當根據設定檔解析出 `clockType = E2E_TC` 時，會將其對應為 `CLOCK_TYPE_E2E`，並在建立 clock 之前做以下檢查：

- **介面數量檢查**：
  - 若 `cfg->n_interfaces < 2`，輸出：`"TC needs at least two interfaces"`，並中止；
  - 這表示 TC 至少需要兩個 port 才有轉發意義。
- **delay_mechanism 檢查**：
  - 若 `config_get_int(cfg, NULL, "delay_mechanism") != DM_E2E`，輸出：`"E2E_TC needs E2E delay mechanism"`，並中止；
  - 也就是 **clockType = E2E_TC 時，delay_mechanism 必須是 E2E**。

若上述檢查都通過，才會呼叫 `clock_create(type, cfg, req_phc)` 建立 clock 物件。

### 3.2 clock.c：clock 建立與 one-step 檢查

在 `clock.c` 中，建立 clock 的流程包括：

- 根據 `clockType` 設定 `c->type`，`CLOCK_TYPE_E2E` 會被接受並作為 TC 類型。
- 從設定中讀取 `maxStepsRemoved` 等參數。
- 呼叫 `config_harmonize_onestep(config)`：
  - 確保 `time_stamping` 與 `twoStepFlag` 的組合是合法且一致的（如 §2.4 所述）。
  - 若不一致且無法修正，回傳錯誤，clock 建立失敗。
- 若 `twoStepFlag` 為 1，則在 `c->dds.flags` 中設置 `DDS_TWO_STEP_FLAG`。
- 讀取 `time_stamping`，決定 clock 是否使用軟體或硬體時間戳，並檢查每個 interface 的實際支援模式與需求是否相符。

對於 **one-step E2E TC**，重點在於：

- `time_stamping` 必須最終為 `TS_ONESTEP`（或相容的一步模式）。
- `twoStepFlag` 必須為 0。
- 若硬體不支援 one-step，`config_harmonize_onestep` 會拒絕這種組合，導致 clock 建立失敗。

### 3.3 port.c：每個 port 的 E2E 條件檢查

在 `port.c` 中的 port 初始化流程中，會根據整體 `clockType` 與每個 port 的設定做一致性檢查：

- 若 `type == CLOCK_TYPE_E2E`（即 E2E TC），且埠不是 UDS（非 Unix Domain Socket）：
  - 若 `p->delayMechanism != DM_E2E`，則輸出：`"E2E TC needs E2E ports"`，並跳到錯誤處理；
  - 這表示 **每個實體 port 的 delay_mechanism 必須是 E2E**。
- 若 `p->hybrid_e2e` 為真但 `p->delayMechanism != DM_E2E`，則輸出 warning：`"hybrid_e2e only works with E2E"`。

因此，即使 global 設為 `delay_mechanism = E2E`，若某 port 被單獨改為 P2P，也會在埠初始化時被擋下。

### 3.4 transport.c / udp.c：UDPv4 傳輸層

- `config.c` 將 `network_transport = UDPv4` 映射為 `TRANS_UDP_IPV4`。
- 在 `ptp4l.c` 等處，會根據 `network_transport` 來選擇對應的 transport 實作：
  - `TRANS_UDP_IPV4` 對應 `udp_transport_create()`，實作在 `udp.c` 中。
- `udp.c` 中實作了：
  - `udp_open` / `udp_close`：開啟/關閉 UDP socket；
  - `udp_send` / `udp_recv`：封裝 UDP 封包的收發；
  - `udp_transport_create`：註冊為一個 `transport` 實例，供上層 `transport.c` 使用。

對使用者來說，只要在設定檔中把 `network_transport` 設成 `UDPv4`（或保留預設）即可，自動使用 IPv4 UDP 的傳輸實作。

### 3.5 tc.c：Transparent Clock 轉發邏輯

在 `tc.c` 裡面與 E2E TC 相關的重點函式包含：

- `tc_forward(struct port *q, struct ptp_message *msg)`：
  - 若啟用了 `tc_spanning_tree`，在處理 ANNOUNCE 訊息時會遞增 `stepsRemoved` 欄位：`stepsRemoved = 1 + stepsRemoved`。
  - 接著遍歷所有 clock 的其他 port，檢查 `tc_blocked(q, p, msg)` 決定是否轉發；
  - 使用 `transport_send(p->trp, &p->fda, TRANS_GENERAL, msg)` 將訊息送出到各 port。
- 其它輔助函式（如 `tc_match_delay`、`tc_match_syfup`、`tc_complete_*` 等）負責比對與完成 sync/follow-up、delay_req/resp 之間的關聯，確保 TC 轉發與計算延遲資訊時的一致性。

這些程式碼對使用者來說通常不需要修改，只需了解在 E2E TC 模式下，TC 會根據接收到的 PTP 封包種類與內容，對 `stepsRemoved` 等欄位做更新並進行多埠轉發。

---

## 4. 實際設定範例（概念示意）

以下給出一個簡化的 `ptp4l` 設定檔片段，示意如何對應到上述條件（請依實際環境調整介面名稱、優先序與 profile）：

```ini
[global]
clockType            E2E_TC
delay_mechanism      E2E
network_transport    UDPv4

# one-step 設定（需硬體支援）
time_stamping        ONESTEP
# 或使用 HARDWARE，讓程式透過 config_harmonize_onestep 自動提升為 ONESTEP
# time_stamping      HARDWARE

twoStepFlag          0

[ens1f0]
# 若有 per-port 覆寫，必須與 E2E TC 一致
# delay_mechanism    E2E

[ens1f1]
# delay_mechanism    E2E
```

重點是：

- 至少要有兩個實體介面（TC needs at least two interfaces）。
- global 與 per-port 的 `delay_mechanism` 必須都是 `E2E`。
- `clockType = E2E_TC` 搭配 `delay_mechanism = E2E`，否則 `ptp4l` 直接報錯離開。
- `network_transport` 為 `UDPv4`（預設或顯式設定），即使用 IPv4 UDP PTP 封包。
- `time_stamping` / `twoStepFlag` 組合必須符合硬體支援與 `config_harmonize_onestep` 的約束，才能順利進入 one-step E2E TC 模式。

---

## 5. 小結

- **啟用 E2E Transparent Clock**：在 `ptp4l` 設定檔中設 `clockType = E2E_TC`，並確保 `delay_mechanism = E2E`、埠數量 ≥ 2。
- **啟用 one-step**：將 `time_stamping` / `twoStepFlag` 組合設為支援 one-step 的合法值（建議 `TS_ONESTEP` + `twoStepFlag = 0`）；程式會透過 `config_harmonize_onestep` 做最後檢查與修正。
- **使用 UDPv4 傳輸**：`network_transport = UDPv4`（預設即為 UDPv4），底層對應到 `udp.c` 的 `udp_transport_create`／`udp_send`／`udp_recv` 等實作。
- **若條件不符**（例如 delay_mechanism 非 E2E、time_stamping 與 twoStepFlag 組合不合法、介面數量不足），`ptp4l` 會在啟動過程即輸出明確錯誤訊息並中止，避免進入錯誤的 TC 模式。
