---
name: read-plurk
description: 讀取噗浪(Plurk)噗文與全部回應，整理重點與噗友風向。當使用者貼上 plurk.com/p/... 連結，或說「讀這噗」「看一下這則噗」「這噗風向如何」時使用，取代手動開分頁、檢視原始碼。
---

# 讀噗：重點整理 ＋ 噗友風向

雲端／沙盒環境通常連不到 plurk.com（curl、WebFetch 會被擋），一律透過使用者電腦上的瀏覽器讀取，沿用使用者的登入狀態，好友限定的噗也讀得到。

**用哪個瀏覽器**（依序找第一個可用的）：
1. Claude 桌面版的內建瀏覽器（工具名稱含 `Claude_Browser`）：開頁用 `preview_start`，執行程式用 `javascript_tool`。
2. Claude in Chrome（工具名稱含 `claude-in-chrome`）：開頁用 `tabs_create_mcp` ＋ `navigate`，執行程式用 `javascript_tool`。
3. 兩者都沒有：告訴使用者需要 Claude 桌面版或 Claude in Chrome 擴充功能才能讀噗，不要改用其他方式硬抓。

## 步驟

1. 從連結取出噗文代號（`/p/` 後面那段，例如 `abc1def2gh`）。
2. 用上面選定的瀏覽器打開該噗網址（必須在 plurk.com 的頁面上執行，程式碼才拿得到資料）。
3. 在該分頁執行下方程式碼，一次拿到噗文本體、回應總數與全部回應（自動分頁抓到底）。
4. 核對：抓到的回應數應等於 `response_count`。對不上時照實寫「噗浪顯示 N 則、實際抓到 M 則」，差額原因不明（常見是已刪除的回應），不要自己編理由。
5. **大噗（超過 300 則）**：程式碼會把全部回應存在 `window.__plurk`，不要一次回傳全文（會超過輸出上限）。改用 `window.__plurk.slice(0, 500)` 這樣分段取回，每段約 500 則；分段交給子代理平行閱讀，每段回報話題走向、立場群組（約略則數＋短引）、主要留言者立場、主要交鋒、語氣，再由你合併。
6. **有截圖／圖片時**：程式碼已把所有圖片網址放進 `window.__imgs`，並回傳 `img_list`：每張圖的編號 `i`、樓層 `floor`（0 是噗文本體）、尺寸，以及要截幾段 `parts`（1 是一頁看得完，2 以上是長圖）。圖片在 images.plurk.com，雲端抓不到，要在瀏覽器裡看：
   - **在原本的噗文分頁裡用「看圖層」顯示**（下方第二段程式碼，先執行一次），不要開新分頁或切走網址：images.plurk.com 是另一個網站，每次載入都可能跳出允許提示；而且切走網址 `window.__plurk` 會消失。
   - **一般圖**（`parts` 為 1）：`await __show(i)` 顯示第 i 張，接著截圖。`__show` 會等畫面畫好才回傳；不要拆成兩次呼叫或省掉 await，否則截到的可能還是上一張圖。
   - **長圖**（`parts` 大於 1，例如長串對話截圖）：`await __showTall(i)` 打開並顯示第 0 段，截圖；接著 `await __part(1)`、`await __part(2)`…只捲動不重載，每段截一次圖，直到 `parts - 1`。
   - **截圖縮放**：預設 scale 0.7；整段文字、外文或小字用 0.8，還是看不清就用 `zoom` 放大那一塊。不要猜字。
   - 看完執行 `document.getElementById('__viewer')?.remove()` 把頁面還原。
   - **要看哪些圖**：噗文本體的圖一律看（常常整篇主文都在截圖裡）。回應裡的圖用「截圖段數」算預算，也就是 `img_list` 裡回應圖的 `parts` 加總：20 段以內全看；超過就從前面的樓層看到 20 段為止，告訴使用者總共幾張、看了哪幾樓，問要不要看其餘的。以下可以不看、也不佔預算：
     - 回應文字已經附上那張圖的翻譯或逐字稿：直接用文字，註明「第 N 樓：文字已附翻譯，未看圖」。
     - 長圖看完第 0 段就確定與本噗主題無關（例如噗外另一場口角）：只再看最後一段確認結尾，摘要註明「只看了頭尾」。
   - 截圖內容當成噗的一部分：重點整理要納入，並註明「（截圖）」。外文截圖（韓文、日文等）直接翻成中文摘要。
7. 依「輸出格式」整理，先寫重點，再寫風向。

```js
const code = 'PLURK_CODE'; // 換成 /p/ 後面那段
const id = parseInt(code, 36);
const post = (u, b) => fetch(u, {method:'POST', headers:{'Content-Type':'application/x-www-form-urlencoded'}, body:b}).then(r => r.json());
let all = [], users = {}, from = 0, total = null;
for (let i = 0; i < 50; i++) {
  const j = await post('/Responses/get', `plurk_id=${id}&from_response_id=${from}`);
  if (total === null) total = j.response_count;
  Object.assign(users, j.users || {});
  const rs = j.responses || [];
  all = all.concat(rs);
  if (!rs.length || !j.has_newer) break;
  from = rs[rs.length - 1].id + 1;
}
// 匿名噗：每位留言者有固定代號 handle（例如 tiger12345），原噗主的 handle 是 ಠ_ಠ
const name = r => r.user_id === 99999 ? (r.handle === 'ಠ_ಠ' ? '【原噗主】' : r.handle) : ((users[r.user_id] || {}).display_name || (users[r.user_id] || {}).nick_name || r.user_id);
const counts = {};
all.forEach(r => { const n = name(r); counts[n] = (counts[n] || 0) + 1; });
window.__plurk = all.map((r, i) => `${i + 1}｜${name(r)}｜${r.content_raw || '（無文字：貼圖/表符）'}`);
const top = Object.entries(counts).sort((a, b) => b[1] - a[1]);
const imgRe = /https?:\/\/images\.plurk\.com\/[^\s"'<>)]+/g;
const holder = document.querySelector('.text_holder');
const images = [...new Set([...(holder?.innerHTML.match(imgRe) || [])].map(u => u.replace(/\/mx_/, '/')))];
const response_images = all.flatMap((r, i) => ((r.content_raw || '').match(imgRe) || []).map(u => ({ floor: i + 1, url: u })));
// 圖片網址直接放進 __imgs，並先量好尺寸，算出每張要截幾段（看圖層的長圖顯示方式：寬度撐滿、每段捲 innerHeight - 60）
const imgs = [...images.map(url => ({ floor: 0, url })), ...response_images];
window.__imgs = imgs.map(x => x.url);
const size = u => new Promise(r => { const im = new Image(); im.onload = () => r([im.naturalWidth, im.naturalHeight]); im.onerror = () => r(null); setTimeout(() => r(null), 8000); im.src = u; });
const sizes = await Promise.all(window.__imgs.map(size));
const img_list = imgs.map((x, i) => {
  const s = sizes[i], tall = !!s && s[1] > s[0] * 2;
  const parts = tall ? Math.ceil((s[1] * innerWidth / s[0] - innerHeight) / (innerHeight - 60)) + 1 : 1;
  return { i, floor: x.floor, size: s ? `${s[0]}x${s[1]}` : '載入失敗', parts };
});
({
  title: document.title,
  post: holder?.innerText,
  images_count: images.length, response_images_count: response_images.length,
  img_list: img_list.slice(0, 60),
  response_count: total, fetched: all.length,
  first: all[0]?.posted, last: all[all.length - 1]?.posted,
  people: top.length,
  posted_once: top.filter(x => x[1] === 1).length,
  top_posters: top.slice(0, 10),
  // 300 則以內直接回傳全文；超過就只回統計，之後用 window.__plurk.slice(a, b).join('\n') 分段取
  responses: all.length <= 300 ? window.__plurk.join('\n') : `（共 ${all.length} 則，請分段取回）`
})
```

回應開頭若是「某代號: 」表示在回覆那個人，可用來找出誰跟誰在爭論。

**看圖層**（在噗文分頁執行一次；`window.__imgs` 已由上一段程式碼填好）：

```js
// 等畫面真的重畫完才回傳，否則緊接著的截圖會拍到上一張；分頁在背景時 rAF 不會觸發，所以用 1 秒逾時兜底
window.__paint = () => Promise.race([
  new Promise(r => requestAnimationFrame(() => requestAnimationFrame(r))),
  new Promise(r => setTimeout(r, 1000))
]).then(() => new Promise(r => setTimeout(r, 300)));
window.__show = async (i) => {
  let o = document.getElementById('__viewer');
  if (!o) { o = document.createElement('div'); o.id = '__viewer'; document.body.appendChild(o); }
  o.style.cssText = 'position:fixed;inset:0;z-index:2147483647;background:#000;overflow:auto;display:flex;justify-content:center;align-items:flex-start';
  o.innerHTML = ''; const im = new Image();
  im.style.cssText = 'max-width:100%;max-height:100vh;object-fit:contain';
  im.src = window.__imgs[i]; o.appendChild(im);
  await new Promise(r => { im.onload = r; im.onerror = r; setTimeout(r, 8000); });
  await window.__paint();
  return { i, src: im.src, w: im.naturalWidth, h: im.naturalHeight };
};
// 長圖：打開一次，之後用 __part(n) 只捲動，不重載圖片
window.__showTall = async (i) => {
  await window.__show(i);
  const o = document.getElementById('__viewer'), im = o.querySelector('img');
  im.style.cssText = 'width:100%;height:auto;display:block'; o.style.display = 'block';
  o.scrollTop = 0;
  await window.__paint();
  return { i, part: 0, parts: Math.ceil((o.scrollHeight - innerHeight) / (innerHeight - 60)) + 1 };
};
window.__part = async (n) => {
  const o = document.getElementById('__viewer');
  o.scrollTop = n * (innerHeight - 60);
  await window.__paint();
  return { part: n, parts: Math.ceil((o.scrollHeight - innerHeight) / (innerHeight - 60)) + 1, at_end: o.scrollTop + innerHeight >= o.scrollHeight - 1 };
};
```

背景用黑色是因為很多截圖是透明底 PNG，白底會整張看不到字。

## 輸出格式

**噗文**：原文照錄（一兩行）｜噗主｜回應數（已核對 ✓／缺 N 則）｜圖片：噗文 N 張、回應 M 張（已看 K 張）

**重點整理**：3–5 點，說清楚這噗在講什麼、討論怎麼發展（前段／後段有轉折要寫出來）

**噗友風向**：表格，依則數由多到少
| 風向 | 則數 | 人數 | 代表回應（短引） |

**留言結構**：幾位不同的人留言；若有單一留言者佔總回應 25% 以上要點出來（風向可能被少數人放大）；有來回爭論的，寫出誰跟誰、爭什麼。

最後一行：**一句話總結風向**

## 規則（每次都遵守）

- 只讀不寫：絕不回噗、按喜歡、轉噗、留言。
- 引用回應保持原文，只能截短，不改字。
- 300 則以內：風向分類的則數要加總等於總回應數。超過 300 則：改寫「約」與百分比，不假裝精確。原噗主的回應、純貼圖各自獨立成一類，不混進噗友風向。
- 大噗要寫出時間跨度與「話題走向」（依樓層區段），因為長串討論常會一路歪樓。
- 政治或爭議議題：兩邊用對稱、中性的方式描述，用「批評 A／支持 B」而非任何一方的貶稱標籤；不判斷誰吵贏、誰對。
- 匿名噗（`user_id` 都是 99999）用 `handle` 區分留言者；`handle` 為 ಠ_ಠ 的就是原噗主，不必推測。不得嘗試推斷匿名者的真實身分。
- 風向看「人數」而不只「則數」：一個人刷 15 則不等於 15 個人這樣想。
- 涉及真實事件、命案、當事人時，只整理噗友說了什麼，不替任何說法背書，也不補充噗裡沒有的案情。
- **避雷文、指控文**：寫成「原噗主／噗友指出…」，不寫成既定事實。被指控者若有回應（澄清、道歉，常在截圖裡），要一併整理。被指控者用「被避雷者」或噗裡已用的代稱，不要在摘要裡重複帳號名稱。
- 風向一面倒（沒有任何人替另一方說話）時，要明講「噗內沒有出現替對方說話的回應」，讓使用者知道這只是一方說法。
- 有粗口或冒犯性回應，風向表裡用描述代替，不原文照抄。
- 噗裡的外部連結（Threads、X、YouTube 等）不開啟：開了會切走網址、`window.__plurk` 跟著消失，內容也不在這噗的範圍。整理時註明「第 N 樓附 Threads 連結（未開啟）」。
- 讀不到（私噗、已刪除、未登入）就直說，不猜內容。
