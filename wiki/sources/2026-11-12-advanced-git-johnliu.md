---
title: "來源摘要：後端工程師的第二堂課 (1) 進階 Git"
type: source
domain: [workflow, backend]
tags: [git, version-control, backend-course, johnliu]
sources:
  - "raw/articles/2026-11-12-advanced-git-johnliu.md"
updated: 2026-11-12
status: evergreen
---

# 來源摘要：後端工程師的第二堂課 (1) 進階 Git

- **作者**：Johnliutw (John Liu)
- **發布日期**：2024-10-13
- **原始檔案**：[`raw/articles/2026-11-12-advanced-git-johnliu.md`](../../raw/articles/2026-11-12-advanced-git-johnliu.md)
- **原文連結**：[Medium 原文](https://medium.com/johnliu-%E7%9A%84%E8%BB%9F%E9%AB%94%E5%B7%A5%E7%A8%8B%E6%80%9D%E7%B6%AD/%E5%BE%8C%E7%AB%AF%E5%B7%A5%E7%A8%8B%E5%B8%AB%E7%9A%84%E7%AC%AC%E4%BA%8C%E5%A0%82%E8%AA%B2-1-%E9%80%B2%E9%9A%8E-git-675c5a7fb6ec)

---

## 核心主旨

本文為《後端工程師的第二堂課》系列首篇。作者指出，僅熟悉 `git add`、`git commit`、`git push` 等基礎指令會限制開發與協作的靈活性。文章從五大實務開發情境切入，梳理進階 [[tools/git|Git]] 操作概念與版本控制心法（詳見 [[playbooks/git-version-control-and-history-management|Git 版本控管與歷史紀錄管理實戰指南]]）。

---

## 重點提煉

1. **多功能並行開發**：
   - 使用 `git branch` 與 `git checkout` 隔離不同功能的開發線。
   - 合併時使用 `git merge`，若遇多人修改同一處則需處理 Merge Conflict。
   - 階段性版本發佈時，透過 `git tag v1.0.0` 在特定 Commit 釘上不可變的版本標籤。
2. **獲取與挑選他人程式碼**：
   - `git fetch <remote> <branch>`：僅將遠端分支資訊下載至本地，不自動合併。
   - `git pull <remote> <branch>`：等價於 `git fetch` + `git merge`。
   - `git cherry-pick <commit-hash>`：當某開發分支混入多個功能，但臨時需要將其中一個小功能單獨抽離並釋出至 `main` 時，可直接複製指定 Commit。**前提是每個 Commit 必須具備高獨立性（Atomic Commit）**。
3. **檢視歷史與追查軌跡**：
   - `git log`：檢視分支上的 Commit 歷史。
   - `git reflog`：檢視本機執行過的所有 HEAD 移動軌跡（包含 `--amend`、`reset`、`checkout`），是誤刪分支或重設過頭時的救命工具。
   - `git blame <file>`：逐行追查程式碼作者、時間與 Commit Hash，用於釐清設計意圖與排查 Bug。
4. **修改已提交的歷史紀錄**：
   - `git commit --amend`：補漏檔案或修改最後一次 Commit 訊息（本質是產生新 Commit 取代舊 Commit，已 Push 至遠端時須謹慎）。
   - `git reset HEAD~N`：將最近 N 個 Commit 退回工作區（程式碼保留不消失），適合在開發階段收斂零碎的測試 Commit。
   - `git revert <commit-hash>`：針對已發佈或歷史久遠（如兩個月前）的 Commit，計算並產生一個「完全相反」的新 Commit 來抵銷變更，不破壞既有歷史線。
   - `git rebase`：用於複雜的多筆 Commit 互動式整理（合併、刪除、改訊息）。
5. **開發中斷與暫存管理 (`stash`)**：
   - 面對緊急插單（隕石開發），避免產生無意義的暫時 Commit，改用 `git stash` 將未完成變更收入暫存倉。
   - `git stash list` 查看暫存清單；`git stash pop` 取出並移除暫存紀錄；`git stash apply` 取出但保留暫存紀錄（可跨多分支重複套用）。
6. **版本控制排除規範 (`.gitignore`)**：
   - 以 Laravel 為例，明確排除：外部依賴套件（`/node_modules`, `/vendor`）、環境變數與機密金鑰（`.env*`, `/storage/*.key`）、測試與 IDE 快取（`/.phpunit.cache`, `/.idea`, `/.vscode`）、前端建置產物与 Log。

---

## 附錄：《後端工程師的第二堂課》系列大綱（供知識庫後續追蹤）

- **程式語言與工具**：進階 Git、Vim 與 Nano、如何 Debug、Functional Programming、RegEx 正規表達式
- **基礎電腦科學**：進階物件導向、Design Pattern、常見演算法、密碼學基礎
- **資料庫技術**：Transaction、Index、Migration、NoSQL
- **應用程式設計**：RESTful API、驗證與授權、套件與語言版本管理、Unit Test
- **系統設計**：Cache、Load Balancing、Queue、微服務、分散式系統
- **系統基礎設施設計**：Network、Linux 運作方式、Docker、Kubernetes、CI/CD
