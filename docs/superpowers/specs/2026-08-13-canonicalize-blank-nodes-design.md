# Dotnet Canonicalize Blank Nodes — 功能对等设计

**日期**: 2026-08-13
**状态**: 待用户审核
**起始 commit**: `820c8ddca8b183d41fa1f4f2a99cf604ef1ba173`
**目标 commit**: `cb228e5f36fcfd3d6908621ba9be08d89d925ae1`
**对应 Python commit**: `061df0483503e39aee798a09eb8766804c9b4e8f`

## 目标

在 Dotnet 侧实现 Python `canonicalize_blank_nodes` 功能对等，同时保持 Rust FFI 层同步。

## 范围

### 需修改的文件

| 层级 | 文件 | 变更内容 |
|------|------|---------|
| Rust FFI | `dotnet/src/oxigraph-dotnet/src/ffi.rs` | 新增 `oxigraph_dataset_canonicalize_blank_nodes` |
| C# 枚举 | `dotnet/src/Oxigraph/RdfFormat.cs` | `CanonicalizationAlgorithm` 新增 `UnstableHashedIds` |
| C# 方法 | `dotnet/src/Oxigraph/Dataset.cs` | 新增 `CanonicalizeBlankNodes()` |
| C# 测试 | `dotnet/tests/Oxigraph.Tests/ModelTests.cs` | 新增单元测试 |
| 文档 | `dotnet/docs/model.md` | 补充 `CanonicalizeBlankNodes` 和 `UnstableHashedIds` 说明 |

### 参考

- Python binding（`python/src/dataset.rs`）— Dotnet binding 实现参考
- Python commit: `061df0483503e39aee798a09eb8766804c9b4e8f`

### 不在范围内

- Python binding 本身（已在 commit 061df048 实现）

## 设计

### 1. Rust FFI 层

**新函数**: `oxigraph_dataset_canonicalize_blank_nodes`

```rust
pub extern "C" fn oxigraph_dataset_canonicalize_blank_nodes(
    handle: DatasetHandle,
    algorithm: *const c_char,
) -> *mut c_char {
    // 1. 解析 algorithm 字符串为 CanonicalizationAlgorithm
    // 2. 调用 dataset.canonicalize_blank_nodes(algorithm)
    // 3. 将 HashMap<BlankNode, BlankNode> 序列化为 JSON:
    //    {"ok": {"<original_id>": "<canonical_id>", ...}}
    // 4. 返回自有 C 字符串（由调用方释放）
}
```

**算法字符串映射**:

| CanonicalizationAlgorithm | FFI 字符串 |
|--------------------------|------------|
| `Unstable` | `"unstable"` |
| `UnstableHashedIds` | `"unstable_hashed_ids"` |
| `Rdfc10 { Sha256 }` | `"rdfc10_sha256"` |
| `Rdfc10 { Sha384 }` | `"rdfc10_sha384"` |

**JSON 响应格式**:

```json
{
  "ok": {
    "b0": "c14n0",
    "b1": "c14n1"
  }
}
```

错误时:
```json
{
  "error": "error message"
}
```

### 2. C# CanonicalizationAlgorithm 枚举

**文件**: `dotnet/src/Oxigraph/RdfFormat.cs`

```csharp
public enum CanonicalizationAlgorithm
{
    /// <summary>Oxigraph 首选算法（不稳定）。</summary>
    Unstable,
    /// <summary>RDFC-1.0 with SHA-256。</summary>
    Rdfc10Sha256,
    /// <summary>RDFC-1.0 with SHA-384。</summary>
    Rdfc10Sha384,
    /// <summary>Oxigraph 首选算法，但输出的 ID 基于哈希值。</summary>
    /// <remarks>
    /// 这使得空白节点 ID 可用于 diff 场景，添加或删除 triple 时影响的空白节点 ID 更少。
    /// 警告：可能在 Oxigraph 版本间发生变化，不保证稳定性。
    /// </remarks>
    UnstableHashedIds,
}
```

### 3. C# Dataset 方法

**文件**: `dotnet/src/Oxigraph/Dataset.cs`

```csharp
/// <summary>
/// 返回当前数据集中空白节点到规范化空白节点的映射，用于创建规范化数据集。
///
/// 详见 <see cref="Canonicalize"/>。
///
/// :param algorithm: 要使用的规范化算法。
/// :rtype: IReadOnlyDictionary{BlankNode, BlankNode}
///
/// </summary>
public IReadOnlyDictionary<BlankNode, BlankNode> CanonicalizeBlankNodes(
    CanonicalizationAlgorithm algorithm = CanonicalizationAlgorithm.Unstable)
{
    var algoStr = algorithm switch
    {
        CanonicalizationAlgorithm.Unstable => "unstable",
        CanonicalizationAlgorithm.UnstableHashedIds => "unstable_hashed_ids",
        CanonicalizationAlgorithm.Rdfc10Sha256 => "rdfc10_sha256",
        CanonicalizationAlgorithm.Rdfc10Sha384 => "rdfc10_sha384",
        _ => "unstable",
    };
    var jsonPtr = OxigraphNative.dataset_canonicalize_blank_nodes(_handle.DangerousGetHandle(), algoStr);
    var response = ReadAndFree(jsonPtr);
    FFIHelper.ThrowIfError(response);

    using var doc = JsonDocument.Parse(response);
    var ok = doc.RootElement.GetProperty("ok");
    var dict = new Dictionary<BlankNode, BlankNode>();
    foreach (var prop in ok.EnumerateObject())
    {
        dict[new BlankNode(prop.Name)] = new BlankNode(prop.Value.GetString()!);
    }
    return dict;
}
```

**决策：无异步变体** — 与用户明确选择一致。

### 4. NativeMethods 入口点

**文件**: `dotnet/src/oxigraph-dotnet/src/ffi.rs`

```rust
[no_mangle]
pub extern "C" fn oxigraph_dataset_canonicalize_blank_nodes(
    handle: DatasetHandle,
    algorithm: *const c_char,
) -> *mut c_char {
    // 实现逻辑
}
```

注意：`NativeMethods.g.cs` 是由 P/Invoke Source Generator 从 `ffi.rs` 自动生成的，无需手动修改 `.g.cs` 文件。

### 5. 错误处理

- FFI 层失败时返回错误 JSON
- `FFIHelper.ThrowIfError(response)` 在出错时抛出 `OxigraphException`
- 不使用 `KeyNotFoundException` — 此方法返回映射，不存在"未找到"的情况

## 数据流

```
C# Dataset.CanonicalizeBlankNodes()
  → NativeMethods.dataset_canonicalize_blank_nodes()
  → ffi.rs oxigraph_dataset_canonicalize_blank_nodes()
  → oxrdf dataset.canonicalize_blank_nodes()
  → HashMap<BlankNode, BlankNode>
  → JSON {"ok": {"b0": "c14n0", ...}}
  → C# 反序列化为 IReadOnlyDictionary<BlankNode, BlankNode>
```

## 测试

### 单元测试（ModelTests.cs）

1. **基本往返**：创建含 BlankNode 的 Dataset → 调用 `CanonicalizeBlankNodes(UNSTABLE)` → 验证返回映射包含原始 ID 和规范 ID
2. **RDFC-1.0**：使用 `Rdfc10Sha256` 算法
3. **UnstableHashedIds**：使用 `UnstableHashedIds` 算法，验证多次调用 ID 稳定
4. **空数据集**：验证返回空字典
5. **无空白节点**：验证无空白节点时返回空字典

### 测试示例

```csharp
[Fact]
public void CanonicalizeBlankNodes_ReturnsCorrectMapping()
{
    var ds = new Dataset(new[]
    {
        new Quad(new BlankNode("a"), new NamedNode("http://example.com/p"), new Literal("b"))
    });

    var mapping = ds.CanonicalizeBlankNodes(CanonicalizationAlgorithm.Unstable);

    Assert.Single(mapping);
    Assert.Contains(new BlankNode("a"), mapping.Keys);
    Assert.NotEqual(new BlankNode("a"), mapping[new BlankNode("a")]);
}
```

## 约束

- FFI 调用**仅同步**（无 Task-based 异步）
- 返回类型为**不可变**（`IReadOnlyDictionary`）以强制只读契约
- 默认算法为 **Unstable**（与 Python 默认值一致）
- 不修改现有 `Canonicalize()` 方法 — 新方法为纯添加
