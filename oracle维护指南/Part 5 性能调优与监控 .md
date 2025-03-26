### 关键要点  

- 性能调优和监控是优化Oracle数据库效率的重要过程，涵盖参数调整、SQL优化和存储管理。  
- 适用于单实例和RAC环境，RAC需额外关注集群性能。  
- 使用Oracle工具如Enterprise Manager和AWR进行监控，定期收集统计信息。  
- 意外细节：RAC环境需监控节点间通信和负载均衡，单实例更关注本地资源利用。  

---

### 性能调优与监控概述  

#### 什么是性能调优与监控  

性能调优旨在通过调整数据库设置和优化查询，提升数据库的运行效率，减少响应时间。监控则通过跟踪性能指标，及时发现潜在问题，确保数据库健康运行。适用于Oracle数据库的单实例和RAC（实时应用集群）环境，方法因环境不同而有所调整。  

#### 关键技术点  

以下是性能调优与监控的主要技术点，分为单实例和RAC环境：  

- **性能指标理解**  
  - 监控CPU使用率、内存使用（SGA、PGA）、I/O性能、网络性能和SQL执行时间。  
  - 使用视图如`v$sysstat`、`v$buffer_pool`查看指标。  

- **数据库参数调优**  
  - 调整SGA_TARGET、PGA_AGGREGATE_TARGET等参数，优化内存使用。  
  - 使用Memory Advisor确定最佳参数值。  

- **SQL性能调优**  
  - 识别慢查询，使用Explain Plan分析执行计划。  
  - 优化索引、收集统计信息，减少硬解析。  

- **存储和I/O优化**  
  - 管理表空间和数据文件分布，提升I/O效率。  
  - RAC环境使用ASM（自动存储管理）优化存储。  

- **内存管理**  
  - 启用自动内存管理（AMM），动态调整SGA和PGA。  
  - 监控共享池和缓冲区缓存使用情况。  

- **并发和锁定管理**  
  - 监控锁和闩锁，减少死锁和资源争用。  
  - 使用`v$lock`查看锁等待情况。  

- **网络性能优化**  
  - 配置监听器和共享服务器，优化客户端连接。  
  - 使用连接池减少网络开销。  

- **监控工具和技术**  
  - 使用Oracle Enterprise Manager（OEM）提供仪表盘和警报。  
  - 通过SQL*Plus查询数据字典视图，第三方工具如Nagios辅助监控。  

- **RAC环境性能监控**  
  - 监控节点间通信（如缓存融合）和负载均衡。  
  - 使用`v$CR_BLOCK_SERVER`查看实例间数据共享效率。  

- **最佳实践和案例**  
  - 定期更新统计信息，设置性能基线和警报。  
  - 案例：通过调整缓冲区缓存减少响应时间。  

---

### 调查笔记  

以下是关于Oracle数据库性能调优与监控的详细补充，涵盖单实例和RAC环境，旨在提供全面的技术点和实践指导，排除故障处理部分，专注于技术方法和策略。  

#### 引言  

性能调优与监控是数据库管理的核心，确保数据库高效运行，满足业务需求。针对单实例和RAC环境，性能调优和监控需考虑不同架构的特点。本部分将详细探讨性能指标、参数调优、SQL优化、存储管理、内存管理、并发控制、网络性能、监控工具以及RAC特定性能监控，并提供实践示例和最佳实践。  

#### 1. 理解性能指标  

性能指标是识别瓶颈和理解数据库行为的关键。关键指标包括：  

- **CPU使用率**：  

  - 数据库整体CPU利用率，进程级CPU使用情况。  

  - 相关等待事件如CPU等待。  

  - 示例查询：  

    ```sql  
    SELECT ROUND(100 * (VALUE / (SELECT VALUE FROM V$SYSSTAT WHERE NAME = 'CPU used by this session'))) AS CPU_Usage_Percent  
    FROM V$SYSSTAT  
    WHERE NAME = 'CPU used by this session';  
    ```

- **内存使用**：  

  - SGA（系统全局区）和PGA（程序全局区）使用情况。  

  - 缓冲区缓存命中率、共享池命中率。  

  - 示例查询：  

    ```sql  
    SELECT COMPONENT, CURRENT_SIZE FROM V$SGASTAT;  
    SELECT (1 - (STAT_VALUE / GETS)) * 100 AS HIT_RATIO  
    FROM V$BUFFER_POOL_STATISTICS  
    WHERE NAME = 'DEFAULT';  
    ```

- **I/O性能**：  

  - 磁盘读写速率、平均读写时间。  
  - 相关等待事件如I/O等待。  
  - 使用`v$iostat`监控I/O性能。  

- **网络性能**：  

  - 活动连接数、吞吐量。  

  - 相关等待事件如网络操作等待。  

  - 示例查询：  

    ```sql  
    SELECT * FROM v$netstat;  
    ```

- **SQL性能**：  

  - SQL语句执行时间、处理行数、等待事件。  
  - 硬解析频率影响性能，使用`V$SQL`监控。  

在RAC环境中，额外关注集群性能指标，如实例间通信延迟和缓存融合统计。例如：  

- 监控缓存融合：  

  ```sql  
  SELECT * FROM V$CR_BLOCK_SERVER;  
  ```

  此视图显示实例间块传输情况，反映数据共享效率。  

#### 2. 数据库参数调优  

数据库参数控制数据库行为和性能，关键参数包括：  

- **SGA_TARGET**：  

  - 控制SGA总大小，包括缓冲区缓存、共享池等。  

  - 使用Memory Advisor确定最佳大小：  

    ```sql  
    EXEC DBMS_ADVISOR.CREATE_TASK('SGA_ADVISOR', 'SGA Advisor Task');  
    EXEC DBMS_ADVISOR.EDIT_TASK('SGA_ADVISOR', 'type', 'sga');  
    EXEC DBMS_ADVISOR.EDIT_TASK('SGA_ADVISOR', 'begin_snap_id', <snap_id>);  
    EXEC DBMS_ADVISOR.EDIT_TASK('SGA_ADVISOR', 'end_snap_id', <snap_id>);  
    EXEC DBMS_ADVISOR.EXECUTE_TASK('SGA_ADVISOR');  
    EXEC DBMS_ADVISOR.get_task_report('SGA_ADVISOR', 'TEXT', 'report', 'html');  
    ```

- **PGA_AGGREGATE_TARGET**：  

  - 控制PGA总大小，用于排序、哈希等操作。  
  - 使用PGA Advisor类似方法调优。  

- **BUFFER_POOL_SIZE**：  

  - 设置缓冲区缓存大小，增大可减少I/O操作。  

- **LOG_BUFFER**：  

  - 设置重做日志缓冲区大小，增大可减少写重做日志频率。  

其他参数如`DB_WRITER_PROCESSES`、`DB_BLOCK_SIZE`也需根据数据库特性调整。  

在RAC环境中，集群相关参数如`CLUSTER_DATABASE`、`INSTANCE_NUMBER`通常在安装时设置，不常调整。  

#### 3. SQL性能调优  

SQL性能是整体性能的关键，调优策略包括：  

- **识别慢查询**：  

  - 使用SQL Tuning Advisor，监控`V$SQL`视图，查找高耗时查询。  

- **使用Explain Plan**：  

  - 显示SQL执行计划，优化访问方法（全表扫描 vs. 索引扫描）、连接方法等。  

  - 示例：  

    ```sql  
    EXPLAIN PLAN FOR SELECT * FROM employees;  
    SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);  
    ```

- **索引优化**：  

  - 确保表有适当索引，加速数据检索。  
  - 避免过度索引，增加DML操作维护成本。  

- **统计信息收集**：  

  - 定期更新表和索引统计信息，帮助成本优化器选择最佳计划。  

  - 示例：  

    ```sql  
    ANALYZE TABLE employees COMPUTE STATISTICS;  
    ```

- **绑定变量**：  

  - 使用绑定变量避免硬解析，提升性能。  

  - 示例：  

    ```sql  
    SELECT * FROM employees WHERE department = :dept;  
    ```

    相比`SELECT * FROM employees WHERE department = 'Sales';`，减少解析开销。  

在RAC环境中，SQL性能受数据分布和服务属性影响，需确保负载均衡。  

#### 4. 存储和I/O优化  

存储管理对性能至关重要：  

- **表空间管理**：  

  - 合理分配和监控表空间，使用`dba_tablespace_usage_metrics`查看使用率。  

- **数据文件放置**：  

  - 将数据文件分布到不同磁盘，提升I/O性能。  

- **ASM使用**：  

  - 使用ASM简化存储管理，优化I/O性能。  

  - 示例：检查ASM磁盘组：  

    ```sql  
    SELECT name, total_mb, free_mb FROM v$asm_diskgroup;  
    ```

在RAC环境中，存储通常为共享存储，ASM尤为重要，确保跨节点一致性。  

#### 5. 内存管理  

内存管理是性能的关键，Oracle提供自动和手动调优：  

- **自动内存管理（AMM）**：  

  - 基于SGA_TARGET和PGA_AGGREGATE_TARGET动态调整内存。  

  - 示例：  

    ```sql  
    ALTER SYSTEM SET SGA_TARGET=4G SCOPE=SPFILE;  
    ALTER SYSTEM SET PGA_AGGREGATE_TARGET=2G SCOPE=SPFILE;  
    ```

- **手动调优**：  

  - 调整共享池、缓冲区缓存、大池等组件。  
  - 监控内存相关等待事件，确保无瓶颈。  

在RAC环境中，每个实例有独立SGA，但某些内存（如ASM）为集群范围。  

#### 6. 并发和锁定管理  

并发控制确保多用户高效访问，减少资源争用：  

- **锁和闩锁**：  

  - 锁控制数据访问，闩锁用于内部结构保护。  

  - 监控锁：  

    ```sql  
    SELECT * FROM v$lock;  
    ```

  - 查找等待锁的会话：  

    ```sql  
    SELECT s.sid, s.username, l.type, l.mode, l.lmode  
    FROM v$session s, v$lock l  
    WHERE s.sid = l.sid AND s.status = 'WAITING';  
    ```

- **死锁和争用**：  

  - 通过适当应用设计减少死锁，优化事务处理。  

在RAC环境中，使用全局排队服务（GES）管理集群资源，局部排队服务（LMS）处理实例资源。  

#### 7. 网络性能优化  

网络性能影响客户端-服务器通信：  

- **监听器配置**：  
  - 确保监听器配置正确，使用TCP/IP协议。  

- **共享服务器**：  
  - 使用共享服务器处理多连接，减少进程数。  
  - 配置`SHARED_SERVERS`参数。  

- **连接池**：  
  - 应用层实现连接池，减少连接开销。  

监控网络性能：  

- 示例：  

  ```sql  
  SELECT * FROM v$netstat;  
  ```

在RAC环境中，使用SCAN IP帮助客户端负载均衡。  

#### 8. 监控工具和技术  

多种工具可用于监控：  

- **Oracle Enterprise Manager (OEM)**：  
  - 提供仪表盘、警报和报告，全面监控和管理。  

- **SQL*Plus**：  
  - 查询数据字典视图获取性能指标，如`v$sysstat`。  

- **第三方工具**：  
  - 如Nagios、SolarWinds等，扩展IT监控。  

此外，Oracle提供Tuning Pack、Diagnostic Pack等高级监控工具。  

#### 9. RAC环境性能监控  

RAC环境需关注集群性能：  

- **实例间通信**：  

  - 监控缓存融合和全局缓存服务，检查块传输效率。  

  - 示例：  

    ```sql  
    SELECT * FROM V$CR_BLOCK_SERVER;  
    ```

- **节点负载均衡**：  

  - 确保工作负载均匀分布，监控每个节点的CPU、内存、I/O使用率。  

- **集群性能**：  

  - 检查集群健康状态，如使用`crsctl status cluster -all`。  

#### 10. 最佳实践和案例  

分享最佳实践和案例，提供实用见解：  

- **最佳实践**：  

  - 定期更新统计信息，设置性能基线和警报。  

  - 使用AWR（自动工作负载仓库）捕获性能数据：  

    ```sql  
    SELECT * FROM v$awr;  
    ```

- **案例**：  

  - 通过调整缓冲区缓存大小，减少响应时间。  
  - 通过索引优化提升SQL执行效率。  

#### 总结与建议  

性能调优和监控是持续过程，需要根据业务需求调整。单实例环境关注本地资源，RAC环境需额外监控集群性能。使用Oracle工具如OEM和RMAN，结合最佳实践，确保数据库高效运行。  

| **环境**     | **调优重点**                 | **监控重点**                   | **工具推荐**          |
| ------------ | ---------------------------- | ------------------------------ | --------------------- |
| 单实例数据库 | 参数调整、SQL优化、I/O管理   | CPU、内存、I/O、网络性能       | OEM、SQL*Plus、Nagios |
| RAC 环境     | 集群参数、缓存融合、负载均衡 | 实例间通信、节点负载、集群健康 | OEM、ASM、crsctl      |