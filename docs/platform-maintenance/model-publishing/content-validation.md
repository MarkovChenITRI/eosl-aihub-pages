# 發布憑證與登錄資訊

## 這份文件會幫你完成什麼

當平台維護者需要交付**發布憑證**（`model_card.yaml`），並確認 GitHub 範例程式庫中的 CI/CD自動交付流程能支援模型發布方把模型 API 上架到 Azure 容器登錄（ACR），再把結果更新到模型小卡時，就看這一頁。

這一頁聚焦在兩件事：第一，平台要頒發給模型發布方的發布憑證欄位格式是什麼；第二，平台隨同 GitHub 範例程式庫提供的 CI/CD自動交付流程，會如何依這份發布憑證完成登入、驗證、映像建置、推送與回報，並把結果更新到發布結果欄位（`sdk_used`、`registry_location`）。

## 平台如何交付發布憑證與更新發布結果

接下來的各節會依序說明平台如何準備發布憑證中的欄位、讓 CI/CD自動交付流程取用這些欄位完成映像推送，並把發布結果更新到模型小卡。

### 發布憑證由哪些欄位組成

發布憑證是 CI/CD自動交付流程開始前要先固定的交付檔。它會保留部署裝置、模型架構資料與 OpenAI SDK 支援資訊，並補上 Azure 容器登錄登入資訊、登錄位置組成欄位與結果回傳 API 位置。欄位如下：

```yaml
card_id: "<card-id>"
registry_name: "<registry-name>"
acr_username: "<username>"
acr_password: "<password>"
publisher_id: "<publisher-id>"
product_specification_option_id: "<product-specification-option-id>"
model_id: "<對應模型識別>"
sdk_used: "<true|false，是否需要 OpenAI 相容呼叫介面驗證>"
image_tag: "<本次允許推送的完整版本標記>"
callback_url: "<AI服務端點的結果回傳 API 位置（callback）>"
callback_token: "<callback-token>"
```

### CI/CD自動交付流程的操作與規範

本節說明 CI/CD自動交付流程（GitHub Actions）如何使用這份發布憑證完成驗證、映像推送與結果回報。接下來會先說流程怎麼用發布憑證中的欄位，再說正式映像如何推送到 Azure 容器登錄（ACR），最後說哪些結果會透過回傳資料（callback payload）回到平台並寫入發布結果欄位。

### 正確性測試：OpenAI SDK 相容性驗證

若發布憑證中的 `sdk_used` 宣告需要 OpenAI SDK 驗證，CI/CD自動交付流程就必須進入對應查驗。依目前流程，實際會核對下列兩個介面。

1. `GET /healthz`：用來確認容器已可接受請求；最小回應形狀可寫成：

    ```json
    {
      "status": "ready"     // 可能值：ready、not_ready
    }
    ```

2. `GET /v1/models`：請求（request）不帶內容本文（body）。回應（response）必須只回傳此容器唯一模型，且模型 ID 必須與發布憑證中的 `model_id` 一致。最小回應形狀可寫成：

    ```json
    {
      "object": "list",
      "data": [
        {
          "id": "<model-id>",
          "object": "model",
        }
      ]
    }
    ```

當這兩項查驗完成後，CI/CD自動交付流程才得把這次 OpenAI SDK 驗證結果寫成 `sdk_used`，作為後續結果回報與資料寫入的正式依據。

> 這裡的查驗只確認 OpenAI 相容介面是否存在，並不代表 OpenAI SDK 已可實際使用；實際支援狀況仍須由模型提供方自行維護與說明。


### 映像建置與推送：推送正式映像至 Azure 容器登錄（ACR）

這一段說明 CI/CD自動交付流程如何把 GitHub 程式碼儲存庫內的交付內容推送到 Azure 容器登錄（ACR）。發布憑證會提供登入資訊與登錄位置組成欄位；流程完成登入、建置與推送後，再把實際登錄位置（`registry_location`）回寫到模型小卡。

先登入目標 Azure 容器登錄（ACR）：

```bash
docker login <registry-name>.azurecr.io -u <acr_username> -p <acr_password>
```

先建置正式映像：

```bash
docker build -t <registry-name>.azurecr.io/<publisher_id>/<product_specification_option_id>/<model_id>:<image_tag> .
```

再推送正式映像至 Azure 容器登錄（ACR）：

```bash
docker push <registry-name>.azurecr.io/<publisher_id>/<product_specification_option_id>/<model_id>:<image_tag>
```

推送成功後，CI/CD自動交付流程才得把這次實際推送成功的完整登錄位置寫成登錄位置（`registry_location`），作為後續結果回報與資料寫入的來源。這個登錄位置會回寫到模型小卡的發布結果欄位；Azure 容器登錄則負責保存正式映像。

> 模型發布方第一次上架模型前，平台維護者需要先建立可推送到指定 Azure 容器登錄的 ACR token，並把產生的 `acr_username` 與 `acr_password` 放入發布憑證。建立方式如下：
>
> ```bash
> az acr scope-map create --name pushScope --registry <registry-name> --repository <repo-name> push
> az acr token create --name demoPushToken --registry <registry-name> --scope-map pushScope
> az acr token credential generate --name demoPushToken --registry <registry-name>
> ```

### 回報登錄結果至平台 API

這一步說明 CI/CD自動交付流程如何把驗證與映像推送結果送回平台。`callback_url` 指出結果回傳 API，授權標頭提供呼叫權限，JSON body 則帶出要寫回模型小卡的 `sdk_used` 與 `registry_location`。

先把結果送到發布憑證指定的結果回傳 API（`callback_url`）。在授權標頭（`Authorization` header）放入 `model_card.yaml` 的 `callback_token` 欄位值，作為這次結果回寫的權杖：

```http
POST /api/model-card-publish/callback
Authorization: Bearer <callback-token>
Content-Type: application/json
```

再在請求內容中填入這次要回寫到模型小卡的欄位：

```json
{
  "card_id": "<card-id>",
  "sdk_used": true,
  "registry_location": "<registry-location>"
}
```

### 發布結果欄位寫入後要保證什麼

走到這裡，前面的內容已經定義好發布憑證欄位、映像推送位置與結果回寫方式。接下來要確認的，是這些結果在後續查證時是否仍然對得回同一筆上架資料。

1. OpenAI SDK 驗證、映像推送與結果回寫，必須都依附同一份發布憑證。
2. 寫回的 `sdk_used` 與 `registry_location`，必須都對回同一張模型小卡。
3. 後續查證這筆模型是否已完成發布時，必須還能沿用同一組發布憑證與發布結果欄位。

當這三件事成立時，發布結果就能支援後續**模型清單**顯示、裝置頁面「支援模型」列表與平台查證。