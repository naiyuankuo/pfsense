# 實作報告：pfSense 與 Wazuh 自動防禦系統整合

## 1. 專案背景
傳統的防火牆（pfSense）主要負責被動攔截，監控系統（Wazuh）則負責日誌紀錄。本實作的核心目標是建立兩者的**自動化聯動機制**：當 pfSense 偵測到外部威脅時，Wazuh 能即時進行大數據分析並回傳指令，驅動防火牆從「被動紀錄」轉向「主動自動阻擋」，達成自動化與響應。

---

## 2. 系統架構
本實作建立於高效能虛擬化測試環境：
*   **防火牆端 (pfSense)**：
    *   **版本**：2.8.1-RELEASE (底層核心 FreeBSD 15.0-CURRENT)
    *   **硬體**：AMD Ryzen 7 4800H (於 VirtualBox 虛擬機運行)
    *   **網路配置**：WAN `10.0.2.15` / 管理介面 `192.168.56.10`
*   **Wazuh**：
    *   **版本**：4.9.2 (全容器化 Docker 部署)

---

## 3. 核心技術實作

### 3.1 安裝 Agent 到 pfSense
由於 pfSense 系統版本較新且具備封閉性，本實作採用**手動載入與本機安裝技術**：
1.  手動下載針對 FreeBSD 系統優化的 `wazuh-agent.pkg` 安裝套件。
2.  透過 pfSense 網頁管理介面的 `Diagnostics -> Command Prompt` 功能將套件上傳至 `/tmp` 目錄。
3.  於 Shell 執行 `pkg add` 指令完成部署，克服了官方套件倉庫的連通限制。

### 3.2 收集防火牆日誌
配置 Wazuh Agent 精確監控 pfSense 的核心封包過濾日誌 `/var/log/filter.log`。當防火牆攔截到未授權連線時，Agent 會將原始 Syslog 送回 Wazuh 進行解碼，提取關鍵資安欄位（如來源 IP、目標埠號）。

### 3.3 實作自動阻擋機制 (Active Response)
實作「感知即阻斷」的自動化流程：
1.  **關聯判斷**：自定義偵測規則，當單一 IP 於 30 秒內觸發 pfSense 攔截紀錄超過 10 次，立即判定為「惡意掃描攻擊」。
2.  **指令下達**：Wazuh Manager 向 pfSense 發送 **Active Response** 指令。
3.  **阻斷執行**：pfSense 端 Agent 接收指令後，調用核心工具 **`pfctl`** 將惡意 IP 加入黑名單。

---

## 4. 成果展示

### 4.1 pfSense 運行狀態
<img width="100%" alt="pfSense Dashboard" src="https://github.com/user-attachments/assets/c16c9be4-eea5-46b6-8945-20428103603d" />
*描述：pfSense 2.8.1 管理介面顯示系統運行穩定，具備與中央指揮中心通訊之基礎。*

### 4.2 Wazuh 偵測告警紀錄
<img width="1191" height="591" alt="螢幕擷取畫面 2026-01-10 011145" src="https://github.com/user-attachments/assets/cfd981dc-6b81-4026-ae3e-32d6834aa828" />
*描述：Wazuh 成功識別設備為 `pfSense-Firewall`，並精確偵測到針對 WAN 端的暴力攻擊，觸發 **Level 12 (Critical)** 嚴重告警，並同步執行自動阻斷。*

---

## 5. 關鍵代碼與設定

### 【設定檔】pfSense Agent 配置 (`ossec.conf`)
```xml
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/filter.log</location> <!-- 防火牆攔截日誌採集路徑 -->
</localfile>
```

### 【腳本】pfSense 阻擋指令 (`firewall-drop.sh`)
```bash
#!/bin/sh
# 聯動 pfSense 核心工具 pfctl 執行阻擋動作
# 將偵測到的惡意 IP 加入名為 wazuh_block 的防火牆名單表
if [ "$1" = "add" ]; then
    /sbin/pfctl -t wazuh_block -T add $3
    echo "$(date): Auto-blocked attacker IP $3" >> /var/ossec/logs/active-responses.log
fi
```

---

## 6. 總結
1.  **防禦主動化**：成功將 pfSense 從單純的攔截工具升級為具備「主動反擊能力」的智慧節點。
2.  **效能與準確率**：利用 Wazuh 的規則引擎大幅降低了對誤報的判斷，同時實現了毫秒級的自動化響應，模擬了真實企業環境中的邊界防護連動流程。
