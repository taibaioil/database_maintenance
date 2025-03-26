#### **1. 重做日志（Redo Log Files）**

- **作用**：记录所有数据变更操作（DML/DDL），用于崩溃恢复，保障事务的**持久性（Durability）**。

- **类型**：

  - **在线重做日志（Online Redo Log）**：循环写入，至少配置2组，每组可多路复用。
  - **归档重做日志（Archived Redo Log）**：归档模式下生成，用于时间点恢复（PITR）和备库同步。

- **核心机制**：

  - 日志切换触发检查点（Checkpoint），确保数据写入数据文件。

- **管理命令**：

  ```sql
  SELECT group#, sequence#, status, archived FROM v$log;  -- 查看状态
  ALTER SYSTEM SWITCH LOGFILE;  -- 强制切换
  ALTER DATABASE ADD LOGFILE MEMBER '/path/redo01b.log' TO GROUP 1;  -- 添加成员
  ```

#### **2. 控制文件（Control File）**

- **作用**：记录数据库物理结构（数据文件、日志文件路径）、检查点信息和RMAN备份元数据，启动时验证一致性。

- **关键配置**：

  - 多路复用：建议至少3份副本。
  - 恢复方法：`RESTORE CONTROLFILE FROM AUTOBACKUP;` 或手动重建。

- **查询信息**：

  ```sql
  SELECT name FROM v$controlfile;  -- 查看路径
  ```

#### **3. Undo日志（Undo Logs）**

- **作用**：支持事务回滚（`ROLLBACK`）、读一致性和闪回查询（`Flashback Query`）。

- **存储位置**：Undo表空间（如`UNDOTBS1`），SMON进程自动清理。

- **监控命令**：

  ```sql
  SELECT tablespace_name, status FROM dba_tablespaces WHERE contents = 'UNDO';
  ```

#### **4. 归档日志（Archived Redo Logs）**

- **作用**：支持PITR和Data Guard备库同步。
- **管理要点**：
  - 路径由`LOG_ARCHIVE_DEST_n`定义。
  - 清理：`RMAN> DELETE ARCHIVELOG ALL COMPLETED BEFORE 'SYSDATE-7';`

#### **5. 警报日志与跟踪文件**

- **警报日志（Alert Log）**：
  - 路径：`$ORACLE_BASE/diag/rdbms/<SID>/<SID>/trace/alert_<SID>.log`。
  - 记录：启动/关闭、检查点、ORA错误（如ORA-01555）。
- **跟踪文件**：
  - 后台进程：如`ora_lgwr_<PID>.trc`。
  - 用户会话：`ALTER SESSION SET sql_trace=TRUE;`。

#### **6. 审计日志（Audit Logs）**

- **作用**：记录用户操作（登录、DDL、DML），满足合规要求。

- **类型**：

  - 标准审计：`AUDIT SELECT TABLE BY scott;`。
  - 精细审计（FGA）：监控特定列。

- **清理**：

  ```sql
  DELETE FROM aud$ WHERE timestamp < SYSDATE - 30;
  ```

#### **7. Flashback日志（Flashback Logs）**

- **作用**：支持`FLASHBACK DATABASE`，回退到过去时间点。
- **存储位置**：Flash Recovery Area（FRA），需启用`DB_FLASHBACK_RETENTION_TARGET`。

#### **8. 逻辑日志（Logical Logs）**

- **作用**：

  - **LogMiner**：解析Redo日志生成SQL，用于审计/恢复。
  - **GoldenGate**：捕获变更日志实现异构同步。

- **使用示例**：

  ```sql
  BEGIN
    DBMS_LOGMNR.ADD_LOGFILE('/u01/archivelog/1_100.arc');
    DBMS_LOGMNR.START_LOGMNR(OPTIONS => DBMS_LOGMNR.DICT_FROM_ONLINE_CATALOG);
  END;
  ```

#### **9. 密码文件（Password File）**

- **作用**：存储特权用户（如`SYS`、`SYSDBA`）密码哈希，用于远程认证。

- **路径**：`$ORACLE_HOME/dbs/orapw<SID>`（Linux）或`%ORACLE_HOME%\database\PWD<SID>.ora`（Windows）。

- **管理命令**：

  ```bash
  orapwd file=orapw<SID> password=<sys_password> entries=10
  ```

- **权限**：需设置为`640`。

#### **10. 网络配置文件**

- **作用**：管理网络连接。

- **核心文件**：

  - **Listener.ora**：

    ```plaintext
    LISTENER = (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = dbhost)(PORT = 1521)))
    SID_LIST_LISTENER = (SID_LIST = (SID_DESC = (GLOBAL_DBNAME = orcl)(SID_NAME = orcl)))
    ```

  - **TNSnames.ora**：

    ```plaintext
    ORCL = (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = dbhost)(PORT = 1521))(CONNECT_DATA = (SERVICE_NAME = orcl)))
    ```

- **工具**：`lsnrctl start/stop`、`tnsping ORCL`。

#### **11. 数据泵文件（Data Pump Files）**

- **作用**：存储`expdp`/`impdp`逻辑备份数据。

- **文件类型**：`.dmp`（数据）、`.log`（日志）、`.sql`（DDL，可选）。

- **命令**：

  ```sql
  expdp scott/tiger DIRECTORY=dpump_dir DUMPFILE=scott.dmp LOGFILE=scott.log
  impdp hr/hr DIRECTORY=dpump_dir DUMPFILE=scott.dmp REMAP_SCHEMA=scott:hr
  CREATE DIRECTORY dpump_dir AS '/u01/dpump';
  ```

- **高级功能**：`PARALLEL=4`、`EXCLUDE=TABLE:"IN ('EMP')"`。

#### **12. ASM磁盘组文件（ASM Diskgroups）**

- **作用**：简化存储管理，支持I/O均衡和冗余。

- **关键组件**：

  - 磁盘组：`EXTERNAL`/`NORMAL`/`HIGH`冗余。
  - ASM实例：管理元数据。

- **命令**：

  ```sql
  SELECT name, total_mb, free_mb FROM v$asm_diskgroup;
  CREATE DISKGROUP DATA NORMAL REDUNDANCY FAILGROUP fg1 DISK '/dev/sdb1', '/dev/sdc1' FAILGROUP fg2 DISK '/dev/sdd1', '/dev/sde1';
  ```

#### **13. 参数文件（Parameter Files）**

- **作用**：定义运行参数。

- **类型**：`PFILE`（`init<SID>.ora`）、`SPFILE`（`spfile<SID>.ora`）。

- **命令**：

  ```sql
  SELECT name, value FROM v$parameter;
  ALTER SYSTEM SET sga_target=4G SCOPE=SPFILE;
  CREATE SPFILE FROM PFILE='/path/initSID.ora';
  ```

#### **14. 数据文件（Data Files）**

- **作用**：存储实际数据。

- **命令**：

  ```sql
  SELECT file_name, tablespace_name, bytes FROM dba_data_files;
  ALTER TABLESPACE users ADD DATAFILE '/u01/oradata/users02.dbf' SIZE 500M;
  ```

#### **15. 临时文件（Temporary Files）**

- **作用**：存储临时表空间数据。

- **命令**：

  ```sql
  SELECT file_name, tablespace_name, bytes FROM dba_temp_files;
  ALTER TABLESPACE temp ADD TEMPFILE '/u01/oradata/temp02.dbf' SIZE 1G;
  ```

#### **16. Change Data Capture (CDC) 日志**

- **作用**：捕获表变化数据。

- **命令**：

  ```sql
  BEGIN
    DBMS_CDC_PUBLISH.CREATE_CHANGE_SET(change_set_name => 'MY_CHANGE_SET', change_source => 'SYNC_SOURCE');
  END;
  ```

#### **17. RMAN 备份文件（Backup Files）**

- **作用**：物理备份。

- **命令**：

  ```sql
  RMAN> LIST BACKUP;
  RMAN> BACKUP DATABASE PLUS ARCHIVELOG;
  ```

#### **18. Oracle Wallet 文件**

- **作用**：存储加密密钥/证书。

- **命令**：

  ```bash
  orapki wallet create -wallet /u01/wallet -pwd <wallet_password>
  ```

#### **19. SQL*Net 日志文件**

- **作用**：网络诊断。

- **配置**：

  ```plaintext
  DIAG_ADR_ENABLED=ON
  LOG_DIRECTORY_SERVER=/u01/logs
  ```

---

#### **总结表格**

| **文件/日志类型**        | **核心作用**             | **关键配置/命令**                               |
| ------------------------ | ------------------------ | ----------------------------------------------- |
| 重做日志                 | 崩溃恢复、事务持久性     | `ALTER SYSTEM SWITCH LOGFILE`                   |
| 控制文件                 | 记录物理结构、检查点信息 | `RESTORE CONTROLFILE FROM AUTOBACKUP`           |
| Undo日志                 | 事务回滚、读一致性       | `SELECT tablespace_name FROM dba_tablespaces`   |
| 归档日志                 | 时间点恢复、备库同步     | `RMAN> DELETE ARCHIVELOG BEFORE 'SYSDATE-7'`    |
| 警报日志与跟踪文件       | 故障诊断、性能分析       | `ALTER SESSION SET sql_trace=TRUE`              |
| 审计日志                 | 安全审计、合规检查       | `DELETE FROM aud$ WHERE timestamp < SYSDATE-30` |
| Flashback日志            | 数据库级回退             | `DB_FLASHBACK_RETENTION_TARGET`                 |
| 逻辑日志                 | 逻辑恢复、数据同步       | `DBMS_LOGMNR.START_LOGMNR`                      |
| 密码文件                 | 特权用户远程认证         | `orapwd file=orapw<SID>`                        |
| 网络配置文件             | 管理监听器与客户端连接   | `lsnrctl start`、`tnsping ORCL`                 |
| 数据泵文件               | 逻辑备份与数据迁移       | `expdp`、`impdp`、`CREATE DIRECTORY`            |
| ASM磁盘组文件            | 自动化存储管理、I/O优化  | `CREATE DISKGROUP DATA`                         |
| 参数文件（PFILE/SPFILE） | 定义数据库运行参数       | `ALTER SYSTEM SET sga_target=4G`                |
| 数据文件                 | 存储实际数据             | `ALTER TABLESPACE ADD DATAFILE`                 |
| 临时文件                 | 临时表空间存储排序数据   | `ALTER TABLESPACE temp ADD TEMPFILE`            |
| Change Data Capture 日志 | 捕获表变化数据           | `DBMS_CDC_PUBLISH.CREATE_CHANGE_SET`            |
| RMAN 备份文件            | 物理备份与恢复           | `RMAN> BACKUP DATABASE`                         |
| Oracle Wallet 文件       | 存储加密密钥与证书       | `orapki wallet create`                          |
| SQL*Net 日志文件         | 网络连接诊断             | `sqlnet.ora` 配置                               |

