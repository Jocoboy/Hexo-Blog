---
title: 基于Wix和AutoUpdater的客户端打包与自动更新
date: 2024-08-19 18:46:44
categories:
- Package-Tool
tags:
- Wix Toolset
- .NET
- C#
---

使用Wix Toolset和AutoUpdater.NET实现Windows桌面应用程序的打包与自动更新功能。

<!--more-->

## Wix Toolset

Wix Toolset是用来打包Windows Installer的工具集，通过编译源代码，然后链接以创建可执行文件。WiX命令行构建工具适用于任何自动化构建系统，MSBuild还支持常见的CI/CD构建系统，如GitHub Actions。

使用WiX Bundle可以创建安装包来安装先决条件，例如.NET Framework和其他运行时环境以及自己的msi文件。WiX Bundle将他们组合成一个可下载的exe文件。

注: 以下使用的Wix Toolset版本为5.0，与低版本的语法有所区别。在Wix语法中，[]内容为库中预定义的全局变量(安装期间可使用)，$()为用户自己定义的全局变量，!(bind.)为绑定的属性，!(wix.)为Wix编译时的变量(安装期间不可使用)，!(loc.)为本地化之后的内容。

### Wix Package

Wix Package的基本功能是将应用程序打包成msi文件。此外，还支持桌面快捷方式、菜单快捷方式、卸载快捷方式添加，开机自启动，引导程序本地化等。

注: 需预先使用NuGet引入依赖包WixToolset.UI、WixToolset.Util

#### 快捷方式添加

Folders.wxs
```xml
<Wix xmlns="http://wixtoolset.org/schemas/v4/wxs">
	<Fragment>
		<!-- 定义桌面快捷方式目录 -->
		<StandardDirectory Id="DesktopFolder" />
		<!-- 定义菜单栏快捷方式目录 -->
		<StandardDirectory Id="ProgramMenuFolder" />
	</Fragment>
</Wix>
```

Package.wxs
```xml
...
<!--桌面快捷方式-->
<Component Id="DesktopShortcutComponent" Guid="{your-guid}">
    <Shortcut Id="DesktopShortcut" Name="$(var.ProductName)" Target="[INSTALLFOLDER]{your-product-name}.exe" Icon="{your-icon-name}" Directory="DesktopFolder" />
    
    <RegistryValue Root="HKCU" Key="Software\{your-company-name}\{your-product-name}" Name="Installed" Type="integer" Value="1" KeyPath="yes" />
</Component>

<!--菜单栏快捷方式-->
<Component Id="StartMenuShortcutComponent" Guid="{your-guid}">
    <Shortcut Id="StartMenuShortcut" Name="$(var.ProductName)" Target="[INSTALLFOLDER]{your-product-name}.exe" Icon="{your-icon-name}" Directory="ProgramMenuFolder" />
    
    <RegistryValue Root="HKCU" Key="Software\{your-company-name}\{your-product-name}" Name="Installed" Type="integer" Value="1" KeyPath="yes" />
</Component>

<!--卸载程序快捷方式-->
<Component Id="UninstallShortcutComponent" Guid="{your-guid}">
    <Shortcut Id="UninstallProduct" Name="Uninstall" Target="[SystemFolder]msiexec.exe" Arguments="/x [ProductCode]" Icon="UninstallIcon" Description="Uninstall {your-product-name}." />
   
    <RegistryValue Id="RegUninstallShortcut" Root="HKCU" Key="Software\{your-company-name}\{your-product-name}" Name="UninstallShortcut" Type="string" Value="" KeyPath="yes" />
</Component>
...
```

#### 快捷方式图标设置

Package.wxs
```xml
...
<!-- 引用应用程序图标 -->
<Icon Id="{your-icon-name}" SourceFile="$(var.{your-product-name}.ProjectDir)Resources\icon.ico" />
<!-- 引用卸载图标 -->
<Icon Id="UninstallIcon" SourceFile="$(var.{your-product-name}.ProjectDir)Resources\uninstall.ico" />
<!-- 控制面板引用应用程序图标 -->
<Property Id="ARPPRODUCTICON" Value="{your-icon-name}" />
...
```

#### 开机自启动

Package.wxs
```xml
...
<Component Id="Register">
    <RegistryKey ForceCreateOnInstall="yes" Id="AutoStartKey" Root="HKLM" Key="SOFTWARE\Microsoft\Windows\CurrentVersion\Run">
        <RegistryValue Id="AutoStartKeyValue" Name="$(var.ProductName)" KeyPath="yes" Type="string" Value="[INSTALLFOLDER]{your-product-name}.exe">
        </RegistryValue>
    </RegistryKey>
</Component>
...
```

#### 卸载或更改前自动关闭程序进程

Package.wxs
```xml
<!--卸载或更改前关闭程序进程-->
...
<Property Id="TASKKILL">
	<DirectorySearch Id="SysDir" Path="[SystemFolder]" Depth="1">
		<FileSearch Id="taskkillExe" Name="taskkill.exe" />
	</DirectorySearch>
</Property>

<util:CloseApplication Id="CloseApp" CloseMessage="yes" Target="{your-product-name}.exe" RebootPrompt="no"  PromptToContinue="yes"  Description="!(loc.CloseBeforeUninstall)"/>

<CustomAction Id="WixCloseApplications" Property="TASKKILL" Execute="immediate" Impersonate="yes" Return="ignore" ExeCommand="/F /FI &quot;IMAGENAME eq {your-product-name}.exe&quot;"/>
...
```

Package.zh-cn.wxl
```xml
<WixLocalization xmlns="http://wixtoolset.org/schemas/v4/wxl" Culture="zh-CN">
    ...
	<String Id="CloseBeforeUninstall" Value="请关闭应用程序后再进行卸载!"></String>
</WixLocalization>
```

#### x86和x64平台区分可执行文件源

Package.wxs
```xml
...
<Package Name="$(var.ProductName)" Manufacturer="$(var.Manufacturer)" Version="!(bind.FileVersion.ExeFile_m2)" UpgradeCode="$(var.UpgradeCode)">
    <MajorUpgrade DowngradeErrorMessage="!(loc.DowngradeError)" />

    <MediaTemplate EmbedCab="yes" />

    <Files Include="$({your-product-name}.TargetDir)\**">
        <Exclude Files="$({your-product-name}.TargetDir)\{your-product-name}.exe"></Exclude>
        <Exclude Files="$({your-product-name}.TargetDir)\{your-product-name}_x86.exe"></Exclude>
    </Files>

    <File Id="ExeFile_m2" Name="{your-product-name}.exe" Source="$({your-product-name}.TargetDir){your-product-name}_x86.exe" Condition="NOT VersionNT64"></File>
    <File Id="ExeFile_m1" Name="{your-product-name}.exe" Source="$({your-product-name}.TargetDir){your-product-name}.exe" Condition="VersionNT64"></File>
</Package>
...
```

### Wix Bundle

Wix Bundle主要用于设置系统必备组件，如各种依赖环境，可以根据目标平台区分依赖的安装包版本。此外还可引入主题文件，定制个性化安装界面。

注: 需预先使用NuGet引入依赖包WixToolset.BootstrapperApplications、WixToolset.Netfx、WixToolset.Util

#### 设置系统必备组件

以下以.NET Core 6.0运行时和WebEdge WebView2运行时为例

Bundle.wxs
```xml
...
<Chain>
	<!-- Define the .NET Core Runtime 6.0 dependency -->
	<PackageGroupRef Id="DESKTOPNETCORERUNTIME6_t"/>
	<!-- Define the Edge WebView2 Runtime dependency -->
	<PackageGroupRef Id="WV_bootstrapper"/>
	<!-- Other packages can be added here -->
	<MsiPackage Id="MainPackage" Compressed="yes" SourceFile= "$({your-product-name}.TargetDir)\zh-CN\{your-product-name}.msi" Visible="no">
		<MsiProperty Name="INSTALLFOLDER" Value="[InstallFolder]" />
	</MsiPackage>
</Chain>
...

<!-- Define the package group for .NET Core Runtime 6.0 -->
<Fragment>
	<netfx:DotNetCoreSearch
	 RuntimeType="desktop"
	 Platform="x86"
	 MajorVersion="6"
	 Variable="DESKTOPNETCORERUNTIME6_x86"/>
		
	<netfx:DotNetCoreSearch
	 RuntimeType="desktop"
	 Platform="x64"
	 MajorVersion="6"
	 Variable="DESKTOPNETCORERUNTIME6_x64"/>

	<WixVariable Id="DesktopNetCoreRuntime_6029_Redist_DetectCondition_x86" Value="DESKTOPNETCORERUNTIME6_x86" Overridable="yes" />
	<WixVariable Id="DesktopNetCoreRuntime_6029_Redist_DetectCondition_x64" Value="DESKTOPNETCORERUNTIME6_x64" Overridable="yes" />
	
	<PackageGroup Id="DESKTOPNETCORERUNTIME6_t">
		<ExePackage Id="NetCoreRuntime6_x86" Cache="remove" Permanent="yes" PerMachine="yes"
		 	DetectCondition="!(wix.DesktopNetCoreRuntime_6029_Redist_DetectCondition_x86)"
			InstallCondition="NOT VersionNT64"
			InstallArguments="/install /quiet"> <!-- 设置静默安装 -->
			<ExePackagePayload
				Name="DESKTOPNETCORERUNTIME6_x86"
				Size="{file-size}"
				DownloadUrl="{dowload-url}"
				Hash="{file-hash}"/>
		</ExePackage>

		<ExePackage Id="NetCoreRuntime6_x64" Cache="remove" Permanent="yes" PerMachine="yes"
		 	DetectCondition="!(wix.DesktopNetCoreRuntime_6029_Redist_DetectCondition_x64)"
			InstallCondition="VersionNT64"
			InstallArguments="/install /quiet">
			<ExePackagePayload
				Name="DESKTOPNETCORERUNTIME6_x64"
				Size="{file-size}"
				DownloadUrl="{dowload-url}"
				Hash="{file-hash}"/>
		</ExePackage>
	</PackageGroup>
</Fragment>
...
<!-- Define the package group for Edge WebView2 Runtime -->
<Fragment>
	<util:RegistrySearch Root="HKLM" Key="Software\Microsoft\EdgeUpdate\Clients\{F3017226-FE2A-4295-8BDF-00C3A9A7E4C5}" Value="pv" Variable="WVRTInstalled_x86"/>
	
	<util:RegistrySearch Root="HKLM" Key="Software\WOW6432Node\Microsoft\EdgeUpdate\Clients\{F3017226-FE2A-4295-8BDF-00C3A9A7E4C5}" Value="pv" Variable="WVRTInstalled_x64"/>
	
	<WixVariable Id="WV_DetectCondition_x86" Value="WVRTInstalled_x86" Overridable="yes" />
	<WixVariable Id="WV_DetectCondition_x64" Value="WVRTInstalled_x64" Overridable="yes" />

	<PackageGroup Id="WV_bootstrapper">
		<ExePackage
			Id="WV_bootstrapper_x86"
			PerMachine="yes"
			DetectCondition="!(wix.WV_DetectCondition_x86)"
			InstallCondition="NOT VersionNT64"
			Vital="yes"
			Permanent="yes"
			Cache="remove" CacheId="1">
			<ExePackagePayload
				Name="WV_bootstrapper_x86"
				DownloadUrl="{dowload-url}"
				Hash="{file-hash}"
				Size="{file-size}"  />
		</ExePackage>
		
		<ExePackage
			Id="WV_bootstrapper_x64"
			PerMachine="yes"
			DetectCondition="!(wix.WV_DetectCondition_x64)"
			InstallCondition="VersionNT64"
			Vital="yes"
			Permanent="yes"
			Cache="remove" CacheId="2"> 
			<ExePackagePayload
				Name="WV_bootstrapper_x64"
				DownloadUrl="{dowload-url}"
				Hash="{file-hash}"
				Size="{file-size}"  />
		</ExePackage>
	</PackageGroup>
</Fragment>
...
```

#### 定制个性化安装界面

注意: CustomTheme.xml定义了安装窗口、页面、按钮等各种UI元素以及样式、响应事件，CustomLocalize.wxl定义了个性化文本内容，具体可参照[Wix Toolset源码](https://github.com/wixtoolset/wix)

Bundle.wxs
```xml
...
<Bundle Name="$(var.ProductName)" Manufacturer="$(var.Manufacturer)"
        Version="!(bind.packageVersion.MainPackage)" UpgradeCode="$(var.UpgradeCode)"
        IconSourceFile="$(var.{your-product-name}.ProjectDir)Resources\{your-company-name}.ico">
    <BootstrapperApplication>
        <bal:WixStandardBootstrapperApplication LicenseUrl="" Theme="rtfLargeLicense" ShowVersion="true"
        LogoFile ="$(var.{your-product-name}.ProjectDir)Resources\{your-company-name}.ico"
        ThemeFile="CustomTheme.xml" LocalizationFile="CustomLocalize.wxl"/>
    </BootstrapperApplication>
</Bundle>
...
```

## AutoUpdater.NET

AutoUpdater.NET是一个类库，可让.NET开发人员轻松地将自动更新功能添加到桌面应用程序项目中。AutoUpdater.NET会从服务器下载包含更新信息的XML文件。它使用该XML文件获取软件最新版本的信息。如果软件的最新版本大于用户PC上安装的软件的当前版本，AutoUpdater.NET就会向用户显示更新对话框。如果用户按下更新按钮更新软件，它就会从XML文件中提供的URL下载更新文件（安装程序），并执行刚刚下载的安装程序文件。此后，安装程序的工作就是执行更新。如果您提供的是zip文件URL而不是安装程序，AutoUpdater.NET将把zip文件的内容解压缩到应用程序目录。

### 配置XML

```xml
<?xml version="1.0" encoding="UTF-8"?>
<item>
  <version>{latest-version}</version>
  <url>{your-download-url}</url>
  <changelog>{your-changelog-url}</changelog>
  <mandatory>false</mandatory>
</item>
```

### 启动更新检查

```c#
AutoUpdater.Start("{your-xml-url}");
```

### 手动处理更新

```c#
AutoUpdater.CheckForUpdateEvent += AutoUpdaterOnCheckForUpdateEvent;

private  async void AutoUpdaterOnCheckForUpdateEvent(UpdateInfoEventArgs args)
{
    if (args.IsUpdateAvailable)
    {
        using var httpClient = new HttpClient();
        try
        {
            string log = await httpClient.GetStringAsync(args.ChangelogURL);

            var textContent = $"当前版本{args.InstalledVersion}, 最新版本{args.CurrentVersion}\n" +
                              "有新版本可用，是否立即更新？\n" + $"{log}";

            MessageBoxResult result = MessageBox.Show(textContent, "更新提示", MessageBoxButton.YesNo, MessageBoxImage.Information);

            if (result == MessageBoxResult.Yes)
            {
                Close();
                AutoUpdater.DownloadUpdate(args);
            }
        }
        catch (HttpRequestException e)
        {
            Console.WriteLine(e.Message);
        }
    }
}
```

## 参考文档

- [Wix Toolset官方文档](https://wixtoolset.org/docs/intro/)

- [Wix Toolset开源项目地址](https://github.com/wixtoolset/wix)

- [AutoUpdater.NET开源项目地址](https://github.com/ravibpatel/AutoUpdater.NET)

- [Windows Installer在安装期间使用的全局变量](https://learn.microsoft.com/zh-cn/windows/win32/msi/properties)
