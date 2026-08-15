---
name: wpf-project-dev
description: "WPF 项目开发规范技能（MVVM + WPFTemplateLib）：从零创建 WPF 项目、将 Python/其他语言项目移植为 C#/WPF、为已有 WPF 项目补充 ViewModel/View/样式/依赖。遵循 MVVM 模式、NuGet 引入第三方库、SimpleBindableBase 通知属性、库默认样式等最佳实践。"
license: MulanPSL-2.0
---
# WPF 项目开发规范（MVVM + WPFTemplateLib）

本技能沉淀自 GitPatchEditorWpf 实战项目（Git Patch 编辑工具，由 Python/tkinter 移植为 C#/WPF）。目标是让 WPF 项目开发保持一致的架构与最佳实践，避免踩已知的坑。

本技能初版由 Marvis 创建，独立观察员（dlgcy）进行审核和后续修改完善。


重要地址：

* WPFTemplateLib 库地址（最新版本以这里显示的为准）：[https://www.nuget.org/packages/WPFTemplateLib/](https://www.nuget.org/packages/WPFTemplateLib/)
* ReadMe 文档：[https://gitee.com/dlgcy/WPFTemplateLib/blob/master/ReadMe.md](https://gitee.com/dlgcy/WPFTemplateLib/blob/master/ReadMe.md)
* SkillHub 地址（本技能的发布更新地址）：[https://skillhub.cn/skills/user_95bc37fb/wpf-project-dev](https://skillhub.cn/skills/user_95bc37fb/wpf-project-dev)


## 适用场景

- 从零创建 WPF 项目
- 将 Python / 其他语言项目移植为 WPF（MVVM）
- 为已有 WPF 项目补充 ViewModel / View / 样式 / 依赖


## 一、项目骨架与目录结构

推荐结构（slnx 放外层目录，项目目录扁平化，**不要嵌套一层子目录**）：

```
E:\SomeProject\
├── SomeProject.slnx                     解决方案（SDK 风格 .slnx）
└── SomeProject\                         项目目录，直接包含项目文件
    ├── SomeProject.csproj
    ├── App.xaml / App.xaml.cs           应用入口（StartupUri=MainWindow.xaml）
    ├── MainWindow.xaml / MainWindow.xaml.cs
    ├── FodyWeavers.xml                  PropertyChanged.Fody 织入配置
    ├── Models\                          数据模型
    ├── Services\                        业务逻辑（解析/生成/IO）
    ├── ViewModels\                      MVVM 视图模型
    └── Views\                           其他 View（多窗口/UserControl 时）
```

- **slnx 路径解析**：slnx 内的项目路径按「相对 slnx 所在目录」解析。移动/改名 slnx 后必须同步修正项目路径（如 `SomeProject/SomeProject.csproj` 移出后要加目录层级）。
- MainWindow.xaml.cs 只负责设置 DataContext（或依赖注入），不写业务逻辑。


## 二、csproj 与 NuGet 依赖

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net472</TargetFramework>
    <UseWPF>true</UseWPF>
    <RootNamespace>SomeProject</RootNamespace>
    <Nullable>enable</Nullable>
    <LangVersion>latest</LangVersion>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="WPFTemplateLib" Version="7.3.3" />
    <PackageReference Include="PropertyChanged.Fody" Version="4.1.0" PrivateAssets="all" />
  </ItemGroup>
</Project>
```


**关键决策**：

- **WPFTemplateLib**（nuget.org 官方源，作者 dlgcy，最新 7.3.3）：提供 MVVM 基础类、样式、转换器、用户控件、附加属性等，优先以 NuGet 包形式引入，不要复制源码。
- **PropertyChanged.Fody**：用于自动属性通知织入，必须加 `PrivateAssets="all"` 消除 FodyPackageReference 警告。
- 第三方库一律优先 NuGet 包形式（用户偏好）。


## 三、FodyWeavers.xml

项目根新增（参照 WPFTemplateLib 自带写法）：

```xml
<Weavers xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="FodyWeavers.xsd">
  <PropertyChanged />
</Weavers>
```



## 四、ViewModel 规范

### 4.1 基类选择

- **`SimpleBindableBase`**：轻量，仅提供属性通知（INotifyPropertyChanged + INotifyPropertyChanging，含 OnPropertyChanged / SetProperty），纯数据项类与普通 VM 都适用。
- `ObservableObject`：SimpleBindableBase 的基类，同样可用（注意包内 `WpfHelpers.BindableBase` 已标记过时，勿用）。
- **`ViewModelBase`**：重量级（额外携带 Dispatcher、Info、Toast、LoadedCmd、INotifyDataErrorInfo 数据校验），复杂功能的 ViewModel 继承使用。
- 也可以使用 `[AddINotifyPropertyChangedInterface]` 特性方式，但继承 SimpleBindableBase 更干净、可省略特性。


### 4.2 常用规范

* 使用 region 给各部分分区，大致可分为如下几部分：成员&构造、属性、命令、方法。


### 4.3 属性写法

Fody 织入对继承基类的派生类同样生效，属性直接写**自动属性**，无需 SetProperty：

```csharp
using WPFTemplateLib.Mvvm;
using WPFTemplateLib.WpfHelpers;

public class MainViewModel : SimpleBindableBase
{
    public string InputPath { get; set; } = "";
    public bool IsSelected { get; set; } = true;
}
```


集合属性用 `ObservableCollection<T>`（如 `public ObservableCollection<PatchFileGroupViewModel> Files { get; } = new();`）。


如果某些情况下需要完整属性，可按照如下写法：

```C#
private string _Name;
/// <summary>
/// 注释
/// </summary>
public string Name { get => _Name; set => SetProperty(ref _Name, value); }
```


需要处理属性变动逻辑时，也可以不用完整属性，使用如下方法即可：

```C#
/// <inheritdoc />
protected override void PropertyChangedHandle(object sender, PropertyChangedEventArgs e)
{
	base.PropertyChangedHandle(sender, e);
	switch(e.PropertyName)
	{
		case nameof(InputPath):
		{
			break;
		}
	}
}
```


### 4.4 命令写法

命令：`WPFTemplateLib.WpfHelpers.RelayCommand`（构造参数：canExecute + execute）。


命令常规写法：

```C#
public ICommand BrowseInputCommand { get; }
public ICommand LoadPatchCommand { get; }

public MainViewModel()
{
    BrowseInputCommand = new RelayCommand(_ => true, _ => BrowseInput());
    LoadPatchCommand = new RelayCommand(_ => true, _ => LoadPatch());
}

private void BrowseInput()
{
    // 业务逻辑，不要写在 View 里
}
```


命令的推荐写法：

```C#
#region [命令] 切换
private RelayCommand _SwitchCmd;
public RelayCommand SwitchCmd => _SwitchCmd ??= new RelayCommand(ExecuteSwitchCmd);
private void ExecuteSwitchCmd()
{
}
#endregion

#region [命令] 注释
private RelayCommand<object> _MyCmd;
/// <summary>
/// [命令] 注释
/// </summary>
public RelayCommand<object> MyCmd => _MyCmd ??= new (ExecuteMyCmd);
private void ExecuteMyCmd(object para)
{
}
#endregion
```


## 五、View 规范

### 5.1 设计时 DataContext 导航（必须）

每个 View 的根元素添加（用于从 View 跳转到 ViewModel，设计器可见绑定）：

```xml
<Window x:Class="SomeProject.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
        xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
        xmlns:vm="clr-namespace:SomeProject.ViewModels"
        mc:Ignorable="d" d:DataContext="{d:DesignInstance Type=vm:MainViewModel}">
```


**注意**：`xmlns:vm` 必须映射到**项目实际命名空间**（如 `clr-namespace:GitPatchEditorWpf.ViewModels`），禁止照抄 `clr-namespace:ViewModels`。


### 5.2 绑定与样式使用

- 按钮默认使用库默认样式（DefaultThemeDictionary 的隐式样式），**不要显式指定按钮样式**（用户偏好）。
- 常用显式样式键（来自 StyleDictionary.xaml）：
  - `LibSty.TextBox.Enhance`：增强文本框
  - `LibSty.Border.Shadow`：带阴影的 Border（卡片效果）
  - `LibSty.WpfUi.Button.Ele.Primary/Default/Success/Info/Warning/Danger`：彩色按钮（需显式指定才用）
- 可读文本展示用 `TextBox IsReadOnly="True"` + `FontFamily="Consolas"`。


### 5.3 界面代码（Xaml）编写规范

* 使用 Grid 布局时，内部元素除了第一个可不指定行列，其余元素都需要显式指定所在的行和列。


### 5.4 WPF 特有内容书写规范

依赖属性：

```C#
#region [依赖属性] 注释
/// <summary>
/// [依赖属性] 注释
/// </summary>
public bool IsShow
{
	get => (bool)GetValue(IsShowProperty);
	set => SetValue(IsShowProperty, value);
}
public static readonly DependencyProperty IsShowProperty =
	DependencyProperty.Register(nameof(IsShow), typeof(bool), typeof(Owner), new PropertyMetadata(false));
#endregion
```


依赖属性（带变动方法）：

```C#
#region [依赖属性] 注释
/// <summary>
/// 注释
/// </summary>
public bool IsShow
{
	get => (bool)GetValue(IsShowProperty);
	set => SetValue(IsShowProperty, value);
}
/// <summary>
/// [依赖属性] 注释
/// </summary>
public static readonly DependencyProperty IsShowProperty =
	 DependencyProperty.Register(nameof(IsShow), typeof(bool), typeof(Owner), new PropertyMetadata(false, IsShowChangedCallback));
/// <summary>
/// 注释 变动处理方法
/// </summary>
private static void IsShowChangedCallback(DependencyObject d, DependencyPropertyChangedEventArgs e)
{
}
#endregion
```


依赖属性（只读）：

```C#
#region [依赖属性(只读)] 注释
/// <summary>
/// [依赖属性(只读)] 注释
/// </summary>
public bool IsShow
{
	get => (bool)GetValue(IsShowProperty);
	set => SetValue(IsShowPropertyKey, value);
}
public static readonly DependencyPropertyKey IsShowPropertyKey =
	DependencyProperty.RegisterReadOnly(nameof(IsShow), typeof(bool), typeof(Owner), new PropertyMetadata(false));
private static readonly DependencyProperty IsShowProperty = IsShowPropertyKey.DependencyProperty;
#endregion
```


附加属性：

```C#
#region [附加属性] 注释
public static int GetMyProperty(DependencyObject obj)
{
	return (int)obj.GetValue(MyPropertyProperty);
}
/// <summary>
/// 设置 注释
/// </summary>
public static void SetMyProperty(DependencyObject obj, int value)
{
	obj.SetValue(MyPropertyProperty, value);
}
/// <summary>
/// [附加属性] 注释
/// </summary>
public static readonly DependencyProperty MyPropertyProperty =
	DependencyProperty.RegisterAttached("MyProperty", typeof(int), typeof(ownerclass), new PropertyMetadata(0));
#endregion
```


附加属性（带变动方法）：

```C#
#region [附加属性] 注释
public static int GetMyProperty(DependencyObject obj)
{
	return (int)obj.GetValue(MyPropertyProperty);
}
/// <summary>
/// 设置 注释
/// </summary>
public static void SetMyProperty(DependencyObject obj, int value)
{
	obj.SetValue(MyPropertyProperty, value);
}
/// <summary>
/// [附加属性] 注释
/// </summary>
public static readonly DependencyProperty MyPropertyProperty =
	DependencyProperty.RegisterAttached("MyProperty", typeof(int), typeof(ownerclass), new PropertyMetadata(0, MyPropertyChangedCallback));
/// <summary>
/// 注释 变化处理方法
/// </summary>
private static void MyPropertyChangedCallback(DependencyObject d, DependencyPropertyChangedEventArgs e)
{
}
#endregion
```


路由事件：

```C#
#region [路由事件] 注释
public static readonly RoutedEvent TapEvent = EventManager.RegisterRoutedEvent("Tap", RoutingStrategy.Bubble, typeof(RoutedEventHandler), typeof(OwnerType));
/// <summary>
/// [路由事件] 注释
/// </summary>
public event RoutedEventHandler Tap { add => AddHandler(TapEvent, value); remove => RemoveHandler(TapEvent, value); }
/// <summary>
/// 路由事件 注释 的触发方法
/// </summary>
/// <param name="originalSource">此参数会传递到事件参数的 OriginalSource 属性中</param>
private void RaiseTapEvent(object originalSource = null)
{
	RaiseEvent(new RoutedEventArgs(TapEvent, originalSource));
} 
#endregion
```


附加事件：

```C#
#region [附加事件] 注释
/// <summary>
/// [附加事件] 注释
/// </summary>
public static readonly RoutedEvent ValueChangedEvent = EventManager.RegisterRoutedEvent("ValueChanged", RoutingStrategy.Bubble, typeof(RoutedEventHandler), typeof(OwnerClass));
/// <summary>
/// 添加 [附加事件] 注释 [处理方法]
/// </summary>
public static void AddValueChangedHandler(DependencyObject dependencyObject, RoutedEventHandler handler)
{
	if(dependencyObject is not UIElement uiElement)
		return;
	uiElement.AddHandler(ValueChangedEvent, handler);
}
public static void RemoveValueChangedHandler(DependencyObject dependencyObject, RoutedEventHandler handler)
{
	if(dependencyObject is not UIElement uiElement)
		return;
	uiElement.RemoveHandler(ValueChangedEvent, handler);
}
#endregion
```


## 六、样式资源引入（App.xaml）

```xml
<Application x:Class="SomeProject.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             StartupUri="MainWindow.xaml">
    <Application.Resources>
        <ResourceDictionary>
            <ResourceDictionary.MergedDictionaries>
                <!-- 基础样式字典（LibSty.* 样式键） -->
                <ResourceDictionary Source="pack://application:,,,/WPFTemplateLib;component/Styles/StyleDictionary.xaml"/>
                <!-- 默认主题（13 个无 Key 隐式样式：Button/CheckBox/ComboBox/DataGrid/Expander/GroupBox/RadioButton/ScrollBar/ScrollViewer/TextBlock/ToggleButton/TreeView/TreeViewItem），不引入则无默认样式 -->
                <ResourceDictionary Source="pack://application:,,,/WPFTemplateLib;component/Styles/DefaultThemeDictionary.xaml"/>
                <!-- 可选：颜色主题（默认蓝色），绿色：.../Themes/Light.Green.xaml -->
            </ResourceDictionary.MergedDictionaries>
        </ResourceDictionary>
    </Application.Resources>
</Application>
```

命名约定：样式以 `LibSty` 开头，控件模板以 `LibTpl` 开头；WPF 系统样式 `SysSty` / `SysTpl`。

库 ReadMe：https://gitee.com/dlgcy/WPFTemplateLib 。


## 七、跨语言移植要点（Python → WPF）

1. **先通读原项目全部源码**，理解功能逻辑（界面 / 核心算法 / 文件处理），再按 MVVM 分层实现。
2. **核心解析/生成逻辑保持字节级等价**：如文本解析要逐行保留原始行尾（`\r\n` / `\n`），读写用 UTF-8 无 BOM（`UTF8Encoding(false)`）。
3. 原 GUI 的交互控件映射：文件对话框 → `Microsoft.Win32.OpenFileDialog/SaveFileDialog`；列表勾选 → ItemsControl + CheckBox 绑定；提示框 → MessageBox。
4. 完成后用临时验证程序对输入输出做字节级比对，确认无损。


## 八、编译与验证流程

```powershell
dotnet build <解决方案>.slnx -c Debug
```

- 目标：**0 警告 0 错误**。
- Fody 织入验证：反射检查生成的 exe，确认 VM 类继承 SimpleBindableBase 且实现 INotifyPropertyChanged。
- 运行时验证：启动 exe 确认窗口正常显示、无 MissingResource / XamlParseException。


## 九、常见坑（务必规避）

| 坑                                    | 现象                                                                                     | 解决                                                |
| ------------------------------------- | ---------------------------------------------------------------------------------------- | --------------------------------------------------- |
| slnx 移动/改名后项目路径未修正        | MSB3202 找不到项目文件                                                                   | 按 slnx 所在目录重新解析相对路径                    |
| write_file 写 App.xaml 自动生成副本   | 两个同名 `x:Class`，App.g.cs 未生成 LoadComponent，启动异常退出（ExitCode=-532462766） | 删除 obj 残留副本产物后全量重建（--no-incremental） |
| 显式按钮样式                          | 用户要求"用库默认样式"                                                                   | 按钮不写 Style，由 DefaultThemeDictionary 隐式接管  |
| 照抄 `clr-namespace:ViewModels`     | 设计时绑定失效                                                                           | 用项目实际命名空间                                  |
| PropertyChanged.Fody 无 PrivateAssets | FodyPackageReference 警告                                                                | 加 `PrivateAssets="all"`                          |
| 包内 BindableBase                     | CS0618 过时警告                                                                          | 用 SimpleBindableBase / ObservableObject            |



## 十、验证清单（完成一项打勾）

- [ ] 目录扁平化：项目文件直接在项目根，slnx 在外层
- [ ] csproj：net472 或 net60 + UseWPF + WPFTemplateLib + PropertyChanged.Fody(PrivateAssets=all)
- [ ] FodyWeavers.xml 已建
- [ ] ViewModel 继承 SimpleBindableBase 或 ViewModelBase，使用自动属性
- [ ] 每个 View 根元素有 d:DataContext 设计时声明，vm 命名空间为实际命名空间
- [ ] App.xaml 合并 StyleDictionary + DefaultThemeDictionary
- [ ] 按钮默认不显式指定样式
- [ ] dotnet build 0 警告 0 错误
- [ ] 启动无资源异常，反射确认 INotifyPropertyChanged 织入
