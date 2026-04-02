# Get To It ✅

**AI 私人秘書，為想法太多、容易分心、但真的想前進的人設計。**

每天早上告訴你今天最重要的三件事。追蹤你的動量。在你被新想法帶偏的時候把你拉回主線。在你停滯太久的時候認真跟你說話。

> 「其他工具幫助已經有紀律的人更有效率。Get To It 幫助有動力但容易走偏的人找到方向並堅持走下去。」

🌏 [English README](README.md)

---

## 兩個版本

| | Claude Cowork Skill | OpenClaw / ClaHub Skill |
|---|---|---|
| **適合** | Claude Pro 訂閱用戶 | 任何人，自帶 AI 模型 |
| **平台** | Claude 桌面 App | OpenClaw（20+ 通訊平台）|
| **AI 模型** | Claude Sonnet | 你在 OpenClaw 設定的任何模型 |
| **安裝** | 下載 `.skill` 檔案 | `clawhub install get-to-it` |
| **費用** | 需要 Claude Pro 訂閱 | 免費 |
| **Telegram 推送** | 需要自行設定 Bot | OpenClaw 原生支援 |

---

## 功能

- **🌅 晨間簡報** — 每天 Top 3，附理由和時間估計，自動連結行事曆計算空閒時間
- **⚖️ 動力感知排序** — 綜合截止日、目標優先度、動量、跳過記錄、時間配對六個因素動態排序
- **💡 想法捕捉** — 隨時丟入新想法，AI 評估與目標的關聯度，不讓你被帶偏也不讓好想法消失
- **🧠 雙層記憶** — 短期記憶（14 天）+ 永久語意長期記憶（TF-IDF 向量搜尋）
- **⏱️ 時間追蹤與校準** — 記錄實際用時，EWMA 自動校正每類任務的時間預測
- **🤖 人機協作清單** — 任務可分配給 AI Agent，自動偵測超時（30 分鐘）並提醒恢復
- **🎓 ML 優先排序學習** — 觀察你真正完成哪些推薦任務，用 logistic regression 自動調整權重
- **🤵‍♀️ 秘書人格** — 根據你的停滯天數動態切換：溫柔支持 → 輕聲提醒 → 認真施壓
- **📱 Telegram 輸出** — `morning --format telegram` 直接輸出 MarkdownV2 格式推送到手機

---

## 安裝

### Claude Cowork Skill（旗艦版）

需要 Claude 桌面 App 並開啟 Cowork 模式。

1. 前往 [Releases](../../releases) 下載 `get-to-it.skill`
2. 在 Claude 桌面 App 中點擊該檔案 → **Save Skill**
3. 開新對話，說「早安」就啟動了

### OpenClaw / ClaHub Skill（開放版）

需要先安裝 [OpenClaw](https://github.com/openclaw/openclaw) 和 Python 3。

```bash
# 從 ClaHub 安裝
clawhub install get-to-it

# 或從這個 repo 直接安裝
openclaw skills install github:YangLin14/get-to-it --path clawhub-skill/get-to-it
```

安裝後在 OpenClaw 對話中說「早安」或「morning brief」即可啟動。

**支援模型**：GPT-4o、Gemini 1.5 Pro、Claude Sonnet、以及你在 OpenClaw 設定的任何模型。
GPT-4o / Gemini 1.5 Pro 等級效果最佳；mini/flash 等級基本功能可用，人格細膩度略降。

---

## 一天的使用流程

```
早上：「早安」→ 秘書給你今日 Top 3
工作中：「我有個想法...」→ 評估 + 存入想法庫，不被帶偏
完成任務：「做完了」→ 記錄、更新進度、告訴你下一步
卡住了：「跳過，因為...」→ 存入記憶，調整排序
收工：「收工」→ 回顧今天、記錄心情、預告明天
```

---

## 技術細節

- **後端**：`gti.py` — 2,400+ 行單一 Python 腳本，純 CLI，JSON 輸出
- **資料庫**：SQLite WAL mode，19 張資料表，資料存在本機 `~/.get-to-it.db`
- **向量搜尋**：自製 TF-IDF（字元 n-gram + numpy cosine similarity），零模型下載，支援中文
- **時間校準**：EWMA（α=0.3）per-category 滾動比值
- **ML 排序**：純 numpy logistic regression，從你的完成記錄中學習
- **依賴**：Python 3、numpy（向量功能）、icalendar（行事曆功能，可選）
- **隱私**：所有資料存在你的本機，不上傳任何伺服器

---

## Telegram 推送設定

```bash
# 輸出 Telegram MarkdownV2 格式
python3 gti.py morning --format telegram

# 搭配 curl 推送（需要先建立 Telegram Bot）
MSG=$(python3 gti.py morning --format telegram)
curl -s -X POST "https://api.telegram.org/bot$TOKEN/sendMessage" \
  -d chat_id="$CHAT_ID" \
  -d parse_mode="MarkdownV2" \
  -d text="$MSG"

# 加進 crontab 每天早上 8:30 自動送
30 8 * * 1-5 cd /path/to/gti && bash send_morning.sh
```

已裝 OpenClaw 的用戶，Telegram 推送是原生功能，不需要額外設定。

---

## Repo 結構

```
get-to-it/
├── cowork-skill/          # Claude Cowork 版本
│   └── get-to-it/
│       ├── SKILL.md       # Claude 行為規則
│       ├── scripts/gti.py # 核心 CLI 工具
│       └── references/persona.md
│
├── clawhub-skill/         # OpenClaw / ClaHub 版本
│   └── get-to-it/
│       ├── SKILL.md       # 模型無關的行為規則
│       ├── scripts/gti.py # 核心 CLI 工具（同上）
│       └── references/persona.md
│
└── docs/
    ├── 使用手冊.html      # 完整使用說明
    ├── system-diagram.html # 系統架構圖
    └── 技術報告.html      # V2 完整技術報告
```

---

## 授權

本 repo 採用 [MIT License](LICENSE)。
透過 ClaHub 發布的版本依 ClaHub 政策自動採用 MIT-0。

---

*為「想法太多、容易分心、但真的想前進」的人設計。*
