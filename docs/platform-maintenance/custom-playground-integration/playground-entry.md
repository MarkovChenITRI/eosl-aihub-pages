# 遊樂場入口

這一頁整理 AI Hub 如何把使用者導向外部自訂遊樂場，以及外部遊樂場收到入口請求後應該如何完成身分驗證、模式判斷與頁面導向。

## 入口來源

AI Hub 目前會從兩種情境導向遊樂場：

- 建立或編輯工作流時，導向可編輯的 Builder 或 Runner。
- 公開 Gallery 或 Agent 詳情頁上，導向唯讀 Runner。

可編輯入口會使用 AI Hub 簽發的短效 handoff token，外部遊樂場再回查 AI Hub 驗證 token。公開唯讀入口則以公開 Agent 設定讀取為主，不要求訪客登入。

## 維護檢查重點

平台維護者排查入口問題時，先確認來源頁面產出的目標 URL，再確認外部遊樂場是否成功驗證 handoff token 或載入公開設定。

| 情境 | AI Hub 入口 | 外部遊樂場責任 |
| --- | --- | --- |
| 建立新工作流 | `/publish/agents/playground/new` | 開啟 Builder，完成 token 驗證後建立新工作流。 |
| 編輯既有工作流 | `/publish/agents/playground/<agent_id>` | 開啟可編輯 Runner，確認使用者可操作該 Agent。 |
| 公開瀏覽工作流 | 公開 Agent 頁面的 Runner 連結 | 開啟唯讀 Runner，從公開設定端點載入工作流。 |

## 留存證據

排查時至少留下來源頁面、目標 URL、使用者帳號、`agent_id`、handoff 驗證回應，以及外部遊樂場最後停留的頁面。若是公開唯讀入口，則留下公開設定讀取結果與 Runner 載入狀態。若使用者已進入 Runner 但回覆前的狀態顯示異常，還要留下 Runner 模式、載入的 config 來源、`python_source` 是否包含 `stage_labels`，以及外部服務是否收到可顯示的階段事件；後續依 [Runner 階段狀態](runner-stage-status.md) 追查。
