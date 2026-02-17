---
layout: post
title: Times-7 RFID 系统 - 完整架构设计文档
date: 2024-01-28
categories: [System Design, Architecture]
tags: [RFID, FastAPI, Supabase, React Native]
---

# Times-7 RFID 系统 - 完整架构设计文档

## 第一部分：系统架构概览

### 1.1 整体架构

```
数据来源层（RFID Reader / Simulators）
    ↓
处理层（FastAPI Gateway）
    ↓
存储层（Supabase PostgreSQL + Storage）
    ↓
展示层（Expo React Native 前端）
```

### 1.2 详细组件说明

#### 数据来源层：三种 Reader

| 来源 | 类型 | 接口 | 数据格式 |
|------|------|------|---------|
| **真实读取器** | Impinj Reader（硬件） | `GET {READER_BASE_URL}/data/stream` | NDJSON 流（tagInventory 事件） |
| **模拟器1** | Stream Simulator（软件） | `GET /data/stream` | NDJSON 流（读取 t7datastream.ndjson） |
| **模拟器2** | Terminal Simulator（命令行） | `POST /api/sim/reader/events {tagIds:[...]}` | HTTP POST + JSON body |

**关键点**：三种来源的数据最终都进入 FastAPI Gateway，格式统一为 `tagInventory` 事件。

---

#### 处理层：FastAPI Gateway

**启动流程**：
```
FastAPI 应用启动
    ↓
app.on_event("startup") 触发 run_reader_stream()
    ↓
连接 Reader 数据流（三选一或多个并行）
    ↓
实时处理 tagInventory 事件
```

**核心模块**：

1. **ActiveTags（3秒滑动窗口）**
   - 功能：维护最近 3 秒内的活跃标签
   - 实现：每次见到标签时 `sync_seen(tag_id, now)`
   - 作用：Dashboard 轮询这个 3 秒窗口内的标签列表

2. **TagInfoCache（24小时 TTL）**
   - 功能：缓存 IAS 查询结果
   - 原因：同一个 tag 的真伪认证结果在 24h 内基本不变，避免重复查询
   - 工作方式：
     - 标签第一次出现 → 调用 IAS Service → 缓存结果
     - 同一标签再次出现 → 直接从缓存读（不查 IAS）
     - 24h 后过期 → 下次出现时重新查 IAS

3. **IAS Lookup（真伪认证）**
   - 类型：可选 mock 或真实服务（由 IAS_MODE 环境变量控制）
   - 输入：tag_id（tidHex）
   - 输出：`{authentic: bool, brand: str, model: str, confidence: float, ...}`

4. **DB Writer（数据持久化）**
   - 触发条件：首次见到该 tag 时（或缓存过期后）
   - 操作：`upsert_latest_tag()` → 写入 Supabase data 表
   - 策略：按 tag_id 的 upsert（只保存每个 tag 的最新记录）

5. **对外 API 端点**
   - `GET /api/active-tags` → 返回 3 秒内的活跃标签列表 `[ScanResult, ...]`
   - `POST /api/sim/reader/events` → Terminal Simulator 注入事件（测试用）
   - `/debug/*` → 调试端点（可选，如清缓存等）

---

#### 存储层：Supabase（PostgreSQL + Storage）

**数据库表**：

| 表名 | 字段 | 说明 |
|------|------|------|
| **data** | id (PK), date, auth, info | 扫描结果快照。id=tag_id（upsert策略），只保存每个tag的最新记录 |
| **product_info** | tid (PK), epc, description, origin, produced_on | 商品元信息。前端新增/编辑商品时维护此表 |
| **product_photo** | id, tid (FK), photo_url, created_at | 商品图片元数据。一个商品可多张图片，记录 Storage 的 public URL |

**对象存储**（Storage Bucket）：

| Bucket 名 | 路径结构 | 用途 |
|-----------|---------|------|
| **product-photos** | `{tid}/{timestamp}.jpg` | 存储商品图片实体，生成 public_url 给前端使用 |

---

#### 展示层：Expo React Native 前端

**5 个主要界面**：

1. **Dashboard（主页）**
   - 显示：实时检测到的标签列表（来自 Gateway /api/active-tags）
   - 功能：
     - 轮询 Gateway（每 1 秒）获取活跃标签
     - 对每个 tag_id 查询 product_info 判断是否已注册
     - 若已注册则显示商品信息，若未注册显示"未注册"
     - 点击标签卡片可打开 ViewItem 或 NewProduct 模态框

2. **Logs 页面（历史记录）**
   - 显示：所有 tag 的最新扫描记录（来自 Supabase data 表）
   - 查询：`SELECT * FROM data ORDER BY date DESC`
   - 点击某条记录可打开 ViewItem 模态框

3. **Search 页面（搜索）**
   - 输入：tid 或 epc（精确匹配）
   - 查询方式：**直接查 Supabase product_info（不走 Gateway）**
     1. 先查 `WHERE tid == input`
     2. 若未找到，再查 `WHERE epc == input`
   - 展示：tid、epc、description、origin、produced_on 五项

4. **NewProduct Modal（新增商品弹窗）**
   - 触发：Dashboard 显示未注册的标签时
   - 操作：
     1. 写入 product_info（tid, epc, description, origin, produced_on）
     2. 可选：上传图片到 Storage bucket → 获得 public_url
     3. 可选：插入 product_photo 记录（tid, photo_url）

5. **ViewItem Modal（查看/编辑商品弹窗）**
   - 触发：点击已注册的标签或 Logs 页面的记录时
   - 操作：
     - 查询 product_info 读取商品数据
     - 查询最新一张 product_photo（`WHERE tid=... ORDER BY created_at DESC LIMIT 1`）
     - 可编辑任何字段 → 更新 product_info
     - 可上传新图片 → 插入新的 product_photo 记录

---

### 1.3 数据流向图（文字版）

```
Reader 数据流
    ↓
FastAPI Gateway
    ├─ ActiveTags 模块 (维护 3 秒窗口)
    ├─ TagInfoCache 模块 (24h 缓存)
    │   └─ 首次查 IAS Service
    │   └─ 结果存入缓存
    ├─ DB Writer (首次或缓存过期时)
    │   └─ upsert 到 Supabase data 表
    └─ API 端点
        └─ GET /api/active-tags (供前端轮询)
                    ↓
前端 Dashboard
    ├─ 每 1s 轮询 Gateway /api/active-tags
    ├─ 获得最新 3s 活跃标签
    ├─ 对每个 tag_id 查 Supabase product_info
    └─ 渲染标签卡片（已注册/未注册）

前端 Logs 页面
    └─ 直接查 Supabase data 表 (ORDER BY date DESC)
        └─ 显示所有 tag 的最新记录

前端 Search 页面
    └─ 直接查 Supabase product_info 表 (精确匹配)
        └─ 显示商品详情

前端 NewProduct/ViewItem Modal
    └─ 查/写 Supabase product_info
    └─ 上传图片到 Storage
    └─ 写 product_photo 记录
```

---

## 第二部分：核心数据流

### 2.1 数据流 A：实时识别流程（Dashboard → Detected Items）

#### A1. 时间线

```
时刻 0ms   : Reader 检测到标签
     0-100ms: 数据通过网络流到 Gateway
     100-200ms: Gateway 更新 ActiveTags
     200-500ms: 缓存检查 + IAS 查询（首次） / 或直接缓存读（重复）
     500-800ms: DB 写入 Supabase data 表
     1000ms  : 前端轮询 GET /api/active-tags
     1000-1500ms: 前端查询 product_info 判断注册状态
     1500ms  : Dashboard 显示标签结果

总延迟：约 1-2 秒（用户看到识别结果）
```

#### A2. 详细流程

**步骤 1：标签被 Reader 检测**
```
Reader（Impinj / Simulator）
    ↓
发送 NDJSON 事件流
    ↓
每个事件包含：
  {
    "eventType": "tagInventory",
    "tagInventoryEvent": {
      "tidHex": "3015E2C1A000000000000001",
      "epcHex": "3034257BF411E4000000001E",
      "rssi": -45
    }
  }
```

**步骤 2：Gateway 接收并处理**
```
FastAPI 的 run_reader_stream() 函数
    ↓
监听 Reader 数据流（循环读 NDJSON）
    ↓
过滤 eventType == "tagInventory"
    ↓
提取 tag_id = tidHex
    ↓
同步到 ActiveTags（标记为"刚刚见过"）
```

**步骤 3：检查 TagInfoCache**
```
if tag_id 在缓存中且未过期:
    ├─ 缓存命中 ✓
    ├─ 读取缓存的 {auth, info}
    └─ 仅更新 ActiveTags 的 last_seen 时间（不查 IAS，不写 DB）

else:
    ├─ 缓存未命中（首次或已过期）
    ├─ 调用 IAS Service
    │  └─ 输入：tag_id
    │  └─ 输出：{authentic: bool, info: {...}}
    ├─ 存入 TagInfoCache（TTL 24h）
    └─ 写入 Supabase data 表
       └─ 操作：UPSERT data (id, date, auth, info) ON CONFLICT(id) DO UPDATE
```

**步骤 4：前端轮询**
```
前端 Dashboard 每 1 秒：
    ↓
GET /api/active-tags
    ↓
Gateway 返回最近 3 秒的活跃标签列表
    [
      {
        "id": "3015E2C1A000000000000001",
        "date": "2024-01-28T10:30:45Z",
        "auth": true,
        "info": {...}
      },
      ...
    ]
    ↓
前端对每个标签查询 product_info
    ├─ 若存在 → 标签卡片显示商品信息 + "已注册" 标记
    └─ 若不存在 → 标签卡片显示 "未注册" + [注册] 按钮
    ↓
渲染到 Dashboard
```

**步骤 5：用户交互（可选）**
```
用户点击"未注册"标签
    ↓
打开 NewProduct Modal
    ↓
填表 + 保存
    ↓
写入 product_info
    ↓
(可选) 上传图片 → Storage → product_photo
    ↓
下一次轮询时，这个标签就显示为"已注册"
```

---

### 2.2 数据流 B：产品注册/编辑流程（NewProduct / ViewItem Modal）

#### B1. 新增商品流程（NewProduct Modal）

```
场景：Dashboard 发现未注册的标签，用户点击"注册"

流程：
  ┌─────────────────────────────────────────┐
  │ 1. Modal 打开                            │
  │    显示表单：                            │
  │    - tid (只读，自动填入)                 │
  │    - epc (手动输入)                      │
  │    - description (手动输入)              │
  │    - origin (手动输入)                   │
  │    - produced_on (日期选择)              │
  │    - [选择图片] (可选)                   │
  └─────────────────────────────────────────┘
                    ↓
  ┌─────────────────────────────────────────┐
  │ 2. 用户填表                              │
  │    - 输入各字段                          │
  │    - (可选) 选择商品图片                  │
  └─────────────────────────────────────────┘
                    ↓
  ┌─────────────────────────────────────────┐
  │ 3. 点击 [保存]                           │
  │    前端验证输入不为空                     │
  └─────────────────────────────────────────┘
                    ↓
  ┌─────────────────────────────────────────┐
  │ 4. upsert product_info                  │
  │    INSERT INTO product_info (           │
  │      tid, epc, description, origin,     │
  │      produced_on                        │
  │    ) VALUES (...) ON CONFLICT DO UPDATE │
  │                                         │
  │    → Supabase 返回成功                   │
  └─────────────────────────────────────────┘
                    ↓
  ┌─────────────────────────────────────────┐
  │ 5. (若用户选了图片)                      │
  │    上传到 Supabase Storage              │
  │    - 路径：product-photos/{tid}/{ts}.jpg│
  │    - 获得 public_url                    │
  └─────────────────────────────────────────┘
                    ↓
  ┌─────────────────────────────────────────┐
  │ 6. (若成功上传)                          │
  │    插入 product_photo 记录               │
  │    INSERT INTO product_photo (          │
  │      tid, photo_url, created_at         │
  │    ) VALUES (...)                       │
  └─────────────────────────────────────────┘
                    ↓
  ┌─────────────────────────────────────────┐
  │ 7. Modal 关闭                            │
  │    Dashboard 刷新或下次轮询时，           │
  │    该标签会显示"已注册"+ 商品信息          │
  └─────────────────────────────────────────┘
```

#### B2. 查看/编辑商品流程（ViewItem Modal）

```
场景 1：用户点击 Dashboard 中的已注册标签
场景 2：用户在 Logs 页面点击某条记录

流程：
  ┌─────────────────────────────────────────┐
  │ 1. Modal 打开时，自动加载                │
  │                                          │
  │    查询 1：                              │
  │    SELECT * FROM product_info           │
  │    WHERE tid = {标签ID}                 │
  │    → 返回：tid, epc, description,       │
  │      origin, produced_on                │
  │                                         │
  │    查询 2：                              │
  │    SELECT * FROM product_photo          │
  │    WHERE tid = {标签ID}                  │
  │    ORDER BY created_at DESC LIMIT 1     │
  │    → 返回：最新一张图片的 photo_url       │
  └─────────────────────────────────────────┘
                    ↓
  ┌─────────────────────────────────────────┐
  │ 2. Modal 显示商品详情                    │
  │    - tid, epc, description, origin      │
  │    - produced_on                        │
  │    - 最新一张图片                        │
  │    - [编辑] 按钮                         │
  │    - [上传新图片] 按钮                   │
  │    - [删除商品] 按钮 (可选)              │
  └─────────────────────────────────────────┘
                    ↓
         用户选择以下操作之一：
                    ↓
  ┌──────────────────────────┬──────────────────────────┐
  │ 操作 A: 编辑字段          │ 操作 B: 上传新图片        │
  ├──────────────────────────┼──────────────────────────┤
  │ 点击某个字段修改          │ 点击 [上传新图片]          │
  │ (如 epc/description)     │                          │
  │       ↓                  │ 选择图片文件              │
  │ UPDATE product_info      │       ↓                  │
  │ SET {字段} = ...         │ POST 到 Storage bucket   │
  │ WHERE tid = ...          │ product-photos/{tid}     │
  │       ↓                  │       ↓                  │
  │ Supabase 更新成功        │ 获得 public_url           │
  │       ↓                  │       ↓                  │
  │ Modal 刷新显示更新内容    │ INSERT product_photo     │
  │                          │ (tid, photo_url, ...)    │
  │                          │       ↓                  │
  │                          │ Modal 显示新图片          │
  └──────────────────────────┴──────────────────────────┘
```

---

### 2.3 数据流 C：历史记录页面（Logs Page）

```
场景：用户打开 Logs 页面查看历史扫描记录

流程：
  ┌────────────────────────────────────────┐
  │ 1. LogsPage 组件加载                   │
  │    触发 useEffect 或 onMount 事件      │
  └────────────────────────────────────────┘
                    ↓
  ┌────────────────────────────────────────┐
  │ 2. 查询 Supabase data 表                │
  │                                        │
  │    SELECT *                            │
  │    FROM data                           │
  │    ORDER BY date DESC                  │
  │    LIMIT 100                           │
  │                                        │
  │    返回：最新 100 条扫描记录             │
  │    (每条是某个 tag 的最新扫描快照)       │
  └────────────────────────────────────────┘
                    ↓
  ┌────────────────────────────────────────┐
  │ 3. 前端渲染列表                         │
  │                                        │
  │    对每一行显示：                       │
  │    - tid (标签 ID)                     │
  │    - date (最后扫描时间)                │
  │    - auth (✓ 真品 / ✗ 可疑)           │
  │    - info (IAS 返回的完整信息)        │
  │                                        │
  │    列表项可点击                        │
  └────────────────────────────────────────┘
                    ↓
  ┌────────────────────────────────────────┐
  │ 4. 用户交互 (可选)                      │
  │                                        │
  │    点击某条记录                         │
  │         ↓                              │
  │    打开 ViewItem Modal                 │
  │    (查看/编辑该标签对应的商品信息)       │
  │    (见数据流 B2)                       │
  └────────────────────────────────────────┘

重要说明：
  data 表 是 upsert 模式（只保存最新）
  所以 Logs 页面展示的不是完整历史轨迹
  而是每个 tag 的最新扫描记录

  如果需要完整历史，应该额外建立
  scan_history 表，每次都 INSERT（不 upsert）
```

---

### 2.4 数据流 D：搜索功能（Search Page）

```
场景：用户在 Search 页面输入 tid 或 epc，查找商品

流程：
  ┌──────────────────────────────────────┐
  │ 1. SearchPage 显示搜索框              │
  │    用户输入查询词 q                   │
  │    (可以是 tid 或 epc)                │
  └──────────────────────────────────────┘
                    ↓
  ┌──────────────────────────────────────┐
  │ 2. 用户点击 [搜索]                    │
  │    前端验证输入不为空                 │
  └──────────────────────────────────────┘
                    ↓
  ┌──────────────────────────────────────┐
  │ 3. 查询 1：精确匹配 tid               │
  │                                      │
  │    SELECT tid, epc, description,     │
  │           origin, produced_on        │
  │    FROM product_info                 │
  │    WHERE tid = q                     │
  │    (精确匹配，不是模糊搜索)            │
  │                                      │
  │    [找到？]                           │
  │    ├─ 是 → 跳到步骤 5 显示结果         │
  │    └─ 否 → 继续步骤 4                 │
  └──────────────────────────────────────┘
                    ↓
  ┌──────────────────────────────────────┐
  │ 4. 查询 2：精确匹配 epc               │
  │                                      │
  │    SELECT tid, epc, description,     │
  │           origin, produced_on        │
  │    FROM product_info                 │
  │    WHERE epc = q                     │
  │    (精确匹配)                         │
  │                                      │
  │    [找到？]                          │
  │    ├─ 是 → 跳到步骤 5 显示结果        │
  │    └─ 否 → 步骤 6 显示"未找到"        │
  └──────────────────────────────────────┘
                    ↓
  ┌──────────────────────────────────────┐
  │ 5. 显示搜索结果卡片                   │
  │                                      │
  │    展示五项信息：                     │
  │    ✓ tid                             │
  │    ✓ epc                             │
  │    ✓ description                     │
  │    ✓ origin                          │
  │    ✓ produced_on                     │
  │                                      │
  │    用户可点击卡片打开 ViewItem Modal   │
  │    进行编辑                           │
  └──────────────────────────────────────┘
                    ↓
                  结束

  ┌──────────────────────────────────────┐
  │ 6. 显示 "Item not found"              │
  │                                      │
  │    tid 和 epc 都查不到                │
  │    提示用户重新输入或到 Dashboard      │
  │    通过扫描来注册新商品                │
  └──────────────────────────────────────┘

关键特点：
  ✓ Search 直接查 Supabase product_info（不走 Gateway）
  ✓ 精确匹配（不支持模糊搜索）
  ✓ 先查 tid，再查 epc
  ✓ 返回完整的 5 项商品信息
```

---

## 第三部分：数据库表结构详解

### 3.1 data 表（扫描结果快照表）

**用途**：存储每个标签的最新扫描结果，后端写入，前端 Logs 页面读取

**表结构**：
```
data (
  id         VARCHAR(255) PRIMARY KEY,      -- tag_id (tidHex)
  date       TIMESTAMP NOT NULL,             -- 最后一次扫描时间
  auth       BOOLEAN NOT NULL,               -- IAS 认证结果
  info       JSONB NOT NULL                  -- IAS 返回的完整信息对象
)
```

**字段详解**：

| 字段 | 数据类型 | 说明 | 示例 |
|------|---------|------|------|
| id | VARCHAR(PK) | 标签的 tidHex，作为主键 | `"3015E2C1A000000000000001"` |
| date | TIMESTAMP | 最后一次扫描时间（自动更新） | `2024-01-28T10:30:45.123Z` |
| auth | BOOLEAN | IAS 鉴定结果（true=真品，false=假品） | `true` 或 `false` |
| info | JSONB | IAS 服务返回的完整信息 | `{"authentic": true, "brand": "Nike", "confidence": 0.99, ...}` |

**关键特性**：
- **upsert 策略**：按 `id`（tag_id）为主键，新标签执行 INSERT，已存在则 UPDATE
- **只保存最新**：每个 tag 只有一条记录（最新的），不是完整历史
- **用途**：Logs 页面 `SELECT * FROM data ORDER BY date DESC` 查询

---

### 3.2 product_info 表（商品元信息表）

**用途**：存储注册过的商品信息，前端 NewProduct/ViewItem Modal 维护

**表结构**：
```
product_info (
  tid          VARCHAR(255) PRIMARY KEY,    -- 标签 ID
  epc          VARCHAR(255) UNIQUE,         -- EPC 码
  description  TEXT,                        -- 商品描述
  origin       VARCHAR(255),                -- 产地
  produced_on  DATE,                        -- 生产日期
  created_at   TIMESTAMP DEFAULT NOW(),     -- 创建时间
  updated_at   TIMESTAMP DEFAULT NOW()      -- 更新时间
)
```

**字段详解**：

| 字段 | 数据类型 | 说明 | 示例 |
|------|---------|------|------|
| tid | VARCHAR(PK) | 标签 ID（与 data 表的 id 关联） | `"3015E2C1A000000000000001"` |
| epc | VARCHAR(UNIQUE) | EPC 码（唯一，用于 Search 精确查询） | `"3034257BF411E4000000001E"` |
| description | TEXT | 商品描述 | `"Nike Air Max 2024"` |
| origin | VARCHAR | 产地 | `"Vietnam"` |
| produced_on | DATE | 生产日期 | `2024-01-15` |
| created_at | TIMESTAMP | 创建时间 | `2024-01-28T10:30:00Z` |
| updated_at | TIMESTAMP | 最后更新时间 | `2024-01-28T11:45:30Z` |

**查询示例**：

```sql
-- Dashboard 判断标签是否已注册
SELECT * FROM product_info WHERE tid = 'tag_id' LIMIT 1;

-- Search 精确查询（先按 tid）
SELECT tid, epc, description, origin, produced_on 
FROM product_info WHERE tid = q LIMIT 1;

-- Search 精确查询（后按 epc）
SELECT tid, epc, description, origin, produced_on 
FROM product_info WHERE epc = q LIMIT 1;

-- ViewItem Modal 读取商品信息
SELECT * FROM product_info WHERE tid = 'tag_id';

-- 编辑时更新
UPDATE product_info SET epc = '...', description = '...' WHERE tid = 'tag_id';
```

---

### 3.3 product_photo 表（商品图片元数据表）

**用途**：记录商品的图片 URL 和元数据，关联 product_info

**表结构**：
```
product_photo (
  id         SERIAL PRIMARY KEY,            -- 自增主键
  tid        VARCHAR(255) NOT NULL,         -- 标签 ID（FK）
  photo_url  VARCHAR(500) NOT NULL,         -- Storage 公开 URL
  created_at TIMESTAMP DEFAULT NOW()        -- 上传时间
  
  FOREIGN KEY (tid) REFERENCES product_info(tid) ON DELETE CASCADE
)
```

**字段详解**：

| 字段 | 数据类型 | 说明 | 示例 |
|------|---------|------|------|
| id | SERIAL(PK) | 自增主键 | `1, 2, 3, ...` |
| tid | VARCHAR(FK) | 标签 ID（关联 product_info） | `"3015E2C1A000000000000001"` |
| photo_url | VARCHAR | Supabase Storage 生成的公开 URL | `"https://xyz.supabase.co/storage/v1/object/public/product-photos/3015E2C1A.../1705049445000.jpg"` |
| created_at | TIMESTAMP | 上传时间 | `2024-01-28T10:30:00Z` |

**关键特性**：
- **一对多关系**：一个商品（tid）可有多张图片
- **级联删除**：删除 product_info 记录会自动删除关联的 product_photo 记录
- **查询最新图片**：`SELECT * FROM product_photo WHERE tid = ? ORDER BY created_at DESC LIMIT 1`

**查询示例**：

```sql
-- ViewItem Modal 获取最新一张图片
SELECT photo_url FROM product_photo 
WHERE tid = 'tag_id' 
ORDER BY created_at DESC LIMIT 1;

-- 获取某商品的所有图片（从旧到新）
SELECT * FROM product_photo 
WHERE tid = 'tag_id' 
ORDER BY created_at ASC;

-- 插入新图片
INSERT INTO product_photo (tid, photo_url) 
VALUES ('tag_id', 'https://...');

-- 删除某张图片
DELETE FROM product_photo WHERE id = photo_id;
```

---

### 3.4 Supabase Storage - product-photos Bucket

**用途**：存储商品图片实体文件

**路径结构**：
```
product-photos/
├─ 3015E2C1A000000000000001/
│  ├─ 1705049445000.jpg        (timestamp in milliseconds)
│  ├─ 1705049512000.jpg
│  └─ ...
├─ 3015E2C1A000000000000002/
│  ├─ 1705049600000.jpg
│  └─ ...
└─ ...
```

**公开 URL 示例**：
```
https://xyz.supabase.co/storage/v1/object/public/product-photos/3015E2C1A000000000000001/1705049445000.jpg
```

**权限设置**：
- 读取：**公开**（任何人可访问，前端直接使用 URL）
- 上传：**需认证**（只有登录用户可上传，通过 supabase-js SDK）

---

## 第四部分：完整 API 参考

### 4.1 Gateway API（FastAPI）

#### 端点 1：GET /api/active-tags

**功能**：获取最近 3 秒内的活跃标签列表

**请求**：
```http
GET http://localhost:3000/api/active-tags
```

**响应** (200 OK)：
```json
[
  {
    "id": "3015E2C1A000000000000001",
    "date": "2024-01-28T10:30:45.123Z",
    "auth": true,
    "info": {
      "authentic": true,
      "brand": "Nike",
      "model": "Air Max 2024",
      "confidence": 0.99
    }
  },
  {
    "id": "3015E2C1A000000000000002",
    "date": "2024-01-28T10:30:47.456Z",
    "auth": false,
    "info": {
      "authentic": false,
      "brand": "Unknown",
      "confidence": 0.45
    }
  }
]
```

**使用场景**：前端 Dashboard 每 1 秒轮询

**前端代码**：
```javascript
const response = await fetch('http://localhost:3000/api/active-tags');
const activeTags = await response.json();
```

---

#### 端点 2：POST /api/sim/reader/events

**功能**：Terminal Simulator 注入模拟标签事件（仅测试用）

**请求**：
```http
POST http://localhost:3000/api/sim/reader/events
Content-Type: application/json

{
  "tagIds": [
    "3015E2C1A000000000000001",
    "3015E2C1A000000000000002",
    "3015E2C1A000000000000003"
  ]
}
```

**响应** (200 OK)：
```json
{
  "status": "ok",
  "count": 3,
  "message": "3 tags injected successfully"
}
```

**测试命令**：
```bash
curl -X POST http://localhost:3000/api/sim/reader/events \
  -H "Content-Type: application/json" \
  -d '{"tagIds": ["tid1", "tid2", "tid3"]}'
```

---

### 4.2 Supabase JavaScript SDK 常用查询

#### 查询 1：获取商品是否已注册

```javascript
const { data, error } = await supabase
  .from('product_info')
  .select('*')
  .eq('tid', tag_id)
  .single();

if (data) {
  // 已注册
  console.log(`商品: ${data.epc}, ${data.description}`);
} else {
  // 未注册
  console.log('该标签未注册');
}
```

---

#### 查询 2：新增或更新商品信息

```javascript
const { data, error } = await supabase
  .from('product_info')
  .upsert({
    tid: tag_id,
    epc: 'EPC_CODE',
    description: 'Nike Air Max 2024',
    origin: 'Vietnam',
    produced_on: '2024-01-15'
  })
  .select();

if (error) {
  console.error('保存失败:', error);
} else {
  console.log('保存成功:', data);
}
```

---

#### 查询 3：上传图片到 Storage

```javascript
const file = selectedImage;  // File 对象
const fileName = `${tag_id}/${Date.now()}.jpg`;

const { data, error } = await supabase.storage
  .from('product-photos')
  .upload(fileName, file);

if (error) {
  console.error('上传失败:', error);
} else {
  // 获取公开 URL
  const { data: urlData } = supabase.storage
    .from('product-photos')
    .getPublicUrl(fileName);
  
  const public_url = urlData.publicUrl;
  console.log('图片 URL:', public_url);
  
  // 插入 product_photo 记录
  await supabase
    .from('product_photo')
    .insert({
      tid: tag_id,
      photo_url: public_url
    });
}
```

---

#### 查询 4：获取商品历史记录（Logs 页面）

```javascript
const { data, error } = await supabase
  .from('data')
  .select('*')
  .order('date', { ascending: false })
  .limit(100);

if (error) {
  console.error('查询失败:', error);
} else {
  // data 是历史记录数组
  data.forEach(record => {
    console.log(
      `${record.id}: ${record.auth ? '✓ 真品' : '✗ 假品'} @ ${record.date}`
    );
  });
}
```

---

#### 查询 5：搜索商品（Search 页面）

```javascript
async function searchProduct(query) {
  // 首先按 tid 精确查询
  let { data, error } = await supabase
    .from('product_info')
    .select('tid, epc, description, origin, produced_on')
    .eq('tid', query)
    .single();
  
  // 如果没找到，按 epc 精确查询
  if (!data && !error) {
    ({ data, error } = await supabase
      .from('product_info')
      .select('tid, epc, description, origin, produced_on')
      .eq('epc', query)
      .single());
  }
  
  return data;  // 找到返回对象，未找到返回 null
}

// 使用
const result = await searchProduct('3015E2C1A000000000000001');
if (result) {
  console.log('找到商品:', result);
} else {
  console.log('Item not found');
}
```

---

#### 查询 6：获取商品的最新图片（ViewItem Modal）

```javascript
const { data: photo, error } = await supabase
  .from('product_photo')
  .select('photo_url')
  .eq('tid', tag_id)
  .order('created_at', { ascending: false })
  .limit(1)
  .single();

if (photo) {
  // 显示图片
  console.log('最新图片:', photo.photo_url);
  // 
} else {
  // 没有图片
  console.log('该商品还没有图片');
}
```

---

## 第五部分：关键设计决策

### 5.1 ActiveTags 窗口为什么是 3 秒？

```
候选方案对比：

1 秒窗口：
  ❌ 标签容易漏掉
  ❌ RFID 识别有网络延迟，1s 太短容易闪烁
  ❌ 读取器扫描频率不稳定

3 秒窗口：
  ✓ 足以覆盖识别延迟
  ✓ 用户看到的是相对稳定的标签列表
  ✓ 业界 RFID 阅读器的标准推荐
  ✓ 前端 1s 轮询也够快

5+ 秒窗口：
  ❌ 已离开的标签还在显示，用户困惑
  ❌ 实时性太差
```

---

### 5.2 TagInfoCache TTL 为什么是 24 小时？

```
为什么要缓存？
  IAS Service 查询成本高：
  - 网络往返延迟
  - 远程服务器处理时间
  - 可能调用数据库

同一标签的真伪认证结果在 24h 内基本不变：
  - 一旦认证过，结果很稳定
  - 不会突然从真变假或反过来
  - 24h 是合理的信任周期

24 小时的权衡：
  ✓ 避免重复查询，提高性能
  ✓ 减轻 IAS 服务负担
  ✓ 用户体验快速（缓存命中）
  ⚠ 如果标签信息在 24h 内变化，需要手动清缓存
```

---

### 5.3 data 表为什么用 upsert 而不是 insert？

```
如果每次都 INSERT：

问题：
  ❌ data 表行数爆炸
     （比如 1000 个标签，扫描 10 次 = 10000 行）
  ❌ 存储空间浪费
  ❌ Logs 页面查询变慢（SELECT 需要扫描更多行）
  ❌ 无法快速了解"当前状态"

用 upsert（按 tag_id）：

优点：
  ✓ data 表只保存最新数据
  ✓ 行数 = 标签总数（可控）
  ✓ 查询快速（每个 tag 只有 1 行）
  ✓ Logs 页面性能好

代价：
  ⚠ 丢失完整扫描历史
     （无法回看"这个标签什么时候被扫过"的完整轨迹）

如果需要完整历史：
  方案：建立额外的 scan_history 表
  - scan_history：每次都 INSERT（不 upsert）
  - data：每次 UPSERT（只保存最新）
  - Logs 可查 scan_history 获得完整历史
```

---

### 5.4 Search 为什么不走 Gateway？

```
如果走 Gateway：

问题：
  ❌ Gateway 没有 product_info 数据（只有扫描数据 data）
  ❌ 需要额外开发 /api/search 端点
  ❌ 增加 Gateway 的职责（混合数据源）
  ❌ 多一层网络延迟（走 API 而不是直查）

直查 Supabase product_info：

优点：
  ✓ product_info 就是前端维护的数据，逻辑清晰
  ✓ supabase-js SDK 一行代码解决
  ✓ 网络路径最短，响应快
  ✓ Gateway 专注扫描数据处理

缺点：
  ⚠ 前端直接访问数据库
  ⚠ 需要正确的 RLS（Row Level Security）策略
  ⚠ 前端有访问整个 product_info 表的权限
```

---

## 第六部分：用于报告的总结段落

### 完整段落（可直接用于项目报告）

> **系统架构**
>
> 本系统采用 FastAPI Gateway + Supabase（Postgres + Storage）+ Expo React Native 的三层架构。
>
> **数据采集与处理**：Reader（真实 Impinj 硬件、Stream Simulator 或 Terminal Simulator）通过 NDJSON 数据流或 HTTP POST 将 tagInventory 事件输入 Gateway。Gateway 使用 ActiveTags（3 秒滑动窗口）维护短时活跃标签，并通过 TagInfoCache（24 小时 TTL）缓存 IAS 鉴定结果以减少重复请求。首次出现或缓存过期的标签会调用 IAS Service 获取真伪认证结果，随后写入 Supabase 的 data 表（按 tag_id 的 upsert 策略，只保存每个标签的最新记录）。
>
> **前端展示与交互**：前端 Dashboard 每 1 秒轮询 Gateway 的 `/api/active-tags` 获取最新活跃标签，并通过查询 Supabase 的 product_info 判断是否已注册商品。若未注册，显示"未注册"提示，用户可通过 NewProduct Modal 填写 tid/epc/description/origin/produced_on 并上传商品图片（存储到 Supabase Storage 的 product-photos bucket）。若已注册，显示商品信息，用户可通过 ViewItem Modal 查看或编辑商品详情、上传新图片（插入 product_photo 记录）。Logs 页面直接查询 Supabase data 表按 date 倒序展示所有标签的最新扫描记录；Search 页面直接查询 product_info 表，支持按 tid 或 epc 精确检索并展示 tid、epc、description、origin、produced_on 五项信息。

---

## 总结清单

```
系统组件：
  ✓ 三种 Reader（真实 Impinj + 两个 Simulator）
  ✓ FastAPI Gateway（ActiveTags + TagInfoCache + IAS + DB Writer）
  ✓ Supabase 数据库（data / product_info / product_photo 表）
  ✓ Supabase Storage（product-photos bucket）
  ✓ Expo React Native 前端（5 个主要界面）

核心功能：
  ✓ 实时识别（3s 窗口 + 1s 前端轮询）
  ✓ 商品注册（NewProduct Modal + 图片上传）
  ✓ 商品编辑（ViewItem Modal + 图片管理）
  ✓ 历史记录（Logs 页面 + 按时间倒序）
  ✓ 精确搜索（Search 页面 + tid/epc）

关键决策：
  ✓ ActiveTags 3 秒 - 平衡识别稳定性和实时性
  ✓ TagInfoCache 24h - 避免重复 IAS 查询
  ✓ data 表 upsert - 保持表大小可控
  ✓ Search 直查 Supabase - 简化架构，快速响应
```

---

## 声明: 

本系统为VUW学校小组作业部分, 整体项目由小组成员Egan, Tane, Rebekka, Laura本人, 四人共同完成. 此文档仅供个人学习之用, 并借助Claude+ChatGPT完成, 并非独创作品.
本人主要贡献部分为前端连接数据库搜索部分. 欢迎阅览学习, 请勿转载!!

