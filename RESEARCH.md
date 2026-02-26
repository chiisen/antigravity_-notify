# Antigravity 監控技術調查報告

> 調查日期：2026-02-26

## 背景

本專案旨在監控 Antigravity AI IDE 的執行狀態，當需要使用者核准時透過 Telegram 發送通知，並支援直接回覆核准。

## 調查方法與結果

### 1. antigravity-trace（Shadow Extension 方式）

**方法**：使用 `antigravity-trace` 工具建立 shadow extension，攔截語言伺服器的所有通訊。

**結果**：❌ 失敗

**原因**：
- Antigravity 優先載入 `/Applications/Antigravity.app/Contents/Resources/app/extensions/antigravity` 內建的 extension
- 忽略 `~/.antigravity/extensions/antigravity` 的 shadow 版本
- Antigravity 新版可能改變了 extension 載入優先順序，關閉了這個漏洞

**過程**：
1. 安裝 antigravity-trace（Python 虛擬環境）
2. 執行安裝腳本，成功建立 shadow extension（version 99.99.99）
3. 重新啟動 Antigravity
4. 驗證語言伺服器仍使用原始 binary，未被 shadow 替換

**相關專案**：
- [ljw1004/antigravity-trace](https://github.com/ljw1004/antigravity-trace)

---

### 2. 本地日誌監控

**方法**：直接讀取 Antigravity 本地產生的日誌檔案。

**結果**：❌ 資訊不足

**原因**：
- Antigravity 將網路請求記錄在 `network.log`，但該檔案為空（大小為 0）
- `renderer.log` 只包含 VSCode 框架的警告/錯誤，無 API 請求細節
- Antigravity.log 只記錄語言伺服器啟動狀態
- 對話內容、核准請求等資訊**未儲存**在本地

**日誌位置**：
```
~/Library/Application Support/Antigravity/logs/<timestamp>/window1/
├── network.log          # 網路請求日誌（目前為空）
├── renderer.log         # 渲染器日誌
└── exthost/google.antigravity/Antigravity.log
```

**本地資料庫檢查**：
- `storage.json`：僅包含 UI 狀態、偏好設定
- `state.vscdb`：VSCode 狀態資料庫
- **無對話歷史儲存**

---

### 3. mitmproxy（HTTPS 攔截）

**方法**：使用 mitmproxy 建立中間人代理，攔截 HTTPS 流量。

**結果**：❌ 失敗

**原因**：
- Antigravity 使用 Google 內部端點（`daily-cloudcode-pa.sandbox.googleapis.com`）
- 連接流程複雜：SSL 握手 + socket/TLS 流程
- 基本代理設定無法可靠攔截流量

**驗證來源**：
- [elad12390/antigravity-proxy](https://github.com/elad12390/antigravity-proxy) 專案已嘗試並記錄失敗經驗

---

### 4. 其他嘗試過的方法（根據 antigravity-proxy 專案）

| 方法 | 結果 |
|------|------|
| HTTP_PROXY/HTTPS_PROXY 環境變數 | ❌ 失敗 |
| DNS 層面攔截 (/etc/hosts) | ❌ 失敗 |
| Binary patching | ❌ 失敗 |

---

## Antigravity 架構分析

```
┌─────────────────────────────────────────────────────────────┐
│  VSCode Extension (TypeScript)                             │
│  /Applications/Antigravity.app/.../extensions/antigravity │
└────────────────────────┬────────────────────────────────────┘
                         │ 多種通訊渠道
┌────────────────────────▼────────────────────────────────────┐
│  Core Agent Binary (Go) - language_server_macos_arm        │
│  - LLM 端點: daily-cloudcode-pa.sandbox.googleapis.com    │
│  - Extension Server (protobuf/HTTPS)                       │
│  - LSP Port                                                │
│  - STDIO                                                   │
└────────────────────────────────────────────────────────────┘
```

**關鍵發現**：
- Antigravity 使用自定義的二進制協定（protobuf）進行內部通訊
- LLM API 調用透過 Google 內部端點，無法被標準代理攔截
- 可能存在憑證 pinning（Certificate Pinning）機制

---

## 結論

### 技術可行性評估

| 方案 | 穩定性 | 風險 |
|------|--------|------|
| Antigravity 官方 API | ⭐⭐⭐⭐⭐ | 需確認是否存在 |
| 本地日誌監控 | ⭐ | 資料不足 |
| Shadow Extension | ❌ | 已被官方封堵 |
| mitmproxy | ❌ | 技術上不可行 |

### 建議後續方向

1. **聯繫 Google**：詢問是否有官方 API 或監控方式
2. **Antigravity 付費版**：可能提供企業級監控功能
3. **功能建議**：向 Google 提出「核准通知 API」需求

---

## 參考資料

- [ljw1004/antigravity-trace](https://github.com/ljw1004/antigravity-trace) - LLM 呼叫監控工具
- [elad12390/antigravity-proxy](https://github.com/elad12390/antigravity-proxy) - MITM 代理實驗（已歸檔）
- Antigravity 版本：1.107.0
- 語言伺服器版本：1.19.4

---

**文件資訊**
- 作者：AI Assistant
- 建立日期：2026-02-26
- 版本：v1.0
