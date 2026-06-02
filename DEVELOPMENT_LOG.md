# 宇平區、世嘉區2026區旅遊相簿 開發日誌

## 2026-06-02

### 架構調整：全部改回 Google Drive

- 使用者決定放棄 Cloudinary 分流，照片與影片全部上傳到 Google Drive。
- 前端檔案：`index.html`
  - 照片：透過 GAS 上傳到 Google Drive。
  - 影片：透過 GAS 上傳到 Google Drive。
  - 相簿：透過 GAS `list` action 讀 Google Drive 目標資料夾內的媒體檔。
  - 移除 Cloudinary unsigned upload、manifest、本機快取與同步流程。
  - 保留上傳進度 UI：
    - 整體進度條顯示全部檔案完成比例。
    - 目前檔案進度條顯示單檔處理階段。
    - 照片與影片都顯示 FileReader 讀取進度，送出 GAS 階段顯示 40% 到 95% 的持續進度動畫。
    - 預覽卡片會標示 `Google Drive 完成` 或 `上傳失敗`。
  - 照片上傳前會由前端壓縮產生一張 JPG 縮圖，一起送到 GAS。
  - 影片上傳前會由前端擷取一張 JPG 縮圖，一起送到 GAS。
  - GAS 後端：`gas-drive-upload/Code.gs`
  - `upload` action 改為接受 `image/*` 與 `video/*`。
  - `list` action 支援 `offset` / `limit` 分頁，前端第一次只讀 50 筆，降低手機端等待時間。
  - 照片與影片縮圖會放在目標 Drive 資料夾底下的 `_album_thumbnails` 子資料夾。
  - 縮圖檔名為 `thumb_<原始檔案ID>.jpg`，相簿會優先拿它們當封面。
  - 讀取相簿時會一次建立縮圖索引，避免每個檔案都重新查一次縮圖資料夾。
  - 移除 `recordPhoto`、`listPhotos`、`syncCloudinaryPhotos` 與 Cloudinary API 設定。

### 目前需要設定

- GAS Web App URL 維持：
  - `https://script.google.com/macros/s/AKfycbwhTDrFxWSqw3mLIzSK34xBfU-rb997U2krPxFHcZ_Cs5-hijQ8DTJiJ6If_dqMcSq7/exec`
- Google Drive 目標資料夾 ID 維持：
  - `1VJsck6LCc2ub4QkGy-qLUWLpXbRgdm_g`

### 驗證待辦

- 重新部署 GAS，讓照片也能透過 `upload` action 進 Google Drive。
- 用小照片測試 Google Drive 上傳與相簿刷新。
- 用小影片測試 Google Drive 上傳、縮圖顯示與點擊開 Drive 預覽。

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
- GAS 回傳 Google Drive 縮圖網址，不直接把縮圖轉成 base64 塞進 JSON，避免手機端讀取逾時。
- 新上傳的照片與影片會額外儲存一張前端產生的 JPG 縮圖到 `_album_thumbnails` 子資料夾；相簿會優先使用這張縮圖。
- 圖片：
  - 顯示縮圖。
  - 點擊後用燈箱預覽縮圖。
- 影片:
  - 顯示縮圖與 `影片` 標籤。
  - 點擊後開啟 Google Drive 預覽頁。
- 前端每次向 GAS 讀取 50 個檔案，有更多時顯示 `載入更多` 再抓下一批。
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
