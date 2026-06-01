# 宇平區、世嘉區2026區旅遊相簿 開發日誌

## 2026-06-02

### 架構調整：照片 Cloudinary、影片 Google Drive

- 使用者決定將照片改回 Cloudinary，影片檔一律放到 Google Drive。
- 相簿只顯示照片，不顯示影片。
- 前端檔案：`index.html`
  - 照片：上傳到 Cloudinary image upload endpoint。
  - 影片：透過 GAS 上傳到 Google Drive。
  - 相簿：透過 GAS 讀 Cloudinary 照片 manifest，只渲染照片。
- GAS 後端：`gas-drive-upload/Code.gs`
  - 保留 `upload` action 給影片上傳 Google Drive。
  - 新增 `recordPhoto` action，記錄 Cloudinary 照片 URL。
  - 新增 `listPhotos` action，回傳相簿照片清單。
  - 照片清單存在 Google Drive 目標資料夾內的 `_cloudinary_album_manifest.json`。

### 目前需要設定

- Cloudinary cloud name 已確認：
  - `dlknzcex3`
- Cloudinary unsigned upload preset 已填入 `index.html`：
  - `november_trip_album_unsigned`
  - Cloudinary preset 內的 asset folder：`november-trip`
- GAS Web App URL 維持：
  - `https://script.google.com/macros/s/AKfycbwhTDrFxWSqw3mLIzSK34xBfU-rb997U2krPxFHcZ_Cs5-hijQ8DTJiJ6If_dqMcSq7/exec`
- Google Drive 目標資料夾 ID 維持：
  - `1VJsck6LCc2ub4QkGy-qLUWLpXbRgdm_g`

### 驗證待辦

- Cloudinary unsigned upload preset 已確認，後續若修改 preset 記得同步測試前端上傳。
- 重新部署 GAS，讓 `recordPhoto` / `listPhotos` 生效。
- 用小照片測試 Cloudinary 上傳、manifest 記錄、相簿刷新。
- 用小影片測試只上傳 Google Drive，且不出現在相簿。

## 2026-06-01

### 專案目標

- 建立一個給旅遊活動使用的網頁，讓參加者方便上傳照片與影片。
- 重點是收集可用於剪輯影片的原始素材，不希望素材經 LINE 或其他通訊軟體壓縮。
- 使用者最好只要打開網頁就能上傳，不需要登入額外帳號。

### 架構演進

1. **原始方案：Google Apps Script + Google Drive**
   - 舊版網頁與 `Code.gs` 都部署在 GAS。
   - 上傳透過 `google.script.run.uploadFileToDrive()`。
   - 相簿透過 `getFilesFromFolder()` 讀 Google Drive 縮圖。
   - 優點是 Google Drive 管理簡單，縮圖顯示快。

2. **中間方案：GitHub Pages + Cloudinary**
   - 將網頁改成可放在 GitHub Pages 的純靜態 `index.html`。
   - 照片上傳到 Cloudinary，利用 Cloudinary 縮圖與 CDN 顯示相簿。
   - 發現影片受 Cloudinary 方案限制，免費/目前帳戶影片單檔約 100 MB，不適合收原始影片素材。

3. **雙上傳方案：Cloudinary + Google Drive**
   - 照片進 Cloudinary 顯示相簿，原始檔再備份到 Google Drive。
   - 影片只備份到 Google Drive。
   - 後來判斷上傳兩份會變慢，而且管理刪除也變複雜。

4. **目前方案：Google Drive Only**
   - 移除 Cloudinary。
   - 所有小於或等於 200 MB 的照片與影片只上傳到 Google Drive。
   - 相簿也從同一個 Google Drive 資料夾讀縮圖。
   - Drive 資料夾原本已有的照片/影片，也會出現在相簿。

### 目前版本行為

- 前端檔案：`index.html`
- GAS 後端：`gas-drive-upload/Code.gs`
- Google Drive 目標資料夾 ID：
  - `1VJsck6LCc2ub4QkGy-qLUWLpXbRgdm_g`
- GAS Web App URL：
  - `https://script.google.com/macros/s/AKfycbwhTDrFxWSqw3mLIzSK34xBfU-rb997U2krPxFHcZ_Cs5-hijQ8DTJiJ6If_dqMcSq7/exec`

#### 上傳規則

- 單檔上限：200 MB。
- 小於或等於 200 MB：
  - 照片上傳到 Google Drive。
  - 影片上傳到 Google Drive。
- 超過 200 MB：
  - 選檔時直接略過。
  - 顯示提示：請用 AirDrop 傳原始檔。

#### 相簿顯示

- 相簿來源：Google Drive 資料夾。
- GAS 使用 `file.getThumbnail()` 產生縮圖，再轉成 base64 data URL 回傳給前端。
- 圖片：
  - 顯示縮圖。
  - 點擊後用燈箱預覽縮圖。
- 影片:
  - 顯示縮圖與 `影片` 標籤。
  - 點擊後開啟 Google Drive 預覽頁。
- 前端每次顯示 50 個檔案，有更多時顯示 `載入更多`。
- GAS 目前最多回傳最新 300 個媒體檔案，避免 Apps Script 太慢或超時。

### 重要決策

- 不再使用 Cloudinary 作為相簿或上傳目的地。
- 原始素材以 Google Drive 資料夾內檔案為準。
- 相簿預覽顯示的是 Google Drive 縮圖，不是原始解析度。
- 原始素材下載、刪除與管理都在 Google Drive 裡處理。
- 200 MB 以上的影片不透過網頁上傳，改用 AirDrop。

### 部署步驟

#### 更新 GitHub Pages

1. 到 GitHub repository：`november-trip-album`。
2. 上傳新版 `index.html` 覆蓋原本檔案。
3. Commit changes。
4. 等 GitHub Pages 更新。

#### 更新 Apps Script

1. 開啟 Apps Script 專案。
2. 將 `gas-drive-upload/Code.gs` 內容貼到 GAS 的 `Code.gs`。
3. 點 `Deploy` -> `Manage deployments`。
4. 編輯原本 Web App deployment。
5. 選 `New version`。
6. 按 `Deploy`。
7. 確認 Web App URL 結尾是 `/exec`。

### 驗證紀錄

- `index.html` JavaScript 語法檢查通過。
- `Code.gs` 基本語法檢查通過。
- 本機 `index.html` 可正常回應 HTTP 200。
- 已確認本機檔案中沒有 Cloudinary 相關程式碼殘留。

### 待辦與注意事項

- 重新部署 GAS 後，測試相簿是否能讀到 Google Drive 既有照片。
- 用小照片測試上傳與相簿刷新。
- 用小影片測試上傳、縮圖顯示與點擊開 Drive 預覽。
- 如果相簿載入變慢，可以降低 `Code.gs` 的 `LIST_LIMIT`，或之後改成分頁 API。
- 如果未來需要顯示原始大圖，需另外設計 Drive 原圖/下載連結，不建議直接把原圖 base64 回傳前端。
