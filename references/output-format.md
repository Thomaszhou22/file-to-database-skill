# 交付物输出格式

database-generator-skill 每次运行产出一套标准交付物，按此布局：

```
output/
├── schema.sql          # DDL（表、约束、索引）
├── erd.md              # Mermaid ERD
├── seed.py             # 从源文件导入数据（或 seed.sql）
├── migrations/
│   └── 001_init.up.sql / 001_init.down.sql
└── report.md           # 建模报告（见下）
```

## schema.sql 要求

- 文件头注释：生成时间、目标引擎、源文件清单
- 每张表一段，表注释写明来源文件
- 每个索引注明服务的查询：`-- serves Q3: SELECT ... WHERE user_id = ? ORDER BY created_at`
- 每个外键注明 ON DELETE 选择理由
- 反范式化的列必须有注释点名服务的查询

## erd.md 要求

Mermaid `erDiagram`，实体间标注基数与外键列。

## report.md 模板

```markdown
# 数据库建模报告

## 输入
- 源文件：…（行数、扫描抽样说明）

## 决策记录
| # | 决策 | 依据（方法论条款） |
|---|------|--------------------|

## 开放问题（已问用户 / 待确认）
- …

## 外键判定
（detect_relations.py 三档结果：adopt / ask_user / ignore）

## 验证结果
- 约束反向测试：X/X 通过
- EXPLAIN 索引命中：Q1 ✓ Q2 ✓ …
- 或：无数据库环境，已按 checklist 人工走查
```
