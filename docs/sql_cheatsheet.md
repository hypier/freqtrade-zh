# SQL 助手

本页面提供一些关于查询你的 sqlite 数据库的帮助。

!!! Tip "其他数据库系统"
    若要使用 PostgreSQL 或 MariaDB 等其他数据库系统，可以使用相同的查询，但需要使用对应数据库系统的客户端。[点此](advanced-setup.md#use-a-different-database-system)了解如何使用 freqtrade 设置不同的数据库系统。

!!! Warning
    如果你不熟悉 SQL，运行数据库查询时应格外小心。  
    在执行任何查询之前，务必备份你的数据库。

## 安装 sqlite3

Sqlite3 是基于终端的 sqlite 应用程序。  
如果你更喜欢图形界面，也可以使用如 SqliteBrowser 之类的可视化数据库编辑器。

### Ubuntu/Debian 安装

```bash
sudo apt-get install sqlite3
```

### 通过 docker 使用 sqlite3

freqtrade 的 Docker 镜像中已包含 sqlite3，因此无需在主机系统上安装任何东西即可编辑数据库。

```bash
docker compose exec freqtrade /bin/bash
sqlite3 <database-file>.sqlite
```

## 打开数据库

```bash
sqlite3
.open <filepath>
```

## 表结构

### 列出所有表

```bash
.tables
```

### 显示表结构

```bash
.schema <table_name>
```

### 获取表中的所有交易记录

```sql
SELECT * FROM trades;
```

## 危险操作的查询

写入（修改、插入或删除）数据库的查询。  
通常情况下不需要手动执行这些操作，因为 freqtrade 会自动处理所有数据库操作——或通过 API 和 Telegram 命令提供。

!!! Warning
    在运行以下任何查询前，请确保已备份你的数据库。

!!! Danger
    在机器人连接到数据库时，**绝对不要**运行任何写入操作（`update`、`insert`、`delete`）。  
    这可能导致数据损坏，且很可能无法恢复。

### 解决在手动退出后交易仍然开启的问题

!!! Warning
    手动在交易所卖出某对资产不会被机器人检测到，机器人仍会尝试再次卖出。  
    尽可能地，应使用 `/forceexit <tradeid>` 来实现同样的效果。  
    强烈建议在手动操作前备份你的数据库文件。

!!! Note
    这在使用 `/forceexit` 后通常不需要，因为通过 force_exit 下单的仓位会在下一轮自动关闭。

```sql
UPDATE trades
SET is_open=0,
  close_date=<close_date>,
  close_rate=<close_rate>,
  close_profit = close_rate / open_rate - 1,
  close_profit_abs = (amount * <close_rate> * (1 - fee_close) - (amount * (open_rate * (1 - fee_open)))),
  exit_reason=<exit_reason>
WHERE id=<trade_ID_to_update>;
```

#### 示例

```sql
UPDATE trades
SET is_open=0,
  close_date='2020-06-20 03:08:45.103418',
  close_rate=0.19638016,
  close_profit=0.0496,
  close_profit_abs = (amount * 0.19638016 * (1 - fee_close) - (amount * (open_rate * (1 - fee_open)))),
  exit_reason='force_exit'  
WHERE id=31;
```

### 从数据库中移除交易

!!! Tip "使用 RPC 方法删除交易"
    建议通过 Telegram 或 REST API 使用 `/delete <tradeid>` 来删除交易。这是推荐的方法。

如果你仍希望直接从数据库中移除交易，可以使用以下查询。

!!! Danger
    一些系统（如 Ubuntu）在 sqlite3 打包时禁用了外键功能。  
    使用 sqlite 时，请确保在执行上述操作前运行 `PRAGMA foreign_keys = ON` 以启用外键支持。

```sql
DELETE FROM trades WHERE id = <tradeid>;

DELETE FROM trades WHERE id = 31;
```

!!! Warning
    该操作会将对应交易从数据库中完全删除。请确保 id 正确，无误后再执行，且**绝不**在没有 `where` 条件的情况下运行此类删除语句。