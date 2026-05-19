# FitnessAI 目标应用上下文

## 1. 目标应用概述

FitnessAI 是一个智能健身辅助系统，基于计算机视觉和人工智能提供实时健身训练指导。系统使用 MediaPipe Pose 检测人体 33 个身体关键点，并通过角度计算、状态机和规则判断实现运动计数、动作质量评分、姿势反馈和训练数据统计。

系统面向用户的核心价值：

- 实时检测用户身体关键点并分析运动姿态。
- 支持多种运动类型，包括深蹲、俯卧撑、平板支撑和开合跳。
- 使用状态机减少误计数和重复计数。
- 对动作质量给出实时反馈。
- 保存训练记录并展示历史趋势、卡路里消耗和运动类型分布。
- 根据用户档案估算 BMI 和运动消耗。

## 2. 核心功能需求

### 2.1 实时姿态分析

系统应接收前端 MediaPipe Pose 输出的身体关键点数据，并根据运动类型进行姿态分析。

关键输入：

- `exerciseType`：运动类型。
- `landmarks`：身体关键点数组，理论长度为 33。
- 每个关键点包含坐标和可见度信息，例如 `x`、`y`、`z`、`visibility`。

关键输出：

- `count`：当前运动计数。
- `score`：动作质量评分，范围为 0-100。
- `feedback`：动作纠正或鼓励提示。
- `state`：当前运动状态。
- `angle`：关键关节角度或分析指标。

关键规则：

- `landmarks` 长度为 33 时才是完整关键点输入。
- 关键点缺失、可见度不足或运动类型非法时，应返回可解释错误或低质量反馈。
- 姿态分析应避免服务端 5xx 错误。

### 2.2 支持的运动类型

FitnessAI 支持以下运动：

| 运动类型 | 英文标识 | 检测方式 | 主要检测逻辑 | 关键点 |
| --- | --- | --- | --- | --- |
| 深蹲 | `SQUAT` | 计数型 | 髋-膝-踝角度监测 | 臀部、膝盖、脚踝 |
| 俯卧撑 | `PUSHUP` | 计数型 | 肘部弯曲和肩部高度追踪 | 肩膀、肘部、手腕 |
| 平板支撑 | `PLANK` | 计时型 | 肩-髋-踝线性度分析 | 肩膀、臀部、脚踝 |
| 开合跳 | `JUMPING_JACK` | 计数型 | 垂直位移和手臂高度变化 | 手腕、肩膀、脚踝 |

运动类型测试应覆盖：

- 合法值：`SQUAT`、`PUSHUP`、`PLANK`、`JUMPING_JACK`。
- 非法值：空值、未知字符串、大小写错误、类型错误。
- 不同运动类型对相同关键点输入的处理差异。

### 2.3 状态机计数与防误检

系统使用有限状态机识别完整动作过程，避免短暂抖动、非法跃迁和重复计数。

典型计数状态：

- `UP`
- `DESCENDING`
- `DOWN`
- `ASCENDING`
- `COOLDOWN`

典型深蹲完整动作循环：

```text
UP -> DESCENDING -> DOWN -> ASCENDING -> UP -> COOLDOWN
```

关键规则：

- 只有完成完整动作循环后，计数才应增加 1。
- 非法短循环不应计数，例如 `UP -> DESCENDING -> UP`。
- 需要连续 2-3 帧确认状态变化，以减少抖动误判。
- 计数后进入约 10 帧冷却期，防止重复计数。
- 深蹲角度阈值约为 140 度。

### 2.4 训练模式

系统支持两类训练模式。

自由模式：

- 用户可自由训练。
- 无固定时间或次数限制。
- 用户可手动重置计数器。

计划模式：

- 系统根据训练难度生成结构化计划。
- 简单难度：深蹲 2 组 × 10 次，俯卧撑 1 组 × 8 次。
- 中等难度：深蹲 3 组 × 15 次，俯卧撑 2 组 × 12 次。
- 困难难度：深蹲 4 组 × 20 次，俯卧撑 3 组 × 15 次。
- 组间包含 30-60 秒休息。
- 用户可以跳过休息进入下一组。

计划模式测试应覆盖：

- 不同难度下的组数、次数和休息时间。
- 完成一组后的状态变化。
- 跳过休息后的下一组切换。
- 计划完成后的结束状态。

### 2.5 训练记录过滤与保存

系统会保存有效训练记录，并过滤无效记录以保证统计数据质量。

关键数据：

- `exerciseType`：运动类型。
- `count`：运动次数。
- `durationSeconds`：训练时长。
- `calories`：卡路里消耗。
- `createdAt`：记录时间。

过滤规则：

```text
如果 count < 3 且 durationSeconds < 30，则该记录视为无效记录，不应写入历史记录和统计数据。
否则，该记录可作为有效训练记录保存。
```

记录过滤测试应覆盖以下决策表：

| count < 3 | durationSeconds < 30 | 预期结果 |
| --- | --- | --- |
| true | true | 过滤，不入库 |
| true | false | 保存 |
| false | true | 保存 |
| false | false | 保存 |

边界值测试应覆盖：

- `count`：0、1、2、3、4。
- `durationSeconds`：0、29、30、31。

### 2.6 用户档案与卡路里估算

系统支持用户档案管理。

关键字段：

- 身高。
- 体重。
- 健身目标，例如减重、增肌。

系统可根据身高体重计算 BMI，并根据 MET 值估算卡路里消耗。

卡路里估算公式：

```text
calories = MET × weightKg × durationHours
```

已知 MET 示例：

- 深蹲：5.0 MET。
- 俯卧撑：8.0 MET。

测试应覆盖：

- 体重缺失、为 0、为负数、极大值。
- 时长为 0、边界值和长时间训练。
- 不同运动类型的 MET 差异。
- 仪表盘展示值是否与公式近似一致。

### 2.7 仪表盘与历史统计

系统提供训练数据分析和可视化统计。

主要统计项：

- 今日训练统计。
- 历史训练记录。
- 最近 30 天活动趋势。
- 卡路里消耗柱状图。
- 运动类型分布饼图。
- 个性化进度追踪。

测试应覆盖：

- 无训练记录时的空状态。
- 只有无效记录时的统计结果。
- 有多种运动类型记录时的分布统计。
- 跨日期记录的趋势统计。
- 保存新记录后仪表盘是否刷新。

## 3. 主要接口

### 3.1 基础接口

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/api` | API 状态检查 |
| `GET` | `/api/health` | 健康检查 |
| `GET` | `/api/system` | 系统信息 |
| `GET` | `/api/exercises` | 获取支持的运动类型 |
| `GET` | `/api/recommendations` | 获取个性化推荐 |

### 3.2 姿态分析接口

接口：

```text
POST /api/analytics/pose
```

请求示例：

```json
{
  "exerciseType": "SQUAT",
  "landmarks": [
    {
      "x": 0.5,
      "y": 0.3,
      "z": 0.1,
      "visibility": 0.9
    }
  ]
}
```

响应示例：

```json
{
  "count": 5,
  "score": 85,
  "feedback": "很好！下蹲姿势正确，请站起来完成动作",
  "state": "DOWN",
  "angle": 120.5
}
```

测试关注点：

- `exerciseType` 合法性。
- `landmarks` 数组长度边界：0、1、32、33、34。
- 关键点坐标范围。
- `visibility` 低于阈值时的反馈。
- 响应字段完整性。
- 状态机连续帧行为。

### 3.3 分析器状态重置接口

接口：

```text
POST /api/analyzer/reset/{exerciseType}
```

测试关注点：

- 重置合法运动类型。
- 重置非法运动类型。
- 重置后计数是否归零。
- 重置后状态是否回到初始状态。

### 3.4 用户与训练记录接口

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/api/user/{userId}/profile` | 获取用户档案 |
| `PUT` | `/api/user/{userId}/profile` | 更新用户档案 |
| `POST` | `/api/user/{userId}/records` | 保存运动记录 |
| `GET` | `/api/user/{userId}/records` | 获取历史记录 |
| `GET` | `/api/user/{userId}/stats/today` | 获取今日统计 |
| `GET` | `/api/user/{userId}/stats/today/{exerciseType}` | 获取今日特定运动统计 |
| `GET` | `/api/user/{userId}/dashboard` | 获取仪表盘数据 |

测试关注点：

- 用户 ID 不存在。
- 用户档案字段缺失或非法。
- 无效训练记录过滤。
- 保存记录后历史和统计是否一致。
- 今日统计按运动类型聚合是否正确。

## 3. 建议重点测试模块

建议将“姿态分析与状态机计数”作为详细测试设计模块，因为它同时具备高风险、复杂状态、边界值、异常输入和可验证输出。

该模块可覆盖：

- 等价类划分：合法/非法运动类型，完整/不完整关键点。
- 边界值分析：landmarks 数量 32、33、34；评分 0、100；角度阈值附近。
- 决策表：记录过滤规则、关键点可见度与错误反馈。
- 状态迁移测试：完整动作循环、非法短循环、冷却期重复输入。
- 白盒建模：运动计数状态机。
- 测试预言：计数变化、状态变化、错误响应、卡路里公式。

## 4. 初始风险清单

| 风险 ID | 风险描述 | 影响 | 可能性 | 优先级 | 测试关注点 |
| --- | --- | --- | --- | --- | --- |
| R-POSE-001 | landmarks 缺失或数量错误导致姿态分析失败 | 高 | 高 | high | landmarks 长度 32、33、34 |
| R-POSE-002 | 状态机非法跃迁导致误计数或漏计数 | 高 | 高 | high | 完整循环与短循环对比 |
| R-POSE-003 | 冷却期失效导致重复计数 | 高 | 中 | high | 计数后连续相同帧输入 |
| R-REC-001 | 无效训练记录未被过滤，污染统计 | 中 | 高 | high | count 与 duration 决策表 |
| R-DASH-001 | 仪表盘统计与历史记录不一致 | 中 | 中 | medium | 保存记录后统计刷新 |
| R-CAL-001 | 卡路里估算公式或 MET 使用错误 | 中 | 中 | medium | 不同体重、时长、运动类型 |
| R-PLAN-001 | 计划模式组数、次数或休息逻辑错误 | 中 | 中 | medium | easy、medium、hard 与 skipRest |
| R-UX-001 | 摄像头权限、低可见度或网络失败时反馈不清楚 | 中 | 中 | medium | 异常提示和恢复流程 |

## 5. 可用于测试设计的结构化需求

| 需求 ID | 功能 | 输入 | 条件/范围 | 预期动作 |
| --- | --- | --- | --- | --- |
| REQ-POSE-001 | 姿态分析 | `exerciseType`, `landmarks` | `exerciseType` 合法，`landmarks.length = 33` | 返回计数、评分、反馈、状态和角度 |
| REQ-POSE-002 | 运动类型校验 | `exerciseType` | 合法集合为 SQUAT/PUSHUP/PLANK/JUMPING_JACK | 合法值进入对应分析器，非法值返回错误 |
| REQ-POSE-003 | 关键点边界 | `landmarks` | 长度为 32、33、34 | 33 正常处理，其他值返回可解释错误 |
| REQ-STATE-001 | 深蹲状态机计数 | 连续帧序列 | 完整循环 UP-DESCENDING-DOWN-ASCENDING-UP | count 增加 1 |
| REQ-STATE-002 | 非法短循环处理 | 连续帧序列 | UP-DESCENDING-UP | count 不增加 |
| REQ-STATE-003 | 冷却期防重复计数 | 连续帧序列 | 计数后 10 帧内重复输入 | 不重复计数 |
| REQ-REC-001 | 训练记录过滤 | `count`, `durationSeconds` | `count < 3 && durationSeconds < 30` | 记录不入库 |
| REQ-REC-002 | 有效训练记录保存 | `count`, `durationSeconds` | `count >= 3` 或 `durationSeconds >= 30` | 记录保存并进入统计 |
| REQ-PLAN-001 | 训练计划生成 | `difficulty` | easy、medium、hard | 生成对应组数、次数和休息时间 |
| REQ-PLAN-002 | 跳过休息 | `skipRest` | 用户完成一组后选择跳过 | 进入下一组 |
| REQ-DASH-001 | 仪表盘统计 | 有效训练记录 | 存在历史记录 | 展示趋势、卡路里和运动分布 |
| REQ-CAL-001 | 卡路里估算 | `weightKg`, `durationHours`, `MET` | 输入合法 | calories = MET × weightKg × durationHours |

## 9. LLM 输出要求建议

上传本文档后，可在 Prompt 中要求 LLM 输出 JSON，并至少包含以下字段：

```json
{
  "inputVariables": [],
  "requirementsStructured": [],
  "coverageItems": [],
  "riskItems": [],
  "equivalencePartitions": [],
  "boundaryValues": [],
  "decisionTableRules": [],
  "stateModel": {},
  "testSuiteOptimization": {},
  "traceability": [],
  "testcases": [],
  "missingItems": [],
  "assumptions": []
}
```

并覆盖：

- 等价类划分。
- 边界值分析。
- 决策表。
- 状态迁移测试。
- 组合测试。
