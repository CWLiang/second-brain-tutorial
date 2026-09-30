# 第二大腦・版本備份 v0.2

> **這是什麼**：給大腦裝一個「可以倒帶」的能力——任何一則筆記被改壞、
> 被誤刪，都救得回來。
>
> **什麼時候需要這一份**：
> - 你開始怕「AI 幫我改筆記，改壞了怎麼辦」
> - 或（Claude Code 使用者）你覺得**「每次要存筆記都要按同意，很煩」**，
>   想放寬——那就**一定要先做完這一份**
>
> **什麼時候不需要**：筆記還很少、而且你每次都看得到 AI 要改什麼 → 先不用急。
>
> **前置**：核心架構裝完、用滿兩週。有常駐主機的話，`machines/always-on-host.md` 也做完。
>
> **你需要什麼**：一個 GitHub 帳號（免費）、大約 30 分鐘。

---

## 平台需求：Claude ✅ / ChatGPT ✅

**主流程（STEP 1–3）完全平台中立。** 它做的是終端機和 launchd 的事，
跟你用哪個 AI 完全無關——git 不知道也不在乎是誰在改你的檔案。

| 這一步在做什麼 | Claude | ChatGPT |
|---|---|---|
| STEP 1　git repo + 推上 GitHub | ✅ | ✅ |
| STEP 2　每晚自動存檔 + 心跳 | ✅ | ✅ |
| STEP 3　AI 開場自檢 | ✅ | ✅ 規則檔完全一樣，差別只在怎麼載入 |
| **選用**：放寬權限（第三部分） | ✅ `--permission-mode` | ❌ 沒有等價的參數 |

最後那一列是**附加選項，不是這份文件的主線**。
ChatGPT 使用者做完 STEP 1–3 就完整了，什麼都沒少。

> ChatGPT 桌面版寫檔案時怎麼問你、問不問，由它的 App 決定，
> **沒有像 Claude Code 那樣可以用參數關掉的開關**。這一點我沒有找到可調的方式，
> 如果你發現有，回來改這一頁。

---

# 這一份在給你什麼

**一句話：你的大腦從此有「昨天的版本」。**

|  | 現在 | 做完這一份之後 |
|---|---|---|
| 一則筆記被改壞 | 只能靠 Google Drive 的版本紀錄一則一則硬翻 | `git checkout` 一行還原 |
| 整包出問題 | 沒辦法 | 退回任何一天 |
| 電腦整台壞掉 | 靠 Drive 救 | `git clone` 就回來了 |

**Drive 和 git 不是二選一**：Drive 管「同步」，git 管「歷史」。兩個一起留著。

---

# 第一部分 · 給 AI 的指示

1. **不要宣稱你做到了還沒確認的事。** 備份掛上去要實際跑一次、去 GitHub 看到 commit。
2. **repo 一定要私有。** 建的時候 GitHub 預設是公開的，要手動改。
3. **STEP 3 不是選配。** 教了自動化就要同一次教「怎麼知道它壞了」。
4. **第三部分的放寬權限是選用的，而且只有 Claude Code 有。**
   一定要在 STEP 1–3 都驗過之後才能做，**順序不可以換**。
   ChatGPT 使用者直接跳過那一節，不要試圖找等價做法。

---

# 第二部分 · 安裝步驟

## STEP 1 — 把大腦放進 git

### 1. 找出 vault 的真實路徑

```bash
ls -d ~/Google\ Drive/*/second-brain   # 做過 multi-device-sync 的人
# 或
ls -d ~/second-brain
```

以下用 `$VAULT` 代表它。

### 2. 建立 repo

```bash
cd "$VAULT"

git init
printf '.obsidian/workspace*\n.DS_Store\n_probe*.md\n' > .gitignore
git add -A && git commit -m "第二大腦：裝上版本備份時的完整快照"
```

`.gitignore` 這三行在排除：Obsidian 的視窗位置（每次開關都變，沒有保存價值）、
macOS 的隱藏檔、以及測試連通用的暫存檔。

### 3. 推到 GitHub 私有 repo

在 `github.com` 開一個新 repo：

- 名字：`second-brain`
- **可見性選 Private**——**這一項最重要，而且 GitHub 預設是 Public**

裡面是他的決定、人際關係、專案細節、可能還有別人託付的事情。
**公開的話，這些會被搜尋引擎索引到。**

然後照 GitHub 頁面上給的兩行指令接上去、推上去。

### 4. 驗收

到 GitHub 的 repo 頁面，應該看得到：

- [ ] 所有筆記都在
- [ ] repo 名稱旁邊寫著 **Private**

**兩項都看到才算完成。**

---

## STEP 2 — 每晚自動存檔

手動 commit 沒有人會記得做。掛一個排程。

### 1. 腳本

```bash
mkdir -p ~/bin && cat > ~/bin/brain-autocommit.sh <<'EOF'
#!/bin/bash
VAULT="/Users/你的帳號/Google Drive/我的雲端硬碟/second-brain"   # 換成真實路徑
cd "$VAULT" || exit 1

git add -A
git diff --cached --quiet || git commit -m "auto-backup: $(date '+%Y-%m-%d %H:%M')"

# 心跳：push 成功才寫，失敗就把錯誤寫進去。
# 這個檔案是 AI 每次開場會看的東西，見 STEP 3。
mkdir -p "$VAULT/_health"
if err=$(git push origin main 2>&1); then
  printf '模組：version-backup\n狀態：正常\n最後成功：%s\n' "$(date '+%Y-%m-%d %H:%M')" \
    > "$VAULT/_health/version-backup.md"
else
  printf '模組：version-backup\n狀態：**失敗**\n時間：%s\n錯誤：%s\n' \
    "$(date '+%Y-%m-%d %H:%M')" "$err" > "$VAULT/_health/version-backup.md"
fi
EOF
chmod +x ~/bin/brain-autocommit.sh
```

兩個地方值得解釋：

- `git diff --cached --quiet ||` 是在說「沒有任何改動就不要 commit」，
  否則每天都會生出一筆空的紀錄
- **心跳那一段是這個模組能不能被發現壞掉的關鍵。** 成功才寫時間戳，
  失敗就把錯誤訊息寫進同一個檔。**腳本整個沒跑的話，這個檔就不會更新**——
  而「沒更新」本身就是訊號，不需要任何東西去偵測失敗

### 2. 排程

`~/Library/LaunchAgents/com.brain.autocommit.plist`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>com.brain.autocommit</string>
  <key>ProgramArguments</key>
  <array><string>/Users/你的帳號/bin/brain-autocommit.sh</string></array>
  <key>StartCalendarInterval</key>
  <dict><key>Hour</key><integer>23</integer><key>Minute</key><integer>30</integer></dict>
  <key>StandardOutPath</key><string>/tmp/brain-autocommit.log</string>
  <key>StandardErrorPath</key><string>/tmp/brain-autocommit.log</string>
</dict>
</plist>
```

載入：`launchctl load ~/Library/LaunchAgents/com.brain.autocommit.plist`

### 3. 驗收

```bash
~/bin/brain-autocommit.sh          # 手動跑一次
cat /tmp/brain-autocommit.log      # 有沒有噴錯
```

然後去 GitHub 看，**要有一筆剛剛產生的 commit**。

**沒看到 commit 就不要往下。** 後面每一步的價值都建立在這一步真的在跑。

---

## STEP 3 — 怎麼知道它壞了（不是選配）

> **自動化的鐵律：教任何自動化，同一次就要教「怎麼知道它壞了」。**

這個備份會**靜默失效**——GitHub 的登入過期、網路斷、git 產生衝突，
全都不會跳通知，而使用的人不會去看 log。

**而它一旦壞掉，如果你做了第三部分的放寬權限，那就變成沒有安全網的裸奔**，
你要到需要還原的那天才會發現。

STEP 2 的腳本已經在寫心跳了（`_health/version-backup.md`）。
這一步是把**讀**那個心跳的規則裝起來。

### 1. 建立健康檢查規則

如果 `_rules/09-健康檢查.md` 還不存在，建立它（內容見 `README.md`
的「共用約定・健康檢查」一節）。已經存在的話跳過這一步——
其他模組可能已經建過了。

### 2. 把這個模組登記進去

在 `_rules/09-健康檢查.md` 的表格裡加一行：

```markdown
| version-backup | `_health/version-backup.md` | 3 天 | 自動備份停了，改壞的筆記救不回來。**如果已經做了第三部分的放寬權限，那就是沒有安全網在裸奔。** |
```

### 3. 驗收（一定要做）

**故意把它弄壞一次，確認 AI 真的會講。**

```bash
mv ~/second-brain/_health/version-backup.md ~/Desktop/   # 路徑換成你的
```

然後開一個新對話，隨便講一句話。**AI 應該在回答你之前先說備份可能壞了。**

沒講的話，代表規則沒被載入——回頭確認 `CLAUDE.md` 有 import
`_rules/09-健康檢查.md`；Cowork 和 ChatGPT 則要確認開場咒語有念到它（三種載入方式見 `README.md` 的「共用約定・健康檢查」）。

驗完把檔案搬回去，或直接手動跑一次 `~/bin/brain-autocommit.sh` 重新產生。

> **沒演練過的監控，跟沒演練過的備份一樣，等於沒有。**

---

# 第三部分 · 選用：放寬權限（**只有 Claude Code**）

> **ChatGPT 使用者跳過這一節。** 沒有等價的設定，不要找。
> 你做完 STEP 1–3 已經是完整的，這一節只是 Claude Code 的一個方便選項。

**STEP 1–3 全部驗過了才能做這一節。**

## 你在換什麼

|  | 現在 | 做完這一節 |
|---|---|---|
| AI 改檔案 | 每次都問你、等你按同意 | 自己改，不問 |
| 改壞了怎麼辦 | 你當下就看得到，可以不按 | **只剩 git 倒帶** |

**這是一組交換，不是升級。** 你放棄「每一次都看得到」，換來「不用一直按」。

所以順序不能反，而且**心跳監控（STEP 3）在這裡從「好習慣」變成「必要條件」**——
備份一旦靜默壞掉，你就是在沒有安全網的狀態下讓 AI 自己改檔案。

### 1. 改掉開機腳本

打開 `~/bin/start-brain-lane.sh`（`always-on-host` STEP 2 建的那支），
在最後那行 `claude` 指令後面加上參數：

```bash
cd "$VAULT" && "$CLAUDE" --bg --name brain --remote-control --permission-mode acceptEdits
```

> **這一步最容易漏。** 只在終端機裡重開一次 lane、忘了改腳本的話，
> 下次重開機又變回每次要按同意，而你會以為是壞了。

### 2. 重開那條 lane

```bash
claude agents                    # 看 brain 在不在
# 在的話先讓它停下來，再跑：
~/bin/start-brain-lane.sh
```

### 3. 驗收

從手機叫它存一則筆記。**它應該直接做完，不再問你同不同意。**

---

# 第四部分 · 改壞了怎麼救

這是你裝這一份的理由，所以要**實際演練一次**，不要等到真的出事才第一次用。

### 情況一：某一則筆記被改壞了

```bash
cd "$VAULT"
git log --oneline -- "vault/某則筆記.md"   # 看它的歷史
git checkout <前一個版本的代號> -- "vault/某則筆記.md"
```

### 情況二：整包都不對，想退回昨天

```bash
cd "$VAULT"
git log --oneline          # 找到昨天那筆 auto-backup
git checkout <代號>         # 先看一眼對不對
git checkout main          # 看完回到現在
```

確定要退回的話再動手還原——**這一步建議請 AI 幫你做**，
它會先確認你要退到哪一筆。

### 情況三：機器整台壞了

在新機器上：

```bash
git clone <你的 repo 網址> second-brain
```

大腦就回來了。

> **演練一次**：現在就故意改壞一則筆記，然後用情況一的方法救回來。
> **沒演練過的備份，等於沒有備份。**

---

# 第五部分 · 這一份刻意不做的事

| 沒有做 | 為什麼 |
|---|---|
| **教 git 的完整概念**（分支、合併、衝突） | 你只需要「存檔」和「倒帶」兩個動作。其他的等真的撞到再學 |
| **GitHub Desktop 圖形介面** | 這一份的動作全部自動化了，平常不用開任何介面。要救檔的時候請 AI 幫你 |
| **把 Google Drive 換掉** | 兩個一起留著沒壞處：Drive 管同步，git 管歷史。**但不要用 git 去同步兩台機器**，那會跟 Drive 打架 |

---

# 附錄 · 這一頁會過期

| 事情 | 現在的做法 |
|---|---|
| 手動存一次檔 | `~/bin/brain-autocommit.sh` |
| 看備份有沒有在跑 | `cat /tmp/brain-autocommit.log`，或去 GitHub 看最後更新時間 |
| 看歷史 | `git log --oneline` |
| 確認排程載入了 | `launchctl list \| grep com.brain` |
| 確認 lane 的權限放寬了 | 叫它存一則筆記，**沒問你就是成功了** |

---

**版本**：v0.2（v0.1 把「放寬權限」當主線，導致整份看起來像 Claude 專屬；
 v0.2 把它降級成選用的第三部分，主流程 STEP 1–3 完全平台中立）
**前置文件**：核心架構裝完、用滿兩週。第三部分的放寬權限另外需要 `machines/always-on-host.md`
