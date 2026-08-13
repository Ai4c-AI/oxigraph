# Canonicalize Blank Nodes 功能对等 — 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**目标**：在 Dotnet 侧实现 `CanonicalizeBlankNodes` 方法和 `UnstableHashedIds` 枚举值，与 Python commit `061df048` 功能对等。

**架构**：Rust FFI 层新增 `oxigraph_dataset_canonicalize_blank_nodes` 函数，通过 JSON over FFI 模式返回 `HashMap<BlankNode, BlankNode>` 序列化结果；C# binding 解析 JSON 并返回 `IReadOnlyDictionary<BlankNode, BlankNode>`。

**技术栈**：Rust（ffi.rs）、C#（RdfFormat.cs、Dataset.cs、ModelTests.cs）、P/Invoke Source Generator

---

## 全局约束

- FFI 调用仅同步（无 Task-based 异步）
- 返回类型为 `IReadOnlyDictionary<BlankNode, BlankNode>`，强制只读契约
- 默认算法为 `CanonicalizationAlgorithm.Unstable`，与 Python 默认值一致
- JSON 格式：`{"ok": {"<original_id>": "<canonical_id>"}}`，错误格式：`{"error": "message"}`

---

## 文件概览

| 文件 | 职责 |
|------|------|
| `dotnet/src/oxigraph-dotnet/src/ffi.rs` | Rust FFI 入口，解析算法字符串，调用 `canonicalize_blank_nodes`，序列化 HashMap 为 JSON |
| `dotnet/src/Oxigraph/RdfFormat.cs` | `CanonicalizationAlgorithm` 枚举新增 `UnstableHashedIds` |
| `dotnet/src/Oxigraph/Dataset.cs` | 新增 `CanonicalizeBlankNodes()` 方法 |
| `dotnet/tests/Oxigraph.Tests/ModelTests.cs` | 新增单元测试 |
| `dotnet/docs/model.md` | 补充 `CanonicalizeBlankNodes` 和 `UnstableHashedIds` 文档 |

---

## Task 1: Rust FFI 层 — 新增 `oxigraph_dataset_canonicalize_blank_nodes`

**文件：**
- 修改：`dotnet/src/oxigraph-dotnet/src/ffi.rs:2257`（在 `oxigraph_dataset_canonicalize` 之后插入）

**接口：**
- 消费：`handle: DatasetHandle`, `algorithm: *const c_char`
- 产生：`oxigraph_dataset_canonicalize_blank_nodes(handle, algorithm) -> *mut c_char`

**参考**：Python `python/src/dataset.rs:232-237`，Rust `lib/oxrdf/src/dataset.rs:598-603`

- [ ] **Step 1: 在 `ffi.rs` 中 `oxigraph_dataset_canonicalize` 函数之后插入新函数**

在第 2257 行后添加：

```rust
/// Returns a map between the current dataset blank nodes and the canonicalized blank nodes.
#[unsafe(no_mangle)]
pub extern "C" fn oxigraph_dataset_canonicalize_blank_nodes(handle: DatasetHandle, algorithm: *const c_char) -> *mut c_char {
    if handle.is_null() {
        return error_json(ErrorKind::InvalidArgument { message: "Dataset handle is null".into() });
    }
    let dataset = unsafe { &mut *(*handle).get() };
    let algo_str = unsafe { c_str_to_str(algorithm) };
    let algo = match algo_str {
        "unstable" => oxigraph::model::dataset::CanonicalizationAlgorithm::Unstable,
        "unstable_hashed_ids" => oxigraph::model::dataset::CanonicalizationAlgorithm::UnstableHashedIds,
        "rdfc10_sha256" => oxigraph::model::dataset::CanonicalizationAlgorithm::Rdfc10 {
            hash_algorithm: oxigraph::model::dataset::CanonicalizationHashAlgorithm::Sha256,
        },
        "rdfc10_sha384" => oxigraph::model::dataset::CanonicalizationAlgorithm::Rdfc10 {
            hash_algorithm: oxigraph::model::dataset::CanonicalizationHashAlgorithm::Sha384,
        },
        _ => return error_json(ErrorKind::InvalidArgument { message: format!("Unknown canonicalization algorithm: {algo_str}") }),
    };
    let mapping = dataset.canonicalize_blank_nodes(algo);
    let map: Map<String, Value> = mapping
        .into_iter()
        .map(|(k, v)| (k.to_string(), Value::String(v.to_string())))
        .collect();
    ok_json(&Value::Object(map))
}
```

- [ ] **Step 2: 运行 Rust 检查**

```bash
cd dotnet/src/oxigraph-dotnet && cargo check
```

预期：编译通过（可能有 unused warnings，不影响）

- [ ] **Step 3: 提交**

```bash
git add dotnet/src/oxigraph-dotnet/src/ffi.rs
git commit -m "feat(ffi): add oxigraph_dataset_canonicalize_blank_nodes"
```

---

## Task 2: C# 枚举 — 新增 `UnstableHashedIds`

**文件：**
- 修改：`dotnet/src/Oxigraph/RdfFormat.cs:43-51`

**接口：**
- 消费：无
- 产生：`CanonicalizationAlgorithm.UnstableHashedIds` 枚举值

- [ ] **Step 1: 在 `RdfFormat.cs` 的 `CanonicalizationAlgorithm` 枚举中添加 `UnstableHashedIds`**

在 `Rdfc10Sha384` 之后添加：

```csharp
/// <summary>Oxigraph 首选算法，但输出的 ID 基于哈希值。</summary>
/// <remarks>
/// 这使得空白节点 ID 可用于 diff 场景，添加或删除 triple 时影响的空白节点 ID 更少。
/// 警告：可能在 Oxigraph 版本间发生变化，不保证稳定性。
/// </remarks>
UnstableHashedIds,
```

- [ ] **Step 2: 验证 build**

```bash
cd dotnet && dotnet build src/Oxigraph/Oxigraph.csproj
```

预期：编译通过

- [ ] **Step 3: 提交**

```bash
git add dotnet/src/Oxigraph/RdfFormat.cs
git commit -m "feat: add CanonicalizationAlgorithm.UnstableHashedIds"
```

---

## Task 3: C# Dataset 方法 — 新增 `CanonicalizeBlankNodes()`

**文件：**
- 修改：`dotnet/src/Oxigraph/Dataset.cs`（在 `Canonicalize` 方法之后插入）

**接口：**
- 消费：`CanonicalizationAlgorithm algorithm`（默认 `Unstable`）
- 产生：`IReadOnlyDictionary<BlankNode, BlankNode>`

- [ ] **Step 1: 在 `Dataset.cs` 的 `Canonicalize` 方法之后添加 `CanonicalizeBlankNodes` 方法**

在第 183 行后添加：

```csharp
/// <summary>
/// 返回当前数据集中空白节点到规范化空白节点的映射，用于创建规范化数据集。
///
/// 详见 <see cref="Canonicalize"/>。
/// </summary>
/// <param name="algorithm">要使用的规范化算法。</param>
/// <returns>原始空白节点到规范化空白节点的映射。</returns>
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

- [ ] **Step 2: 验证 build**

```bash
cd dotnet && dotnet build src/Oxigraph/Oxigraph.csproj
```

预期：编译通过

- [ ] **Step 3: 提交**

```bash
git add dotnet/src/Oxigraph/Dataset.cs
git commit -m "feat: add Dataset.CanonicalizeBlankNodes method"
```

---

## Task 4: C# 测试 — 新增单元测试

**文件：**
- 修改：`dotnet/tests/Oxigraph.Tests/ModelTests.cs`（在 `CanonicalizationAlgorithm` 测试附近添加）

**接口：**
- 消费：`Dataset`、`CanonicalizeBlankNodes()`
- 产生：5 个测试用例

- [ ] **Step 1: 在 `ModelTests.cs` 中添加测试**

在现有 `CanonicalizationAlgorithm` 测试附近添加：

```csharp
[Fact]
public void CanonicalizeBlankNodes_Unstable_ReturnsCorrectMapping()
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

[Fact]
public void CanonicalizeBlankNodes_Rdfc10Sha256_ReturnsCorrectMapping()
{
    var ds = new Dataset(new[]
    {
        new Quad(new BlankNode("a"), new NamedNode("http://example.com/p"), new Literal("b"))
    });

    var mapping = ds.CanonicalizeBlankNodes(CanonicalizationAlgorithm.Rdfc10Sha256);

    Assert.Single(mapping);
    Assert.Contains(new BlankNode("a"), mapping.Keys);
}

[Fact]
public void CanonicalizeBlankNodes_UnstableHashedIds_ReturnsStableIds()
{
    var ds = new Dataset(new[]
    {
        new Quad(new BlankNode("a"), new NamedNode("http://example.com/p"), new Literal("b"))
    });

    var mapping1 = ds.CanonicalizeBlankNodes(CanonicalizationAlgorithm.UnstableHashedIds);
    var mapping2 = ds.CanonicalizeBlankNodes(CanonicalizationAlgorithm.UnstableHashedIds);

    Assert.Single(mapping1);
    Assert.Equal(mapping1[new BlankNode("a")], mapping2[new BlankNode("a")]);
}

[Fact]
public void CanonicalizeBlankNodes_EmptyDataset_ReturnsEmptyDictionary()
{
    var ds = new Dataset();

    var mapping = ds.CanonicalizeBlankNodes();

    Assert.Empty(mapping);
}

[Fact]
public void CanonicalizeBlankNodes_NoBlankNodes_ReturnsEmptyDictionary()
{
    var ds = new Dataset(new[]
    {
        new Quad(new NamedNode("http://example.com/s"), new NamedNode("http://example.com/p"), new Literal("o"))
    });

    var mapping = ds.CanonicalizeBlankNodes();

    Assert.Empty(mapping);
}
```

- [ ] **Step 2: 运行测试验证**

```bash
cd dotnet && dotnet test tests/Oxigraph.Tests/Oxigraph.Tests.csproj --filter "FullyQualifiedName~CanonicalizeBlankNodes"
```

预期：5 个测试全部 PASS

- [ ] **Step 3: 提交**

```bash
git add dotnet/tests/Oxigraph.Tests/ModelTests.cs
git commit -m "test: add CanonicalizeBlankNodes unit tests"
```

---

## Task 5: 文档更新

**文件：**
- 修改：`dotnet/docs/model.md`

**接口：**
- 消费：无
- 产生：更新的文档

- [ ] **Step 1: 更新 `model.md` 文档**

在 `CanonicalizationAlgorithm` 枚举部分添加 `UnstableHashedIds` 说明，在 `Canonicalize` 方法部分添加 `CanonicalizeBlankNodes` 链接。

具体添加位置参考现有 `CanonicalizationAlgorithm` 和 `Canonicalize` 文档格式。

- [ ] **Step 2: 提交**

```bash
git add dotnet/docs/model.md
git commit -m "docs: add CanonicalizeBlankNodes and UnstableHashedIds documentation"
```

---

## 自检清单

**Spec 覆盖检查：**
- [x] Rust FFI 新增 `oxigraph_dataset_canonicalize_blank_nodes` → Task 1
- [x] `CanonicalizationAlgorithm.UnstableHashedIds` 枚举 → Task 2
- [x] `Dataset.CanonicalizeBlankNodes()` 方法 → Task 3
- [x] 单元测试（5 个用例）→ Task 4
- [x] 文档更新 → Task 5

**占位符检查：** 无 "TBD"、"TODO"、未完成步骤

**类型一致性：** `IReadOnlyDictionary<BlankNode, BlankNode>` 返回类型贯穿 Task 3 和 Task 4
