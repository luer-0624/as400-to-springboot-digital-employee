# AS400 → Spring Boot AI 数字员工系统设计方案

**版本**：V1.0（方案评审稿）
**日期**：2026-10-25
**状态**：待评审，尚未实施

**目标**：在现有 Claude Code 与项目级 `.claude/skills/` 能力基础上，构建一个能够后台无人值守运行的迁移数字员工，先完成「单个 AS400 画面及其关联部品」的端到端迁移。

---

## 0. 执行摘要

本方案不建议重新开发一套大而全的 Agent 平台，也不建议继续无止境地增加 Skills。核心工作是把现有 Skills 变成**可编排、可验证、可恢复、可审计**的自动化迁移流程。

第一版目标不是「整个 AS400 系统完全无人干预」，而是让一个有代表性的画面迁移任务从输入到交付形成可靠闭环：

1. 接收迁移工单和输入清单。
2. 校验源代码、画面定义、部品、表结构、依赖和验收要求是否齐全。
3. 调用 Claude Code 与已有 Skills 分析旧系统。
4. 生成可追溯的业务规则、依赖图和迁移计划。
5. 在独立 Git worktree 中生成 Spring Boot 代码。
6. 执行构建、静态检查、单元测试、集成测试和必要的业务对照测试。
7. 对允许自动修复的问题进行有限重试。
8. 由独立检查步骤核验验收证据。
9. 生成报告、保存日志和 Git diff。
10. 完成后通知用户；遇到不确定业务规则、权限问题或重试上限时，进入 BLOCKED 或 FAILED，而不是假装成功。

**基本原则**：大模型负责理解、规划和处理不确定性；确定性程序负责状态、权限、执行、重试、验收门禁和证据保存。

> 本方案中的目录结构、状态名、文件名和技术选型均为建议目标，不代表现有仓库已经具备这些内容。实施前必须先审计当前仓库，增量复用已有 Skills、脚本和规范。

---

## 1. 项目背景与目标边界

### 1.1 当前基础

项目已有若干面向 AS400 / RPG → Spring Boot 迁移的开发 Skills，能力可能涉及：

- RPG 源代码分析；
- DDS 画面定义和关联部品分析；
- 程序调用关系、业务规则和数据流提取；
- 逻辑文件 / 物理文件的关系分析；
- Java / Spring Boot 代码生成；
- SQL、Mapper、DTO、Controller、Service 等代码处理；
- 代码检查、重构和测试生成；
- 测试失败分析及修复。

> 第一阶段必须通过实际文件审计确认每个 Skill 的真实名称、输入输出、可调用方式、依赖和成熟度，不能仅凭描述推定它们都已实现。

### 1.2 第一版目标

- **运行方式**：后台无人值守，任务完成后查看报告。
- **自动化范围**：单个画面及其关联部品的端到端迁移。
- **AI 执行核心**：Claude Code，复用项目级 `.claude/skills/`。
- **首选调度方式**：轻量 Python 服务 / CLI + SQLite。
- **执行隔离**：独立 Git worktree。
- **第一版限制**：单机、低并发或串行执行，先可靠后扩展。
- **交付**：迁移代码、测试证据、映射关系、风险清单和完整运行记录。

### 1.3 第一版不做的事

- 不直接迁移整个 AS400 系统。
- 不允许 AI 直接操作生产数据库或生产环境。
- 不默认自动合并到主分支或自动发布。
- 不先建设复杂的 Web 管理平台。
- 不为了「多 Agent」而把每个 Skill 都拆成一个 Agent。
- 不一开始就引入 Kubernetes、复杂消息队列或多模型路由。
- 不以「模型说成功」作为任务成功的依据。
- 不承诺所有业务逻辑都能在没有参考数据和验收标准的情况下自动证明正确。

---

## 2. 设计原则

### 2.1 可验证优先

任何关键阶段都必须产生可检查的产物：

- **分析阶段**：业务规则、源文件清单、依赖图、未确认问题；
- **规划阶段**：任务清单、步骤依赖、验收标准；
- **生成阶段**：代码差异、生成文件清单；
- **测试阶段**：真实命令、退出码、测试报告；
- **验收阶段**：通过 / 不通过的具体证据。

自然语言总结可用于解释，但**不能替代机器可验证的结果**。

### 2.2 失败关闭（Fail Closed）

当输入缺失、业务规则存在关键歧义、测试不可执行、权限不足或证据不完整时，系统必须停止或标记为阻塞。不能为了让任务「完成」而猜测业务规则、跳过失败测试或伪造结果。

### 2.3 小步可恢复

每个阶段应有清晰的输入、输出、退出码和状态更新。进程被杀死、电脑重启或 Claude 调用失败后，可以从最后一个有效检查点恢复，而不是无条件从头执行或重复覆盖代码。

### 2.4 可追溯

每个迁移产物都要能够追溯到：

- 原始文件及其哈希；
- 使用的 Skill / Agent 版本；
- 迁移计划版本；
- 模型调用的时间、参数摘要和结果；
- 执行命令与退出码；
- Git commit / diff；
- 测试结果与验收结论。

日志中不得泄露 API Key、访问令牌、密码或未脱敏的敏感数据。

### 2.5 权限最小化

调度器和 Claude 只能访问明确授权的工作目录和命令。生产写入、正式数据库结构变更、发布、删除关键文件等高风险动作必须保留人工审批。

### 2.6 先串行，再并行

第一版先实现单任务串行闭环。只有在状态管理、工作区隔离、资源预算和恢复机制稳定后，才增加并行任务。

---

## 3. 总体架构

```
迁移工单 / CLI 输入
        |
        v
输入校验与任务创建
        |
        v
Orchestrator（确定性编排器）
        |
        +--> SQLite：任务状态、检查点、重试次数、事件
        |
        +--> Workspace Manager：独立 Git worktree
        |
        +--> Claude Runner：受控调用 Claude Code CLI / Agent SDK
        |       |
        |       +--> AS400 分析 Skill / Agent
        |       +--> 迁移计划 Skill
        |       +--> Spring Boot 代码生成 Skill / Agent
        |       +--> 测试生成与分析 Skill / Agent
        |       +--> 独立审查 Skill / Agent
        |
        +--> Deterministic Executor：构建、测试、静态检查
        |
        +--> Acceptance Gate：按规则核验真实证据
        |
        +--> Report Generator：Markdown + JSON
        |
        +--> Notification：完成、阻塞或失败通知
```

### 3.1 职责划分

| 角色 | 职责 |
|---|---|
| **Claude Code / 大模型** | 理解 RPG、DDS、业务语义；调用 Skills；生成迁移计划与代码；分析测试失败并提出修复；解释业务差异和风险 |
| **Python Orchestrator** | 管理任务状态；决定当前允许执行的阶段；校验输入输出文件；控制超时、重试次数和预算；启动和终止子进程；管理 worktree；记录事件和检查点；根据实际命令退出码决定是否继续 |
| **确定性验证脚本** | 执行 Maven/Gradle 构建；执行测试；读取测试报告；检查必要产物是否存在；检查高风险项；计算文件哈希和保存 Git diff；生成机器可判断的验收结果 |
| **人工** | 提供无法从源码可靠推断的业务规则；审批高风险动作；处理超过自动修复预算的失败；业务验收阶段确认例外 |

### 3.2 为什么不把所有工作都交给一个 Agent

单一长提示词容易出现上下文膨胀、阶段边界模糊、错误累积和「自己生成、自己宣称正确」的问题。第一版由 Python 按阶段启动不同的 Claude Code 调用，每个阶段读取明确的输入文件并写出明确的输出文件。这样更容易调试、复跑和审计。

---

## 4. 技术选型

| 模块 | 第一版建议 | 说明 |
|---|---|---|
| AI 执行 | Claude Code CLI 非交互模式 | 先复用现有工作流；需在目标环境验证 CLI 参数、认证和权限行为 |
| 未来 SDK 化 | Claude Agent SDK | 当需要更细粒度控制工具、会话和事件时评估 |
| Skills | 项目级 `.claude/skills/` | 保留现有资产，统一接口和输出协议 |
| Agent 定义 | `.claude/agents/` | 仅在确有角色隔离价值时添加 |
| 编排器 | Python 3.12+ | 适合子进程、文件、状态机和测试自动化 |
| 状态库 | SQLite | 单机第一版；需启用事务和适当备份 |
| 输入/输出协议 | JSON Schema + Markdown | JSON 用于机器校验，Markdown 用于人读 |
| 代码隔离 | Git worktree | 每个任务独立分支和工作目录 |
| 构建与测试 | 项目已有 Maven/Gradle Wrapper、JUnit | 不要强行统一到尚未审计的工具 |
| 日志 | 结构化 JSONL + 轮转文本日志 | 避免只有终端输出、无法恢复 |
| 报告 | Markdown + JSON | 人类查看和程序集成兼顾 |
| 后台进程 | WSL2 systemd 用户服务或 tmux 起步 | 需验证 WSL2 的休眠、重启和进程持续性 |
| 通知 | 先写入报告目录 / 控制台；之后接消息渠道 | 不应为通知功能阻塞主流程 |

### 4.1 Claude Runner 封装

```python
class ClaudeRunner:
    def run_stage(
        self,
        task_id: str,
        stage: str,
        prompt_file: str,
        working_dir: str,
        timeout_seconds: int,
    ) -> "StageResult": ...
```

```json
{
  "exit_code": 0,
  "stdout_path": "logs/stage.stdout.log",
  "stderr_path": "logs/stderr.log",
  "result_path": "artifacts/stage-result.json",
  "timed_out": false
}
```

不能仅凭进程退出码为 0 就认定迁移阶段成功，还必须验证输出文件、结构和阶段验收条件。

### 4.2 是否立即使用 Agent SDK

第一版不强制。只有出现以下需求时再评估：需要精细控制工具权限；需要消费结构化事件流；需要管理多轮会话与恢复；CLI 的输出协议不足以支持可靠编排；需要更深入地集成子代理或自定义工具。

---

## 5. 仓库目录设计

```
project-root/
├── CLAUDE.md
├── .claude/
│   ├── settings.json
│   ├── skills/
│   │   ├── migrate-as400/
│   │   │   ├── SKILL.md
│   │   │   ├── templates/
│   │   │   └── scripts/
│   │   ├── existing-as400-analysis/
│   │   ├── existing-code-generation/
│   │   ├── existing-test-generation/
│   │   └── existing-review/
│   ├── agents/
│   │   ├── as400-analyst.md
│   │   ├── migration-developer.md
│   │   ├── test-engineer.md
│   │   └── migration-reviewer.md
│   └── hooks/
│       ├── validate-workspace.sh
│       └── record-event.py
├── digital-employee/
│   ├── pyproject.toml
│   ├── README.md
│   ├── config/migration-policy.yaml
│   ├── schemas/
│   │   ├── migration-request.schema.json
│   │   ├── analysis-result.schema.json
│   │   ├── migration-plan.schema.json
│   │   └── stage-result.schema.json
│   ├── src/digital_employee/
│   │   ├── cli.py
│   │   ├── orchestrator.py
│   │   ├── state_machine.py
│   │   ├── models.py
│   │   ├── database.py
│   │   ├── claude_runner.py
│   │   ├── workspace_manager.py
│   │   ├── command_runner.py
│   │   ├── validators.py
│   │   ├── retry_policy.py
│   │   ├── report_generator.py
│   │   └── notifications.py
│   └── tests/
├── migration-inputs/
│   └── example-screen/
│       ├── source-manifest.yaml
│       ├── RPG/
│       ├── DDS/
│       ├── database-metadata/
│       └── reference-data/
├── migration-runs/
│   └── .gitignore
└── docs/digital-employee/
    ├── ARCHITECTURE.md
    ├── SKILL_INVENTORY.md
    ├── STATE_MACHINE.md
    ├── ACCEPTANCE_CRITERIA.md
    ├── SECURITY.md
    └── OPERATIONS.md
```

---

## 6. 迁移工单与输入规范

### 6.1 工单输入示例

```yaml
schema_version: "1.0"
task_id: "MIG-2026-0001"
target:
  screen_id: "EXAMPLE_SCREEN"
  description: "示例画面：根据产品 ID 与用户 ID 查询统计数据"
source:
  root: "migration-inputs/example-screen"
  rpg_files:
    - "RPG/EXAMPLE.rpgle"
  dds_files:
    - "DDS/EXAMPLE.dspf"
  related_programs: []
  database_metadata:
    - "database-metadata/tables.yaml"
dependencies:
  required_components: []
  known_logical_files: []
  known_physical_files: []
destination:
  project_path: "../target-springboot-project"
  package_root: "com.example.migration"
acceptance:
  build_required: true
  unit_tests_required: true
  integration_tests_required: true
  business_rule_evidence_required: true
constraints:
  allow_production_access: false
  allow_database_schema_change: false
  allow_auto_merge: false
  max_repair_attempts: 3
```

### 6.2 输入完整性检查

至少检查：所有声明的源文件存在；文件在允许的根目录内；缺少被调用程序/部品/表定义；目标 Spring Boot 项目可定位；目标分支和工作区干净；构建工具和测试依赖可用；业务规则和参考数据足够验收；无凭证敏感信息；无编码/字符集问题。

缺失关键业务依赖进入 BLOCKED，不要让模型自行补全不存在的事实。

---

## 7. Skill 设计与统一接口

### 7.1 先建 SKILL_INVENTORY.md

为每个 Skill 记录：名称/路径、主要职责、输入、输出、前置条件、执行方式、可验证性、风险、成熟度。

### 7.2 统一输出约定

每个关键阶段输出机器可读结果：

```json
{
  "schema_version": "1.0",
  "task_id": "MIG-2026-0001",
  "stage": "analysis",
  "status": "SUCCEEDED",
  "artifacts": [{ "path": "artifacts/business-rules.md", "sha256": "computed-at-runtime" }],
  "issues": [],
  "unresolved_questions": [],
  "warnings": []
}
```

哈希必须由程序计算，不能由模型填写。状态由验证器校验后确认，不能直接相信模型输出的 SUCCEEDED。

### 7.3 推荐 Skill 职责

- **AS400 分析 Skill**：读 RPG/DDS/表结构；提取调用关系、字段、条件分支、业务规则；区分源码事实、推断、未知。
- **迁移规划 Skill**：基于分析产物生成有序任务计划；对未知业务规则设阻塞点；不直接改目标代码。
- **代码生成 Skill**：按项目规范生成代码；输出修改文件清单和映射；不修改范围外文件；不改 Schema。
- **测试 Skill**：生成单元/集成/对照测试；说明覆盖范围；不得只生成不执行。
- **审查 Skill**：对照源码和业务规则检查目标代码；检查空值、精度、日期、事务、排序、异常语义；不直接改代码。

---

## 8. 单个画面端到端工作流

| 阶段 | 动作 | 关键输出 |
|---|---|---|
| 0 创建任务 | 工单 Schema 校验，生成 task_id，保存工单快照哈希 | tasks 表记录 |
| 1 准备工作区 | 建分支 `migration/MIG-xxx`、建 worktree、记录起始 commit、确认磁盘/工具 | worktree 路径、基线 SHA |
| 2 分析 AS400 | RPG 控制流、DDS 字段、关联部品、逻辑/物理文件关系、字段类型、业务规则、未知项 | source-inventory.json、dependency-graph.json、business-rules.md、data-mapping.json |
| 3 生成迁移计划 | 有序任务清单，每项有依赖/目标文件/验收条件/风险 | migration-plan.json |
| 4 生成代码 | 读现有项目规范；新增 DTO/Mapper/Service/Controller/异常；保存 Git diff；检查是否超范围 | 代码变更、diff.patch |
| 5 执行构建测试 | 静态检查 → 编译 → 单测 → 集成 → SQL 验证 → 业务对照 → 安全扫描 | 测试报告、退出码 |
| 6 失败分类与有限修复 | 编译错误/测试失败/SQL错误/业务不一致/缺依赖/环境错误/权限错误/未知；分类处理 | retry 记录 |
| 7 独立审查验收 | 新上下文审查；构建真成功、测试真执行、关键规则有覆盖、高风险为零、变更在范围内 | 审查报告 |
| 8 交付清理 | 生成报告，保留 worktree 或提交记录；不自动合并 | report.md/json |

第一版每个任务最多 3 次自动修复尝试，并设置总运行时间、模型调用次数和成本预算。

---

## 9. 状态机

```
PENDING → VALIDATING_INPUT → PREPARING_WORKSPACE → ANALYZING → PLANNING
       → MIGRATING → BUILDING → TESTING → FIXING ↺
       → REVIEWING → VERIFYING → COMPLETED

任意阶段异常 → BLOCKED（缺信息/等人工）/ FAILED（预算耗尽）/ CANCELLED
崩溃恢复 → RECOVERING → 回到最近有效检查点
```

**状态规则**：

- 只有满足前置条件才允许进入下一状态。
- 每次状态变更写入数据库事件表。
- 每个阶段有超时和明确的失败出口。
- `COMPLETED` 只能由验收逻辑设置。
- `BLOCKED` 等人工，不等同执行失败。
- 重启恢复时检查阶段产物哈希和实际工作区状态，不能仅依赖数据库里最后记录的状态。

**检查点保存**：输入文件哈希、提示词/配置版本、起止时间、退出码、超时信息、输出产物清单及哈希、Git HEAD、错误分类、重试次数、当前状态、下一步建议。

---

## 10. SQLite 数据模型

**migration_tasks**：task_id(PK)、status、request_json、source_hash、workspace_path、branch_name、created_at/updated_at、started_at/finished_at、failure_code、report_path。

**task_events**：event_id、task_id、event_type、from_status、to_status、timestamp、details_json、correlation_id。

**stage_runs**：run_id、task_id、stage_name、attempt_number、started_at/finished_at、exit_code、timed_out、stdout_path/stderr_path、result_path、input_hash/output_hash。

**artifacts**：artifact_id、task_id、artifact_type、path、sha256、created_at、sensitivity_level。

---

## 11. 自动化验收标准

### 11.1 不能仅依赖模型声明

「代码看起来正确」「测试应该通过」「已经完成所有修改」「没有发现问题」——都不能单独作为成功证据。

### 11.2 验收矩阵

| 类别 | 通过条件 | 证据 |
|---|---|---|
| 输入 | 所有必需输入存在且路径合法 | 输入校验结果 |
| 分析 | 关键依赖已识别，未知项明确列出 | 分析 JSON / 依赖图 |
| 业务规则 | 关键规则可追溯到源码或权威资料 | 规则清单及来源 |
| 代码生成 | 目标代码在允许范围内 | Git diff / 文件清单 |
| 构建 | 当前代码真实构建成功 | 命令退出码与构建日志 |
| 单元测试 | 必需测试通过 | 测试报告 |
| 集成测试 | 必需集成测试通过 | 测试报告与环境记录 |
| 业务对照 | 关键场景结果符合预期 | 对照测试结果 |
| 安全 | 无未解决的高风险问题 | 静态检查和审查记录 |
| 完整性 | 报告与产物对应同一代码版本 | Git HEAD、哈希 |
| 审计 | 关键步骤有记录 | 事件日志 |

### 11.3 业务对照基准用例

正常输入；不存在的产品/用户 ID；空值和边界值；最大长度/精度/小数位；日期边界；多条匹配；排序汇总；异常分支；旧系统已知特殊规则。

无法访问旧系统时记录验证替代方案和剩余风险，不能把缺少对照验证描述为「完全等价」。

---

## 12. 安全与治理

### 12.1 执行权限

默认禁止生产写；不允许模型自行授权；命令允许列表和路径范围；破坏性命令拒绝或人工审批；子进程设超时/资源上限/环境变量白名单；用参数数组启动进程避免 shell 拼接；对任务输入、源码注释、模型返回内容做不可信数据处理；不把密钥放进提示词/日志/报告。

### 12.2 Git 安全

独立分支/worktree；不覆盖未提交修改；不默认 `git reset --hard`、强删、自动合并；记录基线 commit；每阶段检查 diff；超范围修改自动阻塞；清理 worktree 前确认产物已归档。

### 12.3 数据安全

脱敏样本；限制源码外传；确认 Claude Code 认证方式和数据处理条款；日志对密码/令牌/敏感字段遮蔽；设置产物保留周期；不在 SQLite 存不必要原始业务数据；访问真实系统前先完成合规审查。

### 12.4 供应链和提示注入

源代码注释、文档或外部文件可能包含看似指令的文本。分析 Agent 应将其视为**被分析的数据**，而不是可信的系统指令。工具权限由系统配置控制，不能让源码内容改变权限策略。

---

## 13. 后台运行与运维

### 13.1 执行模型

一个轻量任务循环：从 SQLite 领 PENDING 任务 → 事务标记运行中 → 执行当前阶段 → 保存结果事件 → 进入下一状态 → 失败判断重试或停止 → 处理下一个。第一版只允许一个活动任务。

### 13.2 WSL2 注意事项

WSL2 持续性取决于宿主机/发行版/systemd；笔记本睡眠/Windows 重启/Docker Desktop 停止都会影响；不要把「后台进程已启动」等同于「永久运行」；重启后通过 SQLite 恢复任务；对运行超时/磁盘不足/Claude 未登录/网络失败给出清晰告警。

### 13.3 监控指标

任务成功率；每阶段耗时；自动修复次数；每类失败频率；每画面模型调用量和成本；人工阻塞原因；迁移后业务缺陷数；报告完整率；断点恢复成功率。

> 不要只优化「自动完成率」。如果通过降低测试标准提高完成率，实际价值反而下降。

---

## 14. 测试策略

- **单元测试**：状态转移合法性、重试策略、路径校验、Schema 校验、输出校验、验收门禁、报告生成、失败分类、日志脱敏。
- **集成测试**：用假 Claude Runner 模拟正常完成/非零退出/超时/JSON 错误/输出缺失/修复成功/达到最大重试/进程被杀/worktree 创建失败/输入被改/审查发现高风险。
- **端到端测试**：选一个真实画面人工监督跑通，保存基准输入和预期输出。

---

## 15. 分阶段实施计划

| 阶段 | 产物 | 验收 |
|---|---|---|
| A 只读审计 | CURRENT_STATE.md、SKILL_INVENTORY.md、GAP_ANALYSIS.md、ARCHITECTURE.md | 找出现有 Skills、确认试点画面、不修改生产代码 |
| B 统一迁移入口 | migrate-as400/SKILL.md、输入 Schema、工单模板、分析/计划格式 | 可校验工单、可生成计划、可识别缺失、复用现有 Skill |
| C 单画面端到端闭环 | 代码生成流程、构建测试执行器、基本验收、Markdown/JSON 报告 | 单画面从输入跑到报告；失败不宣称成功 |
| D 任务持久化与恢复 | SQLite、事件记录、重试策略、worktree 管理、超时预算 | 进程中断可恢复；不重复不安全动作；记录与工作区一致 |
| E 安全与独立审查 | 权限策略、文件范围检查、独立审查、高风险阻塞 | 超范围修改被阻止；缺关键规则被阻塞；高风险不能自动跳过 |
| F 无人值守运行 | 后台启动、日志轮转、通知、运维手册、故障恢复手册 | 不用盯终端；结束看报告；失败有告警和恢复路径 |
| G 扩展前评估 | 多任务并行、Web 台、PostgreSQL、工作流引擎、CI/CD、多模块 | 单画面闭环稳定后再评估 |

---

## 16. 风险清单

| 风险 | 影响 | 缓解 |
|---|---|---|
| Skill 输出不统一 | 编排失败 | JSON Schema + 结果校验 |
| 模型误解业务规则 | 代码语义错误 | 规则来源、基准用例、对照测试 |
| 自动修复循环 | 时间成本失控 | 重试上限、重复错误检测、预算 |
| 测试覆盖不足 | 错代码被误判成功 | 关键规则覆盖清单、测试证据 |
| 多 Agent 互相覆盖 | 代码冲突 | 第一版串行、独立 worktree |
| CLI 中断 | 任务丢失 | SQLite 检查点和恢复 |
| WSL2 停止 | 后台任务中断 | 启动检查、恢复机制、告警 |
| 权限过大 | 误删越权 | 最小权限、允许列表、人工审批 |
| 源码含敏感数据 | 数据泄露 | 脱敏、日志过滤、合规审查 |
| 目标项目规范不明 | 生成大量不兼容代码 | 先读现有项目并固定规范 |
| 输入依赖不完整 | 模型臆造 | 完整性校验、BLOCKED |
| 同一 Agent 自审 | 错误被重复确认 | 独立审查 + 确定性测试 |
| 报告与代码不匹配 | 交付不可追溯 | 绑定 commit、哈希、测试报告 |
| 过早建设平台 | 复杂度大、没闭环 | 单画面 MVP，分阶段扩展 |

---

## 17. 第一版完成定义（DoD）

已审计复用现有 Skills；具备工单格式；能识别缺失非法输入；能创建隔离工作区；能分析单画面依赖；能生成结构化计划；能生成 Spring Boot 代码；能真实执行构建和必需测试；能保存日志/退出码/diff；能有限次自动修复；能超预算停止；能识别关键业务规则不确定；能进程中断后恢复；能生成 Markdown+JSON 报告；COMPLETED 由确定性验收逻辑设置；不自动访问生产写接口或自动发布；在一个代表性画面上端到端验证；有运行和故障排查说明。

---

## 18. 下一步：第一轮只读审计提示词

```
请对当前仓库执行 AS400 → Spring Boot AI 数字员工项目的只读审计。
先检查 CLAUDE.md、.claude/skills/、现有 Agents、Hooks、脚本、构建配置、测试结构和 Git 状态。
不要假设任何 Skill 已经具备自动执行能力；逐个记录真实路径、输入、输出、依赖、文件修改行为和可验证方式。

先输出：
1. docs/digital-employee/CURRENT_STATE.md
2. docs/digital-employee/SKILL_INVENTORY.md
3. docs/digital-employee/GAP_ANALYSIS.md
4. docs/digital-employee/ARCHITECTURE.md

本轮不修改应用源码、不创建完整执行器、不删除或重命名现有 Skill、不执行生产操作。
对无法确认的事实标记为未知。
最后列出：推荐试点画面、最小闭环所需 Skill、必须补充能力、主要风险、下一阶段实施顺序。
```

---

## 19. 最终建议

顺序严格坚持：**先盘点能力 → 统一接口 → 打通单画面闭环 → 加入状态持久化 → 加入安全与恢复 → 后台无人值守 → 再扩展多画面**。

方案评审最值得先确定的三件事：

1. 选哪个真实画面做试点；
2. 现有 Skills 哪些已能稳定产出可验证文件；
3. 目标 Spring Boot 项目有哪些不可违反的架构和业务验收规范。
