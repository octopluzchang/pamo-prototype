# 和解書功能可移植規格文件

## 1. 文件目的

本文件整理 `pamo-individual-client` 專案中的「和解書」功能，目標是協助將此功能移植到另一個產品原型中，並保留其核心商業流程、必要資料結構與 API 需求。

此文件聚焦於：

- 功能流程
- 前端畫面組成
- 前端狀態與資料流
- 後端 API 介面需求
- 最小可用搬移範圍
- 建議拆分策略

不聚焦於：

- 現有品牌視覺
- 非關鍵裝飾元件
- 保險、付款、會員等其他模組

## 2. 功能定位

此功能用於處理車禍雙方的線上和解書建立與確認流程，前提條件為：

- 雙方皆無人受傷，或至少流程文案假設此為主要適用情境
- 雙方不走保險理賠流程
- 使用者已登入
- 使用者已有事故紀錄可作為和解書基礎資料

核心價值：

- 先根據事故紀錄建立和解內容
- 對方透過簡訊收到 PDF 與 6 位數同意碼
- 建立方輸入同意碼完成驗證
- 系統產生正式簽署版 PDF

## 3. 使用者流程

### 3.1 主流程

1. 使用者從和解書主選單進入
2. 查看未完成事故紀錄列表
3. 選擇某筆事故紀錄進入預覽頁
4. 點擊「製作和解書」
5. 填寫和解條件與雙方聯絡資訊
6. 送出建立和解書請求
7. 系統回傳預覽版和解書資訊
8. 系統發送簡訊給對方，內含 PDF 與 6 位數同意碼
9. 建立方取得對方同意碼後輸入系統
10. 系統驗證成功後產出正式和解 PDF
11. 使用者可在已完成和解書列表查看正式 PDF

### 3.2 畫面流程圖

```text
和解書主選單
  -> 未完成事故列表
  -> 事故預覽
  -> 和解書表單
  -> 雙方確認 / 輸入同意碼
  -> 和解完成
  -> 已完成和解書列表
```

## 4. 前端模組對應

### 4.1 功能入口頁

- 主選單：`/src/pages/ContractMenu.vue`
- 未完成列表：`/src/pages/EventList.vue`
- 事故預覽：`/src/components/EventView.vue`
- 和解書表單：`/src/components/ContractForm.vue`
- PDF 預覽：`/src/components/PreviewContract.vue`
- 已完成列表：`/src/pages/ContractList.vue`
- 路由入口：`/src/router/routes/contract.js`

### 4.2 和解書功能的實際核心

真正需要移植的核心不在主選單，而在以下三塊：

1. `ContractForm.vue`
2. contracts Vuex module
3. records Vuex module

原因：

- `ContractForm.vue` 承載主要商業規則
- contracts module 封裝建立、發送與驗證 API
- records module 提供事故紀錄資料，作為和解書建立來源

## 5. 畫面與互動規格

### 5.1 和解書表單 Step 0：撰寫合約

畫面目的：

- 收集和解條件
- 帶入我方資料
- 輸入對方資料

欄位分區如下。

#### A. 賠償者

選項：

- `CREATOR`：我方
- `OTHERS`：對方
- `null`：互不賠償

前端來源：Vuex 全域 `compensator_options`

#### B. 賠償方式

選項組合：

- 只賠錢
- 只修車
- 賠錢 + 修車

最終對應欄位：

- `FIXED_AMOUNT`
- `REPAIRMENT`
- `REPAIRMENT_FIXED_AMOUNT`

#### C. 金錢賠償欄位

當選擇包含賠錢時顯示：

- `compensationAmount`
- `paymentType`
  - `CASH`
  - `TRANSFER`
- `transferDays`
  - 僅在 `paymentType === TRANSFER` 時需要

#### D. 修車條件欄位

當選擇包含修車時顯示：

- `repairShopDecisionSide`
  - `CREATOR`
  - `OTHERS`
- `isOemOnly`
  - `true`
  - `false`

#### E. 我方資訊

由登入者 profile 自動帶入：

- `creatorName`
- `creatorMobile`

來源：`state.auth.profile`

#### F. 對方資訊

手動輸入：

- `oppositeName`
- `oppositeMobile`

#### G. 驗證規則

必填：

- 我方姓名
- 我方手機
- 對方姓名
- 對方手機

條件必填：

- 若含賠錢，則 `compensationAmount` 必填
- 若賠錢且付款方式為轉帳，則 `transferDays` 必填

### 5.2 和解書表單 Step 1：雙方確認

畫面目的：

- 告知已發送至對方手機
- 可預覽 PDF
- 輸入對方提供的 6 位數同意碼
- 可重新發送簡訊

功能項目：

- 顯示對方手機號碼
- 開啟 PDF 預覽 modal
- 撥號給對方
- 輸入 `validationCode`
- 點擊重新發送簡訊

### 5.3 和解書表單 Step 2：完成頁

畫面目的：

- 告知和解完成
- 提供撤回告訴狀下載連結

此頁屬於附加說明，可視原型需求簡化。

### 5.4 已完成和解書列表

顯示欄位：

- 地點：`accidentRecord.city + accidentRecord.area`
- 日期：`createdTime`
- 開啟正式 PDF：`signedPdf`

### 5.5 未完成事故列表

顯示欄位：

- 地點：`city + area`
- 日期：`occuredTime`

點擊後進入事故預覽頁，並從該頁進一步建立和解書。

## 6. 資料流規格

### 6.1 資料流總覽

```text
登入者 profile
  -> 自動填入我方資料

事故紀錄 record
  -> 提供事故背景
  -> 提供事故 id
  -> 作為建立和解書的基底

建立和解書 API
  -> 回傳 currentContract
  -> 包含 previewedPdf

發送 challenge API
  -> 發簡訊給對方

驗證 verify API
  -> 驗證同意碼
  -> 完成正式文件生成

完成列表 API
  -> 顯示 signedPdf
```

### 6.2 前端 state 需求

#### records module

需要保存：

- `records`
- `currentRecord`
- `requestingRecords`
- `requestingCurrentRecord`
- `requestingCreateRecord`

#### contracts module

需要保存：

- `contracts`
- `currentContract`
- `requestingContracts`
- `requestingCreateContract`
- `requestingSendContractToOpposite`
- `requestingVerifyContract`

#### auth module

至少需要：

- `profile`
- `token`
- `isAuthenticated`

## 7. 資料模型規格

### 7.1 Accident Record

前端建立和解書最少依賴以下欄位：

```json
{
  "id": "record-id",
  "occuredTime": "2021-01-20T15:10:50Z",
  "zipCode": "114",
  "city": "台北市",
  "area": "內湖區",
  "address": "瑞光路100號",
  "locationDescription": "地點備註",
  "accidentParties": [
    {
      "identity": "DRIVER",
      "plateNumber": "ABC-1234",
      "injured": false
    },
    {
      "identity": "DRIVER",
      "plateNumber": "BBC-1234",
      "injured": false
    }
  ]
}
```

### 7.2 Contract Create Payload

```json
{
  "accidentRecordId": "record-id",
  "compensationAmount": 10000,
  "compensationType": "FIXED_AMOUNT",
  "isOemOnly": false,
  "partyList": [
    {
      "creator": true,
      "mobile": "0912345678",
      "name": "王小明"
    },
    {
      "creator": false,
      "mobile": "0987654321",
      "name": "陳小華"
    }
  ],
  "payer": "CREATOR",
  "paymentType": "CASH",
  "repairShopDecisionSide": "CREATOR",
  "transferDays": 3
}
```

注意：

- `payer` 可為 `null`，表示互不賠償
- 當 `payer === null` 時，部分賠償欄位可不送或由後端忽略
- `isOemOnly` 僅在修車相關情境有意義
- `transferDays` 僅在 `paymentType === TRANSFER` 時有意義

### 7.3 Contract Response

建立成功後前端至少需要這些欄位：

```json
{
  "id": "contract-id",
  "previewedPdf": "path/to/preview.pdf",
  "signedPdf": "path/to/signed.pdf",
  "createdTime": "2021-01-20T15:30:50Z",
  "accidentRecord": {
    "city": "台北市",
    "area": "內湖區"
  }
}
```

注意：

- Step 1 預覽依賴 `previewedPdf`
- 完成列表依賴 `signedPdf`

## 8. API 規格

### 8.1 取得未完成事故列表

`GET /api/account/:profileId/my-record?hasSignedContract=false`

用途：

- 顯示尚未轉成正式和解書的事故紀錄列表

前端需求：

- 回傳陣列

### 8.2 取得單筆事故紀錄

`GET /api/accident-record/:id`

用途：

- 顯示事故預覽頁
- 提供建立和解書時所需的 `accidentRecordId`

### 8.3 建立和解書

`POST /api/accident-contract`

用途：

- 建立和解書草稿或預覽版

請求內容：

- 見第 7.2 節

回應至少需包含：

- `id`
- `previewedPdf`

### 8.4 發送簡訊與同意碼

`POST /api/accident-contract/:id/challenge`

請求內容：

```json
{
  "mobile": "0987654321",
  "mobileCountryCode": "886"
}
```

用途：

- 發送 PDF 與 6 位數同意碼給對方

### 8.5 驗證同意碼

`POST /api/accident-contract/:id/verify`

請求內容：

```json
{
  "code": "123456"
}
```

用途：

- 驗證對方同意碼
- 完成正式文件生成

### 8.6 取得已完成和解書列表

`GET /api/account/:profileId/accident-contract?isSigned=true&page=0&size=10000`

用途：

- 顯示已完成和解書列表

前端需求：

- 目前實作假設後端回傳分頁物件，資料位於 `content`

## 9. 檔案與服務依賴

### 9.1 前端套件依賴

和解書功能目前依賴：

- `vuex`
- `vuelidate`
- `axios`
- `vue-loading-overlay`
- `pdfh5`
- `date-fns`

### 9.2 環境變數

至少需要：

- `VUE_APP_API_HOST`
- `VUE_APP_S3_HOST`

說明：

- `VUE_APP_API_HOST`：API base URL
- `VUE_APP_S3_HOST`：PDF 檔案實際存放位置或 CDN / object storage host

### 9.3 認證需求

目前 API 呼叫依賴 Bearer token。

前端共用請求流程會：

- 檢查 token 是否有效
- 自動帶入 `Authorization: Bearer <token>`

因此新原型若未接登入系統，至少要先提供其中一種替代方案：

- mock token
- mock API
- 暫時移除 auth guard

## 10. 可移植範圍分級

### 10.1 最小可用版 M1

建議最先搬移：

- 事故列表
- 事故預覽
- 和解書表單
- PDF 預覽
- 建立 / 發送 / 驗證 API 串接
- 已完成列表

這一版可以先不包含：

- 主選單特殊文案
- 背景圖與裝飾圖
- 撤回告訴狀下載
- 舊版視覺樣式

### 10.2 增強版 M2

可補回：

- 文案提醒
- loading overlay
- PDF modal 體驗優化
- 已完成列表外部開啟行為

### 10.3 完整版 M3

可再補：

- 品牌化視覺
- 完整事故建檔流程
- 手機號碼區碼國際化
- 更完整錯誤處理與 retry 機制

## 11. 建議搬移策略

### 策略 A：完整複製原流程

適用情境：

- 新 prototype 也有事故紀錄概念
- 可以承接 record -> contract 的流程

優點：

- 商業流程最完整
- 與現有系統行為最接近

缺點：

- 依賴較多
- 需要 records 與 auth 模組一起接入

### 策略 B：抽成獨立和解書精簡版

適用情境：

- 新 prototype 只想展示和解書功能
- 不想先做完整事故建檔

做法：

- 用外部傳入的事故資料取代 `currentRecord`
- 表單直接接受 `recordSummary` 或 `caseContext`
- 保留 `createContract`、`challenge`、`verify`

優點：

- 導入速度最快
- 適合 demo 或 prototype

缺點：

- 與舊系統耦合較少，但要自己定義資料入口

## 12. 建議抽象介面

若要重做成更可移植版本，建議將現有功能抽成以下邊界。

### 12.1 Domain Service

建立 `settlementService`，封裝：

- `fetchPendingRecords()`
- `fetchRecordById(id)`
- `createSettlement(payload)`
- `sendSettlementChallenge(id, mobile, mobileCountryCode)`
- `verifySettlement(id, code)`
- `fetchSignedSettlements()`

### 12.2 UI Components

拆成：

- `SettlementForm`
- `SettlementVerification`
- `SettlementPdfPreview`
- `SettlementList`
- `AccidentRecordSummary`

### 12.3 Data Adapter

建立 adapter 將舊系統欄位轉為新 prototype 欄位，例如：

- `accidentRecord` -> `caseRecord`
- `previewedPdf` -> `previewPdfUrl`
- `signedPdf` -> `finalPdfUrl`

## 13. 已知耦合點與風險

### 13.1 與 auth 強耦合

建立與查詢 API 都依賴 `profile.id` 與 token。

### 13.2 與事故紀錄模組強耦合

和解書不是獨立產生，而是依附事故紀錄 `currentRecord.id`。

### 13.3 PDF URL 非完整 URL

目前前端採 `VUE_APP_S3_HOST + "/" + previewedPdf` 的方式組完整 URL，代表後端回傳的可能是相對路徑而不是完整網址。

### 13.4 國碼寫死

重新發送 challenge 時，`mobileCountryCode` 目前固定寫死為 `886`。若新 prototype 需要國際化，必須改造。

### 13.5 錯誤處理偏簡單

目前大量使用 `alert()`，較適合內部原型，不適合正式產品。

## 14. 建議搬移檔案清單

### 核心必搬

- `src/components/ContractForm.vue`
- `src/components/PreviewContract.vue`
- `src/store/modules/contracts/actions.js`
- `src/store/modules/contracts/mutations.js`
- `src/store/modules/contracts/index.js`
- `src/store/modules/records/actions.js`
- `src/store/modules/records/mutations.js`
- `src/store/modules/records/index.js`
- `src/mixins/contract.js`
- `src/mixins/record.js`

### 視需求搬移

- `src/components/EventView.vue`
- `src/pages/EventList.vue`
- `src/pages/ContractList.vue`
- `src/router/routes/contract.js`
- `src/pages/ContractMenu.vue`
- `src/components/NavBar.vue`
- `src/components/ProgressBar.vue`

### 可不搬或重做

- 背景圖片資產
- PAMO 品牌文案
- 舊版主選單結構
- `withdraw-doc.pdf`

## 15. Prototype 建議實作順序

1. 先建立 `settlementService` 串接 API
2. 先做 `SettlementForm` 的 Step 0 與 Step 1
3. 接上 PDF preview
4. 完成 verify 後的成功頁
5. 再補 pending / signed 列表
6. 最後才處理舊 UI 視覺與裝飾

## 16. 結論

這個「和解書」功能的真正核心，是一個以事故紀錄為基底的雙方確認流程，而不是單純的一張表單。

若要快速移植到另一個產品原型，建議保留以下不變：

- 商業流程順序
- `create -> challenge -> verify` 三段式 API
- `record` 作為上游資料來源
- `previewedPdf` / `signedPdf` 的文件概念

若要縮小導入成本，最適合先做成：

- 一個可接受事故摘要輸入的 `SettlementForm`
- 一個可預覽 PDF 的 modal
- 一個完成後能開啟正式 PDF 的列表頁

如此可以在不完整搬運舊產品 UI 的前提下，保留最有價值的功能骨架。
