# 雪梨行程地圖網頁 — Claude Code 交接文件

> 把這份檔案（連同 `sydney_trip_maps.html`）放進專案資料夾，然後對 Claude Code 說：
> 「請先讀 `CLAUDE_CODE_HANDOFF.md` 了解專案，我接下來要請你幫我修改行程與部署。」

---

## 1. 這個專案是什麼

一個**單一檔案、純靜態的 HTML 網頁**（`sydney_trip_maps.html`），呈現一趟雪梨自由行的**每日行程 + 每日一張互動地圖**。地圖上會用編號標出當天各站點、用不同顏色的線畫出站與站之間的移動軌跡，顏色代表交通方式。每段路線都有一個「導航」按鈕會開啟 Google Maps。

- 前端技術：原生 HTML/CSS/JS + **Leaflet 1.9.4**（地圖引擎，從 cdnjs 載入）
- 地圖圖磚：主要用 **CARTO Voyager**（彩色），失敗時自動 fallback 到 **OpenStreetMap**
- 路線幾何：步行／公車／計程車會呼叫 **Valhalla 公開 API**（`valhalla1.openstreetmap.de`，免金鑰）
  取得沿真實道路的路徑，結果快取在 localStorage；火車、輕軌、渡輪、飛機沒有免費公開路網
  API，仍以 `via` 轉乘點連線示意。抓不到就自動退回直線，不會壞掉。
- 沒有任何 build 步驟、沒有框架、沒有 npm 相依。用瀏覽器直接開 `index.html` 就能跑。
- 語言：**繁體中文／日文雙語**，右上角按鈕即時切換，選擇存在 localStorage（`sydLang`）。
  所有使用者看得到的文字（標題、行程、交通標籤、地圖彈出視窗、頁尾）都要雙語，見第 3 節。

---

## 2. 旅程基本設定（修改時的背景知識）

- **旅客**：2 位成人，預算取向，偏好簡單（單一住宿基地、不換旅館）。
- **住宿（每日起點與終點）**：5 Confectioners Way, Rosebery NSW 2018。最近車站是 **Green Square（T8 機場線）**，步行約 12–15 分鐘。程式裡以常數 `HOME` 代表其座標。
- **航班**：
  - 去程 8/22 台北 TPE → 廈門 XMN 轉機（需台胞證，安排了半日遊）→ 8/23 早上抵雪梨 SYD。
  - 回程 8/30 早上 MF802 離開雪梨 → 廈門過夜 → 8/31 MF887 回桃園。
- **硬限制**：8/30（週日）是**雪梨馬拉松**，市中心大範圍封路，但火車不受影響——這是選 Rosebery 的原因，離境走 T8。
- **四大必去**：雪梨歌劇院、港灣大橋、藍山、塔龍加動物園（已全部排入）。
- **排程原則**：
  - 週末限定的市集排在週末（The Rocks Market 週六日、Bondi 農夫市集週六）。
  - 購物集中在**週四**（雪梨只有週四商店開到 21:00）。
  - 藍山排平日避開人潮。
  - 站點之間盡量順路，交通方式明確標示。
  - 冬末天氣 8–19°C（藍山 0–12°C），日落約 17:25，所以每天早出、傍晚收。

---

## 3. 檔案結構與資料模型（**修改行程主要就是改這裡**）

所有行程資料都在 `<script>` 裡的一個陣列 **`DAYS`**。每一天是一個物件：

```js
{
  id:    'd4',                       // 唯一 id，對應地圖 DOM 容器
  iso:   '2026-08-26',               // ISO 日期，tab/date 文字（含星期）由此自動算出，兩種語言都是
  title: {zh:'塔龍加動物園・Balmoral 海灘', ja:'タロンガ動物園・バルモラル・ビーチ'},
  chips: [{zh:'F2 渡輪＝迷你港灣遊船', ja:'F2フェリー＝ミニ湾内クルーズ気分'}, ...],
  stops: [ ...站點 ... ],            // 見下
  legs:  [ ...路段 ... ],            // 見下
  note:  {zh:'底部黃色 NOTE 區的文字', ja:'...'}   // 可省略（整個 note 欄位不寫即可）
}
```

**雙語欄位一律寫成 `{zh:'...', ja:'...'}` 物件**，用 `t(field)` 這個 helper 依目前語言取值。
不需要雙語的欄位（時間 `t`、座標 `ll`、Google Maps 搜尋字串 `q`）維持純字串。

### 站點 `stops[]`
每個站點：
```js
{
  t:   '09:30',                       // 時間（不分語言）
  n:   {zh:'塔龍加動物園', ja:'タロンガ動物園'},        // 名稱
  d:   {zh:'接駁上山由上往下逛…', ja:'...'},           // 說明（可空字串 ''）
  dur: {zh:'約 4.5 小時', ja:'約4時間30分'},           // 預計在這站停留多久，時間軸上時間旁的小標籤
                                                      // 純過渡點（如「住處出發」）沒有停留時間就寫 null
  ll:  [-33.8433, 151.2411],          // [緯度, 經度] → 決定地圖上編號位置（不分語言）
  q:   'Taronga Zoo Sydney'           // Google Maps 導航用的搜尋字串（不分語言，通常留英文/當地拼音）
}
```

### 路段 `legs[]`
**規則：`legs.length` 一定等於 `stops.length - 1`**（每兩個相鄰站點之間一段）。
```js
{
  m:   'ferry',                      // 交通方式，必須是 MODES 的 key（見下）
  l:   {zh:'F2 渡輪 12 分', ja:'F2フェリーで12分'},   // 顯示文字
  via: [[-33.86,151.21], ...]        // 選填：中途轉乘點座標，讓地圖線條經過這些點
}
```
`via` 是選填的。有填的話，地圖上那段線會依序經過這些中繼點（例如火車經過 Central 轉乘站）。不填就是起點直接連終點的直線。

### 交通方式選擇原則
**能搭大眾運輸就不要用計程車**，除非價錢跟大眾運輸差不多（此時計程車通常更省時間，值得保留，
並在 `l` 文字裡註明理由，例如 d1 機場→住處那段）。目前全站只保留這一段計程車；其餘原本用
計程車的路段已改成公車（如 d4 的 Taronga↔Balmoral 用 238 公車、d0 廈門三段改用 BRT機場專線／
2·29路公車／空港快線輪渡專線）。之後若新增路段，優先查有沒有公車/火車/渡輪可用。

### 交通方式 `MODES`
可用的 key 與對應顏色：
| key | 顯示 | 顏色 | Google Maps 模式 |
|-----|------|------|------------------|
| `walk` | 步行 | 灰虛線 | walking |
| `train` | 火車 | 橘 | transit |
| `bus` | 公車 | 藍 | transit |
| `lightrail` | 輕軌 | 紅 | transit |
| `ferry` | 渡輪 | 藍綠點線 | transit |
| `taxi` | 計程車 | 紫 | driving |
| `flight` | 飛機 | 深灰點線 | （無導航連結）|

### 常用座標常數
檔案上方定義了轉乘節點常數供 `via` 重複使用：
`GS`(Green Square)、`CEN`(Central)、`CQ`(Circular Quay)、`WYN`(Wynyard)、`TH`(Town Hall)、`STR`(Strathfield)、`BJ`(Bondi Junction)、`HOME`(住處)。

另有 **343 公車路廊**常數：`BUS343_IN`（往市區）與 `BUS343_OUT`（回住處）。
343 從住處門口的 Rothschild Ave 上車，沿 Joynton Ave → Elizabeth St → Central → Martin Place
→ Circular Quay，**全程不用轉車**，是往歌劇院／環形碼頭最方便的方式（比走 12–15 分到
Green Square 轉火車省事）。住處往返市區的路段請優先用這組常數。

---

## 4. 常見修改怎麼做（給 Claude Code 的操作指引）

- **改某天某站的時間／名稱／說明**：找到對應 `DAYS` 裡那天的 `stops`，改 `t` 或 `n`/`d`/`dur` 的
  `zh` 和 `ja` 兩個值（**兩個語言都要改**，不要漏掉日文）。
- **新增一個站點**：在該天 `stops` 插入一個站點物件（`n`/`d`/`dur` 都要有 zh/ja），**並且**在
  `legs` 對應位置補一段（維持 `legs.length === stops.length - 1`），否則地圖線會對不上。記得給
  新站點正確的 `ll` 座標和 `q` 搜尋字串。
- **刪除一個站點**：刪 `stops` 那筆，**同時**刪掉相鄰的一段 `legs`。
- **改交通方式**：改該段 `legs` 的 `m`（要用 MODES 既有的 key），優先考慮大眾運輸（見上一節原則）。
- **調整某天預設先顯示**：檔案最底 `showDay(1)`，數字是 `DAYS` 的索引（0 起算，目前 1 = 8/23）。
- **改顏色主題**：CSS `:root` 裡的變數（`--train`、`--bus`… 等）。
- **改介面固定文字**（標題、圖例、按鈕、頁尾等非行程資料）：改 `UI` 物件裡的 `zh`/`ja`。
- **座標怎麼查**：Google Maps 上對地點按右鍵，第一列就是「緯度, 經度」，直接填進 `ll`。
- ⚠️ **不要**破壞 `stops` 與 `legs` 的數量對應關係，這是最容易出錯的地方。
- ⚠️ **雙語欄位不要漏填**：`title`/`chips`/`n`/`d`/`dur`/`l`/`note` 都要同時有 `zh` 和 `ja`，
  否則該語言會顯示空白或退回中文。
- 改完可用 `node --check`（把 `<script>` 內容抽出）或直接開瀏覽器 Console 看有無錯誤，並且
  務必點右上角語言切換按鈕，兩種語言都各看一遍。

---

## 5. 地圖顯示問題（已知狀況，勿誤判為 bug）

站點編號和路線是瀏覽器本地繪製，**一定會顯示**；底圖圖磚是外部圖片，**在受限環境（如某些預覽沙盒、擋 CDN 的 VPN／外掛）會被擋成白底或灰底**。這不是程式錯誤。

- 已加入 fallback：CARTO 圖磚失敗會自動換 OSM。
- **在真正的網站上（GitHub Pages 等）或本機直接開檔案時，底圖會正常顯示。**
- 若部署後仍全白，才需檢查網路／CSP；否則不用改程式。

---

## 6. 部署（GitHub Pages，可持續編輯）

使用者要的是**能反覆編輯**的部署方式，用 GitHub Pages：

1. 把 `sydney_trip_maps.html` 重新命名為 **`index.html`**，放進 repo 根目錄。
2. Push 到 GitHub（Public repo，例如 `sydney-trip`）。
3. repo Settings → Pages → Source 選 `main` 分支、`/ (root)`，Save。
4. 網址：`https://<帳號>.github.io/sydney-trip/`。

**若使用 Claude Code 協助**，理想工作流程是：
- 幫使用者把此專案初始化為 git repo、建立 `.gitignore`（此專案其實不需要，純靜態）。
- 每次改完 `index.html` 後，執行 `git add . && git commit -m "..." && git push`，GitHub Pages 約 1 分鐘自動更新。
- 使用者用中文描述要改的行程，Claude Code 直接改 `index.html` 的 `DAYS` 資料再 push。

> 注意：部署相關的實際 `git push`、建立遠端 repo 等動作，請依 Claude Code 當下的權限規則進行，需要時向使用者確認。

---

## 6.5 共用備註功能（多人同步）

每個站點下方有一塊**永遠顯示、不會收合**的文字框，任何看這個網頁的人都可以直接打字寫備註（不記名，沒有作者欄位），離開輸入框就自動存檔，**設定好 Firebase 後會即時同步**給所有人看到最新內容。故意做得很陽春（自由輸入的文字框，沒有留言卡片、沒有展開/收合），因為使用者明確表示不需要這些。

- 後端用 **Firebase Firestore**（免費、免伺服器，跟 GitHub Pages 這種純靜態網站相容）。
- 程式碼裡的 `firebaseConfig`（在 `<script>` 前段）目前已填入使用者的真實專案設定（`sydney-22301`）。
- Firestore → Rules **必須同時開放 `notes` 和 `edits` 這兩個 collection**（`edits` 是 8.5 節的網頁內建編輯模式用的），範圍都限縮在各自的 collection：
  ```
  rules_version = '2';
  service cloud.firestore {
    match /databases/{database}/documents {
      match /notes/{noteId} { allow read, write: if true; }
      match /edits/{editId} { allow read, write: if true; }
    }
  }
  ```
  ⚠️ 如果只開了 `notes` 沒開 `edits`，備註功能正常但編輯模式的存檔會在背景silently失敗（`saveEdit` 的 `.catch(()=>{})` 會吞掉錯誤，UI 不會報錯，但重新整理後改的東西會消失）。修改 Rules 後記得按 **Publish**。
- 如果 `firebaseConfig` 還是佔位值（`PASTE_YOUR_...`），兩個功能都會自動降級成「只存在瀏覽這台裝置的 localStorage」，不會報錯、UI 一樣能用。
- 這些 `firebaseConfig` 值是**公開識別碼不是密碼**，可以放心留在公開的 GitHub repo 裡；真正的存取控制在 Firestore Rules。因為沒有登入機制，**任何拿到網址的人理論上都能改備註／改行程**，風險等同於一份公開可編輯的雲端文件，適合這種私人小型旅遊網站，但不要拿來存敏感資訊。
- 資料結構：一份文件對應一個站點，doc id 是 `${dayId}_${stopIndex}`（例如 `d4_3`），內容 `{text, updatedAt}`（沒有 author）。

---

## 6.6 網頁內建編輯模式（不用改原始碼也能調整行程）

header 右上角「編輯行程」按鈕：平常是唯讀畫面，按下去後每一站的**時間／名稱／說明／停留時間**、以及每一段路程的**說明文字**都變成可以直接打字修改的欄位，欄位離開焦點（blur）就自動存檔。跟備註共用同一個 Firestore 專案，存在另一個 collection `edits` 裡，所以**誰改的不重要，任何人打開網頁都看得到最新版本**。

- 目前**可編輯**：`stops[].t`／`n`／`d`／`dur`、`legs[].l`。
- 目前**不能**透過編輯模式改：交通方式（`m`，會牽動顏色與路線重新查詢）、座標（`ll`）、Google Maps 搜尋字串（`q`）、當日標題／chips／NOTE。這些還是要請 Claude Code 改原始碼。
- 資料模型：`editsCache` 裡每筆是 `{dayId}_{kind}{index}_{field}[_{lang}]` → `{value, updatedAt}`。`kind` 是 `stop` 或 `leg`；有雙語欄位（`n`/`d`/`dur`/`l`）會依當下語言各自存一份（`_zh`／`_ja`），時間 `t` 不分語言。
- 顯示時一律 `resolveField(key, 原始值)`：有人編輯過就用編輯後的值，沒有就退回 `index.html` 裡寫死的原始值。**這代表原始碼裡的 `DAYS` 陣列永遠是「出廠設定」，網頁上的即時編輯是疊加在上面的一層 override，不會真的動到原始碼**——如果要讓改動變成新的預設值（例如使用者在網頁上調整過後，想讓這份行程變成永久版本），需要回來請 Claude Code 把 `editsCache` 目前的值謄回 `DAYS` 陣列裡。
- 沒有做欄位驗證（時間格式、必填等等），因為是雙人共用的臨時調整工具，故意做得簡單。
- 安全性：因為任何人都能寫、又會被塞進 `innerHTML` 顯示，所有可編輯欄位輸出前都會過 `escapeHtml()`，避免有人貼 HTML/script 造成 stored XSS——**之後如果要擴充可編輯欄位（例如開放編輯 chips 或 NOTE），一定要記得對新欄位也套用 `escapeHtml()`**，這是最容易漏掉、也最容易出安全問題的地方。

---

## 7. 使用者接下來可能想做的事（待辦候選）

- 微調每天的站點順序或時間節奏（使用者偏好反覆迭代）。
- 可能把「獵人谷 Hunter Valley 自駕」重新排入（目前擱置）。
- 補上各景點門票／營業時間連結，或加入預算數字卡片（舊版 `sydney_itinerary` 曾有每日預算與總花費）。
- 火車／輕軌目前仍是轉乘點示意連線；若要真實軌道路徑，需要另外抓 OSM 鐵道資料（Overpass），成本較高。

---

## 8. 一句話摘要給 Claude Code

> 這是一個純靜態的雪梨行程地圖網頁，所有行程資料集中在 `sydney_trip_maps.html`（部署時改名 `index.html`）的 `DAYS` 陣列。修改行程 = 改 `DAYS` 裡的 `stops` 與 `legs`（兩者數量差 1 必須維持）。部署走 GitHub Pages。地圖底圖在受限環境會變白屬正常現象，部署後即正常。
