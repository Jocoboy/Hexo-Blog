---
title: .NET实现断点续传(上传/下载)
date: 2025-03-25 17:28:11
categories:
- Framework
tags:
- .NET
- ASP.NET Core
- WinForm
---

基于.NET实现大文件的断点续传功能，包含上传与下载。其中上传部分包括分片上传与断点续传功能，并借助WinForm实现上传进度反馈、并行上传、分片大小动态调整等辅助功能。

<!--more-->

## 前言

断点续传基于分片上传和分片下载，包含断点续传上传与断点续传下载两部分，下面基于.NET分别对其进行功能实现。

## 断点续传上传

在文件上传场景中，分片上传和断点续传是处理大文件上传的重要技术。

除了基本的断点续传功能，大文件上传还应提供以下辅助功能：

- 分片大小调整：根据网络状况动态调整分片大小
- 并行上传：可以并行上传多个分片以提高速度
- 完整性校验：合并文件后校验哈希值确保文件完整
- 清理机制：定期清理未完成上传的临时分片
- 进度反馈：提供上传进度信息给用户

下面我们来简单实现上述功能。

### 分片上传

分片上传流程如下：

- 客户端首次上传前生成文件唯一ID(通常使用文件内容哈希，此处为了演示重新上传是从0开始的，使用的是Guid)
- 客户端将文件分割为固定大小的块，每块单独上传到服务器
- 服务器接收并临时保存每个分片
- 当所有分片上传完成后，服务器合并分片为完整文件

#### 服务端实现

UploadChunk接口实现了分片的临时存储(存储的临时文件名为fileId_chunkNumber)，以及分片的最终合并。

```c#
[HttpPost]
[AllowAnonymous]
public async Task<IActionResult> UploadChunkAsync(CancellationToken cancellationToken)
{
    var form = await Request.ReadFormAsync(cancellationToken);
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
        await chunk.CopyToAsync(stream, cancellationToken);
    }

    // 如果是最后一个分片，合并文件
    if (chunkNumber == totalChunks - 1)
    {
        await MergeChunksAsync(fileId, totalChunks, fileName, cancellationToken);
        return Ok(new { Message = "Upload complete", FileName = fileName });
    }

    return Ok(new { Message = "Chunk uploaded", ChunkNumber = chunkNumber });
}

private async Task MergeChunksAsync(string fileId, int totalChunks, string fileName, CancellationToken cancellationToken)
{
    var finalPath = Path.Combine(_uploadPath, fileName);

    using (var finalStream = new FileStream(finalPath, FileMode.Create))
    {
        for (int i = 0; i < totalChunks; i++)
        {
            var chunkPath = Path.Combine(_uploadPath, $"{fileId}_{i}");
            using (var chunkStream = System.IO.File.OpenRead(chunkPath))
            {
                await chunkStream.CopyToAsync(finalStream, cancellationToken);
            }
            System.IO.File.Delete(chunkPath); // 合并后删除分片
        }
    }
}
```

#### 客户端实现

首先创建一个ChunkedUploadService类，用来处理本地的文件流，并调用服务端的分片上传接口。

```c#
public class ChunkedUploadService
{
    private readonly HttpClient _httpClient;
    private readonly string _serviceUri;
    private const string UploadChunkApiUri = "/api/Upload/UploadChunk";
    private const int ChunkSize = 1024 * 1024; // 1MB 每块

    public ChunkedUploadService(string serviceUri)
    {
        _serviceUri = serviceUri;
        _httpClient = new HttpClient();
    }

    public async Task UploadInChunksAsync(string filePath, CancellationToken cancellationToken)
    {
        var fileInfo = new FileInfo(filePath);
        var totalChunks = (int)Math.Ceiling((double)fileInfo.Length / ChunkSize);
        var fileId = Guid.NewGuid().ToString(); // 唯一文件标识

        using var fileStream = File.OpenRead(filePath);

        for (int chunkNumber = 0; chunkNumber < totalChunks; chunkNumber++)
        {
            var chunkData = new byte[ChunkSize];
            var bytesRead = await fileStream.ReadAsync(chunkData, 0, ChunkSize, cancellationToken);

            if (bytesRead == 0) break;

            var actualChunkData = new byte[bytesRead]; // 只取实际读取的字节
            Array.Copy(chunkData, actualChunkData, bytesRead);

            await UploadChunkAsync(fileId, chunkNumber, totalChunks, fileInfo.Name, actualChunkData, cancellationToken);
        }
    }

    private async Task UploadChunkAsync(string fileId, int chunkNumber, int totalChunks, string fileName, byte[] chunkData, CancellationToken cancellationToken)
    {
        using var content = new MultipartFormDataContent
        {
            { new StringContent(fileId), "fileId" },
            { new StringContent(chunkNumber.ToString()), "chunkNumber" },
            { new StringContent(totalChunks.ToString()), "totalChunks" },
            { new StringContent(fileName), "fileName" },
            { new ByteArrayContent(chunkData), "chunk", "chunk.dat" }
        };

        var response = await _httpClient.PostAsync($"{_serviceUri}{UploadChunkApiUri}", content, cancellationToken);
        response.EnsureSuccessStatusCode();
    }
}
```

然后在客户端的UI线程中使用异步方式调用ChunkedUploadService类中的分片上传方法。

```c#
var serviceUri = "http://localhost:5001";
var filePath = $"{disk}:\\{srcDir}\\{srcFile}";
var uploader = new ChunkedUploadService(serviceUri);
await uploader.UploadInChunksAsync(filePath, CancellationToken.None);
```

### 断点续传

断点续传流程如下：

- 客户端首次上传前生成文件唯一ID
- 上传前根据文件文件唯一ID查询服务器已接收的分片，继续上传时只上传缺失的分片
- 当所有分片上传完成后合并分片

#### 服务端实现

GetUploadStatus接口根据文件标识fileId查询已上传的分片编号，UploadChunk接口的功能同上。

```c#
[ApiController]
[Route("[controller]/[action]")]
[Authorize]
public class UploadController: ControllerBase
{
    private readonly IWebHostEnvironment _env;
    private readonly string _uploadPath;

    public UploadController(IWebHostEnvironment env)
    {
        _env = env;
        _uploadPath = Path.Combine(_env.WebRootPath, "uploads");
    }

    [HttpGet]
    [AllowAnonymous]
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
    public async Task<IActionResult> UploadChunkAsync(CancellationToken cancellationToken)
    {
        // 同分片上传
    }

    private async Task MergeChunksAsync(string fileId, int totalChunks, string fileName, CancellationToken cancellationToken)
    {
        // 同分片上传
    }
}
```

#### 客户端实现

首先创建一个ResumableUploadService类，调用服务端的文件分片上传状态接口，对于已上传的分片跳过处理，然后调用服务端的分片上传接口继续上传，以实现断点续传功能。

```c#
public class ResumableUploadService
{
    private readonly HttpClient _httpClient;
    private readonly string _serviceUri;
    private const string UploadChunkApiUri = "/api/Upload/UploadChunk";
    private const string GetUploadStatusApiUri = "/api/Upload/GetUploadStatus";
    private const int ChunkSize = 1024 * 1024; // 1MB 每块

    public ResumableUploadService(string serviceUri)
    {
        _serviceUri = serviceUri;
        _httpClient = new HttpClient();
    }

    public async Task UploadWithResumeAsync(string filePath, CancellationToken cancellationToken)
    {
        var fileInfo = new FileInfo(filePath);
        var totalChunks = (int)Math.Ceiling((double)fileInfo.Length / ChunkSize);
        using var fileStream = File.OpenRead(filePath);
        var fileId = GetFileHash(fileStream, HashAlgorithmType.Sha256); // 基于文件内容生成唯一ID

        // 获取已上传的分片信息
        var uploadedChunks = await GetUploadedChunksAsync(fileId, cancellationToken);

        for (int chunkNumber = 0; chunkNumber < totalChunks; chunkNumber++)
        {
            if (uploadedChunks.Contains(chunkNumber))  continue; // 跳过已上传的分片

            fileStream.Seek(chunkNumber * ChunkSize, SeekOrigin.Begin);

            var chunkData = new byte[ChunkSize];
            var bytesRead = await fileStream.ReadAsync(chunkData, 0, ChunkSize, cancellationToken);

            if (bytesRead == 0) break;

            var actualChunkData = new byte[bytesRead];
            Array.Copy(chunkData, actualChunkData, bytesRead);

            await UploadChunkAsync(fileId, chunkNumber, totalChunks, fileInfo.Name, actualChunkData, cancellationToken);
        }
    }

    private static string GetFileHash(Stream stream, HashAlgorithmType hashAlgorithmType)
    {
        switch (hashAlgorithmType)
        {
            case HashAlgorithmType.Md5:
                using (var hashAlgorithm = MD5.Create())
                {
                    var hash = hashAlgorithm.ComputeHash(stream);
                    return BitConverter.ToString(hash).Replace("-", "").ToLower();
                }
            case HashAlgorithmType.Sha256:
                using (var hashAlgorithm = SHA256.Create())
                {
                    var hash = hashAlgorithm.ComputeHash(stream);
                    return BitConverter.ToString(hash).Replace("-", "").ToLower();
                }
            default:
                return string.Empty;
        }
    }

    private async Task<List<int>> GetUploadedChunksAsync(string fileId, CancellationToken cancellationToken)
    {
        try
        {
            var response = await _httpClient.GetAsync($"{_serviceUri}{GetUploadStatusApiUri}?fileId={fileId}");
            response.EnsureSuccessStatusCode();

            var json = await response.Content.ReadAsStringAsync(cancellationToken);
            return JsonSerializer.Deserialize<List<int>>(json);
        }
        catch
        {
            return new List<int>();
        }
    }

    private async Task UploadChunkAsync(string fileId, int chunkNumber, int totalChunks, string fileName, byte[] chunkData, CancellationToken cancellationToken)
    {
       // 同分片上传
    }
}
```

然后在客户端的UI线程中使用异步方式调用ResumableUploadService类中的断点续传方法。

```c#
var serviceUri = "http://localhost:5001";
var filePath = $"{disk}:\\{srcDir}\\{srcFile}";
var uploader = new ResumableUploadService(serviceUri);
await uploader.UploadWithResumeAsync(filePath, CancellationToken.None);
```

### 上传进度反馈与上传取消

通常情况下，大文件上传时需要给用户提供上传进度信息以及取消上传的功能，下面使用WinForm简单实现上述功能。

#### 服务端实现

服务端代码延用[断点续传](#断点续传)，不再展示。

#### 客户端实现

客户端的ResumableUploadService中的主要变化是添加了三个委托事件，分别用来更新上传百分比、状态消息、上传与取消按钮禁用状态。

```c#
public class ResumableUploadService
{
    ...
    // 进度和状态事件
    public event Action<int> ProgressChanged; // 上传百分比
    public event Action<string> StatusChanged; // 状态消息
    public event Action<bool> UploadCompleted; // 完成状态

    public async Task UploadWithResumeAsync(string filePath, CancellationToken cancellationToken)
    {
        try
        {
            StatusChanged?.Invoke("正在准备上传...");

            var fileInfo = new FileInfo(filePath);
            var totalChunks = (int)Math.Ceiling((double)fileInfo.Length / ChunkSize);
            using var fileStream = File.OpenRead(filePath);
            var fileId = GetFileHash(fileStream, HashAlgorithmType.Sha256); // 基于文件内容生成唯一ID

            StatusChanged?.Invoke("正在检查已上传分片...");

            // 获取已上传的分片信息
            var uploadedChunks = await GetUploadedChunksAsync(fileId, cancellationToken);

            for (int chunkNumber = 0; chunkNumber < totalChunks; chunkNumber++)
            {
                cancellationToken.ThrowIfCancellationRequested();

                if (uploadedChunks.Contains(chunkNumber))
                {
                    ProgressChanged?.Invoke((int)((double)chunkNumber / totalChunks * 100));
                    continue;
                }

                fileStream.Seek(chunkNumber * ChunkSize, SeekOrigin.Begin);

                var chunkData = new byte[ChunkSize];
                var bytesRead = await fileStream.ReadAsync(chunkData, 0, ChunkSize, cancellationToken);

                if (bytesRead == 0) break;

                var actualChunkData = new byte[bytesRead];
                Array.Copy(chunkData, actualChunkData, bytesRead);

                StatusChanged?.Invoke($"正在上传分片 {chunkNumber + 1}/{totalChunks}");

                await UploadChunkAsync(fileId, chunkNumber, totalChunks, fileInfo.Name, actualChunkData, cancellationToken);

                var progress = (int)((double)(chunkNumber + 1) / totalChunks * 100);
                ProgressChanged?.Invoke(progress);
            }

            StatusChanged?.Invoke("上传完成！");
            UploadCompleted?.Invoke(true);
        }
        catch (OperationCanceledException)
        {
            StatusChanged?.Invoke("上传已取消");
            UploadCompleted?.Invoke(false);
        }
        catch (Exception ex)
        {
            StatusChanged?.Invoke($"上传失败: {ex.Message}");
            UploadCompleted?.Invoke(false);
        }
    }
    ...
}
```

主窗体MainForm中分别绑定三个委托事件到相应的UI线程上，使用Invoke跨线程更新UI，实现上传过程中实时更新上传进度及状态信息。

Control.InvokeRequired属性指示调用方在对控件进行方法调用时是否必须调用Invoke方法，因为调用方位于创建控件所在的线程以外的线程中。Windows窗体中的控件绑定到特定线程，并且不是线程安全的。因此，如果要从其他线程调用控件的方法，则必须使用控件的调用方法之一来封送对正确线程的调用。

```c#
public partial class MainForm : Form
{
    private readonly ResumableUploadService _uploader;
    private CancellationTokenSource _cts;
    private readonly string _serviceUri;

    public MainForm()
    {
        InitializeComponent();

        _serviceUri = "http://localhost:5001";
        _uploader = new ResumableUploadService(_serviceUri);

        // 绑定事件
        _uploader.ProgressChanged += UpdateProgress;
        _uploader.StatusChanged += UpdateStatus;
        _uploader.UploadCompleted += UploadCompleted;
    }

    private void UpdateProgress(int percent)
    {
        if (progressBar.InvokeRequired)
        {
            progressBar.Invoke(new Action<int>(UpdateProgress), percent);
            return;
        }
        progressBar.Value = percent;
    }

    private void UpdateStatus(string message)
    {
        if (lblStatus.InvokeRequired)
        {
            lblStatus.Invoke(new Action<string>(UpdateStatus), message);
            return;
        }
        lblStatus.Text = message;
    }

    private void UploadCompleted(bool success)
    {
        if (btnUpload.InvokeRequired)
        {
            btnUpload.Invoke(new Action<bool>(UploadCompleted), success);
            return;
        }
        btnUpload.Enabled = true;
        btnCancel.Enabled = false;
    }

    private async void btnUpload_Click(object sender, EventArgs e)
    {
        if (string.IsNullOrEmpty(textBox.Text) || !File.Exists(textBox.Text))
        {
            MessageBox.Show("请选择有效的文件");
            return;
        }

        btnUpload.Enabled = false;
        btnCancel.Enabled = true;
        progressBar.Value = 0;

        _cts = new CancellationTokenSource();
        await _uploader.UploadWithResumeAsync(textBox.Text,_cts.Token);
    }

    private void btnCancel_Click(object sender, EventArgs e)
    {
        _cts?.Cancel();
    }

    private void btnBrowse_Click(object sender, EventArgs e)
    {
        using var openFileDialog = new OpenFileDialog();
        if (openFileDialog.ShowDialog() == DialogResult.OK)
        {
            textBox.Text = openFileDialog.FileName;
        }
    }

    protected override void OnFormClosing(FormClosingEventArgs e)
    {
        _cts?.Cancel();
        base.OnFormClosing(e);
    }
}
```

### 并行上传

并行上传是提升大文件传输效率的有效手段，并行上传可将总时间缩短为单线程上传时间/N(N为并行度)。并行上传适合在不稳定的网络环境(如高延迟)中使用，可充分利用间歇性网络带宽。

并行上传时多个分片同时上报上传进度，进度条呈现更平滑，更适合需要实时进度反馈的场景。

#### 服务端实现

服务端代码延用[断点续传](#断点续传)，不再展示。

#### 客户端实现

客户端基于ResumableUploadService，创建了一个并行上传服务类ParallelUploadService，使用`Parallel.ForEachAsync`实现并行上传，并设置了最大并行度(默认为4)。

```c#
public class ParallelUploadService
{
    ...
    private readonly int _maxParallelism;

    public ParallelUploadService(string serviceUri, int maxParallelism = 4)
    {
        ...
        _maxParallelism = maxParallelism;
    }

    public async Task ParallelUploadFileAsync(string filePath, CancellationToken cancellationToken)
    {
        try
        {
            StatusChanged?.Invoke("正在准备上传...");

            var fileInfo = new FileInfo(filePath);
            var totalChunks = (int)Math.Ceiling((double)fileInfo.Length / ChunkSize);
            using var fileStream = File.OpenRead(filePath);
            var fileId = GetFileHash(fileStream, HashAlgorithmType.Sha256); // 基于文件内容生成唯一ID

            StatusChanged?.Invoke("正在检查已上传分片...");

            // 获取已上传的分片信息
            var uploadedChunks = await GetUploadedChunksAsync(fileId, cancellationToken);

            var chunksToUpload = Enumerable.Range(0, totalChunks).Where(chunk => !uploadedChunks.Contains(chunk)).ToList();

            var progressLock = new object();
            var uploadedCount = 0;
            var totalToUpload = chunksToUpload.Count;

            StatusChanged?.Invoke($"准备上传 {totalToUpload} 个分片...");

            var parallelOptions = new ParallelOptions
            {
                MaxDegreeOfParallelism = _maxParallelism,
                CancellationToken = cancellationToken
            };

            await Parallel.ForEachAsync(chunksToUpload, parallelOptions, async (chunkNumber, cancellationToken) =>
            {
                cancellationToken.ThrowIfCancellationRequested();

                fileStream.Seek(chunkNumber * ChunkSize, SeekOrigin.Begin);

                var chunkData = new byte[ChunkSize];
                var bytesRead = await fileStream.ReadAsync(chunkData, 0, ChunkSize, cancellationToken);

                var actualChunkData = new byte[bytesRead];
                Array.Copy(chunkData, actualChunkData, bytesRead);

                StatusChanged?.Invoke($"正在上传分片 {chunkNumber + 1}/{totalChunks}");

                await UploadChunkAsync(fileId, chunkNumber, totalChunks, fileInfo.Name, actualChunkData, cancellationToken);

                // 使用锁保证进度更新的原子性
                lock (progressLock)
                {
                    uploadedCount++;
                    var progress = (int)((double)uploadedCount / totalToUpload * 100);
                    ProgressChanged?.Invoke(progress);
                }
            });

            StatusChanged?.Invoke("上传完成！");
            UploadCompleted?.Invoke(true);
        }
        catch (OperationCanceledException)
        {
            StatusChanged?.Invoke("上传已取消");
            UploadCompleted?.Invoke(false);
        }
        catch (Exception ex)
        {
            StatusChanged?.Invoke($"上传失败: {ex.Message}");
            UploadCompleted?.Invoke(false);
        }
    }

    ...
}
```

主窗体MainForm中对于ParallelUploadService服务类的调用方式与ResumableUploadService类似，不再展示。

### 动态调整分片大小

根据网络状态动态调整分片大小，可以优化分片上传效率。

#### 服务端实现

由于分片的大小是动态计算得出的，存储的临时文件名由fileId_chunkNumber变更为{fileId}/chunk_{offsetValue}。

服务端的分片存储结构如下：

>uploads/
├── {fileId}/          // 每个文件一个独立目录
│   ├── chunk_{offsetValue1}        // 分片文件按偏移量命名
│   ├── chunk_{offsetValue2}
│   └── ...
└── completed/       // 最终合并的文件

由于上传是并行的，服务端在检查分片上传完整性时，必须以所有分片大小总和等于文件大小为标志，而不能以其中某一分片的偏移量+分片大小等于文件大小为标志。

```c#
[ApiController]
[Route("[controller]/[action]")]
[Authorize]
public class UploadController: ControllerBase
{
    private readonly IWebHostEnvironment _env;
    private readonly string _uploadPath;
    private readonly ILogger<UploadController> _logger;

    public UploadController(IWebHostEnvironment env, ILogger<UploadController> logger)
    {
        _env = env;
        _uploadPath = Path.Combine(_env.WebRootPath, "uploads");
        _logger = logger;
    }

    [HttpGet]
    [AllowAnonymous]
    public IActionResult Ping()
    {
        return Ok(DateTime.UtcNow);
    }

    [HttpGet]
    [AllowAnonymous]
    public IActionResult GetUploadStatus(string fileId)
    {
        var uploadDir = Path.Combine(_uploadPath, fileId);
        if (!Directory.Exists(uploadDir))
        {
            return Ok(new { UploadedChunks = Array.Empty<UploadedChunk>() });
        }

        var chunks = Directory.GetFiles(uploadDir, "chunk_*")
            .Select(f => new FileInfo(f))
            .Select(f => new UploadedChunk
            {
                Offset = long.Parse(f.Name.Split('_')[1]),
                Size = (int)f.Length
            })
            .OrderBy(c => c.Offset)
            .ToList();

        return Ok(new { UploadedChunks = chunks });
    }

    [HttpPost]
    [AllowAnonymous]
    public async Task<IActionResult> UploadChunkAsync(CancellationToken cancellationToken)
    {
        var form = await Request.ReadFormAsync(cancellationToken);
        var fileId = form["fileId"].ToString();
        var chunkIndex = int.Parse(form["chunkIndex"].ToString());
        var chunkOffset = int.Parse(form["chunkOffset"].ToString());
        var chunkSize = int.Parse(form["chunkSize"].ToString());
        var fileSize = int.Parse(form["fileSize"].ToString());
        var fileName = form["fileName"].ToString();
        var chunk = form.Files["chunk"];

        try
        {
            // 验证分片
            if (chunk == null || chunk.Length == 0)
            {
                return BadRequest("无效的分片数据");
            }

            // 创建上传目录
            var uploadDir = Path.Combine(_uploadPath, fileId);
            Directory.CreateDirectory(uploadDir);

            // 保存分片
            var chunkPath = Path.Combine(uploadDir, $"chunk_{chunkOffset}");
            using (var stream = new FileStream(chunkPath, FileMode.Create))
            {
                await chunk.CopyToAsync(stream, cancellationToken);
            }

            _logger.LogInformation($"已接收分片 {chunkIndex} (偏移: {chunkOffset}, 大小: {chunk.Length}, 剩余大小: {fileSize - chunkOffset})");

            // 检查是否完成
            if (IsUploadComplete(fileId, fileSize))
            {
                _logger.LogInformation($"文件 {fileName} 所有分片已上传");
                return Ok(new { Completed = true });
            }

            return Ok(new { Completed = false });
        }
        catch(OperationCanceledException ex)
        {
            // 客户端主动取消上传，最后一个分片可能未完整上传，需要进行删除，否则客户端断点续传时，在对分片信息的更新过程中会发生错误
            System.IO.File.Delete(Path.Combine(Path.Combine(_uploadPath, fileId), $"chunk_{chunkOffset}"));
            var errorMsg = $"分片上传失败:{ex.Message}, 分片 chunk_{chunkOffset} 已删除！";
            _logger.LogError(errorMsg);
            return BadRequest(errorMsg);
        }
        catch (Exception ex)
        {
            var errorMsg = $"分片上传失败:{ex.Message}";
            _logger.LogError(errorMsg);
            return BadRequest(errorMsg);
        }
    }

    [HttpPost]
    [AllowAnonymous]
    public async Task<IActionResult> MergeChunksAsync(MergeFile mergeFile, CancellationToken cancellationToken)
    {
        var uploadDir = Path.Combine(_uploadPath, mergeFile.FileId);
        var finalDir = Path.Combine(_uploadPath, "completed");
        var finalPath = Path.Combine(finalDir, mergeFile.FileName);
        if (!Directory.Exists(finalDir))
        {
            Directory.CreateDirectory(finalDir);
        }

        // 获取所有分片并按偏移量排序
        var chunkFiles = Directory.GetFiles(uploadDir, "chunk_*")
            .Select(f => new
            {
                Path = f,
                Offset = long.Parse(Path.GetFileName(f).Split('_')[1])
            })
            .OrderBy(x => x.Offset)
            .ToList();

        // 合并文件
        using (var finalStream = new FileStream(finalPath, FileMode.Create))
        {
            foreach (var chunk in chunkFiles)
            {
                using (var chunkStream = System.IO.File.OpenRead(chunk.Path))
                {
                    await chunkStream.CopyToAsync(finalStream, cancellationToken);
                }
                System.IO.File.Delete(chunk.Path);
            }
        }

        // 清理临时目录
        Directory.Delete(uploadDir);

        using var stream = System.IO.File.OpenRead(finalPath);
        var mergeFileId = GetFileHash(stream, HashAlgorithmType.Sha256);
        if (!mergeFile.FileId.Equals(mergeFileId))
        {
            stream.Close();
            System.IO.File.Delete(finalPath);
            var errorMsg = $"合并后的文件内容哈希不正确，文件可能已损坏，合并前内容哈希为{mergeFile.FileId}, 合并后内容哈希为{mergeFileId}, 合并后的文件已被删除，请重新上传！";
            _logger.LogError(errorMsg);
            return BadRequest(errorMsg);
        }

        _logger.LogInformation($"文件合并完成: {finalPath}");

        return Ok();
    }

    #region private methods
    private static string GetFileHash(Stream stream, HashAlgorithmType hashAlgorithmType)
    {
        switch (hashAlgorithmType)
        {
            case HashAlgorithmType.Md5:
                using (var hashAlgorithm = MD5.Create())
                {
                    var hash = hashAlgorithm.ComputeHash(stream);
                    return BitConverter.ToString(hash).Replace("-", "").ToLower();
                }
            case HashAlgorithmType.Sha256:
                using (var hashAlgorithm = SHA256.Create())
                {
                    var hash = hashAlgorithm.ComputeHash(stream);
                    return BitConverter.ToString(hash).Replace("-", "").ToLower();
                }
            default:
                return string.Empty;
        }
    }

    private bool IsUploadComplete(string fileId, long fileSize)
    {
        var status = GetUploadStatus(fileId) as OkObjectResult;
        var data = status?.Value as dynamic;
        var chunks = data?.UploadedChunks as IEnumerable<UploadedChunk>;

        if (chunks == null || !chunks.Any())
        {
            return false;
        }

        // 获取文件总大小(最后一个分片的结束位置)
        // 请勿使用chunks.Max(c => c.Offset + c.Size), 该值在单线程环境中可以代表所有分片都已上传完毕, 但在多线程环境中是错误的
        long currentTotalSize = chunks.Sum(c => c.Size);

        // 检查是否覆盖了所有字节
        return currentTotalSize >= fileSize;
    }

    #endregion

    #region public class
    public class UploadedChunk
    {
        public long Offset { get; set; }
        public int Size { get; set; }
    }

    public class MergeFile
    {
        public string FileId { get; set; }
        public string FileName { get; set; }
    }
    #endregion
}
```

调用服务端上传时的日志记录如下，可以看到分片的上传是乱序的：

{% asset_img adaptive_resumable_upload_console.png  服务端日志记录输出 %}

#### 客户端实现

客户端实现了AdaptiveResumableUploadService服务类，在并行上传与断点续传的基础上，添加了实时更新网络指标(上行速度/网络延迟)并动态计算当前分片大小的功能，并使用了异步锁SemaphoreSlim确保分片信息更新时的原子性。

注：请勿使用同步锁lock，否则会报错`Object synchronization method was called from an unsynchronized block of code`

```c#
public class AdaptiveResumableUploadService
{
    private readonly HttpClient _httpClient;
    private readonly string _serviceUri;
    private const string UploadChunkApiUri = "/api/Upload/UploadChunk";
    private const string GetUploadStatusApiUri = "/api/Upload/GetUploadStatus";
    private const string MergeChunksApiUri = "/api/Upload/MergeChunks";
    private const string PingApiUri = "/api/Ping";
    private readonly int _maxParallelism;
    private readonly int _minChunkSize;
    private readonly int _maxChunkSize;
    private readonly int _initialChunkSize;

    // 网络状况监测
    private double _averageUploadSpeedMbps = 1.0; // 初始假设1Mbps
    private double _averageLatencyMs = 100; // 初始延迟100ms
    private readonly object _networkLock = new object();
    private readonly SemaphoreSlim _semaphoreSlim = new SemaphoreSlim(1, 1);

    // 进度和状态事件
    public event Action<int> ProgressChanged; // 上传百分比
    public event Action<string> StatusChanged; // 状态消息
    public event Action<bool> UploadCompleted; // 完成状态

    public AdaptiveResumableUploadService(string serviceUri, 
        int maxParallelism =  4,
        int minChunkSize = 256 * 1024,    // 256KB
        int maxChunkSize = 10 * 1024 * 1024, // 10MB
        int initialChunkSize = 1 * 1024 * 1024) // 1MB
    {
        _serviceUri = serviceUri;
        _httpClient = new HttpClient();
        _httpClient.Timeout = Timeout.InfiniteTimeSpan;
        _maxParallelism = maxParallelism;
        _minChunkSize = minChunkSize;
        _maxChunkSize = maxChunkSize;
        _initialChunkSize = initialChunkSize;
    }

    // 主上传方法
    public async Task UploadFileAdaptiveAsync(string filePath, CancellationToken cancellationToken)
    {
        try
        {
            StatusChanged?.Invoke("正在准备上传...");

            var fileInfo = new FileInfo(filePath);
            using var fileStream = File.OpenRead(filePath);
            var fileId = GetFileHash(fileStream, HashAlgorithmType.Sha256); // 基于文件内容生成唯一ID

            StatusChanged?.Invoke("正在检查已上传分片...");

            // 获取已上传的分片信息
            var uploadedChunks = await GetUploadedChunksAsync(fileId, cancellationToken);

            // 准备上传任务
            var currentChunkSize = _initialChunkSize;
            var fileSize = fileInfo.Length;

            var progressLock = new object();
            long totalUploaded = uploadedChunks.Values.Sum();
            long totalToUpload = fileSize - totalUploaded;

            StatusChanged?.Invoke($"需要上传 {totalToUpload} 字节");

            var parallelOptions = new ParallelOptions
            {
                MaxDegreeOfParallelism = _maxParallelism,
                CancellationToken = cancellationToken
            };


            await Parallel.ForEachAsync(GenerateChunks(fileSize, currentChunkSize, uploadedChunks), parallelOptions, async (chunk, cancellationToken) =>
            {
                cancellationToken.ThrowIfCancellationRequested();

                bool success = await UploadChunkAsync(fileId, chunk.Index, chunk.Offset, chunk.Size, fileSize, fileInfo.Name, fileStream, cancellationToken);

                if (success)
                {
                    // 使用锁保证进度更新的原子性
                    lock (progressLock)
                    {
                        totalUploaded += chunk.Size;
                        int progress = (int)((double)totalUploaded / fileSize * 100);
                        ProgressChanged?.Invoke(progress);
                    }
                }
            });

            ProgressChanged?.Invoke(100);
            await MergeChunksAsync(fileId, fileInfo.Name, cancellationToken);

            StatusChanged?.Invoke("上传完成！");
            UploadCompleted?.Invoke(true);
        }
        catch (OperationCanceledException)
        {
            StatusChanged?.Invoke("上传已取消");
            UploadCompleted?.Invoke(false);
        }
        catch (Exception ex)
        {
            StatusChanged?.Invoke($"上传失败: {ex.Message}");
            UploadCompleted?.Invoke(false);
        }
    }

    // 获取已上传的分片信息
    private async Task<ConcurrentDictionary<long, int>> GetUploadedChunksAsync(string fileId, CancellationToken cancellationToken)
    {
        try
        {
            var response = await _httpClient.GetAsync($"{_serviceUri}{GetUploadStatusApiUri}?fileId={fileId}", cancellationToken);
            response.EnsureSuccessStatusCode();

            var result = await response.Content.ReadFromJsonAsync<UploadStatusResponse>(cancellationToken);
            return new ConcurrentDictionary<long, int>(result.UploadedChunks.ToDictionary(x => x.Offset, x => x.Size));
        }
        catch
        {
            return new ConcurrentDictionary<long, int>();
        }
    }

    // 上传单个分片
    private async Task<bool> UploadChunkAsync(string fileId, int chunkIndex, long chunkOffset, int chunkSize, long fileSize,  string fileName, Stream fileStream, CancellationToken cancellationToken)
    {
        var stopwatch = Stopwatch.StartNew();

        try
        {
            // 测量请求延迟
            var latencyStopwatch = Stopwatch.StartNew();
            var pingResponse = await _httpClient.GetAsync($"{_serviceUri}{PingApiUri}", cancellationToken);
            latencyStopwatch.Stop();

            // 准备分片数据
            var chunkData = new byte[chunkSize];
            fileStream.Seek(chunkOffset, SeekOrigin.Begin);
            var bytesRead = await fileStream.ReadAsync(chunkData, 0, chunkSize, cancellationToken);

            if (bytesRead == 0) return false;

            using var content = new MultipartFormDataContent
            {
                { new StringContent(fileId), "fileId" },
                { new StringContent(chunkIndex.ToString()), "chunkIndex" },
                { new StringContent(chunkOffset.ToString()), "chunkOffset" },                   
                { new StringContent(bytesRead.ToString()), "chunkSize"},
                { new StringContent(fileSize.ToString()), "fileSize"},
                { new StringContent(fileName), "fileName" },
                { new ByteArrayContent(chunkData, 0, bytesRead), "chunk", "chunk.dat" }
            };

            // 上传
            var uploadStopwatch = Stopwatch.StartNew();
            var response = await _httpClient.PostAsync($"{_serviceUri}{UploadChunkApiUri}", content, cancellationToken);
            uploadStopwatch.Stop();

            if (!response.IsSuccessStatusCode)
            {
                var error = await response.Content.ReadAsStringAsync(cancellationToken);
                throw new HttpRequestException(error);
            }

            // 更新网络指标
            UpdateNetworkMetrics(bytesRead, uploadStopwatch.Elapsed, latencyStopwatch.Elapsed);
            return true;
        }
        finally
        {
            stopwatch.Stop();
        }
    }

    // 上传完所有分片后合并分片
    private async Task MergeChunksAsync(string fileId, string fileName, CancellationToken cancellationToken)
    {
        var stringContent = new StringContent(JsonSerializer.Serialize(new { FileId = fileId, FileName = fileName }), Encoding.UTF8, "application/json");
        var response = await _httpClient.PostAsync($"{_serviceUri}{MergeChunksApiUri}", stringContent, cancellationToken);

        if (!response.IsSuccessStatusCode)
        {
            var error = await response.Content.ReadAsStringAsync(cancellationToken);
            throw new HttpRequestException(error);
        }
    }

    private static string GetFileHash(Stream stream, HashAlgorithmType hashAlgorithmType)
    {
        switch (hashAlgorithmType)
        {
            case HashAlgorithmType.Md5:
                using (var hashAlgorithm = MD5.Create())
                {
                    var hash = hashAlgorithm.ComputeHash(stream);
                    return BitConverter.ToString(hash).Replace("-", "").ToLower();
                }
            case HashAlgorithmType.Sha256:
                using (var hashAlgorithm = SHA256.Create())
                {
                    var hash = hashAlgorithm.ComputeHash(stream);
                    return BitConverter.ToString(hash).Replace("-", "").ToLower();
                }
            default:
                return string.Empty;
        }
    }

    #region 更新网络指标并动态计算当前分片大小
    // 动态计算当前分片大小
    private int CalculateDynamicChunkSize()
    {
        lock (_networkLock)
        {
            // 基于当前上传速度和延迟计算理想分片大小
            double targetUploadTimeSec = 2.0; // 目标每个分片上传时间
            double speedFactor = _averageUploadSpeedMbps* 125000; // Mbps -> bytes/sec
            double latencyFactor = Math.Max(1, _averageLatencyMs / 50);

            int idealSize = (int)(speedFactor * targetUploadTimeSec / latencyFactor);

            // 限制在最小和最大值之间
            return (int)Math.Clamp(idealSize, _minChunkSize, _maxChunkSize);
        }
    }

    // 更新网络指标
    private void UpdateNetworkMetrics(long chunkSize, TimeSpan uploadTime, TimeSpan latency)
    {
        lock (_networkLock)
        {
            // 计算速度 (Mbps)
            double speedMbps = (chunkSize * 8 / uploadTime.TotalSeconds) / 1_000_000;
            // 平滑处理速度值（加权平均）
            _averageUploadSpeedMbps = 0.7 * _averageUploadSpeedMbps + 0.3 * speedMbps;
            // 更新延迟
            _averageLatencyMs = 0.8 * _averageLatencyMs + 0.2 * latency.TotalMilliseconds;

            StatusChanged?.Invoke($"网络: {_averageUploadSpeedMbps:F1}Mbps, " +
                                $"延迟: {_averageLatencyMs:F0}ms, " +
                                $"分片: {CalculateDynamicChunkSize() / 1024}KB");
        }
    }

    private IEnumerable<FileChunk> GenerateChunks(long fileSize, int currentChunkSize, ConcurrentDictionary<long, int> uploadedChunks)
    {
        long position = 0;
        int chunkIndex = 0;

        while (position < fileSize)
        {
            _semaphoreSlim.WaitAsync(); // 使用异步锁确保分片信息更新时的原子性
            if (!uploadedChunks.TryGetValue(position, out var chunkSize))
            {
                // 动态调整分片大小
                currentChunkSize = CalculateDynamicChunkSize();
                var actualChunkSize = (int)Math.Min(currentChunkSize, fileSize - position);

                // 惰性求值: 迭代器代码直到开始遍历才会执行，每次迭代时返回一个值，并保持当前执行状态(局部变量、执行位置等)
                yield return new FileChunk
                {
                    Index = chunkIndex,
                    Offset = position,
                    Size = actualChunkSize,
                    RemainingSize = fileSize - (position + actualChunkSize)
                };

                position += actualChunkSize;
            }
            else
            {
                position += chunkSize;
            }

            chunkIndex++;

            _semaphoreSlim.Release();
        }
    }
    #endregion


    #region record
    private record UploadedChunk
    {
        public long Offset { get; init; }
        public int Size { get; init; }
    }

    private record UploadStatusResponse
    {
        public List<UploadedChunk> UploadedChunks { get; init; }
    }

    private record FileChunk
    {
        public int Index { get; init; }
        public long Offset { get; init; }
        public int Size { get; init; }
        public long RemainingSize { get; init; }
    }
    #endregion
}
```


最终实现的效果如下：

{% asset_img adaptive_resumable_upload_in_winform.png  客户端断点续传演示 %}

### 完整性校验

完整性校验包含两个方面：

- 分片完整性校验：校验单个分片的完整性，用于检测网络传输错误(一般使用MD5校验)
- 文件完整性校验：校验合并后最终文件的完整性，确保内容安全(建议使用SHA-256校验)

在计算哈希值时应当仅读取文件内容，避免隐藏的元数据差异(文件创建/修改日期等)造成哈希值不同。

#### 分片完整校验

以下为服务端UploadChunkAsync接口对于分片完整校验的部分代码：

```c#
[HttpPost]
[AllowAnonymous]
public async Task<IActionResult> UploadChunkAsync(CancellationToken cancellationToken)
{
    ...
    var chunkHash = form["chunkHash"].ToString();
    ...

    ...
    using var chunkStream = System.IO.File.OpenRead(chunkPath);
    var revievedChunkHash = GetFileHash(chunkStream, HashAlgorithmType.Md5);
    if (!chunkHash.Equals(revievedChunkHash))
    {
        chunkStream.Close();
        System.IO.File.Delete(chunkPath);
        var errorMsg = $"分片完整性校验失败，原始分片哈希为{chunkHash}, 接收到的分片哈希为{revievedChunkHash}, 分片 {chunkIndex} 已被删除，请重新上传！";
        _logger.LogError(errorMsg);
        return BadRequest(errorMsg);
    }
    ...
}
```

以下为客户端UploadChunkAsync方法对于分片完整性校验的部分代码：

```c#
private async Task<bool> UploadChunkAsync(string fileId, int chunkIndex, long chunkOffset, int chunkSize, long fileSize,  string fileName, Stream fileStream, CancellationToken cancellationToken)
{
    ...
    var chunkData = new byte[chunkSize];
    fileStream.Seek(chunkOffset, SeekOrigin.Begin);
    var bytesRead = await fileStream.ReadAsync(chunkData, 0, chunkSize, cancellationToken);
    using var stream = new MemoryStream(chunkData);
    var chunkHash = GetFileHash(stream, HashAlgorithmType.Md5);
    ...

    ...
    using var content = new MultipartFormDataContent
    {
        ...
        { new StringContent(chunkHash.ToString()), "chunkHash"},
        ...
    };
    ...
}
```

#### 文件完整性校验

以下为服务端MergeChunksAsync接口中对于文件完整性校验的部分代码：

```c#
[HttpPost]
[AllowAnonymous]
public async Task<IActionResult> MergeChunksAsync(MergeFile mergeFile, CancellationToken cancellationToken)
{
    ...
    using var stream = System.IO.File.OpenRead(finalPath);
    var mergeFileId = GetFileHash(stream, HashAlgorithmType.Sha256);
    if (!mergeFile.FileId.Equals(mergeFileId))
    {
        stream.Close();
        System.IO.File.Delete(finalPath);
        var errorMsg = $"合并后的文件内容哈希不正确，文件可能已损坏，合并前内容哈希为{mergeFile.FileId}, 合并后内容哈希为{mergeFileId}, 合并后的文件已被删除，请重新上传！";
        _logger.LogError(errorMsg);
        return BadRequest(errorMsg);
    }
    ...
}
```

### 重传尝试

分片上传过程中存在各种不稳定的因素(如网络波动)，可以为客户端上传服务类添加重传尝试机制，在上传失败后自动重新上传。

```c#
public class AdaptiveResumableUploadService
{
    ...
    private readonly int _maxRetryCount;

    public AdaptiveResumableUploadService(
    ...,
    int maxRetryCount = 10) 
    {
        ...
        _maxRetryCount = maxRetryCount;
    }

    // 主上传方法
    public async Task UploadFileAdaptiveAsync(string filePath, CancellationToken cancellationToken)
    {
        try
        {
            StatusChanged?.Invoke("正在准备上传...");

            var fileInfo = new FileInfo(filePath);
            using var fileStream = File.OpenRead(filePath);
            var fileId = GetFileHash(fileStream, HashAlgorithmType.Sha256); // 基于文件内容生成唯一ID

            StatusChanged?.Invoke("正在检查已上传分片...");

            // 准备上传任务
            var currentChunkSize = _initialChunkSize;
            var fileSize = fileInfo.Length;

            var progressLock = new object();

            var parallelOptions = new ParallelOptions
            {
                MaxDegreeOfParallelism = _maxParallelism,
                CancellationToken = cancellationToken
            };

            var retryCount = 0;
            var completed = false;
            while (!completed && retryCount < _maxRetryCount)
            {
                // 获取已上传的分片信息
                var uploadedChunks = await GetUploadedChunksAsync(fileId, cancellationToken);
                var totalUploaded = uploadedChunks.Values.Sum();
                long totalToUpload = fileSize - totalUploaded;
                StatusChanged?.Invoke($"需要上传 {totalToUpload} 字节");
                try
                {
                    await Parallel.ForEachAsync(GenerateChunks(fileSize, currentChunkSize, uploadedChunks), parallelOptions, async (chunk, cancellationToken) =>
                    {
                        cancellationToken.ThrowIfCancellationRequested();

                        bool success = await UploadChunkAsync(fileId, chunk.Index, chunk.Offset, chunk.Size, fileSize, fileInfo.Name, fileStream, cancellationToken);

                        if (success)
                        {
                            // 使用锁保证进度更新的原子性
                            lock (progressLock)
                            {
                                totalUploaded += chunk.Size;
                                int progress = (int)((double)totalUploaded / fileSize * 100);
                                ProgressChanged?.Invoke(progress);
                            }
                        }
                    });

                    await MergeChunksAsync(fileId, fileInfo.Name, cancellationToken);
                    completed = true;
                    ProgressChanged?.Invoke(100);
                    StatusChanged?.Invoke("上传完成！");
                    UploadCompleted?.Invoke(true);
                }
                catch
                {
                    retryCount++;
                    if (retryCount >= _maxRetryCount) throw;
                    await Task.Delay(100 * retryCount, cancellationToken);
                }
            }
        }
        catch (OperationCanceledException)
        {
            StatusChanged?.Invoke("上传已取消");
            UploadCompleted?.Invoke(false);
        }
        catch (Exception ex)
        {
            StatusChanged?.Invoke($"上传失败: {ex.Message}");
            UploadCompleted?.Invoke(false);
        }
    }
}
```

### 分片定时清理

断点续传留下的分片会随着时间在服务器上越积越多，因此需要有一个分片定时清理策略。

首先在服务端创建一个后台任务配置类，用于读取appsettings.json中的后台任务配置项，并在Program.cs中注册配置项，

```c#
public class BackgroundJobOptions
{
    public int StartHour { get; set; }
    public int StartMinute { get; set; }
    public int StartSecond { get; set; }
    public int IntervalMinute { get; set; }
    public int CleanUpDaysAgo { get; set; }
}
```

```c#
// 注册配置类
builder.Services.Configure<BackgroundJobOptions>(builder.Configuration.GetSection("BackgroundJobOptions"));
```

appsettings.json中的配置项如下所示。

```json
{
  "BackgroundJobOptions": {
    "StartHour": 10,
    "StartMinute": 30,
    "StartSecond": 0,
    "IntervalMinute": 1440,
    "CleanUpDaysAgo": 7
  }
}

```

然后定义IWorkServie和WorkService，作为分片定时清理服务类。

```c#
public interface IWorkService
{
    Task TaskWorkAsync(CancellationToken cancellationToken);
}
```

```c#
public class WorkService : IWorkService
{
    private int executionCount = 0;
    private readonly ILogger<WorkService> _logger;
    private DateTime nextDateTime;
    private readonly IWebHostEnvironment _env;
    private readonly BackgroundJobOptions _backgroundJobOptions;

    public WorkService(ILogger<WorkService> logger, IWebHostEnvironment env, IOptions<BackgroundJobOptions> backgroundJobOptions)
    {
        _logger = logger;
        _env = env;
        _backgroundJobOptions = backgroundJobOptions.Value;
    }

    public async Task TaskWorkAsync(CancellationToken cancellationToken)
    {
        while (!cancellationToken.IsCancellationRequested)
        {
            // 计算下一个时间节点
            var now = DateTime.Now;
            var firstDateTime = new DateTime(now.Year, now.Month, now.Day, _backgroundJobOptions.StartHour, _backgroundJobOptions.StartMinute, _backgroundJobOptions.StartSecond);
            if (executionCount == 0)
            {
                nextDateTime = firstDateTime;
            }
            else
            {
                nextDateTime = nextDateTime.AddMinutes(_backgroundJobOptions.IntervalMinute);
            }

            if (nextDateTime < now)
            {
                var delay = nextDateTime.AddDays(1) - now;
                await Task.Delay(delay, cancellationToken);
            }
            else
            {
                var delay = nextDateTime - now;
                await Task.Delay(delay, cancellationToken);
                    CleanupOldUploads(_backgroundJobOptions.CleanUpDaysAgo);

                var count = Interlocked.Increment(ref executionCount);
                _logger.LogInformation("已完成分片自动清理. 累计清理次数: {Count}", count);
            }

        }
    }

    private void CleanupOldUploads(int daysAgo)
    {
        var cutoff = DateTime.Now.AddDays(-daysAgo);
        var uploadPath = Path.Combine(_env.WebRootPath, "uploads");
        foreach (var dir in Directory.GetDirectories(uploadPath))
        {
            var dirName = dir.Split('\\').Last();
            if (!"completed".Equals(dirName) && Directory.GetCreationTime(dir) < cutoff)
            {
                Directory.Delete(dir, true);
            }
        }
    }
}
```

然后创建一个后台服务类UploadCleanupService继承BackgroundService，并调用WorkService中的分片定时清理服务。

```c#
public class UploadCleanupService : BackgroundService
{
    private readonly IServiceProvider _services;

    public UploadCleanupService(IServiceProvider services)
    {
        _services = services;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var scope = _services.CreateScope();
        //获取服务类
        var taskWorkService = scope.ServiceProvider.GetRequiredService<IWorkService>();
        //执行服务类的定时任务
        await taskWorkService.TaskWorkAsync(stoppingToken);
    }
}
```

最后在Program.cs中注册后台服务。

```c#
// 注册后台服务
builder.Services.AddHostedService<UploadCleanupService>();
```

## 断点续传下载

与断点续传上传类似，断点续传下载同样包含下载进度反馈、下载取消、并行下载等辅助功能。

下面我们来简单实现上述功能。

### 断点续传/下载进度反馈/下载取消

#### 服务端实现

由于断点续传上传中使用了wwwroot静态资源文件夹，服务端的文件可直接通过url访问。确保Program.cs中启用了静态文件服务。

```c#
// 启用静态文件服务
app.UseStaticFiles(); 
```

#### 客户端实现

首先创建一个ResumableDownloadService服务类，根据传入的服务器文件地址，使用Range头请求从断点处继续下载(确保服务器支持Range请求头)，并提供进度条反馈和取消下载功能。

```c#
public class ResumableDownloadService
{
    private readonly HttpClient _httpClient;

    // 进度和状态事件
    public event Action<int> ProgressChanged; // 上传百分比
    public event Action<string> StatusChanged; // 状态消息
    public event Action<bool> DownloadCompleted; // 完成状态

    public ResumableDownloadService()
    {
        _httpClient = new HttpClient();
    }

    public async Task DownloadWithResumeAsync(string fileUrl, string destDir, CancellationToken cancellationToken)
    {
        try
        {
            // 检查目标文件是否已存在部分下载内容
            long existingLength = 0;
            var destFile = fileUrl.Split("/").Last();
            var destinationPath = $"{destDir}\\{destFile}";
            if (!Directory.Exists(destDir))
            {
                Directory.CreateDirectory(destDir);
            }

            if (File.Exists(destinationPath))
            {
                existingLength = new FileInfo(destinationPath).Length;
            }

            // 使用 Range 头请求从断点处继续下载
            using var request = new HttpRequestMessage(HttpMethod.Get, fileUrl);
            if (existingLength > 0)
            {
                request.Headers.Range = new RangeHeaderValue(existingLength, null);
            }

            using var response = await _httpClient.SendAsync(request, HttpCompletionOption.ResponseHeadersRead, cancellationToken);
            if (!response.IsSuccessStatusCode)
            {
                if (response.StatusCode == System.Net.HttpStatusCode.RequestedRangeNotSatisfiable)
                {
                    StatusChanged?.Invoke("下载完成！"); // Range头请求范围超出文件大小，视为下载完成
                    ProgressChanged?.Invoke(100);
                    DownloadCompleted?.Invoke(true);
                    return;
                }
            }

            var fileTotalLength = response.Content.Headers.ContentLength;

            using var contentStream = await response.Content.ReadAsStreamAsync(cancellationToken);
            using var fileStream = new FileStream(destinationPath, existingLength > 0 ? FileMode.Append : FileMode.Create, FileAccess.Write, FileShare.None);
            var buffer = new byte[8192];
            int bytesRead;
            long totalRead = 0;

            while ((bytesRead = await contentStream.ReadAsync(buffer, 0, buffer.Length, cancellationToken)) > 0)
            {
                await fileStream.WriteAsync(buffer, 0, bytesRead, cancellationToken);
                totalRead += bytesRead;

                int progress = (int)((double)totalRead / fileTotalLength * 100);
                ProgressChanged?.Invoke(progress);
                StatusChanged?.Invoke($"当前上传进度: {progress}%");
            }

            StatusChanged?.Invoke("下载完成！");
            DownloadCompleted?.Invoke(true);
        }
        catch (OperationCanceledException)
        {
            StatusChanged?.Invoke("下载已取消");
            DownloadCompleted?.Invoke(false);
        }
        catch (Exception ex)
        {
            StatusChanged?.Invoke($"上传失败: {ex.Message}");
            DownloadCompleted?.Invoke(false);
        }
    }
}
```

与上传类似，主窗体MainForm中的代码如下。

```c#
public partial class MainForm : Form
{
    private readonly string _serviceUri;
    private readonly ResumableDownloadService _downloader;
    private CancellationTokenSource _cts;

    public MainForm()
    {
        InitializeComponent();

        _serviceUri = "http://localhost:5001";
        _downloader = new ResumableDownloadService();

        // 绑定事件
        _downloader.ProgressChanged += UpdateProgress;
        _downloader.StatusChanged += UpdateStatus;
        _downloader.DownloadCompleted += UploadCompleted;
    }

    private void UpdateProgress(int percent)
    {
        if (progressBar.InvokeRequired)
        {
            progressBar.Invoke(new Action<int>(UpdateProgress), percent);
            return;
        }
        progressBar.Value = percent;
    }

    private void UpdateStatus(string message)
    {
        if (lblStatus.InvokeRequired)
        {
            lblStatus.Invoke(new Action<string>(UpdateStatus), message);
            return;
        }
        lblStatus.Text = message;
    }

    private void UploadCompleted(bool success)
    {
        if (btnDownload.InvokeRequired)
        {
            btnDownload.Invoke(new Action<bool>(UploadCompleted), success);
            return;
        }
        btnDownload.Enabled = true;
        btnCancel.Enabled = false;
    }

    private async void btnDownload_Click(object sender, EventArgs e)
    {
        if (string.IsNullOrEmpty(textBox.Text) || !Directory.Exists(textBox.Text))
        {
            MessageBox.Show("请选择有效目录！");
            return;
        }

        btnDownload.Enabled = false;
        btnCancel.Enabled = true;
        progressBar.Value = 0;

        _cts = new CancellationTokenSource();
        string fileUrl = $"{_serviceUri}/uploads/completed/test.mp4";
        string destDir = textBox.Text;
        await _downloader.DownloadWithResumeAsync(fileUrl, destDir, _cts.Token);
    }

    private void btnBrowse_Click(object sender, EventArgs e)
    {
        using var folderBrowserDialog = new FolderBrowserDialog();
        // 设置对话框标题
        folderBrowserDialog.Description = "请选择保存文件的目录";
        // 设置初始目录（可选）
        folderBrowserDialog.SelectedPath = Environment.GetFolderPath(Environment.SpecialFolder.MyDocuments);

        // 显示对话框
        if (folderBrowserDialog.ShowDialog() == DialogResult.OK)
        {
            // 获取用户选择的目录路径
            textBox.Text = folderBrowserDialog.SelectedPath;
        }
    }

    private void btnCancel_Click(object sender, EventArgs e)
    {
        _cts?.Cancel();
    }

    protected override void OnFormClosing(FormClosingEventArgs e)
    {
        _cts?.Cancel();
        base.OnFormClosing(e);
    }
}
```

## 参考文档

- [阿里云OSS断点续传上传操作指南](https://help.aliyun.com/zh/oss/user-guide/resumable-upload)

- [阿里云OSS断点续传下载操作指南](https://help.aliyun.com/zh/oss/user-guide/oss-resumable-download)

- [Control.InvokeRequired属性参考文档](https://learn.microsoft.com/zh-cn/dotnet/api/system.windows.forms.control.invokerequired)