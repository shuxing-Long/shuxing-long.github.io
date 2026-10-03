---
tags:
  - net
  - CSharp
  - async/await
  - 异步编程
  - ASPNETCore
  - HTTP
  - 线程池
---

# .NET 同步 API 改为 `async/await` 异步 API 知识笔记

## 0. 核心结论

- 后端从同步 API 改成 `async/await` 异步 API，**前端通常不需要调整**。
- 前端依赖的是 **HTTP 接口契约**，不是后端 C# 方法签名。
- `async/await` 是服务端内部实现细节，不改变 URL、请求参数、响应 JSON、状态码、错误格式。
- 前端仍然要等待 HTTP 响应；后端 `async/await` 改变的是**等待 I/O 时线程的使用方式**。
- `async/await` 擅长 I/O 密集型场景，不擅长 CPU 密集型计算。

---

## 1. 前端通常不需要调整

### 1.1 后端同步写法

```csharp
[HttpGet("{id}")]
public ActionResult<User> Get(int id)
{
    return Ok(_service.GetUser(id));
}
```

### 1.2 后端异步写法

```csharp
[HttpGet("{id}")]
public async Task<ActionResult<User>> GetAsync(int id)
{
    var user = await _service.GetUserAsync(id);
    return Ok(user);
}
```

### 1.3 前端调用不变

```js
const res = await fetch(`/api/users/${id}`);
const user = await res.json();
```

前端仍然是异步调用，仍然等待网络响应。

---

## 2. 为什么前端不需要调整

### 2.1 前后端边界是 HTTP，不是 C# 方法签名

前端看到的是：

```http
GET /api/users/1
```

以及：

- URL
- HTTP 方法
- 请求头
- 请求体
- 状态码
- 响应 JSON
- 错误格式

前端看不到：

- `Task<T>`
- `async`
- `await`
- 线程池
- 状态机

这些都属于服务端内部实现。

### 2.2 `async/await` 不改变请求和响应内容

只要最终返回的数据一样，JSON 就一样。

同步：

```csharp
return Ok(_service.GetUser(id));
```

异步：

```csharp
var user = await _service.GetUserAsync(id);
return Ok(user);
```

客户端拿到的仍然是：

```json
{ "id": 1, "name": "Alice" }
```

### 2.3 前端本来就是异步调用

浏览器里的 `fetch`、`axios`、`XMLHttpRequest` 本来就是异步的。

```js
const res = await fetch('/api/users/1');
const user = await res.json();
```

这里的 `await` 等的是**网络响应**，不是后端的 C# `Task`。

所以后端同步时前端这么写，后端异步时前端还是这么写。

### 2.4 ASP.NET Core 会透明处理 `Task`

后端返回：

```csharp
Task<ActionResult<User>>
```

ASP.NET Core 会等待这个 `Task` 完成，然后取其中的 `ActionResult<User>`，再写 HTTP 响应。

它不会把 `Task` 对象序列化给前端。

### 2.5 OpenAPI / Swagger / 生成客户端通常不变

`Task<ActionResult<T>>` 和 `ActionResult<T>` 生成的 OpenAPI schema 通常一样。

TypeScript 生成的客户端通常仍然是：

```ts
getUser(id: number): Promise<User>
```

调用方式不变。

---

## 3. 什么情况下前端可能需要调整

虽然单纯加 `async/await` 通常不影响前端，但以下情况可能需要调整：

1. **接口契约变了**
   - URL 变了
   - HTTP 方法变了
   - 请求参数变了
   - 响应结构变了
   - 状态码变了
   - 错误格式变了

2. **前端是 Blazor / .NET 直接调用服务方法**
   - 同步方法改成 `Task<T>` 后，调用处必须 `await`。
   - 否则拿到的是 `Task` 对象，不是结果。

3. **生成的是 C# 客户端**
   - C# 客户端调用处需要 `await`。
   - TypeScript 客户端通常不受影响。

4. **后端行为变化**
   - `async void`
   - 忘记 `await`
   - 异常处理中间件没覆盖
   - 可能导致前端收到空响应、500 或超时。

5. **新增 `CancellationToken`**
   - 如果绑定到请求中止，前端用 `AbortController` 取消请求时，后端可能真的取消操作。

6. **超时、并发、重试策略**
   - 异步通常提升吞吐量，但不保证单个请求更快。
   - 前端 loading、超时、重试策略可按实际表现微调。

判断标准：

> 只要 HTTP API 契约没变，前端就不需要因为 `async/await` 而改。

---

## 4. 前端仍然要等，后端归还线程

### 4.1 前端等的是 HTTP 响应

```js
const res = await fetch('/api/users/1');
```

对前端来说，请求没完成就是没完成。

`await` 等的是网络响应，不是后端的 C# `Task`。

### 4.2 后端 `async/await` 等的是 I/O

```csharp
public async Task<IActionResult> Get()
{
    var data = await _db.QueryAsync(); // 等数据库
    return Ok(data);
}
```

执行到 `await _db.QueryAsync()` 时：

- 如果数据库还没返回；
- 当前线程不会被卡在那里干等；
- 线程可以归还给线程池；
- 线程池可以去处理别的请求；
- 等数据库结果好了，再继续执行后面的代码；
- 最后写 HTTP 响应。

### 4.3 收益不是“单个请求更快”

`async/await` 主要提升：

- 并发能力
- 线程利用率
- 高负载下的可伸缩性

通常不会让单个请求明显变快，甚至可能有一点点状态机开销。

### 4.4 类比

餐厅点餐：

- 顾客：还是要等菜上来。
- 同步后端：厨师站着等食材送到，期间不能做别的。
- 异步后端：厨师把单子挂起，先去炒别的菜，食材到了再继续。

顾客感受还是“我在等菜”，但餐厅整体能接更多单。

### 4.5 例外

- 如果后端是 **CPU 密集计算**，`async/await` 不会归还线程。
- 如果后端改成“立即返回 202，后台慢慢处理”，前端才需要改成轮询、WebSocket 或回调。
- 单纯把同步方法改成 `async/await` 不属于这种情况。

---

## 5. 什么是 CPU 密集计算

CPU 密集型是指：任务的时间主要花在 **CPU 执行指令**上，而不是等网络、磁盘、数据库返回。

典型例子：

- 大量循环、复杂数学运算
- 加密、解密、哈希
- 压缩、解压
- 图像 / 视频处理
- 复杂排序、搜索、算法
- 大对象序列化 / 反序列化
- 正则表达式严重回溯

特点：

- CPU 利用率高
- 几乎没有“我在等别人”的空档
- 快慢主要取决于 CPU 速度和核数

---

## 6. 为什么 CPU 密集计算时 `async/await` 不会归还线程

### 6.1 `async/await` 能归还线程的前提

`async/await` 能在 `await` 处把当前线程归还给线程池，前提是：

> 当前线程遇到一个尚未完成的异步操作，而且这个操作的完成不需要当前线程继续干活。

例如：

```csharp
var html = await httpClient.GetStringAsync(url);
```

执行到 `await` 时：

- 网络请求还没回来；
- 网络 I/O 由操作系统和网卡处理；
- 当前线程不需要参与等待；
- 所以线程可以返回线程池；
- 等网络数据到了，再调度一个线程继续执行。

关键点：

> 等待期间，有“别人”在干活，CPU 线程可以空出来。

### 6.2 CPU 密集计算时没有“别人”替你算

```csharp
long sum = 0;
for (long i = 0; i < 10_000_000_000; i++)
{
    sum += i;
}
```

这段代码执行时：

- CPU 一直在算；
- 线程一直在忙；
- 没有真正的 I/O 等待点；
- 如果当前线程让出去，必须有另一个线程接手继续算；
- 否则这个循环永远不会结束。

所以整体上：

> 总有一个线程被计算占用，线程池并没有多出一个空闲线程。

---

## 7. 常见误区

### 误区一：方法加了 `async` 就会自动让出线程

不是。

```csharp
public async Task<long> CalcAsync()
{
    long sum = 0;
    for (long i = 0; i < 10_000_000_000; i++)
    {
        sum += i;
    }
    return sum;
}
```

这个方法虽然标了 `async`，但里面没有真正的 `await`。

编译器甚至会警告：

> 此异步方法缺少 await 运算符，将同步运行。

调用它时，计算会直接在当前线程上跑完，根本没有归还线程的机会。

### 误区二：`await Task.Run(...)` 就释放了线程

```csharp
var result = await Task.Run(() => HeavyCpuWork());
```

这里调用方确实让出了线程，UI 不会卡，或者请求线程可以回去。

但注意：

- 线程池里有一个线程正在跑 `HeavyCpuWork()`；
- 这个线程一直被占用；
- 并没有“释放计算资源”；
- 只是把计算从当前线程挪到了另一个线程。

如果很多请求都这么干，线程池会被 CPU 计算占满，反而可能导致线程饥饿。

---

## 8. I/O 密集 vs CPU 密集

| 场景 | 线程状态 | `async/await` 能否归还线程 |
|---|---|---|
| I/O 密集 | 等待网络 / 磁盘 / 数据库 | 能，等待期间线程可回池 |
| CPU 密集 | 一直计算 | 不能，总有线程在算 |
| 假 async，无 `await` | 同步执行 | 不能 |
| `await Task.Run(CPU)` | 调用方让出，线程池线程占用 | 整体未释放计算线程 |

一句话对比：

- **I/O 密集**：线程在等网络 / 磁盘，等待期间 CPU 闲着，所以线程可以还回去。
- **CPU 密集**：线程就是用来算的，算的时候 CPU 忙着，线程还不掉。

---

## 9. 实践建议

1. **后端同步改异步时，先看 HTTP 契约是否变化。**
   - 没变：前端通常不用改。
   - 变了：前端按契约调整。

2. **做一次接口回归测试。**
   - 重点看错误响应、超时、取消请求。

3. **CPU 密集场景不要指望 `async/await` 提升吞吐量。**
   - 可考虑：
     - `Task.Run` 移出 UI 线程
     - 并行计算
     - 后台队列
     - 限流
     - 加机器 / 加核

4. **ASP.NET Core 中谨慎随意 `Task.Run`。**
   - 它只是把工作挪到线程池，不是免费释放线程。
   - 大量 CPU 任务可能造成线程池饥饿。

5. **判断标准。**
   - 看接口契约有没有变。
   - 不要只看后端方法有没有加 `async`。

---

## 10. 最终总结

- 后端同步 API 改 `async/await`，前端通常不需要调整。
- 原因：前端依赖 HTTP 契约，不依赖 C# 方法签名。
- 前端仍然要等 HTTP 响应。
- 后端 `async/await` 是在等待 I/O 时把线程让出去。
- 这提升的是服务器并发处理能力，不是让客户端不用等。
- CPU 密集计算没有真正 I/O 等待点，线程必须一直算，所以 `async/await` 不会归还线程。
- `async/await` 擅长 I/O 密集，不擅长 CPU 密集。
- 一句话：**看接口契约，不看方法签名；I/O 能让线程，CPU 不能。**