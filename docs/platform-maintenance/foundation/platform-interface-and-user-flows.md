# 介面設計與使用流程

## 這份文件會幫你完成什麼

AI Hub Portal 的介面層，是平台能力被使用者看見、操作與治理的入口。公開區、AI資源發布工作區與帳戶與平台管理區都屬於同一個 Portal 結構；平台維護者看見 WebUI 畫面、URL、模板檔案、使用者回報或異常現象時，先要判斷這個現象落在哪一段介面責任，再依責任邊界回查 API、資料或部署狀態。

介面層把平台能力拆成三個產品情境：公開區讓未登入使用者理解平台提供的產品、模型、方案與聯繫資訊；AI資源發布工作區讓登入使用者維護產品、模型、方案、部署與代理入口；帳戶與平台管理區讓維護者治理使用者、群組、資源與發布狀態。這三個情境共同形成 Portal 的操作面，圖中的頁面節點呈現它們在產品中的相對位置，表格則把每個頁面落到模板入口、可辨識畫面內容與介面責任。

## 介面層如何區分平台角色

公開區、AI資源發布工作區與帳戶與平台管理區是 Portal 對不同角色開放平台能力的三種介面邊界。公開區承接未登入瀏覽與對外內容，使用者在這裡看到平台有哪些產品、模型、方案與聯繫方式；AI資源發布工作區承接登入後的內容發布、部署與代理操作；帳戶與平台管理區承接帳戶設定、群組治理、使用者治理與資源監測。三區共同保證同一個 Portal 可以同時服務瀏覽、操作與治理，並讓個人操作和平台治理保有各自的畫面責任。

圖與表格分別固定介面層的兩種事實。圖 1 呈現頁面在 Portal 中的相對位置與承接關係，讓維護者看出某個畫面是在公開內容、AI資源發布工作區還是帳戶與平台管理區脈絡中出現；各小節表格則保存每個畫面的模板入口、主要辨識內容與介面用途。若手上是畫面或使用者回報，先用畫面文案與元件判斷區域；若手上是模板檔案，先用 `檔案位置` 對回頁面；若要判斷後續責任，則看 `用途` 是否只停留在介面呈現，或需要交給 API、資料庫或雲端部署資源確認。

## 平台能力總覽

AI Hub Portal 的介面拓撲，從公開區開始揭露平台能力，經由 AI資源發布工作區把能力轉成登入後操作，再由帳戶與平台管理區承接跨使用者與跨資源治理。圖中的英文名稱是頁面節點，用來標示頁面在三個區域中的相對位置；線條只表示相鄰頁面與操作面承接關係，不代表資料流、權限模型或完整使用者旅程。API 呼叫、資料權威與部署狀態仍由服務層、資料層與雲端資源層確認，介面圖只負責說明 Portal 如何把平台能力呈現在不同角色面前。

```mermaid
architecture-beta
    group public(cloud)[Public]
    service home(server)[Home] in public
    service product(server)[Product] in public
    service cards(server)[ModelCards] in public
    service gallery(server)[Solutions] in public
    service contact(server)[Contact] in public

    group publish(cloud)[AI資源發布工作區]
    service login(server)[Login] in publish
    service products(server)[Products] in publish
    service models(server)[Models] in publish
    service solutions(server)[Solutions] in publish
    service deployments(server)[Deployments] in publish
    service agents(server)[Agents] in publish

    group manage(cloud)[帳戶與平台管理區]
    service account(server)[Account] in manage
    service users(server)[Users] in manage
    service groups(server)[Groups] in manage
    service monitoring(server)[Monitoring] in manage

    home:R -- L:product
    product:R -- L:cards
    cards:R -- L:gallery
    gallery:R -- L:contact

    login:T -- B:home
    login:R -- L:products
    login:B -- T:account
    products:T -- B:product
    products:R -- L:models
    models:T -- B:cards
    models:R -- L:solutions
    solutions:T -- B:gallery
    solutions:R -- L:deployments
    deployments:R -- L:agents
    account:R -- L:users
    users:R -- L:groups
    groups:R -- L:monitoring
```

圖 1、頁面分區與相對位置。

後續五組頁面對應 Portal 中五種被維護的介面對象：產品、模型、方案、部署與代理、身份與治理。每一組都可能跨越公開區、AI資源發布工作區或帳戶與平台管理區，表示同一個平台對象會依角色不同而有不同的呈現方式。表格只壓縮模板入口、畫面辨識點與用途；真正的設計判斷在於理解同一對象如何從公開瀏覽、個人操作一路延伸到平台治理。

### 產品與硬體規格

產品與硬體規格頁群從產品提供者在 AI資源發布工作區發布產品開始。發布者需要提供產品名稱、型號、加速器、記憶體等細部規格，並依 Vendor 軟體堆疊補上系統標準安裝手冊，讓使用者與模型開發方有一致的使用與開發基準。平台再把已發布產品呈現在 **Product**，讓公開使用者能比較可用設備，也讓模型發布方能回到同一組硬體規格選定部署目標。這組頁的責任因此會把產品供應、硬體規格、安裝基準與後續模型部署條件接成同一條可追溯的介面主線。

表 1 把產品與硬體規格的介面責任拆成模板入口、畫面辨識點與用途。維護者看這張表時，應確認同一個產品在公開瀏覽、單品內容與內部資產維護中各自以什麼畫面形式出現。

表 1、產品與硬體規格頁面。

| 檔案位置 | 內容 | 用途 |
| --- | --- | --- |
| `pages/products/catalog.html` | **產品目錄**：公開產品列表與瀏覽入口。 | 這一頁是產品公開總頁，先用來確認目前有哪些產品可供瀏覽。 |
| `pages/products/detail.html` | **產品詳情**：breadcrumb、`product-tabs`、產品影像、規格表、型號與 SKU 欄位。 | 這一頁用來查看單一產品的個別內容與對應硬體規格。 |
| `pages/admin/products.html` | **產品管理**：產品統計卡、產品列表、部署類型、處理器與加速器規格、記憶體規格與新增產品輸入欄位。 | 這一頁用來維護平台產品資產，承接產品與硬體規格在管理面的維護工作。 |

### 模型與發布

模型與發布頁群從模型發布方在 AI資源發布工作區送出上架資料開始。發布者新增模型時會固定部署裝置、模型架構與 OpenAI SDK 支援資訊，平台再依這些輸入交付發布憑證。後續 GitHub 與 CI/CD 自動交付流程會建置正式映像、推送 Azure 容器登錄，並把 `sdk_used` 與 `registry_location` 更新到模型小卡。完成發布後，**ModelCards** 承接公開模型瀏覽與部署選擇，帳戶與平台管理區則讓維護者觀察容器化結果是否回到同一筆模型小卡。

表 2 壓縮模型與發布頁群的三種介面角色。維護者可以用它辨識同一張模型卡目前出現在公開瀏覽、上架輸入還是容器化結果觀察位置，再決定畫面問題是否已超出介面層，需要轉往 API 或資料責任。

表 2、模型與發布頁面。

| 檔案位置 | 內容 | 用途 |
| --- | --- | --- |
| `pages/cards.html` | **模型小卡清單**：公開模型小卡列表與篩選入口。 | 這一頁是模型與發布的公開總頁，先用來確認可瀏覽的模型卡內容。 |
| `pages/workspace/workspace.html` | **我的模型**：模型統計卡、`新增模型`、`取得開發模板`、模型摘要區、部署設定欄位與模型欄位區。 | 這一頁用來承接模型發布方在 AI資源發布工作區中的上架資料與發布前輸入內容。 |
| `pages/admin/monitoring.html` | **資源監測**：`dashboardSearchInput`、filter select 與正式主線資源盤點區。 | 這一頁用來監看正式主線資源與模型小卡回寫結果。 |

### 方案與內容

方案與內容頁群從方案提供者在 AI資源發布工作區維護方案內容與公開設定開始。提供者把模型、硬體與應用情境整理成可展示的方案，平台再透過 **Solutions** 呈現公開方案列表與方案詳情，讓使用者先看見團隊能把平台能力組合成什麼成果。這組頁讓方案成為能力展示與合作轉換的入口；當使用者理解方案價值後，才會進一步透過 **Contact** 找到對應團隊洽談合作。

表 3 把方案與內容的生命週期壓縮成可回查的模板與畫面入口。維護者看這張表時，應先判斷問題是出在對外敘事、公開方案內容、單一方案呈現，還是工作區維護入口，再決定是否需要往服務或資料層追查方案資料來源。

表 3、方案與內容頁面。

| 檔案位置 | 內容 | 用途 |
| --- | --- | --- |
| `pages/public/home.html` | **首頁**：導覽列、站點導覽連結、hero title、hero desc、`瞭解更多` CTA、模型整合方式 tabs 與 `開始試用` CTA。 | 這一頁用來承接網站內容入口，先讓維護者確認首頁有哪些主要導覽與內容模組。 |
| `pages/public/contact.html` | **聯繫**：平台聯繫資訊與聯繫頁內容。 | 這一頁用來維護平台聯繫資訊，屬於網站內容頁的一部分。 |
| `pages/public/gallery.html` | **方案展示櫃**：公開方案列表、分類、AI工作流整合區與方案瀏覽入口。 | 這一頁是方案與內容的公開總頁，先用來查看有哪些方案與已分類公開的 Agent 工作流程可供展示。 |
| `pages/public/solution.html` | **方案詳情**：單一方案內容、對應模型卡與裝置資訊。 | 這一頁用來查看單一方案的個別內容與展示結果。 |
| `pages/workspace/solutions.html` | **我的方案**：方案統計卡、方案列表、公開設定、更新時間與新增方案輸入欄位。 | 這一頁用來維護 AI資源發布工作區中的方案內容與公開設定。 |

### 部署與代理

部署與代理頁群建立在硬體產品與模型發布兩條主線之上。使用者在 AI資源發布工作區取得部署與授權入口後，可以依模型小卡與產品安裝指引啟動模型服務，並用 OpenAI SDK 指向相容 API。平台再透過 Agentic SDK 把模型服務組成可執行的 Agent workflow，AI資源發布工作區也承接代理建立、體驗與工作區管理。

表 4 把部署與代理的介面責任拆成公開入口、體驗入口、部署入口與工作區管理入口。維護者看這張表時，應先確認畫面是否只是引導使用者進入下一個能力，或已經呈現了需要 API、授權或執行紀錄支撐的狀態。

表 4、部署與代理頁面。

| 檔案位置 | 內容 | 用途 |
| --- | --- | --- |
| `pages/public/agents.html` | **首頁（Agentic SDK）**：Agentic SDK 應用案例與公開入口內容。 | 這一頁用來承接代理與 SDK 相關的公開入口，先讓維護者確認代理主題如何對外呈現。 |
| `pages/agent_detail.html` | **AI工作流詳情**：公開 Agent 工作流名稱、摘要、工作流程模組、使用案例內容與 Runner 入口。 | 這一頁用來查看單一公開 Agent 工作流；未登入與非擁有者進公開唯讀 Runner，擁有者或管理者可切換檢視/編輯模式後進可編輯 Runner。 |
| `pages/playground.html` | **Playground**：Framework 模擬器規劃中的公開頁面。 | 這一頁用來提供 Playground 的公開入口與體驗位置。 |
| `pages/workspace/deployments.html` | **我的部署**：空狀態標題、`瀏覽模型卡` CTA 與 `上架自己的模型` CTA。 | 這一頁用來承接 AI資源發布工作區中的部署入口，將使用者導向模型卡與模型上架流程。 |
| `pages/workspace/agent_playground.html` | **我的遊樂場**：工作流程統計卡、工作流名稱與摘要、Gallery 類型、檢視與刪除操作，以及 `建立新工作流程`。 | 這一頁用來承接 `/publish/agents` 的 Agent 工作流程管理入口；`/publish/agents/playground/new` 進 Builder，`/publish/agents/playground/<agent_id>` 進既有 Runner。 |

### 帳號與平台治理

帳號與平台治理頁群先由 **Login** 固定身份進入點，再由帳戶與平台管理區維護平台內使用者、角色與群組關係。當部署、模型、方案與群組活動累積後，帳戶與平台管理區會把帳號與群組層級的 Budget / Credit、部署數與資產統計呈現給維護者，讓雙方用量能被持續觀察。這些統計目前是平台計量與治理基礎，後續才有條件延伸成計價、分潤或其他經濟效益模型。這組頁的重點因此在於把身份、組織關係與可累積的用量證據放在同一個治理操作面。

表 5 把身份與治理介面拆成進入、個人帳戶、平台治理與資產檢視幾個入口。維護者看這張表時，要先判斷畫面正在呈現身份狀態、個人設定、群組治理還是使用者資產，再把需要查證的角色、群組或歷程證據交給 API 與資料庫頁。

表 5、帳號與平台治理頁面。

| 檔案位置 | 內容 | 用途 |
| --- | --- | --- |
| `pages/login.html` | **登入**：`登入`、`註冊` tabs、帳號密碼欄位與註冊輸入欄位。 | 這一頁用來承接身份進入點，先確認登入與註冊流程的畫面入口。 |
| `pages/workspace/resources.html` | **帳戶管理**：帳戶層級 `display_name`、`status`、所屬群組文字、密碼調整欄位。 | 這一頁用來維護個人帳戶資料與密碼設定；`display_name` 是使用者顯示名稱，與 Agent 的 `workflow_name` 不同。 |
| `pages/admin/platform.html` | **使用者管理**：`使用者`、`公開方案 Catalog`、`Model Card 資料庫`、`群組` 摘要卡與使用者列表。 | 這一頁用來查看平台內的使用者治理摘要與使用者列表。 |
| `pages/admin/groups.html` | **群組管理**：`groupsSummary`、`groupsMessage`、`groupsDirectory`、`groupWorkspace` 與建立群組欄位。 | 這一頁用來維護群組主檔、成員與群組工作區。 |
| `pages/admin/user_assets.html` | **使用者資產**：單一使用者資產頁內容。 | 這一頁用來查看單一使用者名下的方案、群組與模型資產對應。 |

## 介面層完成後要保證什麼

完成本頁後，平台維護者應理解介面層的核心責任：它把平台能力呈現成不同角色可看見、可操作、可治理的頁面，並把畫面問題整理成相鄰層可以接手的證據。介面頁負責說明畫面位置、模板呈現與頁群操作面；API 行為、資料權威與雲端部署狀態則分別回到服務層、資料層與雲端資源層確認。

第一，介面區域不能和權限或資料事實混淆。公開區、AI資源發布工作區與帳戶與平台管理區說明的是 Portal 如何呈現不同角色的操作面；登入狀態、角色授權與資料可見性仍要由服務與資料層確認。

第二，模板與畫面內容只能證明呈現與入口。`檔案位置` 和畫面元件能協助維護者定位介面實作，但不能單獨證明 API 已正確寫回、資料表已有正式紀錄，或雲端部署資源已完成更新。

第三，介面異常要轉成相鄰層可查證的證據。當畫面內容缺漏、狀態顯示不一致或入口導向錯誤時，維護者應帶著頁面區域、模板位置、畫面辨識點與用途判斷，往 API 服務、資料庫責任或雲端部署資源頁確認真正的行為、資料或部署責任。
