# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 專案目的

這個 repo 是一個 **Claude Code skill**，專門提供 Ragic 平台的開發知識：HTTP API、工作流程設計，以及伺服器端 JavaScript 工作引擎。

## Skill 結構

```
.claude/skills/ragic/
  SKILL.md          # 主檔案：概覽、快速索引、認證方式
  api-reference.md  # 完整 HTTP API 參考（CRUD、Webhook、篩選等）
  js-engine.md      # JS 工作引擎參考（物件、腳本模式、Debug）
```

## 部署方式

**個人使用**：複製整個 `.claude/skills/ragic/` 到 `~/.claude/skills/ragic/`

**團隊使用**：將此 repo clone 到團隊成員的機器，並在各自的 `~/.claude/skills/` 建立 symlink：
```bash
ln -s /path/to/ragic_skill/.claude/skills/ragic ~/.claude/skills/ragic
```

## 更新 Skill 內容

- **API 規格更新**：修改 `api-reference.md`
- **JS 引擎新功能**：修改 `js-engine.md`
- **新增觸發場景**：更新 `SKILL.md` 的 `description` 欄位

## 設計原則

- 說明文字用**繁體中文**，程式碼範例用**英文/JavaScript**
- JS 範例必須符合 **ECMAScript 5.1**（Nashorn 限制）
- 所有範例使用 `var` 而非 `let`/`const`，不用箭頭函式
