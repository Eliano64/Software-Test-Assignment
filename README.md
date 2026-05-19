# AI 驱动 AutoTestDesign 工具（Assignment2）

本项目用于 Assignment2，支持需求结构化、风险分析、黑盒测试设计、白盒建模（可选）、测试预言（可选）与测试套件优化（可选）。目标应用为 FitnessAI。


## 1. 项目结构

```
.
├─ ai-service/                 # FastAPI，负责调用 LLM 生成测试用例
│  ├─ app/main.py
│  ├─ Dockerfile
│  └─ requirements.txt
├─ backend/                    # Express，负责编排、质量分析、落库、历史接口
│  ├─ src/index.js
│  ├─ src/db.js
│  ├─ Dockerfile
│  └─ package.json
├─ frontend/                   # Vue + Vite 前端（聊天式界面）
│  ├─ src/App.vue
│  ├─ src/style.css
│  └─ package.json
├─ infra/postgres/init.sql     # PostgreSQL 初始化脚本
├─ docker-compose.yml
├─ .env.example
└─ README.md
```


## 2. 技术栈与职责

- `frontend`：多源输入、结构化审查、风险评估、测试用例审查与导出（Markdown/JSON/CSV）
- `backend`：
  - 接收生成请求 `/api/testcases/generate`
  - 调用 `ai-service`
  - 质量分析、导出与历史记录
- `ai-service`：
  - 构造 Assignment2 Prompt
  - 调用 OpenAI 兼容接口（支持 `OPENAI_BASE_URL`）
  - 输出结构化 JSON + `llmRawOutput`
- `postgres`：持久化生成记录

## 2.1 目标应用与测试重点

目标应用固定为 FitnessAI（智能健身辅助系统），核心测试范围如下：

- /api/analytics/pose 姿态分析接口
- 状态机计数与防误检
- 训练记录过滤规则
- 训练计划模式
- 仪表盘统计与卡路里估算

参考上下文与范围说明：
- [FitnessAI_LLM_CONTEXT.md](FitnessAI_LLM_CONTEXT.md)
- [DEVELOPMENT_LOG.md](DEVELOPMENT_LOG.md)

---

## 3. 环境准备

### 3.1 先决条件

- Windows + Docker Desktop
- Node.js 20+
- npm 10+（建议）

### 3.2 Assignment2 符合性概览

已覆盖 Assignment2 必做 FR：

- FR 1.0 输入/解析：文件、纯文本、CSV 多源导入
- FR 1.1 需求结构化：输出 requirementsStructured 并支持审查
- FR 2.0 风险分析与优先级：输出 riskItems（impact/likelihood/priority）
- FR 3.0 黑盒测试设计：EP/BVA/DecisionTable 至少三类方法
- FR 6.0 输出与导出：Markdown/JSON/CSV 导出
- 交互式审查：artifacts/testcases/traceability 可编辑并应用

已覆盖可选加分项：

- FR 4.0 白盒建模：输出状态模型与覆盖准则
- FR 5.0 测试预言：用例包含 oracle 字段
- FR 7.0 测试套件优化：输出风险优先或最小化结果

实现细节见：[DEVELOPMENT_LOG.md](DEVELOPMENT_LOG.md)

### 3.3 配置环境变量

在项目根目录复制：

```powershell
Copy-Item .env.example .env
```

`.env.example` 关键项：

```env
POSTGRES_DB=aitest
POSTGRES_USER=aitest
POSTGRES_PASSWORD=aitest123
POSTGRES_PORT=5432

BACKEND_PORT=3000
AI_SERVICE_PORT=8000
FRONTEND_PORT=5173

TEST_TECHNIQUE=black-box

OPENAI_API_KEY=
OPENAI_MODEL=
OPENAI_BASE_URL=
ENABLE_PARSE_FALLBACK=false
```

说明：

- `OPENAI_API_KEY`：填写真实 Key 才会调用线上模型
- `OPENAI_MODEL`：例如 `deepseek-chat`、`gpt-4o-mini` 等
- `OPENAI_BASE_URL`：兼容网关地址（例如 DeepSeek/OpenAI 代理）
- `ENABLE_PARSE_FALLBACK=false`：建议保持 false，避免“解析失败后回退 mock”影响一致性

---

## 4. 一键启动（推荐）

在项目根目录：

```bash
docker compose up -d --build
```

查看状态：

```bash
docker compose ps
```

预期容器：

- `aitest-postgres`
- `aitest-ai-service`
- `aitest-backend`

健康检查：

- 后端健康：`http://localhost:3000/health`
- AI 服务健康：`http://localhost:8000/health`

停止服务：

```bash
docker compose down
```

清空数据库卷（谨慎）：

```bash
docker compose down -v
```

---

## 5. 前端启动

前端本地运行（建议开发时这样做）：

```bash
cd frontend
npm install
npm run dev
```

访问：

- 前端：`http://localhost:5173`

---

## 6. 联调测试流程

### Step 1：确认后端和 AI 服务正常

浏览器访问：

- `http://localhost:3000/health`
- `http://localhost:8000/health`

### Step 2：前端上传文件并发送

在前端页面中：

1. 点击底部“导入文件”上传需求文档（可多文件）
2. 在底部输入框填写 Prompt（可留空使用默认 Prompt）
3. 点击“发送”
4. 在中部结果窗口查看 LLM 输出（Markdown）

### Step 3：验证历史能力

1. 点击右上“历史记录”
2. 选择一条记录“查看详情”回填
3. 观察结果区出现“当前回填: #ID ...”标志
4. 可测试“删除”功能

### Step 4：导出结果

点击结果区“导出 Markdown”，检查下载文件内容。

---

## 7. API 速查

### 7.1 生成测试用例

`POST /api/testcases/generate`

请求示例：

```json
{
  "sourceType": "requirements",
  "content": "",
  "promptMode": "custom",
  "customPrompt": "请输出黑盒测试用例 Markdown",
  "documents": [
    {
      "name": "req.md",
      "type": "text/markdown",
      "content": "登录、锁定、找回密码"
    }
  ],
  "testTechnique": "black-box"
}
```

说明：

- `content` 与 `documents` 至少一个非空
- 前端当前主要走 `documents + customPrompt`

### 7.2 历史记录

- `GET /api/history?limit=20`
- `DELETE /api/history/:id`

### 7.3 实验分析指标

- `GET /api/analysis/experiment?limit=200`

---

## 8. 常见问题

### Q1: 前端点发送后报错 “Failed to generate test cases”

排查顺序：

1. 是否已上传至少一个文件
2. `backend` 与 `ai-service` 是否健康
3. 查看后端日志：

```bash
docker compose logs -f backend
```

4. 查看 AI 服务日志：

```bash
docker compose logs -f ai-service
```

### Q2: Docker 构建 Python 依赖失败（网络问题）

项目已在 `ai-service/Dockerfile` 使用清华源，若仍失败可检查本地网络代理。

### Q3: 结果区数量或内容异常

确认 `.env` 中：

```env
ENABLE_PARSE_FALLBACK=false
```

避免解析失败后回退 mock 造成理解偏差。

### Q4: 端口冲突

修改 `.env` 的 `BACKEND_PORT / AI_SERVICE_PORT / FRONTEND_PORT` 后重启。

---

## 9. 提交建议

- 前端改动主要在：`frontend/src/App.vue`, `frontend/src/style.css`
- 后端接口改动主要在：`backend/src/index.js`
- 生成逻辑改动主要在：`ai-service/app/main.py`
- 每次提交前至少执行一次：

```bash
cd frontend && npm run build
```

并确认健康接口可访问。

---

## 10. 当前版本说明

- 结构化需求 + 风险分析 + 黑盒测试设计 + 交互式审查
- 可选白盒建模、测试预言与测试套件优化
- 导出格式：Markdown / JSON / CSV

## 11. 建议演示流程

1. 启动后端、AI 服务和前端
2. 打开前端页面，说明目标应用为 FitnessAI
3. 点击“填入示例”，展示纯文本与 CSV 多源输入
4. 点击“发送”，生成结构化需求、风险、测试用例、状态模型、预言和优化结果
5. 展示 Assignment2 符合性面板与质量指标
6. 手动修改一个风险项或测试用例并应用，演示交互式审查能力
7. 导出 Markdown、JSON、CSV 作为测试工件

更完整的演示建议与验证清单见：[DEVELOPMENT_LOG.md](DEVELOPMENT_LOG.md)

## 12. 通用 Prompt 模板（建议）

为避免将工具能力误写为被测需求，可在底部 Prompt 输入框粘贴以下内容：

```
请为目标应用 FitnessAI 智能健身辅助系统生成测试设计工件。当前导入的内容是 FitnessAI 的需求、接口说明或 CSV 需求，请只围绕 FitnessAI 进行分析，不要把 AutoTestDesign 工具本身作为被测对象。

请按照系统固定的测试设计要求生成结构化结果：需求结构化、覆盖项识别、风险分析、至少三种黑盒测试设计、状态模型、测试预言、风险优先套件优化和 traceability。请保持后端要求的 JSON 输出格式。
注意：文件/文本/CSV 导入能力和 JSON/CSV/Markdown 导出能力属于 AutoTestDesign 工具自身，不要把这些工具能力转写成 FitnessAI 的被测功能需求。

FitnessAI 重点测试范围：
1. /api/analytics/pose 姿态分析接口。
2. exerciseType 支持 SQUAT、PUSHUP、PLANK、JUMPING_JACK。
3. MediaPipe landmarks 理论长度为 33，需要覆盖 32、33、34 等边界。
4. 深蹲或俯卧撑状态机应覆盖 UP、DESCENDING、DOWN、ASCENDING、UP 完整循环，非法短循环不应计数。
5. count < 3 且 durationSeconds < 30 的训练记录应被过滤。
6. 计划模式应覆盖不同难度、组数、次数、休息时间和跳过休息。
7. 仪表盘应覆盖今日统计、历史趋势、运动类型分布和卡路里估算。
```
