---
name: ragic
description: Use when working with Ragic database platform - calling Ragic HTTP API, writing Ragic JavaScript workflow scripts, designing Ragic form workflows, or debugging Ragic integrations. Also use when the user asks about reading/writing Ragic data, setting up webhooks, executing action buttons, or scripting in Ragic's workflow engine.
---

# Ragic 開發指南

## 概覽 Overview

Ragic 是雲端資料庫平台，提供 RESTful HTTP API 和伺服器端 JavaScript 工作引擎。

- 詳細 API 規格 → 見 `api-reference.md`
- JS 工作引擎腳本 → 見 `js-engine.md`

## 快速索引 Quick Reference

| 操作 | 方法 | 路徑 |
|------|------|------|
| 讀取清單 | GET | `/{account}/{sheet}?v=3&api` |
| 讀取單筆 | GET | `/{account}/{sheet}/{id}?v=3&api` |
| 新增記錄 | POST | `/{account}/{sheet}?v=3&api` |
| 修改記錄 | POST | `/{account}/{sheet}/{id}?v=3&api` |
| 刪除記錄 | DELETE | `/{account}/{sheet}/{id}?v=3&api` |
| 執行 Action Button | POST | `/{account}/{sheet}/{id}?action={buttonId}&api` |
| 查詢篩選 | GET | `/{account}/{sheet}?v=3&api&where={fieldId},{op},{value}` |

**伺服器位置**：依帳號所在選 `www`、`na3`、`ap5` 或 `eu2`
Base URL：`https://{server}.ragic.com`

## 認證 Authentication

### 方法 1：API Key（推薦）

```js
const apiKey = 'YOUR_API_KEY'; // 從 Personal Settings 取得
const headers = {
  'Authorization': 'Basic ' + Buffer.from(apiKey).toString('base64'),
  'Content-Type': 'application/json'
};
```

或用 URL 參數（不推薦，僅備用）：
```
GET https://www.ragic.com/{account}/{sheet}?api&v=3&APIKey=YOUR_KEY
```

### 方法 2：帳號密碼

```js
const credentials = Buffer.from('user@email.com:password').toString('base64');
const headers = { 'Authorization': 'Basic ' + credentials };
```

> **安全提醒**：API Key 擁有完整讀寫權限，請勿硬寫在前端程式碼中。建議為 API 建立專屬使用者帳號。

## 如何找到 API Endpoint

1. 開啟 Ragic 表單頁面
2. 點右上角 **≡ 選單 → Form Settings → APIs**
3. 或直接看網址列：`https://www.ragic.com/{account}/{tab}/{sheetIndex}`

## 名詞定義 Glossary

以下以 `https://www.ragic.com/testAP/testForm/1/3` 為例說明各名詞：

### 結構性名詞

| 名詞 | 範例值 | 定義 |
|------|--------|------|
| **APName** | `testAP` | 使用者的資料庫名稱 |
| **Path** | `/testForm` | 表單所在的頁籤，**前面斜線不可省略** |
| **SheetIndex** | `1` | 表單在頁籤內的索引 |
| **pathToForm** | `/testForm/1` | Path + SheetIndex 的組合，通常作為單一參數使用 |
| **rootNodeId / recordId** | `3` | 這筆資料（紀錄）的 ID，可透過 `getNewNodeId(keyFieldId)` 或 `getOldNodeId(keyFieldId)` 取得 |

### 欄位相關名詞

| 名詞 | 定義 |
|------|------|
| **Key field** | 表單的主鍵，`keyFieldId` 是主鍵欄位的 ID，可在資料庫欄位定義文件中找到，或用 `entry.getKeyFieldId()` 取得 |
| **fieldId** | 欄位的 ID，`fieldName` 是欄位的名稱。fieldId 可在資料庫欄位定義文件中找到，或在設計模式選取欄位後 → 欄位設定 → 基本，**欄位名稱下方的七碼數字**即為 fieldId |
| **Subtable** | 子表格，可想像成表單下方還有一張表單，也有自己的 keyFieldId、fieldId、rootNodeId |
| **subtableId** | 子表格的 ID，可在資料庫欄位定義文件中找到，概念上等同於子表格的 keyFieldId |
| **subtableRowIndex** | 子表格某列的索引，通常用迴圈指定 |
| **subtableFieldId** | 子表格欄位的 ID，可在資料庫欄位定義文件中找到，概念上等同於子表格的 fieldId |
| **subtableRootNodeId** | 子表格某列的 ID，可透過 `getSubtableRootNodeId(subtableId, subtableRowIndex)` 取得，概念上等同於子表格的 rootNodeId |

### JS 工作引擎預定義變數

| 變數 | 說明 |
|------|------|
| **db** | 資料庫操作物件，透過 `getAPIQuery()` 取得查詢物件 |
| **query** | 表單查詢物件，用於檢索或操作記錄 |
| **entry** | 單筆記錄物件，用 `setFieldValue()`/`getFieldValue()` 操作欄位 |
| **param** | 取得欄位新舊值，Pre/Post-workflow 中可用（Global Workflow 不可用） |
| **response** | 設定工作流程執行後的狀態與訊息 |
| **user** | 當前使用者物件，可取得 email、名稱、群組 |
| **mailer** | 撰寫並發送 email 通知 |
| **util** | 發送 HTTP 請求、管理檔案下載/上傳 |
| **approval** | 建立或取消記錄的審核流程 |
| **approvalParam** | 審核 Workflow 中的預定義變數 |

---

## 角色定位：三種協助模式

| 使用者需求 | Claude 的角色 |
|-----------|--------------|
| 「幫我寫呼叫 Ragic 的程式」 | 提供 JS 程式碼範例 |
| 「直接從我的 Ragic 讀/寫資料」 | 需要 API Key + endpoint，代為呼叫 |
| 「設計工作流程/表單結構」 | 顧問角色，提供架構建議 |

## 常見錯誤 Common Mistakes

| 錯誤 | 解法 |
|------|------|
| 401 Unauthorized | API Key 未正確 Base64 編碼，或缺少 `Basic ` 前綴 |
| 資料寫入無效 | POST body 要用 field ID（數字），不是欄位名稱 |
| 子表格欄位遺失 | 子表格欄位要用 `fieldId_-1`、`fieldId_-2` 分行 |
| JS 腳本語法錯誤 | Nashorn 只支援 ES5.1，不可用箭頭函式、`let`、`const`、`Promise` |
| 日期格式錯誤 | 必須用 `yyyy/MM/dd` 或 `yyyy/MM/dd HH:mm:ss` |
