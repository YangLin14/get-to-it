# Get To It — V3 Webapp Handoff

**狀態**: Planned（V2.5 ClawHub Skill 完成後開始）
**目標**: 零安裝、瀏覽器直接使用、自帶 API Key、Freemium 收費

---

## 核心定位

V3 是 Get To It 的**收費引擎**。V1/V2 Cowork Skill 和 V2.5 ClawHub Skill 是品牌建立和社群管道，V3 是實際獲利的地方。

**最大優勢**：「自帶 API Key」(BYOK) 模型讓 Yang 的主機成本幾乎為零——用戶自己付 OpenAI/Anthropic/Google 的費用，Yang 只需要托管一個超輕量後端。收支平衡只需要 ~50 付費用戶。

---

## 定價策略

| 方案     | 價格      | 功能                                                                              |
| -------- | --------- | --------------------------------------------------------------------------------- |
| **Free** | $0        | 1 個目標、3 個專案、無限任務、本地資料（IndexedDB）、無 AI 功能                   |
| **Pro**  | $8/mo     | 無限目標/專案、所有 AI 功能（BYOK）、行事曆整合、長期記憶、ML 排序、Telegram 推送 |
| **Team** | $20/mo/人 | Pro + 多人協作、共享專案、團隊動量追蹤                                            |

> **BYOK = Bring Your Own Key**：用戶在設定頁面貼上自己的 OpenAI / Anthropic / Google API Key，加密存在瀏覽器 LocalStorage，不經過 Yang 的伺服器。AI 呼叫直接從用戶瀏覽器打到模型供應商。

---

## 技術架構

### 後端（極簡）

```
FastAPI (Python)
├── /api/auth          — 簡單 JWT，支援 GitHub / Google OAuth
├── /api/sync          — 多設備資料同步（Pro 功能）
├── /api/users         — 帳號管理、訂閱狀態（Stripe webhook）
└── /api/health        — 健康檢查
```

**重點**：gti.py 的邏輯**不**放在後端。AI 呼叫從前端直接打出去（BYOK），gti.py 的 SQLite 邏輯改寫成同等的後端 DB 操作（PostgreSQL / SQLite for single-user）。

或者更激進的選項：把 gti.py 用 Pyodide（Python in WebAssembly）跑在瀏覽器裡，Free 方案完全不需要後端，真正 local-first。

### 前端

```
React + Vite + Tailwind
├── /morning           — 晨間簡報儀表板
├── /goals             — 目標/專案/任務樹
├── /capture           — 快速捕捉浮層（全局快捷鍵）
├── /review            — 動量時間軸 + 統計
├── /settings          — API Key 設定、行事曆連結、模型選擇
└── /ideas             — 想法庫
```

### AI 整合

```javascript
// 前端直接呼叫，Key 永不離開瀏覽器
const response = await fetch("https://api.openai.com/v1/chat/completions", {
  headers: { "Authorization": `Bearer ${userApiKey}` },
  body: JSON.stringify({ model: "gpt-4o", messages: [...] })
})
```

支援模型：

- OpenAI: gpt-4o, gpt-4o-mini
- Anthropic: claude-sonnet-4-x, claude-haiku-4-x
- Google: gemini-1.5-pro, gemini-1.5-flash
- 本地模型（未來）: Ollama endpoint

### 資料儲存

```
Free 方案:  IndexedDB（瀏覽器本地，零後端）
Pro 方案:   PostgreSQL（Supabase 托管，$25/mo 起，支援 10k 用戶）
```

---

## 功能優先順序

### Phase 1 — MVP Webapp（先能用）

- [ ] 帳號系統（GitHub OAuth）
- [ ] 目標/專案/任務 CRUD（無 AI）
- [ ] 晨間簡報（接 gpt-4o-mini，用自己的 Key）
- [ ] 完成/跳過任務
- [ ] Stripe 訂閱（Free / Pro）

### Phase 2 — 功能完整

- [ ] 行事曆整合（iCal URL）
- [ ] 長期記憶（向量搜尋，用 pgvector 或 browser-side TF-IDF）
- [ ] ML 權重學習（從 V2 的 gti.py 邏輯移植）
- [ ] Telegram 推送（透過 Bot API）
- [ ] 多設備同步

### Phase 3 — 成長

- [ ] Team 方案
- [ ] 公開動量頁面（可分享的進度卡片）
- [ ] 插件系統（連接 Notion、Linear、GitHub Issues）
- [ ] Mobile PWA（離線支援）

---

## 競品對比（定價錨點）

| 工具             | 定價      | 差異                               |
| ---------------- | --------- | ---------------------------------- |
| Motion           | $19/mo    | AI 行程安排，不懂你的動力狀態      |
| Reclaim.ai       | $8-16/mo  | 行事曆優化，無人格層               |
| Todoist          | $4-8/mo   | 純清單，無 AI 優先排序             |
| **Get To It V3** | **$8/mo** | 動力感知 + 秘書人格 + BYOK（隱私） |

---

## 技術棧建議

```
前端:    React 18 + Vite + Tailwind + Zustand（狀態）
後端:    FastAPI + SQLAlchemy + Alembic（遷移）
DB:      Supabase（PostgreSQL + Auth + Realtime）
付款:    Stripe（訂閱 + webhook）
部署:    Vercel（前端）+ Railway / Render（後端）
CI/CD:   GitHub Actions
```

**總月費估算（初期）**：

- Supabase Free → Pro: $25/mo
- Railway backend: $5-20/mo
- Vercel: Free
- **合計: ~$30-45/mo → 收支平衡只需 6 個付費用戶**

---

## gti.py 重用策略

gti.py 的**資料庫 schema** 和**算法邏輯**完全可以移植：

1. **Schema**：把 19 張 SQLite 表直接移到 PostgreSQL（幾乎零修改）
2. **morning_brief 評分邏輯**：複製到後端 Python 函式
3. **TF-IDF 向量引擎**：可以跑在後端（server-side），或用 pgvector 替換
4. **EWMA 校準**：直接複製
5. **Logistic regression 權重學習**：直接複製

**不需要重寫**，只需要把 CLI 入口換成 FastAPI endpoint。

---

## 行銷定位

**目標客群**：知識工作者、獨立創業者、ADHD 族群、創意工作者
**核心訊息**：「給想法太多、容易分心、但真的想前進的人的 AI 私人秘書」
**分發策略**：

1. ClawHub Skill（開源，免費）→ OpenClaw 社群口碑
2. Product Hunt 上線
3. ADHD + 生產力 Reddit / Threads / X 社群
4. YouTube demo（對比 Motion、Notion）

---

_V3 開始時間：V2.5 ClawHub Skill 發布後，收到足夠社群反饋確認方向。_
