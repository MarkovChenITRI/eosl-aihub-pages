# Runner 對話記憶上下文

## 這份文件會幫你完成什麼

當使用者回報外部自訂遊樂場 Runner 接不住前文、重新開啟 Agent 後上下文不如預期、或把對話記憶問題誤判成 AI Hub handoff / bundle save-load 失敗時，就看這一頁。

這一頁接續 [連線方式與責任邊界](connection-and-responsibility.md) 與 [工作流保存與讀回](workflow-save-load.md)。AI Hub 負責保存與讀回同一份 `python_source`；外部自訂遊樂場服務負責用這份 source 建立 Agentic SDK `Workflow`、配置 memory，並在同一個 Runner session 中把使用者訊息交給同一個 workflow 執行。

## memory 上下文的責任邊界

Agentic SDK 的多輪對話不是由 AI Hub 資料庫逐 turn 保存。AI Hub 保存的是可還原 workflow 的設定與檔案包；真正承接對話上下文的是外部服務啟動 Runner 後建立的 memory 物件。

```python
from agentic_sdk import InContextMemory, Workflow

memory = InContextMemory()

workflow = Workflow(
    workflow_name="多輪問答 Agent",
    memory_type=memory,
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

維護者讀這段 source 時，重點是 `memory = InContextMemory()` 與 `memory_type=memory` 是否同時存在。前者建立本次 Runner 要使用的對話記憶物件；後者把這個物件交給 `Workflow`。後續同一段 Runner 對話要延續上下文，外部服務必須持續使用同一個 workflow / memory 組合。

## `memory_type` 可接受的形式

Agentic SDK 的 `Workflow(memory_type=...)` 支援幾種形式。平台維護者不需要替外部服務選實作，但要能讀懂 source 中的意義。

| source 寫法 | 意義 | 維護觀察重點 |
| --- | --- | --- |
| `memory_type="in_context"` | 使用 SDK 預設的 session 內對話工作記憶。 | 適合簡單 Runner，但 source 不會暴露可直接觀察的 memory 變數。 |
| `memory = InContextMemory()` + `memory_type=memory` | 明確建立 memory 物件並交給 workflow。 | 推薦給教學、Playground 產生 source 與除錯；可直接檢查 `memory.turns`。 |
| `memory_type="persistent"` | 使用 SDK 內建可搜尋的記憶策略。 | 外部服務仍要確認資料保存位置與生命週期，不應假設 AI Hub 逐 turn 保存。 |
| 自訂 memory 物件 | 外部服務提供自己的 `MemoryStore` 實作。 | 要留下外部服務 memory 實作、儲存位置與 session 對應規則。 |

## 排查多輪對話時先看哪裡

平台維護者排查時，先把問題切成三段：AI Hub 是否讀回同一份設定、外部服務是否建立同一個 memory、SDK 是否在同一個 session 中累積 turns。

| 問題類型 | 優先檢查 |
| --- | --- |
| 重新開啟 Agent 後 workflow 設定不見 | AI Hub config load / bundle load 是否讀回同一份 `python_source` 與檔案包。 |
| 同一個 Runner 畫面第二輪接不住前文 | 外部服務是否重建了新的 `Workflow` 或新的 `InContextMemory()`；同一段 Runner session 應共用同一個 memory。 |
| 不同使用者或不同任務互相污染 | 外部服務是否把多個使用者共用到同一個 memory 物件或同一個 `session_id`。 |
| `memory.turns` 沒有累積 | 檢查 `Workflow(memory_type=memory)` 是否被正確建立，以及每次 `run()` 是否使用預期的 `session_id`。 |
| stage event 正常但上下文錯誤 | stage event 路徑只證明 workflow 有執行；下一步應查 memory 與 session，而不是 handoff token。 |

## 查完這頁後應該留下什麼

查完這一頁後，平台維護者應該能留下同一筆多輪上下文問題的證據包：使用者入口、Runner 模式、`agent_id`、載入的 `python_source`、source 中的 `memory_type` 寫法、外部服務是否重用同一個 workflow / memory、每輪使用的 `session_id`、`memory.turns` 的長度與角色順序、以及是否同時存在 stage event 與最終回覆。

當 AI Hub 能讀回同一份 source、外部服務能用同一個 memory 物件承接同一段 Runner session、且第二輪後 `memory.turns` 出現 `user -> assistant -> user -> assistant` 順序時，多輪對話上下文流程才算成立。