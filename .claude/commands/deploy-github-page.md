---
description: 將靜態網站部署到 GitHub Pages
argument-hint: [branch]
allowed-tools: Bash(git *), Bash(gh *), Read, Grep, Glob
---

將此專案部署到 GitHub Pages。可選參數 `$ARGUMENTS` 指定來源分支（預設 `main`）。

## 現況

- Git 狀態：`!`git status --short``
- 遠端：`!`git remote get-url origin 2>/dev/null || echo "(no remote)"``
- 目前分支：`!`git branch --show-current``
- Pages 設定：`!`gh api repos/{owner}/{repo}/pages 2>/dev/null || echo "(尚未啟用 GitHub Pages)"``

## 部署流程

依序執行，每步完成後簡短回報結果。

### 1. 確認可部署內容

- 掃描專案根目錄，找出 GitHub Pages 可服務的靜態檔（例如 `index.html`、`assets/`、`css/`、`js/`）。
- 若沒有 `index.html`，在根目錄建立一個最小可用的 `index.html`（內容可沿用專案主題），再繼續。
- 若有建置步驟（`package.json` 的 `build`、`Makefile`、`docs/` 等），先執行建置，並確認輸出目錄正確。

### 2. 整理 Git 狀態

- 若有未提交變更，先向使用者確認是否要一併提交；**未經明確同意不要 commit**。
- 若使用者同意提交，依變更內容撰寫簡潔的 commit message 後提交。
- 將來源分支（`$ARGUMENTS` 或 `main`）推送到 `origin`。

### 3. 啟用 / 更新 GitHub Pages

從 `git remote get-url origin` 解析 `owner` 與 `repo`，然後：

1. 判斷部署來源：
   - **靜態站點在 repo 根目錄** → `source.path` 用 `/`
   - **建置輸出在子目錄**（如 `dist/`、`docs/`、`public/`）→ 用該子目錄路徑
2. 嘗試啟用 Pages（legacy、從分支部署）：

```bash
gh api --method POST /repos/{owner}/{repo}/pages \
  -f build_type=legacy \
  -f 'source[branch]={branch}' \
  -f 'source[path]={path}'
```

若回傳 409（已存在），改用 PUT 更新：

```bash
gh api --method PUT /repos/{owner}/{repo}/pages \
  -f build_type=legacy \
  -f 'source[branch]={branch}' \
  -f 'source[path]={path}'
```

3. 查詢部署狀態與網址：

```bash
gh api /repos/{owner}/{repo}/pages --jq '{url: .html_url, status: .status, source: .source}'
```

### 4. 驗證

- 回報預期網址：`https://{owner}.github.io/{repo}/`
- 若 `gh api` 顯示 `status` 為 `building`，告知需等待 1–3 分鐘。
- 部署完成後，用 `curl -I` 檢查網址是否回傳 200；若尚未就緒，說明如何手動確認。

## 輸出格式

最後用以下格式總結：

```
✅ GitHub Pages 部署完成
- Repo: {owner}/{repo}
- Branch: {branch}
- Path: {path}
- URL: https://{owner}.github.io/{repo}/
- Status: {status}
```

若任一步驟失敗，說明錯誤原因與建議修復方式（權限不足、遠端未設定、repo 為 private 且無 Pages 權限等），不要靜默跳過。
