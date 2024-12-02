---
title: 基于.NET的MSBuild常用命令
date: 2024-12-02 16:01:11
categories:
- Package-Tool
tags:
- MSBuild
---

基于.NET的MSBuild常用命令记录。

<!--more-->

## 前言

Microsoft生成引擎是一个用于生成应用程序的平台。此引擎（也称为 MSBuild）为项目文件提供了一个XML架构，用于控制生成平台处理和生成软件的方式。Visual Studio会使用MSBuild，但MSBuild不依赖于Visual Studio。通过在项目或解决方案文件中调用msbuild.exe或dotnet build，可以在未安装Visual Studio的环境中安排和生成产品。

## 环境设置

以VS 2022为例，MSBuild可执行文件路径为`C:\Program Files\Microsoft Visual Studio\2022\Community\MSBuild\Current\Bin`

## 项目配置文件

```xml
<!--回车符\r &#xD; 换行符\n &#xA; 双引号" &quot;-->

<Target Name="PostBuild" AfterTargets="PostBuildEvent">
    <Exec Command="if '$(IS_Integrated)' == 'true' (&#xD;&#xA;  xcopy $(TargetPath) $(SolutionDir)..\Outputs\  /R /Y&#xD;&#xA;  xcopy $(TargetDir)$(TargetName).dll $(SolutionDir)..\Outputs\  /R /Y&#xD;&#xA;)" />
</Target>

<Target Name="CopyDirectory" AfterTargets="Build">
    <PropertyGroup>
        <CopyCommand Condition="'$(OS)' == 'Windows_NT'">xcopy &quot;$(SolutionDir)..\Outputs\*.dll&quot; &quot;$(ProjectDir)bin\$(Configuration)\netcoreapp3.1\&quot; /E /D /I /Y
        </CopyCommand>
        <CopyCommand Condition="'$(OS)' != 'Windows_NT'">cp -r &quot;$(SolutionDir)..\Outputs\*.dll&quot; &quot;$(ProjectDir)bin\$(Configuration)\netcoreapp3.1\&quot;
        </CopyCommand>
</PropertyGroup>
    <Exec Command="$(CopyCommand)" />
</Target>

<Target Name="CreateFolderIfNotExists" AfterTargets="Build">
    <PropertyGroup>
        <uploadPath>$(ProjectDir)bin\$(Configuration)\upload</uploadPath>
    </PropertyGroup>
    <PropertyGroup>
        <CreateCommand Condition="'$(OS)' == 'Windows_NT'">if not exist &quot;$(uploadPath)&quot; md &quot;$(uploadPath)&quot;</CreateCommand>
        <CreateCommand Condition="'$(OS)'!= 'Windows_NT'">if [! -d &quot;$(uploadPath)&quot; ]; then mkdir -p &quot;$(uploadPath)&quot;; fi</CreateCommand>
    </PropertyGroup>
    <Exec Command="$(CreateCommand)" />
</Target>
```

## 命令行构建

对于单个dotnet项目，可使用以下命令构建

```shell
# 项目文件携带环境变量的情况
MSBuild [project file] /p:PropertyName=Value
# 举例
MSBuild test.csproj /p:IS_Integrated = true
```

## 批处理构建

对于模块较多的dotnet项目，可使用批处理脚本批量构建，并跟踪构建情况

```bat
@echo off
set totalCount=14
set failCount=0
set failList=""
chcp 65001

cd your/test.sln/path
echo 正在构建test模块[进度 1/%totalCount%]...
dotnet build test.sln  > nul 2>&1
if %errorlevel% neq 0 (
    echo test模块构建失败
    set /a failCount=failCount + 1
    set failList="%failList% test"
) else (
    echo test模块已构建成功！
)

set /a successCount=%totalCount% - %failCount%
set /p DUMMY=构建成功%successCount%个，失败%failCount%个，失败的模块有[%failList%]，请尝试手动构建，按回车键继续...
```

## 参考文档

[用于MSBuild命令和属性的常用宏](https://learn.microsoft.com/zh-cn/cpp/build/reference/common-macros-for-build-commands-and-properties)

[MSBuild 属性说明](https://learn.microsoft.com/zh-cn/visualstudio/msbuild/msbuild-properties)

[xcopy命令参数说明](https://learn.microsoft.com/zh-cn/windows-server/administration/windows-commands/xcopy)