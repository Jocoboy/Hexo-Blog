---
title: Web Service远程调用技术
date: 2024-08-18 14:15:36
categories:
- Remote-Procedure-Call
tags:
- Web Service
- .NET Framework
---

Web Service远程调用技术(RPC)的基本概念及实现方式。

<!--more-->

## 前言

Web Service即web服务，是一种跨编程语言和跨操作系统平台的远程调用技术。Web服务包含了一套标准,例如HTTP、XML、SOAP、WSDL、UDDI等，定义了应用程序如何在Web上实现互操作，可以在任何支持这些标准的平台（如Windows、Linux）中使用。

Web Service与Web API相比，更加适合端到端的应用场景(C/S架构)，适合作为内部服务使用。

## 基本概念

### SOAP

SOAP即简单对象访问协议(Simple Object Access Protocol)，是Web Service的通信协议，基于XML文件并绑定在HTTP协议上传递。SOAP消息包括Envelope、Header和Body元素。

一条SOAP消息就是一个普通的XML文档，文档包括下列元素：

- Envelope元素，必选，可把此XML文档标识为一条SOAP消息
- Header元素，可选，包含头部信息
- Body元素，必选，包含所有的调用和响应信息

### WSDL

Web Service描述语言(SebService Definition Language，简称WSDL)就是用机器能阅读的方式提供的一个正式描述文档而基于XML的语言，用于描述Web Service及其函数、参数和返回值。

在WSDL说明书中，描述了

- 对外发布的服务名称（类）
- 接口方法名称（方法）
- 接口参数（方法参数）
- ​服务返回的数据类型（方法返回值）

### UDDI

UDDI(Universal Description，Discovery and Integration)，也就是通用的描述、发现以及整合，是一套基于Web的、分布式的、为WebService提供的、信息注册中心的实现标准规范。用户可以通过UDDI来注册和搜索Web服务。

## SOA架构

面向服务架构(Service Oriented Architecture，简称SOA)，是一个组件模型，它将应用程序的不同功能单元(服务)通过预先定义的接口和契约联系起来。接口是采用中立的方式进行定义的，独立于实现服务的硬件平台、操作系统和编程语言，构建在系统中的服务以一种统一和通用的方式进行交互。

## 实现方式

注: 以下项目基于.NET Framework，.NET中无法使用

### 创建服务

使用ASP.NET Web应用程序(.NET Framework)创建一个Web服务(asmx文件)

```c#
using System.Web.Services;

namespace WebApplicationDemo
{
    [WebService(Namespace = "http://tempuri.org/")] // 定义命名空间
    [WebServiceBinding(ConformsTo = WsiProfiles.BasicProfile1_1)] // 绑定规范
    public class WebServiceTest : WebService
    {
        [WebMethod(Description = "测试方法")]
        public int Sum(int a, int b)
        {
            return a + b;
        }
    }
}
```

## 调用服务

### 静态引用

根据提供的Web Service地址，通过Connected Services添加WCF Web服务引用，生成cs文件，然后直接调用。

```c#
static void Main(string[] args)
{
    WebServiceTestSoapClient client = new WebServiceTestSoapClient();

    var res = client.SumAsync(1, 2);

    Console.WriteLine(res.Result);
}
```

### 反射调用

将Web Service地址存放到配置文件中，通过读取地址生成代理类，动态在项目中生成代理类文件，然后通过反射调用。

```c#
static void Main(string[] args)
{
    int res = (int)WebServiceProxy.InvokeWebService("https://localhost:44319/WebServiceTest.asmx", "WebServiceTest", "Sum", new object[] { 1, 2 });

    Console.WriteLine(res);
}
```

```c#
 /// <summary>
 /// 反射代理类
 /// </summary>
 public class WebServiceProxy
 {
     public static object InvokeWebService(string url,string ns, string methodname, object[] args)
     {
         try
         {
             //获取WSDL
             WebClient wc = new WebClient();
             Stream stream = wc.OpenRead(url + "?WSDL");
             ServiceDescription sd = ServiceDescription.Read(stream);
             string classname = sd.Services[0].Name;
             ServiceDescriptionImporter sdi = new ServiceDescriptionImporter();
             sdi.AddServiceDescription(sd, "", "");
             CodeNamespace cn = new CodeNamespace(ns);

             //生成客户端代理类代码
             CodeCompileUnit ccu = new CodeCompileUnit();
             ccu.Namespaces.Add(cn);
             sdi.Import(cn, ccu);
             CSharpCodeProvider csc = new CSharpCodeProvider();
             ICodeCompiler icc = csc.CreateCompiler();

             //设定编译参数
             CompilerParameters cplist = new CompilerParameters();
             cplist.GenerateExecutable = false;
             cplist.GenerateInMemory = true;
             cplist.ReferencedAssemblies.Add("System.dll");
             cplist.ReferencedAssemblies.Add("System.XML.dll");
             cplist.ReferencedAssemblies.Add("System.Web.Services.dll");
             cplist.ReferencedAssemblies.Add("System.Data.dll");

             //编译代理类
             CompilerResults cr = icc.CompileAssemblyFromDom(cplist, ccu);
             if (true == cr.Errors.HasErrors)
             {
                 System.Text.StringBuilder sb = new System.Text.StringBuilder();
                 foreach (System.CodeDom.Compiler.CompilerError ce in cr.Errors)
                 {
                     sb.Append(ce.ToString());
                     sb.Append(System.Environment.NewLine);
                 }
                 throw new Exception(sb.ToString());
             }

             //生成代理实例，并调用方法
             System.Reflection.Assembly assembly = cr.CompiledAssembly;
             Type t = assembly.GetType(ns + "." + classname, true, true);
             object obj = Activator.CreateInstance(t);
             System.Reflection.MethodInfo mi = t.GetMethod(methodname);

             return mi.Invoke(obj, args);
         }
         catch
         {
             return null;
         }
     }
 }
```


## 参考文档

- [.NET Framework API参考文档](https://learn.microsoft.com/zh-cn/dotnet/api/?view=netframework-4.8.1)