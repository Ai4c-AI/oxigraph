# 将 Rust 绑定到 .NET 10：Oxigraph 的 FFI 桥接实践

本文详细介绍了如何将 [Oxigraph](https://github.com/oxigraph/oxigraph)（一个用 Rust 编写的
高性能图数据库）通过 FFI 桥接的方式绑定到 .NET 10，覆盖工具链设计、数据协议、内存管理、
回调机制和构建流水线的完整过程。

## 一、架构概览

```
┌──────────────────────────────────────────────────────────┐
│  .NET 应用层                                              │
│  ┌──────────┐  ┌────────────────┐  ┌──────────────────┐  │
│  │  Model   │  │     Store      │  │  CustomFunctions │  │
│  │ (record) │  │  (SPARQL+I/O)  │  │  (C#→Rust 回调)  │  │
│  └────┬─────┘  └───────┬────────┘  └────────┬─────────┘  │
│       │                │                    │             │
│  ┌────┴────────────────┴────────────────────┴─────────┐  │
│  │              Interop 层 (FFI/Json)                  │  │
│  │  NativeMethods.g.cs  │  SafeHandles  │  FFIHelper   │  │
│  │  StreamInterop.cs    │  Term JSON Converters        │  │
│  └──────────────────────┬──────────────────────────────┘  │
└─────────────────────────┼─────────────────────────────────┘
                          │ P/Invoke + JSON 字符串
┌─────────────────────────┼─────────────────────────────────┐
│  oxigraph_dotnet.dll    │  (Rust cdylib)                   │
│  ┌──────────────────────┴──────────────────────────────┐  │
│  │  ffi.rs (~2774 行)                                  │  │
│  │  ├─ Store CRUD/Match/SPARQL                         │  │
│  │  ├─ File/Stream/Callback I/O                        │  │
│  │  ├─ Lazy Query Iterator                             │  │
│  │  ├─ Chunked Bulk Loader                             │  │
│  │  ├─ QueryResults Serialization                      │  │
│  │  ├─ Dataset (In-Memory) + Canonicalization          │  │
│  │  └─ Custom Functions / Aggregate                    │  │
│  ├─ stream_ffi.rs: CallbackReader / CallbackWriter     │  │
│  ├─ error.rs:     {"ok":...} / {"error":...} 协议      │  │
│  └─ model_ffi.rs: JSON↔Quad 序列化                     │  │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  oxigraph + oxrdf (Rust crate) + RocksDB            │  │
│  └─────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

核心设计原则：**零复制复杂数据类型，全量 JSON 序列化跨 FFI 边界**。
Rust 侧所有函数签名遵循统一模式：`(*const c_char) → *mut c_char`，
输入和输出都是 JSON 字符串。

## 二、工具链

### 2.1 Rust 侧：cdylib

`dotnet/src/oxigraph-dotnet/Cargo.toml`：

```toml
[package]
name = "oxigraph-dotnet"
version = "0.5.7"
edition = "2024"

[lib]
crate-type = ["cdylib"]   # ← 编译为动态链接库

[dependencies]
oxigraph = { path = "../../../lib/oxigraph" }
oxrdf = { path = "../../../lib/oxrdf", features = ["serde"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"

[profile.release]
opt-level = "z"            # 体积优化（减小 .dll 尺寸）
lto = true                 # 链接时优化
```

产物：
- Windows: `oxigraph_dotnet.dll`
- Linux: `liboxigraph_dotnet.so`
- macOS: `liboxigraph_dotnet.dylib`

### 2.2 .NET 侧：`LibraryImport` 生成源

.NET 10 使用源生成器 `LibraryImport`（取代旧的 `DllImport`）进行 P/Invoke：

```csharp
// NativeMethods.g.cs — 自动生成的 FFI 绑定
internal static partial class OxigraphNative
{
    private const string LibName = "oxigraph_dotnet";

    [LibraryImport(LibName, EntryPoint = "oxigraph_store_open",
                   StringMarshalling = StringMarshalling.Utf8)]
    internal static partial IntPtr store_open(string path);

    [LibraryImport(LibName, EntryPoint = "oxigraph_store_query",
                   StringMarshalling = StringMarshalling.Utf8)]
    internal static partial IntPtr store_query(IntPtr handle, string queryJson);

    [LibraryImport(LibName, EntryPoint = "oxigraph_register_custom_function",
                   StringMarshalling = StringMarshalling.Utf8)]
    internal static partial IntPtr register_custom_function(
        string name, IntPtr callback);
    // ... 共 60+ 个 FFI 函数
}
```

关键差异：`LibraryImport` 是源生成器模式，编译时生成封送代码，
避免了 `DllImport` 的运行时 IL 生成和 AOT 兼容问题。

### 2.3 构建流水线

`build_package.py` 编排本地构建过程：

```python
# 步骤 1: 编译 Rust cdylib
cargo build --release -p oxigraph-dotnet --features rocksdb

# 步骤 2: 编译 C# 项目
dotnet build dotnet/ -c Release

# 步骤 3: 将 .dll 拷贝到测试输出目录
# 步骤 4: 运行 xUnit 测试
dotnet test dotnet/tests/Oxigraph.Tests -c Release
```

在 CI 环境中，构建流水线由 GitHub Actions 自动化编排（详见[十二、CI/CD 流水线](#十二cicd-流水线)）：
- **CI 测试**（`.github/workflows/tests.yml`）：每个 PR 在 Linux/macOS/Windows 三平台上运行 `dotnet test`
- **制品构建**（`.github/workflows/artifacts.yml`）：push main 和 release 时构建多平台原生库并打包为 NuGet
- **版本同步**：CI 从 `dotnet/src/oxigraph-dotnet/Cargo.toml` 提取版本号，传递给 `dotnet pack -p:Version=$VERSION`

## 三、数据协议：JSON over FFI

### 3.1 统一响应格式

所有 Rust FFI 函数返回 `*mut c_char`（JSON 字符串），包含两种可能的格式：

```json
{"ok": <result>}
// 或
{"error": {"kind": "store"|"parse"|"invalid_argument", "message": "..."}}
```

C# 侧 `FFIHelper` 统一处理：

```csharp
// 带返回值的调用
internal static T Call<T>(Func<IntPtr> ffiCall) where T : class
{
    IntPtr jsonPtr = ffiCall();
    string json = ReadAndFree(jsonPtr);  // Marshal.PtrToStringUTF8 + free_string
    ThrowIfError(json);                  // 检查 {"error":...}
    using var doc = JsonDocument.Parse(json);
    return JsonSerializer.Deserialize<T>(
        doc.RootElement.GetProperty("ok").GetRawText())!;
}

// 无返回值的调用
internal static void CallVoid(Func<IntPtr> ffiCall)
{
    IntPtr jsonPtr = ffiCall();
    string json = ReadAndFree(jsonPtr);
    ThrowIfError(json);
}
```

### 3.2 错误映射

Rust 错误类型自动映射为 C# 异常：

```csharp
private static Exception MapError(string kind, string message)
{
    return kind switch
    {
        "store"             => new StoreException(message),
        "parse"             => new ParseException(message),
        "invalid_argument"  => new ArgumentException(message),
        _                   => new OxigraphException($"[{kind}] {message}"),
    };
}
```

### 3.3 不透明句柄（Opaque Handles）

Rust 对象不直接暴露给 C#，而是通过指针传递：

```rust
// Rust 侧：Store 包装为 Box<UnsafeCell<Store>>，返回指针的 u64 表示
fn store_to_handle(store: Store) -> *mut c_char {
    let boxed = Box::new(UnsafeCell::new(store));
    let ptr = Box::into_raw(boxed);
    let handle_value = ptr as u64;
    // 返回 {"ok":{"handle":12345678}}
    ok_json(&handle_value)
}
```

```csharp
// C# 侧：SafeHandle 封装原始指针，确保析构
internal sealed class StoreSafeHandle : SafeHandleZeroOrMinusOneIsInvalid
{
    protected override bool ReleaseHandle()
    {
        OxigraphNative.store_destroy(handle);  // Rust 侧 drop(Box::from_raw(ptr))
        return true;
    }
}
```

所有句柄类型：

| Rust 类型 | C# SafeHandle | 用途 |
|-----------|---------------|------|
| `StoreHandle` = `*mut UnsafeCell<Store>` | `StoreSafeHandle` | Store 读写 |
| `DatasetHandle` = `*mut UnsafeCell<Dataset>` | `DatasetSafeHandle` | Dataset |
| `QuadIterHandle` = `*mut UnsafeCell<ReaderQuadParser>` | `QuadIterSafeHandle` | 懒解析迭代器 |
| `QueryResultsHandle` = `*mut QueryResultsWrapper` | `QueryResultsSafeHandle` | 流式查询结果 |

## 四、Stream 回调机制

对于 .NET Stream 的读写（`LoadFromStream` / `DumpToStream`），无法简单地把 Stream
传过去，因为 Rust 不认识 .NET 的 `Stream` 类型。解决方案：**C 风格回调**。

### 4.1 Rust 侧：实现 `Read` / `Write` trait

```rust
pub type ReadFn = unsafe extern "C" fn(
    context: *mut c_void, buf: *mut u8, buf_size: i32
) -> i32;

pub struct CallbackReader {
    context: *mut c_void,
    callback: ReadFn,
}

impl Read for CallbackReader {
    fn read(&mut self, buf: &mut [u8]) -> io::Result<usize> {
        let result = unsafe {
            (self.callback)(self.context, buf.as_mut_ptr(), buf.len() as i32)
        };
        match result {
            n if n > 0 => Ok(n as usize),
            0 => Ok(0),      // EOF
            -1 => Err(...),  // Error
        }
    }
}
```

`CallbackWriter` 对 `Write` trait 实现类似。

### 4.2 C# 侧：GCHandle 固定的委托

```csharp
[UnmanagedFunctionPointer(CallingConvention.Cdecl)]
internal delegate int ReadCallback(IntPtr context, IntPtr buffer, int bufferSize);

internal sealed class ReadContext : IDisposable
{
    private readonly Stream _stream;
    private readonly byte[] _buffer;
    private readonly GCHandle _gcHandle;  // ← 防止 GC 回收委托

    public readonly IntPtr ContextPtr;
    public readonly ReadCallback Callback;

    public ReadContext(Stream stream)
    {
        _stream = stream;
        Callback = ReadImpl;
        _gcHandle = GCHandle.Alloc(Callback);  // 固定委托
        ContextPtr = (IntPtr)_gcHandle;        // 作为 context 传递
    }

    private int ReadImpl(IntPtr context, IntPtr buffer, int bufferSize)
    {
        try
        {
            int read = _stream.Read(_buffer, 0, Math.Min(bufferSize, _buffer.Length));
            Marshal.Copy(_buffer, 0, buffer, read);
            return read;
        }
        catch { return -1; }
    }

    public void Dispose() { _gcHandle.Free(); }
}
```

调用链：`C# Stream.Read()` → 拷贝到 Rust buffer → Rust `Read` trait → `RdfParser.for_reader()`

## 五、自定义函数回调桥

SPARQL 自定义函数需要 Rust 在执行查询时回调到 C#。这是最复杂的 FFI 场景：
**Rust 调用 C#，C# 再把结果返回给 Rust。**

### 5.1 简单自定义函数

```rust
// Rust 侧：函数指针类型
type CustomFnCallback = unsafe extern "C" fn(args_json: *const c_char) -> *mut c_char;

// 全局注册表
static CUSTOM_FUNCTIONS: LazyLock<Mutex<HashMap<String, CustomFnCallback>>> = ...;

#[unsafe(no_mangle)]
pub extern "C" fn oxigraph_register_custom_function(
    name: *const c_char,
    callback: CustomFnCallback,  // C# 函数的指针
) -> *mut c_char {
    CUSTOM_FUNCTIONS.lock().unwrap().insert(name_str, callback);
    ok_json(&"registered")
}
```

Rust 收到 `callback` 后，在 SPARQL 评估时调用它：
```
Rust SPARQL evaluator → 遇到 my:func(?x)
→ 查找 CUSTOM_FUNCTIONS["http://example.com/myFunc"]
→ 调用 callback(args_json)  // ← 跨 FFI 边界回调到 C#
→ 收到返回的 result_json
→ 继续 SPARQL 评估
```

C# 侧：

```csharp
// BridgeDelegate：接收 JSON 参数数组，返回 JSON Term
private delegate IntPtr BridgeDelegate(IntPtr argsJsonPtr);

private static IntPtr BridgeImpl(IntPtr argsJsonPtr)
{
    var json = Marshal.PtrToStringUTF8(argsJsonPtr);
    // json: ["http://example.com/myFunc", {"type":"literal","value":"hello"}]

    using var doc = JsonDocument.Parse(json);
    var name = doc.RootElement[0].GetString();
    // 解析剩余元素为 ITerm[]
    var terms = ...;
    // 调用 .NET 函数
    var result = _functions[name](terms);
    // 序列化结果返回给 Rust
    return Marshal.StringToHGlobalAnsi(JsonSerializer.Serialize(result));
}

// 固定 BridgeDelegate，防止 GC 回收
private static readonly GCHandle _gcHandle = GCHandle.Alloc(_bridge);
private static readonly IntPtr _bridgePtr =
    Marshal.GetFunctionPointerForDelegate(_bridge);
```

### 5.2 自定义聚合函数

聚合函数需要更复杂的 4 回调协议：

```
Rust 调用:
  new_fn()     → 返回 ctx handle（GCHandle 包装的 C# 对象）
  acc_fn(ctx, term_json)  → 每行数据调用一次
  finish_fn(ctx) → 返回聚合结果
  free_fn(ctx) → 释放 C# 对象
```

```rust
// Rust 适配器：实现 oxigraph::sparql::AggregateFunctionAccumulator
struct CallbackAggregateAccumulator {
    ctx: *mut c_void,
    acc_fn: AggregateAccCallback,
    finish_fn: AggregateFinishCallback,
    free_fn: AggregateFreeCallback,
}

impl AggregateFunctionAccumulator for CallbackAggregateAccumulator {
    fn accumulate(&mut self, element: Term) {
        let json = serde_json::to_string(&element).unwrap();
        let c_str = CString::new(json).unwrap();
        unsafe { (self.acc_fn)(self.ctx, c_str.as_ptr()) };
    }

    fn finish(&mut self) -> Option<Term> {
        let ptr = unsafe { (self.finish_fn)(self.ctx) };
        // 解析 C# 返回的 JSON Term 或 null
        if ptr.is_null() { return None; }
        let json = unsafe { c_str_to_str(ptr) };
        serde_json::from_str(json).unwrap_or(None)
    }
}
```

## 六、RDF 数据类型的 JSON 序列化

Rust 的 serde 格式直接映射到 C# 的 `System.Text.Json`。两端使用相同的
tagged-enum JSON 模式：

```json
// NamedNode
{"type": "uri",     "value": "http://example.com/s"}

// BlankNode
{"type": "bnode",   "value": "b1_abc123"}

// Literal (plain)
{"type": "literal", "value": "hello"}

// Literal (language-tagged)
{"type": "literal", "value": "bonjour", "language": "fr"}

// Literal (typed)
{"type": "literal", "value": "42",
 "datatype": {"type":"uri","value":"http://www.w3.org/2001/XMLSchema#integer"}}

// Triple (RDF-star)
{"type": "triple",
 "subject":   {"type":"uri","value":"http://s"},
 "predicate": {"type":"uri","value":"http://p"},
 "object":    {"type":"uri","value":"http://o"}}

// DefaultGraph
{"type": "default"}
```

C# 侧自定义 Converter：

```csharp
public class TermConverter : JsonConverter<ITerm>
{
    public override ITerm? Read(ref Utf8JsonReader reader, ...)
    {
        using var doc = JsonDocument.ParseValue(ref reader);
        var kind = doc.RootElement.GetProperty("type").GetString();
        return kind switch
        {
            "uri"     => new NamedNode(doc.RootElement.GetProperty("value").GetString()!),
            "bnode"   => new BlankNode(doc.RootElement.GetProperty("value").GetString()!),
            "literal" => /* 解析 value/language/datatype/direction */,
            "triple"  => JsonSerializer.Deserialize<Triple>(root.GetRawText())!,
            _ => throw new JsonException($"Unknown term type: {kind}")
        };
    }
}
```

通过这种 JSON 约定，C# 的 `record` 类型与 Rust 的 serde 序列化保持了精确对应，
无需额外的中间表示层。

## 七、流式查询结果：懒迭代器

SPARQL 查询可能返回百万级结果。为避免将全部结果物化到一个 JSON 数组中，
实现了流式迭代器：

```rust
// oxigraph_store_query_iter：返回不透明句柄，不包含结果数据
pub extern "C" fn oxigraph_store_query_iter(
    handle: StoreHandle,
    query_json: *const c_char,
) -> *mut c_char {
    // ... 执行查询，获取 QueryResults
    // 包装为 QueryResultsWrapper 枚举
    // 返回 {"ok":{"handle":...}}  — 不包含任何数据行
}

// 每次取一行
pub extern "C" fn oxigraph_query_iter_next_solution(
    handle: QueryResultsHandle
) -> *mut c_char {
    // 调用 iter.next()，返回单行 JSON 或 null
}
```

C# 侧 `LazySolutionList`：按需向 Rust 请求下一行，支持 `IReadOnlyList<T>` 接口：

```csharp
private void MaterializeUpTo(int required)
{
    while (_materialized.Count < required)
    {
        var ptr = OxigraphNative.query_iter_next_solution(_handle);
        var json = Marshal.PtrToStringUTF8(ptr) ?? "null";
        OxigraphNative.free_string(ptr);

        if (okVal is null) { _count = _materialized.Count; return; }
        _materialized.Add(new QuerySolution(okVal));
    }
}
```

## 八、Chunked Bulk Loader

大数据加载使用 RocksDB 的批量加载路径，以 10,000 个 quad 为一批：

```csharp
public void BulkExtend(IEnumerable<Quad> quads)
{
    const int chunkSize = 10_000;

    // 1. 开始批量加载器
    var handle = store_bulk_extend_begin();

    // 2. 分批喂入数据
    foreach (var quad in quads)
    {
        chunk.Add(quad);
        if (chunk.Count >= chunkSize) {
            store_bulk_extend_add_chunk(handle, JsonSerializer.Serialize(chunk));
            chunk.Clear();
        }
    }

    // 3. 提交
    store_bulk_extend_add_chunk(handle, JsonSerializer.Serialize(chunk));
    store_bulk_extend_commit(handle);
}

// 异常时自动取消：
// bulkHandle 的 ReleaseHandle 调用 store_bulk_extend_cancel
```

```rust
// Rust 侧：RocksDB BulkLoader
pub extern "C" fn oxigraph_store_bulk_extend_commit(handle) {
    let loader: Box<BulkLoader> = Box::from_raw(handle as *mut _);
    loader.commit()?;
}
```

## 九、内存管理

### 9.1 SafeHandle — 确定性析构

所有 Rust 句柄都包装在 .NET `SafeHandle` 中。即使进程异常终止
（`AppDomain` 卸载、`ThreadAbortException`），CLR 也会调用 `ReleaseHandle`。

```csharp
internal sealed class StoreSafeHandle : SafeHandleZeroOrMinusOneIsInvalid
{
    protected override bool ReleaseHandle()
    {
        OxigraphNative.store_destroy(handle);  // Rust: drop(Box::from_raw(ptr))
        return true;
    }
}
```

### 9.2 GCHandle — 防止委托被 GC 回收

当一个 C# 委托被传递给 Rust 侧持有，必须用 `GCHandle` 固定它：

```csharp
private static readonly GCHandle _gcHandle = GCHandle.Alloc(_bridge);
private static readonly IntPtr _bridgePtr =
    Marshal.GetFunctionPointerForDelegate(_bridge);
```

`GCHandle.Alloc` 确保委托在托管堆上不会被移动或回收，Rust 侧函数指针始终有效。

### 9.3 字符串生命周期

```rust
// Rust 返回的字符串必须由调用方释放
pub extern "C" fn oxigraph_free_string(ptr: *mut c_char) {
    if !ptr.is_null() { unsafe { drop(CString::from_raw(ptr)); } }
}
```

```csharp
// C# 侧：每次 FFI 调用后立即释放 Rust 字符串
private static string ReadAndFree(IntPtr ptr)
{
    string json = Marshal.PtrToStringUTF8(ptr);
    OxigraphNative.free_string(ptr);
    return json;
}
```

## 十、线程安全

Rust 侧使用 `UnsafeCell<Store>` 包装 Store，`UnsafeCell` 不提供任何同步语义：

```rust
pub type StoreHandle = *mut UnsafeCell<Store>;
```

C# 侧明确标注：

```csharp
/// <summary>
/// An RDF store backed by RocksDB (on disk) or in-memory.
/// Thread safety: not guaranteed. Callers must synchronize concurrent access.
/// </summary>
```

异步 API 使用 `Task.Run` 将阻塞操作移到线程池，但不改变线程安全语义。
所有对同一 Store 的并发访问（包括 async 方法）必须由调用方串行化。

## 十一、关键指标

| 维度 | 数值 |
|------|------|
| Rust FFI 函数 | 60+ 个 `#[unsafe(no_mangle)]` |
| C# `LibraryImport` | 60+ 个 `partial` 方法 |
| 协议格式 | JSON（`{"ok":...}` / `{"error":...}`） |
| 句柄类型 | 5 种 SafeHandle |
| 回调方向 | Rust→C#（CustomFunctions、Stream 回调） |
| 流式查询 | 懒迭代器，逐行 FFI 调用 |
| 构建工具 | `cargo` + `dotnet` + `build_package.py` |
| 测试 | 280 xUnit 测试，全部通过 |
| CI/CD | GitHub Actions 多平台矩阵 + NuGet 打包 + GitHub Release |

## 十二、CI/CD 流水线

### 12.1 工作流概览

Oxigraph .NET 绑定使用两个 GitHub Actions 工作流：

| 工作流 | 文件 | 触发条件 | 职责 |
|--------|------|----------|------|
| **Change tests** | `tests.yml` | PR → main/next, push → main/next | 代码质量检查 + 测试 |
| **Artifacts** | `artifacts.yml` | push → main/next, release published | 制品构建 + 发布 |

### 12.2 CI 测试（tests.yml）

`dotnet-tests` 作业在 3 个平台的矩阵上运行（[tests.yml:567-605]）：

```
ubuntu-latest / windows-latest / macos-latest
```

每个平台执行：
1. `cargo build --release -p oxigraph-dotnet`（Linux/macOS 带 `--features oxigraph/rocksdb`）
2. 将原生库复制到测试输出目录
3. `dotnet test dotnet/tests/Oxigraph.Tests/ --verbosity normal`

### 12.3 制品构建（artifacts.yml）

`artifacts.yml` 包含 4 个 dotnet 相关作业：

```
dotnet_native_linux   (ubuntu-latest x64 + ubuntu-24.04-arm arm64)
dotnet_native_macos   (macos-latest x64 + arm64 交叉编译)
dotnet_native_windows (windows-latest x64)
       │
       └──→ dotnet_pack (ubuntu-latest, needs all native jobs)
```

**原生库构建作业**（`dotnet_native_*`）：
1. 安装 Rust + .NET SDK
2. `cargo build --release -p oxigraph-dotnet`（指定 target/features）
3. 将产物拷贝到 `runtimes/<rid>/native/` 目录
4. `upload-artifact` 上传原生库

**打包作业**（`dotnet_pack`）：
1. `download-artifact` 下载所有平台的原生库到 `dotnet/src/Oxigraph/runtimes/`
2. 从 `dotnet/src/oxigraph-dotnet/Cargo.toml` 提取版本号
3. `dotnet build` + `dotnet pack` 生成多架构 NuGet 包
4. `upload-artifact` 上传 `.nupkg`
5. 在 release 事件中：`gh release upload` 发布到 GitHub Release

### 12.4 NuGet 包结构

项目生成**两个独立的 NuGet 包**：

**Oxigraph**（核心库）—— 包含托管程序集 + 所有平台的原生库，利用 .NET 的 `runtimes/` 目录约定自动解析：

```
Oxigraph.0.5.7.nupkg
├── lib/net10.0/
│   └── Oxigraph.dll                        # 托管程序集（跨平台 IL）
├── runtimes/
│   ├── win-x64/native/oxigraph_dotnet.dll
│   ├── linux-x64/native/liboxigraph_dotnet.so
│   ├── linux-arm64/native/liboxigraph_dotnet.so
│   ├── osx-x64/native/liboxigraph_dotnet.dylib
│   └── osx-arm64/native/liboxigraph_dotnet.dylib
└── Oxigraph.nuspec
```

**Oxigraph.Extensions.DotNetRDF**（扩展库）—— 纯托管代码，依赖 `Oxigraph` 和 `dotNetRdf`：

```
Oxigraph.Extensions.DotNetRDF.0.5.7.nupkg
├── lib/net10.0/
│   └── Oxigraph.Extensions.DotNetRDF.dll    # dotNetRDF 互操作适配器
└── Oxigraph.Extensions.DotNetRDF.nuspec
```

```xml
<!-- Oxigraph.Extensions.DotNetRDF 的依赖声明 -->
<PackageReference Include="Oxigraph" Version="0.5.7" />
<PackageReference Include="dotNetRdf" Version="3.3.1" />
```

用户按需安装：
- `dotnet add package Oxigraph` — 核心 RDF 图数据库
- `dotnet add package Oxigraph.Extensions.DotNetRDF` — dotNetRDF 互操作（可选）

### 12.5 发布流程

**方式一：标准 Release 发布（所有资产）**

```
1. 创建 GitHub Release (tag v0.5.7)
         │
2. artifacts.yml 触发 release published 事件
         │
3. 所有资产并行构建（Python wheels、npm、Docker、dotnet native）
         │
4. 各自发布：PyPI、npmjs.org、ghcr.io、crates.io、GitHub Release
         │
5. dotnet_pack 聚合 → dotnet pack → 2 个 NuGet 包
         │
6. gh release upload → GitHub Release 资产
```

**方式二：仅发布 NuGet（nuget-only 开关）**

在 GitHub Actions 页面手动触发 `Artifacts` workflow，勾选 **"仅发布 NuGet 包"**：

```
1. Actions → Artifacts → Run workflow ▾
2. 勾选 ☑ "仅发布 NuGet 包（跳过 Python、npm、Docker、crates 等其他资产）"
3. 点击 Run workflow
         │
4. 所有非 dotnet job 被跳过（if: inputs.nuget-only != true）
5. 仅执行 dotnet_native_* → dotnet_pack → upload-artifact
         │
6. 手动下载 NuGet 资产（workflow_dispatch 不触发 gh release upload）
```

**条件逻辑：**

```yaml
# 触发声明
on:
  workflow_dispatch:
    inputs:
      nuget-only:
        description: '仅发布 NuGet 包'
        type: boolean
        default: false

# 非 dotnet job 的条件守卫
python_sdist:      if: inputs.nuget-only != true
wheel_linux:       if: inputs.nuget-only != true
publish_pypi:      if: github.event_name == 'release' && inputs.nuget-only != true
npm_tarball:       if: (github.event_name == 'release' || github.ref == 'refs/heads/main') && inputs.nuget-only != true

# dotnet job 始终执行
dotnet_pack:       if: github.event_name == 'release' || github.ref == 'refs/heads/main' || github.event_name == 'workflow_dispatch'
```

同步发布的其他资产（已有基础设施）：

| 资产 | 目标 | 发布方式 |
|------|------|----------|
| Python wheels | PyPI | `pypi-publish` OIDC |
| npm tarball | npmjs.org | `npm publish` |
| Docker image | ghcr.io + Docker Hub | `docker push` |
| Rust crate | crates.io | `cargo publish` |
| Full archive | GitHub Release | `gh release upload` |
| **NuGet 包** | **GitHub Release** | **`gh release upload`** |

### 12.6 与 Python/npm/Docker 发布的对比

| 特征 | Python (PyPI) | npm | Docker | .NET (NuGet) |
|------|---------------|-----|--------|---------------|
| 包格式 | `.whl` + `.tar.gz` | `.tgz` | OCI image | `.nupkg` |
| 平台策略 | 每平台独立 wheel | WASM 单包 | 多架构 manifest | 单一多架构包 |
| 原生代码 | 嵌入 wheel | 嵌入 WASM | 容器内 | `runtimes/<rid>/native/` |
| 发布目标 | PyPI | npmjs.org | ghcr.io + Docker Hub | GitHub Release |
| 发布方式 | OIDC trusted publishing | `npm publish` | `docker push` | `gh release upload` |
| 版本来源 | `Cargo.toml` | `package.json` | git tag | `Cargo.toml` → `dotnet pack -p:Version=...` |

## 十三、独立仓库迁移方案

当 dotnet 绑定需要拆分为独立仓库 `oxigraph-dotnet` 时，核心挑战是将 Rust 依赖从 monorepo 路径依赖切换为 crates.io 远程依赖。

### 13.1 新仓库结构

```
oxigraph-dotnet/
├── .github/
│   ├── workflows/
│   │   ├── tests.yml              # dotnet 测试（简化为仅 cargo build + dotnet test）
│   │   └── artifacts.yml          # NuGet 打包 + GitHub Release
│   └── actions/setup-rust/        # 从主仓库复制
├── dotnet/
│   ├── Directory.Build.props      # NuGet 元数据 + 版本号
│   ├── src/
│   │   ├── oxigraph-dotnet/       # Rust cdylib
│   │   │   ├── Cargo.toml         # ← 关键改动：路径依赖 → crates.io
│   │   │   └── src/
│   │   ├── Oxigraph/              # C# 托管库
│   │   └── Oxigraph.Extensions.DotNetRDF/
│   ├── tests/Oxigraph.Tests/
│   └── docs/binding-architecture.md
├── build_package.py
└── README.md
```

### 13.2 关键改动：Cargo.toml 依赖切换

**当前（monorepo 路径依赖）**：

```toml
# dotnet/src/oxigraph-dotnet/Cargo.toml
[package]
name = "oxigraph-dotnet"
version = "0.5.7"       # 从 workspace 继承，迁移后手动维护
edition = "2024"

[lib]
crate-type = ["cdylib"]

[dependencies]
oxigraph = { path = "../../../lib/oxigraph" }
oxrdf = { path = "../../../lib/oxrdf", features = ["serde"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"

[profile.release]
opt-level = "z"
lto = true
```

**迁移后（crates.io 远程依赖）**：

```toml
[package]
name = "oxigraph-dotnet"
version = "0.5.7"
edition = "2024"

[lib]
crate-type = ["cdylib"]

[dependencies]
oxigraph = "0.5.7"      # crates.io，默认 features 已含 rocksdb
oxrdf = { version = "0.3.3", features = ["serde"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"

[profile.release]
opt-level = "z"
lto = true
```

**为什么用 crates.io 而非 git 依赖**：

| 方式 | 优点 | 缺点 |
|------|------|------|
| `oxigraph = "0.5.7"`（crates.io） | 预编译缓存、稳定版本、CI 更快 | 无法跟踪上游最新 commit |
| `oxigraph = { git = "..." }` | 可跟踪 `main` 分支 | 每次 CI 都要完整编译上游 |

只在开发中临时跟踪上游 `main` 时切换为 git 依赖：

```toml
# 临时开发用
oxigraph = { git = "https://github.com/oxigraph/oxigraph", branch = "main" }
```

### 13.3 CI/CD 简化

迁移后不再需要 submodule 和手动 features 配置：

```yaml
# 迁移前（monorepo）
- uses: actions/checkout@v6
  with:
    submodules: true                  # ← 可以去掉
- run: cargo build --release -p oxigraph-dotnet --features oxigraph/rocksdb  # ← 可以去掉

# 迁移后（独立仓库）
- uses: actions/checkout@v6           # 无 submodule
- run: cargo build --release -p oxigraph-dotnet  # oxigraph 从 crates.io 拉取，默认含 rocksdb
```

### 13.4 版本同步策略

版本号出现在两处，需手工同步：

| 文件 | 字段 | 当前值 |
|------|------|--------|
| `dotnet/src/oxigraph-dotnet/Cargo.toml` | `version` | `0.5.7` |
| `dotnet/Directory.Build.props` | `<Version>` | `0.5.7` |

CI 中通过脚本统一提取 Cargo.toml 版本传给 `dotnet pack`：

```bash
# 从 Cargo.toml 提取版本号
grep -m1 '^version' dotnet/src/oxigraph-dotnet/Cargo.toml | sed 's/.*"\(.*\)"/\1/'
# → 0.5.7
# 写入 $GITHUB_OUTPUT → dotnet pack -p:Version=0.5.7
```

**版本对齐建议**：主版本号保持与上游 `oxigraph` 一致（均为 `0.5.7`），补丁号允许独立递增。当上游 `oxigraph` 发新版本时，同步更新依赖版本和 dotnet 版本。

### 13.5 测试套件处理

280 个 xUnit 测试全部在 `dotnet/tests/Oxigraph.Tests/` 内，不依赖主仓库的 `testsuite/` 目录，可直接迁移。

如需引用上游测试数据（如 `testsuite/oxigraph-tests/`），两种方案：

```bash
# 方案 A：Git 子模块
git submodule add https://github.com/oxigraph/oxigraph upstream
# 测试数据路径 → upstream/testsuite/

# 方案 B：CI 中动态 clone
- uses: actions/checkout@v6
  with:
    repository: oxigraph/oxigraph
    path: upstream
- run: dotnet test --environment:TEST_DATA=upstream/testsuite/
```

### 13.6 迁移步骤清单

```
步骤 1 ─ 创建 oxigraph-dotnet 仓库
步骤 2 ─ 从主仓库 dotnet/ 目录复制所有文件
步骤 3 ─ 修改 dotnet/src/oxigraph-dotnet/Cargo.toml
         ├─ 去除 workspace 继承字段（version, authors, license, edition, rust-version）
         └─ 路径依赖切换为 crates.io 依赖
步骤 4 ─ 修改 .github/workflows/
         ├─ 去掉 submodules: true
         ├─ 去掉 --features oxigraph/rocksdb
         └─ 删除 Python/npm/Docker/crates 等非 dotnet job
步骤 5 ─ 本地验证
         ├─ cargo build --release -p oxigraph-dotnet
         ├─ python build_package.py --release
         └─ dotnet test dotnet/tests/Oxigraph.Tests/
步骤 6 ─ 在主仓库 dotnet/ 目录添加 README.md 指向新仓库
         └─ "Oxigraph .NET bindings have moved to https://github.com/oxigraph/oxigraph-dotnet"
步骤 7 ─ 设置仓库 Secrets（GH_TOKEN 用于 Release 上传）
步骤 8 ─ 推送 → 验证 CI 通过 → 创建初始 Release
```

## 十四、总结

将 Oxigraph 从 Rust 绑定到 .NET 的核心挑战在于设计一个**简单可靠的 FFI 协议**。
本文采用的方案——JSON over raw C strings + 不透明指针句柄 + GCHandle 固定的回调——
在保持代码简洁的同时，提供了完整的跨语言互操作能力。

这一模式适用于任何需要将 Rust 原生库暴露给 .NET 的场景：
1. 用 `cdylib` 编译 Rust 到动态链接库
2. 用 `LibraryImport` 源生成器做 P/Invoke
3. 用 JSON 作为数据交换格式（两端的序列化库天然兼容）
4. 用 `SafeHandle` + `GCHandle` 管理跨语言生命周期
5. 用 C 风格回调实现双向调用
6. 用 `Task.Run` 包装提供异步 API