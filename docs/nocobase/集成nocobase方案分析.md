# Inventory 集成 NocoBase 方案分析

## 概述

本文档分析了 Inventory 项目与 NocoBase 集成的五种可行方案，基于项目现有的集成架构（Airtable、Snipe-IT）和 NocoBase 的 REST API 能力。

---

## 方案一：双向同步集成（推荐 ⭐⭐⭐⭐⭐）

### 适用场景
希望在 NocoBase 中管理资产数据，同时保持移动应用的完整功能

### 架构设计
```
Inventory App (CouchDB) ←→ 双向同步 ←→ NocoBase (MySQL/PostgreSQL)
```

### 实现方式
1. 创建 `packages/integration-nocobase` 包
2. 参考 Airtable 集成模式，实现双向数据同步
3. 映射数据模型：
   - `collection` → NocoBase Collection（数据表）
   - `item` → NocoBase Records（记录）
   - `image` → NocoBase 附件字段

### 技术要点

#### 1. 推送到 NocoBase
```typescript
// 创建/更新记录
POST /api/collections/{name}:create
POST /api/collections/{name}:update

// 批量更新
POST /api/collections/{name}:update?filter[id][$in]=1,2,3
```

#### 2. 从 NocoBase 拉取
```typescript
// 获取记录列表
GET /api/collections/{name}:list

// 过滤更新时间
GET /api/collections/{name}:list?filter[updated_at][$gt]=lastSyncTime

// 分页获取
GET /api/collections/{name}:list?page=1&pageSize=100
```

#### 3. 图片同步
```typescript
// 上传附件
POST /api/attachments:create

// 关联到记录字段
PATCH /api/collections/{name}:update?filterByTk={id}
{
  "image_field": [{ "id": attachmentId }]
}
```

### 核心同步逻辑

参考 Airtable 集成的实现：

```typescript
// packages/integration-nocobase/lib/syncWithNocoBase.ts

export default async function* syncWithNocoBase(
  {
    integrationId,
    secrets,
    fullSync,
  }: {
    integrationId: string;
    secrets: { nocobase_api_token: string; nocobase_base_url: string };
    fullSync?: boolean;
  },
  context: SyncContext
) {
  // 1. 初始化 NocoBase API 客户端
  const api = new NocoBaseAPI({
    fetch,
    baseUrl: secrets.nocobase_base_url,
    apiToken: secrets.nocobase_api_token,
  });

  // 2. 同步 Collections
  yield* syncCollections(api, context);

  // 3. 同步 Items
  yield* syncItems(api, context);

  // 4. 同步 Images
  yield* syncImages(api, context);
}
```

### 优势
- ✅ 充分利用 NocoBase 的低代码界面和权限管理
- ✅ 支持多用户协作和审批流程
- ✅ 保留移动端 RFID 扫描能力
- ✅ 数据双向同步，灵活性高
- ✅ 可利用 NocoBase 的工作流、自动化功能

### 劣势
- ❌ 需要处理数据冲突（最后修改时间优先）
- ❌ 开发工作量较大（预计 2-3 周）
- ❌ 需要维护同步逻辑

### 开发任务清单
- [ ] 创建 `packages/integration-nocobase` 包
- [ ] 实现 NocoBase API 客户端
- [ ] 实现数据转换逻辑（Collection/Item/Image）
- [ ] 实现同步调度器（增量同步/全量同步）
- [ ] 处理冲突解决策略
- [ ] 实现图片上传下载
- [ ] 移动端集成配置界面
- [ ] 单元测试和集成测试

---

## 方案二：单向同步 - NocoBase 作为数据源（⭐⭐⭐⭐）

### 适用场景
NocoBase 作为主要数据管理平台，Inventory App 用于现场盘点

### 架构设计
```
NocoBase (主数据) → 单向同步 → Inventory App (只读/部分可写)
```

### 实现方式
参考 Snipe-IT 集成，实现从 NocoBase 到 CouchDB 的单向同步

```typescript
// packages/integration-nocobase/one-way-sync.ts

async function syncFromNocoBase() {
  // 1. 从 NocoBase 获取所有 Collections
  const collections = await nocobaseAPI.getCollections();
  
  // 2. 创建或更新到 CouchDB
  for (const collection of collections) {
    const existingCollection = await getDatum('collection', collection.id);
    if (existingCollection) {
      // 更新
      await saveDatum({
        ...existingCollection,
        name: collection.name,
        // ... 其他字段
      });
    } else {
      // 创建
      await saveDatum({
        __type: 'collection',
        __id: collection.id,
        name: collection.name,
        // ... 其他字段
      });
    }
  }
  
  // 3. 同步 Items
  const items = await nocobaseAPI.getItems();
  for (const item of items) {
    // 类似逻辑
  }
}
```

### 优势
- ✅ 逻辑简单，易于实现和维护
- ✅ 避免数据冲突问题
- ✅ 开发周期短（预计 1 周）
- ✅ 适合已有 NocoBase 系统的场景

### 劣势
- ❌ 移动端创建的数据需手动导入 NocoBase
- ❌ 灵活性较低
- ❌ 不支持移动端离线创建数据自动回传

### 开发任务清单
- [ ] 创建单向同步脚本
- [ ] 实现数据拉取逻辑
- [ ] 实现增量同步（基于时间戳）
- [ ] 配置定时任务
- [ ] 错误处理和日志记录

---

## 方案三：NocoBase 作为 Web 前端（⭐⭐⭐）

### 适用场景
替代自建 Web 界面，使用 NocoBase 作为 CouchDB 的管理前端

### 架构设计
```
CouchDB ←→ REST API Wrapper ←→ NocoBase REST API Data Source ←→ NocoBase UI
```

### 实现方式

#### 1. 扩展 couchdb-public-server

```javascript
// packages/couchdb-public-server/server.js

// 添加完整的 REST API 端点

// 获取 Collections 列表
app.get('/api/:database/collections', async (req, res) => {
  const { database } = req.params;
  const db = couch.use(database);
  
  const result = await db.find({
    selector: { type: 'collection' },
    limit: 1000
  });
  
  res.json({
    data: result.docs.map(doc => ({
      id: doc._id,
      name: doc.name,
      collection_reference_number: doc.collection_reference_number,
      created_at: doc.__created_at,
      updated_at: doc.__updated_at
    }))
  });
});

// 创建 Collection
app.post('/api/:database/collections', async (req, res) => {
  const { database } = req.params;
  const db = couch.use(database);
  
  const newDoc = {
    _id: `zz01-collection-${uuid()}`,
    type: 'collection',
    ...req.body,
    __created_at: Date.now(),
    __updated_at: Date.now()
  };
  
  const result = await db.insert(newDoc);
  res.json({ data: { id: result.id, ...newDoc } });
});

// 更新 Collection
app.patch('/api/:database/collections/:id', async (req, res) => {
  const { database, id } = req.params;
  const db = couch.use(database);
  
  const doc = await db.get(id);
  const updated = {
    ...doc,
    ...req.body,
    __updated_at: Date.now()
  };
  
  await db.insert(updated);
  res.json({ data: updated });
});

// 删除 Collection
app.delete('/api/:database/collections/:id', async (req, res) => {
  const { database, id } = req.params;
  const db = couch.use(database);
  
  const doc = await db.get(id);
  await db.destroy(doc._id, doc._rev);
  res.json({ success: true });
});

// Items 的类似端点
app.get('/api/:database/items', async (req, res) => { /* ... */ });
app.post('/api/:database/items', async (req, res) => { /* ... */ });
app.patch('/api/:database/items/:id', async (req, res) => { /* ... */ });
app.delete('/api/:database/items/:id', async (req, res) => { /* ... */ });
```

#### 2. 在 NocoBase 中配置数据源

在 NocoBase 管理界面：
1. 数据源管理 → 添加数据源
2. 选择 "REST API" 类型
3. 配置：
   - Base URL: `http://your-server:3000/api/your_database`
   - 认证方式：Bearer Token（可选）
4. 映射资源：
   - Collections → `/collections`
   - Items → `/items`

### 优势
- ✅ 无需编写同步代码
- ✅ 快速搭建（1-2 天）
- ✅ 实时数据，无延迟
- ✅ 充分利用 NocoBase 的 UI 能力
- ✅ 开发成本最低

### 劣势
- ❌ 受限于 CouchDB API 能力
- ❌ 复杂查询支持有限
- ❌ 不支持 NocoBase 的高级功能（工作流、审批等）
- ❌ 只能查看和基础编辑

### 开发任务清单
- [ ] 扩展 couchdb-public-server 添加完整 CRUD API
- [ ] 实现分页、排序、过滤
- [ ] 添加身份认证
- [ ] 在 NocoBase 中配置数据源
- [ ] 测试数据读写

---

## 方案四：NocoBase Plugin 模式（⭐⭐⭐⭐⭐ 高级方案）

### 适用场景
深度集成，将 Inventory 功能作为 NocoBase 插件

### 架构设计
```
NocoBase + @nocobase/plugin-inventory (完全融合)
```

### 实现方式

#### 1. 插件目录结构

```
packages/plugins/@nocobase/plugin-inventory/
├── src/
│   ├── server/
│   │   ├── collections/          # 数据表定义
│   │   │   ├── collections.ts
│   │   │   ├── items.ts
│   │   │   └── images.ts
│   │   ├── actions/              # 自定义 API
│   │   │   ├── rfid-scan.ts
│   │   │   └── barcode-gen.ts
│   │   ├── migrations/           # 数据迁移
│   │   └── plugin.ts
│   ├── client/
│   │   ├── pages/               # 页面组件
│   │   │   ├── ItemList.tsx
│   │   │   ├── ItemDetail.tsx
│   │   │   └── RFIDScanner.tsx
│   │   ├── schemas/             # UI Schema
│   │   └── plugin.ts
│   └── locale/                  # 国际化
├── package.json
└── README.md
```

#### 2. 定义数据表

```typescript
// src/server/collections/collections.ts

export default {
  name: 'inventory_collections',
  fields: [
    {
      type: 'string',
      name: 'name',
      required: true,
    },
    {
      type: 'string',
      name: 'icon_name',
    },
    {
      type: 'string',
      name: 'icon_color',
    },
    {
      type: 'string',
      name: 'collection_reference_number',
      unique: true,
      required: true,
    },
    {
      type: 'hasMany',
      name: 'items',
      target: 'inventory_items',
      foreignKey: 'collection_id',
    },
  ],
};
```

```typescript
// src/server/collections/items.ts

export default {
  name: 'inventory_items',
  fields: [
    {
      type: 'string',
      name: 'name',
      required: true,
    },
    {
      type: 'belongsTo',
      name: 'collection',
      target: 'inventory_collections',
      foreignKey: 'collection_id',
    },
    {
      type: 'string',
      name: 'model_name',
    },
    {
      type: 'text',
      name: 'notes',
    },
    {
      type: 'string',
      name: 'rfid_tag_epc_memory_bank_contents',
    },
    {
      type: 'integer',
      name: 'serial',
    },
    {
      type: 'hasMany',
      name: 'images',
      target: 'inventory_images',
      foreignKey: 'item_id',
    },
    {
      type: 'decimal',
      name: 'purchase_price',
    },
    {
      type: 'date',
      name: 'purchase_date',
    },
    {
      type: 'date',
      name: 'expiry_date',
    },
  ],
};
```

#### 3. 自定义 API Action

```typescript
// src/server/actions/rfid-scan.ts

export default {
  name: 'rfid:scan',
  type: 'resource',
  resourceName: 'inventory_items',
  handler: async (ctx, next) => {
    const { epc } = ctx.action.params;
    
    // 根据 EPC 查找 Item
    const item = await ctx.db.getRepository('inventory_items').findOne({
      filter: {
        rfid_tag_epc_memory_bank_contents: epc,
      },
    });
    
    if (!item) {
      ctx.throw(404, 'Item not found');
    }
    
    ctx.body = item;
    await next();
  },
};
```

#### 4. 移动端对接

```typescript
// App/app/data/nocobase-adapter.ts

import { DataTypeWithID } from '@deps/data/types';

export class NocoBaseAdapter {
  private baseUrl: string;
  private token: string;

  constructor(baseUrl: string, token: string) {
    this.baseUrl = baseUrl;
    this.token = token;
  }

  async getCollections(): Promise<DataTypeWithID<'collection'>[]> {
    const response = await fetch(`${this.baseUrl}/api/inventory_collections:list`, {
      headers: {
        'Authorization': `Bearer ${this.token}`,
      },
    });
    const data = await response.json();
    return data.data.map(this.mapCollection);
  }

  async getItems(collectionId: string): Promise<DataTypeWithID<'item'>[]> {
    const response = await fetch(
      `${this.baseUrl}/api/inventory_items:list?filter[collection_id]=${collectionId}`,
      {
        headers: {
          'Authorization': `Bearer ${this.token}`,
        },
      }
    );
    const data = await response.json();
    return data.data.map(this.mapItem);
  }

  private mapCollection(nocobaseData: any): DataTypeWithID<'collection'> {
    return {
      __type: 'collection',
      __id: nocobaseData.id,
      name: nocobaseData.name,
      collection_reference_number: nocobaseData.collection_reference_number,
      icon_name: nocobaseData.icon_name,
      icon_color: nocobaseData.icon_color,
      config_uuid: '', // 需要适配
      __created_at: new Date(nocobaseData.createdAt).getTime(),
      __updated_at: new Date(nocobaseData.updatedAt).getTime(),
    };
  }

  private mapItem(nocobaseData: any): DataTypeWithID<'item'> {
    return {
      __type: 'item',
      __id: nocobaseData.id,
      name: nocobaseData.name,
      collection_id: nocobaseData.collection_id,
      model_name: nocobaseData.model_name,
      notes: nocobaseData.notes,
      config_uuid: '', // 需要适配
      __created_at: new Date(nocobaseData.createdAt).getTime(),
      __updated_at: new Date(nocobaseData.updatedAt).getTime(),
    };
  }
}
```

### 优势
- ✅ 深度集成，用户体验最佳
- ✅ 统一的权限、工作流、审批系统
- ✅ 可扩展性强
- ✅ 适合企业级应用
- ✅ 充分利用 NocoBase 生态（插件市场、主题等）
- ✅ 支持多租户

### 劣势
- ❌ 开发工作量最大（1-2 个月）
- ❌ 需要重构移动端数据层
- ❌ 依赖 NocoBase 生态
- ❌ 学习曲线较陡

### 开发任务清单
- [ ] 创建 NocoBase 插件骨架
- [ ] 定义所有数据表（Collections, Items, Images）
- [ ] 实现数据迁移脚本（从 CouchDB 迁移）
- [ ] 开发前端页面组件
- [ ] 实现 RFID 相关自定义 API
- [ ] 移动端适配（切换到 NocoBase API）
- [ ] 权限配置
- [ ] 工作流集成（审批流程）
- [ ] 单元测试和 E2E 测试
- [ ] 插件文档

---

## 方案五：混合模式（实用方案 ⭐⭐⭐⭐）

### 适用场景
快速启动，逐步迁移

### 架构设计

#### Phase 1: 只读 Web 界面（1-2 天）
```
CouchDB → REST API Wrapper → NocoBase (只读)
```

#### Phase 2: 增加双向同步（2-3 周）
```
CouchDB ←→ 同步服务 ←→ NocoBase
```

#### Phase 3: 完全迁移（可选，1-2 个月）
```
NocoBase (主存储) + 移动端适配
```

### 实现路线图

#### Phase 1: 快速搭建（完成时间：2 天）
1. 扩展 `couchdb-public-server` 提供只读 REST API
2. 在 NocoBase 中配置为外部数据源
3. 创建查看界面（列表、详情、搜索）

**交付物**：
- 可以在 NocoBase 中查看所有资产
- 基础搜索和筛选功能

#### Phase 2: 双向同步（完成时间：3 周）
1. 开发同步插件 `packages/integration-nocobase`
2. 实现增量同步逻辑
3. 移动端添加同步配置界面
4. 支持在 NocoBase 中编辑数据

**交付物**：
- 自动/手动同步功能
- 冲突解决机制
- 同步历史记录

#### Phase 3: 深度集成（完成时间：2 个月，可选）
1. 开发 NocoBase 插件
2. 移动端切换到 NocoBase API
3. 数据完全迁移到 NocoBase
4. 下线 CouchDB（可选）

**交付物**：
- 完整的 NocoBase 插件
- 移动端完全适配
- 数据迁移工具

### 优势
- ✅ 渐进式迁移，风险低
- ✅ 每个阶段都可交付价值
- ✅ 灵活调整方向
- ✅ 可以在任何阶段停止
- ✅ 学习成本分散

### 劣势
- ❌ 总体周期较长
- ❌ 中间状态需要维护两套系统
- ❌ 可能产生技术债务

### 决策点

**在 Phase 1 结束后评估**：
- 是否满足需求？如果是，可以停止
- NocoBase 表现如何？
- 团队是否愿意深入集成？

**在 Phase 2 结束后评估**：
- 同步是否稳定？
- 是否需要更深度的集成？
- 成本收益分析

---

## 方案对比总结

| 维度 | 方案一 | 方案二 | 方案三 | 方案四 | 方案五 |
|------|--------|--------|--------|--------|--------|
| **开发周期** | 2-3周 | 1周 | 1-2天 | 1-2月 | 分阶段 |
| **技术难度** | ⭐⭐⭐ | ⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| **维护成本** | 中 | 低 | 低 | 高 | 中→低 |
| **功能完整性** | 高 | 中 | 低 | 最高 | 低→高 |
| **灵活性** | 高 | 中 | 低 | 最高 | 高 |
| **风险** | 中 | 低 | 低 | 高 | 最低 |
| **适用场景** | 需保留现有架构 | NocoBase 为主 | 快速搭建 | 企业深度集成 | 渐进迁移 |
| **推荐指数** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |

---

## 技术栈对比

### NocoBase API 能力

| 功能 | 方案一 | 方案二 | 方案三 | 方案四 |
|------|--------|--------|--------|--------|
| CRUD 操作 | ✅ | ✅ | ✅ | ✅ |
| 关联查询 | ✅ | ✅ | ⚠️ | ✅ |
| 批量操作 | ✅ | ✅ | ❌ | ✅ |
| 文件上传 | ✅ | ✅ | ⚠️ | ✅ |
| 工作流 | ❌ | ❌ | ❌ | ✅ |
| 权限控制 | ⚠️ | ⚠️ | ❌ | ✅ |
| 自定义 API | ❌ | ❌ | ❌ | ✅ |

### 移动端适配

| 功能 | 方案一 | 方案二 | 方案三 | 方案四 |
|------|--------|--------|--------|--------|
| 离线支持 | ✅ | ✅ | ✅ | ⚠️ |
| RFID 扫描 | ✅ | ✅ | ✅ | ✅ |
| 本地数据 | ✅ | ✅ | ✅ | ❌ |
| 实时同步 | ⚠️ | ❌ | ❌ | ✅ |

---

## 决策建议

### 快速启动推荐
**方案三 + 方案五（Phase 1）**
- 投入：1-2 天
- 产出：可用的 Web 查看界面
- 决策点：评估是否需要继续

### 稳定集成推荐
**方案一（双向同步）**
- 投入：2-3 周
- 产出：完整的双向同步方案
- 适合：需要保留现有移动端架构

### 企业级推荐
**方案四（NocoBase 插件）**
- 投入：1-2 个月
- 产出：深度集成的企业解决方案
- 适合：长期规划，企业级应用

### 保守推荐
**方案五（混合模式）**
- 投入：分阶段
- 产出：渐进式价值交付
- 适合：不确定需求，希望降低风险

---

## 下一步行动

根据选择的方案，可以开始以下工作：

### 如果选择方案一（双向同步）
1. 创建 `packages/integration-nocobase` 目录
2. 参考 `packages/integration-airtable` 实现
3. 设计数据映射关系
4. 实现 NocoBase API 客户端

### 如果选择方案三（REST 数据源）
1. 扩展 `packages/couchdb-public-server/server.js`
2. 添加完整的 CRUD API
3. 在 NocoBase 中配置数据源
4. 测试数据读写

### 如果选择方案四（插件开发）
1. 学习 NocoBase 插件开发文档
2. 创建插件骨架
3. 设计数据表结构
4. 实现核心功能

### 如果选择方案五（混合模式）
1. 从 Phase 1 开始（方案三）
2. 评估效果
3. 决定是否进入 Phase 2

---

## 附录

### 相关资源
- [NocoBase 官方文档](https://docs.nocobase.com/)
- [NocoBase API 文档](https://docs.nocobase.com/api/http/)
- [NocoBase 插件开发指南](https://docs.nocobase.com/development)
- [Inventory 项目 GitHub](https://github.com/zetavg/Inventory)

### 现有集成参考
- `packages/integration-airtable` - 双向同步示例
- `packages/integration-snipe-it` - 单向同步示例
- `packages/couchdb-public-server` - REST API 服务器基础

---

*文档创建时间：2026-01-07*
*分析人：AI Assistant*
