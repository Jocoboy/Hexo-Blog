---
title: HttpClient使用详解
date: 2025-03-04 17:03:08
categories:
- Network-Protocol
tags:
- HttpClient
- .NET
- ASP.NET Core
---

使用HttpClient发送HTTP请求的方式介绍。

<!--more-->

## 前言

在c#中常见发送http请求的方式有以下三种：

- HttpWebRequest: .NET平台原生提供，这是.NET创建者最初开发用于使用HTTP请求的标准类。使用HttpWebRequest可以让开发者控制请求/响应流程的各个方面，如timeouts,cookies,headers,protocols
- WebClient: .NET平台原生提供，WebClient是一种更高级别的抽象，是HttpWebRequest为了简化最常见任务而创建的，但也因此缺少了HttpWebRequest的灵活性
- HttpClient: .NET平台原生提供，目前主流使用的Http请求工具类，直接基于Socket开发，提供了异步友好的代码编写方式，是线程安全的

## HTTP Content-Type

在HTTP协议中，客户端和服务器之间通过请求和响应进行通信。在这个过程中，传输的数据有各种不同的格式类型，为了确保双方能够正确理解和处理数据，需要正确设置Content-Type。Content-Type是指示发送端内容的媒体类型的HTTP头部，广泛用于请求和响应中。

媒体类型(Media Type)，也称为MIME类型(Multipurpose Internet Mail Extensions)，指定了内容的格式和编码方式。常见的Content-Type有以下几种：

- text/plain: 表示内容是纯文本，不包含格式化信息。通常用于简单的文本内容，没有特殊的意义标记
- text/html: 表示该内容是HTML文档
- text/xml: 表示内容是XML文档，可用于soap1.1协议。在text/xml中，XML头指定的编码格式无效，必须在HTTP头部的Content-Type中指定才会生效
- application/soap+xml：表示内容是XML文档，可用于soap1.2协议
- application/xml: 表示内容是XML文档，可直接在XML头指定编码格式，是常规XML格式的首选类型
- application/octet-stream: 表示二进制流数据，可在下载文件类型未知的情况下使用
- application/json: 表示内容是JSON格式。JSON是一种轻量级的数据交换格式，广泛用于API的请求和响应中
- application/x-www-form-urlencoded: 表示数据以键值对形式进行编码，是HTML表单默认的Content-Type类型
- multipart/form-data: 通常用于表单数据中包含文件上传的情景。在这种类型中，数据被拆分成多个部分，每个部分包含自己独立的头部信息

## HttpClient使用配置

当我们发送http请求时，可以使用SocketsHttpHandler处理一些事情，比如是否自动处理cookie，是否自动重定向以及最多重定向几次等。以下展示了常见的配置项：

```c#
// 在.net core 2.1之后，默认所有的http请求都会交给SocketsHttpHandler处理
var socketsHttpHandler = new SocketsHttpHandler()
{
    UseCookies = false, //是否自动处理cookie

    AllowAutoRedirect = true,//是否自动重定向, 默认true
    MaxAutomaticRedirections = 50,//自动重定向的最大次数, 默认50

    MaxConnectionsPerServer = 100, //每个请求连接的最大数量, 默认是int.MaxValue
    PooledConnectionIdleTimeout = TimeSpan.FromMinutes(2), //连接池中TCP连接最多可以闲置多久, 默认2分钟
    PooledConnectionLifetime = Timeout.InfiniteTimeSpan, //连接最长的存活时间, 默认是不限制的

    AutomaticDecompression = DecompressionMethods.GZip, //是否压缩，默认是None，即不压缩

   ConnectTimeout = Timeout.InfiniteTimeSpan,  //建立TCP连接时的超时时间, 默认不限制
   Expect100ContinueTimeout = TimeSpan.FromSeconds(1), //等待服务返回statusCode=100的超时时间, 默认1秒

   MaxResponseHeadersLength = 64, //响应头大小限制，单位kb
};

using var httpClient = new HttpClient(socketsHttpHandler);

```

此外，可以给HttpClient设置基地址，当HttpClient发送的请求不包含前缀时，将自动拼接上，否则不予拼接。还可以设置默认的http版本、请求头。

```c#
httpClient.BaseAddress = new Uri($"{protocol}://{ip}:{port}/");
httpClient.DefaultRequestVersion = HttpVersion.Version20;
httpClient.DefaultRequestHeaders.Add("parameter1", "value1");
```

## HttpClient请求方法

### GET

GET请求不需要设置Content-Type，因为GET请求的参数是通过URL传递的，而不是在请求体中。通过GET请求数据的示例如下：

```c#
using var httpClient = new HttpClient();
var url = $"{protocol}://{ip}:{port}/{api}?{parameter1}={value1}&{parameter2}={value2}";
var response = await httpClient.GetAsync(url);
var responseContent = await response.Content.ReadAsStringAsync();
var res = JsonConvert.DeserializeObject<TData>(responseContent);
```

若要通过GET方法下载文件，可以通过如下方式：

```c#
using var httpClient = new HttpClient();
var downloadUrl = $"{protocol}://{ip}:{port}/{srcDir}/{srcFile}"; // 文件虚拟路径
var downloadPath = $"{disk}:\\{desDir}\\{desFile}"; // 下载物理路径
var response = await httpClient.GetAsync(downloadUrl);
var fileStream = new FileStream(downloadPath, FileMode.OpenOrCreate, FileAccess.Write);
await response.Content.CopyToAsync(fileStream);
fileStream.Close();
```

若要下载超过2GB的文件，则需要通过如下方式：

```c#
using var httpClient = new HttpClient();
var downloadUrl = $"{protocol}://{ip}:{port}/{srcDir}/{srcFile}"; // 文件虚拟路径
var downloadPath = $"{disk}:\\{desDir}\\{desFile}"; // 下载物理路径
var response = await httpClient.GetAsync(downloadUrl, HttpCompletionOption.ResponseHeadersRead); // 拿到响应头就返回
var fileStream = new FileStream(downloadPath, FileMode.OpenOrCreate, FileAccess.Write);
await response.Content.CopyToAsync(fileStream);
fileStream.Close();
```

若要给大文件下载添加进度提示，可以事先通过http响应头的Content-Length获取到文件大小，具体实现如下：

```c#
using var httpClient = new HttpClient();
var downloadUrl = $"{protocol}://{ip}:{port}/{srcDir}/{srcFile}"; // 文件虚拟路径
var downloadPath = $"{disk}:\\{desDir}\\{desFile}"; // 下载物理路径
var response = await httpClient.GetAsync(downloadUrl, HttpCompletionOption.ResponseHeadersRead);
var fileStream = new FileStream(downloadPath, FileMode.OpenOrCreate, FileAccess.Write);

var contentStream = await response.Content.ReadAsStreamAsync();
var totalLength = response.Content.Headers.ContentLength;
long readLength = 0L;
int length;
byte[] buffer = new byte[5 * 1024]; // 5KB缓存
while ((length = await contentStream.ReadAsync(buffer)) != 0)
{
    readLength += length;
    if (totalLength > 0)
    {
        Console.WriteLine("下载进度: " + Math.Round((double)readLength / totalLength.Value * 100, 2) + "%");
    }
    else
    {
        Console.WriteLine("已下载: " + Math.Round(readLength / 1024.0, 2) + "KB");
    }
    fileStream.Write(buffer, 0, length);
}

fileStream.Close();
```

### POST

根据Content-Type类型，常见的请求类型有application/json、multipart/form-data、application/x-www-form-urlencoded等。

#### application/json

使用POST请求application/json类型的示例如下(.NET中对应接口参数使用FromBody特性，或直接使用类接收)：

```c#
using var httpClient = new HttpClient();
var url = $"{protocol}://{ip}:{port}/{api}";
var stringContent = new StringContent(JsonConvert.SerializeObject(new { parameter1 = "value1", parameter2 = "value2" }), Encoding.UTF8, "application/json");
var response = await httpClient.PostAsync(url, stringContent);
var responseContent = await response.Content.ReadAsStringAsync();
var res = JsonConvert.DeserializeObject<TData>(responseContent);
```

#### multipart/form-data

使用POST请求multipart/form-data类型的示例如下(.NET中对应接口参数使用FromForm特性)：

```c#
using var httpClient = new HttpClient();
var url = $"{protocol}://{ip}:{port}/{api}"; 
using var formData = new MultipartFormDataContent
{
    { new StringContent("value1"), "parameter1" },
    { new StringContent("value2"), "parameter2" }
};
var response = await httpClient.PostAsync(url, formData);
var responseContent = await response.Content.ReadAsStringAsync();
var res = JsonConvert.DeserializeObject<TData>(responseContent);
```

若要添加文件流参数(.NET中使用IFormFile接收), 可使用以下方式：

```c#
var downloadUrl = Path.Combine(new DirectoryInfo(AppContext.BaseDirectory).Parent.FullName, "Upload", fileMd5);
using FileStream stream = System.IO.File.OpenRead(downloadUrl);
using var memoryStream = new MemoryStream();
stream.CopyTo(memoryStream);
byte[] fileBytes = memoryStream.ToArray();

formData.Add(new ByteArrayContent(fileBytes), "parameter3", Path.GetFileName(fileName));
```

#### application/x-www-form-urlencoded

使用POST请求application/x-www-form-urlencoded类型的示例如下(.NET中对应接口参数使用FromForm特性)：

```c#
using var httpClient = new HttpClient();
var url = $"{protocol}://{ip}:{port}/{api}";
var urlEncodedContent = new FormUrlEncodedContent(new List<KeyValuePair<string, string>>()
{
    new KeyValuePair<string, string>("parameter1","value1"),
    new KeyValuePair<string, string>("parameter2[0]","value2"), // parameter2为数组类型
    new KeyValuePair<string, string>("parameter2[1]","value3")
});
var response = await httpClient.PostAsync(url, urlEncodedContent);
var responseContent = await response.Content.ReadAsStringAsync();
var res = JsonConvert.DeserializeObject<TData>(responseContent);
```

## 参考文档

- [Http请求中各种Content-Type类型详解大全](https://config.net.cn/tools/HttpContentType.html)

- [SocketsHttpHandler类使用参考](https://learn.microsoft.com/zh-cn/dotnet/api/system.net.http.socketshttphandler)

- [使用HttpClient类发出HTTP请求](https://learn.microsoft.com/zh-cn/dotnet/fundamentals/networking/http/httpclient)