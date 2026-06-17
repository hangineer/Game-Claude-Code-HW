---
description: 將靜態網站部署到 GitHub Pages
argument-hint: [branch]
allowed-tools: Bash(git *), Bash(gh *), Read, Grep, Glob
---

將此專案部署到 GitHub Pages。可選參數 `$ARGUMENTS` 指定來源分支（預設 `main`）。

## 現況

- GitHub 登入：`!`gh auth status 2>&1 | head -n 1``
- 是否為 Git 專案：`!`git rev-parse --is-inside-work-tree 2>/dev/null || echo "false"``
- Git 狀態：`!`git status --short 2>/dev/null``
- 遠端：`!`git remote get-url origin 2>/dev/null || echo "(no remote)"``
- 目前分支：`!`git branch --show-current 2>/dev/null``
- Pages 設定：`!`gh api repos/{owner}/{repo}/pages 2>/dev/null || echo "(尚未啟用 GitHub Pages)"``

## 部署流程

依序執行，每步完成後簡短回報結果。

### 1. 確認登入與專案狀態

- 檢查 `gh auth status`。若未登入，**停止後續步驟**，告知使用者尚未登入 GitHub，並引導其執行 `gh auth login`（提示可用 `! gh auth login` 在對話中直接執行，依畫面指示完成瀏覽器或裝置碼驗證）。確認登入成功後才繼續。
- 檢查是否在 Git 專案中（`git rev-parse --is-inside-work-tree`）。若不是，執行 `git init` 建立本地儲存庫。
- 檢查是否已設定遠端 `origin`。若沒有：
  1. 詢問使用者要建立的 repo 名稱（預設用目前資料夾名稱）與可見度（public/private）。
  2. 若尚無任何 commit，先取得使用者同意後建立初始 commit。
  3. 執行 `gh repo create {name} --{public|private} --source=. --remote=origin --push` 建立新 GitHub repo 並設為 `origin`。
  4. 若建立失敗（例如名稱重複），回報錯誤並請使用者提供新名稱或改用既有 repo。

### 2. 確認可部署內容

- 掃描專案根目錄，找出 GitHub Pages 可服務的靜態檔（例如 `index.html`、`assets/`、`css/`、`js/`）。
- 若沒有 `index.html`，在根目錄建立一個最小可用的 `index.html`（內容可沿用專案主題），再繼續。
- 若有建置步驟（`package.json` 的 `build`、`Makefile`、`docs/` 等），先執行建置，並確認輸出目錄正確。

### 3. 整理 Git 狀態

- 若有未提交變更，先向使用者確認是否要一併提交；**未經明確同意不要 commit**。
- 若使用者同意提交，依變更內容撰寫簡潔的 commit message 後提交。
- 將來源分支（`$ARGUMENTS` 或 `main`）推送到 `origin`（步驟 1 已確保 `origin` 存在）。

### 4. 啟用 / 更新 GitHub Pages

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

### 5. 驗證

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
