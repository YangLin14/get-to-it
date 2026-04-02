# Get To It — V2 開發交接文件

> **目的**：讓新的 chat session 可以直接銜接 V2 開發，無需重新理解背景。

---

## 一、MVP（V1）現況總結

### 已完成的核心功能

| 模組 | 狀態 | 說明 |
|------|------|------|
| SQLite 資料庫 | ✅ 完成 | WAL mode，concurrent-safe，6 張 tables |
| 7 因素優先排序引擎 | ✅ 完成 | morning brief 輸出 Top 3 + 理由 |
| 想法捕捉系統 | ✅ 完成 | 自動評估關聯度、動力、建議行動 |
| 14 天短期記憶 | ✅ 完成 | 儲存跳過原因、情緒背景 |
| 3-strike 秘書介入 | ✅ 完成 | 任務被跳過 3 次自動觸發 check-in |
| 動量計算 | ✅ 完成 | 客觀（完成數）+ 主觀（每日 emoji mood） |
| 人機協作任務 | ✅ 完成 | assignee = human / agent，6 種狀態機 |
| 秘書人格系統 | ✅ 完成 | 4 個模式：歡迎 / 正常 / 溫柔施壓 / 強力施壓 |
| Cowork Skill 封裝 | ✅ 完成 | get-to-it.skill 可安裝 |
| 使用手冊 | ✅ 完成 | 使用手冊.html（繁體中文） |

### 檔案位置（已安裝 skill 路徑）

```
Get To It/
├── skill/get-to-it/
│   ├── SKILL.md              ← AI 秘書行為規範
│   ├── scripts/gti.py        ← 完整 CLI + 資料庫層（~550 行）
│   └── references/persona.md ← 語氣範本和對話模板
├── system-diagram.html        ← v0.2 系統架構圖
├── 使用手冊.html              ← 對外說明文件
├── get-to-it.skill            ← 可安裝封裝
└── V2-handoff.md              ← 本文件
```

### gti.py 指令清單

```bash
init                          # 初始化資料庫
morning [--hours N]           # 早間簡報 Top 3
capture "text"                # 快速捕捉想法
add-goal / add-project / add-task  # 新增目標/專案/任務
complete / skip / pause / resume   # 任務操作
review [--project ID]         # 進度回顧
mood fire|neutral|tired       # 記錄今日情緒
ideas [--all]                 # 查看想法庫
promote-idea IDEA_ID PRJ_ID   # 想法升格為任務
memory / remember             # 短期記憶操作
log [--days N]                # 行動日誌
today-hours [N]               # 查詢或設定今日可用時間
status                        # 全系統狀態（JSON）
list-goals / list-projects    # 列表查詢
```

### 目前 tasks 資料表 schema（V2 需擴充）

```sql
CREATE TABLE tasks (
    id, project_id, title, description,
    assignee TEXT DEFAULT 'human',
    status TEXT DEFAULT 'pending',     -- pending/in_progress/completed/skipped/paused/agent_processing/agent_failed
    estimated_minutes INTEGER,
    skip_count INTEGER DEFAULT 0,
    skip_reasons TEXT DEFAULT '[]',
    deadline TEXT,
    priority INTEGER DEFAULT 5,
    created_at, updated_at, completed_at
);
```

---

## 二、V2 功能規劃

### 功能 1：🕐 實際完成時間追蹤 + 預測時間自動校正（新增，Yang 特別要求）

**核心概念**：現在 `estimated_minutes` 是人工設定的靜態值。V2 要讓系統學習 Yang 真實的完成速度，持續修正每個任務類型的時間預測。

**資料庫變更**：

```sql
-- tasks 表新增欄位
ALTER TABLE tasks ADD COLUMN actual_minutes INTEGER;        -- 實際花費時間（分鐘）
ALTER TABLE tasks ADD COLUMN started_at TEXT;               -- 開始時間戳

-- 新增 time_estimates 表，學習每種任務的個人時間模型
CREATE TABLE time_estimates (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    task_category TEXT,              -- 任務類別（e.g. "coding", "writing", "research"）
    goal_id INTEGER,                 -- 也可以按目標學習
    estimated_minutes_median REAL,   -- 滾動中位數（實際完成時間）
    sample_count INTEGER DEFAULT 0,  -- 樣本數
    accuracy_ratio REAL DEFAULT 1.0, -- actual/estimated 的滾動平均比值
    updated_at TEXT
);
```

**新指令**：

```bash
gti.py start TASK_ID              # 記錄開始時間（started_at = now）
gti.py complete TASK_ID           # 自動計算 actual_minutes = now - started_at
gti.py complete TASK_ID --minutes N  # 手動輸入實際時間（如果忘記 start）
gti.py time-stats                 # 顯示預測準確度統計
```

**校正邏輯**：

```python
def update_time_model(task_id, actual_minutes):
    # 1. 取得該任務的 estimated_minutes 和 category
    # 2. 計算 accuracy_ratio = actual / estimated
    # 3. 用滾動加權平均更新 time_estimates.accuracy_ratio
    # 4. 下次同類任務預測：new_estimate = user_input * accuracy_ratio

def smart_estimate(category, user_raw_estimate):
    ratio = get_accuracy_ratio(category)
    return round(user_raw_estimate * ratio)  # 自動校正
```

**SKILL.md 新增規則**：
- 建議使用者在開始任務時說「開始做 #3」自動記錄
- 完成時自動計算並顯示：「✅ 完成！實際花了 45 分鐘，你預估了 30 分鐘。你在這類任務通常會超時 1.5x，下次我幫你自動調整。」
- 當預測準確度 < 70%（估算偏差超過 30%）時，morning brief 顯示校正後的時間

---

### 功能 2：📅 Calendar API 整合（V2 重點）

**目標**：讀取使用者當天的行事曆，把「真實可用時間」自動帶入 morning brief，不再手動輸入。

**支援來源**（優先順序）：
1. Google Calendar API（OAuth）
2. Apple Calendar（ical 本地讀取）

**整合點**：
- `morning_brief()` 自動讀取今日 events，計算空白時段 → `available_hours`
- 若有重要 meeting 前有任務，可標示 "⚠️ 在 10:00 會議前需完成"
- SKILL.md 新增：「如果連結了行事曆，不要再問今天幾小時可以工作」

**MVP 實作方式**（最低成本）：
```bash
# 讓使用者提供 ical URL（Google Calendar 可匯出）
gti.py connect-calendar "https://calendar.google.com/calendar/ical/.../basic.ics"
gti.py sync-calendar   # 每次 morning 前自動呼叫
```

---

### 功能 3：🧠 向量記憶層（長期脈絡）

**目前 V1 問題**：短期記憶只有 14 天，超過就消失。使用者說過的重要背景（「我討厭做報告」「這個專案是給老闆看的」）沒有長期保留。

**V2 做法**：
- 本地 Vector DB（Chroma 或 FAISS）
- 每次互動的重要陳述自動嵌入
- Morning brief 時 semantic search 找相關記憶帶入提示

**實作路徑**：
```python
# pip install chromadb sentence-transformers
import chromadb
client = chromadb.PersistentClient(path=f"{DB_DIR}/.gti-vectors")
collection = client.get_or_create_collection("user_context")

def store_long_term(text, metadata):
    collection.add(documents=[text], metadatas=[metadata], ids=[uuid])

def recall_relevant(query, n=3):
    return collection.query(query_texts=[query], n_results=n)
```

---

### 功能 4：🤖 Agent 任務失敗自動處理（V2 完善）

**V1 現況**：`agent_processing` 和 `agent_failed` 狀態已定義，但沒有 fallback 邏輯。

**V2 新增**：
- 任務進入 `agent_failed` 時，自動通知使用者並詢問是否重試 / 改由人工完成
- Agent 超時定義（e.g. 30 分鐘無更新視為 timeout）
- 新指令：`gti.py agent-status` 顯示所有 agent 任務進度

---

### 功能 5：📊 ML 權重自動調整（V2 長期）

**目標**：根據使用者實際完成了哪些被推薦的任務，自動學習哪些因素對他最有預測力。

**方式**：
- 記錄每次 morning brief 的 Top 3 和最終完成哪一個
- 用 logistic regression 更新 6 個因素的權重
- 存在 `user_preferences` 表

**注意**：需要足夠樣本量（建議 >30 天資料）才有意義，可作為後期功能。

---

## 三、V2 開發優先順序

```
Priority 1（核心價值）：
  ✦ 實際時間追蹤 + 預測校正（Yang 特別要求）
  ✦ Calendar 整合（減少手動操作）

Priority 2（深度提升）：
  ✦ 向量長期記憶
  ✦ Agent 失敗處理完善

Priority 3（未來）：
  ✦ ML 權重自動調整
  ✦ Telegram Bot 整合（via OpenClaw）
```

---

## 四、技術注意事項

### 環境設定

```bash
# 每次執行前必須設定
export GTI_DB_DIR="/path/to/Get To It"  # 注意路徑有空格要加引號
python gti.py <command>
```

### 已知問題與解法

| 問題 | 解法 |
|------|------|
| 路徑有空格時 inline env 會斷掉 | 用 `export GTI_DB_DIR=...` 分開設定 |
| skills/ 目錄為 read-only | 在工作目錄建好再打包成 .skill |
| 打包 .skill 要在 /tmp 做 | `zip` 到臨時路徑再 `cp` 進 mnt |

### SQLite Schema 更新策略

V2 新增欄位用 `ALTER TABLE` 而不是 drop/recreate，避免使用者資料遺失。在 `init` 指令中加入 migration logic：

```python
def migrate_db(conn):
    """Run on every init — idempotent migrations"""
    migrations = [
        "ALTER TABLE tasks ADD COLUMN actual_minutes INTEGER",
        "ALTER TABLE tasks ADD COLUMN started_at TEXT",
        "CREATE TABLE IF NOT EXISTS time_estimates (...)",
    ]
    for sql in migrations:
        try:
            conn.execute(sql)
        except sqlite3.OperationalError:
            pass  # Column already exists
```

---

## 五、給新 Session 的第一步

1. **讀取現有 gti.py**：`/sessions/.../mnt/Get To It/skill/get-to-it/scripts/gti.py`
2. **讀取 SKILL.md**：`/sessions/.../mnt/Get To It/skill/get-to-it/SKILL.md`
3. **從「實際時間追蹤」開始**（Priority 1，Yang 特別要求）
4. 在現有 gti.py 基礎上 **用 migrate_db 模式** 新增欄位，不破壞現有資料
5. 完成後重新打包 .skill 並讓使用者重新安裝

---

*最後更新：2026-03-31*
