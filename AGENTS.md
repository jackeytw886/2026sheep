# AI 交接說明：2026sheep（雲羊逐月 × 三羊共福 3D 模型＋奔跑逐月）

> 給接手的 AI（Claude、ChatGPT／Codex、Gemini、Copilot、Cursor…）看的交接檔。
> 先讀完這份再動手；改完程式請同步更新本檔的「目前狀態」與 README 的版本紀錄。
> 最後更新：2026-10-17（v1.1，新增「奔跑逐月」）

## 1. 這是什麼

2026 羊年燈組「雲羊逐月 × 三羊共福」的 3D 燈光秀模型（給燈會提案、施工溝通與展示用）。兩座燈：

- **雲羊逐月**（piece `yun`）：山羊踏雲追明月；明月、雲羊、羊角、雲氣光帶、光階梯（8 階）、雲海、花朵、星點
- **三羊共福**（piece `san`）：新月與燈籠、櫻花樹、大羊、兩隻小羊、瀑布溪水、祥雲、花朵

頁面功能：180 秒時間軸燈光秀、2D／3D／並排、6 種視角、夜間／白模、3 個「羊角祈福」語音互動站點（ASRPRO 吉祥話）、燈具監看卡，以及 v1.1 新增的攝影機互動「**奔跑逐月**」。

- 線上網址（GitHub Pages）：https://jackeytw886.github.io/2026sheep/
- 相關 repo：`2026sheepgate` 是另一個專案（祥雲迎賓門 燈光控制模擬），和這裡無關，不要混在一起改

## 2. 檔案結構（重要：`index.html` 約 4.6 MB）

| 檔案 | 內容 |
|---|---|
| `index.html` | 全部內容的單一檔案（約 2100 行，但有幾行超長） |
| `README.md` | 給人看的說明：玩法、版本紀錄 |
| `AGENTS.md`／`CLAUDE.md` | 本交接檔／Claude Code 入口 |

`index.html` 裡有大型內嵌資料，**不要整檔印出或整檔丟進上下文**：

| 位置（約略行號） | 內容 | 大小 |
|---|---|---|
| `<script id="meta" type="application/json">` | 燈區頻道 `ch`、群組、燈具座標 `fix`、深度圖等 | ~114 KB（單行） |
| `<script id="glb" type="text/plain">` | 3D 模型 GLB（base64） | ~2.1 MB（單行） |
| `<script id="sim2d" type="text/plain">` | 原本的 2D 燈光模擬台整個 HTML（base64，在 iframe 裡跑） | ~2.2 MB（單行） |
| module 內 `const ZD={...}` | 2D 燈區多邊形與尺寸 | ~26 KB（單行） |

建議編輯流程：

1. 用 `grep -n` 找位置，`Read` 只讀需要的行段（跳過上面幾行超長資料）
2. 用唯一的字串錨點做替換（例如 Python `str.replace` 並先確認 `count == 1`）
3. 把 `<script type="module">…</script>` 內容抽出成 `.mjs`，跑 `node --check` 檢查語法
4. base64 區塊（glb、sim2d）裡的內容不要用文字搜尋去改

## 3. 燈光秀引擎（原有程式）

module 用 importmap 載入 `three@0.160.0`（jsdelivr）：GLTFLoader、OrbitControls、CSS2DRenderer、EffectComposer＋UnrealBloomPass。

| 名稱 | 作用 |
|---|---|
| `PIECES.yun`／`PIECES.san` | 各座燈的時間軸與燈區邏輯：`sig(l, G)` 依本地時間 `l` 產生訊號物件 `S`（例：`S.moon`、`S.ram`、`S.stair(o)`、`S.lambL`、`S.reveal`、`S.mov`）；`zoneI(z, S)` 把訊號轉成每個燈區亮度 |
| `VER` | 兩座燈「合演」設定：`LOOP = 180`、`tmap.yun/san`（全域時間 → 各座本地時間）、`SEGS`（12 段）、`floorSig()`（地面光帶）、`stations`（站點） |
| `STLIB` | 三個羊角祈福站點 s1 月光台／s2 三陽台／s3 團圓台（光效模式 st1／st2／st3） |
| `frameAt(t)` | 每幀算出 `FR.yun`、`FR.san`，再疊上站點光效 `showMod` |
| `updLantern()` | 把 `FR` 寫進每個燈區的 shader uniform（亮度、由下往上 `uRev`、螺旋 `uCurl`…）；也處理地面光帶與雲羊／小羊的機構擺動 `MOVS` |
| `upd*()` | 帕燈、搖頭燈、雷射、頻閃、追光、星空、天空燈、洗牆、光束燈條、冷焰火 `sparkA()`／`updSpark()`、低煙、投影 |
| `SYN.*`／`playCue()` | 合成音效（bell、mallet、sparkle、bleat、pluck、chime、drum…）；`SND.on` 控制開關 |
| `loop(now)` | 主迴圈：推進 `t` → `frameAt` → 各 `upd*` → 音效 → 運鏡 → 時間軸 UI → render |
| `resize()` | 版面；燈光監看卡（或奔跑面板）開著時用 `camera.setViewOffset` 讓出空間 |
| `window.__api` | 原有測試掛勾：`setT`、`play`、`mode`、`view`、`station`、`state()` |

時間點對照（全域 `t`）：0–12 月色初現 … 54–70 逐月前行（光階梯逐階亮、約 66.8 秒抵達明月）… 148–164 圓滿祝福（冷焰火、全場齊亮）… 164–180 萬家團圓。

## 4. 奔跑逐月（v1.1 新增）

在攝影機前**原地跑步**，跑越快燈越亮，先「逐到明月」者勝。程式集中在 module 後段 `/* 奔跑逐月：攝影機原地跑步互動（1～3 人） */` 區塊；另有 CSS（`/* ===== 奔跑逐月 ===== */`）與 HTML（`#runBtn`、`#runHud`、`#runPanel`、`#runCount`、`#runRes`）。

### 玩法與燈光對應

| 人數 | 玩家 → 燈 | 燈組反應 |
|---|---|---|
| 1 | 雲羊 | 光階梯隨進度一階階亮（每階一個音）、雲羊由下往上亮、羊角捲開、眼睛亮；抵達＝明月爆亮＋雲羊側冷焰火 |
| 2 | 小羊（左）、小羊（右） | 各自的小羊越跑越亮、每步跳一下；光階梯跟著領先者 |
| 3 | 小羊（左）、大羊、小羊（右） | 同上；大羊由下往上亮、羊角捲開、身後暖光 |

共通：雲氣光帶、瀑布、祥雲與地面光帶的流速跟著大家的平均步頻。勝負揭曉 3 秒後自動跳到 `t = 148` 播「圓滿祝福」，並顯示名次卡。1 人模式把最佳秒數存在 `localStorage`（鍵 `sheepRunBest` + 步數）。

### 流程狀態 `RUN.state`

`idle` → 開面板 `ready`（燈光秀停在 `RUN_BASE_T = 12`）→ 按開始或舉雙手 1 秒 `count`（3、2、1）→ `race` → 有人抵達 `win`（冷焰火、明月閃）→ `finale`（恢復燈光秀並顯示結果）→「再跑一次」回到 `ready`。兩分鐘沒人抵達會自動結束。

### 和原引擎的掛接點（改原程式時要留意）

1. `loop()`：`t` 推進加了 `&& !RUN.holdT`；`frameAt(t)` 前呼叫 `runDetect(now); runTick(dt);`，之後呼叫 `runMod()`
2. `runMod()`：在 `frameAt` 之後**覆寫** `FR.yun`／`FR.san` 的訊號（只在 `RUN.lights` 為真時）
3. `VER.floorSig` 被包一層：比賽中改由步頻決定地面光帶亮度與流速
4. `sparkA(ci)` 開頭加了提前回傳：`RUN.sparkUntil` 內噴冷焰火（1 人只噴雲羊側 `ci < 3`）
5. `resize()`：`#runPanel` 開著時和燈光監看卡一樣讓出畫面（桌機往左、手機往上）
6. 比賽中隱藏 3D 標籤與 `.stat`（`.app.racing`）；開面板會自動收起燈光監看卡，關閉時還原時間與運鏡

### 步伐偵測（`runPoses()`）

- MediaPipe `@mediapipe/tasks-vision@0.10.21`、`pose_landmarker_lite`，`numPoses: 3`，VIDEO 模式，攝影機要求 960×540（寬畫面方便多人）
- 多人時把鏡像畫面分成 N 格，左到右＝玩家 1～N；每格取軀幹最大（最近）的人
- 長度單位 `T` = 肩膀中心到髖部中心（髖部看不到時用肩寬 × 1.5）
- 訊號優先序：雙膝高度差 `(y26 − y25)/T`（門檻 0.10）→ 雙腳踝（0.12）→ 雙手腕擺動（0.22）；門檻乘上靈敏度倍率 `RUN_SENS`（高 0.75／中 1／低 1.3）
- 訊號平滑（`sig = sig*.4 + s*.6`），越過正負門檻換邊＝一步，兩步最短間隔 0.11 秒
- 步頻＝最近 2 秒步數 ÷ 2；`act` = 步頻 / 3.5（0～1，平滑）驅動流速與擺動
- 舉雙手（雙手腕高於鼻子 0.3T）1 秒＝開始
- 備援：鍵盤玩家1 `A/S`、玩家2 `K/L`、玩家3 `←/→`（同鍵連按不算）；點跑道也算一步

### 測試掛勾

```js
window.__run.state()        // 狀態、人數、各玩家步數／進度／是否入鏡／偵測來源、t
window.__run.sim(poses, ts) // 餵一幀假骨架陣列（每人 33 個 landmark），ts 為秒
window.__run.step(i)        // 直接讓玩家 i 踏一步
```

先前用 Playwright＋Chromium（swiftshader）在桌機 1366×800、手機 390×800 驗證過：1 人完賽與成績、3 人模擬骨架（不同步頻得出正確名次）、假攝影機流程、關閉後燈光秀恢復。環境連不到 CDN 時，用 npm 裝 `three@0.160.0`、`@mediapipe/tasks-vision@0.10.21`，再以 `page.route()` 導到本機；Google Fonts 可直接擋掉。模型載入要 1～2 分鐘（swiftshader 很慢），等 `#load` 被隱藏再操作。

## 5. 目前狀態與待辦

- ✅ v1.1：奔跑逐月已推上 `main`（commit `701fe64`）
- ⚠️ **攝影機對真人的步伐偵測還沒實測**（測試環境下載不到姿勢模型，只用模擬骨架驗證）。若作者回報不準：先看是否膝蓋入鏡，再調膝蓋門檻 0.10 與 `RUN_SENS`
- ⚠️ GitHub Pages 網址是照同帳號其他專案的格式寫的，尚未實際打開確認
- 💡 可能的下一步：實體版用 WebSerial／Arduino 把比賽狀態送到真的燈具（作者的燈組控制用 DMX512／Arduino Mega）

## 6. 和作者合作的慣例

- 對話、介面文字、程式註解都用**繁體中文**（例外：天問 ASRPRO 程式碼裡的中文字串用簡體，例如站點命令詞 `asr:['云羊逐月',…]`）
- 偏好**單檔 HTML**；給檔案時直接給原始檔，**不要打包成壓縮檔**
- 偏好小步快跑：先做出能用的版本，再依回饋微調
- 正式文件（規格、接線、投標服務建議書）走「文化溫潤」風格：墨黑＋朱紅、傳統質感；這個頁面的配色變數在 `:root`（`--zhu` 朱紅、`--jade` 石綠、`--amber` 琥珀）
- 所有專案集中放在 GitHub 管理（作者稱「ghub」）；改完要 commit＋push，commit 訊息用繁體中文

---

## Shared Management Rules — 共用管理規則

- 本倉庫固定代號是 P005；先讀 README.md、HANDOFF.md、PROJECT.json 與本次任務相關原始檔。
- 對話、交接與一般文件使用繁體中文；既有專案規格優先保留。天問 ASRPRO 程式內中文字依其專案規範使用簡體中文。
- 開始時確認實際倉庫、分支、commit SHA 與工作目錄變更；分辨文件記載、已實作與已執行驗證。
- 依當次使用者任務與工具環境規則工作；在授權範圍內完成修改、提交、推送或發布，保留他人尚未提交的變更。
- 需要並行修改時使用獨立分支與工作目錄；以任務或 PR 記錄範圍，不假設交接文件具有自動鎖。
- 有修改時更新 HANDOFF.md，記錄版本依據、修改路徑、驗證結果、限制與下一步；只閱讀時在回答中提供摘要即可。
- 長期決策與現場流程分別保存於 docs/DECISIONS.md、docs/RUNBOOK.md；文件只建立已知規格，未知欄位明確標註。
- 圖片、模型與大型內嵌資料先參考 docs/FILE-MAP.md；僅讀取本次需要的段落，避免把大型二進位或 base64 整份載入上下文。
- 密碼與金鑰以環境或平台秘密管理，範例使用虛構個人資料；保留現有倉庫與資料的存取界線。
- GitHub Pages 已啟用的倉庫，改名或搬動入口前先核對網站、QR code、kiosk 與引用；本次管理整理保留現有部署入口。
