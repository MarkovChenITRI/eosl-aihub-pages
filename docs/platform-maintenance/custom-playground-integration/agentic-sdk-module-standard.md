# Agentic SDK 模組規範

## 這份文件會幫你完成什麼

當你是平台維護者，正在確認外部自訂遊樂場產生的 `python_source` 是否能被 Agentic SDK Runner 正確執行、Runner 階段狀態是否能對回模組、或自訂模組輸出後續模組讀不到資料時，就看這一頁。

這一頁接續 [Runner 階段狀態](runner-stage-status.md) 與 [Runner 對話記憶上下文](runner-memory-context.md)。AI Hub 負責保存與讀回 `python_source`；外部自訂遊樂場服務負責建立與執行 workflow；Agentic SDK 負責依五大模組共同規範推進流程。AI Hub 不解析模組內部邏輯，但維護者要能判斷 source 是否符合 SDK 可執行標準。

## 五大模組共用哪些標準

Agentic SDK 的五大模組是 `perceive`、`plan`、`retrieve`、`action`、`reflect`。它們在 workflow 裡負責的任務不同，但共同遵守同一個最小介面：

| 標準 | 維護者要確認什麼 |
| --- | --- |
| `name` | 模組在 workflow 裡對應哪個階段，例如 `action` 或 `retrieve`。 |
| `__call__(state)` | workflow 執行模組時能以 `WorkflowState` 呼叫它。 |
| `WorkflowState` | 模組從同一份執行狀態讀取使用者訊息、前一步 payload、entries 與 memory。 |
| `ModuleOutput.next_module` | 模組執行後交給哪一站；如果是 `None`，流程結束。 |
| `ModuleOutput.payload` | 寫回 `state.entities`，讓後續模組或最終結果能讀到。 |
| `ModuleOutput.context_updates` | 留下 `ContextEntry`，供 Reflect、debug、Runner 狀態或維護證據使用。 |

這些標準讓外部自訂遊樂場服務可以替換單一模組，而不需要改寫整條 workflow。維護者排查時，不需要先理解模組內部演算法；先確認它是否能讀 `WorkflowState`、回 `ModuleOutput`，以及輸出是否落在後續模組讀得到的位置。

## AI Hub 與外部服務的責任邊界

AI Hub 保存的是同一筆 Agent 的 `python_source`、`workflow_name`、`description` 與檔案包狀態。外部自訂遊樂場服務把 source 交給 Agentic SDK 執行，並把 stage event 顯示給使用者。模組是否符合共同規範，由外部服務與 SDK 執行結果驗證；AI Hub 的維護證據集中在 source 是否保存、讀回與對回同一筆 `agent_id`。

| 主體 | 維護責任 | 可觀察證據 |
| --- | --- | --- |
| AI Hub | 保存與讀回 source，不解析模組內部邏輯。 | config save/load API、SQL 中的 `python_source`、工作區 Agent 狀態。 |
| 外部自訂遊樂場服務 | 產生符合 SDK 規範的 source，建立 workflow，執行 Runner。 | 生成的 source、Runner 執行紀錄、stage event、最終回覆。 |
| Agentic SDK | 依 `WorkflowState` 與 `ModuleOutput` 推進五大模組。 | `visit_counts`、`result.entities`、`result.entries`、錯誤或 abort reason。 |

## 排查時先看哪裡

| 問題類型 | 優先檢查 |
| --- | --- |
| Runner 開啟後 workflow 不能執行 | source 是否存在 `Workflow(...)`，自訂模組是否有 `name` 與 `__call__(state)`。 |
| 自訂 action 有執行但沒有最後回覆 | `ModuleOutput.payload` 是否寫入 `latest_final_message`，`next_module` 是否為 `None` 或正確下一站。 |
| Reflect 或 UI 看不到前一步紀錄 | 模組是否在 `context_updates` 放入對應的 `ContextEntry`。 |
| 階段狀態顯示不符合預期 | `name`、`stage_labels`、SDK stage event 的 `stage` / `label` 是否一致。 |
| AI Hub 工作區顯示正常但 Runner 結果錯 | AI Hub 保存/讀回已成立，下一步查外部服務 source 生成與 SDK 執行結果。 |

## 查完這頁後應該留下什麼

查完這一頁後，平台維護者應該能留下同一筆模組規範問題的證據包：`agent_id`、讀回的 `python_source`、自訂模組名稱、模組 `name`、`__call__(state)` 是否存在、執行後的 `visit_counts`、`result.entities`、`result.entries`，以及外部服務收到的 stage event 或錯誤訊息。

當 AI Hub 能讀回同一份 source，外部服務能建立 workflow，SDK 執行後能在 `result.entities` 與 `result.entries` 看到模組輸出，且 Runner 畫面能對回同一個 stage event 時，模組規範這一段才算成立。