#### 安装与初始配置

##### 单机环境  

1. **安装前准备**  

   - 检查操作系统依赖：确保安装 binutils、gcc、libaio 等包。  
     示例：  

     ```bash
     rpm -q binutils compat-libstdc++-33 gcc glibc ksh libaio libgcc make sysstat
     ```

   - 调整内核参数：编辑 `/etc/sysctl.conf`，设置共享内存、文件句柄数等。  
     示例：  

     ```bash
     kernel.shmmax = 68719476736
     kernel.shmall = 4294967296
     fs.file-max = 6815744
     net.ipv4.ip_local_port_range = 9000 65000
     ```

     应用更改：  

     ```bash
     sysctl -p
     ```

2. **静默安装**  

   - 配置响应文件 `db_install.rsp`，示例：  

     ```ini
     oracle.install.option=INSTALL_DB_SWONLY
     ORACLE_HOSTNAME=oradb01.example.com
     UNIX_GROUP_NAME=oinstall
     ORACLE_HOME=/u01/app/oracle/product/19c/dbhome_1
     ```

   - 执行安装：  

     ```bash
     ./runInstaller -silent -responseFile /path/to/db_install.rsp -ignorePrereq
     ```

   - 验证日志：`/u01/app/oraInventory/logs/installActions*.log`。

3. **数据库创建**  

   - 使用 DBCA：  

     ```bash
     dbca -silent -createDatabase -templateName General_Purpose.dbt -gdbName orcl -sid orcl -sysPassword SysPass123 -systemPassword SysPass123 -datafileDestination /u01/oradata
     ```

   - 手动创建：编辑 `init.ora`，示例：  

     ```ini
     db_name=orcl
     control_files=('/u01/oradata/orcl/control01.ctl', '/u02/oradata/orcl/control02.ctl')
     ```

     SQL 创建：  

     ```sql
     CREATE DATABASE orcl
     MAXLOGFILES 16
     MAXLOGMEMBERS 3
     DATAFILE '/u01/oradata/orcl/system01.dbf' SIZE 500M;
     ```

4. **初始化参数配置**  

   - 关键参数：SGA_TARGET（建议物理内存 40%-60%）、PGA_AGGREGATE_TARGET（SGA 的 1/2）、PROCESSES（根据并发用户调整）。  
     示例：  

     ```sql
     ALTER SYSTEM SET SGA_TARGET=4G SCOPE=SPFILE;
     ALTER SYSTEM SET PGA_AGGREGATE_TARGET=2G SCOPE=SPFILE;
     ALTER SYSTEM SET PROCESSES=500 SCOPE=SPFILE;
     ```

5. **环境变量设置**  

   - 编辑 `.bash_profile`：  

     ```bash
     export ORACLE_HOME=/u01/app/oracle/product/19c/dbhome_1
     export ORACLE_SID=orcl
     export PATH=$ORACLE_HOME/bin:$PATH
     export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
     ```

##### RAC 环境  

1. **安装前准备**  

   - 硬件与网络：多节点服务器，私有网络（节点间通信）、公共网络（客户端访问），SCAN IP（至少 1 个，推荐 3 个）。  
   - 共享存储：使用 ASM 或集群文件系统（如 OCFS2）。

2. **Grid Infrastructure 安装**  

   - 配置响应文件 `grid_install.rsp`，示例：  

     ```ini
     oracle.install.option=HA_CONFIG
     ORACLE_HOSTNAME=node1.example.com
     oracle.install.crs.config.scanName=scan-rac.example.com
     oracle.install.crs.config.gpnp.scanPort=1521
     ```

   - 执行：  

     ```bash
     ./gridSetup.sh -silent -responseFile /path/to/grid_install.rsp
     ```

   - 集群验证：  

     ```bash
     cluvfy stage -post crsinst -n node1,node2 -verbose
     ```

3. **RAC 数据库安装与创建**  

   - 安装数据库软件：  

     ```bash
     ./runInstaller -silent -responseFile /path/to/db_install.rsp -clusternodes node1,node2
     ```

   - 使用 DBCA 创建 RAC 数据库：  

     ```bash
     dbca -silent -createDatabase -templateName General_Purpose.dbt -gdbName racdb -sid racdb -sysPassword SysPass123 -nodelist node1,node2 -storageType ASM -datafileDestination +DATA
     ```

4. **RAC 初始化参数**  

   - 全局参数：  

     ```sql
     ALTER SYSTEM SET CLUSTER_DATABASE=TRUE SCOPE=SPFILE;
     ALTER SYSTEM SET INSTANCE_NUMBER=1 SID='racdb1' SCOPE=SPFILE;
     ALTER SYSTEM SET THREAD=1 SID='racdb1' SCOPE=SPFILE;
     ```

   - 节点特定参数：node2 示例：  

     ```sql
     ALTER SYSTEM SET INSTANCE_NUMBER=2 SID='racdb2' SCOPE=SPFILE;
     ALTER SYSTEM SET THREAD=2 SID='racdb2' SCOPE=SPFILE;
     ```

5. **环境变量**  

   - node1：  

     ```bash
     export ORACLE_SID=racdb1
     ```

   - node2：  

     ```bash
     export ORACLE_SID=racdb2
     ```

#### 用户与权限管理

##### 单机环境  

1. **用户创建**  

   - 示例：  

     ```sql
     CREATE USER app_user IDENTIFIED BY "SecurePass123#"
     DEFAULT TABLESPACE users
     TEMPORARY TABLESPACE temp
     QUOTA 100M ON users;
     ```

   - 修改密码策略：  

     ```sql
     ALTER PROFILE DEFAULT LIMIT PASSWORD_LIFE_TIME 180 PASSWORD_REUSE_MAX 10;
     ```

2. **权限分配**  

   - 系统权限：  

     ```sql
     GRANT CREATE SESSION, CREATE TABLE TO app_user;
     ```

   - 对象权限：  

     ```sql
     GRANT SELECT, INSERT ON hr.employees TO app_user;
     ```

3. **角色管理**  

   - 创建角色：  

     ```sql
     CREATE ROLE app_role;
     GRANT SELECT ON hr.departments TO app_role;
     GRANT app_role TO app_user;
     ```

4. **审计**  

   - 启用审计：  

     ```sql
     AUDIT SESSION BY app_user;
     AUDIT SELECT ON hr.employees BY ACCESS;
     ```

   - 查询审计结果：  

     ```sql
     SELECT username, action_name, timestamp FROM dba_audit_trail;
     ```

##### RAC 环境  

1. **用户管理**  

   - 用户全局共享：  

     ```sql
     CREATE USER rac_user IDENTIFIED BY "RacPass123#"
     DEFAULT TABLESPACE users;
     ```

2. **权限与服务关联**  

   - 分配权限：  

     ```sql
     GRANT CONNECT, RESOURCE TO rac_user;
     ```

   - 验证跨节点：  

     ```bash
     sqlplus rac_user/RacPass123#@racdb
     ```

3. **审计跨节点**  

   - 查询：  

     ```sql
     SELECT node_name, username, action_name FROM dba_audit_trail WHERE node_name IN ('node1', 'node2');
     ```

#### 表空间与数据文件管理

##### 单机环境  

1. **表空间创建**  

   - 永久表空间：  

     ```sql
     CREATE TABLESPACE app_data DATAFILE '/u01/oradata/app_data01.dbf' SIZE 1G
     AUTOEXTEND ON NEXT 100M MAXSIZE 10G;
     ```

   - 临时表空间：  

     ```sql
     CREATE TEMPORARY TABLESPACE temp_ts TEMPFILE '/u01/oradata/temp01.dbf' SIZE 500M;
     ```

2. **数据文件管理**  

   - 添加数据文件：  

     ```sql
     ALTER TABLESPACE app_data ADD DATAFILE '/u01/oradata/app_data02.dbf' SIZE 1G;
     ```

   - 调整大小：  

     ```sql
     ALTER DATABASE DATAFILE '/u01/oradata/app_data01.dbf' RESIZE 2G;
     ```

3. **监控与优化**  

   - 使用率：  

     ```sql
     SELECT tablespace_name, ROUND((bytes_used / bytes_total) * 100, 2) AS used_pct
     FROM dba_tablespace_usage_metrics;
     ```

   - Undo 表空间：  

     ```sql
     ALTER SYSTEM SET UNDO_RETENTION=3600 SCOPE=BOTH;
     ```

##### RAC 环境  

1. **表空间创建**  

   - 使用 ASM：  

     ```sql
     CREATE TABLESPACE rac_data DATAFILE '+DATA/racdb/datafile/rac_data01.dbf' SIZE 1G;
     ```

2. **数据文件管理**  

   - 数据文件在共享磁盘组，无需节点特定配置。  

   - 添加：  

     ```sql
     ALTER TABLESPACE rac_data ADD DATAFILE '+DATA' SIZE 1G;
     ```

3. **监控与优化**  

   - 检查 ASM 磁盘组：  

     ```sql
     SELECT name, total_mb, free_mb FROM v$asm_diskgroup;
     ```

   - Undo 管理：  

     ```sql
     CREATE UNDO TABLESPACE undo_rac2 DATAFILE '+DATA' SIZE 1G;
     ALTER SYSTEM SET UNDO_TABLESPACE='undo_rac2' SID='racdb2';
     ```

#### 网络配置

##### 单机环境  

1. **监听器配置**  

   - 编辑 `listener.ora`：  

     ```ini
     LISTENER =
       (DESCRIPTION =
         (ADDRESS = (PROTOCOL = TCP)(HOST = oradb01)(PORT = 1521))
       )
     ```

   - 启动：  

     ```bash
     lsnrctl start
     ```

2. **TNS 配置**  

   - 示例 `tnsnames.ora`：  

     ```ini
     ORCL =
       (DESCRIPTION =
         (ADDRESS = (PROTOCOL = TCP)(HOST = oradb01)(PORT = 1521))
         (CONNECT_DATA = (SERVICE_NAME = orcl))
       )
     ```

   - 故障排查：  

     - 检查状态：  

       ```bash
       lsnrctl status
       ```

     - 测试：  

       ```bash
       tnsping ORCL
       ```

##### RAC 环境  

1. **监听器与 SCAN 配置**  

   - SCAN 由 Grid 自动生成，无需手动配置。  

   - 本地监听器：  

     ```ini
     LISTENER_NODE1 =
       (ADDRESS = (PROTOCOL = TCP)(HOST = node1-vip)(PORT = 1521))
     ```

2. **TNS 配置**  

   - 客户端连接 RAC：  

     ```ini
     RACDB =
       (DESCRIPTION =
         (ADDRESS = (PROTOCOL = TCP)(HOST = scan-rac.example.com)(PORT = 1521))
         (CONNECT_DATA = (SERVICE_NAME = racdb))
       )
     ```

3. **负载均衡与故障转移**  

   - 服务配置：  

     ```sql
     srvctl add service -d racdb -s app_service -r racdb1,racdb2 -l PRIMARY -q TRUE -j SHORT
     srvctl start service -d racdb -s app_service
     ```

4. **故障排查**  

   - 检查集群监听：  

     ```bash
     srvctl status scan_listener
     ```

   - 网络问题：  

     ```bash
     crsctl stat res -t | grep listener
     ```

#### 备份与恢复（概述）

- **单机环境**  

  - 启用归档模式：  

    ```sql
    SHUTDOWN IMMEDIATE;
    STARTUP MOUNT;
    ALTER DATABASE ARCHIVELOG;
    ALTER DATABASE OPEN;
    ```

  - RMAN 配置：  

    ```sql
    RMAN> CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 7 DAYS;
    RMAN> CONFIGURE CONTROLFILE AUTOBACKUP ON;
    ```

- **RAC 环境**  

  - 归档日志路径：  

    ```sql
    ALTER SYSTEM SET log_archive_dest_1='LOCATION=+RECO' SCOPE=SPFILE;
    ```

  - RMAN 多节点备份：  

    ```sql
    RMAN> CONFIGURE CHANNEL 1 DEVICE TYPE DISK CONNECT 'sys/SysPass2023@racdb1';
    RMAN> CONFIGURE CHANNEL 2 DEVICE TYPE DISK CONNECT 'sys/SysPass2023@racdb2';
    ```

#### 日常维护任务（概述）

- **统计信息收集**  

  - 示例：  

    ```sql
    EXEC DBMS_STATS.GATHER_SCHEMA_STATS('HR', cascade => TRUE);
    ```

- **索引维护**  

  - 重建索引：  

    ```sql
    ALTER INDEX hr.emp_idx REBUILD ONLINE;
    ```

- **日志清理**  

  - 检查告警日志：  

    ```bash
    ls -lh /u01/app/oracle/diag/rdbms/prodDB/*/trace/alert*.log
    ```

  - 清理：  

    ```sql
    EXEC DBMS_SYSTEM.PURGE_LOG;
    ```

#### 最佳实践与故障排查

- **单机**  

  - 定期备份配置文件，启用自动内存管理（AMM）。  

  - 故障排查：监听器无法启动，检查端口占用：  

    ```bash
    netstat -tulnp | grep 1521
    ```

- **RAC**  

  - 使用 SCAN 连接客户端，确保私有网络延迟低于 1ms。  
  - 故障排查：节点通信失败，检查日志 `/u01/app/grid/diag`。

| **环境** | **安装重点**                      | **用户管理特点**         | **网络配置重点**        |
| -------- | --------------------------------- | ------------------------ | ----------------------- |
| 单机     | 操作系统依赖、静默安装、DBCA 创建 | 本地用户，权限独立管理   | 监听器和 TNS 配置       |
| RAC      | Grid 安装、集群配置、共享存储     | 用户全局共享，跨节点一致 | SCAN 配置、负载均衡服务 |