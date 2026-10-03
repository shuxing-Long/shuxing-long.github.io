## 所属学科

这两个概念属于以下学科体系：

| 层级 | 学科 | 说明 |
|------|------|------|
| **一级学科** | **计算机科学与技术** (Computer Science) | ALC/BCL 属于编程语言运行时系统的核心机制 |
| **二级学科** | **软件工程** (Software Engineering) | 涉及组件化架构、模块隔离、插件系统设计模式 |
| **具体方向** | **.NET 运行时 / CLR (Common Language Runtime)** | CLR 是 .NET 平台的虚拟机实现，ALC 和 BCL 是其两大关键子系统 |
| **相关理论** | **程序集加载与类型系统** (Assembly Loading & Type Identity) | 研究运行时如何定位、加载、缓存程序集，以及类型标识如何在不同加载上下文中保持唯一性 |
| **架构模式** | **插件架构 (Plugin Architecture)** / **依赖隔离 (Dependency Isolation)** | ALC 是实现插件热加载与卸载的核心机制，是软件可扩展性设计的底层支撑 |

### 学科背景简述

- **BCL（基础类库）** 属于**标准库设计**范畴，类似 Java 的 Class Library、C++ 的 Standard Library。它定义了开发者与运行时交互的基础契约。
- **ALC（程序集加载上下文）** 属于**运行时虚拟化**范畴，类似 Java 的 `ClassLoader`、OSGi 的 Bundle ClassLoader。它解决的核心问题是：在同一进程中如何让多个组件（插件）各自拥有独立的类型空间，互不干扰，且支持动态卸载。
- 二者共同构成了 .NET **插件隔离架构**的基石，是《软件工程》中「高内聚、低耦合」原则在运行时层面的工程实践。

---

## 一、BCL（Base Class Library）

### 是什么

BCL 是 [.Net 运行时](#/viewNotes?id=A-007-net%2F%E8%BF%90%E8%A1%8C%E6%97%B6%E6%A6%82%E5%BF%B5%E8%A7%A3%E9%87%8A)自带的**基础类库**，随 .NET SDK/Runtime 一起安装。所有 .NET 程序无需额外 NuGet 引用就能直接使用。

### 常见 BCL 类型

| 命名空间 | 典型类型 | 用途 |
|----------|---------|------|
| `System` | `String`, `Int32`, `DateTime`, `Console`, `GC` | 基础类型、控制台、垃圾回收 |
| `System.IO` | `File`, `Directory`, `Path`, `Stream` | 文件系统操作 |
| `System.Threading` | `Thread`, `Task`, `CancellationToken` | 线程和异步编程 |
| `System.Threading.Channels` | `Channel<T>`, `ChannelReader<T>`, `ChannelWriter<T>` | 生产者-消费者内存队列 |
| `System.Collections` | `List<T>`, `Dictionary<K,V>` | 集合类型 |
| `System.Reflection` | `Assembly`, `Type`, `MethodInfo` | 运行时类型信息 |
| `System.Text.Json` | `JsonSerializer`, `JsonDocument` | JSON 序列化 |
| `System.Linq` | `Enumerable` 扩展方法 | 集合查询 |
| `System.Net.Http` | `HttpClient` | HTTP 网络请求 |
| `Microsoft.Extensions.*` | `IConfiguration`, `ILogger<T>` | DI 容器、配置、日志抽象 |

### 关键特性

- **不需要打包进 ZIP**：插件 ZIP 里不用包含 `System.Runtime.dll` 之类的 BCL 程序集
- **所有 ALC 共享同一份 BCL**：默认 ALC 和自定义 ALC 共用同一套 BCL 类型
- **跨 ALC 传 BCL 类型是安全的**：`string`、`int`、`byte[]` 等 BCL 类型在任何 ALC 中都是同一个类型，不会出现"同名但不同类型"的问题

## 二、ALC（AssemblyLoadContext）

### 是什么

`AssemblyLoadContext`（程序集加载上下文）是 .NET 中用于**隔离加载程序集**的机制。每个 ALC 有自己的程序集缓存，不同 ALC 加载的同名 DLL 被视为不同的程序集，类型也互不兼容。

### 默认 ALC（Default ALC）

- .NET 运行时启动时自动创建
- 加载主程序（`.exe`/.`dll`）及其所有直接/传递依赖
- 所有 `AssemblyLoadContext.Default` 共享同一个实例

### 自定义 ALC

- 通过继承 `AssemblyLoadContext` 创建
- 可设置 `isCollectible: true`，支持卸载（释放内存）
- 重写 `Load(AssemblyName)` 方法控制加载策略

### 本项目的 TaskAssemblyLoadContext

```csharp
// 三级加载策略（实现在 CardInsertionMachine-Quartz/Services/AppDomainHotPluggableTaskBase.cs）

public class TaskAssemblyLoadContext : AssemblyLoadContext
{
    protected override Assembly? Load(AssemblyName assemblyName)
    {
        // 第一级：临时目录（ZIP 解压后的插件 DLL，最高优先级）
        var localPath = Path.Combine(_basePath, assemblyName.Name + ".dll");
        if (File.Exists(localPath))
            return LoadFromAssemblyPath(localPath);

        // 第二级：deps.json → NuGet 缓存（第三方依赖，如 SQLite Provider）
        if (_resolver != null)
        {
            var resolvedPath = _resolver.ResolveAssemblyToPath(assemblyName);
            if (resolvedPath != null)
                return LoadFromAssemblyPath(resolvedPath);
        }

        // 第三级：返回 null，委托给 Default ALC（BCL 类型、框架程序集）
        return null;
    }
}
```

### 为什么需要 ALC 隔离

```
无 ALC 隔离（错误方式）：
  宿主加载 ScheduledTasks.dll → 全局缓存已存在
  宿主加载 ScheduledTasks.Desktop.dll → 同名类型冲突 ❌

有 ALC 隔离（正确方式）：
  宿主（Default ALC）→ 不加载插件
  插件 ALC-1 → 加载 ScheduledTasks.dll（独立缓存）
  插件 ALC-2 → 加载 ScheduledTasks.Desktop.dll（独立缓存）
  两个 ALC 的类型互不可见，不会冲突 ✅
```

## 三、ALC 类型隔离带来的问题

### 核心问题

不同 ALC 加载的同名程序集是**不同的程序集**，其中的同名类型是**不同的类型**。

```
Default ALC:   CardInsertionMachine-Desktop.exe
               ├── LogMessageModel  (类型标识: [Default ALC]+[主程序集])

Plugin ALC:    ScheduledTasks.Desktop.dll
               ├── LogMessageModel  (类型标识: [Plugin ALC]+[插件程序集])
```

这两个 `LogMessageModel` 在运行时是**不同的类型**，即使字段完全一样也不能互换。

### 实际影响

```csharp
// 宿主（Default ALC）
var msg = new LogMessageModel { Content = "hello" };
Channel<LogMessageModel>.Writer.WriteAsync(msg);  // 写入的是 Default ALC 的 LogMessageModel

// 插件（Plugin ALC）
await foreach (var m in reader.ReadAllAsync())  // 期望读取 Plugin ALC 的 LogMessageModel
{
    // ❌ 类型不匹配，抛出 InvalidCastException！
}
```

### 解决方案

| 方案 | 原理 | 适用场景 |
|------|------|----------|
| **共享 Contracts 程序集** | 将共享类型放入单独的 DLL，插件 ALC 走 fallback（返回 null → Default ALC），从而复用宿主已加载的同版本 DLL | 需要跨 ALC 传递复杂对象 |
| **BCL 类型传递** | 使用 `string`、`byte[]` 等 BCL 类型，这些在所有 ALC 中统一 | 简单数据传递，需自行序列化 |
| **避免跨 ALC 通信** | 架构上不传递对象，插件只收配置 JSON | **本项目采用的方式** |

## 四、本项目中的应用

### 插件加载全流程

```
1. 用户上传 ScheduledTasks.Desktop.zip → 存入 PluginZips/
2. Quartz 触发任务 → PluginLoaderService
3. 解压 ZIP → PluginExtracted/{JobName}/
4. 创建 TaskAssemblyLoadContext (isCollectible: true)
5. ALC.LoadFromAssemblyPath(mainDll)  → 获取 Type
6. 反射调用构造函数 new PluginClass(configJson)
7. 反射调用 Execute() → await Task
8. 任务完成 → ALC.Unload() → 删除临时目录
```

### ALC 加载 DLL 时三级查找实例

插件 `ScheduledTasks.Desktop.dll` 引用了 `Microsoft.EntityFrameworkCore.Sqlite.dll`：

```
请求加载: Microsoft.EntityFrameworkCore.Sqlite.dll
  ↓
第一级: PluginExtracted/{JobName}/Microsoft.EntityFrameworkCore.Sqlite.dll
  → 不存在（未打包进 ZIP，由 deps.json 描述）
  ↓
第二级: deps.json → NuGet 缓存 C:\Users\xxx\.nuget\packages\microsoft.entityframeworkcore.sqlite\...
  → 找到，加载 ✅
```

插件引用了 `System.Runtime.dll`（BCL）：

```
请求加载: System.Runtime.dll
  ↓
第一级: 不存在
  ↓
第二级: deps.json 不包含 BCL
  ↓
第三级: 返回 null → Default ALC 提供
  → 使用宿主已加载的 BCL ✅
```

### 为什么桌面版不搞 Channel 消费模式

`Channel<LogMessageModel>` 的泛型参数 `LogMessageModel` 如果是自定义类型：
- 宿主的 `LogMessageModel` 在 Default ALC
- 插件的 `LogMessageModel` 在 Plugin ALC
- 两者是不同类型 → 无法通过 Channel 传递

**结论**：跨 ALC 传递自定义对象需要额外的架构设计（共享程序集、序列化等），与"保持插件独立性"的目标冲突。桌面版选择**不走这条路**，插件只通过 configJson 接收配置，自主执行。

---

