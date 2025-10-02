# 線上 JSON 格式化工具 (Online JSON Formatter)

這是一個基於 Spring Boot 的簡單 Web 應用程式，提供 JSON 字串的格式化和美化功能。使用者可以透過網頁介面貼上雜亂的 JSON 字串，並得到一個格式清晰、易於閱讀的版本。同時，它也提供了一個 REST API 端點供開發者以程式化方式使用。

## ✨ 功能

- **網頁介面**：提供一個簡單的文字區，用於輸入和顯示格式化後的 JSON。
- **即時格式化**：將未格式化的 JSON 字串轉換為帶有適當縮排和換行的標準格式。
- **語法驗證**：在格式化過程中隱含地檢查 JSON 語法是否正確。
- **REST API**：提供 `/api/format` 端點，方便與其他應用程式整合。

## 🛠️ 技術棧

- **後端**:
  - Java
  - Spring Boot (Web, MVC)
  - Maven (專案管理)
- **前端**:
  - HTML
  - CSS
  - JavaScript
- **測試**:
  - JUnit 5
  - Mockito

## 🚀 快速開始

請確保您的開發環境已安裝以下軟體：

- JDK 17 或更高版本
- Apache Maven

### 執行步驟

1.  **克隆儲存庫**
    ```bash
    git clone <your-repository-url>
    cd <repository-folder>
    ```

2.  **使用 Maven 執行**
    在專案根目錄下，執行以下指令：
    ```bash
    mvn spring-boot:run
    ```

3.  **開啟應用程式**
    應用程式啟動後，在瀏覽器中開啟 `http://localhost:8080` 即可看到操作介面。

## 🔌 API 使用說明

本專案提供了一個 RESTful API 端點來格式化 JSON。

### 格式化 JSON

- **URL** : `/api/format`
- **Method** : `POST`
- **Content-Type** : `application/json`

#### 請求內容 (Request Body)

請求主體需要包含一個名為 `rawJson` 的欄位，其值為您想要格式化的 JSON 字串。

**範例:**
```json
{
  "rawJson": "{\"name\":\"John Doe\",\"age\":30,\"isStudent\":false,\"courses\":[{\"title\":\"History\",\"credits\":3},{\"title\":\"Math\",\"credits\":4}]}"
}
```

#### 成功回應 (Success Response)

- **Status Code** : `200 OK`
- **Content-Type** : `application/json`
- **Body** : 格式化後的 JSON 物件。

**範例:**
```json
{
    "name": "John Doe",
    "age": 30,
    "isStudent": false,
    "courses": [
        {
            "title": "History",
            "credits": 3
        },
        {
            "title": "Math",
            "credits": 4
        }
    ]
}
```

#### 錯誤回應 (Error Response)

如果提供的字串不是有效的 JSON 格式，伺服器將回傳 `400 Bad Request`。