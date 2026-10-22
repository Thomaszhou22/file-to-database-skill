# 其他引擎的差异（PostgreSQL 之外的默认值）

| 关注点 | MySQL 8 | SQLite | MongoDB |
|---|---|---|---|
| 主键 | `BIGINT AUTO_INCREMENT` 或 `CHAR(36)` UUID | `INTEGER PRIMARY KEY AUTOINCREMENT` | `_id: ObjectId` |
| 字符串 | `VARCHAR(255)` 起步（MySQL 的 text 不能无默认值索引） | `TEXT` | string |
| 时间 | `DATETIME`（存 UTC，应用层转换） | `TEXT` ISO-8601 或 unix epoch int | date |
| 金额 | `DECIMAL(12,2)` | `NUMERIC` / 存整数的分 | 用 Decimal128，禁用 double |
| 布尔 | `TINYINT(1)`（MySQL 无真 boolean） | `INTEGER` 0/1 | bool |
| 枚举 | `ENUM(...)`（或 VARCHAR+CHECK） | `TEXT + CHECK` | 无 schema，靠应用层校验 |
| 外键 | InnoDB 支持，需显式声明 | 需 `PRAGMA foreign_keys=ON` | 手动维护（或 lookup pattern） |
| 索引外键列 | MySQL 自动给 FK 建索引 | 手动 | 手动 |
| 迁移工具约定 | 裸 SQL / Alembic | 裸 SQL / Drizzle | mongoose migrations |

## Supabase（= PostgreSQL + 约定）

- 用 Supabase migrations 目录管理 DDL，不用 dashboard 手点
- RLS：面向公网的项目，每张表默认 `ENABLE ROW LEVEL SECURITY`，策略显式写
- `auth.users` 是保留表，业务用户表用 `user_id uuid references auth.users(id)` 关联，主键不叫 id 时注意与 Supabase 客户端约定
