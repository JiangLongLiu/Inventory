# 附件管理API

<cite>
**本文档中引用的文件**  
- [getAttachAttachmentToDatum.ts](file://packages/data-storage-couchdb/lib/functions/getAttachAttachmentToDatum.ts)
- [getGetAttachmentFromDatum.ts](file://packages/data-storage-couchdb/lib/functions/getGetAttachmentFromDatum.ts)
- [getGetAttachmentInfoFromDatum.ts](file://packages/data-storage-couchdb/lib/functions/getGetAttachmentInfoFromDatum.ts)
- [getGetAllAttachmentInfoFromDatum.ts](file://packages/data-storage-couchdb/lib/functions/getGetAllAttachmentInfoFromDatum.ts)
- [attachments.ts](file://Data/lib/attachments.ts)
- [types.ts](file://Data/lib/types.ts)
- [EditImagesUI.tsx](file://App/app/features/inventory/components/EditImagesUI.tsx)
- [ImagesSliderBox.tsx](file://App/app/features/inventory/components/ImagesSliderBox.tsx)
</cite>

## 目录
1. [简介](#简介)
2. [核心API接口定义](#核心api接口定义)
3. [附件存储策略与限制](#附件存储策略与限制)
4. [安全访问控制](#安全访问控制)
5. [附件版本管理与清理机制](#附件版本管理与清理机制)
6. [库存系统中图片管理的实际用法](#库存系统中图片管理的实际用法)
7. [错误处理与调试](#错误处理与调试)

## 简介

附件管理API为库存管理系统提供了完整的二进制文件（如图片、文档）上传、下载和元数据查询功能。该系统基于CouchDB/PouchDB数据库的附件功能，实现了对物品图片等附件的安全存储和高效访问。API设计遵循类型安全原则，通过TypeScript接口定义确保附件操作的正确性，并在库存管理场景中实现了图片的版本化存储和按需加载。

**Section sources**
- [getAttachAttachmentToDatum.ts](file://packages/data-storage-couchdb/lib/functions/getAttachAttachmentToDatum.ts#L1-L37)
- [getGetAttachmentFromDatum.ts](file://packages/data-storage-couchdb/lib/functions/getGetAttachmentFromDatum.ts#L1-L45)

## 核心API接口定义

附件管理API提供了三个核心函数：`getAttachAttachmentToDatum`、`getGetAttachmentFromDatum`和`getGetAttachmentInfoFromDatum`，分别用于附件的上传、下载和元数据查询。

### 附件上传：getAttachAttachmentToDatum

该函数用于将二进制数据作为附件附加到数据记录上。它不直接与数据库交互，而是将附件信息暂存到数据对象的`__raw`字段中，等待后续的`saveDatum`操作将数据持久化到数据库。

```mermaid
sequenceDiagram
participant Client as "客户端"
participant API as "附件管理API"
participant Data as "数据对象"
Client->>API : 调用attachAttachmentToDatum
API->>Data : 验证__raw结构
Data->>Data : 创建或初始化_attachments对象
Data->>Data : 添加附件信息(content_type, data)
Data->>Data : 更新__updated_at时间戳
API-->>Client : 返回更新后的数据对象
Note over API,Data : 附件数据暂存于内存中<br/>需通过saveDatum持久化
```

**Diagram sources**
- [getAttachAttachmentToDatum.ts](file://packages/data-storage-couchdb/lib/functions/getAttachAttachmentToDatum.ts#L1-L37)

**Section sources**
- [getAttachAttachmentToDatum.ts](file://packages/data-storage-couchdb/lib/functions/getAttachAttachmentToDatum.ts#L1-L37)
- [types.ts](file://Data/lib/types.ts#L166-L174)

### 附件下载：getGetAttachmentFromDatum

该函数用于从数据记录中获取附件的完整信息，包括元数据和二进制数据。它首先通过`getGetAttachmentInfoFromDatum`获取附件元数据，然后根据数据库类型（PouchDB或CouchDB）调用相应的API获取实际的二进制数据。

```mermaid
sequenceDiagram
participant Client as "客户端"
participant API as "附件管理API"
participant Info as "元数据查询"
participant DB as "数据库"
Client->>API : 调用getAttachmentFromDatum
API->>Info : 调用getAttachmentInfoFromDatum
Info-->>API : 返回附件元数据
API->>API : 检查元数据是否存在
alt 元数据存在
API->>DB : 根据dbType调用getAttachment
DB-->>API : 返回二进制数据
API-->>Client : 返回包含数据的完整附件信息
else 元数据不存在
API-->>Client : 返回null
end
```

**Diagram sources**
- [getGetAttachmentFromDatum.ts](file://packages/data-storage-couchdb/lib/functions/getGetAttachmentFromDatum.ts#L1-L45)

**Section sources**
- [getGetAttachmentFromDatum.ts](file://packages/data-storage-couchdb/lib/functions/getGetAttachmentFromDatum.ts#L1-L45)
- [types.ts](file://Data/lib/types.ts#L180-L188)

### 元数据查询：getGetAttachmentInfoFromDatum

该函数用于查询附件的元数据信息，包括内容类型、大小和摘要。它优先使用数据对象中的`__raw`字段，如果不存在则从数据库中获取完整数据。

```mermaid
flowchart TD
Start([开始]) --> CheckRaw["检查d.__raw是否存在"]
CheckRaw --> RawValid{"__raw有效且包含_attachments?"}
RawValid --> |是| UseRaw["使用现有__raw数据"]
RawValid --> |否| FetchFromDB["调用getDatum从数据库获取"]
FetchFromDB --> CheckDoc{"文档存在且为对象?"}
CheckDoc --> |否| ThrowError["抛出类型错误"]
CheckDoc --> |是| ExtractAttachments["提取_attachments对象"]
ExtractAttachments --> CheckAttachments{"attachments存在且为对象?"}
CheckAttachments --> |否| ReturnNull["返回null"]
CheckAttachments --> |是| ValidateInfo["验证attachmentInfo"]
ValidateInfo --> CheckContentType{"content_type为字符串?"}
CheckContentType --> |否| ThrowContentTypeError["抛出内容类型错误"]
CheckContentType --> |是| CheckSize{"size为数字?"}
CheckSize --> |否| ThrowSizeError["抛出大小错误"]
CheckSize --> |是| CheckDigest{"digest存在且为字符串?"}
CheckDigest --> |否| ReturnInfo["返回元数据对象"]
CheckDigest --> |是| ReturnInfo
ReturnInfo --> End([结束])
```

**Diagram sources**
- [getGetAttachmentInfoFromDatum.ts](file://packages/data-storage-couchdb/lib/functions/getGetAttachmentInfoFromDatum.ts#L1-L63)

**Section sources**
- [getGetAttachmentInfoFromDatum.ts](file://packages/data-storage-couchdb/lib/functions/getGetAttachmentInfoFromDatum.ts#L1-L63)
- [types.ts](file://Data/lib/types.ts#L176-L179)

### 批量元数据查询：getGetAllAttachmentInfoFromDatum

该函数用于获取数据记录中所有附件的元数据信息，返回一个以附件名称为键的对象。

```mermaid
flowchart TD
Start([开始]) --> GetRaw["获取原始文档数据"]
GetRaw --> ValidateRaw{"__raw为对象?"}
ValidateRaw --> |否| ThrowError["抛出类型错误"]
ValidateRaw --> |是| ExtractAttachments["提取_attachments对象"]
ExtractAttachments --> ValidateAttachments{"attachments存在且为对象?"}
ValidateAttachments --> |否| ReturnEmpty["返回空对象"]
ValidateAttachments --> |是| ProcessEntries["遍历attachments条目"]
ProcessEntries --> LoopStart
LoopStart --> GetEntry["获取单个附件信息"]
GetEntry --> ValidateContentType{"验证content_type"}
ValidateContentType --> ValidateSize{"验证size"}
ValidateSize --> ValidateDigest{"验证digest"}
ValidateDigest --> CreateEntry["创建元数据条目"]
CreateEntry --> AddToResult["添加到结果对象"]
AddToResult --> MoreEntries{"还有更多条目?"}
MoreEntries --> |是| LoopStart
MoreEntries --> |否| ReturnResult["返回结果对象"]
ReturnResult --> End([结束])
```

**Diagram sources**
- [getGetAllAttachmentInfoFromDatum.ts](file://packages/data-storage-couchdb/lib/functions/getGetAllAttachmentInfoFromDatum.ts#L1-L67)

**Section sources**
- [getGetAllAttachmentInfoFromDatum.ts](file://packages/data-storage-couchdb/lib/functions/getGetAllAttachmentInfoFromDatum.ts#L1-L67)

## 附件存储策略与限制

附件存储策略通过`attachments.ts`文件中的`attachment_definitions`进行定义，为不同数据类型规定了附件的存储规范。

### 存储策略定义

```mermaid
classDiagram
class AttachmentDefn {
+content_types : string[]
+required? : boolean
}
class AttachmentNameOfDataType {
+T : DataTypeName
+returns : keyof attachment_definitions[T]
}
class AttachmentContentType {
+T : DataTypeName
+N : AttachmentNameOfDataType<T>
+returns : content_types[number]
}
class attachment_definitions {
+image : {
thumbnail-128 : AttachmentDefn
image-1440 : AttachmentDefn
}
}
AttachmentNameOfDataType --> attachment_definitions : "类型约束"
AttachmentContentType --> attachment_definitions : "类型约束"
attachment_definitions --> AttachmentDefn : "包含"
```

**Diagram sources**
- [attachments.ts](file://Data/lib/attachments.ts#L1-L47)

**Section sources**
- [attachments.ts](file://Data/lib/attachments.ts#L1-L47)

### 存储限制

根据`attachment_definitions`的定义，系统对图片附件实施了以下存储限制：

- **内容类型限制**：仅允许`image/jpeg`和`image/png`格式的图片
- **必需附件**：每个图片必须包含`thumbnail-128`和`image-1440`两个版本
- **版本化存储**：同一图片存储为不同分辨率的多个版本，以适应不同使用场景
- **大小限制**：虽然代码中未明确限制大小，但通过`size`字段记录附件大小，可用于后续的大小验证和清理

## 安全访问控制

系统通过多层机制确保附件的安全访问：

1. **类型安全控制**：通过TypeScript泛型和条件类型，确保只能访问为特定数据类型定义的附件
2. **数据验证**：在获取附件信息时，严格验证`content_type`、`size`和`digest`字段的类型
3. **访问权限**：附件访问依赖于底层数据库的认证机制，只有经过认证的用户才能访问数据库中的附件
4. **数据完整性**：通过`digest`字段验证附件的完整性，防止数据损坏

```mermaid
sequenceDiagram
participant User as "用户"
participant App as "应用"
participant Auth as "认证服务"
participant DB as "数据库"
User->>App : 请求访问附件
App->>Auth : 验证用户身份
Auth-->>App : 返回认证令牌
App->>DB : 带令牌请求附件
DB->>DB : 验证令牌权限
alt 权限有效
DB-->>App : 返回附件数据
App-->>User : 显示附件
else 权限无效
DB-->>App : 返回403错误
App-->>User : 显示访问被拒绝
end
```

**Diagram sources**
- [getGetAttachmentFromDatum.ts](file://packages/data-storage-couchdb/lib/functions/getGetAttachmentFromDatum.ts#L21-L41)
- [getGetAttachmentInfoFromDatum.ts](file://packages/data-storage-couchdb/lib/functions/getGetAttachmentInfoFromDatum.ts#L39-L52)

**Section sources**
- [getGetAttachmentFromDatum.ts](file://packages/data-storage-couchdb/lib/functions/getGetAttachmentFromDatum.ts#L21-L41)
- [getGetAttachmentInfoFromDatum.ts](file://packages/data-storage-couchdb/lib/functions/getGetAttachmentInfoFromDatum.ts#L39-L52)

## 附件版本管理与清理机制

系统通过以下机制实现附件的版本管理和清理：

### 版本管理

- **多版本存储**：同一图片存储为`thumbnail-128`和`image-1440`两个版本，分别用于列表显示和详细查看
- **时间戳更新**：每次修改附件时，自动更新数据对象的`__updated_at`字段
- **历史记录**：通过`getGetDatumHistories`等函数可以查询数据的历史变更记录

### 清理机制

虽然代码中未直接实现清理功能，但提供了以下基础：

- **大小监控**：通过`size`字段记录每个附件的大小，可用于实现基于大小的清理策略
- **使用统计**：可以结合访问日志分析附件的使用频率，识别不常用的附件
- **批量操作**：`getAllAttachmentInfoFromDatum`函数支持批量获取附件信息，便于实现批量清理

```mermaid
flowchart TD
Start([开始]) --> GetAllInfo["获取所有附件信息"]
GetAllInfo --> FilterBySize["按大小过滤"]
FilterBySize --> FilterByDate["按更新时间过滤"]
FilterByDate --> AnalyzeUsage["分析使用频率"]
AnalyzeUsage --> IdentifyCandidates["识别清理候选"]
IdentifyCandidates --> ConfirmDeletion["确认删除"]
ConfirmDeletion --> DeleteAttachments["删除附件"]
DeleteAttachments --> UpdateData["更新数据对象"]
UpdateData --> SaveChanges["保存变更"]
SaveChanges --> End([结束])
```

**Section sources**
- [getGetAllAttachmentInfoFromDatum.ts](file://packages/data-storage-couchdb/lib/functions/getGetAllAttachmentInfoFromDatum.ts#L1-L67)
- [getSaveDatum.ts](file://packages/data-storage-couchdb/lib/functions/getSaveDatum.ts#L17-L40)

## 库存系统中图片管理的实际用法

在库存系统中，附件管理API主要用于物品图片的管理，主要通过`EditImagesUI`和`ImagesSliderBox`组件实现。

### 图片上传流程

```mermaid
sequenceDiagram
participant UI as "EditImagesUI"
participant Selector as "图片选择器"
participant API as "附件API"
participant DB as "数据库"
UI->>Selector : 用户点击添加图片
Selector-->>UI : 返回选中的图片数据
UI->>UI : 为每张图片生成唯一ID
loop 每张图片
UI->>API : 调用attachAttachmentToDatum
API->>UI : 返回更新后的数据对象
UI->>UI : 更新本地状态
end
UI->>DB : 调用saveDatum保存所有变更
DB-->>UI : 返回保存结果
UI-->>用户 : 显示保存状态
```

**Section sources**
- [EditImagesUI.tsx](file://App/app/features/inventory/components/EditImagesUI.tsx#L1-L200)

### 图片下载与显示流程

```mermaid
sequenceDiagram
participant UI as "ImagesSliderBox"
participant API as "附件API"
participant DB as "数据库"
UI->>API : 调用getGetDatum获取图片元数据
API->>DB : 查询图片记录
DB-->>API : 返回图片数据
API-->>UI : 返回图片元数据
UI->>API : 调用getGetAttachmentFromDatum获取缩略图
API->>DB : 获取thumbnail-128附件
DB-->>API : 返回缩略图数据
API-->>UI : 返回缩略图
UI->>UI : 显示缩略图
UI->>API : 调用getGetAttachmentFromDatum获取高清图
API->>DB : 获取image-1440附件
DB-->>API : 返回高清图数据
API-->>UI : 返回高清图
UI->>UI : 按需显示高清图
```

**Section sources**
- [ImagesSliderBox.tsx](file://App/app/features/inventory/components/ImagesSliderBox.tsx#L1-L200)

### 实际代码示例

在`EditImagesUI.tsx`中，图片管理的实现包括：

1. 使用`useImageSelector`钩子选择图片
2. 通过`getAttachAttachmentToDatum`将图片作为附件附加到数据对象
3. 使用`getSaveDatum`保存包含附件的完整数据

在`ImagesSliderBox.tsx`中，图片显示的实现包括：

1. 使用`getGetDatum`获取图片元数据
2. 通过`getGetAttachmentFromDatum`获取不同分辨率的图片数据
3. 将Base64编码的图片数据用于React Native的Image组件显示

## 错误处理与调试

系统实现了完善的错误处理机制，确保附件操作的可靠性：

- **类型验证**：在获取附件信息时，严格验证各字段的类型，防止类型错误
- **空值处理**：当附件不存在时，返回`null`而不是抛出异常，使调用方可以优雅地处理缺失附件
- **异步错误**：所有异步操作都包含适当的错误处理，防止未捕获的Promise拒绝
- **调试支持**：通过`humanFileSize`等工具函数，便于在开发和调试过程中查看附件信息

**Section sources**
- [getGetAttachmentInfoFromDatum.ts](file://packages/data-storage-couchdb/lib/functions/getGetAttachmentInfoFromDatum.ts#L25-L52)
- [PouchDBAttachmentScreen.tsx](file://App/app/screens/dev-tools/pouchdb/PouchDBAttachmentScreen.tsx#L1-L173)