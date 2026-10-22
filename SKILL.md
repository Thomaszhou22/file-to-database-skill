---
name: database-generator-skill
version: 1.0.0
description: "Use when the user wants to build a database from data files (CSV/Excel/JSON/JSONL/TSV), design a schema from raw data, or turn flat files into tables — AND when the user later talks to that database in natural language (add, query, update, delete records; 'how many...'; 'show me...'). Scans files, infers column types, detects candidate foreign keys, applies the methodology in references/methodology.md to produce DDL, ERD, seed scripts, and validation; runtime operations follow references/operations.md (intent → SQL → safe execution → natural-language report). Do NOT use for ORM code generation or pure database concept discussions."
---

# DB from Files

把数据文件（CSV / Excel / JSON / JSONL / TSV）直接变成一套经过验证的数据库：
**扫描 → 推断 → 建模 → DDL → 验证**，全程遵循 `references/methodology.md` 里的可插拔方法论。

## 启动流程

1. 读 `references/methodology.md`（方法论是本 Skill 的灵魂，必须先读）
2. 检测运行环境：
   - `python3 --version` ≥ 3.8 → 脚本引擎可用
   - 不可用 → 降级为纯 LLM 模式（agent 自己分析文件样本，精度略低，需告知用户）
3. 让用户提供文件路径（或目录），确认目标引擎（默认 SQLite；需要多用户并发/云端时 PostgreSQL/Supabase；支持 MySQL、MongoDB）

## 入口分流（非数据文件必读）

用户给的文件**不属于** CSV/Excel/JSON/JSONL/TSV（如 PDF 教材、PPTX、DOCX、图片）时，禁止直接进入建库流程，也禁止拒绝服务——按 `references/intake.md` 分流：

- 意图已明确（如"把这本书的题建库"）→ 走内容结构化路径：LLM 抽取成 JSONL 中间格式（带页码可核对）→ 抽样 10 条让用户确认 → 回到标准管道
- 意图不明 → 带选项问一次：①提取内容建库 ②仅阅读/总结 ③是否发错文件；**不代替用户猜任务**
- 文件类型决定能力边界，用户意图决定工作流

## 主流程（staged loop，每步留下可检查的产物）

### Step 1: 扫描画像（scripts/infer_schema.py）
对每个数据文件生成统计画像：
```bash
python3 scripts/infer_schema.py --input <file-or-dir> [--sample N] --json
```
产出：每列的类型推断（int/float/bool/date/uuid/text）、空值率、基数、唯一性、最长值、样本值。
**完成标准**：每个文件一份画像，agent 复核有歧义的列（混合类型、超长文本）。

### Step 2: 关系发现（scripts/detect_relations.py）
```bash
python3 scripts/detect_relations.py --profiles <profiles.json> --json
```
产出：主键候选、外键候选（名称匹配 + 值集包含检测，给出置信度）。
**完成标准**：每个外键候选都有置信度和证据；低置信度的列出来问用户，不要瞎猜。

### Step 3: 数据建模（按 methodology.md 执行）
- 实体识别：一个文件 → 一张表（宽表按方法论拆分规则决定是否范式化）
- 基数标注：1:1 / 1:N / N:M（N:M 需要显式中间表）
- 默认 3NF；只为方法论中点名的查询反范式化，且必须注明服务哪条查询
- 开放问题清单：方法论要求把 spec 没回答的问题列出来问用户，不许自行脑补

### Step 4: 产出交付物
按 `references/output-format.md` 的布局输出：
1. `schema.sql`（DDL：表、约束、索引，每个索引注明服务的查询）
2. `ERD`（Mermaid erDiagram）
3. `seed.py` 或 `seed.sql`（从源文件导入数据的脚本，含类型转换）
4. `migrations/`（可回滚迁移，expand-and-contract 模式）

### Step 5: 验证（有数据库就跑，没有就人工走查）
```bash
# 有本地 Postgres / Docker / Supabase branch 时：
psql -f schema.sql && python3 seed.py
# 每个约束一条反向测试：插入应被拒绝的行，断言失败
# EXPLAIN 关键查询，确认索引真的被用上
```
无数据库时：逐条走查方法论 Step 7 的 checklist，输出走查报告。

## 运行模式（建库之后）

库建好后进入**运行模式**：用户用自然语言对数据库增删改查，Agent 负责
**意图识别 → SQL 生成 → 安全执行 → 自然语言汇报**。完整规则见 `references/operations.md`（触发规则、SQL 生成规则、安全门槛、汇报模板）。要点：

- 用户消息含「实体+操作词 / 数据疑问 / 变更动词 / 隐式指代」即触发；纯概念讨论不触发
- UPDATE/DELETE 先跑同条件 SELECT 报影响行数，>5 行或无 WHERE 必须用户确认
- 报错翻译成人话（unique 冲突 → "已存在"；外键冲突 → "引用的记录不存在"）
- 查询结果先给结论再给表格；变更报告说明影响行数与可撤销性
- 每次变更后自动交付可视化包：库本体文件（SQLite 默认单文件）+ Mermaid ERD（含 mermaid.live 在线链接）+ snapshot.html 数据快照，规则见 `references/operations.md` 的「变更后可视化交付」节

## 工程化加固层（references/engineering.md）

建库与运行的基础流程之上，必读并执行 `references/engineering.md` 的四项：

1. **迁移账本**：`.applied.json` 防重复/乱序执行；checksum 变化的历史迁移必须停下报错；交付前实戰演练一次回滚
2. **数据质量门**：seed 后自动断言（行数、主键空值、外键悬空、值域漂移）；失败停止并给样本
3. **增量导入**：文件更新绝不重建库——画像 diff → 结构变化走新迁移 / 纯新增行走 UPSERT（主键冲突默认跳过并记录）
4. **权限与安全**：凭证只走环境变量、不入任何文件；SQLite 0600 / PG 最小权限角色 / Supabase RLS；PII 列标记+快照掩码；全部写操作记录 audit.log 随交付包发送

**诚实边界**（生成但不执行）：生产部署、监控告警、压测、容量规划、凭证托管。

## 失败与降级

| 情况 | 处理 |
|---|---|
| 文件太大无法全量扫描 | `--sample N` 抽样 + 告知用户置信度下降 |
| Python 不可用 | 纯 LLM 模式：读文件头 100 行手动画像 |
| 无法确定外键关系 | 列出候选问用户，绝不静默选择 |
| DDL 执行失败 | 打印完整错误 + 定位列，修复后重跑验证 |

## 环境感知

- 目标引擎从 `docker-compose.yml` / ORM 配置 / Supabase 项目推断；推断不出就问
- 用户已有迁移工具（Prisma/Drizzle/Alembic）→ 匹配其迁移文件格式，不另起炉灶
