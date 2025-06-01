---
title: "sqlite命令行下的命令解释"
date: 2025-06-01T10:44:00+08:00
summary: "sqlite 的命令作用解释"
---

## 目录

[TOC]

---

## 前言

sqlite 和 mysql/postgresql 类似，除了支持标准的 SQL 语句（DML，DDL）之外，还拥有完整的非标准命令和许多扩展功能。

这些非标准命令主要是为了方便管理数据库元数据（如查看表、索引、配置），这些操作超出了标准 SQL 的 DML 和 DDL 范畴，SQL 标准未定义这些管理功能，这些命令目的是提供简洁、直观的交互方式。通常可以当作客户端工具的“快捷方式”，而不是数据库核心引擎的一部分。

比如，查看所有数据库，sqlite 是 `.databases`，mysql 是 `SHOW DATABASES`，pg 则是 `\l`，sqlite 的内置命令都以点开头，pg 则以反斜杠开头，mysql 则依赖类似 SQL 的语法。

---

## sqlite 的内置命令

通过 `.help` 可以查看 sqlite 所有支持的命令：

```sh
sqlite> .help
.archive ...             Manage SQL archives
.auth ON|OFF             Show authorizer callbacks
.backup ?DB? FILE        Backup DB (default "main") to FILE
.bail on|off             Stop after hitting an error.  Default OFF
.cd DIRECTORY            Change the working directory to DIRECTORY
.changes on|off          Show number of rows changed by SQL
.check GLOB              Fail if output since .testcase does not match
.clone NEWDB             Clone data into NEWDB from the existing database
.connection [close] [#]  Open or close an auxiliary database connection
.databases               List names and files of attached databases
.dbconfig ?op? ?val?     List or change sqlite3_db_config() options
.dbinfo ?DB?             Show status information about the database
.dump ?OBJECTS?          Render database content as SQL
.echo on|off             Turn command echo on or off
.eqp on|off|full|...     Enable or disable automatic EXPLAIN QUERY PLAN
.excel                   Display the output of next command in spreadsheet
.exit ?CODE?             Exit this program with return-code CODE
.expert                  EXPERIMENTAL. Suggest indexes for queries
.explain ?on|off|auto?   Change the EXPLAIN formatting mode.  Default: auto
.filectrl CMD ...        Run various sqlite3_file_control() operations
.fullschema ?--indent?   Show schema and the content of sqlite_stat tables
.headers on|off          Turn display of headers on or off
.help ?-all? ?PATTERN?   Show help text for PATTERN
.import FILE TABLE       Import data from FILE into TABLE
.indexes ?TABLE?         Show names of indexes
.limit ?LIMIT? ?VAL?     Display or change the value of an SQLITE_LIMIT
.lint OPTIONS            Report potential schema issues.
.load FILE ?ENTRY?       Load an extension library
.log FILE|on|off         Turn logging on or off.  FILE can be stderr/stdout
.mode MODE ?OPTIONS?     Set output mode
.nonce STRING            Suspend safe mode for one command if nonce matches
.nullvalue STRING        Use STRING in place of NULL values
.once ?OPTIONS? ?FILE?   Output for the next SQL command only to FILE
.open ?OPTIONS? ?FILE?   Close existing database and reopen FILE
.output ?FILE?           Send output to FILE or stdout if FILE is omitted
.parameter CMD ...       Manage SQL parameter bindings
.print STRING...         Print literal STRING
.progress N              Invoke progress handler after every N opcodes
.prompt MAIN CONTINUE    Replace the standard prompts
.quit                    Stop interpreting input stream, exit if primary.
.read FILE               Read input from FILE or command output
.recover                 Recover as much data as possible from corrupt db.
.restore ?DB? FILE       Restore content of DB (default "main") from FILE
.save ?OPTIONS? FILE     Write database to FILE (an alias for .backup ...)
.scanstats on|off|est    Turn sqlite3_stmt_scanstatus() metrics on or off
.schema ?PATTERN?        Show the CREATE statements matching PATTERN
.separator COL ?ROW?     Change the column and row separators
.session ?NAME? CMD ...  Create or control sessions
.sha3sum ...             Compute a SHA3 hash of database content
.shell CMD ARGS...       Run CMD ARGS... in a system shell
.show                    Show the current values for various settings
.stats ?ARG?             Show stats or turn stats on or off
.system CMD ARGS...      Run CMD ARGS... in a system shell
.tables ?TABLE?          List names of tables matching LIKE pattern TABLE
.timeout MS              Try opening locked tables for MS milliseconds
.timer on|off            Turn SQL timer on or off
.trace ?OPTIONS?         Output each SQL statement as it is run
.version                 Show source, library and compiler versions
.vfsinfo ?AUX?           Information about the top-level VFS
.vfslist                 List all available VFSes
.vfsname ?AUX?           Print the name of the VFS stack
.width NUM1 NUM2 ...     Set minimum column widths for columnar output
```

大约有 60 个命令，这里按照不同功能进行简单的分类，然后逐一介绍。

| index | category             | command                                                      |
| ----- | -------------------- | ------------------------------------------------------------ |
| 1     | 数据库文件与备份操作 | .archive .backup .clone .open .recover .restore .save .vfsinfo/.vfslist/.vfsname .filectrl |
| 2     | 数据导入导出         | .dump .excel .import .once .output .read                     |
| 3     | 元数据与结构查询     | .databases .fullschema .indexes .schema .tables .dbinfo      |
| 4     | 查询输出格式化       | .header .mode .nullvalue .separator .width .show             |
| 5     | 系统配置与调试       | .bail .cd .dbconfig .eqp .explain .expert .limit .lint .scanstats .timer .trace |
| 6     | 扩展与外部工具       | .load .shell .system .sha3sum                                |
| 7     | 会话与状态管理       | .changes .connection .log .parameter .progress .prompt .session |
| 8     | 辅助工具与帮助       | .auth .check .echo .exit .help .nonce .print .version .timeout |

---

## 数据库文件与备份操作



---

## 数据导入导出



---

## 元数据与结构查询



---

## 查询输出格式化



---

## 系统配置与调试



---

## 扩展与外部工具



---

## 会话与状态管理



---

##  辅助工具与帮助



---

## 参考

1. https://sqlite.org/cli.html
2. https://www.sqlitetutorial.net/sqlite-commands/
3. https://www.w3resource.com/sqlite/sqlite-dot-commands.php