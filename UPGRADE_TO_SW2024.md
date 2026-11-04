# SW2URDF 插件适配 SOLIDWORKS 2024 修改记录

本文档记录了将 [solidworks_urdf_exporter](https://github.com/ros/solidworks_urdf_exporter)（SW2URDF）插件从 SOLIDWORKS 2021 适配升级到 **SOLIDWORKS 2024** 过程中的排查思路与具体修改点，供参考学习。

## 背景

项目原本是针对 SOLIDWORKS 2021（及更早版本）编写的 VSTO/COM 插件，目标框架为 .NET Framework 4.5.2 / 4.8。随着 SOLIDWORKS 升级到 2024 版本，需要确认插件是否仍然可以正常编译、注册、加载和运行。

## 排查与修改步骤

### 1. 确认项目中没有硬编码的版本号

检查了整个代码库，确认：

- COM 注册表路径是通用的，不绑定具体 SOLIDWORKS 版本：

  ```csharp
  // SW2URDF/SW/SwAddin.cs
  string keyname = "SOFTWARE\\SolidWorks\\Addins\\{" + t.GUID.ToString() + "}";
  keyname = "Software\\SolidWorks\\AddInsStartup\\{" + t.GUID.ToString() + "}";
  ```

  这意味着插件注册后，任意版本的 SOLIDWORKS（只要启动时读取这个注册表项）都能识别并加载该插件，不需要针对 2024 修改注册逻辑。

- 代码中没有出现 `2021`、`SwVersion` 等版本判断逻辑，说明没有版本相关的硬编码分支需要调整。

**结论：无需修改业务代码来“支持”新版本，关键在于引用的 Interop 库版本和编译环境。**

### 2. 检查 SOLIDWORKS Interop 引用路径

`SW2URDF/SW2URDF.csproj` 和 `TestRunner/TestRunner.csproj` 中引用了 SOLIDWORKS 安装目录下的互操作程序集：

```xml
<Reference Include="SolidWorks.Interop.sldworks">
  <HintPath>..\..\..\..\..\Program Files\SOLIDWORKS Corp\SOLIDWORKS\SolidWorks.Interop.sldworks.dll</HintPath>
  <EmbedInteropTypes>True</EmbedInteropTypes>
</Reference>
<Reference Include="SolidWorks.Interop.swconst">
  <HintPath>..\..\..\..\..\Program Files\SOLIDWORKS Corp\SOLIDWORKS\SolidWorks.Interop.swconst.dll</HintPath>
  <EmbedInteropTypes>True</EmbedInteropTypes>
</Reference>
<Reference Include="SolidWorks.Interop.swpublished">
  <HintPath>..\..\..\..\..\Program Files\SOLIDWORKS Corp\SOLIDWORKS\SolidWorks.Interop.swpublished.dll</HintPath>
  <EmbedInteropTypes>True</EmbedInteropTypes>
</Reference>
<Reference Include="solidworkstools, Version=2.0.0.0, ...">
  <HintPath>..\..\..\..\..\Program Files\SOLIDWORKS Corp\SOLIDWORKS\solidworkstools.dll</HintPath>
</Reference>
```

由于设置了 `EmbedInteropTypes = True`，编译时会把**当前机器上实际安装**的 SOLIDWORKS 版本对应的接口定义嵌入到生成的程序集中。

> **关键点**：这个相对路径（`..\..\..\..\..\Program Files\SOLIDWORKS Corp\SOLIDWORKS\`）是否生效，取决于你的工程目录和 SOLIDWORKS 安装目录的相对层级关系。
>
> 如果你的 SOLIDWORKS 2024 没有安装在这个相对路径能解析到的位置，需要手动修改 `HintPath` 指向你机器上实际的安装目录，例如：
>
> ```xml
> <HintPath>D:\Program Files\SOLIDWORKS Corp\SOLIDWORKS\SolidWorks.Interop.sldworks.dll</HintPath>
> ```

同时注意 `SW2URDF.csproj` 中调试启动程序的路径（用于 F5 调试时自动拉起 SOLIDWORKS）：

```xml
<StartProgram>C:\PROGRA~1\SOLIDW~1\SOLIDW~1\\SldWorks.exe</StartProgram>
```

如果 2024 版本安装路径不同，也要同步修改，否则不影响正常生成，只影响调试时自动启动。

### 3. 以管理员身份编译（COM 注册）

项目启用了"注册为 COM 互操作"（Register for COM Interop），生成时会自动调用 `regasm` 把插件注册到：

- `HKEY_CLASSES_ROOT\SW2URDF.SW.SwAddin`
- `HKEY_LOCAL_MACHINE\SOFTWARE\SolidWorks\Addins\{GUID}`

如果不以管理员身份运行 Visual Studio，生成时会报错：

```
无法注册程序集"...\SW2URDF.dll" - 拒绝访问。
请确保您正在以管理员身份运行应用程序。
```

**解决方法**：关闭 Visual Studio，右键选择"以管理员身份运行"，重新打开解决方案并生成。

### 4. 修复窗口"闪退"问题（Part 导出窗口一闪而过）

升级后测试零件导出功能时，点击菜单后导出窗口一闪就消失。通过日志（`%HOMEDRIVE%%HOMEPATH%\sw2urdf_logs\sw2urdf.log`）确认没有抛出任何异常，流程正常执行到显示窗口这一步。

定位到 `SW2URDF/SW/SwAddin.cs` 中 `SetupPartExporter()` 方法存在逻辑缺陷：

```csharp
// 原代码（有问题）
PartExportForm exportForm = new PartExportForm((SldWorks)SwApp);
logger.Info("Showing part");
exportForm.Show();       // Show() 是非模态、非阻塞调用
exportForm.Dispose();    // 紧接着立刻释放/销毁窗口！
```

`Form.Show()` 是非阻塞方法，显示窗口后代码会立刻往下执行，而下一行 `Dispose()` 立即销毁了刚显示的窗口，造成窗口一闪而过。

**修复方式**：改用模态的 `ShowDialog()`，阻塞直到用户关闭窗口后再释放资源：

```csharp
// 修复后
using (PartExportForm exportForm = new PartExportForm((SldWorks)SwApp))
{
	logger.Info("Showing part");
	exportForm.ShowDialog();
}
```

## 总结

| 问题 | 原因 | 解决方式 |
|---|---|---|
| 编译时 COM 注册失败（拒绝访问） | 未以管理员身份运行 Visual Studio | 以管理员身份重新打开 VS 并生成 |
| 构建时 DLL 被占用无法复制 | SOLIDWORKS 进程正在使用旧版 DLL | 生成前先关闭 SOLIDWORKS |
| 零件导出窗口一闪而过 | `Form.Show()` 后立即 `Dispose()`，导致窗口刚显示就被销毁 | 改为 `ShowDialog()`（模态显示），在窗口关闭后再释放资源 |
| Interop 引用路径找不到 | SOLIDWORKS 实际安装路径与 `HintPath` 相对路径不一致 | 按需修改 `.csproj` 中的 `HintPath` 为实际安装路径 |

整体来看，SW2URDF 插件的 COM 注册机制和核心逻辑并没有绑定具体的 SOLIDWORKS 版本号，从 2021 迁移到 2024 主要是**编译环境配置问题**（管理员权限、DLL 占用、Interop 引用路径）和**一个已存在的 UI 代码缺陷**（窗口 Show/Dispose 时序问题），并非需要大规模重写代码。

## 验证步骤

1. 关闭 SOLIDWORKS，以管理员身份打开 Visual Studio。
2. 确认 `.csproj` 中 SOLIDWORKS Interop 的 `HintPath` 能正确解析到本机 SOLIDWORKS 2024 安装目录。
3. 重新生成（Rebuild）整个解决方案，确认无编译错误。
4. 打开 SOLIDWORKS 2024，在"插件"管理窗口中勾选启用 SW2URDF。
5. 打开一个零件/装配体文件，测试导出功能是否正常弹窗、正常导出 URDF 文件。
