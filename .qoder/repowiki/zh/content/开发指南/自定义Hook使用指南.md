# 自定义Hook使用指南

<cite>
**本文档中引用的文件**  
- [useDB.ts](file://App/app/db/hooks/useDB.ts)
- [useData.ts](file://App/app/data/hooks/useData.ts)
- [useConfig.ts](file://App/app/data/hooks/useConfig.ts)
- [usePersistedState.ts](file://App/app/hooks/usePersistedState.ts)
- [useDeepCompare.ts](file://App/app/hooks/useDeepCompare.ts)
- [useTabBarInsets.ts](file://App/app/hooks/useTabBarInsets.ts)
- [useScrollViewContentInsetFix.ts](file://App/app/hooks/useScrollViewContentInsetFix.ts)
- [useDataCount.ts](file://App/app/data/hooks/useDataCount.ts)
- [useRelated.ts](file://App/app/data/hooks/useRelated.ts)
- [useSave.ts](file://App/app/data/hooks/useSave.ts)
- [useView.ts](file://App/app/data/hooks/useView.ts)
</cite>

## 目录
1. [简介](#简介)
2. [数据相关Hook](#数据相关hook)
   1. [useDB：数据库实例访问](#usedb数据库实例访问)
   2. [useData：数据记录查询](#usedata数据记录查询)
   3. [useConfig：应用配置管理](#useconfig应用配置管理)
   4. [其他数据相关Hook](#其他数据相关hook)
3. [状态管理Hook](#状态管理hook)
   1. [usePersistedState：持久化本地状态](#usepersistedstate持久化本地状态)
   2. [useDeepCompare：深度比较依赖项](#usedeepcompare深度比较依赖项)
4. [UI相关Hook](#ui相关hook)
   1. [useTabBarInsets：处理底部安全区](#usetabbarinsets处理底部安全区)
   2. [useScrollViewContentInsetFix：解决滚动视图布局问题](#usescrollviewcontentinsetfix解决滚动视图布局问题)
5. [最佳实践与注意事项](#最佳实践与注意事项)
6. [总结](#总结)

## 简介
本指南系统介绍了项目中提供的各类自定义Hook及其适用场景。重点说明数据相关Hook、状态管理Hook和UI相关Hook的使用方法、TypeScript类型定义、调用示例和注意事项，帮助开发者正确使用这些抽象，避免常见陷阱如无限循环渲染或内存泄漏。

## 数据相关Hook

### useDB：数据库实例访问
`useDB` Hook用于访问数据库实例，是所有数据操作的基础。它从数据库模块中获取数据库连接，并提供给其他数据相关Hook使用。

**TypeScript类型定义**
```typescript
function useDB(): {
  db: PouchDB.Database | null;
  loading: boolean;
}
```

**调用示例**
```typescript
const { db } = useDB();
if (db) {
  // 使用数据库实例进行操作
}
```

**注意事项**
- 该Hook返回的`db`实例可能为`null`，在使用前需要进行空值检查
- 数据库连接的初始化是异步的，因此存在`loading`状态
- 该Hook被其他数据Hook（如`useData`、`useConfig`）内部使用，通常不需要直接调用

**Section sources**
- [useDB.ts](file://App/app/db/hooks/useDB.ts#L1-L3)

### useData：数据记录查询
`useData` Hook用于查询特定类型的数据记录，支持通过ID、ID数组或条件对象进行查询，并可指定排序、分页等选项。

**TypeScript类型定义**
```typescript
function useData<T extends DataTypeName, CT extends GetDataConditions<T> | string>(
  type: T,
  cond: CT,
  options?: {
    sort?: SortOption<DataType<T>>;
    limit?: number;
    skip?: number;
    showAlertOnError?: boolean;
    disable?: boolean;
    startWithLoadingState?: boolean;
    onInitialLoad?: () => void;
  }
): {
  loading: boolean;
  data: null | (CT extends string ? ValidDataTypeWithID<T> | InvalidDataTypeWithID<T> : ReadonlyArray<ValidDataTypeWithID<T> | InvalidDataTypeWithID<T>>);
  reload: () => Promise<void>;
  refresh: () => Promise<void>;
  refreshing: boolean;
}
```

**调用示例**
```typescript
// 查询单个记录
const { data: item } = useData('item', 'item-123');

// 查询多个记录
const { data: items } = useData('item', ['item-123', 'item-456']);

// 条件查询
const { data: activeItems } = useData('item', { active: true }, { sort: { name: 'asc' } });
```

**注意事项**
- `cond`参数可以是字符串（ID）、字符串数组（ID列表）或条件对象
- 使用`useFocusEffect`确保在屏幕获得焦点时重新加载数据
- 内部使用`deep-object-diff`库进行深度比较，确保条件变化时正确触发重新加载
- `reload`和`refresh`方法的区别：`reload`立即执行，而`refresh`会设置`refreshing`状态

**Section sources**
- [useData.ts](file://App/app/data/hooks/useData.ts#L1-L225)

### useConfig：应用配置管理
`useConfig` Hook用于管理应用配置，提供加载、更新和刷新配置的功能。

**TypeScript类型定义**
```typescript
function useConfig(): {
  loading: boolean;
  config: ConfigType | null;
  updateConfig: (cfg: Partial<ConfigType>) => Promise<boolean>;
  reload: () => void;
  refresh: () => void;
  refreshing: boolean;
}
```

**调用示例**
```typescript
const { config, updateConfig, loading } = useConfig();

// 更新配置
const handleUpdate = async () => {
  await updateConfig({ theme: 'dark' });
};

// 使用配置
if (config) {
  const theme = config.theme;
}
```

**注意事项**
- 配置数据存储在数据库中，因此需要数据库连接
- `updateConfig`方法返回布尔值表示更新是否成功
- 使用`useFocusEffect`确保在屏幕获得焦点时重新加载配置
- 错误处理中包含`showAlert: true`选项，会在出现错误时显示警告

**Section sources**
- [useConfig.ts](file://App/app/data/hooks/useConfig.ts#L1-L90)

### 其他数据相关Hook
除了上述主要Hook外，项目还提供了其他数据相关Hook：

- `useDataCount`：查询满足条件的记录数量
- `useRelated`：查询相关联的数据记录
- `useSave`：创建、更新或删除数据记录
- `useView`：查询视图数据

这些Hook都遵循类似的模式：返回加载状态、数据、重新加载和刷新功能。

**Section sources**
- [useDataCount.ts](file://App/app/data/hooks/useDataCount.ts#L1-L110)
- [useRelated.ts](file://App/app/data/hooks/useRelated.ts#L1-L182)
- [useSave.ts](file://App/app/data/hooks/useSave.ts#L1-L115)
- [useView.ts](file://App/app/data/hooks/useView.ts#L1-L114)

## 状态管理Hook

### usePersistedState：持久化本地状态
`usePersistedState` Hook用于持久化本地状态，将状态保存到`AsyncStorage`中，即使应用重启也能恢复。

**TypeScript类型定义**
```typescript
function usePersistedState<T>(
  key: string,
  initialState: T | (() => T)
): [T, React.Dispatch<React.SetStateAction<T>>, boolean]
```

**调用示例**
```typescript
const [theme, setTheme, isInitialized] = usePersistedState<string>('app-theme', 'light');

// 状态变化会自动保存到AsyncStorage
useEffect(() => {
  if (isInitialized) {
    // 状态已初始化，可以安全使用
    console.log('Current theme:', theme);
  }
}, [theme, isInitialized]);
```

**注意事项**
- 状态通过`AsyncStorage`持久化，键名为`@use_persisted_state/${key}`
- 返回第三个值`isInitialized`表示状态是否已从存储中加载完成
- 在`useEffect`中设置状态时，依赖项应包含`key`以确保正确性
- 初始状态可以是值或返回值的函数，与`useState`一致

**Section sources**
- [usePersistedState.ts](file://App/app/hooks/usePersistedState.ts#L1-L31)

### useDeepCompare：深度比较依赖项
`useDeepCompare` Hook用于深度比较两个对象，通常用于`useEffect`或`useMemo`的依赖项数组中，避免因对象引用变化导致的不必要重新渲染。

**TypeScript类型定义**
```typescript
function useDeepCompare(obj1: unknown, obj2: unknown): boolean
```

**调用示例**
```typescript
// 在自定义Hook中使用
function useMyCustomHook(options) {
  const hasChanged = useDeepCompare(prevOptions, options);
  
  useEffect(() => {
    if (hasChanged) {
      // 选项已改变，执行相应逻辑
      console.log('Options changed');
    }
  }, [hasChanged]);
}

// 直接在useEffect中使用
useEffect(() => {
  // 执行副作用
}, [useDeepCompare(dependency1, dependency2)]);
```

**注意事项**
- 使用`deep-object-diff`库进行深度比较，能准确识别对象内容的变化
- 返回布尔值：`true`表示两个对象相等，`false`表示不相等
- 在`useMemo`中使用，确保只有当对象内容真正改变时才重新计算
- 避免在大型复杂对象上频繁使用，以免影响性能

**Section sources**
- [useDeepCompare.ts](file://App/app/hooks/useDeepCompare.ts#L1-L32)

## UI相关Hook

### useTabBarInsets：处理底部安全区
`useTabBarInsets` Hook用于处理iOS设备底部标签栏的安全区，计算适当的内边距以避免内容被标签栏遮挡。

**TypeScript类型定义**
```typescript
function useTabBarInsets(): {
  bottom: number;
  scrollViewBottom: number;
}
```

**调用示例**
```typescript
const { bottom, scrollViewBottom } = useTabBarInsets();

return (
  <View style={{ marginBottom: bottom }}>
    <ScrollView contentContainerStyle={{ paddingBottom: scrollViewBottom }}>
      {/* 内容 */}
    </ScrollView>
  </View>
);
```

**注意事项**
- 仅在iOS平台上应用特定的计算逻辑
- `bottom`值用于普通视图的底部边距
- `scrollViewBottom`值用于滚动视图的内容底部内边距，通常为`bottom`的一半加上额外偏移
- 使用`react-native-safe-area-context`获取安全区信息
- 在Android上返回默认值0，不进行特殊处理

**Section sources**
- [useTabBarInsets.ts](file://App/app/hooks/useTabBarInsets.ts#L1-L18)

### useScrollViewContentInsetFix：解决滚动视图布局问题
`useScrollViewContentInsetFix` Hook用于解决滚动视图的内容内边距问题，通过强制滚动到特定位置来修复布局异常。

**TypeScript类型定义**
```typescript
function useScrollViewContentInsetFix(
  scrollViewRef: React.MutableRefObject<{
    scrollTo: (o: { x?: number; y?: number; animated?: boolean }) => void;
  } | null>
): void
```

**调用示例**
```typescript
const scrollViewRef = useRef(null);
useScrollViewContentInsetFix(scrollViewRef);

return (
  <ScrollView ref={scrollViewRef}>
    {/* 内容 */}
  </ScrollView>
);
```

**注意事项**
- 接收滚动视图的引用作为参数
- 在`useEffect`中使用`scrollTo`方法滚动到`(-80, -80)`位置，以修复内容内边距
- 动画设置为`false`，实现瞬间跳转
- 依赖项为`scrollViewRef`，确保引用变化时重新应用修复
- 主要用于解决iOS上滚动视图初始渲染时的内容偏移问题

**Section sources**
- [useScrollViewContentInsetFix.ts](file://App/app/hooks/useScrollViewContentInsetFix.ts#L1-L15)

## 最佳实践与注意事项
1. **避免无限循环渲染**：
   - 在`useEffect`或`useMemo`的依赖项中使用`useDeepCompare`
   - 确保依赖项数组中的对象不会在每次渲染时创建新引用

2. **内存泄漏预防**：
   - 在`useEffect`中返回清理函数，清除定时器、订阅等
   - 对于异步操作，检查组件是否仍挂载再更新状态

3. **性能优化**：
   - 使用`useMemo`缓存计算结果
   - 使用`useCallback`缓存函数引用
   - 避免在渲染过程中进行昂贵的计算

4. **错误处理**：
   - 在数据操作中妥善处理数据库连接不可用的情况
   - 使用`try-catch`捕获异步操作中的错误
   - 提供适当的用户反馈机制

5. **类型安全**：
   - 充分利用TypeScript的泛型特性
   - 为自定义Hook提供精确的类型定义
   - 避免使用`any`类型

## 总结
本指南详细介绍了项目中的各类自定义Hook，包括数据访问、状态管理和UI处理等方面。这些Hook通过封装常见模式和复杂逻辑，提高了代码的复用性和可维护性。开发者在使用时应遵循最佳实践，注意避免常见陷阱，充分发挥React Hook的优势。