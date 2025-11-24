# 海島七號防水工程師端 App — 技術架構與 V1 開發計畫

## 1. 目標與範圍
- **平台**：iOS 15+，iPhone 為主（iPad 先以放大模式兼容）。
- **主要角色**：Admin、Engineer（Back Office 視未來版本加入）。
- **V1 聚焦**：登入/權限、案件管理（基本）、現場評估、基礎估價、報價、委託書與電子簽名、P1 收款、每日施工日誌（含離線暫存）、PDF 匯出（核心文件）。

## 2. 系統架構概覽
- **用戶端**：SwiftUI + Combine，搭配 **Realm** 作為離線優先的本地資料庫（支援加密與同步衝突解決策略）。
- **API 層**：RESTful JSON API（未來可平滑升級 GraphQL）。
- **身份驗證**：OAuth 2.0 / OpenID Connect（使用 Authorization Code + PKCE），存取 Token 保存在 Keychain；Refresh Token 受 iOS File Protection 限制。
- **檔案與圖片**：照片先落地加密 sandbox，再透過後台預簽名 URL 上傳至物件儲存（S3 兼容）。
- **PDF 生成**：客戶端使用 **PDFKit** 生成核心 PDF，模板以 JSON + 本地化字串定義；重型批次（未來）可交由後端服務。
- **推播**：V1 可先不做；V2+ 可用 APNs/FCM 轉接。

## 3. 關鍵技術決策
- **離線優先與同步**
  - 本地資料：Realm 加密（`NSFileProtectionCompleteUnlessOpen`）。
  - 同步機制：前端以背景 task 監測網路與電量，採「樂觀更新 + 版本號（`updated_at` + `version`）」；衝突時保留伺服器版本並另存本地變更為新草稿，提示合併。
  - 圖片：僅路徑與 metadata 進入資料表，實體檔案與上傳任務佇列分離。
- **模組化**：按領域拆分 Feature Modules（Auth, Cases, Survey, Estimate/Quote, Contract/Payment, DailyLog, Documents）。
- **安全性**：
  - Keychain 保存 Token；Realm 加密；照片暫存使用 FileProtection；HTTP 全站 TLS；禁用非 SSL。
  - 權限由後端 ACL 驗證，前端僅用於 UI 控管。
- **設定化**：材料單耗、等級成本、PDF 樣板以 Remote Config/版本化 JSON 拉取，便於無痛更新。

## 4. API 粗略契約（V1 範圍）
- `POST /auth/login`：取得 access/refresh token + 角色。
- `GET /cases?query=`：案件列表（依角色過濾）。
- `POST /cases`：新增案件。
- `GET/PUT /cases/{id}`：案件主檔與狀態欄位（stage、payment_status）。
- `POST /cases/{id}/surveys` + `POST /cases/{id}/surveys/{surveyId}/areas`
- `POST /cases/{id}/estimates`、`POST /cases/{id}/quotations`
- `POST /cases/{id}/contracts` + `/signatures`
- `POST /cases/{id}/payments`（期別：P1）
- `POST /cases/{id}/daily-logs`
- `POST /documents/render`：接受 JSON 輸入，回傳 PDF 下載網址（或 base64），V1 可選在前端自行渲染。

## 5. 本地資料模型（iOS）
- `UserSession`: tokens, role, expiresAt.
- `Case`: id, number, clientName, phone, address, type[], buildingType, age, engineerId, stage, paymentStatus (P1/2/3), warrantyStatus, timestamps.
- `Survey`: id, caseId, notes, status; `SurveyArea`: id, surveyId, code, category, locationDesc, areaSize, crackLength, issueDesc, level, photos[]
- `Estimate`: id, caseId, items[], materialCost, laborCost, totalCost, suggestedPrice.
- `Quotation`: id, caseId, lineItems[], total, paymentTerms, warrantySummary.
- `Contract`: id, caseId, terms, signatures (client/engineer), pdfPath.
- `Payment`: id, caseId, phase (P1), amount, date, method, receiptPhotos[].
- `DailyLog`: id, caseId, date, weather, zones[], content, materialsUsed[], workers, hours, photos[], isRainDay.
- `Document`: id, caseId, type, path, remoteUrl, version.
- 附加欄位：`syncStatus`（pending/synced/conflict），`updatedAt` 供衝突判斷。

## 6. PDF 模板（V1 必要）
- 現場評估表、估價表、報價單、委託書（含簽名）、P1 請款/收款單、每日施工日誌匯出。
- 模板設計：
  - JSON 定義：標題、欄位順序、樣式 token。
  - 本地化字串檔：中英對照，便於未來拓展。
  - PDFKit 生成，圖片壓縮與 lazy load，避免 OOM。

## 7. 品質與測試策略
- **單元測試**：資料模型、離線同步邏輯、金額計算公式（估價/付款）。
- **快照測試**：SwiftUI 主要畫面（案件列表、Dashboard、表單）。
- **整合測試**：API Mock（e.g., Moya + stub data）檢驗流程：案件建立 → 評估 → 估價 → 報價 → 委託 → P1 → 日誌。
- **離線情境測試**：飛航模式編輯/新增資料，恢復網路後同步並驗證衝突流程。

## 7.1 本地執行與測試步驟（How to Run & Test）
1) **前置環境**：macOS 13+、Xcode 15+、Swift 5.9、iOS 17 模擬器；需安裝最新 Command Line Tools。
2) **取得程式碼**：`git clone` 後，以 Swift Package Manager 管理第三方套件（不需 CocoaPods）。
3) **開啟專案**：雙擊 `EngineerApp.xcodeproj` 或 `EngineerApp.xcworkspace`（若使用多模組），確保 Scheme 已設定為 **EngineerApp**（App）與 **EngineerAppTests**（測試）。
4) **設定環境變數**：在 `Config/Debug.xcconfig` 與 `Config/Release.xcconfig`（或 Xcode Scheme 的 Environment Variables）填入 API_BASE_URL、OIDC_CLIENT_ID、OBJECT_STORAGE_BASE_URL 等；若後端尚未就緒，可在 Scheme 參數中開啟「Use Stub API」旗標以啟用本機 Mock（Moya stub）。
5) **跑 App（模擬器）**：Xcode 直接 Run，或用 CLI：
   ```bash
   xcodebuild -scheme EngineerApp -destination 'platform=iOS Simulator,name=iPhone 15,OS=17.2' clean build
   ```
6) **執行測試**：
   ```bash
   xcodebuild -scheme EngineerApp -destination 'platform=iOS Simulator,name=iPhone 15,OS=17.2' test
   ```
   - 覆蓋率報告可搭配 `-enableCodeCoverage YES`，並匯出 `.xcresult` 供 CI 上傳。
7) **SwiftLint／格式檢查（若有啟用）**：
   ```bash
   swiftlint lint
   ```
8) **離線情境手動驗證**：在模擬器或實機啟用飛航模式，新增/修改案件、日誌與照片 → 關閉飛航模式後確認同步佇列與衝突解決流程是否符合預期。
9) **PDF 確認**：在執行流程（評估、估價、報價、委託、P1、日誌）後，於文件中心檢視 PDF 頁面，確認樣板填充與簽名嵌入正確。

## 8. 開發排程（10 週建議，2 週 Sprint）
- **Sprint 1**：架構底座（SwiftUI/Combine/Realm）、Auth Flow、案件列表/詳細框架。
- **Sprint 2**：案件建立與狀態欄位、角色 UI 控管、基本離線同步（讀取）。
- **Sprint 3**：現場評估與分區 CRUD + 離線暫存、照片本地存取與上傳佇列。
- **Sprint 4**：估價計算（可配置公式）、估價表 PDF。
- **Sprint 5**：報價單產出、付款條件呈現、報價單 PDF。
- **Sprint 6**：委託書生成與電子簽名、PDF 嵌入簽名。
- **Sprint 7**：P1 請款/收款紀錄與狀態更新、材料清單占位 API（若後端未備妥可暫緩）。
- **Sprint 8**：每日施工日誌（含雨天標記）、離線暫存與同步、照片上傳。
- **Sprint 9**：文件匯出整合（集中下載列表）、角色權限驗證強化。
- **Sprint 10**：穩定性與性能優化、UAT、上架前準備（隱私權聲明、App 審核素材）。

## 9. V1 交付清單
- 可登入並依角色顯示可用功能。
- 案件列表/新增/基本狀態管理。
- 現場評估與分區管理（含 PDF）。
- 估價與報價流程（含 PDF）。
- 委託書生成 + 雙方簽名嵌入 PDF。
- P1 請款與收款紀錄，更新案件付款狀態。
- 每日施工日誌（雨天勾選）與離線暫存；網路恢復後自動同步。
- PDF 匯出入口，至少覆蓋 V1 文件類型。

## 10. 未來版本預留
- V2：施工期程與延期、變更單整合 P2、第二期請款/收款、案件進度條更完整。
- V3：完工通知、驗收紀錄與驗收證明、第三期尾款、保固主檔/保固處理、統計報表。

## 11. 工程治理與 DevOps
- 專案管理：Jira/Linear 以 FR 編號作 issue key。
- PR 流程：小步提交、PR 模板（變更摘要、風險、測試）。
- CI：Fastlane 單元測試 + Lint；danger/SwiftLint 檢查；產出測試覆蓋率。
- 發佈：TestFlight 分支版控（`develop` → `release/*` → `main`），自動打包內部分發。

## 12. 風險與緩解
- **離線同步複雜度**：先從單方向合併（伺服器優先 + 本地另存草稿），減少資料遺失；同步日誌可供 Debug。
- **估價公式未定**：以配置檔 + 版本化策略，允許後端下發新公式；客戶端僅負責展示與計算。
- **PDF 樣板變更頻繁**：採 JSON 模板與遠端配置，避免每次調整需發版。
- **影像容量**：照片壓縮、上傳前去除 EXIF 定位；非 Wi‑Fi 時可提示延後上傳。

