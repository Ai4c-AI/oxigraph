# CanonicalizeBlankNodesAsync 功能对等设计

**日期**: 2026-08-13
**状态**: 待用户审核

## 目标

为 `Dataset.CanonicalizeBlankNodes` 添加异步版本 `CanonicalizeBlankNodesAsync`，与现有 `CanonicalizeAsync` 模式保持一致。

## 范围

### 需修改的文件

| 层级 | 文件 | 变更内容 |
|------|------|---------|
| C# 方法 | `dotnet/src/Oxigraph/Dataset.cs` | 新增 `CanonicalizeBlankNodesAsync` |
| C# 测试 | `dotnet/tests/Oxigraph.Tests/ModelTests.cs` | 新增异步版本测试 |
| 文档 | `dotnet/docs/model.md` | 补充 `CanonicalizeBlankNodesAsync` 说明 |

## 测试

### 单元测试（ModelTests.cs）

在现有 `CanonicalizeBlankNodes` 测试附近添加异步版本：

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

## 设计

### 1. C# Dataset 方法

**文件**: `dotnet/src/Oxigraph/Dataset.cs`

```csharp
/// <inheritdoc cref="CanonicalizeBlankNodes" />
public Task<IReadOnlyDictionary<BlankNode, BlankNode>> CanonicalizeBlankNodesAsync(
    CanonicalizationAlgorithm algorithm = CanonicalizationAlgorithm.Unstable,
    CancellationToken ct = default)
    => Task.Run(() => CanonicalizeBlankNodes(algorithm), ct);
```

### 2. 文档更新

**文件**: `dotnet/docs/model.md`

在 `CanonicalizeBlankNodes` 方法说明后添加 `CanonicalizeBlankNodesAsync`。

## 约束

- FFI 调用通过 `Task.Run` 封装到 ThreadPool，与现有 `CanonicalizeAsync` 模式一致
- 参数、返回值与同步版本完全相同
- 不修改 Rust FFI 层
