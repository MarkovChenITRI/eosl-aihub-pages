# 連線方式與責任邊界

## 這份文件會幫你完成什麼

當你是平台維護者，正在確認外部自訂遊樂場登入失敗、使用者 Agent 清單缺漏、保存後工作區清單更新失敗，或訪客模式行為和登入模式混在一起時，就看這一頁。這一頁固定 AI Hub、外部自訂遊樂場服務、已登入使用者與訪客模式之間的請求順序，讓你知道每一段請求應該送到哪個系統，以及成功後要在哪裡看到結果。

這一頁說明工作流設定的連線方式與責任邊界。工作流設定包含 `python_source`、`workflow_name` 與 `description`；外部自訂遊樂場服務負責使用者在畫面中如何編輯工作流、如何聊天，以及如何操作互動介面。需要保存或讀回 `agentic_playground_bundle.zip` 時，接到 [工作流保存與讀回](workflow-save-load.md)。

## 平台如何完成連線與工作流設定保存

外部自訂遊樂場服務連到 AI Hub 時，平台維護者先把流程拆成登入驗證、Agent 選擇、工作流設定讀取、工作流設定保存、短效檔案連結簽發與工作區顯示幾個檢查點。這些檢查點共用同一個 `agent_id`，讓外部服務、AI 服務端點、SQL 資料庫與 AI Hub 工作區都能對回同一筆 Agent。

### Playground 導航與 handoff 對照

Agentic SDK 文件已定義 AI Hub 已登入使用者進 Playground 的正式 handoff API：建立新 workflow 走 `POST /playground/aihub/navigation/builder`，開啟既有 Agent 走 `POST /playground/aihub/navigation/runner`。這兩個 endpoint 由 Playground 建立 server-side session，成功後 redirect 到 `/playground/builder` 或 `/playground/run`。AI Hub handoff 表單帶入短效 token、AI Hub API base URL 與必要的 `agent_id`；AI Hub 不把使用者明文密碼交給瀏覽器 handoff。`owner` 與 `agentId` 在公開唯讀 Runner URL 中用來定位已公開的 Agent；登入後的 Builder/Runner 編輯入口則以 handoff token 作為正式導航憑證。

| AI Hub 位置 | 使用者意圖 | 正式或預期入口 | AI Hub 狀態 |
| --- | --- | --- | --- |
| 產品導覽的自訂遊樂場 | 使用者先進 Playground，自行決定下一步 | `GET /playground` | 不傳 token、不預設新建 |
| 首頁的前往自訂遊樂場，未登入 | 訪客先進 Playground | `GET /playground` | 不傳 token、不預設新建 |
| 首頁的前往自訂遊樂場，已登入 | 建立新的 workflow | `POST /playground/aihub/navigation/builder`，成功後 `/playground/builder` | AI Hub 發短效 handoff token，由瀏覽器 top-level POST 到 Playground。 |
| 我的遊樂場的建立新工作流程 | 從 AI Hub 工作區建立新的 workflow | `POST /playground/aihub/navigation/builder`，成功後 `/playground/builder` | AI Hub 發短效 handoff token，由瀏覽器 top-level POST 到 Playground。 |
| 我的遊樂場清單的檢視 | 擁有者開啟自己的 workflow | `POST /playground/aihub/navigation/runner`，成功後 `/playground/run` | AI Hub 發短效 handoff token，並帶 `agent_id`。 |
| Gallery 卡片的進入遊樂場，擁有者或管理者 | 可編輯者從公開展示區開啟 workflow | `POST /playground/aihub/navigation/runner`，成功後 `/playground/run` | AI Hub 發短效 handoff token，失敗時 fallback 到公開唯讀 Runner。 |
| Gallery 卡片的進入遊樂場，非擁有者或未登入 | 使用者試用別人的 workflow | `GET /playground/run?owner=<owner>&agentId=<agent_id>` | Playground 以公開載入 API 讀取 config，進入 `aihub_readonly`；不顯示擁有者限定操作。 |

### 登入驗證與 Agent 選擇

外部自訂遊樂場服務的登入流程會先向 AI Hub 送出使用者帳號與密碼。AI Hub 回傳帳密驗證結果，外部服務再依結果顯示可操作的 Agent 清單。驗證成功後，外部服務向 AI Hub 取得該使用者擁有的 Agent，並在自訂遊樂場畫面中讓使用者選擇要操作哪一筆 Agent。

平台維護者在這一段要留下三個證據：外部服務的來源網域是否被允許、帳密驗證 API 是否回傳有效結果，以及 Agent 清單中是否包含使用者要操作的同一個 `agent_id`。

### 工作流設定讀取與保存

使用者選擇 Agent 後，外部自訂遊樂場服務用同一個 `agent_id` 讀取最近一次匯出的 Python 設定內容、工作流名稱與摘要。使用者在自訂遊樂場中更新工作流後，外部服務再把 `python_source`、`workflow_name`、`description` 與既有 `agent_id` 保存回 AI Hub。若 `python_source` 中包含 `Workflow(stage_labels=...)`，這些 Runner 階段提示文字也屬於同一份工作流設定內容，會隨 `python_source` 一起保存與讀回。

第一次保存新 Agent 時，外部服務省略 `agent_id`，AI Hub 建立新 Agent 並回傳 `201` 與新的 `agent_id`。後續保存同一筆 Agent 時，外部服務帶回 AI Hub 先前發出的 `agent_id`，AI Hub 更新同一筆 Agent 並回傳 `200`。

工作流設定保存成功後，AI Hub 工作區中的 Agent 工作流程清單應能顯示外部服務傳入的工作流名稱與摘要，並讓使用者選擇 Gallery 類型。類型預設為無分類；只有選成智慧大健康或智慧工廠次系統的 Agent 才會出現在公開 Gallery。若工作區顯示未更新，維護者應帶著同一個 `agent_id` 回查 API 回應、SQL 欄位與工作區清單重新讀取結果。

### Runner 唯讀模式與 AI Hub 保存邊界

非擁有者或未登入使用者從 Gallery 進入工作流時，AI Hub 直接導向公開唯讀 Runner URL，並由 Playground 依 `owner` 與 `agentId` 載入公開 config。這條路徑不需要 AI Hub 登入，也不提供保存、重新載入為擁有者或其他擁有者限定操作。維護者看到唯讀聊天模式異常時，先確認 Agent 是否已保存 public config 且已在 AI Hub 工作區選擇 Gallery 類型；看到登入帳號、Agent 清單、工作流設定讀寫或短效檔案連結失敗時，再回到 AI Hub 端點與資料責任追查。

## AI Hub 與外部服務的責任邊界

AI Hub 與外部自訂遊樂場服務的責任分工如下。平台維護者排查時，先找問題停在哪個責任主體，再決定要查 API、資料庫、外部服務還是 Azure 儲存體。

| 主體 | 維護責任 | 可觀察證據 |
| --- | --- | --- |
| AI Hub | 驗證帳密或 handoff token、檢查來源網域、產生與保存 `agent_id`、讀寫使用者自有 Agent 的工作流設定、保存 Gallery 類型、把導覽位置對應到 Agentic SDK handoff API、公開 `/playground` 或公開唯讀 Runner、更新工作區清單狀態。 | API 回應、SQL 中同一筆 Agent、工作區 Agent 清單、handoff endpoint 或公開入口路徑。 |
| 外部自訂遊樂場服務 | 顯示登入畫面、承接公開 `/playground`、`/playground/aihub/navigation/builder`、`/playground/aihub/navigation/runner`、控制工作流載入與聊天/編輯狀態、保存外部畫面狀態、讓使用者操作工作流，並把 Agentic SDK 的 stage event 顯示成 Runner 即時狀態。 | 外部服務畫面、handoff request、工作流編輯狀態、Runner 模式、stage event 或由它轉成的 process event。 |
| Azure 儲存體 | 保存同一個 `agent_id` 底下的固定工作流檔案包。 | 上傳/下載回應、檔案儲存位置、短效連結到期時間。 |

`agent_id` 由 AI Hub 產生與發放。外部自訂遊樂場服務保存並帶回 `agent_id`，讓後續讀取、更新與檔案保存都對回同一筆 Agent。AI Hub 把工作流設定保存成 Python 設定內容與顯示資料；工作流內部節點語意、檔案包內容與檢索索引還原由外部服務、Agentic SDK 與 SemanticRetriever 處理。

## 工作流設定 API 參考

表 1 整理工作流設定使用的端點。維護者使用這張表時，先確認請求來源網域、使用者帳號、`agent_id` 與回應狀態，再把結果對回 SQL 與工作區畫面。

| 操作 | 方法與端點 | 送出的主要內容 | 成功結果 |
| --- | --- | --- | --- |
| 驗證登入帳密 | `POST /api/playground/auth/verify` | `username`、`password` | 回傳 `valid=true` 或 `valid=false`，成功時包含 `username` 與帳戶層級 `display_name`。 |
| 驗證 handoff token | `POST /api/playground/handoff/verify` | `token` | 回傳 token 對應的 `username`、`agent_id` 與帳戶層級 `display_name`。 |
| 列出使用者的所有 Agent | `POST /api/playground/agents` | `username`、`password`；或 handoff token 對應的同一帳號 | 回傳 `items` 陣列，每個 Agent 包含 `agent_id`、`workflow_name`、`description`、`has_playground_config`、`playground_exported_at`、`created_at`、`updated_at`。 |
| 讀取 Agent 最近設定 | `POST /api/playground/agents/<agent_id>/config/load` | `username`、`password`；或 handoff token 對應的同一帳號與 `agent_id` | 回傳該 Agent 最近一次匯出的 Python 設定內容、`workflow_name`、`description` 與匯出時間。 |
| 讀取公開 Agent 設定 | `POST /api/playground/agents/<agent_id>/config/public/load` | 空 JSON body；不需要登入帳密 | 回傳已公開 Agent 的 Python 設定內容、`workflow_name`、`description` 與匯出時間，供公開唯讀 Runner 使用。 |
| 保存或新建 Agent 設定 | `POST /api/playground/config/save` | `username`、`password`；或 handoff token；另帶 `python_source`、`workflow_name`、`description`，更新時提供 `agent_id` | 更新既有 Agent 時回傳 `200`；第一次建立新 Agent 時回傳 `201` 與新 `agent_id`。 |

外部自訂遊樂場服務呼叫這些 API 時，帶入合法的來源網域。`display_name` 是帳戶層級顯示名稱，會用於 Playground 問候與身份顯示；`workflow_name` 與 `description` 是 Agent 層級 metadata，會同步到 AI Hub 工作區清單，讓維護者與使用者能辨識每一筆 Agent 的目前狀態。Agent 是否出現在 Gallery 由工作區清單中的 Gallery 類型決定，無分類不會出現在公開 Gallery。

## 查完這頁後應該留下什麼

查完這一頁後，平台維護者應該能留下同一筆連線問題的證據包：使用者進入的是公開 `/playground`、Builder handoff、Runner handoff，還是公開唯讀 Runner；外部服務來源網域、使用者帳號、`agent_id`、呼叫過的工作流設定 API、回應狀態、SQL 中同一筆 Agent 的工作流名稱、摘要與 Gallery 類型，以及 AI Hub 工作區清單顯示結果。

1. 登入驗證問題：留下來源網域、帳密驗證回應與錯誤格式。
2. Agent 清單問題：留下使用者帳號、Agent 清單回應與缺漏的 `agent_id`。
3. 工作流設定保存問題：留下 請求中的 `workflow_name`、`description`、`python_source`、`agent_id` 與回應狀態。
4. 工作區顯示問題：留下 AI Hub 工作區清單畫面、同一筆 Agent 的資料庫欄位與最近匯出時間。
5. Runner 階段狀態問題：留下 Runner 模式、`python_source` 中的 `stage_labels`、外部服務收到的 stage event 或 process event，以及最終回覆是否已開始輸出。

若問題已確認停在 Runner 執行期間的階段顯示，下一步接到 [Runner 階段狀態](runner-stage-status.md)。若問題已確認停在工作流檔案包的保存或讀回，下一步接到 [工作流保存與讀回](workflow-save-load.md)。