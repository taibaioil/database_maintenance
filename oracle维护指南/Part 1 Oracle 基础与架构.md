
#### **1.1 概述**  
- **Oracle 数据库版本历史与特性对比**  
  - **版本演进**：  
    - Oracle 8i（1999）：支持 Internet 应用，引入 Java 虚拟机（JVM）。  
    - Oracle 9i（2001）：RAC 技术成熟，提供 Data Guard 和 Oracle Streams。  
    - Oracle 10g（2003）：网格计算（Grid Computing），自动化管理（AWR/ADDM）。  
    - Oracle 11g（2007）：引入 Exadata，支持分区压缩和 SQL Plan Management。  
    - Oracle 12c（2013）：多租户架构（CDB/PDB），In-Memory 列存储。  
    - Oracle 19c/21c（2019+）：长期支持版本（LTS），增强 JSON 和区块链支持。  
  - **特性对比**：  
    - 12c 开始支持多租户，19c 优化了内存和自动化索引管理。  

- **单机架构 vs 集群架构（RAC）**  
  - **单机架构**：  
    - 单一实例访问本地存储，适合中小规模应用。  
    - 故障恢复依赖备份和日志，扩展性有限。  
  - **RAC（Real Application Clusters）**：  
    - 多实例共享存储（ASM/SAN），节点故障自动切换。  
    - 优点：高可用性（HA）、负载均衡、线性扩展。  
    - 缺点：配置复杂，需管理集群网络（Interconnect）。  

- **核心组件**  
  - **实例（Instance）**：运行时内存和进程的集合（SGA + 后台进程）。  
  - **数据库文件**：物理存储（数据文件、控制文件、日志文件）。  
  - **内存结构**：SGA（共享全局区）、PGA（程序全局区）。  

---

#### **1.2 数据库实例与存储结构**  

- **实例（Instance）与数据库（Database）的关系**  
  - **实例**：动态的，由内存和进程组成，通过参数文件（SPFILE）启动。  
  - **数据库**：静态的，由物理文件（数据文件、日志文件等）组成。  
  - **关系**：一个实例可以挂载一个数据库（单机），或多个实例共享一个数据库（RAC）。  

- **内存结构**  
  - **SGA（System Global Area）**：  
    - **Buffer Cache**：缓存数据块，减少磁盘 I/O。  
    - **Shared Pool**：存储 SQL 解析树和执行计划（Library Cache）、数据字典（Row Cache）。  
    - **Redo Log Buffer**：临时存储重做条目，由 LGWR 写入磁盘。  
    - **Large Pool**：用于 RMAN 备份、并行查询。  
    - **Java Pool**：支持 Java 应用。  
  - **PGA（Program Global Area）**：  
    - 每个会话私有，存储排序区（Sort Area）、哈希区（Hash Area）、会话变量。  

- **后台进程**  
  - **DBWn（Database Writer）**：将脏缓冲区写入数据文件（默认 1 个，可配置多个）。  
  - **LGWR（Log Writer）**：将 Redo Log Buffer 写入重做日志文件（同步提交）。  
  - **CKPT（Checkpoint Process）**：触发检查点，更新控制文件和数据文件头。  
  - **SMON（System Monitor）**：实例恢复（前滚+回滚）、清理临时段。  
  - **PMON（Process Monitor）**：清理异常会话，释放锁和资源。  
  - **ARCn（Archiver）**：归档模式下，将重做日志复制到归档日志（可选进程）。  

- **物理存储**  
  - **数据文件（Data Files）**：存储表、索引等实际数据（扩展名 `.dbf`）。  
  - **控制文件（Control File）**：记录数据库结构（数据文件/日志文件位置）、检查点信息（多路复用必需）。  
  - **重做日志文件（Redo Log Files）**：记录事务变化（至少 2 组，每组可多成员）。  
  - **临时文件（Temp Files）**：用于排序、临时表操作。  

- **逻辑存储**  
  - **表空间（Tablespace）**：逻辑容器（如 `SYSTEM`, `USERS`, `TEMP`），由多个数据文件组成。  
  - **段（Segment）**：表、索引等对象占用的空间（如 `TABLE_SEGMENT`）。  
  - **区（Extent）**：段分配的连续块组（自动扩展或手动管理）。  
  - **块（Block）**：最小 I/O 单元（默认 8KB，可调整）。  

---

#### **1.3 补充图表与示例**  

- **Oracle 单机架构示意图**  
  ```plaintext
  +-----------------------+
  |      Oracle Instance  |
  |  +-----------------+  |
  |  |      SGA        |  |
  |  | - Buffer Cache  |  |
  |  | - Shared Pool   |  |
  |  | - Redo Log Buff |  |
  |  +-----------------+  |
  |  +-----------------+  |
  |  |  后台进程        |  |
  |  | - DBWn, LGWR    |  |
  |  | - SMON, PMON    |  |
  |  +-----------------+  |
  +-----------+-----------+
              |
              v
  +-----------------------+
  |      Database Files   |
  | - Data Files (.dbf)   |
  | - Control Files (.ctl)|
  | - Redo Logs (.log)    |
  +-----------------------+
  ```

- **逻辑存储层次示例**  
  ```sql
  -- 查询表空间和数据文件  
  SELECT tablespace_name, file_name FROM dba_data_files;  

  -- 查询段和区信息  
  SELECT segment_name, extent_id, blocks FROM dba_extents WHERE owner='SCOTT';  
  ```

---

#### **1.4 关键参数与配置**  

- **初始化参数文件**  
  - **PFILE（init.ora）**：文本文件，手动编辑后需重启生效。  
  - **SPFILE（spfile.ora）**：二进制文件，支持动态修改（`ALTER SYSTEM SET`）。  

- **重要参数**  
  - `SGA_TARGET`：自动管理 SGA 组件大小。  
  - `PGA_AGGREGATE_TARGET`：控制 PGA 总内存。  
  - `DB_CACHE_SIZE`：Buffer Cache 大小。  
  - `SHARED_POOL_SIZE`：Shared Pool 大小。  
 

