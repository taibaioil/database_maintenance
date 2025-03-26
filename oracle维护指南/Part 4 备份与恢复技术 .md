### 关键要点  

- 备份与恢复技术是数据库管理的重要部分，确保数据在故障时可恢复。  
- 包括全备份、增量备份和差异备份三种类型，适合单实例和 RAC 环境。  
- RMAN 是 Oracle 推荐的工具，支持冷备份和热备份。  
- 单实例和 RAC 环境在备份策略上有差异，RAC 需要跨节点协调。  

---

### 备份类型与方法  

**备份类型**  

- **全备份**：复制数据库所有数据，提供完整恢复点，耗时较长。  
- **增量备份**：只备份自上次全备份或增量备份后更改的数据，节省资源。  
- **差异备份**：备份自上次全备份后更改的所有数据，介于全备份和增量备份之间。  

**备份方法**  

- **冷备份（离线备份）**：数据库关闭期间备份，确保一致性，但需要停机，适合小型数据库。  
- **热备份（在线备份）**：数据库运行中备份，需要处理持续事务，适合大型数据库。  

### 备份与恢复工具  

- **RMAN（恢复管理器）**：Oracle 内置工具，支持冷热备份，具备块级损坏检测功能。  
- **SQL*Plus**：可用于手动备份，但效率较低，不适合大型数据库。  
- **操作系统工具**：如 `cp`、`tar`，用于冷备份或 RMAN 不可用时。  

### 备份策略  

**单实例数据库**  

- 管理简单，可用 RMAN 或其他工具备份，备份文件存储在本地或外部介质。  

**RAC 环境**  

- 因多节点和共享存储复杂，需用 RMAN 跨节点协调备份。  
- 可使用 ASM 管理存储，与 RMAN 集成，简化操作。  

### 恢复场景  

- **完整数据库恢复**：恢复到备份时的状态，适用于数据库丢失或损坏。  
- **点时间恢复（PITR）**：恢复到特定时间点，修正逻辑错误或数据损坏。  
- **实例恢复**：Oracle 使用重做日志自动恢复实例故障。  

### 实践示例  

**单实例数据库**  

- 启用归档日志模式：  

  ```sql  
  SHUTDOWN IMMEDIATE;  
  STARTUP MOUNT;  
  ALTER DATABASE ARCHIVELOG;  
  ALTER DATABASE OPEN;  
  ```

- 全备份：  

  ```sql  
  RMAN> BACKUP DATABASE;  
  ```

- 恢复：  

  ```sql  
  RMAN> RESTORE DATABASE;  
  RMAN> RECOVER DATABASE;  
  RMAN> STARTUP;  
  ```

**RAC 环境**  

- 配置 RMAN：  

  ```sql  
  RMAN> CONNECT CATALOG rman_user/rman_pass@racdb;  
  RMAN> CONNECT TARGET sys/SysPass123@racdb1;  
  RMAN> CONNECT TARGET sys/SysPass123@racdb2;  
  ```

- 全备份：  

  ```sql  
  RMAN> BACKUP DATABASE;  
  ```

- 恢复：  

  ```sql  
  RMAN> RESTORE DATABASE;  
  RMAN> RECOVER DATABASE;  
  RMAN> STARTUP;  
  ```

---

### 调查笔记  

以下是关于数据库备份与恢复技术的详细补充，涵盖单实例和 RAC 环境，旨在提供全面的技术点和实践指导，排除故障处理部分，专注于技术方法和策略。  

#### 引言  

备份与恢复是数据库管理的核心，确保数据在各种故障场景下可恢复。针对单实例和 RAC 环境，备份与恢复技术需考虑不同架构的特点。本部分将详细探讨备份类型、方法、工具、策略以及恢复场景，并提供实践示例。  

#### 备份类型与方法  

##### 备份类型  

1. **全备份**  
   - 定义：复制数据库所有数据，包括数据文件、控制文件和重做日志。  
   - 特点：提供完整恢复点，适合定期执行，但耗时长，占用资源多。  
   - 适用场景：初始备份或关键数据定期快照。  

2. **增量备份**  
   - 定义：只备份自上次全备份或增量备份后更改的数据（基于块级或事务级）。  
   - 特点：高效，节省存储和时间，需结合全备份使用。  
   - 适用场景：日常备份，减少备份窗口。  

3. **差异备份**  
   - 定义：备份自上次全备份后所有更改的数据，不依赖前次增量备份。  
   - 特点：比增量备份更全面，恢复时只需最新全备份和差异备份。  
   - 适用场景：平衡资源和恢复效率。  

##### 备份方法  

1. **冷备份（离线备份）**  
   - 过程：数据库关闭，备份所有相关文件（如数据文件、控制文件、归档日志）。  
   - 优点：确保数据一致性，无事务干扰。  
   - 缺点：需要停机，影响业务连续性。  
   - 适用场景：小型数据库或计划内维护窗口。  

2. **热备份（在线备份）**  
   - 过程：数据库运行中备份，需启用归档日志模式，RMAN 可处理持续事务。  
   - 优点：无停机，适合 24/7 运行的数据库。  
   - 缺点：复杂性高，需确保日志完整性。  
   - 适用场景：大型生产环境，业务连续性要求高。  

#### 备份与恢复工具  

##### RMAN（恢复管理器）  

- 功能：Oracle 提供的专用备份和恢复工具，支持冷热备份、增量备份、块级校验等。  
- 优势：  
  - 自动管理备份集和镜像副本。  
  - 集成压缩和加密功能，节省存储。  
  - 支持跨平台恢复和并行处理。  
- 使用示例：见实践部分。  
- 文档参考：[Oracle Database Backup and Recovery User's Guide](https://docs.oracle.com/en/database/oracle/oracleDatabase/19/brack/index.html)。  

##### SQL*Plus  

- 功能：通过 SQL 命令手动执行备份，如导出数据文件或控制文件。  

- 局限：缺乏自动化，适合小型数据库或临时操作。  

- 示例：  

  ```sql  
  ALTER DATABASE BACKUP CONTROLFILE TO '/backup/control.bkp';  
  ```

- 适用场景：简单环境或 RMAN 不可用时。  

##### 操作系统工具  

- 功能：使用系统命令（如 `cp`、`tar`）复制数据库文件，适合冷备份。  

- 局限：无法处理热备份，需手动确保一致性。  

- 示例：  

  ```bash  
  tar -czf /backup/db_backup.tar.gz /u01/oradata/*  
  ```

- 适用场景：小型数据库或测试环境。  

#### 备份策略  

##### 单实例数据库  

- **管理特点**：单节点，备份文件通常存储在本地磁盘或外部介质。  
- **推荐工具**：RMAN，结合操作系统工具用于冷备份。  
- **策略建议**：  
  - 每周全备份，日常增量备份。  
  - 启用归档日志模式，确保点时间恢复能力。  
- **存储考虑**：备份文件可存储在 NFS 或云存储，定期校验完整性。  

##### RAC 环境  

- **管理特点**：多节点共享存储，需跨节点协调备份。  
- **推荐工具**：RMAN，支持多通道并行备份，集成 ASM 存储。  
- **策略建议**：  
  - 全备份可按节点轮流执行，减少资源竞争。  
  - 增量备份需确保所有节点日志同步。  
  - 使用 SCAN 配置简化客户端访问备份服务。  
- **存储考虑**：备份存储在共享磁盘组（如 +RECO），支持高可用性。  

#### 恢复场景  

1. **完整数据库恢复**  

   - 目标：恢复数据库到备份时的完整状态。  
   - 过程：使用 RMAN 恢复数据文件和控制文件，应用归档日志。  
   - 适用场景：磁盘故障导致数据库丢失。  

2. **点时间恢复（PITR）**  

   - 目标：恢复到特定时间点，如 2025-03-25 23:59:59。  

   - 过程：RMAN 支持基于时间或 SCN（系统变更号）恢复。  

   - 示例命令：  

     ```sql  
     RMAN> RECOVER DATABASE UNTIL TIME "TO_DATE('2025-03-25 23:59:59','YYYY-MM-DD HH24:MI:SS')";  
     ```

   - 适用场景：误操作删除数据需回滚。  

3. **实例恢复**  

   - 目标：自动恢复实例故障，Oracle 使用重做日志和控制文件。  
   - 过程：实例重启时，BG（后台进程）自动应用未提交事务。  
   - 适用场景：单实例或 RAC 节点崩溃。  

#### 实践示例  

##### 单实例数据库  

1. **启用归档日志模式**  

   - 确保数据库支持热备份：  

     ```sql  
     SHUTDOWN IMMEDIATE;  
     STARTUP MOUNT;  
     ALTER DATABASE ARCHIVELOG;  
     ALTER DATABASE OPEN;  
     ```

2. **全备份**  

   - 使用 RMAN 执行：  

     ```sql  
     RMAN> BACKUP DATABASE;  
     ```

   - 可添加压缩：  

     ```sql  
     RMAN> BACKUP AS COMPRESSED BACKUPSET DATABASE;  
     ```

3. **增量备份**  

   - 示例：  

     ```sql  
     RMAN> BACKUP INCREMENTAL LEVEL 1 DATABASE;  
     ```

   - 需先执行全备份作为基线。  

4. **恢复**  

   - 完整恢复：  

     ```sql  
     RMAN> RESTORE DATABASE;  
     RMAN> RECOVER DATABASE;  
     RMAN> STARTUP;  
     ```

   - PITR 示例：  

     ```sql  
     RMAN> RESTORE DATABASE UNTIL TIME "TO_DATE('2025-03-25 23:59:59','YYYY-MM-DD HH24:MI:SS')";  
     RMAN> RECOVER DATABASE UNTIL TIME "TO_DATE('2025-03-25 23:59:59','YYYY-MM-DD HH24:MI:SS')";  
     RMAN> STARTUP;  
     ```

##### RAC 环境  

1. **配置 RMAN**  

   - 连接所有实例：  

     ```sql  
     RMAN> CONNECT CATALOG rman_user/rman_pass@racdb;  
     RMAN> CONNECT TARGET sys/SysPass123@racdb1;  
     RMAN> CONNECT TARGET sys/SysPass123@racdb2;  
     ```

   - 配置多通道以利用多节点资源：  

     ```sql  
     RMAN> CONFIGURE DEVICE TYPE DISK PARALLELISM 2;  
     ```

2. **全备份**  

   - 示例：  

     ```sql  
     RMAN> BACKUP DATABASE;  
     ```

   - 存储在共享磁盘组：  

     ```sql  
     RMAN> BACKUP DATABASE FORMAT '+RECO/%U';  
     ```

3. **恢复**  

   - 完整恢复：  

     ```sql  
     RMAN> RESTORE DATABASE;  
     RMAN> RECOVER DATABASE;  
     RMAN> STARTUP;  
     ```

   - PITR 示例：  

     ```sql  
     RMAN> RESTORE DATABASE UNTIL TIME "TO_DATE('2025-03-25 23:59:59','YYYY-MM-DD HH24:MI:SS')";  
     RMAN> RECOVER DATABASE UNTIL TIME "TO_DATE('2025-03-25 23:59:59','YYYY-MM-DD HH24:MI:SS')";  
     RMAN> STARTUP;  
     ```

#### 总结与建议  

备份与恢复技术需根据数据库规模和业务需求选择合适策略。单实例环境简单，RAC 环境需关注节点间协调和共享存储。RMAN 是推荐工具，支持自动化和高效管理。实践示例展示了常见操作，管理员可根据需求调整。  

| **环境**     | **备份工具**  | **推荐策略**               | **恢复重点**         |
| ------------ | ------------- | -------------------------- | -------------------- |
| 单实例数据库 | RMAN, OS 工具 | 每周全备份，日常增量备份   | 完整恢复和 PITR      |
| RAC 环境     | RMAN, ASM     | 跨节点协调，全备份轮流执行 | 多节点恢复，日志同步 |