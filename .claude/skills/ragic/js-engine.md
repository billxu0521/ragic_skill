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
| **Approval Workflow** | 審核狀態改變時 | db, mailer, approvalParam, response, user, log |

執行順序：`Global → Pre → [DB 寫入] → Post`

> **注意**：Pre-Workflow 中不可使用 `entry.getFieldValue()`，請改用 `param.getNewValue()`

---

## API 參考 — 預定義系統全域變數

以下變數可直接使用，無需定義。

---

### `db` — 資料庫操作

| 方法 | 說明 |
|------|------|
| `getApname()` | 取得資料庫名稱（APName） |
| `getPath()` | 取得表單的 Path |
| `getSheetIndex()` | 取得表單的 SheetIndex |
| `getAPIQuery(pathToForm)` | 取得表單的查詢物件（query） |
| `deleteOldRecords(pathToForm, daysOld)` | 刪除建立時間超過 daysOld 天的資料，上限 500 筆 |
| `deleteOldRecords(pathToForm, daysOld, hasTime)` | hasTime=true 時以執行當下時間為基準判斷 |
| `recalculateAll(pathToForm)` | 重算整張表單所有資料的所有欄位公式 |
| `recalculateAll(pathToForm, fieldId...)` | 重算整張表單指定欄位的公式（多個 fieldId 以逗號分隔） |
| `loadLinkAndLoadAll(pathToForm)` | 同步整張表單的連結與載入欄位 |
| `setLogRecalcCostTime(boolean)` | 設為 true 可在「Workflow 表單公式重算執行時間紀錄」中顯示重算耗時 |
| `entryCopier(jsonConfig)` | 複製記錄到另一張表 |

```js
var query = db.getAPIQuery('/testAP/testForm/1');

// 刪除 30 天前建立的資料（以日期為基準）
db.deleteOldRecords('/testAP/testForm/1', 30);
// 刪除 30 天前建立的資料（以執行當下時間為基準）
db.deleteOldRecords('/testAP/testForm/1', 30, true);

// 重算指定欄位
db.recalculateAll('/testAP/testForm/1', '1000001', '1000002');

// entryCopier 完整用法 — JSON 設定格式
// COPY 的 key 為目標欄位 ID，value 為來源欄位 ID
db.entryCopier(JSON.stringify({
  THIS_PATH: '/testAP/sourceForm/1',     // 來源表單
  THIS_NODEID: nodeId,                     // 來源記錄 ID
  NEW_PATH: '/testAP/targetForm/2',        // 目標表單
  COPY: {
    2000001: 1000001,  // 目標欄位:來源欄位
    2000002: 1000002,
    2000010: 1000010   // 子表格欄位也可對應
  }
}), response);

// 複製後取得新記錄 ID，再做進一步操作
var newNodeId = response.getRootNodeId();
var targetQuery = db.getAPIQuery('/testAP/targetForm/2');
var newEntry = targetQuery.getAPIEntry(newNodeId);
newEntry.setFieldValue('2000003', 'New');
var current = new Date();
newEntry.setFieldValue('2000004', current.getFullYear() + '/' + (current.getMonth() + 1) + '/' + current.getDate());
newEntry.save();
```

---

### `query` — 查詢物件（由 `db.getAPIQuery()` 取得）

| 方法 | 說明 |
|------|------|
| `getAPIResult()` | 取得查詢第一筆記錄（entry 物件） |
| `getAPIResultsFull()` | 取得 iterator，用 `next()` 逐筆取得（大資料集用） |
| `getAPIResultList()` | 取得查詢結果陣列 |
| `getAPIEntry(rootNodeId)` | 依 Record ID 取得單筆記錄 |
| `insertAPIEntry()` | 新增一筆記錄，回傳新 entry 物件 |
| `getKeyFieldId()` | 取得此表單的主鍵欄位 ID |
| `getFieldIdByName(fieldName)` | 依欄位名稱取得欄位 ID（有多個同名欄位時回傳第一個） |
| `addFilter(fieldId, operand, value)` | 新增篩選條件 |
| `setIfIgnoreFixedFilter(boolean)` | 是否忽略表單固定篩選條件 |
| `setIfIncludeInfo(boolean)` | 設為 true 可取得簽核狀態等系統欄位（`_approve_status` 等） |
| `setOrder(fieldId, orderDir)` | 排序：1=升冪, 2=降冪, 3=次要升冪, 4=次要降冪 |
| `setLimitSize(count)` | 設定每次查詢回傳筆數（預設 1000，不建議設太大） |
| `setLimitFrom(offset)` | 設定起始偏移量，用於分頁 |
| `deleteEntry(nodeId)` | 依 Record ID 刪除記錄 |
| `deleteEntryToRecycleBin(nodeId)` | 依 Record ID 將記錄移到回收站 |
| `setGetUserNameAsSelectUserValue(boolean)` | false 時，選擇用戶欄位回傳 email 而非使用者名稱 |
| `setUpdateMode()` | Post-workflow 中使用，配合 `setIgnoreAllMasks` 取得未遮罩值 |
| `setIgnoreAllMasks(boolean)` | true 時取消所有遮罩欄位，取得原始值 |
| `addIgnoreMaskDomain(fieldId)` | 取消指定欄位的遮罩，僅解除特定欄位 |
| `setFullTextSearch(queryTerm)` | 全文檢索篩選，替代 `addFilter` 使用 |
| `setFieldValueModeAsNodeIdOnly(boolean)` | true 時，欄位值回傳關聯資料的 nodeId 而非顯示值 |
| `setFieldValueModeAsNodeIdAndValue(boolean)` | true 時，同時回傳 nodeId 和顯示值（用於 loadAllDefaultValues 行為異常時） |

**篩選運算子（addFilter）**

| 運算子 | 說明 |
|--------|------|
| `=` | 等於 |
| `<` | 小於 |
| `>` | 大於 |
| `<=` | 小於等於 |
| `>=` | 大於等於 |
| `like` | 包含（模糊比對） |
| `regex` | 正規表達式 |

```js
var query = db.getAPIQuery('/testAP/testForm/1');
query.addFilter('1000001', '=', '台北');
query.addFilter('1000002', '>', 100);
query.addFilter('1000003', 'like', '關鍵字');
query.setOrder('1000001', 1);   // 升冪
query.setLimitSize(50);
query.setLimitFrom(0);

// 取得 iterator（大資料集）
var iter = query.getAPIResultsFull();
while (iter.hasNext()) {
  var entry = iter.next();
  var val = entry.getFieldValue('1000001');
}

// 取得陣列
var list = query.getAPIResultList();

// 包含簽核狀態等系統欄位
query.setIfIncludeInfo(true);
var entry = query.getAPIResult();
var status = entry.getFieldValue('_approve_status');

// 新增記錄
var newEntry = query.insertAPIEntry();
newEntry.setFieldValue('1000001', '值');
newEntry.save();
```

**篩選進階模式**

```js
// OR 條件：同一欄位多次 addFilter，視為 OR
var query = db.getAPIQuery('/testAP/testForm/1');
query.addFilter('1000002', '=', 'Green');
query.addFilter('1000002', '=', 'Red');
// 結果：1000002 為 Green 或 Red 的記錄

// Range 查詢：同一欄位用 > 和 < 組合
var query2 = db.getAPIQuery('/testAP/testForm/1');
query2.addFilter('1000006', '>', '20');
query2.addFilter('1000006', '<', '40');
// 結果：1000006 大於 20 且小於 40 的記錄

// 全文檢索（替代 addFilter）
var query3 = db.getAPIQuery('/testAP/testForm/1');
query3.setFullTextSearch('關鍵字');
var results = query3.getAPIResultsFull();
```

> **篩選規則**：不同欄位的 `addFilter` 為 AND；同一欄位的 `addFilter` 若運算子相同（如都是 `=`）為 OR，若運算子不同（如 `>` 和 `<`）為 Range。日期篩選格式需為 `yyyy/MM/dd` 或 `yyyy/MM/dd HH:mm:ss`。

---

### `entry` — 記錄物件

| 方法 | 說明 |
|------|------|
| `getFieldValue(fieldId)` | 取得欄位值（不含子表格欄位） |
| `getFieldValueByName(fieldName)` | 依欄位名稱取得欄位值（有多個同名時回傳第一個） |
| `getFieldValues(fieldId)` | 多選欄位：取得所有值的陣列 |
| `getFieldIdByName(fieldName)` | 依名稱取得欄位 ID |
| `getKeyFieldId()` | 取得此表單的主鍵欄位 ID |
| `getRootNodeId()` | 取得此記錄的 ID（rootNodeId） |
| `getJSON()` | 取得整筆記錄的 JSON |
| `setFieldValue(fieldId, value)` | 設定欄位值 |
| `setFieldValue(fieldId, value, appendValue)` | appendValue=true 用於多選欄位（附加而非覆蓋） |
| `setFieldFile(fieldId, fileName, fileContent)` | 上傳檔案到文件/圖片欄位 |
| `setFieldFile(fieldId, fileName, fileContent, appendValue)` | appendValue=true 時不覆蓋原有檔案 |
| `getSubtableSize(subtableId)` | 取得子表格列數 |
| `getSubtableRootNodeId(subtableId, rowNumber)` | 依子表格 ID 和列號取得該列的 rootNodeId |
| `getSubtableFieldValue(subtableId, rowIndex, fieldId)` | 取得子表格某列的欄位值 |
| `setSubtableFieldValue(fieldId, subtableRootNodeId, value)` | 設定子表格欄位值 |
| `setSubtableFieldValue(fieldId, subtableRootNodeId, value, appendValue)` | appendValue=true 用於多選子表格欄位 |
| `setSubtableFieldFile(fieldId, subtableRootNodeId, fileName, fileContent)` | 上傳檔案到子表格的文件/圖片欄位 |
| `setSubtableFieldFile(fieldId, subtableRootNodeId, fileName, fileContent, appendValue)` | appendValue=true 時不覆蓋原有檔案 |
| `deleteSubtableRow(subtableId, subtableRootNodeId)` | 依 rootNodeId 刪除子表格列 |
| `deleteSubtableRowByRowNumber(subtableId, rowNumber)` | 依列號刪除子表格列 |
| `deleteSubtableRowAll(subtableId)` | 刪除子表格所有列 |
| `loadAllLinkAndLoad()` | 同步所有連結與載入欄位 |
| `loadLinkAndLoad(linkFieldId)` | 同步指定連結欄位的整組載入欄位 |
| `loadLinkAndLoadField(loadFieldId)` | 同步連結與載入中的單一載入欄位 |
| `recalculateAllFormulas()` | 重算所有含公式的欄位 |
| `recalculateFormula(fieldId)` | 重算指定欄位的公式 |
| `recalculateFormula(fieldId, cellName)` | 重算指定位置（如 A1、C2）的公式，用於多個欄位共享同一 fieldId 時 |
| `loadAllDefaultValues(user)` | 載入所有設有預設值的欄位 |
| `loadDefaultValue(fieldId, user)` | 載入指定欄位的預設值 |
| `lock()` | 鎖定記錄 |
| `unlock()` | 解鎖記錄 |
| `isLocked()` | 判斷記錄是否已鎖定（boolean） |
| `save()` | 儲存記錄 |
| `setCreateHistory(boolean)` | 設定是否建立歷史記錄 |
| `isCreateHistory()` | 取得是否設定為建立歷史記錄 |
| `setIfExecuteWorkflow(boolean)` | 設定是否執行 Pre/Post-workflow |
| `setIgnoreEmptyCheck(boolean)` | 設定是否忽略「不為空」欄位檢查 |
| `setRecalParentFormula(boolean)` | 設定是否重新計算父表單公式（此表單為子表格表單時） |
| `setIfDoLnls(boolean)` | 觸發其他表單的連結與載入同步，**僅支援 Action Button** |
| `isFieldValueModeIsNodeIdOnly()` | 檢查目前欄位取值模式是否為僅回傳 nodeId |

**簽核與系統欄位**（需先對 `query` 呼叫 `setIfIncludeInfo(true)`）：

| 系統欄位 key | 說明 |
|-------------|------|
| `_approve_status` | 簽核狀態：`F`=已簽核，`REJ`=拒絕，`P`=處理中，空字串=未啟動 |
| `_approve_next` | 下一個應簽核此記錄的人 |
| `_create_date` | 記錄建立日期 |
| `_create_user` | 記錄建立者 email |

```js
// 基本讀寫
var val = entry.getFieldValue('1000001');
entry.setFieldValue('1000001', '新值');
entry.save();

// 子表格操作
var rowCount = entry.getSubtableSize('1000010');
for (var i = 0; i < rowCount; i++) {
  var rowId = entry.getSubtableRootNodeId('1000010', i);
  entry.setSubtableFieldValue('1000011', rowId, '新值');
}

// 刪除子表格所有列後重新插入
entry.deleteSubtableRowAll('1000010');

// 用負數 ID 建立子表格新列（同一負數 ID 的值歸到同一列）
entry.setSubtableFieldValue('1000011', -100, '第一列值A');
entry.setSubtableFieldValue('1000012', -100, '第一列值B');  // 與上面同一列
entry.setSubtableFieldValue('1000011', -101, '第二列值A');  // 新的一列
entry.setSubtableFieldValue('1000012', -101, '第二列值B');
entry.save();
```

---

### `param` — 欄位新舊值（Pre/Post-Workflow）

| 方法 | 說明 |
|------|------|
| `getNewValue(fieldId)` | 儲存後的欄位值 |
| `getOldValue(fieldId)` | 儲存前的欄位值 |
| `getNewValues(fieldId)` | 儲存後的多選欄位值（陣列） |
| `getOldValues(fieldId)` | 儲存前的多選欄位值（陣列） |
| `getNewNodeId(fieldId)` | 儲存後的欄位節點 ID（整數） |
| `getOldNodeId(fieldId)` | 儲存前的欄位節點 ID（整數） |
| `getUpdatedEntry()` | **僅 Post-workflow**：回傳剛儲存的 entry 物件 |
| `getSubtableEntry(fieldId)` | 回傳子表格 iterator，可逐列操作 |
| `isCreateNew()` | 是否為新增記錄（true=新增，false=修改） |

> **注意**：`param` 不可在 Global Workflow 中使用

```js
var newVal = param.getNewValue('1000001');
var oldVal = param.getOldValue('1000001');
var isNew = param.isCreateNew();

// Post-Workflow：取得已儲存的 entry
var entry = param.getUpdatedEntry();
entry.setFieldValue('1000050', '後處理值');
entry.save();

// 子表格遍歷（iterator 方式）
var subtableIter = param.getSubtableEntry('1000010');
while (subtableIter.hasNext()) {
  var row = subtableIter.next();
  var itemName = row.getNewValue('1000011');
}

// 子表格遍歷（toArray 方式，可用索引存取）
var arr = param.getSubtableEntry('1000010').toArray();
for (var i = 0; i < arr.length; i++) {
  var date = arr[i].getNewValue('1000003');
  var amount = arr[i].getNewValue('1000004');
}
```

---

### `response` — 回應控制

| 方法 | 說明 |
|------|------|
| `getStatus()` | 取得目前回應狀態 |
| `setStatus(status)` | 設定回應狀態：`SUCCESS`、`WARN`、`CONFIRM`、`INVALID`、`ERROR` |
| `setMessage(message)` | 設定顯示訊息（可多次呼叫，所有訊息會同時顯示） |
| `numOfMessages()` | 回傳已設定的訊息數量 |
| `setOpenURL(url)` | 儲存後重新導向至指定 URL |
| `setOpenURLInNewTab(boolean)` | 是否在新分頁開啟（預設 true） |

**狀態說明**：

| Status | 說明 |
|--------|------|
| `SUCCESS` | 成功，繼續流程 |
| `WARN` | 警告，仍繼續儲存 |
| `ERROR` | 錯誤 |
| `INVALID` | 驗證失敗，**Pre-Workflow 專用**，阻止儲存 |
| `CONFIRM` | 顯示確認框，**Action Button 不支援** |

```js
// Pre-Workflow 驗證
var amount = parseFloat(param.getNewValue('1000010'));
if (isNaN(amount) || amount < 0) {
  response.setStatus('INVALID');
  response.setMessage('金額必須為正數');
}

// 儲存後跳轉
response.setStatus('SUCCESS');
response.setOpenURL('https://www.ragic.com/testAP/testForm/1/3');
response.setOpenURLInNewTab(false);
```

---

### `mailer` — 電子郵件與推播通知

| 方法 | 說明 |
|------|------|
| `compose(to, cc, from, fromPersonal, subject, content)` | 撰寫郵件，to/cc 可用逗號分隔多個地址 |
| `send()` | 同步發送 |
| `sendAsync()` | 非同步發送（每次 Workflow 執行只能呼叫一次） |
| `attach(url)` | 附加檔案（需完整 https:// URL） |
| `setBcc(email)` | 設定密件副本（多人用逗號分隔） |
| `setSendRaw(boolean)` | true 時不將內容轉換為 HTML |
| `sendAppNotification(email, message)` | 發送 App 推播通知 |
| `sendAppNotification(email, message, pathToForm, nodeId)` | 發送推播並指定點擊後跳轉的記錄（pathToForm 格式：`/forms/1`，不含帳號名） |

```js
mailer.compose(
  'to@example.com,to2@example.com',
  'cc@example.com',
  'sender@example.com',
  '寄件人名稱',
  '郵件主旨',
  '<b>HTML 內容</b>'
);
mailer.attach('https://www.ragic.com/path/to/file.pdf');
mailer.setBcc('bcc@example.com');
mailer.send();

// App 推播（點擊後跳轉到指定記錄）
mailer.sendAppNotification('user@example.com', '通知訊息', '/testForm/1', 3);
```

---

### `util` — HTTP 請求與檔案操作

| 方法 | 說明 |
|------|------|
| `getURL(url)` | HTTP GET |
| `postURL(url, body)` | HTTP POST |
| `putURL(url, body)` | HTTP PUT |
| `deleteURL(url)` | HTTP DELETE |
| `setHeader(name, value)` | 設定後續 HTTP 請求的 Header |
| `removeHeader(name)` | 移除已設定的 Header |
| `ignoreSSL()` | 忽略 SSL 憑證驗證 |
| `downloadFile(fileUrl)` | 將 URL 的檔案下載到資料庫 upload 資料夾 |
| `downloadFile(fileUrl, postBody)` | 以 POST 方式下載檔案 |
| `postFile(sourceUrl, destinationUrl)` | 將來源檔案上傳到指定 URL |
| `logWorkflowError(text)` | 在工作流程日誌中記錄錯誤訊息 |

```js
util.setHeader('Authorization', 'Bearer TOKEN');
util.setHeader('Content-Type', 'application/json');

var result = util.postURL('https://api.example.com/data', JSON.stringify({ key: 'val' }));
var data = JSON.parse(result);

// 下載檔案並寫入欄位
var fileContent = util.downloadFile('https://example.com/file.pdf');
entry.setFieldFile('1000020', 'report.pdf', fileContent);
entry.save();
```

---

### `user` — 當前使用者

| 方法 | 說明 |
|------|------|
| `getEmail()` | 使用者 email |
| `getUserName()` | 使用者全名 |
| `isInGroup(groupName)` | 是否屬於指定群組（boolean） |

---

### `account` — 帳號資訊

| 方法 | 說明 |
|------|------|
| `getUserName(email)` | 依 email 取得使用者全名 |
| `getUserEmail(userName)` | 依使用者名稱取得 email |
| `getTimeZoneOffset()` | 時區時差（毫秒） |
| `getTimeZoneOffsetInHours()` | 時區時差（小時） |
| `reset()` | 清除帳號相關快取並重新載入頁面 |

---

### `approval` — 審核流程（Post-Workflow）

| 方法 | 說明 |
|------|------|
| `create(wfSigner)` | 建立審核流程，wfSigner 為 JSON.stringify 後的陣列 |
| `cancel()` | 取消審核流程 |

```js
// 建立多步驟簽核
var signers = [];
signers.push({ 'stepIndex': '0', 'approver': 'boss@example.com', 'stepName': '主管' });
signers.push({ 'stepIndex': '1', 'approver': 'hr@example.com',   'stepName': 'HR' });
approval.create(JSON.stringify(signers));

// 取消簽核
approval.cancel();
```

**動態簽核人（依組織樹）**：

| 格式 | 說明 |
|------|------|
| `$DS` | 當前使用者的直屬主管 |
| `$SS` | 當前使用者主管的主管 |
| `$DSL` | 前一個簽核人的主管 |

```js
signers.push({ 'stepIndex': '1', 'approver': '$DSL', 'stepName': '' });
```

> **注意**：`approver` 必須符合設計模式中設定的群組規則，否則簽核不會建立。`stepName` 使用特殊格式時可為空字串。

---

### `approvalParam` — 審核 Workflow 參數（僅 Approval Workflow）

| 方法 | 說明 |
|------|------|
| `getEntryRootNodeId()` | 取得相關記錄的 rootNodeId |
| `getApprovalAction()` | 取得審核動作：`CREATE`、`APPROVE`、`FINISH`、`CANCEL`、`REJECT` |

```js
var action = approvalParam.getApprovalAction();
if (action === 'APPROVE' || action === 'FINISH') {
  var recordId = approvalParam.getEntryRootNodeId();
  var query = db.getAPIQuery('/testAP/testForm/1');
  var entry = query.getAPIEntry(recordId);
  entry.setFieldValue('1000050', 'APPROVED');
  entry.save();
}
```

---

### `log` — 除錯記錄

| 方法 | 說明 |
|------|------|
| `setToConsole(boolean)` | 輸出到 console |
| `println(message)` | 印出訊息（可在 Script Log 查看） |

---

## 常見腳本模式

### 1. 驗證欄位（Pre-Workflow）
```js
var amount = parseFloat(param.getNewValue('1000010'));
if (isNaN(amount) || amount < 0) {
  response.setStatus('INVALID');
  response.setMessage('金額必須為正數');
}
```

### 2. 儲存後同步到另一張表（Post-Workflow）
```js
var query = db.getAPIQuery('/testAP/inventory/2');
var newEntry = query.insertAPIEntry();
newEntry.setFieldValue('2000001', param.getNewValue('1000001'));
newEntry.setFieldValue('2000002', param.getNewValue('1000002'));
newEntry.save();
```

### 3. 跨表查詢後更新（Post-Workflow）
```js
var customerId = param.getNewValue('1000005');
var query = db.getAPIQuery('/testAP/customers/1');
query.addFilter('3000001', '=', customerId);
var customer = query.getAPIResult();
if (customer) {
  var entry = param.getUpdatedEntry();
  entry.setFieldValue('1000030', customer.getFieldValue('3000010'));
  entry.save();
}
```

### 4. Global Workflow 共用函式
```js
// Global Workflow 定義
function sendNotification(email, orderId) {
  mailer.compose(email, null, 'system@co.com', 'System', '訂單 #' + orderId, '訂單已成立');
  mailer.send();
}

// Post-Workflow 呼叫
sendNotification(param.getNewValue('1000020'), param.getNewValue('1000001'));
```

### 5. 呼叫外部 API（Daily Workflow）
```js
util.setHeader('Authorization', 'Bearer TOKEN');
var result = util.getURL('https://api.example.com/rate');
var data = JSON.parse(result);
var query = db.getAPIQuery('/testAP/settings/5');
var entry = query.insertAPIEntry();
entry.setFieldValue('5000001', String(data.rate));
entry.save();
```

### 6. 多選欄位複製（覆寫目標欄位）
```js
// 從來源多選欄位取得所有值，用 | 分隔後寫入目標欄位
var values = entry.getFieldValues('1000010');
var combined = '';
for (var i = 0; i < values.length; i++) {
  if (i === 0) {
    combined = values[i];
  } else {
    combined = combined + '|' + values[i];
  }
}
entry.setFieldValue('1000020', combined, false);  // false = 覆寫而非附加
entry.save();
```

### 7. Post-Workflow 取得未遮罩值
```js
// 表單有遮罩欄位時，param.getUpdatedEntry() 會取得遮罩後的值
// 需用 setUpdateMode + setIgnoreAllMasks 組合取得原始值
var recordId = param.getNewNodeId(param.getUpdatedEntry().getKeyFieldId());
var query = db.getAPIQuery('/testAP/testForm/1');
query.setUpdateMode();
query.setIgnoreAllMasks(true);  // 取消所有遮罩
var entry = query.getAPIEntry(recordId);
var realPhone = entry.getFieldValue('1000030');  // 取得未遮罩的電話號碼

// 若只需解除特定欄位的遮罩
var query2 = db.getAPIQuery('/testAP/testForm/1');
query2.setUpdateMode();
query2.addIgnoreMaskDomain('1000030');  // 僅解除此欄位
var entry2 = query2.getAPIEntry(recordId);
```

### 8. 觸發其他表單的 Workflow
```js
// 修改其他表單的記錄時，預設不會觸發該表單的 workflow
// 需用 setIfExecuteWorkflow(true) 啟用
var query = db.getAPIQuery('/testAP/otherForm/2');
var entry = query.getAPIEntry(targetRecordId);
entry.setFieldValue('2000001', '新值');
entry.setIfExecuteWorkflow(true);  // 觸發 otherForm 的 pre/post-workflow
entry.save();
```

### 9. 審核通過後更新狀態（Approval Workflow）
```js
var action = approvalParam.getApprovalAction();
if (action === 'FINISH') {
  var recordId = approvalParam.getEntryRootNodeId();
  var query = db.getAPIQuery('/testAP/orders/1');
  var entry = query.getAPIEntry(recordId);
  entry.setFieldValue('1000050', 'APPROVED');
  entry.save();
}
```

---

## Debug 技巧

1. 用 `log.println()` 輸出變數值，在 **Script Log** 查看
2. 用 `util.logWorkflowError()` 記錄錯誤到資料庫維護頁的 Workflow 日誌
3. 用 `try/catch` 包住主邏輯，catch 中記錄 `e.message`
4. Java 陣列轉換：`var arr = Array.prototype.slice.call(javaList);`

---

## 常見錯誤排查

| 錯誤現象 | 原因 | 解法 |
|---------|------|------|
| `SyntaxError` | 使用 ES6 語法 | 改用 `function(){}` 取代箭頭函式 |
| `.map()` 失敗 | Java List 不支援 JS 陣列方法 | 先 `Array.prototype.slice.call()` 轉換 |
| 欄位值為 `null` | fieldId 錯誤或無值 | 確認七碼 fieldId，加 null 檢查 |
| 腳本未執行 | 工作流程未啟用 | 確認 Form Settings → Workflow 已開啟 |
| `INVALID` 無效 | 在 Post-Workflow 呼叫 | INVALID 只在 Pre-Workflow 有效 |
| `CONFIRM` 無效 | 在 Action Button 使用 | Action Button 不支援 CONFIRM |
| `param` 未定義 | 在 Global/Daily Workflow 使用 | param 只在 Pre/Post/Action Button 可用 |
| 簽核未建立 | approver 不符合設計模式規則 | 確認 approver email 在指定群組內 |
| `setIfDoLnls` 無效 | 在 Post-Workflow 使用 | 此方法僅支援 Action Button |
