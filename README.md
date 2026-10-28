# read-plurk：讓 Claude 讀噗浪

貼上噗浪連結，Claude 會自動抓下噗文與全部回應，整理成：

- **重點整理**：這噗在講什麼、討論怎麼發展
- **噗友風向**：各種立場的則數、人數與代表回應
- **留言結構**：有多少人留言、是不是少數人在洗版、誰跟誰在吵

不用再手動開分頁、檢視原始碼。匿名噗、上千則回應的大噗、附截圖的噗都能處理。

## 使用條件

雲端環境通常連不到 plurk.com，所以需要以下其中一種，讓 Claude 透過你電腦上的瀏覽器讀噗：

- Claude 桌面版（內建瀏覽器）
- Chrome ＋ Claude in Chrome 擴充功能

好友限定的噗，需要先在那個瀏覽器登入噗浪。

## 安裝

### Claude 網頁版／桌面版（claude.ai）

1. 下載本 repo 的 [`dist/read-plurk.zip`](dist/read-plurk.zip)
2. 到 Claude 設定裡的 Skills 頁面，上傳這個 zip

### Claude Code

```
/plugin marketplace add helenlin0621/read-plurk-skill
/plugin install read-plurk@read-plurk-skill
```

## 使用方式

直接貼連結就好，例如：

> 幫我讀這噗 https://www.plurk.com/p/xxxxxxxx

## 注意

- 只讀不寫：絕不會替你回噗、按喜歡或轉噗。
- 政治或爭議議題會以中立方式描述兩邊，不判斷誰對。
- 本工具使用噗浪網頁本身的資料介面，噗浪改版時可能失效。

## 修改 Skill 之後

改完 `skills/read-plurk/SKILL.md`，發佈前要做三件事（指令以 Windows cmd.exe 為準，在 repo 根目錄執行）：

1. 把 `.claude-plugin/plugin.json` 的 `version` 往上調。版本號沒變，`claude plugin update` 不會抓新內容。
2. 重新打包給 claude.ai 上傳用的 zip：

   ```
   tar -a -cf dist\read-plurk.zip -C skills read-plurk
   ```

   Windows 內建的 `tar` 才能打成 zip；Git Bash 的 `tar` 打出來的不是 zip。

3. 已經裝好的 Claude Code 要更新才會吃到新版：

   ```
   claude plugin update read-plurk@read-plurk-skill
   ```

## 授權

MIT
