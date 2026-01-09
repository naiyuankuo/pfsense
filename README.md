# 實作報告：pfSense 與 Wazuh 自動防禦系統整合

## 1. 專案背景
傳統的防火牆（pfSense）只負責攔截，監控系統（Wazuh）只負責記錄。本實作的目的是將兩者串聯：讓 pfSense 偵測到威脅後，Wazuh 能即時分析並下達指令，讓防火牆從「被動紀錄」轉為「主動執行阻擋」。

## 2. 系統環境與規格
*   **防火牆端 (pfSense)**：
    *   版本：2.8.1-RELEASE (FreeBSD 15.0)
    *   硬體：AMD Ryzen 7 4800H (VirtualBox 虛擬機)
    *   網路：WAN 10.0.2.15 / 管理介面 192.168.56.10
*   **Wazuh**：
    *   版本：4.9.2 (Docker 部署)

## 3. 核心技術實作

### 3.1 安裝 Agent 到 pfSense
由於版本相容性限制，本實作採用手動安裝：
1.  下載針對 FreeBSD 系統優化的 `wazuh-agent.pkg` 安裝包。
2.  透過 pfSense 網頁介面上傳至 `/tmp` 目錄。
3.  使用 `pkg add` 指令完成本機部署，確保資料採集引擎正常運作。

### 3.2 收集防火牆日誌
設定 Wazuh Agent 持續監控 pfSense 的防火牆日誌檔案 `/var/log/filter.log`。當防火牆攔截到任何連線時，Agent 會即時將來源 IP 與目標埠號送回 Wazuh 進行結構化分析。

### 3.3 實作自動阻擋機制 (Active Response)
建立「偵測 $\rightarrow$ 判斷 $\rightarrow$ 執行」的自動化流程：
1.  **判斷規則**：設定當單一 IP 在 30 秒內被攔截超過 10 次，Wazuh 即判定為惡意掃描。
2.  **執行阻擋**：Wazuh 下令給 pfSense 執行 `pfctl` 指令。
3.  **阻斷成果**：將惡意 IP 加入防火牆黑名單清單，達成秒級的自動化防禦。

## 4. 成果展示

### 4.1 pfSense 運行狀態
<img width="100%" alt="pfSense Dashboard" src="https://github.com/user-attachments/assets/c16c9be4-eea5-46b6-8945-20428103603d" />
*描述：pfSense 虛擬機環境架設完成，管理介面運行正常。*

### 4.2 Wazuh 偵測告警紀錄
<img width="1191" height="591" alt="螢幕擷取畫面 2026-01-10 011145" src="https://github.com/user-attachments/assets/13e6dec7-d6ec-4d15-abf6-c27c806620f4" />
*描述：Wazuh 成功識別 pfSense 設備，並在偵測到多次攻擊後觸發 Level 12 嚴重告警。*

## 5. 關鍵代碼與設定

### 【設定檔】pfSense Agent 配置 (`ossec.conf`)
```xml
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/filter.log</location>
</localfile>
```

### 【腳本】pfSense 阻擋指令 (`firewall-drop.sh`)
```bash
#!/bin/sh
# 聯動 pfSense 核心工具進行阻擋
if [ "$1" = "add" ]; then
    /sbin/pfctl -t wazuh_block -T add $3
fi
```

## 6. 總結
1.  **防禦自動化**：成功讓防火牆具備主動反擊能力。
2.  **實踐價值**：模擬了真實企業環境中的邊界防護與中心監控連動流程。
