# 检测调度引擎工作机制深度剖析

## 目录

- [一、系统架构概览](#一系统架构概览)
- [二、核心模块详解](#二核心模块详解)
- [三、完整生命周期时序图](#三完整生命周期时序图)
- [四、关键技术问题分析](#四关键技术问题分析)
- [五、高负载瓶颈与优化空间](#五高负载瓶颈与优化空间)
- [六、新人入职阅读指南](#六新人入职阅读指南)

---

## 一、系统架构概览

### 1.1 整体架构

本系统是一个**服务健康监控平台**，采用前后端分离架构：

```
┌─────────────────────────────────────────────────────────┐
│                        Frontend                         │
│  (React + Vite)  StatusPage / AdminPage / Charts        │
└──────────────────────────┬──────────────────────────────┘
                           │ REST API + WebSocket
┌──────────────────────────▼──────────────────────────────┐
│                     Backend (Node.js)                   │
│  ┌─────────┐  ┌───────────┐  ┌────────┐  ┌──────────┐  │
│  │  Routes │  │ Scheduler │  │ Checker│  │ Notifier │  │
│  └────┬────┘  └─────┬─────┘  └───┬────┘  └────┬─────┘  │
│       │             │            │            │        │
│       └─────────────┴─────┬──────┴────────────┘        │
│                           │                             │
│                      ┌────▼─────┐    ┌──────────┐      │
│                      │ Storage  │    │  Status  │      │
│                      │ (SQLite) │    │  (计算)   │      │
│                      └──────────┘    └──────────┘      │
└─────────────────────────────────────────────────────────┘
```

### 1.2 核心文件清单

| 文件 | 职责 | 行数 |
|------|------|------|
| `server.js` | 服务入口，初始化各模块，启动 HTTP 服务器 | ~78 |
| `scheduler.js` | 调度引擎核心，管理定时器和检测任务 | ~147 |
| `checker.js` | 实际执行检测（HTTP/HTTPS/TCP） | ~142 |
| `storage.js` | SQLite 数据持久化，数据库 CRUD | ~252 |
| `notifier.js` | WebSocket 实时推送 | ~90 |
| `status.js` | 服务状态计算、可用性统计、趋势数据 | ~125 |
| `routes.js` | REST API 路由定义 | ~261 |
| `config.js` / `config.json` | 配置加载 | ~14 / ~9 |
| `constants.js` | 常量定义（状态、类型、默认值等） | ~56 |
| `utils.js` | 工具函数（防抖、事件发射器等） | ~171 |

---

## 二、核心模块详解

### 2.1 服务配置加载机制

配置采用**双层加载策略**：静态 JSON 文件 + 运行时数据库配置。

#### 第一层：全局配置（config.json + config.js）

```javascript
// config.js - 配置加载
const config = require('./config.json');
const path = require('path');
const fs = require('fs');

// 确保数据目录和日志目录存在
const dataDir = path.join(__dirname, 'data');
const logDir = path.join(dataDir, 'logs');
if (!fs.existsSync(dataDir)) fs.mkdirSync(dataDir, { recursive: true });
if (!fs.existsSync(logDir)) fs.mkdirSync(logDir, { recursive: true });

module.exports = {
  ...config,
  dbPath: path.resolve(config.dbPath),    // 转换为绝对路径
  logDir: path.resolve(config.logDir)     // 转换为绝对路径
};
```

**配置项说明**：
- `port`: 服务监听端口（默认 3001）
- `dbPath`: SQLite 数据库文件路径
- `logDir`: 检测日志目录
- `dataRetentionDays`: 数据保留天数（默认 30 天）
- `defaultCheckIntervalSeconds`: 默认检测间隔（默认 30 秒）
- `defaultTimeoutMs`: 默认超时时间（默认 5000ms）
- `trendWindowHours`: 趋势数据时间窗口（默认 24 小时）

#### 第二层：服务级配置（数据库存储）

每个被监控服务有独立的配置，存储在 SQLite 的 `services` 表中：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | INTEGER | 主键，自增 |
| `name` | TEXT | 服务名称 |
| `type` | TEXT | 服务类型（http/https/tcp） |
| `target` | TEXT | 检测目标（URL 或主机名） |
| `port` | INTEGER | 端口号（TCP 专用） |
| `method` | TEXT | HTTP 方法（默认 GET） |
| `expectedStatus` | INTEGER | 期望的 HTTP 状态码（默认 200） |
| `interval_seconds` | INTEGER | 检测间隔秒数（默认 30） |
| `timeout_ms` | INTEGER | 超时时间毫秒（默认 5000） |
| `enabled` | INTEGER | 是否启用（1=启用，0=禁用） |

**配置加载流程**：
1. 启动时调用 `scheduler.startAll()`
2. 从数据库加载所有服务：`storage.services.getAll()`
3. 遍历服务，对 `enabled = 1` 的服务调用 `startServiceCheck()`

### 2.2 定时器创建与管理

#### 核心数据结构

```javascript
// scheduler.js:8-9
const serviceTimers = new Map()   // key: serviceId, value: { timer, interval, service }
const lastStatuses = new Map()    // key: serviceId, value: 'up' | 'down' | 'maintenance'
```

- `serviceTimers`: 管理所有活跃的定时器
- `lastStatuses`: 记录每个服务的上一次状态，用于检测状态变化

#### 定时器创建流程

```javascript
// scheduler.js:65-84
function startServiceCheck(service) {
  stopServiceCheck(service.id)  // 先停止已有定时器（幂等性保障）

  if (!service.enabled) return  // 未启用则跳过

  const interval = getInterval(service)

  // 立即执行一次检测（冷启动优化）
  runCheck(service).catch(err => {
    console.error(`[Scheduler] Initial check error for ${service.name}:`, err.message)
  })

  // 创建周期性定时器
  const timer = setInterval(() => {
    runCheck(service).catch(err => {
      console.error(`[Scheduler] Check error for ${service.name}:`, err.message)
    })
  }, interval)

  serviceTimers.set(service.id, { timer, interval, service })
  console.log(`[Scheduler] Started monitoring "${service.name}" every ${interval / 1000}s`)
}
```

**关键设计点**：
1. **先停后启**：调用 `stopServiceCheck()` 确保不会重复创建定时器
2. **立即执行**：启动后立即执行一次检测，避免等待首个周期
3. **错误隔离**：每个检测任务有独立的 catch，单个服务失败不影响全局
4. **间隔安全**：通过 `getInterval()` 和 `clamp()` 限制间隔范围（5秒 ~ 24小时）

#### 间隔计算与安全限制

```javascript
// scheduler.js:11-18
function getInterval(service) {
  const seconds = clamp(
    Number(service.interval_seconds) || DEFAULT_CONFIG.DEFAULT_INTERVAL_SECONDS,
    DEFAULT_CONFIG.MIN_INTERVAL_SECONDS,  // 5 秒
    86400                                  // 24 小时
  )
  return seconds * 1000
}
```

### 2.3 并发控制机制

#### 当前实现的并发模型

本系统采用**"每个服务独立定时器 + 异步并发执行"**的模型：

```
每个服务一个 setInterval
        │
        ├── 定时器触发时，调用 runCheck()（异步）
        ├── runCheck() 内部使用 async/await
        └── 多个服务的检测任务并行执行
```

**并发特点**：
1. **服务间并发**：不同服务的检测完全并行，互不阻塞
2. **服务内串行**：单个服务的检测由 setInterval 触发，但需要注意**任务重叠问题**

> ⚠️ **重要发现**：当前实现没有检测任务重叠的机制。如果某个检测任务的执行时间（如超时 5 秒）超过了检测间隔（如 3 秒），会导致多个检测任务同时在途，造成任务堆积。

#### 无显式并发限制的原因

1. **IO 密集型**：检测任务主要是网络 IO，Node.js 的事件循环天然支持高并发 IO
2. **规模预期**：设计目标是中小规模监控（几十 ~ 几百个服务）
3. **超时限制**：每个检测有独立的超时（默认 5 秒），限制了单个任务的最长耗时

### 2.4 检测任务执行流程

#### 检测入口

```javascript
// scheduler.js:20-63
async function runCheck(service) {
  const timestamp = new Date().toISOString()

  // 1. 检查是否在维护窗口内
  let isMaintenance = false
  try {
    const active = await storage.maintenance.getActive(service.id, timestamp)
    isMaintenance = active.length > 0
  } catch (e) {
    console.error(`[Scheduler] Maintenance check error for #${service.id}:`, e.message)
  }

  // 2. 执行实际检测
  const rawResult = await checkService(service)

  // 3. 组装存储结果
  const storedResult = {
    service_id: service.id,
    timestamp,
    success: isMaintenance ? 1 : (rawResult.success ? 1 : 0),
    response_time_ms: rawResult.response_time_ms ?? null,
    error_message: isMaintenance ? null : (rawResult.error_message || null),
    status_code: rawResult.status_code ?? null,
    is_maintenance: isMaintenance ? 1 : 0
  }

  // 4. 持久化结果
  try {
    await storage.checkResults.insert(storedResult)
  } catch (e) {
    console.error(`[Scheduler] DB insert error for ${service.name}:`, e.message)
  }

  // 5. 计算状态摘要并推送通知
  try {
    const summary = await status.getServiceSummary(service.id)
    notifier.notifyNewCheck(service.id, storedResult, summary)

    // 检测状态变化
    const previous = lastStatuses.get(service.id)
    if (previous !== summary.status) {
      lastStatuses.set(service.id, summary.status)
      if (previous !== undefined) {
        notifier.notifyStatusChange(service.id, summary.status, summary)
      }
    }
  } catch (e) {
    console.error(`[Scheduler] Summary calc error for ${service.name}:`, e.message)
  }
}
```

#### 实际检测实现（checker.js）

**HTTP/HTTPS 检测**：
- 使用 `axios` 库发送 HTTP 请求
- 支持自定义方法、期望状态码
- HTTPS 模式下忽略证书校验（`rejectUnauthorized: false`）
- 精确测量响应时间（`Date.now() - startTime`）

**TCP 检测**：
- 使用 Node.js 原生 `net.Socket`
- 手动管理连接生命周期
- 支持 `host:port` 格式或独立 port 字段
- 通过 `finished` 标志防止重复回调

### 2.5 结果持久化机制

#### 数据库选型：SQLite (sql.js)

使用 `sql.js`（纯 JavaScript 实现的 SQLite），特点：
- **纯内存操作**：数据库完全在内存中，读写极快
- **定期落盘**：每 5 秒将内存数据写入磁盘文件
- **零依赖**：不需要安装原生 SQLite 库

#### 持久化策略

```javascript
// storage.js:9-27
let dirty = false;  // 脏数据标志

function saveDB() {
  if (!db || !dirty) return;  // 无脏数据则跳过
  try {
    const data = db.export();
    const buffer = Buffer.from(data);
    const dbDir = path.dirname(config.dbPath);
    if (!fs.existsSync(dbDir)) fs.mkdirSync(dbDir, { recursive: true });
    const tmpPath = config.dbPath + '.tmp';  // 临时文件
    fs.writeFileSync(tmpPath, buffer);
    fs.renameSync(tmpPath, config.dbPath);   // 原子替换
    dirty = false;
  } catch (e) {
    console.error('[Storage] Save DB error:', e.message);
  }
}

setInterval(saveDB, 5000);  // 每 5 秒落盘一次
```

**持久化触发时机**：
1. **定时触发**：每 5 秒执行一次 `saveDB()`
2. **内存阈值**：堆内存超过 512MB 时强制落盘（`storage.js:197`）
3. **关键操作后**：服务增删改、维护窗口变更后立即落盘
4. **进程退出**：`beforeExit`、`SIGINT`、`SIGTERM` 信号触发

**原子写入保障**：
- 先写入 `.tmp` 临时文件
- 再通过 `rename` 原子替换原文件
- 确保崩溃时不会损坏数据库文件

#### 双写机制：数据库 + 日志文件

```javascript
// storage.js:111-126
function appendLog(serviceId, result) {
  try {
    const logFile = path.join(config.logDir, `service-${serviceId}-${moment().format('YYYY-MM-DD')}.log`);
    const line = JSON.stringify({
      ts: result.timestamp,
      success: result.success ? 1 : 0,
      rt: result.response_time_ms,
      msg: result.error_message || '',
      status: result.status_code || '',
      maint: result.is_maintenance ? 1 : 0
    }) + '\n';
    fs.appendFileSync(logFile, line, 'utf8');
  } catch (e) {
    console.error('[Storage] Log append error:', e.message);
  }
}
```

**双写设计考量**：
1. **数据冗余**：日志文件作为数据库的补充备份
2. **离线分析**：日志文件可直接用 grep/awk 等工具分析
3. **按天分割**：每天一个日志文件，便于归档和清理
4. **JSON Lines 格式**：每行一个 JSON，方便程序解析

### 2.6 WebSocket 实时推送机制

#### 架构设计

```javascript
// notifier.js:3-4
let wss = null;
const clients = new Set();  // 所有连接的客户端
```

- 采用 `ws` 库实现 WebSocket 服务
- 挂载在同一个 HTTP server 上（路径 `/ws`）
- 使用 `Set` 管理所有客户端连接

#### 消息类型

| 类型 | 触发时机 | 数据内容 |
|------|----------|----------|
| `hello` | 客户端连接成功 | 欢迎消息 |
| `new_check` | 每次检测完成 | 检测结果 + 状态摘要 |
| `status_change` | 服务状态变化时 | 新状态 + 摘要 |
| `service_update` | 服务配置变更时 | 服务配置信息 |
| `service_deleted` | 服务被删除时 | 服务 ID |
| `maintenance_change` | 维护窗口变更时 | 维护窗口信息 |

#### 广播机制

```javascript
// notifier.js:25-36
function broadcast(message) {
  const payload = JSON.stringify(message);
  for (const client of clients) {
    if (client.readyState === WebSocket.OPEN) {
      try {
        client.send(payload);
      } catch (err) {
        clients.delete(client);  // 发送失败则移除
      }
    }
  }
}
```

**推送流程**：
1. 检测任务完成 → `scheduler.js` 调用 `notifier.notifyNewCheck()`
2. 状态发生变化 → 额外调用 `notifier.notifyStatusChange()`
3. `broadcast()` 遍历所有客户端，逐一发送消息
4. 发送失败的客户端被自动清理

---

## 三、完整生命周期时序图

### 3.1 系统启动时序

```
start() 启动函数
   │
   ├───► storage.initDB() ────────────────────► 加载/创建 SQLite 数据库
   │                                              │
   │                                              ├── 初始化 services 表
   │                                              ├── 初始化 check_results 表
   │                                              ├── 初始化 maintenance_windows 表
   │                                              └── 创建索引
   │
   ├───► notifier.init(server) ───────────────► 创建 WebSocket 服务
   │                                              │
   │                                              └── 监听 connection 事件
   │
   ├───► scheduler.startAll() ────────────────► 启动所有服务监控
   │                                              │
   │                                              ├── 从 DB 加载所有服务
   │                                              │
   │                                              └── 遍历服务：
   │                                                    │
   │                                                    ├── enabled? ──否──► 跳过
   │                                                    │
   │                                                    └── 是 ──► startServiceCheck()
   │                                                                  │
   │                                                                  ├── 立即执行 runCheck()
   │                                                                  └── setInterval 定时触发
   │
   ├───► setInterval(cleanupOldData, 1h) ────► 每小时清理旧数据
   │
   └───► server.listen() ────────────────────► 开始监听端口
```

### 3.2 单次检测任务执行时序

```
setInterval 触发
      │
      ▼
runCheck(service) ────────────────────────────────────────────┐
      │                                                        │
      ├── 记录当前 timestamp                                   │
      │                                                        │
      ├──► storage.maintenance.getActive() ◄──────────────────┼── 检查维护窗口
      │     │                                                  │
      │     └── 返回活跃的维护窗口列表                          │
      │                                                        │
      ├──► checkService(service) ◄────────────────────────────┼── 执行检测
      │     │                                                  │
      │     ├── (HTTP) axios 请求 ──► 目标服务                 │
      │     │     │                                            │
      │     │     └── 返回 { success, response_time_ms, ... }  │
      │     │                                                  │
      │     └── (TCP) net.Socket 连接 ──► 目标主机:端口        │
      │           │                                            │
      │           └── 返回 { success, response_time_ms, ... }  │
      │                                                        │
      ├── 组装 storedResult                                    │
      │     │                                                  │
      │     └── 维护中则强制 success=1                         │
      │                                                        │
      ├──► storage.checkResults.insert() ◄─────────────────────┼── 持久化
      │     │                                                  │
      │     ├── 写入 SQLite（内存）                             │
      │     ├── 追加日志文件                                   │
      │     └── 检查内存阈值，超 512MB 则落盘                   │
      │                                                        │
      ├──► status.getServiceSummary() ◄────────────────────────┼── 计算状态
      │     │                                                  │
      │     ├── 取最近 24 小时检测结果                          │
      │     ├── 计算可用率、平均响应时间                        │
      │     └── 取最新一条结果判断当前状态                      │
      │                                                        │
      ├──► notifier.notifyNewCheck() ◄─────────────────────────┼── 推送新检测结果
      │     │                                                  │
      │     └── broadcast 给所有 WebSocket 客户端              │
      │                                                        │
      └── 检查状态变化                                         │
            │                                                  │
            ├── 状态未变 → 结束                                │
            │                                                  │
            └── 状态变化 → notifier.notifyStatusChange()       │
                  │                                            │
                  └── broadcast 状态变更通知                   │
```

### 3.3 服务配置变更时序（动态调整）

```
用户请求 PUT /api/services/:id
      │
      ▼
routes.js - PUT /services/:id
      │
      ├── 从 DB 读取现有服务
      ├── 校验参数合法性
      ├── 更新数据库
      │
      ├──► scheduler.restartServiceCheck(updated) ◄──────────┐
      │     │                                                 │
      │     ├── stopServiceCheck(id)                          │
      │     │     │                                           │
      │     │     ├── 从 serviceTimers 获取 timer             │
      │     │     ├── clearInterval(timer)                    │
      │     │     ├── serviceTimers.delete(id)                │
      │     │     └── lastStatuses.delete(id)                 │
      │     │                                                 │
      │     └── startServiceCheck(updated)                    │
      │           │                                           │
      │           ├── 计算新的 interval                        │
      │           ├── 立即执行一次 runCheck                    │
      │           └── 创建新的 setInterval                     │
      │                                                       │
      └──► notifier.notifyServiceUpdate() ◄───────────────────┘
            │
            └── broadcast 服务更新通知
```

### 3.4 WebSocket 消息流向图

```
┌──────────────┐
│  检测调度器   │
│ (scheduler)  │
└──────┬───────┘
       │
       ├── 检测完成 → notifyNewCheck() ──┐
       │                                  │
       └── 状态变化 → notifyStatusChange() ──┐
                                            │
┌──────────────┐     ┌──────────────────▼─────┐
│  配置管理     │     │      notifier.js       │
│  (routes)    │────►│  - clients Set         │
│              │     │  - broadcast()         │
└──────────────┘     └─────────┬──────────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
         ┌────▼───┐       ┌────▼───┐       ┌────▼───┐
         │ Client1│       │ Client2│       │ ClientN│
         │ (WS)   │       │ (WS)   │       │ (WS)   │
         └────────┘       └────────┘       └────────┘
```

---

## 四、关键技术问题分析

### 4.1 定时器漂移问题

#### 什么是定时器漂移？

JavaScript 的 `setInterval` 不能保证精确的时间间隔，实际触发时间可能晚于设定时间。造成漂移的原因：
1. **事件循环阻塞**：同步代码或 CPU 密集型任务阻塞了事件循环
2. **任务排队**：事件循环中已有大量待处理任务
3. **系统调度**：操作系统进程调度延迟

#### 当前系统如何处理漂移？

**现状：未做特殊处理**

当前实现直接使用原生 `setInterval`，没有漂移校正机制。但有以下缓解因素：

1. **漂移累积有限**：`setInterval` 会尝试"追赶"，虽然每次触发可能延迟，但间隔是相对于上一次**执行完成**还是**开始**，取决于事件循环状态
2. **检测任务短**：正常检测通常几十到几百毫秒完成，漂移量相对于检测间隔（默认 30 秒）很小
3. **IO 密集型**：主要是网络等待，不阻塞事件循环

#### 潜在风险与改进方向

```
当前（固定间隔）：
   触发──►执行───完成        触发──►执行─────完成
   ├───────间隔───────┤      ├───────间隔───────┤

理想（固定频率，漂移校正）：
   触发──►执行───完成    触发──►执行─────完成    触发
   ├─────周期─────┤      ├─────周期─────┤      ├──┤
```

**改进方案**：使用 `setTimeout` 递归调用，每次根据实际时间偏差调整下一次延迟。

```javascript
// 改进思路：动态调整的定时器
function startAccurateTimer(fn, interval) {
  let expected = Date.now() + interval;
  let timeoutId;

  function step() {
    const drift = Date.now() - expected;
    fn(); // 执行任务
    expected += interval;
    timeoutId = setTimeout(step, Math.max(0, interval - drift));
  }

  timeoutId = setTimeout(step, interval);
  return {
    stop: () => clearTimeout(timeoutId)
  };
}
```

### 4.2 任务堆积问题

#### 什么是任务堆积？

当检测任务的**执行时间**超过**检测间隔**时，上一次检测还没完成，下一次检测又开始了，导致多个检测任务同时在途，越积越多。

#### 当前系统的风险点

```
检测间隔: 5 秒
检测超时: 5 秒

极端情况（目标服务全部超时）：
  0s: 第1次检测开始
  5s: 第1次检测超时 → 第2次检测又开始了
  10s: 第2次检测超时 → 第3次检测又开始了
  ...
  结果：同时有多个检测任务在途
```

**当前防护措施**：
1. **最小间隔限制**：`MIN_INTERVAL_SECONDS = 5`（`constants.js:31`）
2. **超时限制**：默认 `timeout_ms = 5000`，最长 5 秒

**但是**：当间隔 = 5秒且检测全部超时时，刚好可能发生任务重叠。

#### 改进方案：任务去重叠机制

```javascript
// 方案：增加 inProgress 标志，防止任务重叠
const inProgress = new Set(); // 记录正在执行检测的服务

async function runCheck(service) {
  if (inProgress.has(service.id)) {
    console.warn(`[Scheduler] Skipping check for ${service.name}: previous check still in progress`);
    return;
  }
  
  inProgress.add(service.id);
  try {
    // ... 原有检测逻辑 ...
  } finally {
    inProgress.delete(service.id);
  }
}
```

### 4.3 内存泄漏问题

#### 潜在的内存泄漏点

**1. 定时器未清理**
```javascript
// 当前代码在 stopServiceCheck 中已清理
function stopServiceCheck(serviceId) {
  const existing = serviceTimers.get(serviceId)
  if (existing) {
    clearInterval(existing.timer)  // ✓ 已清理
    serviceTimers.delete(serviceId) // ✓ 已清理
    lastStatuses.delete(serviceId)  // ✓ 已清理
  }
}
```
✅ **已处理**：定时器和状态缓存都有对应清理逻辑。

**2. WebSocket 客户端未清理**
```javascript
// notifier.js:13-19
ws.on('close', () => {
  clients.delete(ws);  // ✓ 正常关闭时清理
});
ws.on('error', () => {
  clients.delete(ws);  // ✓ 错误时清理
});
```
✅ **已处理**：close 和 error 事件都清理客户端。

**3. 数据库内存增长**
```javascript
// storage.js:197
if (process.memoryUsage().heapUsed > 512 * 1024 * 1024) saveDB();
```
⚠️ **部分处理**：内存阈值触发落盘，但只是把内存数据写入磁盘，**并没有减少内存占用**。SQLite 内存数据库会持续增长，直到数据清理。

**4. axios 连接池**
- `checker.js` 每次检测都创建新的 axios 请求
- 未显式配置连接池，可能导致 TCP 连接堆积

#### 数据清理机制

```javascript
// storage.js:104-109
async function cleanupOldData() {
  const cutoff = moment().subtract(config.dataRetentionDays, 'days').toISOString();
  db.run('DELETE FROM check_results WHERE timestamp < ?', [cutoff]);
  dirty = true;
  saveDB();
}

// server.js:44-46 - 每小时执行一次
setInterval(() => {
  storage.cleanupOldData().catch(e => console.error('[Server] Cleanup error:', e.message));
}, 60 * 60 * 1000);
```

**清理策略**：
- 保留期限：`config.dataRetentionDays`（默认 30 天）
- 执行频率：每小时一次
- 清理对象：`check_results` 表中的过期记录

### 4.4 为什么选择 setInterval 而非 node-cron？

#### 对比分析

| 维度 | setInterval | node-cron |
|------|-------------|-----------|
| **精度** | 毫秒级 | 秒级（cron 表达式最小单位是秒） |
| **灵活性** | 任意毫秒间隔 | 必须符合 cron 表达式 |
| **依赖** | 原生，零依赖 | 需要安装 node-cron 包 |
| **复杂度** | 简单直观 | 需要学习 cron 语法 |
| **场景** | 固定间隔轮询 | 特定时间点执行 |

#### 选择 setInterval 的原因

1. **场景匹配**：健康检测是**固定间隔**的轮询任务，不是"每天凌晨 3 点"这种定时任务
2. **粒度更细**：支持 5 秒、7 秒等任意间隔，cron 表达式在小间隔场景下不方便
3. **更轻量**：不需要额外的依赖库，减少包体积和攻击面
4. **更直观**：`setInterval(fn, 30000)` 一眼就能看出是 30 秒一次
5. **动态调整方便**：服务配置变更时，直接 `clearInterval` + 重新 `setInterval` 即可

#### node-cron 的适用场景

虽然当前选择了 `setInterval`，但 `node-cron` 在以下场景更有优势：
- 需要在**特定时间点**执行任务（如 "每天 0 点清理"）
- 需要复杂的调度规则（如 "工作日 9-18 点每 5 分钟一次"）
- 需要支持 cron 表达式配置

> 💡 **补充发现**：package.json 中已经安装了 `node-cron`（`"node-cron": "^3.0.3"`），但代码中**并未使用**。这可能是历史遗留，或者为未来功能预留。

---

## 五、高负载瓶颈与优化空间

### 5.1 性能瓶颈分析

#### 瓶颈 1：SQLite 内存数据库单线程

**问题**：
- sql.js 是单线程的，所有 DB 操作串行执行
- 检测结果写入、状态查询、趋势计算都在同一个线程
- 服务数量增多时，DB 操作可能成为瓶颈

**影响评估**：
- SQLite 内存数据库读写很快，百级服务规模下不是问题
- 千级服务规模下可能需要考虑优化

**优化方向**：
1. 读写分离：检测结果写入走队列，查询走独立连接
2. 替换为真正的数据库（PostgreSQL/MySQL）
3. 增加缓存层（Redis）

#### 瓶颈 2：状态计算每次都查全量数据

**问题**：
```javascript
// status.js:103-118
async function getServiceSummary(serviceId, hours = config.trendWindowHours) {
  const to = moment().toISOString();
  const from = moment().subtract(hours, 'hours').toISOString();
  const results = await storage.checkResults.getByTimeRange(serviceId, from, to);
  // ... 重新计算所有统计 ...
}
```
每次检测完成后都要重新计算最近 24 小时的所有统计数据。

**优化方向**：
1. **增量计算**：维护滑动窗口的统计值，新结果进来时增量更新
2. **结果缓存**：短时间内重复查询直接返回缓存
3. **预聚合**：按小时/天预聚合统计数据

#### 瓶颈 3：每个服务独立定时器

**问题**：
- 100 个服务就有 100 个 `setInterval`
- 每个定时器独立触发，可能造成"惊群效应"（大量检测同时开始）
- 管理成本随服务数量线性增长

**优化方向**：
1. **时间轮算法**：用单个定时器 + 时间轮管理所有任务
2. **错峰调度**：将任务均匀分布在时间轴上，避免集中触发
3. **任务队列**：使用任务队列 + 工作池模式

#### 瓶颈 4：WebSocket 广播 O(n)

**问题**：
```javascript
// notifier.js:25-36
function broadcast(message) {
  const payload = JSON.stringify(message);
  for (const client of clients) {
    if (client.readyState === WebSocket.OPEN) {
      try {
        client.send(payload);
      } catch (err) {
        clients.delete(client);
      }
    }
  }
}
```
每次广播遍历所有客户端，时间复杂度 O(n)。

**影响评估**：
- 对于内部监控系统，客户端数量有限，通常不是问题
- 如果有大量客户端（上千），可以优化

**优化方向**：
1. 消息队列化，批量发送
2. 按房间/频道分组，只推送给订阅者
3. 使用 Redis Pub/Sub 做消息分发

#### 瓶颈 5：日志文件同步追加

**问题**：
```javascript
// storage.js:122
fs.appendFileSync(logFile, line, 'utf8');
```
使用 `appendFileSync` 同步写入，阻塞事件循环。

**优化方向**：
1. 改为异步写入 `fs.promises.appendFile`
2. 使用日志库（如 winston/pino）做缓冲写入
3. 日志批量化，减少 IO 次数

### 5.2 可观测性不足

**当前状态**：
- 只有控制台日志
- 没有内部指标监控（调度延迟、任务堆积数、检测耗时分布等）
- 没有告警机制

**改进建议**：
1. 暴露 Prometheus 指标接口
2. 添加内部统计：平均检测耗时、成功率、调度延迟
3. 集成告警（邮件、钉钉、企业微信等）

### 5.3 容错与弹性不足

**当前问题**：
- 数据库写入失败只打日志，不重试
- WebSocket 消息发送是"发出去就不管"
- 没有死信队列或失败重试机制

**改进建议**：
1. 关键操作增加重试机制
2. 失败任务进入死信队列
3. 断路器模式：连续失败的服务暂时降低检测频率

### 5.4 安全隐患

1. **HTTPS 检测忽略证书校验**（`checker.js:24`）：
   ```javascript
   httpsAgent = new https.Agent({ rejectUnauthorized: false })
   ```
   - 风险：可能被中间人攻击
   - 建议：增加配置选项，默认开启证书校验

2. **SSRF 风险**：
   - 用户可以输入任意 URL 进行检测
   - 可能被利用来探测内网
   - 建议：增加目标地址白名单或内网地址检测

### 5.5 优化路线图建议

| 优先级 | 优化项 | 预期收益 | 复杂度 |
|--------|--------|----------|--------|
| 🔴 高 | 任务去重叠机制 | 防止任务堆积，稳定性提升 | 低 |
| 🔴 高 | 内存使用监控与预警 | 提前发现内存问题 | 低 |
| 🟡 中 | 状态计算增量优化 | 状态查询性能提升 5-10 倍 | 中 |
| 🟡 中 | 日志异步写入 | 减少事件循环阻塞 | 低 |
| 🟡 中 | 错峰调度 | 避免检测集中爆发 | 中 |
| 🟢 低 | 时间轮算法 | 大规模服务下调度更高效 | 高 |
| 🟢 低 | 替换为专业数据库 | 支持更大规模数据 | 高 |

---

## 六、新人入职阅读指南

### 6.1 阅读顺序建议

按照**"从宏观到微观，从入口到核心"**的顺序，建议分 4 个阶段阅读：

#### 第一阶段：建立全局认知（30 分钟）

**目标**：了解项目是做什么的、整体架构、有哪些模块

| 顺序 | 文件 | 阅读重点 | 预计时间 |
|------|------|----------|----------|
| 1 | `package.json` | 项目依赖、启动命令 | 5 分钟 |
| 2 | `config.json` | 有哪些配置项 | 5 分钟 |
| 3 | `constants.js` | 有哪些常量、状态定义、类型定义 | 10 分钟 |
| 4 | `server.js` | 系统入口、各模块初始化顺序 | 10 分钟 |

**读完后你应该能回答**：
- 这个系统是做什么的？
- 有哪些主要模块？
- 系统启动时做了哪些事情？

#### 第二阶段：理解核心调度（1 小时）

**目标**：搞懂检测任务是怎么被调度和执行的

| 顺序 | 文件 | 阅读重点 | 预计时间 |
|------|------|----------|----------|
| 1 | `scheduler.js` | 定时器管理、`runCheck` 流程、状态变化检测 | 25 分钟 |
| 2 | `checker.js` | HTTP/TCP 检测的具体实现 | 20 分钟 |
| 3 | `utils.js` | 用到的工具函数（重点看 `clamp`） | 15 分钟 |

**读完后你应该能回答**：
- 一个检测任务是怎么被触发的？
- 检测结果包含哪些字段？
- 维护窗口是怎么影响检测结果的？

#### 第三阶段：数据与通知（1 小时）

**目标**：理解数据如何存储、如何推送到前端

| 顺序 | 文件 | 阅读重点 | 预计时间 |
|------|------|----------|----------|
| 1 | `storage.js` | 数据库表结构、CRUD 操作、持久化策略 | 25 分钟 |
| 2 | `status.js` | 可用性计算、趋势数据生成、状态判断逻辑 | 20 分钟 |
| 3 | `notifier.js` | WebSocket 管理、消息类型、广播机制 | 15 分钟 |

**读完后你应该能回答**：
- 检测结果存在哪里？怎么存的？
- 服务的可用率是怎么算出来的？
- 前端是怎么实时收到更新的？

#### 第四阶段：API 与交互（45 分钟）

**目标**：理解系统对外提供的接口

| 顺序 | 文件 | 阅读重点 | 预计时间 |
|------|------|----------|----------|
| 1 | `routes.js` | 所有 API 端点、请求处理流程 | 30 分钟 |
| 2 | `config.js` | 配置是怎么加载的 | 5 分钟 |
| 3 | 回顾 `scheduler.js` | 动态启停服务的逻辑 | 10 分钟 |

**读完后你应该能回答**：
- 怎么新增一个监控服务？
- 怎么修改检测间隔？
- API 调用后会触发哪些内部变化？

### 6.2 核心概念清单

阅读时重点关注这些概念：

| 概念 | 所在文件 | 说明 |
|------|----------|------|
| `serviceTimers` | scheduler.js:8 | 服务 ID 到定时器的映射 |
| `lastStatuses` | scheduler.js:9 | 服务上一次状态，用于检测变化 |
| `runCheck` | scheduler.js:20 | 单次检测的完整流程 |
| `dirty` 标志 | storage.js:9 | 数据库脏数据标志，控制落盘 |
| `check_results` 表 | storage.js:67 | 检测结果表 |
| `availability` 计算 | status.js:5 | 可用率计算公式 |
| `clients` Set | notifier.js:4 | WebSocket 客户端集合 |
| `broadcast` | notifier.js:25 | 消息广播函数 |

### 6.3 动手实践建议

看完代码后，可以通过以下操作加深理解：

1. **启动系统**：运行 `npm start`，观察控制台输出
2. **添加一个服务**：调用 `POST /api/services` 添加一个测试服务，观察日志
3. **修改检测间隔**：调用 `PUT /api/services/:id` 修改间隔，观察定时器是否重启
4. **查看数据库**：用 SQLite 工具打开 `data/monitor.db`，看看表里存了什么
5. **连接 WebSocket**：用 wscat 等工具连接 ws://localhost:3001/ws，观察消息推送
6. **模拟故障**：把服务目标改成一个不存在的地址，观察状态变化通知

### 6.4 常见疑问解答

**Q: 为什么用 sql.js 而不是原生 sqlite3？**
A: sql.js 是纯 JavaScript 实现，不需要编译原生模块，跨平台兼容性好，安装简单。适合中小规模应用。

**Q: 为什么检测结果既要存数据库又要写日志文件？**
A: 数据库用于查询和统计，日志文件用于备份和离线分析。双写提供了数据冗余。

**Q: setInterval 会不会不准？**
A: 会有毫秒级的漂移，但对于健康检测场景（秒级间隔）完全可以接受。如果需要更高精度，可以用 setTimeout 递归实现漂移校正。

**Q: 服务检测失败会不会影响其他服务？**
A: 不会。每个服务有独立的定时器和错误处理，单个服务的异常会被 catch 住，不会扩散。

**Q: 怎么保证数据不丢？**
A: 三层保障：1) 内存数据库操作很快；2) 每 5 秒落盘一次；3) 进程退出时强制落盘。极端情况下最多丢失最近 5 秒的数据。

---

## 附录：关键代码索引

### 调度引擎核心
- 定时器创建：`scheduler.js:65-84`
- 检测执行：`scheduler.js:20-63`
- 服务启停：`scheduler.js:65-99`
- 批量管理：`scheduler.js:101-126`

### 检测逻辑
- HTTP 检测：`checker.js:6-52`
- TCP 检测：`checker.js:54-110`
- 检测入口：`checker.js:112-136`

### 数据持久化
- 数据库初始化：`storage.js:29-102`
- 结果写入：`storage.js:189-204`
- 数据清理：`storage.js:104-109`
- 定时落盘：`storage.js:11-27`

### 实时推送
- WebSocket 初始化：`notifier.js:6-23`
- 消息广播：`notifier.js:25-36`
- 状态变更通知：`notifier.js:38-46`
- 新检测通知：`notifier.js:48-63`

### 状态计算
- 可用率计算：`status.js:5-25`
- 服务状态：`status.js:68-101`
- 趋势数据：`status.js:27-66`
- 摘要计算：`status.js:103-118`
