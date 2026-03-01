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
