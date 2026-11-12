---
title: "後端工程師的第二堂課 (1) 進階 Git"
author: "Johnliutw (John Liu)"
source_url: "https://medium.com/johnliu-%E7%9A%84%E8%BB%9F%E9%AB%94%E5%B7%A5%E7%A8%8B%E6%80%9D%E7%B6%AD/%E5%BE%8C%E7%AB%AF%E5%B7%A5%E7%A8%8B%E5%B8%AB%E7%9A%84%E7%AC%AC%E4%BA%8C%E5%A0%82%E8%AA%B2-1-%E9%80%B2%E9%9A%8E-git-675c5a7fb6ec"
published_at: 2024-10-13
ingested_at: 2026-11-12
---

# 後端工程師的第二堂課 (1) 進階 Git

> 原文連結：[Medium - Johnliutw](https://medium.com/johnliu-%E7%9A%84%E8%BB%9F%E9%AB%94%E5%B7%A5%E7%A8%8B%E6%80%9D%E7%B6%AD/%E5%BE%8C%E7%AB%AF%E5%B7%A5%E7%A8%8B%E5%B8%AB%E7%9A%84%E7%AC%AC%E4%BA%8C%E5%A0%82%E8%AA%B2-1-%E9%80%B2%E9%9A%8E-git-675c5a7fb6ec)
> 發布日期：2024-10-13

這篇是 **後端工程師的第一堂課** 系列文章的進階篇，在第一堂課中，主要是想幫助讀者對於後端技術有個基本的認知。而在第二堂課的系列中，我們會深入後端技術的各個議題，並且討論一些實務上會需要取捨，或是花些時間研究的部分，如果你已經有一些軟體的開發經驗，那相信這個系列的內容會非常適合你!

這個系列會談到的議題包含:

### 程式語言與工具
1. 進階 Git
2. 純文字程式編輯器 — Vim 與 Nano
3. 如何 Debug
4. Functional Programming
5. RegEx 正規表達式

### 基礎電腦科學
1. 進階物件導向
2. Design Pattern
3. 常見演算法
4. 密碼學基礎

### 資料庫技術
1. 保證資料一致性: Transaction
2. 保證資料速度: Index
3. 保證資料更動紀錄: Migration
4. 非結構化資料: NoSQL

### 應用程式設計
1. API 設計: Restful
2. 驗證與授權
3. 套件與語言版本管理工具
4. 自動化測試: Unit Test

### 系統設計
1. 資料加速: Cache
2. 流量保護者，Load Balancing
3. 非同步任務處理，Queue
4. 微服務
5. 分散式系統

### 系統基礎設施設計
1. Network
2. Linux 的運作方式
3. 統一你的系統環境，Docker
4. 容器化協作系統，Kubernetes
5. CI/CD

【後端工程師的第二堂課】全系列: https://medium.com/@johnliutw/list/1145f50673e1

---

Git，是每位工程師基本上日常都會用到的工具，或是說一項技術。筆者在後端工程師的第一堂課，也曾經用一篇文章介紹了 Git 的最基礎指令。透過這些最基本的指令，幾乎可以完成所有日常基本的開發工作。但只會這些指令，或是說只會這些 Git 的「使用概念」，會限制我們的程式開發能力的發揮。

因此，這篇文章，會介紹使用 Git 的多項進階重要概念，並提供常用的相關指令，希望幫助讀者在使用 Git 的時候，能夠感覺，真正的在自由控制程式碼的版本。

## 第一部分: 多功能同時開發的情境

在一個稍有規模的團隊，使用 Git 一個最常見功能，就是當同時正在開發不同的功能時，要怎麼確保這些程式碼可以根據不同的功能，來累積不同的版本紀錄，避免彼此互相傷害的狀況。

### 1. git branch

首先，我們要先介紹 git 的世界裡，有個分支 (branch) 的觀念。以筆者撰寫文章的當下，所有新建立的 git 版本紀錄裡，都會先預設幫忙建立一個分支叫做 `main` ，也就是說，所有新的程式碼版本，其實都是基於 `main` 這樣的 branch，例如:

![Image 7](https://miro.medium.com/v2/resize:fit:700/1*RNOavWZT_hUeWep5uX2cuw@2x.png)

而如果我們想要額外拉出一條軟體的生產線來開發新的功能，我們時常會使用一個詞彙叫做「開分枝」，而這個詞彙實際上來說就是執行以下的指令，來建立一個叫做 `feature_1` 的新分支

```bash
$ git branch feature_1
```

當我們想確認目前有哪些分支時，就可以執行:

```bash
$ git branch
```

結果會如下:

![Image 8](https://miro.medium.com/v2/resize:fit:272/1*mEQtvl1ppN_pFWTs6zgLnw@2x.png)

同時，你會看到 `main` 這個分支有不同的顏色，這也是代表我們目前所在的位置在該分支。

那我們要怎麼跳到我們新建立的 `feature_1` 分支呢?

### 2. git checkout

非常直觀的，可以直接使用以下指令:

```bash
$ git checkout feature_1
```

就可以切換到我們新建立的分支進行開發囉，結果如下:

![Image 9](https://miro.medium.com/v2/resize:fit:700/1*2LPGG9y1t5TwfJDT5PMj1w@2x.png)

### 3. git merge

美好的旅程終有終點，如果我們開發好一個功能，想要將這些新的程式碼放入原本的分支中時，就可以先用 git 切換到原本的分支，並且使用

```bash
$ git merge {要合併進來的分支名稱}
```

例如:

```bash
$ git merge feature_1
```

就可以將新的程式碼放入目前所在的分支中。

那有時候，原本的分支可能其他人有在跟你改一樣的程式碼，這時候就會出現「CONFLICT」，例如這個情況:

![Image 10](https://miro.medium.com/v2/resize:fit:661/1*PEiLBkWmb5ZNDmFCSUoOiA.png)

而解決方式當然不是去跟那個人打一架看要用誰的程式碼，而是有一套解決流程。

### 4. git tag

如果事情一切順利，那我們先在針對目前的原本分支，想要對外宣告說: 我們完成了 v1.0.0 版本，請隨意取用，就可以使用 `git tag` 這個指令。

其使用方式也很直觀，就是先切換到你想要加上 tag 的分支，並執行:

```bash
$ git tag {你想宣告的版本}
```

例如:

```bash
$ git tag v1.0.0
```

這樣，git 就會幫你在目前分支所在位置的 **commit**，多加上 `v1.0.0` 這個 tag。當有人想來使用你的軟體時，就會知道要切換到哪個版本來使用 v1.0.0。

而當我們繼續開發，新增新的程式碼時，該 v1.0.0 tag 就仍會被保留在原本的 commit 中，例如這個 git 的紀錄圖:

![Image 11](https://miro.medium.com/v2/resize:fit:628/1*IpZdPGrAa1GMuGWmHYJzbA.png)

可以看到 v1.0.0 仍舊保留在原本舊的 commit 中。

## 第二部分: 如何獲取他人的程式碼

就像文章前半段所提，團隊開發的情境總是會有程式碼在雲端中，需要你下載下來到你的電腦中。最常見的有兩種需求:

* 下載他人的分支
* 在特定分支，下載他人新加的程式碼 (commit)

### 1. git fetch

如果想滿足第一點需求，我們可以用 `git fetch` 去抓取他人的分支，執行起來就像:

```bash
$ git fetch {建立的 remote 位置} {分支名稱}
```

例如:

```bash
$ git fetch remote feature_1
```

這時後，只要使用 `git checkout feature_1`，就可以直接指定切換到該新的 feature_1 分支囉，例如:

![Image 12](https://miro.medium.com/v2/resize:fit:700/1*BcRtzP5xzMS-rv1HDq7pjQ.png)

### 2. git pull

假設你下載下來的新分支，別人有更新程式碼了，我們可以使用 `git pull` 來把新的程式碼下載下來:

```bash
$ git pull {建立的 remote 位置} {分支名稱}
```

例如:

```bash
$ git pull origin feature_1
```

這個 pull 其實是兩種動作的簡化，會先使用 `fetch` 拿到最新資訊，再使用 `merge` 將程式碼更新到你的分支中，是不是很方便呢 ?

### 3. git cherry-pick

現在我們可以假設一個情境:

feature_1 開始開發已經一陣子了，且中間混入了許多其他的小功能。可是這時候呢，我們偉大的老闆想要把某個小功能，直接加在 main 這個核心的分支中，直接釋出給他人使用。

這時候，除了翻出當時小功能的程式碼，然後一筆一筆 copy-paste 外，就可以使用 `cherry-pick` 這個指令，來將特定的「 **commit** 」，直接複製到指定的分支中。

例如，我們該小功能的 commit 專屬值是 `abc123` ，那我們就可以先切換到 `main` 的分支，並且執行:

```bash
$ git cherry-pick abc123
```

git 就會很聰明的從茫茫 commit 海中，直接找到指定的 commit，然後把裡面所有的程式碼複製貼上到你所在的分支。並且會很貼心的直接幫你產生一個新的 commit，讓你不用再另外 `add` , `commit` 喔!

這裡我們可以學到，讓每個 commit 盡量具有獨立性是很重要的，這時我們就可以很輕鬆的自由搬動，而不會有太大的苦痛需要克服~

## 第三部分: 如何查看 git 紀錄

### 1. git log

正如你在前面所看到，如果要看到 git 在特定分支的紀錄，只需要使用 `git log` 就能查看所有的 commit，例如:

![Image 13](https://miro.medium.com/v2/resize:fit:700/1*8Oha4pVOvVq2kCJsgOpm6w@2x.png)

需要離開此畫面，只需要按下 `q` ，也就是 `quit` 之意，去關閉畫面。

### 2. git reflog

如果我們不只是想看到已產生的 log 紀錄，而還想要看到我們自己執行過哪些 git 指令，就可以使用 `git reflog` 指令，來看看這個例子:

![Image 14](https://miro.medium.com/v2/resize:fit:700/1*Gos8f20O_aVKjSqx0bzpww@2x.png)

你可以看到，筆者曾經的 `commit --amend` , `pull` , `checkout` 指令都被抓出，無所遁形。因此這個指令很適合用在不小心刪掉特定分支，或是忘記自己剛剛所在的分支時使用喔！

### 3. git blame

在找 bug 或是確認設計細節時，時常會有需求是去找特定程式碼是誰寫的，來確認意圖，這時 `git blame` 就可以幫你逐行分析該段程式碼是來自哪位作者。其語法為:

```bash
$ git blame {要被確認的檔案}
```

例如:

```bash
$ git blame notificationService.php
```

這時候的範例畫面如下:

![Image 15](https://miro.medium.com/v2/resize:fit:700/1*dfTcOqGikdeFK1fvldSGcQ@2x.png)

可以非常清楚地看到哪位作者 + 哪個時間 + 做了哪段程式碼。

## 第四部分: 如何修改已經 commit 的紀錄

我們現在可以收穫大量 commit 資訊了，但是呢，commit 這種東西，如果設定好就不能改，也太古板了。因此，我們來學會幾個可以修改 commit 的方法吧!

### 1. git commit --amend

如果我們在恍神中，不小心送出了 commit，忘記加到一些該附上的檔案，就可以使用 `git commit` 的小功能 `--amend` 。

具體來說，步驟是:

* 先使用 `git add` ，將剛剛忘記附上的檔案加入暫存區
* 執行 `git commit --amend`

這時，畫面會開啟你的電腦預設的文字編輯器，會有類似如下的畫面:

![Image 16](https://miro.medium.com/v2/resize:fit:687/1*bNf3y4yrGQoGb-9sBNUS4A.png)

這是你上一隻 commit 的紀錄，這時候只要修改完 commit 的訊息，並且儲存，就會更新完畢囉!

不過提醒一下，雖然我們看起來是編輯上一個 commit，但是其實 git 會產生一個新 commit 來覆蓋掉我們編輯的 commit。因此，如果 commit 已經 push 更新到遠端，這時候就要注意兩邊版本會不一致的問題喔!

那如果，你只是前一個 commit 的訊息想改字，可以快速使用:

```bash
$ git commit --amend -m '你的新訊息'
```

### 2. git reset

那如果我並沒有想要加任何的程式碼，而且發現剛剛做的 2隻 commit 都要復原才對，那就需要使用 `git reset` 這個指令。

比如:

```bash
$ git reset HEAD~2
```

`HEAD` 指的是你當前分支最新的 commit。因此這個指令會從最新的指令，開始往前，總共復原最後兩個 commit。

例如這邊有三個 commit，新增了 `a.txt` , `b.txt` , `c.txt`

![Image 17](https://miro.medium.com/v2/resize:fit:700/1*NIC0mRfBNF8fGX8OWigqIQ@2x.png)

![Image 18](https://miro.medium.com/v2/resize:fit:700/1*79H9kXqRSRIHuHzKJkCT6A@2x.png)

那我如果想要刪除最後兩個，使用 `git reset HEAD~2` 後，就可以移除 `c.txt` 和 `b.txt` ，並保留 `a.txt` 這個 commit。

而程式碼不會消失喔! 不需要擔心，你可以透過 `git status` 查看被 reset 的 commit 中的內容。

![Image 19](https://miro.medium.com/v2/resize:fit:700/1*dH52-L5pWEfSqWlFQSuM0A@2x.png)

reset 技巧很適合用在開發過程中，可以先產生很多測試用的 commit，再透過 reset 復原 commit，最後總整理一次的情境喔!

### 3. git revert

雖然我們學會了 git reset 來復原程式碼，但假設我想要復原的是 2 個月前的某個 commit，但我卻不想把我這 2個月的所有心血成果都重設，然後重新設置 commit，該怎麼辦呢?

這就是 `git revert` 的任務，它可以直接針對你想要復原的 commit，直接運算產生一個「**完全相反**」的新 commit 。

例如你的程式碼是新增 `1 + 1 = 2` ，那 revert 就會刪除它 ; 如果程式碼是刪除 `1 + 1 = 2` ，那 revert 就會把運算式加回來。其語法為:

```bash
$ git revert {commit 值}
```

例如:

```bash
$ git revert abc123
```

執行 revert 後，會產生如同 `git commit --amend` 執行後的畫面，因為是產生一個新的 commit：

![Image 20](https://miro.medium.com/v2/resize:fit:700/1*Y32C-lGjSTNjkXz8o8F4rQ@2x.png)

使用你的文字編輯器關閉後，就可以透過 `git log` 看到你產生的 commit，並且還能看到 git 很貼心的附上被 revert 的 commit 值

![Image 21](https://miro.medium.com/v2/resize:fit:700/1*cx-LSpThEYZx3N_hMR3Zmg@2x.png)

### 4. git rebase

如果你需要更複雜的行為，例如最新的 5 隻 commit 中，2 隻 commit 要合併、1 隻 commit 要刪除、1 隻 commit 要修改訊息。

這時候，你可以參考 `git rebase` 指令，也是實務中很常運用的指令。

## 第五部分: 如何暫時儲存當下開發的程式碼

假設你現在開發的功能有很多個，可是來得又快又急又隕石，那當下的程式碼該怎麼辦呢?

雖然有個很直觀的做法，就是可以先 `git commit` 產生一個暫時的 commit，然後再切換分支做開發。但這樣的壞處則是需要頻繁地使用 `git reset` 來整理 commit，以避免造成後續歷史雜亂。

另外如果有透過 `git rebase` 或是 `git squash` 整理 commit 時，也會增加複雜性。因此，如果需要暫時儲存程式碼時，推薦使用 `git stash` 來處理。

正如 stash 本身的字「倉」的意思，跟 stash 所有相關的指令，都是來討論怎麼處理程式碼暫存的問題。而我們現在就來看看有哪些 stash 相關的語法可以使用吧!

### 1. git stash, git stash list

當開發到一半的時候，就可以隨時使用以下指令:

```bash
$ git stash
```

![Image 22](https://miro.medium.com/v2/resize:fit:700/1*24KQXL3dgAD5V_eS3O61iw.png)

執行完畢後，可以使用 `stash list` 看到暫存下來的程式碼紀錄。例如:

```bash
$ git stash list
```

![Image 23](https://miro.medium.com/v2/resize:fit:485/1*q85UUUMlxpuv9H_73hCfjg.png)

其中，git 預設就會附上使用 stash 的資訊，包括
* 分支名稱: `main`
* 最後的 commit 值: `b42f924`
* 最後的 commit 訊息: `Revert "b.txt"`

### 2. git stash pop, git stash apply

在前個步驟我們知道怎麼儲存暫時的程式碼了，那就要來學會，怎麼使用這些程式碼。

首先，你可以直接執行 `pop` 的指令，就可以將程式碼從 stash 倉庫中拉出來，比如說:

```bash
$ git stash pop
```

你就可以將程式碼從倉庫中搬出來，直接使用在你當下的分支中，例如:

![Image 24](https://miro.medium.com/v2/resize:fit:700/1*HvjqKeDc-mcTNAJ_qb86Eg.png)

那如果說，你是想要「**使用**」該段程式碼，但不想要移除這段紀錄，就可以使用 `apply` 的方式，例如:

```bash
$ git stash apply
```

就可以使用最新儲存的程式碼，但又不會把它給刪除，可以在不同的分支使用。

### 番外篇: 有些檔案不用進 git 怎麼辦?

如果有些檔案，我們不希望他跑進去我們的 git 版本控制系統，例如一些產生的 log 檔案、或是套件檔案等等，我們就可以在專案的目錄建立 `.gitignore` 的檔案，並且附上哪些檔案不需要被 git 追蹤。

我們來看看 Laravel 框架中，預先設定好的檔案:

![Image 25](https://miro.medium.com/v2/resize:fit:167/1*AIvGHHCIerim-21wnK4sgw.png)

```gitignore
/.phpunit.cache  
/node_modules  
/public/build  
/public/hot  
/public/storage  
/storage/*.key  
/vendor  
.env  
.env.backup  
.env.production  
.phpunit.result.cache  
Homestead.json  
Homestead.yaml  
auth.json  
npm-debug.log  
yarn-error.log  
/.fleet  
/.idea  
/.vscode
```

簡單分析一下常見可以略過的檔案:

1. `/node_modules` , `/vendors` : Node.js 和 PHP 的外部套件
2. `.env` , `.env.backup` , `.env.production` : 環境變數用檔案
3. `/.phpunit.cache` , `.phpunit.result.cache` , `.fleet` : 快取用的資料
4. `/public/build` , `/public/hot` , `/public/storage` : 前端建構後產生的檔案
5. `/storage/*.key` : 密鑰相關的檔案，可以注意到，可以透過 `*` ，一次模糊比對所有符合條件的檔案名稱，例如: `/storage/a.key` , `/storage/b.key` 都可以符合
6. `npm-debug.log` , `yarn-error.log` : 前端相關的 log

## 結語

只要有人寫程式，就會有版本。未來，就算 AI 幫忙寫的程式越來越多，版本控制的問題仍會越來越複雜。希望這篇文章能夠幫助你在這個系列有個好的開始～
