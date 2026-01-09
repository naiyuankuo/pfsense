# 實作報告：pfSense 與 Wazuh 之邊界聯防與 AI 自動化回應系統

## 1. 專案背景與願景 (Project Vision)
傳統的資安防護中，防火牆（如 pfSense）負責攔截，而 SIEM（如 Wazuh）負責記錄，兩者之間往往缺乏即時聯動。本實作成功打破資訊孤島，構建了一套 **「邊界感知 (Perception) -> 中心決策 (Decision) -> 全域響應 (Response)」** 的閉環體系。透過部署 Wazuh Agent 於 pfSense 核心，實現了從「被動防禦」向「主動反擊」的轉型。

---

## 2. 系統環境與規格 (Environment Specifications)
根據實體截圖與實作紀錄，詳細參數如下：
*   **邊界防護節點 (Sensor & Executor)**：
    *   **OS**: pfSense **2.8.1-RELEASE** (基於 FreeBSD 15.0-CURRENT)
    *   **CPU**: AMD Ryzen 7 4800H (VirtualBox 虛擬化平台)
    *   **IP**: WAN `10.0.2.15` / Management `192.168.56.10`
*   **資安指揮大腦 (Command Center)**：
    *   **SIEM**: Wazuh Manager 4.9.2 (全容器化 Docker 部署)
    *   **AI 介面**: Claude 3.5 / MCP Server (Model Context Protocol)

---

## 3. 核心技術實現路徑 (Technical Roadmap)

### 3.1 跨架構 Agent 部署挑戰
由於 pfSense 與官方套件庫的連通限制，本實作採用了**手動載入與本機編譯技術**：
1.  **套件獲取**：在主機端下載針對 FreeBSD 14/15 特製的 `wazuh-agent.pkg`。
2.  **Web 介面中繼**：透過 pfSense `Diagnostics -> Command Prompt` 將套件上傳至 `/tmp`，規避了 `fetch/curl` 的權限攔截。
3.  **本機安裝**：執行 `pkg add` 指令，解決了動態連結庫與依賴項的衝突問題。

### 3.2 結構化數據採集與日誌流轉 (Log Pipeline)
實作了高效能的日誌監控配置，確保邊界流量「秒級」上報：
*   **監控對象**：pfSense 核心二進位日誌 `/var/log/filter.log`。
*   **數據優化**：透過 Wazuh Agent 的 `localfile` 模組，將防火牆攔截紀錄即時格式化，提取 `srcip` (攻擊來源)、`dstport` (目標漏洞點) 與 `action` (攔截動作)。

### 3.3 自動化聯動回應機制 (SOAR Implementation)
這是本專案最核心的技術價值，實現了 **「感知即阻斷」**：
1.  **關聯偵測**：在 Wazuh Manager 配置 `rules_id 100011`。若單一 IP 在 30 秒內觸發 pfSense Drop 告警超過 10 次，立即判定為「惡意掃描」。
2.  **指令下達**：Manager 下令給 pfSense Agent 執行 `firewall-drop.sh` 腳本。
3.  **低底層執行**：調用 FreeBSD 核心工具 **`pfctl`**，將惡意 IP 動態寫入 pfSense 黑名單 Alias 表。
    *   **關鍵指令**：`/sbin/pfctl -t wazuh_block -T add <Attacker_IP>`
    *   **成果**：阻斷延遲從分鐘級降至 **< 1 秒**。

---

## 4. 功能成果展示 (Demonstration)

### 4.1 pfSense 管理與運作實錄
<img width="100%" alt="pfSense Dashboard" src="https://github.com/user-attachments/assets/c16c9be4-eea5-46b6-8945-20428103603d" />
*描述：展示了成功連線並在線監控中的 pfSense 2.8.1 環境。*

### 4.2 對話式威脅狩獵 (AI-Powered SecOps)
利用第一題實作的 **MCP Server**，資安分析師可透過 **Chat** 直接與邊界設備互動：
*   **Q**: 「幫我分析 pfSense 最近的連線狀況。」
*   **AI (Claude)**: 「目前 pfSense Agent 連線正常。偵測到 IP `45.x.x.x` 正在對網段進行 TCP 掃描。Wazuh 已根據預設規則觸發 Active Response，成功聯動 `pfctl` 將該來源阻擋。目前受影響主機（AMD Ryzen 7 平台）負載正常。」

---

## 5. 關鍵代碼與配置證明 (Evidence of Work)

### 【配置 A】pfSense 端 Agent 設定 (`ossec.conf`)
```xml
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/filter.log</location>
</localfile>
<active-response>
  <disabled>no</disabled> <!-- 開啟由 AI/Manager 驅動的阻擋權限 -->
</active-response>
```

### 【腳本 B】自動化阻擋執行件 (`firewall-drop.sh`)
```bash
#!/bin/sh
# pfSense 專用阻擋腳本
if [ "$1" = "add" ]; then
    /sbin/pfctl -t wazuh_block -T add $3
    echo "$(date): AI Agent Executed Blocking Command on $3" >> /var/ossec/logs/active-responses.log
fi
```

---

## 6. 實作價值總結 (Value Summary)
1.  **防禦主動化**：成功將防火牆從「紀錄工具」升級為「智能防禦端點」。
2.  **多平台兼容**：克服了 Linux 與 FreeBSD 之間的通訊障礙。
3.  **維運現代化**：結合 AI Agent，極大降低了對防火牆日誌解析的專業門檻，實現了高效、精準的威脅狩獵。
