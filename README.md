# Ragic Skill for Claude Code

一個 [Claude Code](https://claude.ai/code) skill，讓 Claude 具備 Ragic 平台的開發知識，可以直接協助你呼叫 Ragic API、撰寫 JavaScript 工作流程腳本，以及設計表單與業務流程。

## 包含內容

| 檔案 | 說明 |
|------|------|
| `SKILL.md` | 概覽、快速索引、認證方式、常見錯誤 |
| `api-reference.md` | 完整 HTTP API 參考（CRUD、篩選、分頁、Webhook、Action Button） |
| `js-engine.md` | JS 工作引擎（所有預設物件、腳本模式、Debug 技巧） |

## 安裝

### 方法 1：複製檔案（個人使用）

```bash
git clone https://github.com/billxu0521/ragic_skill.git
cp -r ragic_skill/.claude/skills/ragic ~/.claude/skills/ragic
```

### 方法 2：Symlink（團隊推薦，方便同步更新）

```bash
git clone https://github.com/billxu0521/ragic_skill.git
ln -s "$(pwd)/ragic_skill/.claude/skills/ragic" ~/.claude/skills/ragic
```

安裝後重新啟動 Claude Code，skill 即生效。

## 使用方式

安裝後，只要在 Claude Code 中提出 Ragic 相關問題，Claude 會自動載入此 skill：

- 「幫我寫一段用 JavaScript 呼叫 Ragic API 讀取資料的程式」
- 「我要寫一個 Ragic Post-Workflow，儲存後自動發送 email 通知」
- 「Ragic 的篩選條件參數怎麼寫？」
- 「幫我 debug 這個 Ragic JS 腳本」
- 「設計一個訂單審核的 Ragic 工作流程」

## Skill 涵蓋範圍

### HTTP API
- 認證：API Key（HTTP Basic Auth）與帳號密碼兩種方式
- CRUD：讀取清單、讀取單筆、新增、修改、刪除
- 查詢：篩選條件、分頁、排序
- 進階：檔案上傳、Action Button 觸發、Lock/Unlock、Webhook

### JavaScript 工作引擎
- 五種工作流程類型：Global、Pre、Post、Daily、Action Button
- 所有預設物件：`db`、`param`、`response`、`mailer`、`user`、`util`、`log`
- ECMAScript 5.1（Nashorn）限制與常見坑
- 常見腳本模式：跨表同步、資料驗證、外部 API 呼叫、排程任務

## 更新

```bash
cd ragic_skill
git pull
```

若使用 symlink 方式安裝，pull 後即自動生效，無需重新複製。

## 相關連結

- [Ragic 官方文件](https://www.ragic.com/intl/zh-TW/doc/)
- [Ragic API 開發指南](https://www.ragic.com/intl/en/doc-api)
- [Ragic JS 工作引擎](https://www.ragic.com/intl/en/doc/15/javascript-workflow-engine)
