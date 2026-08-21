# SplitDB v2.3 架构方案

> SplitDB 多分片 SQLite 嵌入式高性能存储架构 v2.3
>
> **SplitDB — Sharded SQLite for the rest of us.**

| 项目 | 内容 |
| --- | --- |
| 文档用途 | 架构定稿白皮书、开发规范、项目技术文档 |
| 适用项目 | Sosite 社区底层存储引擎 |
| 基线版本 | v2.3（冻结，正式启动开发） |

---

## 前言与架构总览

本方案为 SplitDB v2.3 最终定稿工程方案，面向中小型私有化社区、独立站长设计，基于原生 SQLite 构建分片存储系统。

**核心思路**：用时序分区 + 哈希分桶解决 SQLite 单文件锁竞争；采用分片作为唯一数据源，索引 / 搜索 / 统计为异步二级视图，具备完整自愈能力，无需额外部署 MySQL、Redis、消息队列。

方案精准适配低资源服务器（1 核 512M ~ 1 核 1G），DAU 0~8000 稳定舒适运行；文档包含设计理念、目录规范、建表语句、完整核心 PHP 源码、运维脚本、开发红线、测试验收清单，所有内容固化，可直接进入开发。

### 潜在风险提示

- `global_id.sqlite` 为写入单点，高并发场景后续可扩展 ID 预分配机制；
- 异步架构存在最终一致性，允许短暂读写延迟，业务层需接受该特性；
- 浏览计数模块依赖 APCu 扩展，需预备无 APCu 降级方案；
- 文件原子写入主要针对 Linux 环境优化，Windows 环境建议使用 WSL 或确保文件系统支持原子重命名操作。

### 优化增补规范

- 备份优先级：bucket 分片文件（真相源）＞ 其他索引库；索引库全部支持从分片一键重建；
- 文档底部补充部署环境最低要求：PHP ≥ 7.4（支持 PDO RETURNING、APCUIterator）；
- 系统支持从 Bucket 真相源重建二级索引。

---

## 一、核心设计理念

| 编号 | 原则 |
| --- | --- |
| 1 | Bucket 分片 = 唯一真实数据源（永不丢失、永不错乱、可全量重建所有索引） |
| 2 | 全局索引、搜索、统计 = 二级缓存视图（最终一致性，允许异步延迟） |
| 3 | 读无限扩容、写可控分片、复杂查询全部预计算 / 异步化 |

---

## 二、适用场景与边界

### ✅ 适用场景

- 中小型社区、个人站长、私有化部署
- 低资源服务器（1 核 512M / 1 核 1G 流畅运行）
- DAU 0~8000 完全舒适，DAU 8000~15000 限流可控稳定运行
- 无需 MySQL、无独立数据库进程、低内存、低 IO、部署极简

### ❌ 不适用场景

- 超高并发刷屏、电商秒杀、复杂多表 JOIN 统计、大型门户
- 应对策略：可后期平滑切换至 MySQL 驱动（适配器模式已预留）

---

## 三、品牌标识

| 项目 | 内容 |
| --- | --- |
| 项目名称 | SplitDB |
| 中文别称 | 分片之光 |
| 标语 | Sharded SQLite for the rest of us. |
| Logo | 断裂的 S + 数据桶，绿灰配色 |
| 配色 | SQLite 绿（#8FBC8F）+ 深灰（#2F2F2F） |
| 开源协议 | MIT |
| 命名空间 | SplitDB |
| Docker 镜像 | sosite/splitdb:latest |

---

## 四、整体架构目录（定稿）

```text
data/
├── meta/
│   ├── main_index.sqlite    # 帖子主索引（列表、分页、路由）
│   ├── search_index.sqlite  # FTS5 全文检索库
│   ├── stats_cache.sqlite   # 预计算全站统计
│   ├── task_queue/          # 多队列目录
│   │   ├── queue_0.sqlite
│   │   ├── queue_1.sqlite
│   │   └── queue_2.sqlite
│   └── global_id.sqlite     # 全局 ID 生成器
├── bucket/
│   ├── active/
│   │   ├── 2026Q3/
│   │   │   └── 0.sqlite ~ 31.sqlite
│   │   └── 2026Q4/
│   │       └── 0.sqlite ~ 31.sqlite
│   └── archive/             # 18 个月前冷数据
├── extern/
│   └── 2026/Q3/0/12345.txt
├── runtime/
├── lock/
└── log/
```

---

## 五、分区与路由规则

### 5.1 一级分区：季度时序分区

- 新数据永远写入当前季度目录
- 超过 18 个月的数据自动归档至 `archive/`

### 5.2 二级分区：哈希分桶

- 默认每季度 32 桶（可后台配置）
- 写入路由：`ID % 桶数量`

### 5.3 路由重大优化

- **写入时**：计算桶路径
- **读取时**：永远以 `main_index.bucket_path` 为准，不再二次哈希
- 未来季度可直接升级 32 → 64 → 128 桶，历史数据零迁移、零失效

---

## 六、SQLite 统一强制底层参数

```sql
PRAGMA busy_timeout = 5000;
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;
PRAGMA cache_size = -20000;
PRAGMA temp_store = MEMORY;
PRAGMA foreign_keys = OFF;
```

---

## 七、数据表结构

### 7.1 Bucket 分片内表结构

**topic（主帖表）**

```sql
CREATE TABLE topic (
    id INTEGER PRIMARY KEY,
    uid INTEGER,
    title TEXT,
    create_time INTEGER,
    update_time INTEGER,
    status TINYINT DEFAULT 0,
    extern_path TEXT
);
```

**reply（回复表）**

```sql
CREATE TABLE reply (
    id INTEGER PRIMARY KEY,
    pid INTEGER,
    uid INTEGER,
    create_time INTEGER,
    extern_path TEXT,
    status TINYINT DEFAULT 0
);
```

### 7.2 main_index.sqlite

```sql
CREATE TABLE topic_index (
    id INTEGER PRIMARY KEY,
    uid INTEGER,
    title TEXT,
    create_time INTEGER,
    last_reply_time INTEGER,
    view_count INTEGER DEFAULT 0,
    status TINYINT DEFAULT 0,
    bucket_path TEXT
);

CREATE INDEX idx_create_time ON topic_index (create_time DESC);
```

### 7.3 global_id.sqlite

```sql
CREATE TABLE id_generator (
    table_name TEXT PRIMARY KEY,
    last_id INTEGER DEFAULT 0
);

INSERT OR IGNORE INTO id_generator (table_name, last_id) VALUES ('topic', 0);
INSERT OR IGNORE INTO id_generator (table_name, last_id) VALUES ('reply', 0);
```

### 7.4 search_index.sqlite（FTS5）

```sql
CREATE VIRTUAL TABLE search_index USING fts5 (
    title,
    summary,
    topic_id UNINDEXED
);
```

### 7.5 stats_cache.sqlite

> 版块总数、今日发帖、用户发帖、热度数据（凌晨预计算）

### 7.6 task_queue（多队列）

```sql
CREATE TABLE task_queue (
    id INTEGER PRIMARY KEY,
    type TEXT,
    data TEXT,
    status TEXT DEFAULT 'pending',
    create_time INTEGER,
    update_time INTEGER DEFAULT 0,
    retry_count INTEGER DEFAULT 0
);

CREATE INDEX idx_status ON task_queue (status);
CREATE INDEX idx_update_time ON task_queue (update_time);
```

---

## 八、读写流程

### 8.1 写入流程

| 步骤 | 操作 |
| --- | --- |
| 1 | 从 global_id.sqlite 原子获取下一个 ID（带重试机制） |
| 2 | 计算季度分区 + 哈希桶：ID % 桶数 |
| 3 | LRU 取桶连接，写入 Bucket（含 ID、标题、正文外置） |
| 4 | 随机写入 3 个队列文件之一（打散锁竞争） |
| 5 | 前端直接返回成功（带上 topic_id 和 bucket_path） |

### 8.2 读取流程

| 场景 | 数据源 |
| --- | --- |
| 列表、首页、分页 | main_index |
| 帖子详情 | bucket_path → 单桶 |
| 搜索 | search_index → 按需补详情 |

---

## 九、PHP-FPM 适配核心方案

### 9.1 单进程 LRU 轻量连接池

- 限制单进程最多 8 个常驻句柄
- 请求结束自动清空

### 9.2 禁止持久连接

- 禁用 `ATTR_PERSISTENT`

### 9.3 系统层兜底

```bash
ulimit -n 65535
php-fpm rlimit_files = 65535
```

---

## 十、五个核心模块代码（v2.3 最终版）

### 模块一：数据库连接池

```php
namespace SplitDB;

use PDO;

class DBFactory {
    private static array $connections = [];
    private const MAX_HANDLES = 8;

    public static function getConnection(string $path): PDO {
        $key = md5($path);
        if (isset(self::$connections[$key])) {
            return self::$connections[$key];
        }
        if (count(self::$connections) >= self::MAX_HANDLES) {
            array_shift(self::$connections);
        }

        $pdo = new PDO("sqlite:" . $path, null, null, [
            PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
            PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
            PDO::ATTR_PERSISTENT         => false,
        ]);

        $pdo->exec("
            PRAGMA busy_timeout = 5000;
            PRAGMA journal_mode = WAL;
            PRAGMA synchronous = NORMAL;
            PRAGMA cache_size = -20000;
            PRAGMA temp_store = MEMORY;
            PRAGMA foreign_keys = OFF;
        ");

        self::$connections[$key] = $pdo;
        return $pdo;
    }
}
```

### 模块二：全局 ID 生成器（带重试机制）

```php
namespace SplitDB;

use PDO;
use PDOException;

class IDGenerator {
    public static function nextId(string $table, PDO $db, int $maxRetries = 3): int {
        $attempts = 0;
        while ($attempts < $maxRetries) {
            try {
                $stmt = $db->prepare("
                    UPDATE id_generator
                    SET last_id = last_id + 1
                    WHERE table_name = :table
                    RETURNING last_id
                ");
                $stmt->execute([':table' => $table]);
                return (int)$stmt->fetchColumn();
            } catch (PDOException $e) {
                $attempts++;
                if ($attempts >= $maxRetries) throw $e;
                usleep(rand(5000, 20000));
            }
        }
        throw new \RuntimeException("Failed to generate ID after {$maxRetries} attempts.");
    }
}
```

### 模块三：原子写外置文件

```php
namespace SplitDB;

class ExternStorage {
    public static function write(string $baseDir, int $topicId, int $bucketId, string $content): string {
        $year = date('Y');
        $quarter = 'Q' . ceil(date('n') / 3);
        $relDir = "extern/{$year}/{$quarter}/{$bucketId}";
        $fullDir = rtrim($baseDir, '/') . '/' . $relDir;
        if (!is_dir($fullDir)) mkdir($fullDir, 0755, true);

        $filePath = "{$fullDir}/{$topicId}.txt";
        $tmpPath = "{$filePath}." . uniqid('', true) . '.tmp';
        file_put_contents($tmpPath, $content, LOCK_EX);
        rename($tmpPath, $filePath);

        if (function_exists('apcu_store')) {
            apcu_store("extern_{$topicId}", $content, 600);
        }
        return "{$relDir}/{$topicId}.txt";
    }

    public static function read(string $baseDir, string $relPath, int $topicId): string {
        if (function_exists('apcu_fetch')) {
            $cached = apcu_fetch("extern_{$topicId}", $success);
            if ($success) return $cached;
        }

        $fullPath = rtrim($baseDir, '/') . '/' . $relPath;
        if (!file_exists($fullPath)) return '';

        $content = file_get_contents($fullPath);
        if (function_exists('apcu_store') && $content !== false) {
            apcu_store("extern_{$topicId}", $content, 600);
        }
        return $content ?: '';
    }
}
```

### 模块四：队列消费者（CAS 乐观锁 + 防挂起）

```php
namespace SplitDB;

use PDO;

class QueueConsumer {
    private PDO $db;
    private int $timeout = 300;

    public function __construct(PDO $db) {
        $this->db = $db;
    }

    public function consume(): void {
        $this->recoverTimeoutTasks();
        while (true) {
            $task = $this->fetchTask();
            if (!$task) { sleep(1); continue; }
            $this->processTask($task);
        }
    }

    private function fetchTask(): ?array {
        $row = $this->db->query("
            SELECT id, type, data FROM task_queue
            WHERE status = 'pending'
            LIMIT 1
        ")->fetch();

        if (!$row) return null;

        $stmt = $this->db->prepare("
            UPDATE task_queue
            SET status = 'processing', update_time = strftime('%s', 'now')
            WHERE id = :id AND status = 'pending'
        ");
        $stmt->execute([':id' => $row['id']]);

        // CAS 核心：必须校验是否真正抢占成功
        if ($stmt->rowCount() === 0) {
            return null;
        }
        return $row;
    }

    private function processTask(array $task): void {
        try {
            // 业务逻辑...
            $this->db->prepare("UPDATE task_queue SET status = 'done' WHERE id = :id")
                ->execute([':id' => $task['id']]);
        } catch (\Throwable $e) {
            // 异常时不标记失败，等待超时后自动恢复
        }
    }

    private function recoverTimeoutTasks(): void {
        $this->db->prepare("
            UPDATE task_queue
            SET status = 'pending', retry_count = retry_count + 1
            WHERE status = 'processing'
            AND update_time < strftime('%s', 'now') - :timeout
            AND retry_count < 3
        ")->execute([':timeout' => $this->timeout]);

        $this->db->prepare("
            UPDATE task_queue
            SET status = 'failed'
            WHERE status = 'processing'
            AND update_time < strftime('%s', 'now') - :timeout
            AND retry_count >= 3
        ")->execute([':timeout' => $this->timeout]);
    }
}
```

### 模块五：浏览量防刷落盘（APCuIterator）

```php
namespace SplitDB;

use PDO;

class ViewCounter {
    public static function inc(int $topicId): void {
        apcu_inc("topic_views_{$topicId}", 1);
    }

    public static function flush(PDO $db): void {
        if (!class_exists('APCUIterator')) return;

        $iterator = new APCUIterator('/^topic_views_/', APC_ITER_KEY | APC_ITER_VALUE);
        $db->beginTransaction();
        try {
            foreach ($iterator as $item) {
                $key = $item['key'];
                $views = (int)$item['value'];
                if ($views <= 0) continue;

                $id = (int)substr($key, 12);
                apcu_dec($key, $views);
                $db->exec("UPDATE topic_index SET view_count = view_count + {$views} WHERE id = {$id}");
            }
            $db->commit();
        } catch (\Throwable $e) {
            $db->rollBack();
            throw $e;
        }
    }
}
```

---

## 十一、WAL 回收脚本 & 定时任务

### WAL 回收脚本 `splitdb_wal_reclaim.sh`

```bash
#!/bin/bash
supervisorctl stop splitdb-worker:*
sleep 2
find /www/splitdb/data -name "*.sqlite" -exec sqlite3 {} "PRAGMA wal_checkpoint(TRUNCATE);" ;
supervisorctl start splitdb-worker:*
```

### Crontab 定时配置

```bash
0 4 * * * /usr/local/bin/splitdb_wal_reclaim.sh >> /var/log/splitdb_wal.log 2>&1
0 3 * * * php /www/splitdb/cli/audit.php >> /var/log/splitdb_audit.log 2>&1
0 2 1 * * php /www/splitdb/cli/archive.php >> /var/log/splitdb_archive.log 2>&1
0 5 * * 0 php /www/splitdb/cli/rebuild_search.php >> /var/log/splitdb_search.log 2>&1
```

---

## 十二、开发红线（强制遵守）

- ❌ 禁止前台实时跨桶统计
- ❌ 禁止网页同步更新索引
- ❌ 禁止全局 LIKE 遍历桶文件
- ❌ 禁止长事务、长快照
- ❌ 禁止高峰期 truncate WAL
- ❌ 禁止开启 PDO 持久连接
- ❌ 禁止依赖 Bucket 分片的自增 ID
- ❌ 禁止 HTTP 线程实时更新浏览量

---

## 部署环境最低要求

> 依据「前言 · 优化增补规范」补充。

- PHP ≥ 7.4（支持 PDO RETURNING、APCUIterator）
- 推荐 1 核 512M 及以上资源
- Linux 环境（文件原子写入基于 Linux 系统规范，Windows 存在兼容性差异）

---

## 整理说明

- 本文档由 `Splitdb V2.3 白皮书.txt` 整理为 Markdown，结构调整为章节标题 / 表格 / 代码块，内容未增删改。
- 原文为文本提取产物，代码块中缺失的 `$` 变量符号、被截断的标识符（如 `is_dir`、`file_exists`、`file_get_contents`、`LOCK_EX`、`$this->`、`$item['value']`）及 `find -name "*.sqlite"` 通配符等提取损坏项已按原意恢复；业务逻辑未改动。
