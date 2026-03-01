# Ragic HTTP API 參考 Reference

## URL 結構

```
https://{server}.ragic.com/{account}/{tab}/{sheetIndex}[/{recordId}]?v=3&api
```

- `server`：`www`（預設）、`na3`、`ap5`、`eu2`
- `account`：Ragic 帳號名稱（網址第一段）
- `tab`：分頁資料夾名稱
- `sheetIndex`：表單編號（從 Form Settings → APIs 查看）
- `recordId`：記錄 ID（操作單筆時使用）

---

## 讀取資料 GET

### 取得清單

```js
const response = await fetch(
  'https://www.ragic.com/{account}/{tab}/{sheetIndex}?v=3&api',
  { headers }
);
const data = await response.json();
// 回傳：{ "1": { fieldId: value, ... }, "2": { ... }, ... }
```

### 取得單筆

```js
const response = await fetch(
  'https://www.ragic.com/{account}/{tab}/{sheetIndex}/{recordId}?v=3&api',
  { headers }
);
```

### 篩選 Filter

```
?where={fieldId},{operator},{value}
```

| 運算子 | 說明 |
|--------|------|
| `eq` | 等於 |
| `ne` | 不等於 |
| `gt` | 大於 |
| `lt` | 小於 |
| `contains` | 包含 |
| `notcontains` | 不包含 |
| `begins` | 開頭為 |

多條件用 `&where=` 連接：
```
?where=1000001,eq,台北&where=1000002,gt,100
```

### 分頁 Pagination

```
?limit=20&offset=0   # 每頁 20 筆，從第 0 筆開始
```

### 排序 Sorting

```
?order={fieldId}      # 升冪
?order=-{fieldId}     # 降冪
```

### 其他 GET 參數

| 參數 | 說明 |
|------|------|
| `info=1` | 回傳欄位名稱資訊 |
| `naming=1` | 欄位 key 改用名稱（不建議，名稱可能重複） |
| `subtable=1` | 展開子表格 |

---

## 新增記錄 POST（Create）

```js
const body = {
  '1000001': '台北市',       // 欄位 ID: 值
  '1000002': 'John',
  '1000003': '2024/01/15',  // 日期格式：yyyy/MM/dd
};

const response = await fetch(
  'https://www.ragic.com/{account}/{tab}/{sheetIndex}?v=3&api',
  {
    method: 'POST',
    headers: { ...headers, 'Content-Type': 'application/json' },
    body: JSON.stringify(body)
  }
);
const result = await response.json();
// result._ragicId = 新記錄 ID
```

### 子表格欄位

```js
{
  '1000010_-1': '項目A',   // 子表格第 1 行
  '1000011_-1': '100',
  '1000010_-2': '項目B',   // 子表格第 2 行
  '1000011_-2': '200'
}
```

### 多選欄位（Multiple Selection）

```js
// 傳陣列或重複 key（建議用 FormData 方式）
const formData = new FormData();
formData.append('1000005', '選項A');
formData.append('1000005', '選項B');
```

---

## 修改記錄 POST（Update）

```js
const response = await fetch(
  `https://www.ragic.com/{account}/{tab}/{sheetIndex}/{recordId}?v=3&api`,
  {
    method: 'POST',
    headers: { ...headers, 'Content-Type': 'application/json' },
    body: JSON.stringify({ '1000001': '新值' })
  }
);
```

> 只需傳要修改的欄位，不傳的欄位不會被清空。

---

## 刪除記錄 DELETE

```js
const response = await fetch(
  `https://www.ragic.com/{account}/{tab}/{sheetIndex}/{recordId}?v=3&api`,
  { method: 'DELETE', headers }
);
```

---

## 執行 Action Button

```js
// buttonId 從 Form Settings → Action Buttons 查看
const response = await fetch(
  `https://www.ragic.com/{account}/{tab}/{sheetIndex}/{recordId}?action={buttonId}&api`,
  { method: 'POST', headers }
);
```

---

## 上傳檔案 File Upload

```js
const formData = new FormData();
formData.append('1000020', fileBlob, 'filename.pdf');

const response = await fetch(
  `https://www.ragic.com/{account}/{tab}/{sheetIndex}?v=3&api`,
  { method: 'POST', headers: { Authorization: headers.Authorization }, body: formData }
);
```

---

## Lock / Unlock 記錄

```js
// Lock
await fetch(url + '?lock=1&api', { method: 'POST', headers });
// Unlock
await fetch(url + '?lock=0&api', { method: 'POST', headers });
```

---

## Webhook 設定

Ragic 可在記錄新增/修改時發送 POST 通知到指定 URL。

### 驗證 Webhook 簽章

```js
const crypto = require('crypto');

function verifyWebhook(payload, signature, secret) {
  const hmac = crypto.createHmac('sha256', secret);
  hmac.update(JSON.stringify(payload));
  const computed = hmac.digest('hex');
  return computed === signature;
}
```

---

## 回傳格式 Response Format

### 清單回傳

```json
{
  "1": {
    "_ragicId": 1,
    "1000001": "欄位值",
    "1000002": "..."
  },
  "2": { ... }
}
```

### 寫入成功回傳

```json
{
  "status": "SUCCESS",
  "_ragicId": 42
}
```

### 錯誤回傳

```json
{
  "status": "ERROR",
  "errMsg": "錯誤訊息"
}
```

---

## API 限制 Limits

- 每次 GET 預設最多回傳 1000 筆
- 大量操作建議用分頁（`limit` + `offset`）
- 速率限制依方案而定，請勿在短時間內大量請求

---

## 完整 JS 範例（Node.js）

```js
const BASE = 'https://www.ragic.com';
const ACCOUNT = 'your_account';
const TAB = 'your_tab';
const SHEET = '1';  // sheet index
const API_KEY = process.env.RAGIC_API_KEY;

const headers = {
  'Authorization': 'Basic ' + Buffer.from(API_KEY).toString('base64'),
  'Content-Type': 'application/json'
};

// 讀取清單（篩選）
async function listRecords(filter) {
  const url = `${BASE}/${ACCOUNT}/${TAB}/${SHEET}?v=3&api${filter ? '&where=' + filter : ''}`;
  const res = await fetch(url, { headers });
  return res.json();
}

// 新增記錄
async function createRecord(data) {
  const res = await fetch(`${BASE}/${ACCOUNT}/${TAB}/${SHEET}?v=3&api`, {
    method: 'POST',
    headers,
    body: JSON.stringify(data)
  });
  return res.json();
}

// 修改記錄
async function updateRecord(id, data) {
  const res = await fetch(`${BASE}/${ACCOUNT}/${TAB}/${SHEET}/${id}?v=3&api`, {
    method: 'POST',
    headers,
    body: JSON.stringify(data)
  });
  return res.json();
}

// 刪除記錄
async function deleteRecord(id) {
  const res = await fetch(`${BASE}/${ACCOUNT}/${TAB}/${SHEET}/${id}?v=3&api`, {
    method: 'DELETE',
    headers
  });
  return res.json();
}
```
