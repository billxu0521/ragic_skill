# Ragic HTTP API 參考 Reference

## URL 結構

```
https://{server}.ragic.com/{account}/{tab}/{sheetIndex}[/{recordId}]?v=3&api
```

- `server`：`www`（預設）、`na3`、`ap5`、`eu2`（依帳號所在選擇）
- `account`：Ragic 帳號名稱
- `tab`：頁籤名稱
- `sheetIndex`：表單索引（從 Form Settings → APIs 查看）
- `recordId`：記錄 ID（操作單筆時使用）

---

## 認證 Authentication

### 方法 1：API Key（推薦）

```js
const headers = {
  'Authorization': 'Basic ' + Buffer.from(API_KEY).toString('base64'),
  'Content-Type': 'application/json'
};
```

或以 URL 參數傳遞（備用）：
```
?api&APIKey=YOUR_API_KEY
```

### 方法 2：帳號密碼（取得 Session ID）

```js
// 1. 登入取得 sid
const res = await fetch('https://www.ragic.com/{account}?login', {
  method: 'POST',
  body: new URLSearchParams({ u: 'email@example.com', p: 'password' })
});
const { sessionId } = await res.json();

// 2. 後續請求帶入 sid
const url = `https://www.ragic.com/{account}/{tab}/{sheetIndex}?api&sid=${sessionId}`;
```

> **安全提醒**：API Key 擁有完整讀寫權限，請勿硬寫在前端程式碼，建議為 API 建立專屬使用者帳號。

---

## API 版本

```
?version=2025-01-01   # 指定版本（YYYY-MM-DD 格式）
?v=3                  # 等同現行版本
```

---

## 讀取資料 GET

### 取得清單

```js
const res = await fetch(
  'https://www.ragic.com/{account}/{tab}/{sheetIndex}?v=3&api',
  { headers }
);
const data = await res.json();
// 回傳：{ "1": { _ragicId: 1, fieldId: value, ... }, "2": { ... }, ... }
```

### 取得單筆

```js
const res = await fetch(
  'https://www.ragic.com/{account}/{tab}/{sheetIndex}/{recordId}?v=3&api',
  { headers }
);
```

### GET 篩選參數（where）

```
?where={fieldId},{operator},{value}
```

多條件（AND）：
```
?where=1000001,eq,台北&where=1000002,gt,100
```

**支援的運算子**：

| 運算子 | 說明 |
|--------|------|
| `eq` | 等於 |
| `eqeq` | 等於（依 nodeId） |
| `gt` | 大於 |
| `gte` | 大於等於 |
| `lt` | 小於 |
| `lte` | 小於等於 |
| `like` | 包含（模糊比對） |
| `regex` | 正規表達式 |

其他篩選參數：
```
?fts=關鍵字           # 全文檢索
?filterId={id}        # 使用表單上的共通篩選條件
?ignoreFixedFilter=true  # 忽略表單固定篩選條件
```

### 分頁 Pagination

```
?limit=20&offset=0    # 每次 20 筆，從第 0 筆開始
```

預設每次最多回傳 1000 筆。

### 排序 Sorting

```
?order={fieldId},ASC   # 升冪
?order={fieldId},DESC  # 降冪
?reverse=true          # 反轉預設排序
```

### 其他 GET 參數

| 參數 | 說明 |
|------|------|
| `subtables=0` | 不包含子表格資料 |
| `listing=true` | 只回傳列表頁欄位 |
| `info=true` | 包含建立者、建立時間等資訊 |
| `comment=true` | 包含留言紀錄 |
| `approval=true` | 包含簽核資訊 |
| `conversation=true` | 包含信件資訊 |
| `history=true` | 包含修改歷史紀錄 |
| `bbcode=true` | 保留欄位原始 BBCode 格式 |
| `ignoreMask=true` | 不遮罩欄位值 |
| `naming=EID` | 欄位 key 改用欄位 ID |
| `naming=FNAME` | 欄位 key 改用欄位名稱 |
| `callback={funcName}` | JSONP 支援 |

### 下載記錄報表

```
/{recordId}.xhtml           # HTML 友善列印
/{recordId}.pdf             # PDF
/{recordId}.xlsx            # Excel
/{recordId}.custom?cid={id} # 合併列印
/{recordId}.carbone?fileFormat={格式}&ragicCustomPrintTemplateId={id}  # 客製報表
```

---

## 新增記錄 POST（Create）

```js
const body = {
  '1000001': '台北市',
  '1000002': 'John',
  '1000003': '2024/01/15',   // 日期：yyyy/MM/dd 或 yyyy/MM/dd HH:mm
};

const res = await fetch(
  'https://www.ragic.com/{account}/{tab}/{sheetIndex}?v=3&api',
  {
    method: 'POST',
    headers: { ...headers, 'Content-Type': 'application/json' },
    body: JSON.stringify(body)
  }
);
const result = await res.json();
// result._ragicId = 新記錄 ID
```

### 子表格欄位

```js
{
  '1000010_-1': '項目A',  // 子表格第 1 行
  '1000011_-1': '100',
  '1000010_-2': '項目B',  // 子表格第 2 行
  '1000011_-2': '200'
}
```

### 多選欄位（Multiple Selection）

```js
// 使用 FormData 傳遞多個相同 key
const formData = new FormData();
formData.append('1000005', '選項A');
formData.append('1000005', '選項B');

await fetch(url, { method: 'POST', headers: { Authorization: headers.Authorization }, body: formData });
```

### 建立/更新附加參數

| 參數 | 說明 |
|------|------|
| `doFormula=true` | 儲存後重新計算公式 |
| `doDefaultValue=true` | 儲存後載入預設值 |
| `doLinkLoad=true` | 執行連結與載入 |
| `doLinkLoad=first` | 只執行第一筆連結與載入 |
| `doWorkflow=true` | 執行 Pre/Post-workflow |
| `notification=true` | 發送通知（預設依設定） |
| `notification=false` | 強制不發送通知 |
| `checkLock=true` | 若記錄已鎖定則回傳錯誤 |

```
POST /account/tab/sheet?v=3&api&doFormula=true&doWorkflow=true
```

---

## 修改記錄 POST（Update）

```js
const res = await fetch(
  `https://www.ragic.com/{account}/{tab}/{sheetIndex}/{recordId}?v=3&api`,
  {
    method: 'POST',
    headers: { ...headers, 'Content-Type': 'application/json' },
    body: JSON.stringify({ '1000001': '新值' })
  }
);
```

> 只需傳要修改的欄位，其他欄位不受影響。

### 刪除子表格列

```js
// 在 POST body 中加入 DELSUB_ 參數
const body = { 'DELSUB_1000010': subtableRowId };
```

---

## 刪除記錄 DELETE

```js
const res = await fetch(
  `https://www.ragic.com/{account}/{tab}/{sheetIndex}/{recordId}?v=3&api`,
  { method: 'DELETE', headers }
);
```

---

## 上傳檔案 File Upload

```js
const formData = new FormData();
formData.append('1000020', fileBlob, 'filename.pdf');

await fetch(
  `https://www.ragic.com/{account}/{tab}/{sheetIndex}?v=3&api`,
  { method: 'POST', headers: { Authorization: headers.Authorization }, body: formData }
);
```

---

## 留言 Comment

```js
// POST /{recordId}?api&comment=true
const formData = new FormData();
formData.append('c', '留言內容');           // 必填
formData.append('at', fileBlob, 'file.jpg'); // 附件（選填）

await fetch(
  `https://www.ragic.com/{account}/{tab}/{sheetIndex}/{recordId}?api&comment=true`,
  { method: 'POST', headers: { Authorization: headers.Authorization }, body: formData }
);
```

---

## 從網址匯入 Import From URL

```js
// 需要 SYSAdmin 權限
await fetch(
  `https://www.ragic.com/{account}/{tab}/{sheetIndex}?v=3&api`,
  {
    method: 'POST',
    headers,
    body: JSON.stringify({ importData: 'https://example.com/data.csv' })
  }
);
```

---

## Lock / Unlock 記錄

```js
// 鎖定
await fetch(`https://www.ragic.com/{account}/{tab}/{sheetIndex}/{recordId}?api&lock`, {
  method: 'POST', headers
});

// 解鎖
await fetch(`https://www.ragic.com/{account}/{tab}/{sheetIndex}/{recordId}?api&unlock`, {
  method: 'POST', headers
});
```

---

## 執行 Action Button

```js
// 1. 取得按鈕 ID
const meta = await fetch(
  'https://www.ragic.com/{account}/{tab}/{sheetIndex}/metadata/actionButton?api&category=massOperation',
  { headers }
);
const buttons = await meta.json();

// 2. 執行指定按鈕
await fetch(
  `https://www.ragic.com/{account}/{tab}/{sheetIndex}/{recordId}?api&bId={buttonId}`,
  { method: 'POST', headers }
);
```

---

## 大量操作 Mass Operation

非同步執行，回傳 `taskId`，可用 taskId 查詢進度。

```
POST /massOperation/{操作類型}?api
```

| 操作類型 | 說明 | 主要參數 |
|---------|------|---------|
| `massLock` | 大量鎖定/解鎖 | `action="lock"` 或 `"unlock"` |
| `massApproval` | 批次簽核 | `action`（簽核動作），可加 `comment` |
| `massActionButton` | 大量執行動作按鈕 | `buttonId` |
| `massUpdate` | 大量修改欄位 | `action`（欄位值陣列） |

---

## Webhook

Ragic 可在記錄新增/修改/刪除時發送 POST 通知到指定 URL（在表單工具列「同步」中設定）。

### Webhook 回應格式

僅 ID 清單：
```json
[1, 2, 4]
```

含完整資料：
```json
{
  "data": { ... },
  "apname": "testAP",
  "path": "/testForm",
  "sheetIndex": "1",
  "eventType": "update"
}
```

### 驗證 Webhook 簽章

```js
const crypto = require('crypto');
const https = require('https');

// 1. 取得 Ragic 公鑰
function getPublicKey(callback) {
  https.get('https://www.ragic.com/api/http/getWebhookSignaturePublicKey.jsp', (res) => {
    let data = '';
    res.on('data', chunk => data += chunk);
    res.on('end', () => callback(data));
  });
}

// 2. 驗證簽章（SHA256withRSA）
function verifyWebhook(payload, signature, publicKey) {
  // 將 data 屬性的 key 排序後序列化
  const dataStr = JSON.stringify(
    Object.keys(payload.data).sort().reduce((obj, key) => {
      obj[key] = payload.data[key];
      return obj;
    }, {})
  );
  const verify = crypto.createVerify('SHA256');
  verify.update(dataStr);
  return verify.verify(publicKey, signature, 'base64');
}
```

---

## 回應格式 Response Format

### 清單回傳

```json
{
  "1": {
    "_ragicId": 1,
    "_star": false,
    "_index_title_": "主鍵值",
    "1000001": "欄位值",
    "_subtable_1000010": {
      "101": { "1000011": "子表格欄位值" },
      "102": { "1000011": "子表格欄位值" }
    }
  }
}
```

### 寫入成功回傳

```json
{ "status": "SUCCESS", "_ragicId": 42 }
```

### 錯誤回傳

```json
{ "status": "ERROR", "errMsg": "錯誤訊息", "errCode": 101 }
```

---

## 錯誤代碼 Error Codes

| HTTP 狀態碼 | 說明 |
|------------|------|
| `200` | 成功 |
| `400` | 請求格式錯誤 |
| `401` | 未授權（API Key 錯誤） |
| `402` | 請求失敗（如記錄已鎖定） |
| `404` | 資料或路徑未找到 |
| `500-504` | 伺服器錯誤 |

| errCode 範圍 | 說明 |
|-------------|------|
| `101–109` | 帳號、路徑、權限問題 |
| `201–204` | 參數錯誤、執行失敗、頻率限制 |
| `301–304` | 逾時、過期、無效 API Key |
| `402` | 記錄已鎖定 |
| `404` | 記錄不存在 |

---

## API 限制 Limits

- 最大排隊請求數：50
- 超過 **5 次/秒**觸發人工審核
- GET 請求應等待上一個回應後再發送下一個
- 每次 GET 預設最多回傳 **1000 筆**

---

## 完整 JS 範例（Node.js）

```js
const BASE = 'https://www.ragic.com';
const ACCOUNT = 'testAP';
const SHEET = '/testForm/1';
const API_KEY = process.env.RAGIC_API_KEY;

const headers = {
  'Authorization': 'Basic ' + Buffer.from(API_KEY).toString('base64'),
  'Content-Type': 'application/json'
};

// 讀取清單（篩選 + 分頁）
async function listRecords({ fieldId, op, value, limit = 20, offset = 0 } = {}) {
  let url = `${BASE}/${ACCOUNT}${SHEET}?v=3&api&limit=${limit}&offset=${offset}`;
  if (fieldId) url += `&where=${fieldId},${op},${encodeURIComponent(value)}`;
  const res = await fetch(url, { headers });
  return res.json();
}

// 取得單筆
async function getRecord(id) {
  const res = await fetch(`${BASE}/${ACCOUNT}${SHEET}/${id}?v=3&api`, { headers });
  return res.json();
}

// 新增記錄
async function createRecord(data) {
  const res = await fetch(`${BASE}/${ACCOUNT}${SHEET}?v=3&api`, {
    method: 'POST', headers, body: JSON.stringify(data)
  });
  return res.json();
}

// 修改記錄
async function updateRecord(id, data) {
  const res = await fetch(`${BASE}/${ACCOUNT}${SHEET}/${id}?v=3&api`, {
    method: 'POST', headers, body: JSON.stringify(data)
  });
  return res.json();
}

// 刪除記錄
async function deleteRecord(id) {
  const res = await fetch(`${BASE}/${ACCOUNT}${SHEET}/${id}?v=3&api`, {
    method: 'DELETE', headers
  });
  return res.json();
}
```
