# Immer 源码解析

## 1. 概述

Immer 是一个不可变数据结构库，它通过提供一种简单的 API 来实现不可变状态更新。Immer 的核心理念是让开发者能够以直接修改对象的方式来创建不可变数据的下一个状态。Immer 使用了 JavaScript 的 Proxy 特性来实现这一功能。

## 2. 源码结构

```
src/
├── core/                # 核心实现
│   ├── immerClass.ts    # Immer 主类
│   ├── proxy.ts         # Proxy 实现
│   ├── scope.ts         # 作用域管理
│   ├── finalize.ts      # 状态最终化处理
│   └── current.ts       # 获取当前状态
├── plugins/             # 插件系统
├── utils/               # 辅助函数
├── types/               # 类型定义
├── immer.ts             # 主入口
└── internal.ts          # 内部导出
```

## 3. 核心实现解析

### 3.1 `immer.ts` - 主入口文件

```typescript
// 创建一个 Immer 实例
const immer = new Immer()

/**
 * produce 函数接收一个初始状态和一个修改函数
 * 修改函数可以自由地修改传入的草稿状态
 * 所有修改只会应用到初始状态的副本上
 */
export const produce = immer.produce
```

主要职责：
- 创建 Immer 实例
- 导出核心 API 函数（produce, produceWithPatches 等）

### 3.2 `immerClass.ts` - Immer 类实现

```typescript
export class Immer implements ProducersFns {
  // 是否自动冻结对象
  autoFreeze_: boolean = true 
  
  // produce 函数实现
  produce: IProduce = (base: any, recipe?: any, patchListener?: any) => {
    // 处理柯里化调用
    if (typeof base === "function" && typeof recipe !== "function") {
      // ...
    }
    
    // 只有可草稿化的对象才会被处理
    if (isDraftable(base)) {
      const scope = enterScope(this)
      const proxy = createProxy(base, undefined)
      try {
        result = recipe(proxy)
        // ...
      } finally {
        // ...
      }
      return processResult(result, scope)
    } else {
      // 处理原始类型
      // ...
    }
  }
  
  // 带补丁的 produce 函数
  produceWithPatches: IProduceWithPatches = (base: any, recipe?: any): any => {
    // ...返回 [result, patches, inversePatches]
  }
  
  // 手动创建草稿
  createDraft<T extends Objectish>(base: T): Draft<T> {
    // 创建草稿但不立即处理
  }
  
  // 完成草稿处理
  finishDraft<D extends Draft<any>>(draft: D, patchListener?: PatchListener) {
    // 完成草稿处理并返回结果
  }
  
  // 应用补丁
  applyPatches<T extends Objectish>(base: T, patches: readonly Patch[]): T {
    // 应用补丁到对象上
  }
}
```

关键函数解释：
- `produce`: 核心函数，处理状态转换逻辑
- `produceWithPatches`: 类似 produce，但也返回补丁信息
- `createDraft`/`finishDraft`: 手动创建和完成草稿的函数
- `applyPatches`: 应用补丁到对象

### 3.3 `proxy.ts` - 代理实现

```typescript
// 创建代理对象
export function createProxyProxy<T extends Objectish>(
  base: T,
  parent?: ImmerState
): Drafted<T, ProxyState> {
  const isArray = Array.isArray(base)
  const state: ProxyState = {
    // 对象类型（数组或对象）
    type_: isArray ? ArchType.Array : ArchType.Object,
    // 当前作用域
    scope_: parent ? parent.scope_ : getCurrentScope()!,
    // 是否被修改
    modified_: false,
    // 是否已完成
    finalized_: false,
    // 跟踪哪些属性被赋值或删除
    assigned_: {},
    // 父状态
    parent_: parent,
    // 基础状态（原始对象）
    base_: base,
    // 代理对象（将设置）
    draft_: null as any,
    // 对象的副本（延迟创建）
    copy_: null,
    // 撤销函数
    revoke_: null as any,
    // 是否为手动模式
    isManual_: false
  }
  
  // 创建代理
  const {revoke, proxy} = Proxy.revocable(
    isArray ? [state] as any : state as any,
    isArray ? arrayTraps : objectTraps
  )
  state.draft_ = proxy as any
  state.revoke_ = revoke
  return proxy as any
}

// 对象代理拦截器
export const objectTraps: ProxyHandler<ProxyState> = {
  // 获取属性
  get(state, prop) {
    // 处理特殊属性和代理状态获取
    if (prop === DRAFT_STATE) return state
    
    const source = latest(state)
    const value = source[prop]
    
    // 如果值可以被草稿化且尚未被修改
    if (!state.finalized_ && isDraftable(value)) {
      // 检查是否需要创建子代理
      if (value === peek(state.base_, prop)) {
        prepareCopy(state)
        return (state.copy_![prop] = createProxy(value, state))
      }
    }
    return value
  },
  
  // 设置属性
  set(state, prop, value) {
    // 处理 setter
    const desc = getDescriptorFromProto(latest(state), prop)
    if (desc?.set) {
      desc.set.call(state.draft_, value)
      return true
    }
    
    // 检查是否需要标记为修改
    if (!state.modified_) {
      const current = peek(latest(state), prop)
      // 如果值相同，无需修改
      if (is(value, current) && (value !== undefined || has(state.base_, prop)))
        return true
      
      // 准备副本并标记为已修改
      prepareCopy(state)
      markChanged(state)
    }
    
    // 设置新值
    state.copy_![prop] = value
    state.assigned_[prop] = true
    return true
  },
  
  // 删除属性
  deleteProperty(state, prop) {
    // 标记为已删除
    if (peek(state.base_, prop) !== undefined || prop in state.base_) {
      state.assigned_[prop] = false
      prepareCopy(state)
      markChanged(state)
    }
    if (state.copy_) {
      delete state.copy_[prop]
    }
    return true
  },
  
  // 其他拦截器...
}
```

关键函数解释：
- `createProxyProxy`: 创建代理对象，跟踪对象状态
- `objectTraps`: 对象代理拦截器，处理属性读取/写入/删除
- `arrayTraps`: 数组代理拦截器，特殊处理数组操作
- `markChanged`: 标记对象已修改
- `prepareCopy`: 准备对象副本（延迟复制优化）

### 3.4 `finalize.ts` - 最终化处理

```typescript
// 处理 produce 函数的结果
export function processResult(result: any, scope: ImmerScope) {
  scope.unfinalizedDrafts_ = scope.drafts_.length
  const baseDraft = scope.drafts_![0]
  const isReplaced = result !== undefined && result !== baseDraft
  
  if (isReplaced) {
    // 处理返回新值的情况
    if (isDraftable(result)) {
      result = finalize(scope, result)
      // 可能冻结结果
      if (!scope.parent_) maybeFreeze(scope, result)
    }
    
    // 生成替换补丁
    if (scope.patches_) {
      getPlugin("Patches").generateReplacementPatches_(/*...*/)
    }
  } else {
    // 处理修改草稿的情况
    result = finalize(scope, baseDraft, [])
  }
  
  // 清理和补丁处理
  revokeScope(scope)
  if (scope.patches_) {
    scope.patchListener_!(scope.patches_, scope.inversePatches_!)
  }
  
  return result !== NOTHING ? result : undefined
}

// 将草稿最终化为不可变对象
function finalize(rootScope: ImmerScope, value: any, path?: PatchPath) {
  // 已冻结对象直接返回
  if (isFrozen(value)) return value
  
  const state: ImmerState = value[DRAFT_STATE]
  
  // 处理普通对象（非草稿）
  if (!state) {
    each(value, (key, childValue) =>
      finalizeProperty(rootScope, state, value, key, childValue, path)
    )
    return value
  }
  
  // 不处理其他作用域的草稿
  if (state.scope_ !== rootScope) return value
  
  // 未修改的草稿，返回（可能被冻结的）原始值
  if (!state.modified_) {
    maybeFreeze(rootScope, state.base_, true)
    return state.base_
  }
  
  // 最终化已修改但未完成的草稿
  if (!state.finalized_) {
    state.finalized_ = true
    state.scope_.unfinalizedDrafts_--
    
    // 处理所有子属性的最终化
    // ...
    
    // 可能冻结结果
    maybeFreeze(rootScope, state.copy_, false)
    
    // 生成补丁
    if (path && rootScope.patches_) {
      getPlugin("Patches").generatePatches_(/*...*/)
    }
  }
  
  return state.copy_
}
```

关键函数解释：
- `processResult`: 处理 produce 函数的结果
- `finalize`: 将草稿转换为最终的不可变状态
- `finalizeProperty`: 处理单个属性的最终化
- `maybeFreeze`: 在适当条件下冻结对象

### 3.5 `scope.ts` - 作用域管理

```typescript
/** 每个作用域代表一次 produce 调用 */
export interface ImmerScope {
  // 补丁和反向补丁
  patches_?: Patch[]
  inversePatches_?: Patch[]
  // 是否可以自动冻结
  canAutoFreeze_: boolean
  // 当前作用域的草稿列表
  drafts_: any[]
  // 父作用域
  parent_?: ImmerScope
  // 补丁监听器
  patchListener_?: PatchListener
  // Immer 实例
  immer_: Immer
  // 未完成的草稿数量
  unfinalizedDrafts_: number
}

// 进入一个新的作用域
export function enterScope(immer: Immer) {
  return (currentScope = createScope(currentScope, immer))
}

// 离开当前作用域
export function leaveScope(scope: ImmerScope) {
  if (scope === currentScope) {
    currentScope = scope.parent_
  }
}

// 撤销作用域
export function revokeScope(scope: ImmerScope) {
  leaveScope(scope)
  scope.drafts_.forEach(revokeDraft)
  scope.drafts_ = null as any
}
```

关键函数解释：
- `enterScope`: 进入新的作用域，创建新的上下文
- `leaveScope`: 离开当前作用域，恢复父作用域
- `revokeScope`: 撤销作用域，释放所有草稿
- `usePatchesInScope`: 在作用域中启用补丁跟踪

### 3.6 `common.ts` - 通用工具函数

```typescript
/** 判断值是否为草稿 */
export function isDraft(value: any): boolean {
  return !!value && !!value[DRAFT_STATE]
}

/** 判断值是否可以被草稿化 */
export function isDraftable(value: any): boolean {
  if (!value) return false
  return (
    isPlainObject(value) ||
    Array.isArray(value) ||
    !!value[DRAFTABLE] ||
    !!value.constructor?.[DRAFTABLE] ||
    isMap(value) ||
    isSet(value)
  )
}

/** 获取草稿的原始对象 */
export function original(value: Drafted<any>): any {
  if (!isDraft(value)) die(15, value)
  return value[DRAFT_STATE].base_
}

/** 获取当前的对象状态（副本或原始） */
export function latest(state: ImmerState): any {
  return state.copy_ || state.base_
}

/** 浅拷贝对象 */
export function shallowCopy(base: any, strict: StrictMode) {
  // 根据不同类型创建浅拷贝
  if (isMap(base)) return new Map(base)
  if (isSet(base)) return new Set(base)
  if (Array.isArray(base)) return Array.prototype.slice.call(base)
  
  // 处理普通对象
  // ...
}

/** 冻结对象 */
export function freeze<T>(obj: any, deep: boolean = false): T {
  if (isFrozen(obj) || isDraft(obj) || !isDraftable(obj)) return obj
  // 冻结不同类型的对象
  // ...
}
```

关键函数解释：
- `isDraft`: 检查对象是否为草稿
- `isDraftable`: 检查对象是否可被草稿化
- `original`: 获取草稿的原始对象
- `latest`: 获取最新状态（副本或原始）
- `shallowCopy`: 创建对象的浅拷贝
- `freeze`: 冻结对象，可选深度冻结

## 4. 核心流程

Immer 的工作流程可以总结为以下几个步骤：

1. **初始化**：创建 Immer 实例
2. **调用 produce**：传入原始状态和修改函数
3. **创建代理**：为原始状态创建代理对象（草稿）
4. **执行用户函数**：执行修改函数，对草稿进行修改
5. **跟踪变更**：通过 Proxy 拦截器跟踪所有修改
6. **最终化**：将草稿转换为最终的不可变状态
   - 未修改的属性直接引用原始状态
   - 已修改的属性创建新副本
7. **生成补丁**（可选）：如果指定了补丁监听器，生成变更补丁
8. **返回结果**：返回新的不可变状态

## 5. 性能优化

Immer 采用了多种优化措施：

1. **结构共享**：未修改的部分直接引用原始状态，避免不必要的复制
2. **延迟复制**：只有在属性被修改时才创建副本（copy-on-write）
3. **缓存检测**：使用 `modified_` 标志避免重复检查

## 6. 使用方法

### 6.1 基本用法

```javascript
import produce from "immer"

const baseState = {
  users: [{ name: "张三" }],
  settings: { theme: "dark" }
}

const nextState = produce(baseState, draft => {
  draft.users.push({ name: "李四" })
  draft.settings.theme = "light"
})
```

### 6.2 柯里化用法

```javascript
const updateUser = produce((draft, name) => {
  draft.users.push({ name })
})

const nextState = updateUser(baseState, "王五")
```

### 6.3 补丁和撤销

```javascript
import { produceWithPatches, applyPatches } from "immer"

const [nextState, patches, inversePatches] = produceWithPatches(
  baseState,
  draft => {
    draft.users.push({ name: "赵六" })
  }
)

// 应用补丁
const patchedState = applyPatches(baseState, patches)

// 撤销操作（应用反向补丁）
const revertedState = applyPatches(nextState, inversePatches)
```

## 7. 总结

Immer 的核心优势在于：

1. **简化复杂嵌套状态更新**：提供直接修改的编程模型
2. **保持不可变性**：内部自动处理不可变转换
3. **结构共享**：优化性能和内存使用
4. **补丁功能**：支持变更跟踪和时间旅行功能

Immer 通过巧妙使用 JavaScript Proxy，实现了一种既易于使用又保持不可变性的状态管理方案，是现代前端状态管理的优秀工具。 