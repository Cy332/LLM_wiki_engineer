---
title: "Git 分散式版本控制系統實務"
type: tool
domain: [workflow, backend]
tags: [git, version-control, cli, devtools]
sources:
  - "[[sources/2026-11-12-advanced-git-johnliu]]"
updated: 2026-11-12
status: evolving
---

# Git 分散式版本控制系統實務

## 1. 核心解決什麼問題

在多人協作與多任務並行的軟體工程中，`Git` 解決了以下核心痛點：
- **多開發線隔離**：透過輕量級指標（Branch）讓多個 Feature、Bugfix 同時進行而不互相污染。
- **完整歷史追溯與課責**：紀錄每次變更的作者、時間與上下文（`log`、`blame`），並提供本機操作快照（`reflog`）防止誤操作遺失程式碼。
- **靈活的變更搬移與回滾**：支援跨分支精準摘取變更（`cherry-pick`）、本地歷史重整（`reset`、`rebase`）與安全線上退版（`revert`）。

> 關於團隊日常開發的標準作業流程與歷史改寫決策樹，請參閱：[[playbooks/git-version-control-and-history-management|Git 版本控管與歷史紀錄管理實戰指南]]。

---

## 2. 核心機制與常用指令對照

### 2.1 分支切換與新舊語法對比 (`checkout` vs `switch` / `restore`)

在傳統教材與實務中（如 [[sources/2026-11-12-advanced-git-johnliu|進階 Git]]），常使用 `git checkout` 處理分支切換。但因 `checkout` 同時承擔「切換分支」與「還原檔案」兩種職責，Git 2.23+ 引入了語意更明確的 `switch` 與 `restore`：

| 操作目的 | 傳統語法 (`checkout`) | 現代語法 (`switch` / `restore`) | 實務備註 |
| :--- | :--- | :--- | :--- |
| 建立並切換至新分支 | `git checkout -b feature_1` | `git switch -c feature_1` | 兩者皆常用，腳本中 `checkout` 仍佔多數 |
| 切換至既有分支 | `git checkout feature_1` | `git switch feature_1` | `switch` 會防止誤將檔名當成分支名 |
| 放棄工作區某檔案的修改 | `git checkout -- file.php` | `git restore file.php` | 強烈建議用 `restore`，避免語意混淆 |
| 將檔案從暫存區 (Staging) 退回 | `git reset HEAD file.php` | `git restore --staged file.php` | 取消 `git add` 的直觀做法 |

---

### 2.2 遠端同步機制 (`fetch` vs `pull`)

- **`git fetch <remote> <branch>`**：
  - **行為**：只向遠端儲存庫抓取最新的 Commit 與分支指標（更新本地的 `origin/<branch>`），**不會**修改當前工作目錄的程式碼。
  - **適用場景**：想先檢視他人寫了什麼、或確認遠端與本地差異（`git diff origin/main`）後再決定如何整合。
- **`git pull <remote> <branch>`**：
  - **行為**：封裝了 `git fetch` + `git merge`（若設定 `--rebase` 則為 `fetch` + `rebase`）。
  - **常見坑點**：在多人同時推送同一條分支時，直接 `git pull` 常會自動產生一個多餘的 `Merge branch '...'` Commit，使線圖出現分岔迴圈。實務上建議使用 `git pull --rebase` 維持線性歷史。

---

### 2.3 暫存區管理 (`git stash`)

當正在開發 `feature_A`，突然遇到緊急 Hotfix 或隕石插單，手邊程式碼尚未完成、不適合產生正式 Commit 時使用：

```bash
# 1. 將當前工作區與暫存區的修改收進 Stash 倉庫（建議加上 -u 包含未追蹤的新檔案，並用 -m 寫備註）
$ git stash push -u -m "WIP: user auth validation"

# 2. 查看目前所有的 Stash 清單（包含所屬分支與基於哪個 Commit）
$ git stash list
# 輸出範例：stash@{0}: On main: WIP: user auth validation

# 3A. 取出最新一筆 Stash 並從清單中移除（最常用）
$ git stash pop

# 3B. 取出最新一筆 Stash 但「保留」在清單中（適合將同一份修改套用到多個不同分支）
$ git stash apply stash@{0}
```

---

### 2.4 歷史檢視與追查三劍客 (`log` / `reflog` / `blame`)

1. **`git log`（檢視分支歷史）**：
   - 實務推薦別名指令：`git log --oneline --graph --decorate --all`，可在終端機畫出清晰的 ASCII 分支合併圖。
2. **`git blame <file>`（逐行追查意圖）**：
   - 用途：排查 Bug 或重構遺留程式碼（Legacy Code）時，查出某一行是誰、在什麼時間、透過哪一筆 Commit 寫入的，進而再去讀該 Commit Message 了解當時的商業邏輯脈絡。
   - 進階參數：`git blame -L 50,80 notificationService.php`（只看第 50 到 80 行）。
3. **`git reflog`（本機時光機 / 後悔藥）**：
   - **運作原理**：即使你執行了 `git reset --hard`、`git commit --amend` 或不小心 `git branch -D` 刪除了未合併的分支，只要該 Commit 曾經在本機產生過，Git 預設會在本地保留 30~90 天的 Reference Log。
   - **救援範例**：
     ```bash
     $ git reflog
     # 找到誤刪或 reset 前的那筆操作，記下前面的 hash（例如 a1b2c3d 或 HEAD@{2}）
     $ git reset --hard a1b2c3d
     ```

---

## 3. 版控衛生與 `.gitignore` 安全規範

專案初始化時必須配置 `.gitignore`，避免將環境依賴、編譯產物與敏感資訊推上儲存庫。

### 必忽略的五大類別
1. **外部依賴套件**：`/node_modules` (Node.js)、`/vendor` (PHP/Go)、`.venv/` (Python)。這些應透過 Lock file (`package-lock.json`, `composer.lock`, `poetry.lock`) 還原。
2. **環境變數與機密憑證（極重要！）**：`.env`、`.env.*`（保留 `.env.example` 範本即可）、`*.key`、`*.pem`、`auth.json`。
3. **框架與建置產物**：`/public/build`、`/public/hot`、`dist/`、`target/`、`__pycache__/`。
4. **執行期快取與測試報告**：`/.phpunit.cache`、`.phpunit.result.cache`、`.pytest_cache/`。
5. **個人 IDE 與作業系統檔案**：`/.idea`、`/.vscode`、`/.fleet`、`.DS_Store` (macOS)。

---

## 4. 常見坑點 (Gotchas & Anti-patterns)

- **坑點 1：事後補加 `.gitignore` 卻發現檔案還是被追蹤**
  - **原因**：`.gitignore` 只對「尚未被 Git 追蹤 (Untracked)」的檔案生效。如果 `.env` 已經被 commit 過，事後加進 `.gitignore` 仍會持續被追蹤。
  - **解法**：使用 `git rm --cached .env` 將檔案從 Git 索引中移除（同時保留本機實體檔案），再提交一次 Commit。*（注意：若機密金鑰已推上遠端，除了移除追蹤，必須立即到雲端平台 Rotate / 撤銷該金鑰！）*
- **坑點 2：`git stash` 預設不會收納「全新建立 (Untracked)」的檔案**
  - **原因**：直接打 `git stash` 只會暫存「已追蹤 (Tracked)」檔案的修改。若你新建了檔案卻沒 `git add`，切過去別的分支時該檔案還會留在工作區。
  - **解法**：養成使用 `git stash -u`（`--include-untracked`）的習慣。
- **坑點 3：對已 Push 到遠端的共享分支執行 `--amend` 或 `reset`**
  - **後果**：這會改寫 Commit SHA-1 值，導致本地與遠端歷史分岔（必須 `git push --force`），若同事已經基於舊 Commit 開發，會造成嚴重的歷史混亂。詳見 [[playbooks/git-version-control-and-history-management|Git 版本控管與歷史紀錄管理實戰指南]]。

---

## 5. 關聯頁面
- [[playbooks/git-version-control-and-history-management|Git 版本控管與歷史紀錄管理實戰指南]]
- [[sources/2026-11-12-advanced-git-johnliu|來源摘要：後端工程師的第二堂課 (1) 進階 Git]]
