# Runner 階段狀態

## 這份文件會幫你完成什麼

當你是平台維護者，正在確認外部自訂遊樂場 Runner 已成功開啟，但畫面在最終回覆前沒有顯示目前 workflow 階段、顯示舊的階段文字、可編輯 Runner 與公開唯讀 Runner 顯示不一致，或使用者把階段顯示問題誤判成 AI Hub handoff 失敗時，就看這一頁。

這一頁接續 [連線方式與責任邊界](connection-and-responsibility.md)。AI Hub 負責保存與讀回同一筆 Agent 的 `python_source`、`workflow_name` 與 `description`；外部自訂遊樂場服務負責用這份設定建立 Runner、執行 Agentic SDK workflow，並把 SDK 送出的 stage event 顯示成畫面狀態。AI Hub 不解析 workflow 執行期間的 stage event，也不自行決定每個模組要顯示什麼文字。

## Runner 如何顯示 workflow 階段

Runner 階段狀態用來讓使用者在最終文字開始輸出前，看見目前 workflow 正在處理哪一段。外部自訂遊樂場服務執行 Agentic SDK workflow 時，SDK 會透過 `event_callback` 送出 stage event；Runner 收到開始事件後，顯示事件中的 `label`。

```python
{
    "type": "stage",
    "phase": "start",
    "status": "running",
    "stage": "retrieve",
    "label": "正在查找參考資料",
    "module": "retrieve",
    "module_class": "KeywordRetrieve",
    "workflow_id": "...",
    "session_id": "...",
}
```

維護者讀這個事件時，只需要把欄位分成兩類。`label` 是畫面要顯示的文字；`stage`、`module`、`module_class`、`workflow_id` 與 `session_id` 是排查時用來對回哪個 workflow、哪個模組與哪一次執行的證據。`state`、`output` 這類內部物件不屬於 AI Hub WebUI 的顯示契約，不應要求 Portal 端解析。

## SDK event 與畫面狀態責任邊界

Runner 階段狀態的責任分工如下。平台維護者排查時，先確認問題是設定沒有讀到、SDK 沒有送出事件、外部服務沒有轉成畫面狀態，還是最終回覆本身執行失敗。

| 主體 | 維護責任 | 可觀察證據 |
| --- | --- | --- |
| AI Hub | 保存與讀回 `python_source`、`workflow_name`、`description`，並依入口情境提供 handoff 或公開 config load。 | config save/load API 回應、SQL 中同一筆 Agent、工作區清單最近更新時間、公開 config load 回應。 |
| Agentic SDK | 依 `Workflow(stage_labels=...)` 與預設階段文字產生 stage event，並在 workflow 執行時交給 `event_callback`。 | SDK event、`stage`、`label`、`module_class`、`workflow_id`、`session_id`。 |
| 外部自訂遊樂場服務 | 用讀回的 `python_source` 建立 workflow，接收 SDK stage event，將 `type == "stage"` 且 `status == "running"` 的 `label` 顯示成 Runner 狀態。 | Runner 畫面狀態列、stream process event、Runner 模式、最後回覆輸出狀態。 |

這個分工讓 AI Hub 不需要知道每個 SDK 模組的畫面文案。畫面文字由 `Workflow(stage_labels=...)` 或 SDK 預設值決定；AI Hub 只保存 source，外部服務只顯示 SDK 事件提供的 `label`。

## stage_labels 保存與讀回

`stage_labels` 是 `Workflow` 建立參數的一部分。當外部自訂遊樂場服務把工作流設定保存回 AI Hub 時，它會隨 `python_source` 一起保存；使用者重新開啟同一筆 Agent 時，外部服務再從 AI Hub 讀回同一份 `python_source`，重建 workflow 並保留原本的階段提示文字。

```python
workflow = Workflow(
    workflow_name="WebUI 階段提示 Agent",
    stage_labels={
        "perceive": "正在理解你的問題",
        "retrieve": "正在查找參考資料",
        "action": "正在準備回覆",
    },
    perceive=PassThroughPerceive(),
    retrieve=KeywordRetrieve(...),
    action=DirectAnswerAction(),
)
```

如果 Runner 顯示的是舊文字，維護者不要先查 handoff token。應先確認 AI Hub 是否讀到最新 `python_source`、外部服務是否使用最新 config 建立 Runner、以及目前 Runner session 是否仍停留在舊設定。

## 維護檢查重點

| 問題類型 | 優先檢查 |
| --- | --- |
| Runner 只顯示正在產生回覆，沒有階段變化 | 外部服務是否收到 SDK stage event，stream 是否有由 stage event 轉成的 process event。 |
| 階段文字不符合預期 | `python_source` 中的 `Workflow(stage_labels=...)` 是否保存並讀回，Runner 是否用最新 config 建立 workflow。 |
| 可編輯 Runner 正常，公開唯讀 Runner 不正常 | 公開 config load 是否讀到同一份 `python_source`，公開 Agent 是否已保存可載入的 config。 |
| 有階段狀態但沒有最終回覆 | stage event 路徑已成立；下一步查 Action、模型端點、工具呼叫或 workflow 執行錯誤，不要把問題歸因到 handoff。 |
| 顯示舊文字 | 檢查 Agent 最近 config 保存時間、工作區清單更新時間、Runner session 是否仍使用舊 config。 |

## 查完這頁後應該留下什麼

查完這一頁後，平台維護者應該能留下同一筆 Runner 階段狀態問題的證據包：使用者入口、Runner 模式、`agent_id`、載入的 config 來源、`python_source` 中的 `stage_labels`、外部服務收到的 stage event 或 process event、畫面實際顯示文字、最後回覆是否開始輸出，以及若最終回覆失敗時對應的 Action、模型端點或工具錯誤。

當 `python_source` 可讀回、SDK 有送出 stage event、外部服務能把 `label` 顯示到 Runner 狀態列，且最終回覆開始後狀態能切換到回覆輸出時，Runner 階段狀態流程才算成立。