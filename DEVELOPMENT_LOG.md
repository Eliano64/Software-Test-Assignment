# DEVELOPMENT_LOG - Assignment2 AutoTestDesign 改进记录

## 1. 本次开发目标

本项目定位为 Assignment2 要求的 AI 驱动 AutoTestDesign 工具。目标应用明确选定为 `FitnessAI` 智能健身辅助系统，所有风险分析、测试计划、详细测试设计与执行说明均围绕 FitnessAI。

本次改进重点：

- 补齐 Assignment2 必做 FR：FR 1.0、FR 1.1、FR 2.0、FR 3.0、FR 6.0 与交互式审查能力。
- 增强可选加分项：FR 4.0 白盒状态建模、FR 5.0 测试预言、FR 7.0 测试套件优化。
- 将 FitnessAI 的核心模块和风险直接内置到工具流程中，便于课堂演示。
- 优化前端页面，使测试设计流程更清晰、结果更可审查、导出更完整。

## 2. 改进内容

#### 前端 `frontend/src/App.vue`

- 新增 FitnessAI 目标应用工作区卡片，明确展示被测系统、核心模块和重点风险。
- 新增 Assignment2 符合性检查条，覆盖 FR 1.0、FR 1.1、FR 2.0、FR 3.0、FR 6.0、交互式审查及可选加分项。
- 新增纯文本需求输入和 CSV 需求输入，满足“CSV、纯文本或直接用户输入”等多源导入要求。
- 新增“填入示例”按钮，自动写入 FitnessAI 姿态分析、状态机计数、记录过滤、计划模式和仪表盘统计需求。
- 生成请求现在会自动附带 FitnessAI 目标应用上下文，包括核心模块和重点风险。
- 新增结构化质量概览，显示质量评分、结构化需求数、覆盖项数、风险条目数、测试用例数和测试设计方法数。
- 新增后端返回的 Assignment2 符合性展示，能够直观看到各 FR 是否已覆盖。
- 保留并强化交互式审查：测试人员可以编辑 artifacts、testcases、traceability 三类 JSON，并点击“应用修改”更新当前会话。
- Markdown 导出在 LLM 原始输出为空时，会基于审查后的结构化内容生成一份可提交的测试设计报告。

#### 前端样式 `frontend/src/style.css`

- 重构页面为“顶部说明 + 目标应用概览 + 结果审查 + 底部输入”的工作台布局。
- 新增目标应用卡片、输入卡片、Assignment2 符合性面板、指标卡片和响应式布局。
- 优化小屏幕布局，保证在演示设备上也能查看目标应用、输入区和结果区。

#### 后端 `backend/src/index.js`

- 新增 `targetApplication` 元数据，固定目标应用为 FitnessAI，并列出姿态分析、状态机计数、训练计划、记录过滤和仪表盘统计等测试项。
- 新增 `/api/target-application` 接口，用于返回目标应用信息。
- 健康检查 `/health` 增加目标应用名称，方便联调确认当前工具配置。
- 新增 `buildAssignmentCompliance()`，在每次生成后分析 Assignment2 符合性：
  - FR 1.0 输入/解析
  - FR 1.1 需求结构化
  - FR 2.0 风险分析与优先级
  - FR 3.0 至少三类黑盒测试设计方法
  - FR 6.0 输出与导出
  - 交互式审查
  - FR 4.0、FR 5.0、FR 7.0 可选加分项
- `/api/testcases/generate` 返回 `assignmentCompliance`，前端可直接展示符合性证据。

#### AI 服务 `ai-service/app/main.py`

- Prompt 版本升级为 `autotestdesign-v4-fitnessai-assignment`。
- 默认 Prompt 明确要求：所有风险、覆盖项、测试计划和测试用例必须面向 FitnessAI，而不是 AutoTestDesign 工具。Assignment2 的必要要求被转译为 LLM 输出职责，例如需求结构化、风险优先级、至少三种黑盒测试技术、状态模型、测试预言、结构化导出和追溯关系，不再把工具自身的输入/导出 FR 当作 FitnessAI 被测需求传给 LLM。
- Prompt 强化 FitnessAI 关键测试点：
  - `/api/analytics/pose` 姿态分析接口
  - MediaPipe landmarks 数量 32/33/34 边界
  - `exerciseType` 等价类
  - UP/DESCENDING/DOWN/ASCENDING/COOLDOWN 状态机
  - `count < 3 && durationSeconds < 30` 记录过滤决策表
  - 训练计划难度与跳过休息组合
  - 仪表盘卡路里预言
- 离线 mock 产物扩展为 10 条 FitnessAI 测试用例，覆盖 EP、BVA、Combinatorial、StateTransition、DecisionTable 五类方法。
- 离线 mock artifacts 增强为完整结构化产物：
  - 5 条结构化需求
  - 覆盖项列表
  - 风险评分与优先级
  - 状态模型
  - 风险优先测试套件优化
  - 需求-覆盖项-测试用例追溯关系
  - 测试预言 oracle

#### Docker 配置 `docker-compose.yml`

- 将 `ENABLE_PARSE_FALLBACK` 注入 AI 服务容器，保证环境变量与 `.env.example` 一致。

## 3. Assignment2 FR 对应关系

| 要求 | 当前实现 |
| --- | --- |
| FR 1.0 输入/解析 | 支持文件上传、纯文本输入、CSV 输入，并可附带目标应用上下文 |
| FR 1.1 需求结构化 | AI 输出 `requirementsStructured`，前端可审查和编辑 |
| FR 2.0 风险分析与优先级 | AI 输出 `riskItems`，包含 impact、likelihood、riskScore、priority |
| FR 3.0 黑盒测试设计 | 生成 EP、BVA、Combinatorial、StateTransition、DecisionTable 测试用例 |
| FR 4.0 白盒测试建模 | 生成 FitnessAI 运动计数状态模型与覆盖准则 |
| FR 5.0 测试预言生成 | 测试用例包含 oracle 字段 |
| FR 6.0 输出与导出 | 支持 Markdown、JSON、CSV 导出，后端也保留 `/api/export` |
| FR 7.0 测试套件优化 | 输出 `testSuiteOptimization`，采用风险优先策略 |
| 交互式设计审查 | artifacts、testcases、traceability 均可人工修改并应用 |

## 4. FitnessAI 测试重点

- 姿态分析接口：验证运动类型、关键点数量、响应字段和错误处理。
- 状态机计数：验证完整动作循环计数、非法短循环不计数、冷却期防重复计数。
- 记录过滤：验证 `count < 3 && durationSeconds < 30` 的决策表规则。
- 训练计划：验证不同难度的组数、次数、休息时间和跳过休息流程。
- 仪表盘统计：验证趋势、运动分布和卡路里计算预言。

## 5. 建议演示流程

1. 启动后端、AI 服务和前端。
2. 打开前端页面，说明目标应用为 FitnessAI。
3. 点击“填入示例”，展示纯文本与 CSV 多源输入。
4. 点击“发送”，生成结构化需求、风险、测试用例、状态模型、预言和优化结果。
5. 展示 Assignment2 符合性面板和质量指标。
6. 手动修改一个风险项或测试用例，点击“应用修改”，说明交互式审查能力。
7. 分别导出 Markdown、JSON、CSV，作为测试工件。

## 6. 验证建议

- 前端构建：`cd frontend && npm run build`
- 后端健康检查：`GET http://localhost:3000/health`
- AI 服务健康检查：`GET http://localhost:8000/health`
- 生成接口：`POST http://localhost:3000/api/testcases/generate`
- 目标应用接口：`GET http://localhost:3000/api/target-application`

## 7. 后续可扩展项

- 将前端 JSON 编辑区拆分为表格编辑器，降低人工审查门槛。
- 增加测试计划 PDF 模板导出，直接覆盖 Assignment2 报告第 3 项。
- 增加 FitnessAI 的自动化执行脚本模板，例如 PyTest 或 Selenium。
- 增加成本估算页面，对比使用 AutoTestDesign 与手工测试的人时差异。

## 8. 完整项目运行命令

### 8.1 需要更新 Docker 容器

原因是本次开发修改了 `backend`、`ai-service`、`frontend` 和 `docker-compose.yml`。如果继续使用旧容器，后端 Assignment2 符合性分析、AI 服务 FitnessAI Prompt、`ENABLE_PARSE_FALLBACK` 环境变量等改动不会生效。

建议执行重新构建：

```powershell
cd "c:\Users\Project"
docker compose up -d --build
```

如果只是想彻底重启容器但保留数据库历史记录：

```powershell
cd "c:\Users\Project"
docker compose down
docker compose up -d --build
```

不要随意执行 `docker compose down -v`，因为它会删除 PostgreSQL 数据卷，历史生成记录会被清空。

### 8.2 首次运行准备

在项目根目录执行：

```powershell
cd "c:\Users\Project"
Copy-Item .env.example .env
```

如果没有真实大模型 Key，可以保持 `.env` 中 `OPENAI_API_KEY=` 为空。此时 AI 服务会使用内置 FitnessAI mock 产物，仍然可以完整演示结构化需求、风险分析、测试用例、状态模型、测试预言、套件优化和追溯关系。

如果需要连接真实模型，编辑 `.env`：

```env
OPENAI_API_KEY=你的API_KEY
OPENAI_MODEL=deepseek-chat
OPENAI_BASE_URL=你的OpenAI兼容接口地址
ENABLE_PARSE_FALLBACK=false
```

### 8.3 启动后端、AI 服务和数据库

```powershell
cd "c:\Users\Project"
docker compose up -d --build
docker compose ps
```

预期至少看到以下容器处于 `running` 或 `healthy`：

- `aitest-postgres`
- `aitest-ai-service`
- `aitest-backend`

查看日志：

```powershell
docker compose logs -f ai-service
docker compose logs -f backend
```

### 8.4 启动前端

前端当前建议本地运行：

```powershell
cd "c:\Users\Project\frontend"
npm install
npm run dev
```

浏览器访问：

```text
http://localhost:5173
```

### 8.5 健康检查

在浏览器访问：

```text
http://localhost:3000/health
http://localhost:8000/health
http://localhost:3000/api/target-application
```

也可以使用 PowerShell：

```powershell
Invoke-RestMethod -Uri "http://localhost:3000/health"
Invoke-RestMethod -Uri "http://localhost:8000/health"
Invoke-RestMethod -Uri "http://localhost:3000/api/target-application"
```

预期结果：

- 后端健康检查返回 `status: ok`、`service: backend`、`targetApplication: FitnessAI`。
- AI 服务健康检查返回 `status: ok`、`service: ai-service`。
- 目标应用接口返回 FitnessAI 名称、说明和测试重点模块。

## 9. 已实现功能测试流程

### 9.0 通用 Prompt 模板（替代上传 Assignment2.md）

推荐在底部 Prompt 输入框粘贴以下内容。该输入框主要补充本次要分析的 FitnessAI 范围和测试重点；Assignment2.md 不上传给 LLM，作业中的必要测试设计要求已由 `main.py` 固定 Prompt 注入。

```text
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

### 9.1 前端完整手工测试流程

1. 打开 `http://localhost:5173`。
2. 确认页面顶部显示 `FitnessAI Test Design Workspace`。
3. 在目标应用卡片中确认核心模块包含：
   - 实时姿态分析
   - 状态机计数
   - 训练计划
   - 记录过滤
   - 仪表盘统计
4. 点击“填入示例”。
5. 确认“纯文本需求”和“CSV 需求”输入框被自动填充。
6. 在底部 Prompt 输入框粘贴 `9.0 通用 Prompt 模板` 的完整内容。不要上传 `Assignment2.md`。

7. 点击“发送”。
8. 预期页面状态显示“生成完成”。
9. 在结果区检查以下内容：
   - 质量评分不为空。
   - 结构化需求数量大于 0。
   - 覆盖项数量大于 0。
   - 风险条目数量大于 0。
   - 测试用例数量大于 0。
   - 设计方法数量至少为 3，离线 mock 场景下应覆盖 5 种方法。
10. 检查 Assignment2 符合性面板：
   - FR 1.0 输入/解析显示已覆盖。
   - FR 1.1 需求结构化显示已覆盖。
   - FR 2.0 风险分析与优先级显示已覆盖。
   - FR 3.0 黑盒测试设计显示已覆盖。
   - FR 6.0 输出与导出显示已覆盖。
   - Interactive Review 交互式审查显示已覆盖。
   - FR 4.0、FR 5.0、FR 7.0 在 mock 场景下应显示已覆盖。
11. 展开“结构化产物与风险信息”，找到 `riskItems`，把某个风险项的 `priority` 从 `medium` 改为 `high`。
12. 点击“应用修改”。
13. 预期页面状态显示“已应用审查修改”，说明交互式审查能力可用。
14. 点击“导出 Markdown”“导出 JSON”“导出 CSV”。
15. 预期浏览器下载对应测试工件文件。
16. 点击右上角“历史记录”。
17. 点击最新记录的“查看详情”。
18. 预期历史结果可以回填到结果区。
19. 可点击历史记录中的“删除”，验证历史删除功能。

### 9.2 文件导入测试流程

文件导入用于上传 FitnessAI 目标应用资料。测试设计产物要求已经整理在 `9.0 通用 Prompt 模板` 中，应粘贴到 Prompt 输入框。

新建一个本地文本文件，例如 `FITNESSAI_LLM_CONTEXT.md`，内容如下：

```text
FitnessAI 文件导入测试需求：
1. 姿态分析接口必须校验 exerciseType 是否属于 SQUAT、PUSHUP、PLANK、JUMPING_JACK。
2. landmarks 数组理论长度为 33，长度为 32 或 34 时应返回可解释错误。
3. 深蹲计数必须经历 UP、DESCENDING、DOWN、ASCENDING、UP 完整状态循环。
4. count < 3 且 durationSeconds < 30 的训练记录不应写入历史记录。
5. 仪表盘应根据体重、运动类型和运动时长估算卡路里。
```

测试步骤：

1. 点击页面底部“导入文件”。
2. 选择 `FITNESSAI_LLM_CONTEXT.md`。
3. 底部 Prompt 输入框粘贴 `9.0 通用 Prompt 模板` 的完整内容，并在末尾追加：

```text
请基于上传的 FitnessAI 需求文件生成测试设计，重点覆盖等价类划分、边界值分析、决策表和状态迁移测试，并输出风险优先级。
```

4. 点击“发送”。
5. 预期生成结果包含文件中的五类需求，并在测试用例中体现 `exerciseType`、`landmarks.length`、状态机和记录过滤。

### 9.3 纯文本输入测试内容

如果不使用“填入示例”，可以手动在“纯文本需求”输入框粘贴：

```text
FitnessAI 是一个智能健身辅助系统。系统通过 MediaPipe Pose 提供 33 个身体关键点，后端根据 exerciseType 分析姿态并返回 count、score、feedback、state 和 angle。支持 SQUAT、PUSHUP、PLANK、JUMPING_JACK 四种运动。深蹲和俯卧撑需要状态机计数，只有完整动作循环才增加计数。训练记录保存时，count 小于 3 且 durationSeconds 小于 30 的记录应被过滤。仪表盘需要展示今日统计、历史趋势、运动分布和卡路里估算。
```

底部 Prompt 输入：

```text
请基于上述 FitnessAI 需求生成测试设计工件：
1. 重点覆盖姿态分析、运动类型校验、landmarks 长度边界、状态机计数、记录过滤和仪表盘统计。
2. 对状态机计数给出状态模型和可验证 oracle。
3. 不要把 AutoTestDesign 的输入、导出、历史记录等工具能力当作 FitnessAI 需求。
4. 沿用系统固定的结构化 JSON 输出格式。
```

预期结果：

- `requirementsStructured` 中出现姿态分析、状态机计数、记录过滤、仪表盘统计。
- `riskItems` 中姿态分析和状态机计数应为高风险。
- `testcases` 至少包含 EP、BVA、DecisionTable 三类黑盒测试方法。

### 9.4 CSV 输入测试内容

在“CSV 需求”输入框粘贴：

```csv
id,feature,input,condition,expected
REQ-POSE-001,姿态分析,exerciseType+landmarks,exerciseType合法且landmarks长度为33,返回count/score/feedback/state/angle
REQ-POSE-002,状态机计数,连续帧序列,完成UP-DESCENDING-DOWN-ASCENDING-UP循环,count增加1
REQ-POSE-003,非法状态迁移,连续帧序列,UP-DESCENDING-UP短循环,count不增加
REQ-REC-001,记录过滤,count+durationSeconds,count小于3且durationSeconds小于30,记录不入库
REQ-DASH-001,仪表盘统计,weight+exerciseType+durationSeconds,存在有效训练记录,更新趋势图和卡路里统计
```

底部 Prompt 输入：

```text
请基于上述 FitnessAI CSV 需求生成测试设计工件：
1. CSV 每一行都应转化为一条结构化需求，识别 id、feature、input、condition、expected。
2. 优先覆盖高风险姿态分析与状态机需求，对状态机需求使用状态迁移测试。
3. 记录过滤需求应形成决策表测试，仪表盘需求应给出卡路里估算 oracle。
4. 输出需求-覆盖项-测试用例的 traceability，便于测试人员交互式审查和修改。
5. 不要把 AutoTestDesign 的文件导入或导出功能当作 FitnessAI 被测需求；沿用系统固定的结构化 JSON 输出格式。
```

预期结果：

- 结构化需求 ID 应对应 `REQ-POSE-001`、`REQ-POSE-002`、`REQ-POSE-003`、`REQ-REC-001`、`REQ-DASH-001`。
- 边界值测试应包含 landmarks 数量 32、33、34。
- 决策表测试应包含记录过滤的有效/无效组合。

### 9.5 后端接口测试流程

使用 PowerShell 直接调用生成接口：

```powershell
$body = @{
  sourceType = "requirements"
  content = @"
目标应用: FitnessAI 智能健身辅助系统
需求:
1. /api/analytics/pose 接收 exerciseType 和 33 个 landmarks。
2. exerciseType 支持 SQUAT、PUSHUP、PLANK、JUMPING_JACK。
3. landmarks 长度为 33 时正常分析，长度为 32 或 34 时应返回错误。
4. 深蹲计数必须完成 UP-DESCENDING-DOWN-ASCENDING-UP 状态循环。
5. count < 3 且 durationSeconds < 30 的记录应被过滤。
"@
  promptMode = "custom"
  customPrompt = @"
请基于 content 中的 FitnessAI 目标应用需求生成测试设计工件。不要把 Assignment2.md 当作上传文件，也不要分析 AutoTestDesign 工具本身。

请重点覆盖 /api/analytics/pose、exerciseType 合法性、landmarks 长度 32/33/34、深蹲状态机完整循环、记录过滤决策表和可验证 oracle。
作业要求和 JSON schema 已由 main.py 固定 Prompt 补充，请沿用该结构化输出格式，并输出 coverageItems、riskItems、testcases、traceability，方便测试人员后续审查和修改。
"@
  documents = @()
  testTechnique = "black-box"
  includeWhitebox = $true
  includeOracle = $true
  includeOptimization = $true
  whiteboxDescription = "深蹲状态机: UP -> DESCENDING -> DOWN -> ASCENDING -> UP，完整循环后计数加1，进入COOLDOWN防重复计数。"
  coverageCriterion = "all-states"
} | ConvertTo-Json -Depth 8

Invoke-RestMethod `
  -Uri "http://localhost:3000/api/testcases/generate" `
  -Method Post `
  -ContentType "application/json" `
  -Body $body
```

预期返回字段：

- `message`: `AutoTestDesign artifacts generated`
- `technique`: `black-box`
- `quality.caseCount`: 大于 0
- `quality.coveredMethods`: 包含多种黑盒方法
- `assignmentCompliance.targetApplication.name`: `FitnessAI`
- `assignmentCompliance.requiredScore`: 接近或等于 `1`
- `artifacts.requirementsStructured`: 非空数组
- `artifacts.riskItems`: 非空数组
- `artifacts.stateModel`: 非空对象
- `data.testcases`: 非空数组

### 9.6 历史记录与导出接口测试

查询历史记录：

```powershell
Invoke-RestMethod -Uri "http://localhost:3000/api/history?limit=5"
```

预期：

- 返回 `records` 数组。
- 最新记录包含 `sourceType`、`modelName`、`quality`、`generatedCases`、`riskItems`、`traceability`。

导出 JSON：

```powershell
Invoke-RestMethod -Uri "http://localhost:3000/api/export?format=json&limit=10"
```

导出 CSV 到文件：

```powershell
Invoke-WebRequest `
  -Uri "http://localhost:3000/api/export?format=csv&limit=10" `
  -OutFile "autotestdesign-export.csv"
```

预期：

- JSON 导出包含历史记录列表。
- CSV 文件包含 `id`、`sourceType`、`technique`、`modelName`、`promptVersion`、`qualityScore`、`tokensEstimate`、`createdAt` 等列。

### 9.7 前端构建和语法验证

前端构建：

```powershell
cd "c:\Users\Project\frontend"
npm run build
```

后端语法检查：

```powershell
cd "c:\Users\Project\backend"
node --check src/index.js
```

AI 服务语法检查：

```powershell
cd "c:\Users\Project\ai-service"
python -m py_compile app/main.py
```

预期：

- `npm run build` 显示 Vite build 成功。
- `node --check` 无输出且退出码为 0。
- `python -m py_compile` 无输出且退出码为 0。

## 10. 演示验收检查清单

- Docker 容器已重新构建并启动。
- 前端可访问 `http://localhost:5173`。
- 后端健康检查通过。
- AI 服务健康检查通过。
- 可以通过“填入示例”生成 FitnessAI 测试设计。
- 可以使用文件、纯文本、CSV 三种输入方式。
- 文件上传时不上传 `Assignment2.md`，而是在 Prompt 输入框填写 `9.0 通用 Prompt 模板` 中的作业必要要求。
- 结果中能看到结构化需求、风险评分、黑盒测试用例、状态模型、测试预言、套件优化和追溯关系。
- 可以手动修改 JSON 审查内容并应用。
- 可以导出 Markdown、JSON、CSV。
- 历史记录可以查看、回填和删除。
