# Ragic JavaScript 工作引擎 Workflow Engine

## 技術限制（重要）

- 引擎：**Nashorn**（Java 內建），支援 **ECMAScript 5.1**
- **不可用**：箭頭函式 `=>`、`let`/`const`、`Promise`、`async/await`、`Array.from`、`setTimeout`、DOM API
- **可用**：`var`、`function`、`for`/`while`、`try/catch`、`JSON.parse`/`stringify`
- Java 傳入的陣列（如子表格）不支援 `.map()`、`.join()` 等，需先 `Array.prototype.slice.call()` 轉換

---

## 工作流程類型

| 類型 | 觸發時機 | 可用物件 |
|------|---------|---------|
| **Global Workflow** | 不直接執行，作為共用函式庫 | db, mailer, util |
| **Pre-Workflow** | 儲存前執行，可中止儲存 | db, mailer, param, response, user, log |
| **Post-Workflow** | 儲存後執行 | db, mailer, param, response, user, log |
| **Daily Workflow** | 每日排程（預設 UTC 19:00） | db, mailer, util, log |
| **Action Button** | 使用者點擊按鈕時 | db, mailer, param, response, user, log |

執行順序：`Global → Pre → [DB 寫入] → Post`

---

## 預設物件 Default Objects

### `db` — 資料庫操作

```js
// 查詢記錄
var query = db.getQuery('your_account', 'tab/sheetIndex');
query.equalTo('1000001', '台北');
query.greaterThan('1000002', 100);
query.setLimit(50);
var results = query.executeQuery();

// 讀取查詢結果
for (var i = 0; i < results.size(); i++) {
  var entry = results.get(i);
  var value = entry.getFieldValue('1000001');
}

// 新增記錄
var newEntry = db.addNode('your_account', 'tab/sheetIndex');
newEntry.setFieldValue('1000001', '新值');
newEntry.save();

// 刪除記錄
db.delNode('your_account', 'tab/sheetIndex', recordId);

// AutoGenerate 流水號
var current = db.getAutoGenerate('your_account', 'tab/sheetIndex', '1000005');
db.incrementAutoGenerate('your_account', 'tab/sheetIndex', '1000005');
```

### `param` — 存取當前記錄欄位值

> 在 Pre/Post Workflow 和 Action Button 中使用

```js
// 取得欄位新舊值
var newVal = param.getNewValue('1000001');
var oldVal = param.getOldValue('1000001');

// 取得記錄 ID
var recordId = param.getNewNodeId();

// 子表格
var subtableRows = param.getSubtableEntry('1000010');  // 子表格欄位 ID
for (var i = 0; i < subtableRows.size(); i++) {
  var row = subtableRows.get(i);
  var itemName = row.getNewValue('1000011');
}
```

### `response` — 控制執行流程與訊息

```js
// 顯示訊息（不中止儲存）
response.setMessage('操作成功');

// Pre-Workflow：中止儲存並顯示錯誤
response.setError('金額不得為負數');

// 在 Action Button 中跳轉
response.setRedirect('https://www.ragic.com/...');
```

### `mailer` — 發送通知

```js
mailer.sendMail(
  'recipient@example.com',          // 收件人
  '主旨：訂單通知',                  // 主旨
  '<b>您的訂單已確認</b>',           // HTML 內容
  'sender@example.com'              // 寄件人（可選）
);

// 傳送給多人
mailer.sendMail('a@x.com,b@x.com', '主旨', '內容');
```

### `user` — 當前使用者資訊

```js
var username = user.getUserName();   // 使用者名稱
var email = user.getEmail();
var groups = user.getGroups();       // 所屬群組（Java List）
var isAdmin = user.isAdmin();
```

### `util` — HTTP 請求與工具

```js
// 發送 HTTP GET
var result = util.httpGet('https://api.example.com/data');

// 發送 HTTP POST（JSON）
var result = util.httpPost(
  'https://api.example.com/webhook',
  JSON.stringify({ key: 'value' }),
  'application/json'
);

// 解析 JSON 回傳
var data = JSON.parse(result);
```

### `log` — 除錯記錄

```js
log.info('處理記錄 ID: ' + param.getNewNodeId());
log.warn('警告訊息');
log.error('錯誤：' + e.message);
```

> 日誌可在 Ragic 後台 **Script Log** 查看

---

## 常見腳本模式

### 1. 驗證欄位（Pre-Workflow）

```js
var amount = parseFloat(param.getNewValue('1000010'));
if (isNaN(amount) || amount < 0) {
  response.setError('金額必須為正數');
}
```

### 2. 儲存後自動同步資料到另一張表（Post-Workflow）

```js
var orderId = param.getNewValue('1000001');
var amount = param.getNewValue('1000002');

// 新增到庫存表
var entry = db.addNode('my_account', 'inventory/2');
entry.setFieldValue('2000001', orderId);
entry.setFieldValue('2000002', amount);
entry.save();
```

### 3. 跨表查詢後更新（Post-Workflow）

```js
var customerId = param.getNewValue('1000005');

var query = db.getQuery('my_account', 'customers/1');
query.equalTo('3000001', customerId);
var results = query.executeQuery();

if (results.size() > 0) {
  var customer = results.get(0);
  var level = customer.getFieldValue('3000010');
  // 更新當前記錄
  // （在 Post-Workflow 中直接用 param 的 entry 物件修改）
}
```

### 4. Global Workflow（共用函式）

```js
// 在 Global Workflow 定義
function sendOrderNotification(email, orderId) {
  mailer.sendMail(
    email,
    '訂單 #' + orderId + ' 確認通知',
    '您的訂單已成立，感謝您的訂購。'
  );
}

// 在 Post-Workflow 呼叫
sendOrderNotification(
  param.getNewValue('1000020'),
  param.getNewValue('1000001')
);
```

### 5. 呼叫外部 API（Daily Workflow）

```js
var response_data = util.httpGet(
  'https://api.exchangerate.host/latest?base=USD&symbols=TWD'
);
var data = JSON.parse(response_data);
var rate = data.rates.TWD;

// 將匯率寫入設定表
var entry = db.addNode('my_account', 'settings/5');
entry.setFieldValue('5000001', String(rate));
entry.setFieldValue('5000002', new Date().toLocaleDateString('zh-TW'));
entry.save();
```

---

## Debug 技巧

1. 用 `log.info()` 輸出變數值，到 **Script Log** 查看
2. 用 `try/catch` 包住主邏輯，`catch` 中 `log.error(e.message)`
3. Pre-Workflow 可用 `response.setError()` 顯示中間值來測試
4. 確認 Java 陣列問題：

```js
// 將 Java List 轉 JS 陣列
var jsArray = Array.prototype.slice.call(javaList);
jsArray.forEach(function(item) { ... });
```

---

## 常見錯誤排查

| 錯誤訊息 | 原因 | 解法 |
|---------|------|------|
| `SyntaxError: arrow function` | 使用了 ES6 語法 | 改用 `function(){}` |
| `TypeError: undefined is not a function` | 用了 `.map()` 在 Java List 上 | 先轉 JS 陣列 |
| `null` 回傳 | 欄位 ID 錯誤或欄位無值 | 確認 field ID，加 null 檢查 |
| 腳本無執行 | 工作流程未啟用 | 確認 Form Settings → Workflow 已開啟 |
| `response.setError` 無效 | 在 Post-Workflow 中呼叫 | setError 只在 Pre-Workflow 有效 |
