# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目性质

xCAD.NET — Xarial Pty Limited 维护的 **CAD 无关 .NET 框架**，用于在 SOLIDWORKS 与 Autodesk Inventor 上构建插件、Property Manager Page、自定义宏特征与独立控制台应用。仓库本身是该框架的源码 + NuGet 打包配置 + `dotnet new` 模板。

仓库已声明的开源协议见 `LICENSE.md`；发布包与官网在 <https://xcad.net> 与 <https://www.nuget.org/packages/Xarial.XCad>。当前版本前缀 `0.8.3$(BuildNumber)`（`common/common.props`）。

## 解决方案结构（xcad.sln）

顶层目录按职责切分——只看目录名就能判断模块归属：

| 目录 | 角色 | 备注 |
| --- | --- | --- |
| `src/Base` | CAD-agnostic 公共接口（`Xarial.XCad`） | TFM `net7.0;net6.0;netstandard2.1;net461` |
| `src/Toolkit` | CAD-agnostic 工具实现（`Xarial.XCad.Toolkit`） | 同上 TFM，被三个 CAD 包项目引用 |
| `src/SolidWorks` | SOLIDWORKS COM 实现（`Xarial.XCad.SolidWorks`） | `Microsoft.NET.Sdk.WindowsDesktop` + `thirdpty/SolidWorks.Interop.*.dll` |
| `src/SwDocumentManager` | SOLIDWORKS Document Manager 实现 | 同上 WindowsDesktop |
| `src/Inventor` | Autodesk Inventor 实现 | 同上 WindowsDesktop |
| `samples/` | 示例工程（`SwAddIn`、`StandAlone`、`StandAlone.Inventor`） | 仅作演示 |
| `templates/` | `dotnet new install` 的模板聚合 `Xarial.XCad.Templates.csproj` | 子目录 SwAddIn / SwAddInVB / SwMacroFeatureAddIn / Console / Wpf |
| `tools/InventorStandAloneConnectorAddIn` | Inventor 独立模式连接器 | **必须先编译**——Inventor 包从 `tools/_build/` 拾取 `Xarial.XCad.Inventor.Tools.StandAloneConnector.dll` 并打进 NuGet |
| `tests/<Name>.Tests/` | 单元测试（NUnit + Moq） | 全部以 net461 编译 |
| `tests/integration/SolidWorks.Tests.Integration`、`SolidWorksDocMgr.Tests.Integration` | 集成测试 | 需宿主 SOLIDWORKS + 测试数据 |
| `docs/_src` | DocFX 文档源 | `Docs.csproj` |
| `build/` | MSBuild 目标、PowerShell 初始化脚本、Azure Pipelines yml | 见下方"自动注册" |
| `thirdpty/` | 厂商 COM interop（不可嵌入，`EmbedInteropTypes=false`） | 使用方需要在自家项目把对应 `*.tlb` 设为 Embed Interop=True |
| `common/common.props` | 全局 NuGet 元数据（公司、版权、SourceLink、符号包格式 `snupkg`） | 所有发布包项目都 `Import` 它 |

## 架构（高层抽象）

整个框架围绕 `IXApplication` 这个 CAD-agnostic 顶层接口展开，每个 CAD 提供自己的具体实现与工厂：

- **`IXApplication`**（`src/Base/IXApplication.cs`）— 顶层对象；`Documents` 仓库、`MemoryGeometryBuilder`、`CreateProgress()`、`ShowMessageBox` 等。工厂：`SwApplicationFactory.Create(SwVersion_e.Sw2022, ApplicationState_e.Silent)` / `SwDmApplicationFactory.Create(licenseKey)`。
- **`IXObject`**（`src/Base/IXObject.cs`）— 所有包装器的根接口；持有 `OwnerApplication`、`OwnerDocument`、`IsAlive`、`Tags`、`Serialize`。所有 `IXFoo` 接口继承自它。
- 对象树：`IXApplication → IXDocumentRepository → IXDocument → IXPart / IXAssembly / IXDrawing → IXFeature → IXBody / IXSketch ...`。
- **三个面向插件开发者的抽象**（即 README 着重展示的部分）：
  1. `SwAddInEx`（`src/SolidWorks/SwAddInEx.cs`）— COM add-in 基类，重写 `OnConnect` 注册 CommandGroup。
  2. `IXPropertyPage<TDataModel>`（在 `Toolkit/PageBuilder/`）— **声明式 Property Manager Page**：定义数据模型类加 `[ComVisible]` 后，框架反射属性自动构建 PMP 控件并双向绑定；基类是 `SwPropertyManagerPageHandler`。
  3. `SwMacroFeatureDefinition<TParam, TData>`（`Toolkit/CustomFeature/BaseCustomFeatureEditor.cs` + `CustomFeatureParametersParser.cs`）— 宏特征：重写 `CreateGeometry` 即可；参数解析器读 `TParam` 上的属性。
- **依赖注入**：内部用 `IXServiceCollection` + `ServiceProvider`（`src/Toolkit/ServiceCollection.cs`、`ServiceProvider.cs`）管理 `DocumentsHandler`、`EventsHandler`、`BaseIconsCreator` 等跨切面服务。

子目录含义（看名字即可）：
- `src/Base/` 与 `src/SolidWorks/`、`src/SwDocumentManager/` 都有相同的 `Documents/、Features/、Geometry/、Graphics/、Sketch/、Data/` 镜像目录——Base 放 `IXFoo`，CAD 子包放 `SwFoo : IXFoo`。
- `src/Base/Annotations/`、`src/Base/UI/` 放抽象层；`src/Base/Attributes/` 与 `src/Base/Reflection/` 是基础设施。
- `Toolkit/CustomFeature/` 只在 Toolkit；`Toolkit/PageBuilder/` 是声明式 Page 构建器（`Base/Binders/Constructors/Core/PageElements/Attributes/Internal`）。

## 构建与运行

仓库是标准 .NET SDK 解决方案，但包含 Windows-only 子项目。**所有 `Microsoft.NET.Sdk.WindowsDesktop` 项目（`SolidWorks`、`SwDocumentManager`、`Inventor`）只能在 Windows 上编译。**

| 任务 | 命令 |
| --- | --- |
| 还原 + 编译全部 | `dotnet build xcad.sln -c Release` |
| 仅编译 Base 接口 | `dotnet build src/Base/Base.csproj` |
| 仅编译 Toolkit | `dotnet build src/Toolkit/Toolkit.csproj -f net7.0` |
| 编译 Inventor 工具（Pack 前置） | `dotnet build tools/InventorStandAloneConnectorAddIn/InventorStandAloneConnectorAddIn.csproj -c Release`（输出到 `tools/_build/`，被 Inventor 包拾取） |
| 单测项目 | `dotnet test tests/Base.Tests/Base.Tests.csproj -c Release` |
| 跑 SolidWorks 单测 | `dotnet test tests/SolidWorks.Tests/SolidWorks.Tests.csproj`（引用 `thirdpty/SolidWorks.Interop.*.dll`） |
| 跑集成测试 | `dotnet test tests/integration/SolidWorks.Tests.Integration/SolidWorks.Tests.Integration.csproj`（需设置环境变量，见下） |
| 打包 NuGet | `dotnet pack src/Base/Base.csproj -c Release -o /tmp/nupkgs`（不走 `common.props` 的 `_build/`，但 CI 用的是该路径） |
| 打包模板 | `dotnet pack templates/Xarial.XCad.Templates.csproj -c Release -o /tmp/nupkgs` |

**集成测试前置（README 原文）**：
- 下载 Test Data（OneDrive 链接在 README），解压到任意目录。
- 设置环境变量 `XCAD_TEST_DATA` → 该目录路径。
- Document Manager 测试还需 `SW_DM_KEY` → 你的 Document Manager 许可证密钥。
- 必须在本机安装对应版本的 SOLIDWORKS。

**测试栈**：`NUnit 3.13.3`、`NUnit3TestAdapter 4.5.0`、`Moq 4.18.3`、`Microsoft.NET.Test.Sdk 17.7.2`。

## 自动注册（SOLIDWORKS add-in）

`build/Xarial.XCad.SolidWorks.targets` 被打入 `Xarial.XCad.SolidWorks` NuGet 包，**消费方引用此 NuGet 即自动注册/卸载 COM add-in**：

- .NET Framework → 用 `%windir%\Microsoft.NET\Framework64\v4.0.30319\RegAsm.exe /codebase`（或加 `/tlb`）。
- .NET Core / .NET 5/6/7 → **消费方 csproj 必须加** `<EnableComHosting>true</EnableComHosting>`，否则构建报 `<Error>` 提示。
- 想关闭：`在 PropertyGroup 加 `XCadRegDll>false`。
- .NET Core（不含 .NET 6+）还要手动写 `[ComRegisterFunction]`/`[ComUnregisterFunction]` 调 `SwAddInEx.RegisterFunction/UnregisterFunction`——README 末尾有完整代码。

注册发生在 `AfterBuild`，卸载发生在 `BeforeClean`。

## CI / 发布

`build/pipeline.yml`（Azure DevOps，多 stage）：

1. `CompileTools` → 编译 `tools/**/*.csproj`，签名为 `xarial.snk`，产物发到 `tools` artifact。
2. `SignTools`（仅 master / !alpha / !beta）→ 用本机签名工具二次签名，并做签名校验。
3. `CompileAndPack` → restore `**/*.sln`；先跑 `tests\*.Tests\*.csproj` 的单元测试；再编译 `src/**/*.csproj`；按分支名打 `packages/<versionSuffix>/` 的 NuGet。
4. `PublishDevNuGetPackage`（仅 `dev` 分支）→ 推到 `xcad/xcad-dev` 内部 feed。
5. `PublishPublicPackage`（仅 master / !alpha / !beta）→ 推到 nuget.org。

分支与版本后缀对应：`master` → release；`dev` → dev；`!alpha` → alpha；`!beta` → beta。

`build/templates-pipeline.yml` 单独打包模板 NuGet（含 `dotnet nuget sign` 用 `xarial-code-signing.pfx`）。

## 编码约定（来自代码本身）

- 文件头统一 `//xCAD / Copyright(C) <year> Xarial Pty Limited / Product URL / License`——新增 C# 文件照搬即可。
- `AssemblyInfo.cs` 极简，不写 `[assembly: ...]` 元数据（公司、版本等都在 `common.props`）。
- 命名空间严格随目录：`Xarial.XCad` 在 `src/Base` 根；`Xarial.XCad.Base.Documents`、`Xarial.XCad.Toolkit.PageBuilder` 等子命名空间。CAD 实现：`Xarial.XCad.SolidWorks.Documents` 等。
- `[ComVisible(true)]` 在 add-in 与跨 COM 边界类上显式标注（参考 `docs/_src/Extension/CommandsManager/CommandsAddIn.cs`）。
- 泛型数据模型驱动 UI：`SwPropertyManagerPageHandler<T>`、`SwMacroFeatureDefinition<TParam, TData>`——不要绕过它去手动构 PMP。
- 强命名由 CI 加；本地 csproj `SignAssembly=false` 且 `AssemblyOriginatorKeyFile` 留空，**不要本地加 key**。
- 项目内的公共扩展（`XServiceCollectionExtension`、`XLoggerExtension`、`XRepositoryExtension`）写在 `src/Base/` 旁边的小文件里，命名 `<Name>Extension.cs`。

## 不要做的事

- 不要修改 `common/common.props` 里的版本号/版权——CI 通过 MSBuild 覆盖。
- 不要把 `thirdpty/*.dll` 改成 `EmbedInteropTypes=true`，消费方才有权控制嵌入。
- 不要在 `src/SolidWorks` 或 `src/Inventor` 里改 TFM 列表——`Microsoft.NET.Sdk.WindowsDesktop` 是为 SOLIDWORKS COM 互操作刻意选的。
- 不要跳过 `tools/_build/` 编译就去 pack Inventor 包；Inventor 包的 `<None Include="..\..\tools\_build\...">` 会拿不到文件。
- 写新 add-in 示例时不要直接复制 `Xarial.XCad.Toolkit` 里的反射工具，**消费方只用 `Xarial.XCad.SolidWorks` 包**，Toolkit 是私有基础设施。