# Lei-k.github.io

## 這是什麼

GitHub Pages 帳號頁（`lei-k.github.io`）的靜態內容集合，目前包含兩類彼此獨立的東西：

- **一個歷史 Octopress 靜態輸出**（`index.html`、`blog/`、`atom.xml`、`stylesheets/`、`javascripts/` 等）。`git log` 顯示最早兩個 commit 是 `Octopress init` 與一次 2015 年的自動發布；頁面標題仍是預留文字「My Octopress Blog」、作者欄位是「Your Name」，[`blog/archives/`](blog/archives/index.html) 底下沒有任何文章。這是建置後留下的預設 shell，**不是持續維護的部落格或作品集**。
- **[`secure-key-drop/`](secure-key-drop/index.html)**：一個獨立的單頁瀏覽器端金鑰加密工具，細節見下。

## 適合誰

維護這個 repo、或需要理解 `secure-key-drop` 加密行為並自行判斷是否可信任的人。不是給讀者訂閱文章用的。

## 能得到什麼

- 所有內容都是純靜態檔案，可在本機直接預覽（見「最短路徑」）。
- `secure-key-drop/index.html` 的加密邏輯可直接讀原始碼確認，見下方摘要。

## 成熟度與限制

- Repo 裡沒有 `Gemfile`、`Rakefile`、`_config.yml` 或 Octopress 的 `source/` 目錄，只有建置後的靜態輸出，**本 repo 不具備可重現的 Octopress 原始碼/建置/部署流程**。要更新部落格內容，需要另外重建 Octopress 環境。
- `secure-key-drop` 未經第三方安全稽核。
- 以下對 `secure-key-drop` 的描述僅涵蓋這個 repo 能證明的部分：瀏覽器端的頁面行為。**接收端如何解密、儲存、輪替金鑰不在本 repo 內**，本文件不對整條 Telegram 收發流程的安全性做保證。

## 最短路徑：本機預覽

在 repo 根目錄執行（只需系統自帶的 Python 3，不需安裝任何依賴）：

```bash
python3 -m http.server 8000
```

瀏覽 `http://localhost:8000/`（歷史 Octopress shell）或 `http://localhost:8000/secure-key-drop/`。

## `secure-key-drop` 的瀏覽器端行為（可由 [`secure-key-drop/index.html`](secure-key-drop/index.html) 原始碼證明）

- **嚴格 CSP**：`default-src 'none'`，只允許以 sha256 pin 住的內嵌 `<script>`/`<style>`；`connect-src 'none'`、`frame-src 'none'` 等其餘來源全部關閉，頁面不會對外發出任何網路請求。
- **拒絕被嵌入**：頁面載入時檢查 `window.self !== window.top`，被 iframe 嵌入時直接擋下整段程式碼。
- **混合加密**：以 RSA-OAEP-256 包裝一把當場產生的隨機 AES-256-GCM 金鑰，再用該金鑰加密使用者輸入的欄位。輸出格式是 `HERMES-SECRET-ENVELOPE-V1:` 前綴加上一段 base64url 編碼的 JSON；JSON 內的 `wrapped_key`、`iv`、`ciphertext` 欄位使用標準 base64。私鑰不在頁面內。
- **加密成功後清除明文欄位**：金鑰值輸入框在加密完成後被清空；欄位在加密或複製期間被變更時，已產生的密文會被作廢並從畫面移除。

## 維護者注意事項：輪替金鑰時要一併更新的邊界

輪替時需同步更新 `secure-key-drop/index.html` 內三個值：`PUBLIC_KEY_B64`（base64 編碼的 DER/SPKI 公鑰）、`KEY_ID`，以及畫面上的「接收端指紋」。目前 `KEY_ID`／指紋的定義都是 **DER/SPKI 公鑰 bytes 的 SHA-256 hex 前 24 字元**：

```bash
# 將新的 DER/SPKI 公鑰轉為頁面常數並推導 KEY_ID／指紋
PUBLIC_KEY_B64=$(base64 < public-key.der | tr -d '\n')
KEY_ID=$(sha256sum public-key.der | cut -c1-24)
printf 'PUBLIC_KEY_B64=%s\nKEY_ID=%s\n' "$PUBLIC_KEY_B64" "$KEY_ID"
```

更新公鑰或 `KEY_ID` 會改變 `<script>` 內容，因此必須重新計算 CSP；畫面上的指紋是 `<script>` 外的 HTML 文字，**只改這段文字本身不會改變 script hash**。以下命令會從 HTML 擷取 inline `<script>`／`<style>` 的精確內容，印出應填回 CSP `<meta>` 的 hash：

```bash
python3 - <<'PY'
from pathlib import Path
import base64, hashlib, re

html = Path('secure-key-drop/index.html').read_text(encoding='utf-8')
for tag in ('script', 'style'):
    body = re.search(rf'<{tag}>(.*?)</{tag}>', html, re.S).group(1).encode()
    digest = base64.b64encode(hashlib.sha256(body).digest()).decode()
    print(f"{tag}-src 'sha256-{digest}'")
PY
```

最後用「最短路徑」啟動本機 HTTP server，確認瀏覽器 console 沒有 CSP violation、頁面顯示的指紋等於新 `KEY_ID`，並實際加密一筆測試資料，確認輸出具有 `HERMES-SECRET-ENVELOPE-V1:` 前綴且接收端能用對應新私鑰解密；未完成接收端 round trip 前不要部署。
