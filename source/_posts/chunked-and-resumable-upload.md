---
title: 分片上传与断点续传
date: 2025-03-25 17:28:11
categories:
- Network-Protocol
tags:
- c#
- .NET
- ASP.NET Core
---

使用C#实现大文件上传中的分片上传与断点续传功能。

<!--more-->

## 前言

在文件上传场景中，分片上传和断点续传是处理大文件上传的重要技术。

- 分片上传：将大文件分割成多个小块分别上传，最后在服务器端合并
- 断点续传：记录已上传的分片，以便在中断后可以从中断点继续上传

## 分片上传

### 服务端实现

UploadChunk接口实现了分片的临时存储(存储的临时文件名为fileId_chunkNumber)，以及分片的最终合并。

```c#
[HttpPost]
[AllowAnonymous]
public async Task<IActionResult> UploadChunk()
{
    var form = await Request.ReadFormAsync();
    var fileId = form["fileId"].ToString();
    var chunkNumber = int.Parse(form["chunkNumber"].ToString());
    var totalChunks = int.Parse(form["totalChunks"].ToString());
    var fileName = form["fileName"].ToString();
    var chunk = form.Files["chunk"];

    // 确保上传目录存在
    Directory.CreateDirectory(_uploadPath);

    // 临时保存分片
    var chunkPath = Path.Combine(_uploadPath, $"{fileId}_{chunkNumber}");
    using (var stream = new FileStream(chunkPath, FileMode.Create))
    {
        await chunk.CopyToAsync(stream);
    }

    // 如果是最后一个分片，合并文件
    if (chunkNumber == totalChunks - 1)
    {
        await MergeChunksAsync(fileId, totalChunks, fileName);
        return Ok(new { Message = "Upload complete", FileName = fileName });
    }

    return Ok(new { Message = "Chunk uploaded", ChunkNumber = chunkNumber });
}

private async Task MergeChunksAsync(string fileId, int totalChunks, string fileName)
{
    var finalPath = Path.Combine(_uploadPath, fileName);

    using (var finalStream = new FileStream(finalPath, FileMode.Create))
    {
        for (int i = 0; i < totalChunks; i++)
        {
            var chunkPath = Path.Combine(_uploadPath, $"{fileId}_{i}");
            using (var chunkStream = System.IO.File.OpenRead(chunkPath))
            {
                await chunkStream.CopyToAsync(finalStream);
            }
            System.IO.File.Delete(chunkPath); // 合并后删除分片
        }
    }
}
```

### 客户端实现

首先创建一个FileUploader类，用来处理本地的文件流，并调用服务端的分片上传接口。

```c#
public class FileUploader
{
    private const int ChunkSize = 1024 * 1024; // 1MB 每块

    public async Task UploadFileInChunksAsync(string filePath, string apiUrl)
    {
        var fileInfo = new FileInfo(filePath);
        var totalChunks = (int)Math.Ceiling((double)fileInfo.Length / ChunkSize);
        var fileId = Guid.NewGuid().ToString(); // 唯一文件标识

        using var fileStream = File.OpenRead(filePath);

        for (int chunkNumber = 0; chunkNumber < totalChunks; chunkNumber++)
        {
            var chunkData = new byte[ChunkSize];
            var bytesRead = await fileStream.ReadAsync(chunkData, 0, ChunkSize);

            if (bytesRead == 0) break;

            // 只取实际读取的字节
            var actualChunkData = new byte[bytesRead];
            Array.Copy(chunkData, actualChunkData, bytesRead);

            await UploadChunkAsync(apiUrl, fileId, chunkNumber, totalChunks,
                                    fileInfo.Name, actualChunkData);
        }
    }

    private async Task UploadChunkAsync(string apiUrl, string fileId,
                                        int chunkNumber, int totalChunks,
                                        string fileName, byte[] chunkData)
    {
        using var httpClient = new HttpClient();
        using var content = new MultipartFormDataContent
        {
            { new StringContent(fileId), "fileId" },
            { new StringContent(chunkNumber.ToString()), "chunkNumber" },
            { new StringContent(totalChunks.ToString()), "totalChunks" },
            { new StringContent(fileName), "fileName" },
            { new ByteArrayContent(chunkData), "chunk", "chunk.dat" }
        };

        var response = await httpClient.PostAsync(apiUrl, content);
        response.EnsureSuccessStatusCode();
    }
}
```

然后在客户端的UI线程中使用异步方式调用FileUploader类中的分片上传方法。

```c#
var uploader = new FileUploader();

var serviceUrl = "http://localhost:5001";
var apiUrl = $"{serviceUrl}/api/UploadChunk";
var filePath = $"{disk}:\\{srcDir}\\{srcFile}"

await uploader.UploadFileInChunksAsync(filePath, apiUrl);
```

## 断点续传

### 服务端实现

GetUploadStatus接口根据文件标识fileId(此处为文件MD5)查询已上传的分片编号，UploadChunk接口的功能同上。

```c#
[ApiController]
[Route("api/upload")]
public class UploadController : ControllerBase
{
    [HttpGet("status")]
    public IActionResult GetUploadStatus(string fileId)
    {
        var uploadedChunks = new List<int>();

        if (Directory.Exists(_uploadPath))
        {
            var chunkFiles = Directory.GetFiles(_uploadPath, $"{fileId}_*");
            foreach (var chunkFile in chunkFiles)
            {
                if (int.TryParse(Path.GetFileName(chunkFile).Split('_').Last(), out var chunkNumber))
                {
                    uploadedChunks.Add(chunkNumber);
                }
            }
        }

        return Ok(uploadedChunks);
    }

    [HttpPost]
    [AllowAnonymous]
    public async Task<IActionResult> UploadChunk()
    {
        var form = await Request.ReadFormAsync();
        var fileId = form["fileId"].ToString();
        var chunkNumber = int.Parse(form["chunkNumber"].ToString());
        var totalChunks = int.Parse(form["totalChunks"].ToString());
        var fileName = form["fileName"].ToString();
        var chunk = form.Files["chunk"];

        // 确保上传目录存在
        Directory.CreateDirectory(_uploadPath);

        // 临时保存分片
        var chunkPath = Path.Combine(_uploadPath, $"{fileId}_{chunkNumber}");
        using (var stream = new FileStream(chunkPath, FileMode.Create))
        {
            await chunk.CopyToAsync(stream);
        }

        // 如果是最后一个分片，合并文件
        if (chunkNumber == totalChunks - 1)
        {
            await MergeChunksAsync(fileId, totalChunks, fileName);
            return Ok(new { Message = "Upload complete", FileName = fileName });
        }

        return Ok(new { Message = "Chunk uploaded", ChunkNumber = chunkNumber });
    }

    private async Task MergeChunksAsync(string fileId, int totalChunks, string fileName)
    {
        var finalPath = Path.Combine(_uploadPath, fileName);

        using (var finalStream = new FileStream(finalPath, FileMode.Create))
        {
            for (int i = 0; i < totalChunks; i++)
            {
                var chunkPath = Path.Combine(_uploadPath, $"{fileId}_{i}");
                using (var chunkStream = System.IO.File.OpenRead(chunkPath))
                {
                    await chunkStream.CopyToAsync(finalStream);
                }
                System.IO.File.Delete(chunkPath); // 合并后删除分片
            }
        }
    }
}
```

### 客户端实现

首先创建一个ResumableUploader类，调用服务端的文件分片上传状态接口，对于已上传的分片跳过处理，然后调用服务端的分片上传接口继续上传，以实现断点续传功能。

```c#
public class ResumableUploader
{
    private const int ChunkSize = 1024 * 1024; // 1MB 每块

    public async Task UploadWithResumeAsync(string filePath, string apiUrl)
    {
        var fileInfo = new FileInfo(filePath);
        var totalChunks = (int)Math.Ceiling((double)fileInfo.Length / ChunkSize);
        var fileId = GetFileId(filePath); // 基于文件内容生成唯一ID

        // 获取已上传的分片信息
        var uploadedChunks = await GetUploadedChunksAsync(apiUrl, fileId);

        using var fileStream = File.OpenRead(filePath);

        for (int chunkNumber = 0; chunkNumber < totalChunks; chunkNumber++)
        {
            if (uploadedChunks.Contains(chunkNumber))
                continue; // 跳过已上传的分片

            fileStream.Seek(chunkNumber * ChunkSize, SeekOrigin.Begin);

            var chunkData = new byte[ChunkSize];
            var bytesRead = await fileStream.ReadAsync(chunkData, 0, ChunkSize);

            if (bytesRead == 0) break;

            var actualChunkData = new byte[bytesRead];
            Array.Copy(chunkData, actualChunkData, bytesRead);

            await UploadChunkAsync(apiUrl, fileId, chunkNumber, totalChunks,
                                fileInfo.Name, actualChunkData);
        }
    }

    private string GetFileId(string filePath)
    {
        using var md5 = MD5.Create();
        using var stream = File.OpenRead(filePath);
        var hash = md5.ComputeHash(stream);
        return BitConverter.ToString(hash).Replace("-", "").ToLower();
    }

    private async Task<List<int>> GetUploadedChunksAsync(string apiUrl, string fileId)
    {
        try
        {
            using var httpClient = new HttpClient();
            var response = await httpClient.GetAsync($"{apiUrl}/status?fileId={fileId}");
            response.EnsureSuccessStatusCode();

            var json = await response.Content.ReadAsStringAsync();
            return JsonSerializer.Deserialize<List<int>>(json);
        }
        catch
        {
            return new List<int>();
        }
    }

    private async Task UploadChunkAsync(string apiUrl, string fileId,
                                    int chunkNumber, int totalChunks,
                                    string fileName, byte[] chunkData)
    {
        using var httpClient = new HttpClient();
        using var content = new MultipartFormDataContent
        {
            { new StringContent(fileId), "fileId" },
            { new StringContent(chunkNumber.ToString()), "chunkNumber" },
            { new StringContent(totalChunks.ToString()), "totalChunks" },
            { new StringContent(fileName), "fileName" },
            { new ByteArrayContent(chunkData), "chunk", "chunk.dat" }
        };

        var response = await httpClient.PostAsync(apiUrl, content);
        response.EnsureSuccessStatusCode();
    }
}
```

然后在客户端的UI线程中使用异步方式调用ResumableUploader类中的断点续传方法。

```c#
var uploader = new ResumableUploader();

var serviceUrl = "http://localhost:5001";
var apiUrl = $"{serviceUrl}/api/upload";
var filePath = $"{disk}:\\{srcDir}\\{srcFile}"

await uploader.UploadWithResumeAsync(filePath, apiUrl);
```

## 参考文档

- [阿里云OSS断点续传上传操作指南](https://help.aliyun.com/zh/oss/user-guide/resumable-upload)