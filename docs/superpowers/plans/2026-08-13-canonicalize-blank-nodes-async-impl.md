# CanonicalizeBlankNodesAsync 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**目标**：为 `Dataset.CanonicalizeBlankNodes` 添加异步版本 `CanonicalizeBlankNodesAsync`。

**架构**：`Task.Run` 封装同步调用到 ThreadPool，与 `CanonicalizeAsync` 模式完全一致。

**技术栈**：C#（Dataset.cs、ModelTests.cs、model.md）

---

## 全局约束

- `Task.Run` 模式与现有 `CanonicalizeAsync` 一致
- 参数、返回值与同步版本完全相同
- 不修改 Rust FFI 层

---

## Task 1: C# Dataset 方法 — CanonicalizeBlankNodesAsync

**文件：**
- 修改：`dotnet/src/Oxigraph/Dataset.cs`

**接口：**
- 消费：`CanonicalizationAlgorithm algorithm`（默认 `Unstable`），`CancellationToken ct`
- 产生：`Task<IReadOnlyDictionary<BlankNode, BlankNode>>`

- [ ] **Step 1: 在 `CanonicalizeAsync` 之后添加 `CanonicalizeBlankNodesAsync` 方法**

在 `Dataset.cs` 的 Async API 区域添加：

```csharp
/// <inheritdoc cref="CanonicalizeBlankNodes" />
public Task<IReadOnlyDictionary<BlankNode, BlankNode>> CanonicalizeBlankNodesAsync(
    CanonicalizationAlgorithm algorithm = CanonicalizationAlgorithm.Unstable,
    CancellationToken ct = default)
    => Task.Run(() => CanonicalizeBlankNodes(algorithm), ct);
```

- [ ] **Step 2: 验证 build**

```bash
cd dotnet && dotnet build src/Oxigraph/Oxigraph.csproj
```

预期：编译通过

- [ ] **Step 3: 提交**

```bash
git add dotnet/src/Oxigraph/Dataset.cs
git commit -m "feat: add Dataset.CanonicalizeBlankNodesAsync"
```

---

## Task 2: C# 测试 — 新增单元测试

**文件：**
- 修改：`dotnet/tests/Oxigraph.Tests/ModelTests.cs`

**接口：**
- 消费：`Dataset.CanonicalizeBlankNodesAsync()`
- 产生：2 个测试用例

- [ ] **Step 1: 在现有 `CanonicalizeBlankNodes` 测试后添加异步测试**

在 `CanonicalizeBlankNodes_EmptyDataset_ReturnsEmptyDictionary` 测试之后添加：

```csharp
[Fact]
public async Task CanonicalizeBlankNodesAsync_ReturnsCorrectMapping()
{
    using var ds = new Dataset();
    ds.Add(new Quad(new BlankNode("a"), new NamedNode("http://example.com/p"), new Literal("b"), new DefaultGraph()));

    var mapping = await ds.CanonicalizeBlankNodesAsync(CanonicalizationAlgorithm.Unstable);

    Assert.Single(mapping);
    Assert.Contains(new BlankNode("a"), mapping.Keys);
}

[Fact]
public async Task CanonicalizeBlankNodesAsync_CancellationToken_Cancels()
{
    using var ds = new Dataset();
    ds.Add(new Quad(new BlankNode("a"), new NamedNode("http://example.com/p"), new Literal("b"), new DefaultGraph()));
    using var cts = new CancellationTokenSource();
    cts.Cancel();

    await Assert.ThrowsAsync<TaskCanceledException>(
        () => ds.CanonicalizeBlankNodesAsync(CanonicalizationAlgorithm.Unstable, cts.Token));
}
```

- [ ] **Step 2: 运行测试验证**

```bash
cd dotnet && dotnet test tests/Oxigraph.Tests/Oxigraph.Tests.csproj --filter "FullyQualifiedName~CanonicalizeBlankNodesAsync"
```

预期：2 个测试全部 PASS

- [ ] **Step 3: 提交**

```bash
git add dotnet/tests/Oxigraph.Tests/ModelTests.cs
git commit -m "test: add CanonicalizeBlankNodesAsync unit tests"
```

---

## Task 3: 文档更新

**文件：**
- 修改：`dotnet/docs/model.md`

**接口：**
- 消费：无
- 产生：更新的文档

- [ ] **Step 1: 更新 `model.md` 文档**

在 `CanonicalizeBlankNodes` 方法行添加 `CanonicalizeBlankNodesAsync`：

```
| `CanonicalizeBlankNodes(CanonicalizationAlgorithm)` → `IReadOnlyDictionary<BlankNode, BlankNode>` | Returns a mapping from original blank nodes to canonicalized blank nodes |
| `CanonicalizeBlankNodesAsync(CanonicalizationAlgorithm, CancellationToken)` → `Task<IReadOnlyDictionary<BlankNode, BlankNode>>` | Async version of CanonicalizeBlankNodes |
```

- [ ] **Step 2: 提交**

```bash
git add dotnet/docs/model.md
git commit -m "docs: add CanonicalizeBlankNodesAsync documentation"
```

---

## 自检清单

**Spec 覆盖检查：**
- [x] `CanonicalizeBlankNodesAsync` 方法 → Task 1
- [x] 异步测试（2 个用例）→ Task 2
- [x] 文档更新 → Task 3

**占位符检查：** 无 "TBD"、"TODO"、未完成步骤

**类型一致性：** `Task<IReadOnlyDictionary<BlankNode, BlankNode>>` 返回类型贯穿 Task 1 和 Task 2
