---
title: "來源摘要：【PM夥伴攻略】如何跟QA合作？ (3PM Lab)"
type: source
domain: [workflow]
tags: [pm, qa, testing, agile, feature-flags, shift-left, collaboration]
sources:
  - "raw/articles/2026-11-12-how-to-work-with-qa-3pmlab.md"
updated: 2026-11-12
status: evergreen
---

# 來源摘要：【PM夥伴攻略】如何跟QA合作？ (3PM Lab)

- **作者**：Anne Hsiao (3PM Lab 產品三眼怪實驗室)
- **發布日期**：2019-03-02
- **原始檔案**：[`raw/articles/2026-11-12-how-to-work-with-qa-3pmlab.md`](../../raw/articles/2026-11-12-how-to-work-with-qa-3pmlab.md)
- **原文連結**：[Medium 原文](https://medium.com/3pm-lab/how-to-work-with-qa-71c4621abf50)

---

## 核心主旨

本文從產品經理（PM）與研發團隊協作的視角，梳理軟體開發生命週期中不同角色的測試職責分工（工程師 Unit Test ➔ PM/設計師 Use Case Review ➔ QA E2E 與整合測試 ➔ DevOps/SRE 上線前自動化回歸測試），並提出五大高效協作心法。相關規範已整合至 [[playbooks/cross-functional-collaboration-pm-eng-qa|跨職能研發協作指南：PM、前後端、QA 與 SRE 職責邊界與交付標準]]。

---

## 重點提煉

1. **三層品質把關與職責分工**：
   - **工程師 (FE/BE)**：撰寫單元測試（Unit Test），並在 Code Review 時互相把關程式碼與測試覆蓋率。
   - **PM 與 UI/UX 設計師**：開發告一段落後進行 Use Case Review（對照 PRD 與 Mockup 進行初步驗收）。
   - **QA (Quality Assurance)**：負責 End-to-End (E2E) 與 Integration Testing。
     - **手動測試**：除正常流程外，專注於極端情境（Edge Cases）、錯誤處理、跨版本/跨裝置與權限邊界。
     - **自動化測試**：在程式碼 Merge 進主分支、準備上線前與 DevOps/SRE 緊密配合，針對核心既有模組（如註冊、登入、購物車、結帳）進行回歸測試（Regression Testing），防止改 A 壞 B。
2. **與 QA 高效協作的五大實務原則**：
   - **提供結構化測試維度清單**：除 PRD 外，主動列出「正常/極端/錯誤情境」、「未登入/已登入」、「功能開關狀態組合」、「目標裝置與瀏覽器」供 QA 展開 Test Cases（如管理於 TestRail）。
   - **測試左移 (Shift-Left Testing)**：讓熟悉全系統模組的 QA 在 **PRD 規劃階段** 就介入審閱，及早抓出規格邏輯漏洞與跨模組相依性（Dependencies）衝突。
   - **切小交付範圍 + 善用 Feature Flags**：避免大功能全部做完才一次丟給 QA（一次爆出 10 個 Bug、修 A 壞 B）。將大功能拆成 5 個小任務，測完一個就搭配 **Feature Flag（功能開關）** 隱藏上線，降低單次開發、Code Review 與測試的認知負荷。
   - **預留充足的測試與修 Bug Buffer Time**：改一行邏輯可能只需 10 分鐘，但展開多種邊界測試、錄製重現步驟（Reproduction Steps）、工程師修復到再次驗證（Retest）非常耗時，排程必須納入來回修正的 Buffer。
   - **建立健康的品質文化**：上線前被 QA 抓出 Defect 值得慶賀，遠勝於上線後引發客戶賠錢、用戶流失或公關危機。
