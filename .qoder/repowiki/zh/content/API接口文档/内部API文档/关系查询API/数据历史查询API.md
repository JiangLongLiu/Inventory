# 数据历史查询API

<cite>
**本文档引用的文件**
- [getGetDatumHistories.ts](file://packages/data-storage-couchdb/lib/functions/getGetDatumHistories.ts)
- [getGetHistoriesInBatch.ts](file://packages/data-storage-couchdb/lib/functions/getGetHistoriesInBatch.ts)
- [getListHistoryBatchesCreatedBy.ts](file://packages/data-storage-couchdb/lib/functions/getListHistoryBatchesCreatedBy.ts)
- [types.ts](file://packages/data-storage-couchdb/lib/types.ts)
- [CouchDBData.ts](file://packages/data-storage-couchdb/lib/CouchDBData.ts)
- [index.ts](file://App/app/data/functions/index.ts)
- [HistoryBatchesModalScreen.tsx](file://App/app/screens/data-history/HistoryBatchesModalScreen.tsx)
- [HistoryBatchModalScreen.tsx](file://App/app/screens/data-history/HistoryBatchModalScreen.tsx)
</cite>

## 目录
1. [简介](#简介)
2. [核心功能](#核心功能)
3. [版本控制机制](#版本控制机制)
4. [时间戳处理](#时间戳处理)
5. [历史记录分页](#历史记录分页)
6. [数据存储策略](#数据存储策略)
7. [清理机制](#清理机制)
8. [性能考虑](#性能考虑)
9. [与主数据查询的集成](#与主数据查询的集成)
10. [一致性保证](#一致性保证)
11. [实际示例](#实际示例)

## 简介
数据历史查询API为库存管理系统提供了完整的数据变更追踪功能。该API允许用户查询物品或集合的修改历史，支持按时间、用户和批次进行过滤。系统通过CouchDB/PouchDB数据库实现数据版本控制，确保所有变更都有据可查。

**本节不包含具体文件分析，因此不提供来源**

## 核心功能
数据历史查询API的核心功能由`getGetDatumHistories`函数实现，该函数提供对特定数据项变更历史的查询能力。API支持多种查询参数，包括数据类型、ID、分页限制和时间戳过滤。

```mermaid
flowchart TD
Start([开始]) --> ValidateInput["验证输入参数"]
ValidateInput --> InputValid{"输入有效?"}
InputValid --> |否| ReturnError["返回错误响应"]
InputValid --> |是| CheckIndex["检查索引存在性"]
CheckIndex --> CreateIndex["创建索引(如需要)"]
CreateIndex --> ExecuteQuery["执行数据库查询"]
ExecuteQuery --> ParseResults["解析查询结果"]
ParseResults --> ValidateData["验证数据完整性"]
ValidateData --> FilterResults["过滤结果"]
FilterResults --> ReturnResults["返回历史记录"]
ReturnError --> End([结束])
ReturnResults --> End
```

**图表来源**
- [getGetDatumHistories.ts](file://packages/data-storage-couchdb/lib/functions/getGetDatumHistories.ts#L18-L103)

**本节来源**
- [getGetDatumHistories.ts](file://packages/data-storage-couchdb/lib/functions/getGetDatumHistories.ts#L18-L103)

## 版本控制机制
系统采用基于文档的历史记录存储方式，每个数据变更都会生成一个历史记录文档。历史记录包含变更前后的数据状态，支持完整的版本回溯。

```mermaid
classDiagram
class DataHistory {
+string created_by
+number batch
+string event_name
+string data_type
+string data_id
+number timestamp
+object original_data
+object new_data
}
class Datum {
+string __id
+string __type
+object data
+string __rev
}
class HistoryManager {
+getDatumHistories(type, id, options)
+getHistoriesInBatch(batch, options)
+listHistoryBatchesCreatedBy(createdBy, options)
+restoreHistory(history, options)
}
DataHistory --> Datum : "关联到"
HistoryManager --> DataHistory : "查询"
HistoryManager --> Datum : "恢复"
```

**图表来源**
- [types.ts](file://packages/data-storage-couchdb/lib/types.ts#L3-L12)
- [getGetDatumHistories.ts](file://packages/data-storage-couchdb/lib/functions/getGetDatumHistories.ts#L18-L103)

**本节来源**
- [types.ts](file://packages/data-storage-couchdb/lib/types.ts#L3-L12)
- [getGetDatumHistories.ts](file://packages/data-storage-couchdb/lib/functions/getGetDatumHistories.ts#L18-L103)

## 时间戳处理
系统使用毫秒级时间戳来精确记录数据变更时间。时间戳在查询中作为排序和过滤的关键字段，支持按时间范围查询历史记录。

```mermaid
sequenceDiagram
participant Client as "客户端"
participant API as "历史查询API"
participant DB as "数据库"
Client->>API : 查询历史(类型, ID, after=时间戳)
API->>API : 验证参数
API->>API : 构建查询条件
API->>DB : 执行查询(时间戳 < after)
DB-->>API : 返回历史记录
API->>API : 按时间戳降序排序
API-->>Client : 返回排序后的历史记录
```

**图表来源**
- [getGetDatumHistories.ts](file://packages/data-storage-couchdb/lib/functions/getGetDatumHistories.ts#L46-L47)
- [getGetHistoriesInBatch.ts](file://packages/data-storage-couchdb/lib/functions/getGetHistoriesInBatch.ts#L43-L44)

**本节来源**
- [getGetDatumHistories.ts](file://packages/data-storage-couchdb/lib/functions/getGetDatumHistories.ts#L28-L29)
- [getGetHistoriesInBatch.ts](file://packages/data-storage-couchdb/lib/functions/getGetHistoriesInBatch.ts#L21-L23)

## 历史记录分页
API提供分页功能，支持大规模历史数据的高效查询。通过limit和after参数实现分页，避免一次性加载过多数据。

```mermaid
flowchart TD
Start([开始]) --> SetDefault["设置默认分页参数"]
SetDefault --> BuildQuery["构建查询条件"]
BuildQuery --> AddLimit["添加limit参数"]
AddLimit --> AddAfter["添加after参数(如提供)"]
AddAfter --> ExecuteQuery["执行分页查询"]
ExecuteQuery --> CheckResults["检查结果数量"]
CheckResults --> HasMore{"结果数量=limit?"}
HasMore --> |是| SetNextAfter["设置下次查询的after值"]
HasMore --> |否| NoMoreResults["标记无更多结果"]
SetNextAfter --> ReturnResults["返回结果和分页信息"]
NoMoreResults --> ReturnResults
ReturnResults --> End([结束])
```

**图表来源**
- [getGetDatumHistories.ts](file://packages/data-storage-couchdb/lib/functions/getGetDatumHistories.ts#L28-L29)
- [HistoryBatchesModalScreen.tsx](file://App/app/screens/data-history/HistoryBatchesModalScreen.tsx#L69-L71)

**本节来源**
- [getGetDatumHistories.ts](file://packages/data-storage-couchdb/lib/functions/getGetDatumHistories.ts#L28-L29)
- [HistoryBatchesModalScreen.tsx](file://App/app/screens/data-history/HistoryBatchesModalScreen.tsx#L69-L71)

## 数据存储策略
历史数据存储采用专门的文档类型`_history`，与主数据分离存储。每个历史记录包含完整的变更信息，包括操作者、批次、事件名称和时间戳。

```mermaid
erDiagram
HISTORY {
string _id PK
string type
string created_by
number batch
string event_name
string data_type
string data_id
number timestamp
object original_data
object new_data
}
DATUM {
string __id PK
string __type
object data
string __rev
}
HISTORY ||--o{ DATUM : "关联到"
```

**图表来源**
- [getGetDatumHistories.ts](file://packages/data-storage-couchdb/lib/functions/getGetDatumHistories.ts#L43-L45)
- [getSaveDatum.ts](file://packages/data-storage-couchdb/lib/functions/getSaveDatum.ts#L108-L109)

**本节来源**
- [getGetDatumHistories.ts](file://packages/data-storage-couchdb/lib/functions/getGetDatumHistories.ts#L7-L16)
- [getSaveDatum.ts](file://packages/data-storage-couchdb/lib/functions/getSaveDatum.ts#L98-L115)

## 清理机制
系统通过批次(batch)机制支持历史数据的批量清理。虽然当前代码未实现自动清理功能，但数据结构支持按批次删除历史记录。

```mermaid
flowchart TD
Start([开始]) --> IdentifyBatch["识别要清理的批次"]
IdentifyBatch --> FindHistories["查找该批次的所有历史记录"]
FindHistories --> DeleteHistories["删除匹配的历史记录"]
DeleteHistories --> UpdateIndex["更新相关索引"]
UpdateIndex --> LogCleanup["记录清理操作"]
LogCleanup --> End([结束])
```

**图表来源**
- [getListHistoryBatchesCreatedBy.ts](file://packages/data-storage-couchdb/lib/functions/getListHistoryBatchesCreatedBy.ts#L12-L13)
- [getGetHistoriesInBatch.ts](file://packages/data-storage-couchdb/lib/functions/getGetHistoriesInBatch.ts#L37-L38)

**本节来源**
- [getListHistoryBatchesCreatedBy.ts](file://packages/data-storage-couchdb/lib/functions/getListHistoryBatchesCreatedBy.ts#L25-L28)
- [getGetHistoriesInBatch.ts](file://packages/data-storage-couchdb/lib/functions/getGetHistoriesInBatch.ts#L20-L23)

## 性能考虑
API设计考虑了性能优化，通过预创建索引、错误重试机制和结果缓存来提高查询效率。索引按类型、数据类型、数据ID和时间戳建立，确保快速查询。

```mermaid
flowchart TD
Start([开始]) --> CheckIndexFirst["检查alwaysCreateIndexFirst"]
CheckIndexFirst --> |是| CreateIndex["立即创建索引"]
CheckIndexFirst --> |否| TryQuery["尝试执行查询"]
CreateIndex --> ExecuteQuery["执行查询"]
TryQuery --> ExecuteQuery
ExecuteQuery --> QuerySuccess{"查询成功?"}
QuerySuccess --> |是| ReturnResults["返回结果"]
QuerySuccess --> |否| RetryCount{"重试次数>3?"}
RetryCount --> |否| CreateIndexOnFail["创建索引"]
RetryCount --> |是| ThrowError["抛出异常"]
CreateIndexOnFail --> IncrementRetry["重试次数+1"]
IncrementRetry --> TryQuery
ReturnResults --> End([结束])
ThrowError --> End
```

**图表来源**
- [getGetDatumHistories.ts](file://packages/data-storage-couchdb/lib/functions/getGetDatumHistories.ts#L30-L86)
- [getGetHistoriesInBatch.ts](file://packages/data-storage-couchdb/lib/functions/getGetHistoriesInBatch.ts#L24-L77)

**本节来源**
- [getGetDatumHistories.ts](file://packages/data-storage-couchdb/lib/functions/getGetDatumHistories.ts#L30-L86)
- [getGetHistoriesInBatch.ts](file://packages/data-storage-couchdb/lib/functions/getGetHistoriesInBatch.ts#L24-L77)

## 与主数据查询的集成
历史查询API与主数据查询系统紧密集成，通过统一的CouchDBData类暴露所有数据访问功能。这种设计确保了数据访问的一致性和便利性。

```mermaid
classDiagram
class CouchDBData {
+getConfig
+updateConfig
+getDatum
+getData
+saveDatum
+getDatumHistories
+listHistoryBatchesCreatedBy
+getHistoriesInBatch
+restoreHistory
}
class MainDataService {
+getDatum
+saveDatum
+deleteDatum
}
class HistoryService {
+getDatumHistories
+getHistoriesInBatch
+listHistoryBatchesCreatedBy
+restoreHistory
}
CouchDBData --> MainDataService : "包含"
CouchDBData --> HistoryService : "包含"
MainDataService --> CouchDBData : "使用"
HistoryService --> CouchDBData : "使用"
```

**图表来源**
- [CouchDBData.ts](file://packages/data-storage-couchdb/lib/CouchDBData.ts#L42-L96)
- [index.ts](file://App/app/data/functions/index.ts#L88-L90)

**本节来源**
- [CouchDBData.ts](file://packages/data-storage-couchdb/lib/CouchDBData.ts#L56-L59)
- [index.ts](file://App/app/data/functions/index.ts#L88-L90)

## 一致性保证
系统通过事务性操作和错误处理机制确保数据一致性。查询操作包含重试逻辑，索引创建有容错处理，数据验证使用Zod库确保类型安全。

```mermaid
sequenceDiagram
participant Client as "客户端"
participant Service as "历史服务"
participant DB as "数据库"
participant Validator as "验证器"
Client->>Service : 请求历史记录
Service->>Service : 验证输入参数
Service->>DB : 查询数据库
alt 查询失败
DB--xService : 返回错误
Service->>Service : 检查重试次数
Service->>Service : 创建索引
Service->>DB : 重试查询
else 查询成功
DB-->>Service : 返回原始数据
Service->>Validator : 验证数据结构
Validator-->>Service : 返回验证结果
Service->>Service : 过滤有效数据
end
Service-->>Client : 返回过滤后的历史记录
```

**图表来源**
- [getGetDatumHistories.ts](file://packages/data-storage-couchdb/lib/functions/getGetDatumHistories.ts#L57-L86)
- [types.ts](file://packages/data-storage-couchdb/lib/types.ts#L3-L12)

**本节来源**
- [getGetDatumHistories.ts](file://packages/data-storage-couchdb/lib/functions/getGetDatumHistories.ts#L88-L99)
- [types.ts](file://packages/data-storage-couchdb/lib/types.ts#L3-L12)

## 实际示例
以下示例展示如何使用数据历史查询API获取物品或集合的修改历史。

```mermaid
flowchart TD
Start([开始]) --> GetService["获取数据服务实例"]
GetService --> QueryItemHistory["查询物品历史"]
QueryItemHistory --> SetParams["设置查询参数"]
SetParams --> CallAPI["调用getDatumHistories"]
CallAPI --> ProcessResults["处理返回结果"]
ProcessResults --> DisplayHistory["显示历史记录"]
DisplayHistory --> End([结束])
```

**图表来源**
- [HistoryBatchesModalScreen.tsx](file://App/app/screens/data-history/HistoryBatchesModalScreen.tsx#L61-L68)
- [HistoryBatchModalScreen.tsx](file://App/app/screens/data-history/HistoryBatchModalScreen.tsx#L60-L67)

**本节来源**
- [HistoryBatchesModalScreen.tsx](file://App/app/screens/data-history/HistoryBatchesModalScreen.tsx#L55-L84)
- [HistoryBatchModalScreen.tsx](file://App/app/screens/data-history/HistoryBatchModalScreen.tsx#L54-L76)