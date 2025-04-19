# 高级安装后任务

本文档介绍一些在机器人安装后可以执行的高级任务和配置选项，在某些环境下可能会有所帮助。

如果你不清楚这里提到的内容是什么意思，可能你其实不需要这些操作。

## 在同一台机器上运行多个Freqtrade实例

本节将向你展示如何在同一台设备上同时运行多个机器人。

### 需要考虑的事项

* 使用不同的数据库文件。
* 使用不同的Telegram机器人（需要多个不同的配置文件；仅在启用Telegram时适用）。
* 使用不同的端口（仅在启用Freqtrade REST API网络服务器时适用）。

### 不同的数据库文件

为了跟踪你的交易、利润等信息，freqtrade使用一个SQLite数据库，存储各种类型的信息，比如你过去的交易和你目前持有的仓位。这让你可以检测到你的盈利情况，更重要的是，即使机器人进程重启或意外终止，也能持续跟踪活动。

Freqtrade默认会为模拟交易（dry-run）和实盘交易使用不同的数据库文件（前提是配置中和命令行参数中都没有指定database-url）。
对于实盘模式，默认数据库为`tradesv3.sqlite`；模拟交易则为`tradesv3.dryrun.sqlite`。

指定这些文件路径的可选参数是`--db-url`，它需要一个有效的SQLAlchemy URL。
因此，当你只用配置和策略参数在Dry-run模式下启动机器人时，以下两个命令效果是相同的：

```bash
freqtrade trade -c MyConfig.json -s MyStrategy
# 等同于
freqtrade trade -c MyConfig.json -s MyStrategy --db-url sqlite:///tradesv3.dryrun.sqlite
```

这意味着如果你在两个不同的终端上运行trade命令，例如测试在USDT和BTC两种不同的交易场景，你必须使用不同的数据库。

如果你指定了一个不存在的数据库URL，freqtrade会自动创建一个同名的数据库文件。因此，假如你想用两个不同的stake货币（比如BTC和USDT）测试自定义策略，可以在两个终端中使用如下命令：

```bash
# 终端1：
freqtrade trade -c MyConfigBTC.json -s MyCustomStrategy --db-url sqlite:///user_data/tradesBTC.dryrun.sqlite
# 终端2：
freqtrade trade -c MyConfigUSDT.json -s MyCustomStrategy --db-url sqlite:///user_data/tradesUSDT.dryrun.sqlite
```

反过来，如果你想在生产环境中做相同的事情，也需要创建至少一个新的数据库（除了默认的那个），并指定“实盘”数据库的路径，例如：

```bash
# 终端1：
freqtrade trade -c MyConfigBTC.json -s MyCustomStrategy --db-url sqlite:///user_data/tradesBTC.live.sqlite
# 终端2：
freqtrade trade -c MyConfigUSDT.json -s MyCustomStrategy --db-url sqlite:///user_data/tradesUSDT.live.sqlite
```

关于如何手动进入或删除交易记录的更详细信息，请参考[SQL Cheatsheet](sql_cheatsheet.md)。

### 使用Docker运行多实例

想用Docker运行多个freqtrade实例，你需要编辑`docker-compose.yml`文件，将所有想要作为独立服务的实例添加进去。记住，你可以把配置拆分成多个文件，建议考虑模块化管理，这样如果要修改所有机器人的某个共通部分，只需修改一个配置文件即可。

示例配置如下：

```yaml
---
version: '3'
services:
  freqtrade1:
    image: freqtradeorg/freqtrade:stable
    # image: freqtradeorg/freqtrade:develop
    # 使用绘图镜像
    # image: freqtradeorg/freqtrade:develop_plot
    # 构建步骤——仅在需要额外依赖时使用
    # build:
    #   context: .
    #   dockerfile: "./docker/Dockerfile.custom"
    restart: always
    container_name: freqtrade1
    volumes:
      - "./user_data:/freqtrade/user_data"
    # 绑定接口端口（仅本地访问）
    # 请在启用前阅读 https://www.freqtrade.io/en/latest/rest-api/ 文档
    # 启用此项会暴露API端口
    ports:
     - "127.0.0.1:8080:8080"
    # 启动命令
    command: >
      trade
      --logfile /freqtrade/user_data/logs/freqtrade1.log
      --db-url sqlite:////freqtrade/user_data/tradesv3_freqtrade1.sqlite
      --config /freqtrade/user_data/config.json
      --config /freqtrade/user_data/config.freqtrade1.json
      --strategy SampleStrategy
  
  freqtrade2:
    image: freqtradeorg/freqtrade:stable
    # image: freqtradeorg/freqtrade:develop
    # 使用绘图镜像
    # image: freqtradeorg/freqtrade:develop_plot
    # 构建步骤——仅在需要额外依赖时使用
    # build:
    #   context: .
    #   dockerfile: "./docker/Dockerfile.custom"
    restart: always
    container_name: freqtrade2
    volumes:
      - "./user_data:/freqtrade/user_data"
    # 绑定接口端口（仅本地访问）
    # 请在启用前阅读 https://www.freqtrade.io/en/latest/rest-api/ 文档
    # 启用此项会暴露API端口
    ports:
      - "127.0.0.1:8081:8080"
    # 启动命令
    command: >
      trade
      --logfile /freqtrade/user_data/logs/freqtrade2.log
      --db-url sqlite:////freqtrade/user_data/tradesv3_freqtrade2.sqlite
      --config /freqtrade/user_data/config.json
      --config /freqtrade/user_data/config.freqtrade2.json
      --strategy SampleStrategy
```

你可以采用任何你喜欢的命名规则，freqtrade1和2仅为示例。注意，每个实例都需要使用不同的数据库文件、端口映射和Telegram配置。

## 使用其他数据库系统

Freqtrade基于SQLAlchemy，支持多种数据库系统，因此理论上支持多种数据库。
Freqtrade不依赖或额外安装数据库驱动程序，请参考[SQLAlchemy官方文档](https://docs.sqlalchemy.org/en/14/core/engines.html#database-urls)获取不同数据库系统的安装说明。

已测试并确认可以与Freqtrade配合使用的数据库包括：

* sqlite（默认）
* PostgreSQL
* MariaDB

!!! 警告
    使用以下任意数据库系统，意味着你应具备相应系统的管理能力。Freqtrade团队不会提供关于这些数据库系统的安装、维护（或备份）方面的支持。

### PostgreSQL

安装：
```bash
pip install psycopg2-binary
```

使用：
```bash
... --db-url postgresql+psycopg2://<用户名>:<密码>@localhost:5432/<数据库名>
```

Freqtrade在启动时会自动创建所需的表。

如果你运行多个实例，必须为每个实例设置不同的数据库，或者使用不同的用户/模式。

### MariaDB / MySQL

Freqtrade支持MariaDB，且依赖SQLAlchemy。

安装：
```bash
pip install pymysql
```

使用：
```bash
... --db-url mysql+pymysql://<用户名>:<密码>@localhost:3306/<数据库名>
```


## 将机器人配置为systemd服务运行

将`freqtrade.service`文件复制到你的systemd用户目录（通常是`~/.config/systemd/user`），并根据你的实际情况修改`WorkingDirectory`和`ExecStart`。

!!! 注意
    某些系统（如Raspbian）不会从用户目录加载服务单元文件。在这种情况下，将`freqtrade.service`复制到`/etc/systemd/user/`（需要超级用户权限）后再使用。

之后，你可以用以下命令启动守护进程：

```bash
systemctl --user start freqtrade
```

如果希望在用户注销后还能保持运行，需要启用`linger`：

```bash
sudo loginctl enable-linger "$USER"
```

如果你以服务方式运行机器人，可以利用systemd的服务管理功能，将其作为软件看门狗，监控freqtrade状态并在失败时自动重启。前提是，在配置中启用了`internals.sd_notify`参数或使用`--sd-notify`命令行参数，机器人会通过sd_notify协议向systemd发送心跳并报告状态（运行中、暂停或停止）。

`freqtrade.service.watchdog`文件包含了使用systemd作为看门狗的示例服务单元配置。

!!! 注意
    如果机器人在Docker容器中运行，sd_notify协议将无法正常工作。

## 高级日志设置

Freqtrade使用Python默认的日志模块。
Python允许进行详细的[日志配置](https://docs.python.org/3/library/logging.config.html#logging.config.dictConfig)，远超本文档所能覆盖的内容。

如果没有提供`log_config`参数，则会使用默认的彩色终端输出。使用`--logfile logfile.log`会启用滚动文件处理器（RotatingFileHandler）。
如果你对日志格式不满意，或者对默认的滚动文件处理器配置不满意，可以自行定制日志。

默认配置的大致示意如下——包括文件处理器，但未启用：

```json hl_lines="5-7 13-16 27"
{
  "log_config": {
      "version": 1,
      "formatters": {
          "basic": {
              "format": "%(message)s"
          },
          "standard": {
              "format": "%(asctime)s - %(name)s - %(levelname)s - %(message)s"
          }
      },
      "handlers": {
          "console": {
              "class": "freqtrade.loggers.ft_rich_handler.FtRichHandler",
              "formatter": "basic"
          },
          "file": {
              "class": "logging.handlers.RotatingFileHandler",
              "formatter": "standard",
              // "filename": "someRandomLogFile.log",
              "maxBytes": 10485760,
              "backupCount": 10
          }
      },
      "root": {
          "handlers": [
              "console",
              // "file"
          ],
          "level": "INFO"
      }
  }
}
```

!!! 提示“高亮部分”
   上述代码块中的高亮行定义了Rich日志处理器，两者相互关联。  
   `"standard"`格式化器和`"file"`处理器属于FileHandler。

每个处理器必须绑定一个已定义的格式化器（按名称），且其类必须是存在的合法日志类。
要启用某个处理器，必须在`root`部分的`handlers`列表中添加。而如果省略此部分，freqtrade就不会输出任何日志（即使定义了处理器，也不会生效）。

!!! 小技巧“显式日志配置”
   建议将日志配置单独抽离到主配置之外，通过[多配置文件](configuration.md#multiple-configuration-files)功能载入，避免重复代码。

---

在许多Linux系统中，你可以将机器人配置为向`syslog`或`journald`系统服务发送日志信息。在Windows上，也可以将日志发送到远程`syslog`服务器。`--logfile`命令行参数的特殊值可实现这一功能。

### 发送到syslog

使用`"log_config"`配置项来设置，将Freqtrade日志信息发往本地或远程syslog。

```json
{
  // ...
  "log_config": {
    "version": 1,
    "formatters": {
      "syslog_fmt": {
        "format": "%(name)s - %(levelname)s - %(message)s"
      }
    },
    "handlers": {
      // 其他处理器？ 
      "syslog": {
         "class": "logging.handlers.SysLogHandler",
          "formatter": "syslog_fmt",
          // 可以用其他地址代替
          "address": "/dev/log"
      }
    },
    "root": {
      "handlers": [
        // 其他处理器
        "syslog"
      ]
    }
  }
}
```

[其他高级日志处理器](#advanced-logging)也可以配置，以实现例如在控制台中输出日志。

#### Syslog使用说明

日志消息会发送到`syslog`的`user`系统组。可以用如下命令查看：

* `tail -f /var/log/user`
* 或在安装了图形界面的系统上，使用“日志文件查看器”等工具。

在许多系统中，`syslog`（如`rsyslog`）会从`journald`中采集数据（反之亦然），可以同时用`journalctl`和专业的syslog查看工具查看消息。你可以根据需要任意组合。

在`rsyslog`中，可以将机器人日志重定向到专门的日志文件，例如在配置文件（如`/etc/rsyslog.d/50-default.conf`）末尾添加：

```
if $programname startswith "freqtrade" then -/var/log/freqtrade.log
```

如果启用`Reduce`模式，系统会减少重复信息。例如，多条心跳消息会合并为一条，除非有别的事件发生。这可以在`/etc/rsyslog.conf`中设置：

```
# 去重过滤
$RepeatedMsgReduction on
```

#### syslog地址配置

syslog地址可以是Unix域套接字（socket文件路径），也可以是IP和端口组成的UDP套接字，比如：

* `"address": "/dev/log"` —— 使用`/dev/log`套接字，将日志发到syslog（rsyslog），多数系统支持。
* `"address": "/var/run/syslog"` —— 使用`/var/run/syslog`套接字（MacOS下常用）。
* `"address": "localhost:514"` —— 通过UDP端口514，将日志发往本地syslog。
* `"address": "<ip>:514"` —— 远程IP地址和514端口的syslog。

!!! 信息“已废弃——命令行配置syslog”

  `--logfile syslog:<syslog_address>` —— 通过指定`<syslog_address>`，将日志消息发往syslog。

syslog地址可为Unix套接字路径或IP+端口组合，示例如下：

* `--logfile syslog:/dev/log` —— 适用大部分系统的标准配置。
* `--logfile syslog` —— 等同于上面，默认路径为`/dev/log`。
* `--logfile syslog:/var/run/syslog` —— MacOS下使用。
* `--logfile syslog:localhost:514` —— 远程发到本地或远程syslog。
* `--logfile syslog:<ip>:514` —— 远程发往指定IP的syslog服务器。

### 发送到journald

需要安装Python包`cysystemd`（`pip install cysystemd`），但此包在Windows上不可用。因此，Windows平台无法使用journald日志功能。

要将Freqtrade日志发往`journald`，在配置中加入：

```json
{
  // ...
  "log_config": {
    "version": 1,
    "formatters": {
      "journald_fmt": {
        "format": "%(name)s - %(levelname)s - %(message)s"
      }
    },
    "handlers": {
      // 其他处理器？ 
      "journald": {
         "class": "cysystemd.journal.JournaldLogHandler",
          "formatter": "journald_fmt"
      }
    },
    "root": {
      "handlers": [
        // ..
        "journald"
      ]
    }
  }
}
```

[其他高级日志处理器](#advanced-logging)也可配置，以实现控制台输出等。

日志消息会发往`journald`的`user`系统组，查看方法包括：

* `journalctl -f` —— 监控`journald`中的所有日志（包括Freqtrade）。
* `journalctl -f -u freqtrade.service` —— 当机器人以`systemd`服务方式运行时用此命令。

此外，`journalctl`提供丰富的过滤选项，详情请参考其手册。

在许多系统中，`syslog`和`journald`会互相同步，既可用`--logfile syslog`也可用`--logfile journald`，两者的日志都可用`journalctl`和syslog查看。

!!! 信息“已废弃——命令行配置journald”
欲将频交易机器人日志发给`journald`，使用如下参数：

`--logfile journald` —— 发送日志到`journald`。

### JSON格式的日志

你还可以将默认输出格式改为JSON。`"fmt_dict"`属性定义了输出的键名，与Python的`logging.LogRecord`属性对应（详见[Python官方文档](https://docs.python.org/3/library/logging.html#logrecord-attributes)）。

示例配置如下，默认输出会改为JSON格式，也可结合`RotatingFileHandler`使用，但建议保持一种便于阅读的格式。

```json
{
  // ...
  "log_config": {
    "version": 1,
    "formatters": {
       "json": {
          "()": "freqtrade.loggers.json_formatter.JsonFormatter",
          "fmt_dict": {
              "timestamp": "asctime",
              "level": "levelname",
              "logger": "name",
              "message": "message"
          }
      }
    },
    "handlers": {
      // 其他处理器？ 
      "jsonStream": {
          "class": "logging.StreamHandler",
          "formatter": "json"
      }
    },
    "root": {
      "handlers": [
        // ..
        "jsonStream"
      ]
    }
  }
}
```