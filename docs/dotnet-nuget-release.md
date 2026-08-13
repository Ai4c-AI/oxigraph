# Oxigraph .NET NuGet 包发布流程

## 版本对齐

当前版本统一为 **0.6.0-dev**，与 PyOxigraph 版本对齐。

| 项目 | 版本 | 文件 |
|------|------|------|
| PyOxigraph | `0.6.0-dev` | `python/Cargo.toml` |
| oxigraph-dotnet | `0.6.0-dev` | `dotnet/src/oxigraph-dotnet/Cargo.toml` |
| Oxigraph NuGet | `0.6.0-dev` | `dotnet/Directory.Build.props` |

## 发布方式

### 方式一：本地构建（仅 Windows）

适用于仅生成 Windows 原生 DLL 的 NuGet 包，**不含 Linux/macOS 原生库**。

```bash
# 1. 构建 Rust 原生库
cd dotnet/src/oxigraph-dotnet
cargo build --release

# 2. 复制到 runtimes 目录
mkdir -p dotnet/src/Oxigraph/runtimes/win-x64/native
cp target/release/oxigraph_dotnet.dll dotnet/src/Oxigraph/runtimes/win-x64/native/

# 3. 打包 Oxigraph
cd dotnet
dotnet pack src/Oxigraph/Oxigraph.csproj -c Release -o ./nupkg

# 4. 打包 DotNetRDF 扩展
dotnet pack src/Oxigraph.Extensions.DotNetRDF/Oxigraph.Extensions.DotNetRDF.csproj -c Release -o ./nupkg

# 5. 查看生成的包
ls nupkg/
```

**本地构建的局限性：**
- 只能生成 Windows (`win-x64`) 原生 DLL
- 不包含 Linux/macOS 原生库

### 方式二：CI Workflow（推荐，完整多平台）

通过 GitHub Actions 构建全平台原生库并打包 NuGet。

#### 1. 触发 Workflow

```bash
# 安装 gh CLI
winget install --id GitHub.cli -e --accept-source-agreements

# 登录 GitHub
gh auth login

# 触发 workflow（仅构建 NuGet，跳过其他资产）
gh workflow run artifacts.yml -f nuget-only=true --ref dotnet
# 注：--ref dotnet 指定使用 dotnet 分支的 workflow 配置和代码
```

#### 2. 查看构建状态

```bash
gh run view <run-id>
# 或访问
https://github.com/Ai4c-AI/oxigraph/actions/workflows/artifacts.yml
```

#### 3. 下载构建产物

Workflow 运行完成后，下载 `oxigraph_nuget` artifact：

```bash
gh run download <run-id> --name oxigraph_nuget
```

artifact 包含：
- `Oxigraph.0.6.0-dev.nupkg` — 含 win-x64 原生 DLL
- `Oxigraph.Extensions.DotNetRDF.0.6.0-dev.nupkg`

**注意：** 由于 `nuget-only=true`，artifact 中**不包含** Linux/macOS 原生库。如需多平台支持，需触发完整 workflow（去掉 `nuget-only=true`）。

#### 4. 手工上传到 nuget.org

1. 访问 https://www.nuget.org/packages/manage/upload
2. 上传 `.nupkg` 文件
3. 填写包信息并发布

## CI Workflow 说明

### 相关 Job

| Job | 平台 | 构建内容 |
|-----|------|---------|
| `dotnet_native_linux` | Ubuntu | `liboxigraph_dotnet.so` (x64, arm64) |
| `dotnet_native_macos` | macOS | `liboxigraph_dotnet.dylib` (x64, arm64) |
| `dotnet_native_windows` | Windows | `oxigraph_dotnet.dll` (x64) |
| `dotnet_pack` | Ubuntu | 合并所有平台原生库，打包 NuGet |

### Workflow 触发条件

| 触发方式 | 条件 |
|---------|------|
| Push to `main` | 自动触发完整构建 |
| Push to `dotnet` | 自动触发完整构建 |
| Release published | 自动触发完整构建 |
| Manual dispatch | `workflow_dispatch` 触发（可设置 `nuget-only`） |

### 本地修复 gh CLI

```bash
# 如果 gh 命令找不到，使用完整路径
"/c/Program Files/GitHub CLI/gh.exe" workflow run artifacts.yml -f nuget-only=true --ref dotnet
```

## NuGet 包结构

Oxigraph NuGet 包采用 **Native Library 模式**：

```
nuget_package/
├── lib/net10.0/Oxigraph.dll          # C# managed assembly
├── runtimes/
│   ├── win-x64/native/oxigraph_dotnet.dll
│   ├── linux-x64/native/liboxigraph_dotnet.so
│   ├── linux-arm64/native/liboxigraph_dotnet.so
│   ├── osx-x64/native/liboxigraph_dotnet.dylib
│   └── osx-arm64/native/liboxigraph_dotnet.dylib
└── README.md
```

运行时，.NET 会根据运行平台自动加载对应的原生库。

## 相关文件

| 文件 | 说明 |
|------|------|
| `dotnet/src/Oxigraph/Oxigraph.csproj` | NuGet 包配置（含 runtimes 条件引入） |
| `dotnet/src/Oxigraph/runtimes/` | CI 下载原生库的目标目录 |
| `dotnet/src/oxigraph-dotnet/Cargo.toml` | Rust cdylib 构建配置 |
| `.github/workflows/artifacts.yml` | CI workflow 定义 |

## 常见问题

**Q: 本地打包的 NuGet 包缺少 Linux DLL？**
A: 正常，本地只能构建 Windows DLL。需通过 CI 构建多平台库。

**Q: `dotnet pack` 报错 "PackageReference not found"？**
A: 确保先运行 `dotnet restore`。

**Q: NuGet 包上传失败？**
A: 检查版本号格式，`0.6.0-dev` 是预发布版本，需勾选 "Pre-release" 选项。
