# Part 98: Cloud & Azure Integration
## ขั้นตอนที่ 971-980: Cloud และ Azure

---

## บทนำ

ในยุคปัจจุบัน Cloud Computing กลายเป็นส่วนสำคัญของการพัฒนาซอฟต์แวร์ระดับองค์กร Microsoft Azure เป็นหนึ่งในแพลตฟอร์ม Cloud ชั้นนำที่ให้บริการครอบคลุมตั้งแต่ Infrastructure จนถึง AI Services ในส่วนนี้เราจะเรียนรู้การใช้งาน Azure ร่วมกับ .NET อย่างครบครัน ตั้งแต่การจัดการ Secrets, Blob Storage, Service Bus ไปจนถึง Cognitive Services และการวางแผนด้าน Cost Optimization

---

## ขั้นตอนที่ 971: Azure SDK for .NET

### ภาพรวม Azure SDK

Azure SDK for .NET คือชุดไลบรารีที่ช่วยให้นักพัฒนาสามารถใช้งานบริการต่างๆ ของ Azure ได้อย่างสะดวก โดยมีการออกแบบที่สอดคล้องกัน (consistent design) และรองรับ async/await อย่างสมบูรณ์

### หลักการสำคัญของ Azure SDK

1. **DefaultAzureCredential** - วิธีการ authenticate ที่แนะนำ รองรับหลาย identity provider
2. **Managed Identity** - การ authenticate โดยไม่ต้องจัดการ credentials เอง
3. **Azure Identity Library** - ไลบรารีกลางสำหรับ authentication

### การติดตั้ง NuGet Packages

```xml
<!-- ใน .csproj -->
<PackageReference Include="Azure.Identity" Version="1.12.0" />
<PackageReference Include="Azure.Storage.Blobs" Version="12.22.0" />
<PackageReference Include="Azure.Messaging.ServiceBus" Version="7.18.1" />
<PackageReference Include="Azure.Security.KeyVault.Secrets" Version="4.6.0" />
<PackageReference Include="Azure.AI.FormRecognizer" Version="4.1.0" />
<PackageReference Include="Microsoft.Azure.Functions.Worker" Version="1.23.0" />
<PackageReference Include="Microsoft.ApplicationInsights.AspNetCore" Version="2.22.0" />
```

### DefaultAzureCredential

`DefaultAzureCredential` เป็น credential chain ที่ลองวิธี authenticate ตามลำดับ:
1. EnvironmentCredential (Environment Variables)
2. WorkloadIdentityCredential (Kubernetes)
3. ManagedIdentityCredential (Azure Services)
4. VisualStudioCredential (Development)
5. AzureCliCredential (CLI)
6. AzurePowerShellCredential
7. InteractiveBrowserCredential

```csharp
using Azure.Identity;
using Azure.Storage.Blobs;

// =====================================================
// Step 971: Azure SDK Overview และ Authentication
// =====================================================

public class AzureAuthenticationDemo
{
    // DefaultAzureCredential - วิธีที่แนะนำสำหรับ Production
    public static BlobServiceClient CreateBlobClientWithDefaultCredential(string accountUrl)
    {
        // ใช้งาน DefaultAzureCredential - จะลองหลายวิธีตามลำดับ
        var credential = new DefaultAzureCredential();
        return new BlobServiceClient(new Uri(accountUrl), credential);
    }

    // Managed Identity - สำหรับ Azure Services
    public static BlobServiceClient CreateBlobClientWithManagedIdentity(
        string accountUrl,
        string? userAssignedClientId = null)
    {
        ManagedIdentityCredential credential;

        if (userAssignedClientId != null)
        {
            // User-Assigned Managed Identity
            credential = new ManagedIdentityCredential(userAssignedClientId);
        }
        else
        {
            // System-Assigned Managed Identity
            credential = new ManagedIdentityCredential();
        }

        return new BlobServiceClient(new Uri(accountUrl), credential);
    }

    // ClientSecretCredential - สำหรับ Service Principal
    public static BlobServiceClient CreateBlobClientWithServicePrincipal(
        string tenantId,
        string clientId,
        string clientSecret,
        string accountUrl)
    {
        var credential = new ClientSecretCredential(tenantId, clientId, clientSecret);
        return new BlobServiceClient(new Uri(accountUrl), credential);
    }

    // EnvironmentCredential - อ่านจาก Environment Variables
    // AZURE_TENANT_ID, AZURE_CLIENT_ID, AZURE_CLIENT_SECRET
    public static BlobServiceClient CreateBlobClientFromEnvironment(string accountUrl)
    {
        var credential = new EnvironmentCredential();
        return new BlobServiceClient(new Uri(accountUrl), credential);
    }
}

// การตั้งค่า Dependency Injection สำหรับ Azure Services
public static class AzureServiceRegistration
{
    public static IServiceCollection AddAzureServices(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        // ลงทะเบียน DefaultAzureCredential
        services.AddSingleton<TokenCredential>(new DefaultAzureCredential(
            new DefaultAzureCredentialOptions
            {
                // ระบุ Managed Identity Client ID (ถ้าใช้ User-Assigned)
                ManagedIdentityClientId = configuration["Azure:ManagedIdentityClientId"],
                // ข้ามการลองบาง credential ใน Development
                ExcludeSharedTokenCacheCredential = true
            }
        ));

        return services;
    }
}
```

### Azure SDK Retry Policy

```csharp
using Azure.Core;
using Azure.Core.Pipeline;

public class AzureRetryConfiguration
{
    public static BlobServiceClient CreateClientWithCustomRetry(string connectionString)
    {
        var options = new BlobClientOptions
        {
            Retry =
            {
                MaxRetries = 5,
                Delay = TimeSpan.FromSeconds(2),
                MaxDelay = TimeSpan.FromSeconds(30),
                NetworkTimeout = TimeSpan.FromSeconds(100),
                Mode = RetryMode.Exponential
            }
        };

        return new BlobServiceClient(connectionString, options);
    }
}
```

---

## ขั้นตอนที่ 972: Azure Blob Storage

### ภาพรวม Azure Blob Storage

Azure Blob Storage คือบริการจัดเก็บข้อมูลแบบ Object Storage ที่รองรับข้อมูลไม่มีโครงสร้าง (unstructured data) เช่น ไฟล์รูปภาพ วิดีโอ เอกสาร และข้อมูล Binary อื่นๆ

### ประเภทของ Blob
- **Block Blob** - สำหรับข้อความและ binary data ทั่วไป
- **Append Blob** - สำหรับ log files ที่เพิ่มข้อมูลเรื่อยๆ
- **Page Blob** - สำหรับ Virtual Machine disks

```csharp
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Models;
using Azure.Storage.Blobs.Specialized;
using Azure.Storage.Sas;
using System.Text;

// =====================================================
// Step 972: Azure Blob Storage Operations
// =====================================================

public class BlobStorageService
{
    private readonly BlobServiceClient _blobServiceClient;
    private readonly ILogger<BlobStorageService> _logger;

    public BlobStorageService(
        BlobServiceClient blobServiceClient,
        ILogger<BlobStorageService> logger)
    {
        _blobServiceClient = blobServiceClient;
        _logger = logger;
    }

    // อัปโหลดไฟล์จาก Stream
    public async Task<string> UploadFromStreamAsync(
        string containerName,
        string blobName,
        Stream content,
        string contentType = "application/octet-stream",
        CancellationToken cancellationToken = default)
    {
        var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);

        // สร้าง container ถ้ายังไม่มี
        await containerClient.CreateIfNotExistsAsync(
            PublicAccessType.None,
            cancellationToken: cancellationToken);

        var blobClient = containerClient.GetBlobClient(blobName);

        var uploadOptions = new BlobUploadOptions
        {
            HttpHeaders = new BlobHttpHeaders
            {
                ContentType = contentType
            },
            // Metadata เพิ่มเติม
            Metadata = new Dictionary<string, string>
            {
                ["uploadedAt"] = DateTime.UtcNow.ToString("O"),
                ["environment"] = Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT") ?? "Production"
            },
            // ใช้ Transfer Options สำหรับไฟล์ขนาดใหญ่
            TransferOptions = new StorageTransferOptions
            {
                MaximumConcurrency = 8,
                MaximumTransferSize = 4 * 1024 * 1024 // 4MB per chunk
            }
        };

        var response = await blobClient.UploadAsync(content, uploadOptions, cancellationToken);
        _logger.LogInformation("อัปโหลดสำเร็จ: {BlobName}, ETag: {ETag}",
            blobName, response.Value.ETag);

        return blobClient.Uri.ToString();
    }

    // อัปโหลดจาก byte array
    public async Task<Uri> UploadBytesAsync(
        string containerName,
        string blobName,
        byte[] data,
        string contentType,
        CancellationToken cancellationToken = default)
    {
        using var stream = new MemoryStream(data);
        await UploadFromStreamAsync(containerName, blobName, stream, contentType, cancellationToken);

        var blobClient = _blobServiceClient
            .GetBlobContainerClient(containerName)
            .GetBlobClient(blobName);

        return blobClient.Uri;
    }

    // ดาวน์โหลดไฟล์
    public async Task<Stream> DownloadAsync(
        string containerName,
        string blobName,
        CancellationToken cancellationToken = default)
    {
        var blobClient = _blobServiceClient
            .GetBlobContainerClient(containerName)
            .GetBlobClient(blobName);

        if (!await blobClient.ExistsAsync(cancellationToken))
        {
            throw new FileNotFoundException($"ไม่พบไฟล์: {containerName}/{blobName}");
        }

        var response = await blobClient.DownloadStreamingAsync(cancellationToken: cancellationToken);
        return response.Value.Content;
    }

    // ดาวน์โหลดพร้อม Metadata
    public async Task<BlobDownloadResult> DownloadWithMetadataAsync(
        string containerName,
        string blobName,
        CancellationToken cancellationToken = default)
    {
        var blobClient = _blobServiceClient
            .GetBlobContainerClient(containerName)
            .GetBlobClient(blobName);

        var response = await blobClient.DownloadContentAsync(cancellationToken);
        return response.Value;
    }

    // สร้าง SAS Token สำหรับการเข้าถึงชั่วคราว
    public Uri GenerateSasToken(
        string containerName,
        string blobName,
        TimeSpan validity,
        BlobSasPermissions permissions = BlobSasPermissions.Read)
    {
        var blobClient = _blobServiceClient
            .GetBlobContainerClient(containerName)
            .GetBlobClient(blobName);

        // ตรวจสอบว่า client มี credential ที่รองรับ SAS
        if (!blobClient.CanGenerateSasUri)
        {
            throw new InvalidOperationException(
                "BlobClient ต้องใช้ StorageSharedKeyCredential เพื่อสร้าง SAS Token");
        }

        var sasBuilder = new BlobSasBuilder
        {
            BlobContainerName = containerName,
            BlobName = blobName,
            Resource = "b", // "b" = blob
            ExpiresOn = DateTimeOffset.UtcNow.Add(validity)
        };

        sasBuilder.SetPermissions(permissions);

        return blobClient.GenerateSasUri(sasBuilder);
    }

    // สร้าง SAS Token สำหรับทั้ง Container
    public Uri GenerateContainerSasToken(
        string containerName,
        TimeSpan validity,
        BlobContainerSasPermissions permissions = BlobContainerSasPermissions.Read)
    {
        var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);

        if (!containerClient.CanGenerateSasUri)
        {
            throw new InvalidOperationException("ต้องใช้ StorageSharedKeyCredential");
        }

        var sasBuilder = new BlobSasBuilder
        {
            BlobContainerName = containerName,
            Resource = "c", // "c" = container
            ExpiresOn = DateTimeOffset.UtcNow.Add(validity)
        };

        sasBuilder.SetPermissions(permissions);

        return containerClient.GenerateSasUri(sasBuilder);
    }

    // ลิสต์ไฟล์ใน Container
    public async IAsyncEnumerable<BlobItem> ListBlobsAsync(
        string containerName,
        string? prefix = null,
        [System.Runtime.CompilerServices.EnumeratorCancellation]
        CancellationToken cancellationToken = default)
    {
        var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);

        await foreach (var blobItem in containerClient.GetBlobsAsync(
            prefix: prefix,
            cancellationToken: cancellationToken))
        {
            yield return blobItem;
        }
    }

    // ลบไฟล์
    public async Task<bool> DeleteAsync(
        string containerName,
        string blobName,
        CancellationToken cancellationToken = default)
    {
        var blobClient = _blobServiceClient
            .GetBlobContainerClient(containerName)
            .GetBlobClient(blobName);

        return await blobClient.DeleteIfExistsAsync(
            DeleteSnapshotsOption.IncludeSnapshots,
            cancellationToken: cancellationToken);
    }

    // Copy Blob ระหว่าง Containers
    public async Task CopyBlobAsync(
        string sourceContainer,
        string sourceBlobName,
        string destContainer,
        string destBlobName,
        CancellationToken cancellationToken = default)
    {
        var sourceBlobClient = _blobServiceClient
            .GetBlobContainerClient(sourceContainer)
            .GetBlobClient(sourceBlobName);

        var destContainerClient = _blobServiceClient.GetBlobContainerClient(destContainer);
        await destContainerClient.CreateIfNotExistsAsync(cancellationToken: cancellationToken);

        var destBlobClient = destContainerClient.GetBlobClient(destBlobName);

        // เริ่ม Copy Operation
        var copyOperation = await destBlobClient.StartCopyFromUriAsync(
            sourceBlobClient.Uri,
            cancellationToken: cancellationToken);

        // รอจนกว่าจะ Copy เสร็จ
        await copyOperation.WaitForCompletionAsync(cancellationToken);

        _logger.LogInformation("คัดลอก Blob สำเร็จ: {Source} -> {Dest}",
            sourceBlobName, destBlobName);
    }
}

// Stream Upload แบบ Chunked สำหรับไฟล์ขนาดใหญ่
public class LargeFileUploadService
{
    private readonly BlobServiceClient _blobServiceClient;

    public LargeFileUploadService(BlobServiceClient blobServiceClient)
    {
        _blobServiceClient = blobServiceClient;
    }

    public async Task UploadLargeFileAsync(
        string containerName,
        string blobName,
        string localFilePath,
        IProgress<long>? progress = null,
        CancellationToken cancellationToken = default)
    {
        var blobClient = _blobServiceClient
            .GetBlobContainerClient(containerName)
            .GetBlobClient(blobName);

        using var fileStream = File.OpenRead(localFilePath);

        var options = new BlobUploadOptions
        {
            ProgressHandler = progress,
            TransferOptions = new StorageTransferOptions
            {
                // อัปโหลดแบบ Parallel
                MaximumConcurrency = 4,
                // ขนาด Chunk ต่อครั้ง (8MB)
                MaximumTransferSize = 8 * 1024 * 1024,
                // ขนาดขั้นต่ำในการทำ Parallel Upload (256KB)
                InitialTransferSize = 256 * 1024
            }
        };

        await blobClient.UploadAsync(fileStream, options, cancellationToken);
    }
}
```

---

## ขั้นตอนที่ 973: Azure Service Bus

### ภาพรวม Azure Service Bus

Azure Service Bus คือ Enterprise Message Broker ที่รองรับ Queue และ Topic/Subscription pattern ใช้สำหรับการสื่อสารระหว่าง Microservices อย่างน่าเชื่อถือ

### แนวคิดหลัก
- **Queue** - FIFO messaging ระหว่างผู้ส่งและผู้รับ
- **Topic/Subscription** - Publish/Subscribe pattern
- **Dead Letter Queue** - เก็บ messages ที่ไม่สามารถ process ได้
- **Sessions** - การจัดกลุ่ม messages

```csharp
using Azure.Messaging.ServiceBus;
using System.Text.Json;

// =====================================================
// Step 973: Azure Service Bus
// =====================================================

// Message Models
public record OrderCreatedEvent(
    string OrderId,
    string CustomerId,
    decimal TotalAmount,
    DateTime CreatedAt);

public record OrderProcessingResult(
    string OrderId,
    bool Success,
    string? ErrorMessage);

// Service Bus Sender Service
public class ServiceBusSenderService : IAsyncDisposable
{
    private readonly ServiceBusClient _client;
    private readonly ServiceBusSender _sender;
    private readonly ILogger<ServiceBusSenderService> _logger;

    public ServiceBusSenderService(
        string connectionString,
        string queueOrTopicName,
        ILogger<ServiceBusSenderService> logger)
    {
        _client = new ServiceBusClient(connectionString, new ServiceBusClientOptions
        {
            TransportType = ServiceBusTransportType.AmqpWebSockets
        });
        _sender = _client.CreateSender(queueOrTopicName);
        _logger = logger;
    }

    // ส่ง Message เดียว
    public async Task SendMessageAsync<T>(
        T payload,
        string? sessionId = null,
        string? correlationId = null,
        Dictionary<string, object>? properties = null,
        CancellationToken cancellationToken = default) where T : class
    {
        var json = JsonSerializer.Serialize(payload);
        var message = new ServiceBusMessage(json)
        {
            ContentType = "application/json",
            MessageId = Guid.NewGuid().ToString(),
            CorrelationId = correlationId,
            SessionId = sessionId
        };

        // เพิ่ม custom properties
        if (properties != null)
        {
            foreach (var (key, value) in properties)
            {
                message.ApplicationProperties[key] = value;
            }
        }

        // เพิ่ม type information
        message.ApplicationProperties["MessageType"] = typeof(T).Name;
        message.ApplicationProperties["SentAt"] = DateTime.UtcNow.ToString("O");

        await _sender.SendMessageAsync(message, cancellationToken);
        _logger.LogInformation("ส่ง message สำเร็จ: {MessageId}, Type: {Type}",
            message.MessageId, typeof(T).Name);
    }

    // ส่ง Messages หลายอันพร้อมกัน (Batch)
    public async Task SendMessageBatchAsync<T>(
        IEnumerable<T> payloads,
        CancellationToken cancellationToken = default) where T : class
    {
        using var messageBatch = await _sender.CreateMessageBatchAsync(cancellationToken);

        foreach (var payload in payloads)
        {
            var json = JsonSerializer.Serialize(payload);
            var message = new ServiceBusMessage(json)
            {
                ContentType = "application/json",
                MessageId = Guid.NewGuid().ToString()
            };
            message.ApplicationProperties["MessageType"] = typeof(T).Name;

            if (!messageBatch.TryAddMessage(message))
            {
                // Batch เต็ม ส่ง batch ปัจจุบันก่อน
                await _sender.SendMessagesAsync(messageBatch, cancellationToken);
                _logger.LogInformation("ส่ง batch สำเร็จ");
            }
        }

        // ส่ง messages ที่เหลือ
        if (messageBatch.Count > 0)
        {
            await _sender.SendMessagesAsync(messageBatch, cancellationToken);
        }
    }

    // Scheduled Message - ส่งในเวลาที่กำหนด
    public async Task<long> ScheduleMessageAsync<T>(
        T payload,
        DateTimeOffset scheduledTime,
        CancellationToken cancellationToken = default) where T : class
    {
        var json = JsonSerializer.Serialize(payload);
        var message = new ServiceBusMessage(json)
        {
            ContentType = "application/json",
            MessageId = Guid.NewGuid().ToString()
        };

        var sequenceNumber = await _sender.ScheduleMessageAsync(
            message, scheduledTime, cancellationToken);

        _logger.LogInformation("จัดตาราง message ไว้ที่: {Time}, SequenceNumber: {Seq}",
            scheduledTime, sequenceNumber);

        return sequenceNumber;
    }

    // ยกเลิก Scheduled Message
    public async Task CancelScheduledMessageAsync(
        long sequenceNumber,
        CancellationToken cancellationToken = default)
    {
        await _sender.CancelScheduledMessageAsync(sequenceNumber, cancellationToken);
    }

    public async ValueTask DisposeAsync()
    {
        await _sender.DisposeAsync();
        await _client.DisposeAsync();
    }
}

// Service Bus Processor - รับ Messages
public class OrderProcessorService : IHostedService, IAsyncDisposable
{
    private readonly ServiceBusClient _client;
    private ServiceBusProcessor? _processor;
    private readonly ILogger<OrderProcessorService> _logger;

    public OrderProcessorService(
        string connectionString,
        string queueName,
        ILogger<OrderProcessorService> logger)
    {
        _client = new ServiceBusClient(connectionString);
        _logger = logger;

        // สร้าง Processor สำหรับ Queue
        _processor = _client.CreateProcessor(queueName, new ServiceBusProcessorOptions
        {
            MaxConcurrentCalls = 5,           // จำนวน messages ที่ process พร้อมกัน
            AutoCompleteMessages = false,      // จัดการ complete เอง
            MaxAutoLockRenewalDuration = TimeSpan.FromMinutes(5),
            PrefetchCount = 10                // Pre-fetch messages เพื่อประสิทธิภาพ
        });

        _processor.ProcessMessageAsync += HandleMessageAsync;
        _processor.ProcessErrorAsync += HandleErrorAsync;
    }

    public async Task StartAsync(CancellationToken cancellationToken)
    {
        await _processor!.StartProcessingAsync(cancellationToken);
        _logger.LogInformation("Service Bus Processor เริ่มทำงาน");
    }

    public async Task StopAsync(CancellationToken cancellationToken)
    {
        await _processor!.StopProcessingAsync(cancellationToken);
        _logger.LogInformation("Service Bus Processor หยุดทำงาน");
    }

    private async Task HandleMessageAsync(ProcessMessageEventArgs args)
    {
        try
        {
            var body = args.Message.Body.ToString();
            _logger.LogInformation("ได้รับ message: {MessageId}", args.Message.MessageId);

            // Deserialize message
            var order = JsonSerializer.Deserialize<OrderCreatedEvent>(body);
            if (order == null)
            {
                await args.DeadLetterMessageAsync(
                    args.Message,
                    "InvalidPayload",
                    "ไม่สามารถ deserialize message ได้");
                return;
            }

            // ประมวลผล Order
            await ProcessOrderAsync(order, args.CancellationToken);

            // Acknowledge message
            await args.CompleteMessageAsync(args.Message);
            _logger.LogInformation("ประมวลผล Order {OrderId} สำเร็จ", order.OrderId);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "เกิดข้อผิดพลาดในการประมวลผล message {MessageId}",
                args.Message.MessageId);

            // ตรวจสอบจำนวนครั้งที่ลองใหม่
            if (args.Message.DeliveryCount >= 3)
            {
                // ส่งไป Dead Letter Queue
                await args.DeadLetterMessageAsync(
                    args.Message,
                    "MaxRetriesExceeded",
                    ex.Message);
            }
            else
            {
                // ปล่อยให้ retry
                await args.AbandonMessageAsync(args.Message);
            }
        }
    }

    private async Task ProcessOrderAsync(
        OrderCreatedEvent order,
        CancellationToken cancellationToken)
    {
        // Logic การประมวลผล Order
        _logger.LogInformation(
            "กำลังประมวลผล Order: {OrderId}, จำนวนเงิน: {Amount}",
            order.OrderId,
            order.TotalAmount);

        await Task.Delay(100, cancellationToken); // จำลองงาน
    }

    private Task HandleErrorAsync(ProcessErrorEventArgs args)
    {
        _logger.LogError(args.Exception,
            "เกิดข้อผิดพลาดใน Service Bus Processor: {Source}",
            args.ErrorSource);
        return Task.CompletedTask;
    }

    public async ValueTask DisposeAsync()
    {
        if (_processor != null)
        {
            await _processor.DisposeAsync();
        }
        await _client.DisposeAsync();
    }
}

// Dead Letter Queue Processor
public class DeadLetterQueueProcessor
{
    private readonly ServiceBusClient _client;

    public DeadLetterQueueProcessor(string connectionString)
    {
        _client = new ServiceBusClient(connectionString);
    }

    public async Task ProcessDeadLetterMessagesAsync(
        string queueName,
        CancellationToken cancellationToken = default)
    {
        // สร้าง receiver สำหรับ Dead Letter Queue
        var dlqPath = ServiceBusReceiverOptions.GetDeadLetterQueueName(queueName);
        var receiver = _client.CreateReceiver(
            queueName,
            new ServiceBusReceiverOptions
            {
                SubQueue = SubQueue.DeadLetter
            });

        // รับ messages จาก DLQ
        var messages = await receiver.ReceiveMessagesAsync(
            maxMessages: 100,
            maxWaitTime: TimeSpan.FromSeconds(5),
            cancellationToken: cancellationToken);

        foreach (var message in messages)
        {
            Console.WriteLine($"DLQ Message: {message.MessageId}");
            Console.WriteLine($"Dead Letter Reason: {message.DeadLetterReason}");
            Console.WriteLine($"Body: {message.Body}");

            // ลบออกจาก DLQ หลังจาก log
            await receiver.CompleteMessageAsync(message, cancellationToken);
        }

        await receiver.DisposeAsync();
    }
}
```

---

## ขั้นตอนที่ 974: Azure Functions v4 กับ .NET

### ภาพรวม Azure Functions

Azure Functions คือ Serverless compute service ที่ให้เรา run code โดยไม่ต้องจัดการ infrastructure โมเดล v4 ใน .NET มีการปรับปรุงสำคัญได้แก่ isolated worker model

```csharp
// =====================================================
// Step 974: Azure Functions v4 with .NET
// =====================================================

// ไฟล์ Program.cs สำหรับ Azure Functions v4
// host.json และ local.settings.json ตั้งค่าผ่าน IHostBuilder

using Microsoft.Azure.Functions.Worker;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

// Program.cs
var host = new HostBuilder()
    .ConfigureFunctionsWebApplication()
    .ConfigureServices(services =>
    {
        services.AddApplicationInsightsTelemetryWorkerService();
        services.ConfigureFunctionsApplicationInsights();

        // ลงทะเบียน Services
        services.AddSingleton(new BlobServiceClient(
            Environment.GetEnvironmentVariable("AzureWebJobsStorage")));
        services.AddScoped<IOrderService, OrderService>();
    })
    .Build();

await host.RunAsync();

// HTTP Trigger Function
public class HttpTriggerFunction
{
    private readonly IOrderService _orderService;
    private readonly ILogger<HttpTriggerFunction> _logger;

    public HttpTriggerFunction(
        IOrderService orderService,
        ILogger<HttpTriggerFunction> logger)
    {
        _orderService = orderService;
        _logger = logger;
    }

    [Function("CreateOrder")]
    public async Task<IActionResult> CreateOrderAsync(
        [HttpTrigger(AuthorizationLevel.Function, "post", Route = "orders")] HttpRequest req,
        FunctionContext context)
    {
        _logger.LogInformation("รับคำขอสร้าง Order");

        // อ่าน Body
        string requestBody;
        using (var reader = new StreamReader(req.Body))
        {
            requestBody = await reader.ReadToEndAsync();
        }

        var createOrderDto = JsonSerializer.Deserialize<CreateOrderDto>(requestBody);
        if (createOrderDto == null)
        {
            return new BadRequestObjectResult("ข้อมูลไม่ถูกต้อง");
        }

        var order = await _orderService.CreateOrderAsync(createOrderDto);
        return new OkObjectResult(order);
    }

    [Function("GetOrder")]
    public async Task<IActionResult> GetOrderAsync(
        [HttpTrigger(AuthorizationLevel.Function, "get", Route = "orders/{id}")] HttpRequest req,
        string id,
        FunctionContext context)
    {
        var order = await _orderService.GetOrderAsync(id);
        if (order == null)
        {
            return new NotFoundResult();
        }

        return new OkObjectResult(order);
    }
}

// Timer Trigger Function - ทำงานตามกำหนดเวลา
public class TimerTriggerFunction
{
    private readonly ILogger<TimerTriggerFunction> _logger;
    private readonly IOrderService _orderService;

    public TimerTriggerFunction(
        ILogger<TimerTriggerFunction> logger,
        IOrderService orderService)
    {
        _logger = logger;
        _orderService = orderService;
    }

    // ทำงานทุกวันเวลา 02:00 UTC
    [Function("DailyOrderCleanup")]
    public async Task RunDailyCleanupAsync(
        [TimerTrigger("0 0 2 * * *")] TimerInfo timerInfo,
        FunctionContext context)
    {
        _logger.LogInformation("เริ่ม Daily Order Cleanup: {Time}", DateTime.UtcNow);

        if (timerInfo.ScheduleStatus?.Last != null)
        {
            _logger.LogInformation("ครั้งที่แล้ว: {LastRun}",
                timerInfo.ScheduleStatus.Last);
        }

        var cleanedCount = await _orderService.CleanOldOrdersAsync(
            olderThan: TimeSpan.FromDays(90));

        _logger.LogInformation("ลบ orders เก่า {Count} รายการ", cleanedCount);
    }

    // ทำงานทุก 5 นาที
    [Function("HealthCheck")]
    public void RunHealthCheck(
        [TimerTrigger("0 */5 * * * *")] TimerInfo timerInfo,
        FunctionContext context)
    {
        _logger.LogInformation("Health Check: {Time}", DateTime.UtcNow);
    }
}

// Service Bus Trigger Function
public class ServiceBusTriggerFunction
{
    private readonly ILogger<ServiceBusTriggerFunction> _logger;
    private readonly IOrderProcessor _orderProcessor;

    public ServiceBusTriggerFunction(
        ILogger<ServiceBusTriggerFunction> logger,
        IOrderProcessor orderProcessor)
    {
        _logger = logger;
        _orderProcessor = orderProcessor;
    }

    [Function("ProcessOrderMessage")]
    public async Task ProcessOrderAsync(
        [ServiceBusTrigger("orders-queue",
            Connection = "ServiceBusConnection")] ServiceBusReceivedMessage message,
        ServiceBusMessageActions messageActions,
        FunctionContext context)
    {
        _logger.LogInformation("ได้รับ message: {MessageId}", message.MessageId);

        try
        {
            var order = JsonSerializer.Deserialize<OrderCreatedEvent>(
                message.Body.ToString());

            if (order == null)
            {
                await messageActions.DeadLetterMessageAsync(
                    message,
                    deadLetterReason: "InvalidPayload");
                return;
            }

            await _orderProcessor.ProcessAsync(order);
            await messageActions.CompleteMessageAsync(message);

            _logger.LogInformation("ประมวลผล Order {OrderId} สำเร็จ", order.OrderId);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "เกิดข้อผิดพลาด: {Message}", ex.Message);
            await messageActions.AbandonMessageAsync(message);
        }
    }

    // Topic Subscription Trigger
    [Function("ProcessTopicMessage")]
    public async Task ProcessTopicMessageAsync(
        [ServiceBusTrigger(
            "orders-topic",
            "high-priority-subscription",
            Connection = "ServiceBusConnection")] ServiceBusReceivedMessage message,
        ServiceBusMessageActions messageActions)
    {
        _logger.LogInformation("ได้รับจาก Topic: {MessageId}", message.MessageId);

        var payload = message.Body.ToString();
        // ประมวลผล...

        await messageActions.CompleteMessageAsync(message);
    }
}

// Blob Trigger Function
public class BlobTriggerFunction
{
    private readonly ILogger<BlobTriggerFunction> _logger;

    public BlobTriggerFunction(ILogger<BlobTriggerFunction> logger)
    {
        _logger = logger;
    }

    [Function("ProcessUploadedImage")]
    public async Task ProcessImageAsync(
        [BlobTrigger("images/{name}", Connection = "AzureWebJobsStorage")] Stream imageStream,
        string name,
        FunctionContext context)
    {
        _logger.LogInformation("ประมวลผลรูปภาพ: {Name}, ขนาด: {Size} bytes",
            name, imageStream.Length);

        // ประมวลผลรูปภาพ...
        await Task.Delay(100);

        _logger.LogInformation("ประมวลผลรูปภาพ {Name} เสร็จสิ้น", name);
    }
}

// Interfaces ที่ใช้ใน Functions
public interface IOrderService
{
    Task<OrderCreatedEvent> CreateOrderAsync(CreateOrderDto dto);
    Task<OrderCreatedEvent?> GetOrderAsync(string id);
    Task<int> CleanOldOrdersAsync(TimeSpan olderThan);
}

public interface IOrderProcessor
{
    Task ProcessAsync(OrderCreatedEvent order);
}

public record CreateOrderDto(
    string CustomerId,
    List<OrderItem> Items);

public record OrderItem(
    string ProductId,
    int Quantity,
    decimal UnitPrice);

// Placeholder implementations
public class OrderService : IOrderService
{
    public Task<OrderCreatedEvent> CreateOrderAsync(CreateOrderDto dto)
        => Task.FromResult(new OrderCreatedEvent(
            Guid.NewGuid().ToString(),
            dto.CustomerId,
            dto.Items.Sum(i => i.Quantity * i.UnitPrice),
            DateTime.UtcNow));

    public Task<OrderCreatedEvent?> GetOrderAsync(string id)
        => Task.FromResult<OrderCreatedEvent?>(null);

    public Task<int> CleanOldOrdersAsync(TimeSpan olderThan)
        => Task.FromResult(0);
}
```

---

## ขั้นตอนที่ 975: Azure Application Insights

### ภาพรวม Application Insights

Application Insights คือบริการ APM (Application Performance Monitoring) ที่ช่วยติดตาม performance, ตรวจหา anomalies และวิเคราะห์ behavior ของแอปพลิเคชัน

```csharp
using Microsoft.ApplicationInsights;
using Microsoft.ApplicationInsights.DataContracts;
using Microsoft.ApplicationInsights.Extensibility;
using System.Diagnostics;

// =====================================================
// Step 975: Azure Application Insights
// =====================================================

// การตั้งค่าใน Program.cs
// builder.Services.AddApplicationInsightsTelemetry(configuration["ApplicationInsights:ConnectionString"]);

public class ApplicationInsightsTelemetryService
{
    private readonly TelemetryClient _telemetryClient;
    private readonly ILogger<ApplicationInsightsTelemetryService> _logger;

    public ApplicationInsightsTelemetryService(
        TelemetryClient telemetryClient,
        ILogger<ApplicationInsightsTelemetryService> logger)
    {
        _telemetryClient = telemetryClient;
        _logger = logger;
    }

    // ส่ง Custom Event
    public void TrackCustomEvent(
        string eventName,
        Dictionary<string, string>? properties = null,
        Dictionary<string, double>? metrics = null)
    {
        _telemetryClient.TrackEvent(eventName, properties, metrics);
        _logger.LogInformation("ส่ง event: {EventName}", eventName);
    }

    // ส่ง Custom Metric
    public void TrackMetric(string metricName, double value, IDictionary<string, string>? properties = null)
    {
        _telemetryClient.TrackMetric(metricName, value, properties);
    }

    // ส่ง Custom Exception
    public void TrackException(
        Exception exception,
        Dictionary<string, string>? properties = null)
    {
        _telemetryClient.TrackException(exception, properties);
    }

    // Track การเรียก Dependency (Database, External API)
    public async Task<T> TrackDependencyAsync<T>(
        string dependencyType,
        string dependencyName,
        string data,
        Func<Task<T>> operation)
    {
        var startTime = DateTimeOffset.UtcNow;
        var stopwatch = Stopwatch.StartNew();
        var success = false;

        try
        {
            var result = await operation();
            success = true;
            return result;
        }
        finally
        {
            stopwatch.Stop();

            var dependency = new DependencyTelemetry(
                dependencyType: dependencyType,
                target: dependencyName,
                dependencyName: data,
                data: data,
                startTime: startTime,
                duration: stopwatch.Elapsed,
                resultCode: success ? "200" : "500",
                success: success);

            _telemetryClient.TrackDependency(dependency);
        }
    }

    // Track HTTP Request แบบ Custom
    public async Task<T> TrackRequestAsync<T>(
        string requestName,
        string url,
        Func<Task<T>> operation)
    {
        var requestTelemetry = new RequestTelemetry
        {
            Name = requestName,
            Url = new Uri(url),
            Timestamp = DateTimeOffset.UtcNow
        };

        var stopwatch = Stopwatch.StartNew();

        try
        {
            var result = await operation();
            requestTelemetry.ResponseCode = "200";
            requestTelemetry.Success = true;
            return result;
        }
        catch (Exception ex)
        {
            requestTelemetry.ResponseCode = "500";
            requestTelemetry.Success = false;
            _telemetryClient.TrackException(ex);
            throw;
        }
        finally
        {
            stopwatch.Stop();
            requestTelemetry.Duration = stopwatch.Elapsed;
            _telemetryClient.TrackRequest(requestTelemetry);
        }
    }
}

// Custom Telemetry Initializer - เพิ่ม Property ให้ทุก Telemetry
public class CustomTelemetryInitializer : ITelemetryInitializer
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public CustomTelemetryInitializer(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }

    public void Initialize(ITelemetry telemetry)
    {
        if (telemetry is not ISupportProperties properties)
            return;

        // เพิ่ม User ID จาก HTTP Context
        var userId = _httpContextAccessor.HttpContext?.User?.FindFirst("sub")?.Value;
        if (!string.IsNullOrEmpty(userId))
        {
            properties.Properties["UserId"] = userId;
        }

        // เพิ่ม Application Version
        properties.Properties["ApplicationVersion"] =
            System.Reflection.Assembly.GetExecutingAssembly()
                .GetName().Version?.ToString() ?? "Unknown";

        // เพิ่ม Environment
        properties.Properties["Environment"] =
            Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT") ?? "Production";
    }
}

// Custom Metric เพื่อ Monitor Business KPIs
public class BusinessMetricsService
{
    private readonly TelemetryClient _telemetryClient;

    public BusinessMetricsService(TelemetryClient telemetryClient)
    {
        _telemetryClient = telemetryClient;
    }

    // บันทึก Order Revenue
    public void TrackOrderRevenue(decimal amount, string currency)
    {
        _telemetryClient.TrackMetric("OrderRevenue", (double)amount,
            new Dictionary<string, string>
            {
                ["Currency"] = currency
            });
    }

    // บันทึก Active Users
    public void TrackActiveUsers(int count)
    {
        _telemetryClient.TrackMetric("ActiveUsers", count);
    }

    // บันทึก Order Completion Rate
    public void TrackOrderCompletion(bool success, string failureReason = "")
    {
        var properties = new Dictionary<string, string>
        {
            ["Status"] = success ? "Success" : "Failed"
        };

        if (!success && !string.IsNullOrEmpty(failureReason))
        {
            properties["FailureReason"] = failureReason;
        }

        _telemetryClient.TrackEvent("OrderCompleted", properties);
    }
}

// Middleware สำหรับ Log Request/Response
public class TelemetryMiddleware
{
    private readonly RequestDelegate _next;
    private readonly TelemetryClient _telemetryClient;

    public TelemetryMiddleware(
        RequestDelegate next,
        TelemetryClient telemetryClient)
    {
        _next = next;
        _telemetryClient = telemetryClient;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        using var operation = _telemetryClient.StartOperation<RequestTelemetry>(
            context.Request.Path);

        operation.Telemetry.Properties["HttpMethod"] = context.Request.Method;
        operation.Telemetry.Properties["QueryString"] = context.Request.QueryString.ToString();

        try
        {
            await _next(context);
            operation.Telemetry.ResponseCode = context.Response.StatusCode.ToString();
            operation.Telemetry.Success = context.Response.StatusCode < 400;
        }
        catch (Exception ex)
        {
            _telemetryClient.TrackException(ex);
            operation.Telemetry.Success = false;
            throw;
        }
    }
}
```

---

## ขั้นตอนที่ 976: Azure Key Vault

### ภาพรวม Azure Key Vault

Azure Key Vault ช่วยจัดการ secrets, keys และ certificates อย่างปลอดภัย แทนการเก็บไว้ใน config files หรือ environment variables

```csharp
using Azure.Security.KeyVault.Secrets;
using Azure.Security.KeyVault.Keys;
using Azure.Security.KeyVault.Certificates;

// =====================================================
// Step 976: Azure Key Vault
// =====================================================

public class KeyVaultService
{
    private readonly SecretClient _secretClient;
    private readonly KeyClient _keyClient;
    private readonly CertificateClient _certificateClient;
    private readonly ILogger<KeyVaultService> _logger;

    public KeyVaultService(
        string keyVaultUrl,
        TokenCredential credential,
        ILogger<KeyVaultService> logger)
    {
        var vaultUri = new Uri(keyVaultUrl);
        _secretClient = new SecretClient(vaultUri, credential);
        _keyClient = new KeyClient(vaultUri, credential);
        _certificateClient = new CertificateClient(vaultUri, credential);
        _logger = logger;
    }

    // ดึง Secret
    public async Task<string> GetSecretAsync(
        string secretName,
        string? version = null,
        CancellationToken cancellationToken = default)
    {
        var secret = await _secretClient.GetSecretAsync(
            secretName, version, cancellationToken);

        _logger.LogInformation("ดึง secret สำเร็จ: {SecretName}", secretName);
        return secret.Value.Value;
    }

    // ตั้งค่า Secret
    public async Task SetSecretAsync(
        string secretName,
        string secretValue,
        DateTimeOffset? expiresOn = null,
        Dictionary<string, string>? tags = null,
        CancellationToken cancellationToken = default)
    {
        var properties = new SecretProperties(secretName);

        if (expiresOn.HasValue)
        {
            properties.ExpiresOn = expiresOn;
        }

        if (tags != null)
        {
            foreach (var (key, value) in tags)
            {
                properties.Tags[key] = value;
            }
        }

        var secret = new KeyVaultSecret(secretName, secretValue)
        {
            Properties = { ExpiresOn = expiresOn }
        };

        await _secretClient.SetSecretAsync(secret, cancellationToken);
        _logger.LogInformation("บันทึก secret สำเร็จ: {SecretName}", secretName);
    }

    // ลบ Secret
    public async Task DeleteSecretAsync(
        string secretName,
        CancellationToken cancellationToken = default)
    {
        var deleteOperation = await _secretClient.StartDeleteSecretAsync(
            secretName, cancellationToken);

        await deleteOperation.WaitForCompletionAsync(cancellationToken);
        _logger.LogInformation("ลบ secret สำเร็จ: {SecretName}", secretName);
    }

    // ลิสต์ Secrets ทั้งหมด
    public async IAsyncEnumerable<string> ListSecretNamesAsync(
        [System.Runtime.CompilerServices.EnumeratorCancellation]
        CancellationToken cancellationToken = default)
    {
        await foreach (var secretProperties in _secretClient.GetPropertiesOfSecretsAsync(
            cancellationToken))
        {
            yield return secretProperties.Name;
        }
    }

    // สร้าง RSA Key
    public async Task<KeyVaultKey> CreateRsaKeyAsync(
        string keyName,
        int keySize = 2048,
        CancellationToken cancellationToken = default)
    {
        var options = new CreateRsaKeyOptions(keyName)
        {
            KeySize = keySize,
            KeyOperations =
            {
                KeyOperation.Sign,
                KeyOperation.Verify,
                KeyOperation.Encrypt,
                KeyOperation.Decrypt
            }
        };

        var key = await _keyClient.CreateRsaKeyAsync(options, cancellationToken);
        _logger.LogInformation("สร้าง RSA Key สำเร็จ: {KeyName}", keyName);
        return key.Value;
    }

    // เข้ารหัสและถอดรหัสด้วย Key Vault Key
    public async Task<byte[]> EncryptAsync(
        string keyName,
        byte[] plaintext,
        CancellationToken cancellationToken = default)
    {
        var key = await _keyClient.GetKeyAsync(keyName, cancellationToken: cancellationToken);
        var cryptoClient = new Azure.Security.KeyVault.Keys.Cryptography.CryptographyClient(
            key.Value.Id, _keyClient.Pipeline);

        var result = await cryptoClient.EncryptAsync(
            Azure.Security.KeyVault.Keys.Cryptography.EncryptionAlgorithm.RsaOaep,
            plaintext,
            cancellationToken);

        return result.Ciphertext;
    }

    public async Task<byte[]> DecryptAsync(
        string keyName,
        byte[] ciphertext,
        CancellationToken cancellationToken = default)
    {
        var key = await _keyClient.GetKeyAsync(keyName, cancellationToken: cancellationToken);
        var cryptoClient = new Azure.Security.KeyVault.Keys.Cryptography.CryptographyClient(
            key.Value.Id, _keyClient.Pipeline);

        var result = await cryptoClient.DecryptAsync(
            Azure.Security.KeyVault.Keys.Cryptography.EncryptionAlgorithm.RsaOaep,
            ciphertext,
            cancellationToken);

        return result.Plaintext;
    }
}

// การใช้งาน Key Vault ผ่าน Configuration Provider
// Program.cs
public static class KeyVaultConfigurationExtension
{
    public static WebApplicationBuilder AddKeyVaultConfiguration(
        this WebApplicationBuilder builder,
        string keyVaultUrl)
    {
        var credential = new DefaultAzureCredential();

        // เพิ่ม Key Vault เป็น Configuration Source
        builder.Configuration.AddAzureKeyVault(
            new Uri(keyVaultUrl),
            credential);

        return builder;
    }
}

// Caching Secret เพื่อลด API Calls
public class CachedKeyVaultService
{
    private readonly KeyVaultService _keyVaultService;
    private readonly IMemoryCache _cache;
    private static readonly TimeSpan CacheDuration = TimeSpan.FromMinutes(5);

    public CachedKeyVaultService(
        KeyVaultService keyVaultService,
        IMemoryCache cache)
    {
        _keyVaultService = keyVaultService;
        _cache = cache;
    }

    public async Task<string> GetSecretAsync(
        string secretName,
        CancellationToken cancellationToken = default)
    {
        var cacheKey = $"kv:{secretName}";

        if (_cache.TryGetValue<string>(cacheKey, out var cached) && cached != null)
        {
            return cached;
        }

        var secret = await _keyVaultService.GetSecretAsync(secretName, null, cancellationToken);

        _cache.Set(cacheKey, secret, CacheDuration);
        return secret;
    }

    public void InvalidateCache(string secretName)
    {
        _cache.Remove($"kv:{secretName}");
    }
}
```

---

## ขั้นตอนที่ 977: Azure Container Apps

### ภาพรวม Azure Container Apps

Azure Container Apps คือ Serverless container platform ที่สร้างบน Kubernetes รองรับ Dapr (Distributed Application Runtime) สำหรับ Microservices patterns

```csharp
using Dapr.Client;
using Dapr.Extensions.Configuration;

// =====================================================
// Step 977: Azure Container Apps กับ Dapr
// =====================================================

// การตั้งค่า Dapr ใน Program.cs
// builder.Services.AddDaprClient();
// app.UseRouting();
// app.MapSubscribeHandler();

// Dapr State Management
public class DaprStateService
{
    private readonly DaprClient _daprClient;
    private readonly ILogger<DaprStateService> _logger;
    private const string StateStoreName = "statestore";

    public DaprStateService(DaprClient daprClient, ILogger<DaprStateService> logger)
    {
        _daprClient = daprClient;
        _logger = logger;
    }

    // บันทึก State
    public async Task SaveStateAsync<T>(
        string key,
        T value,
        StateOptions? options = null,
        CancellationToken cancellationToken = default)
    {
        await _daprClient.SaveStateAsync(
            StateStoreName, key, value, options, cancellationToken: cancellationToken);
        _logger.LogInformation("บันทึก state สำเร็จ: {Key}", key);
    }

    // ดึง State
    public async Task<T?> GetStateAsync<T>(
        string key,
        CancellationToken cancellationToken = default)
    {
        var result = await _daprClient.GetStateAsync<T>(
            StateStoreName, key, cancellationToken: cancellationToken);
        return result;
    }

    // ลบ State
    public async Task DeleteStateAsync(
        string key,
        CancellationToken cancellationToken = default)
    {
        await _daprClient.DeleteStateAsync(
            StateStoreName, key, cancellationToken: cancellationToken);
    }

    // Transaction - อัปเดตหลาย State พร้อมกัน
    public async Task ExecuteTransactionAsync(
        IEnumerable<StateTransactionRequest> operations,
        CancellationToken cancellationToken = default)
    {
        await _daprClient.ExecuteStateTransactionAsync(
            StateStoreName,
            operations.ToList(),
            cancellationToken: cancellationToken);
    }
}

// Dapr Pub/Sub
public class DaprPubSubService
{
    private readonly DaprClient _daprClient;
    private readonly ILogger<DaprPubSubService> _logger;
    private const string PubSubName = "pubsub";

    public DaprPubSubService(DaprClient daprClient, ILogger<DaprPubSubService> logger)
    {
        _daprClient = daprClient;
        _logger = logger;
    }

    // Publish Event
    public async Task PublishEventAsync<T>(
        string topicName,
        T eventData,
        CancellationToken cancellationToken = default) where T : class
    {
        await _daprClient.PublishEventAsync(
            PubSubName, topicName, eventData, cancellationToken);
        _logger.LogInformation("Publish event สำเร็จ: {Topic}", topicName);
    }
}

// Dapr Subscribe Controller
[ApiController]
[Route("[controller]")]
public class OrdersController : ControllerBase
{
    private readonly ILogger<OrdersController> _logger;

    public OrdersController(ILogger<OrdersController> logger)
    {
        _logger = logger;
    }

    // Subscribe to Topic
    [Topic("pubsub", "orders")]
    [HttpPost("process")]
    public async Task<ActionResult> ProcessOrderAsync(
        [FromBody] OrderCreatedEvent order)
    {
        _logger.LogInformation("ได้รับ order จาก Dapr Pub/Sub: {OrderId}", order.OrderId);
        // ประมวลผล...
        return Ok();
    }
}

// Dapr Service Invocation
public class DaprServiceInvoker
{
    private readonly DaprClient _daprClient;

    public DaprServiceInvoker(DaprClient daprClient)
    {
        _daprClient = daprClient;
    }

    // เรียก Service อื่นผ่าน Dapr
    public async Task<TResponse?> InvokeServiceAsync<TRequest, TResponse>(
        string appId,
        string methodName,
        TRequest request,
        HttpMethod? httpMethod = null,
        CancellationToken cancellationToken = default)
    {
        var method = httpMethod ?? HttpMethod.Post;

        var response = await _daprClient.InvokeMethodAsync<TRequest, TResponse>(
            method, appId, methodName, request, cancellationToken);

        return response;
    }
}

// containerapp.yaml - Container Apps Manifest (แสดงเป็น comment)
/*
apiVersion: 2024-03-01
name: my-api
resourceGroup: my-resource-group
location: Southeast Asia

properties:
  managedEnvironmentId: /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.App/managedEnvironments/{env}
  configuration:
    ingress:
      external: true
      targetPort: 8080
    dapr:
      enabled: true
      appId: my-api
      appPort: 8080
  template:
    containers:
    - image: myregistry.azurecr.io/my-api:latest
      name: my-api
      resources:
        cpu: 0.5
        memory: 1Gi
      env:
      - name: ASPNETCORE_ENVIRONMENT
        value: Production
    scale:
      minReplicas: 1
      maxReplicas: 10
      rules:
      - name: http-rule
        http:
          metadata:
            concurrentRequests: "100"
*/

// Scale Rules ใน Code (Container Apps SDK)
public class ContainerAppScaleConfiguration
{
    // การกำหนด Scale Rules ผ่าน Bicep/ARM Template
    // หรือ Azure CLI

    public static string GenerateScaleRuleJson(
        int minReplicas,
        int maxReplicas,
        int httpConcurrentRequests = 100,
        string? serviceBusQueueName = null,
        int serviceBusQueueLength = 10)
    {
        var rules = new List<object>
        {
            new
            {
                name = "http-scaling",
                http = new
                {
                    metadata = new
                    {
                        concurrentRequests = httpConcurrentRequests.ToString()
                    }
                }
            }
        };

        if (!string.IsNullOrEmpty(serviceBusQueueName))
        {
            rules.Add(new
            {
                name = "servicebus-scaling",
                custom = new
                {
                    type = "azure-servicebus",
                    metadata = new
                    {
                        queueName = serviceBusQueueName,
                        queueLength = serviceBusQueueLength.ToString()
                    }
                }
            });
        }

        return JsonSerializer.Serialize(new
        {
            minReplicas,
            maxReplicas,
            rules
        }, new JsonSerializerOptions { WriteIndented = true });
    }
}
```

---

## ขั้นตอนที่ 978: Azure Cognitive Services

### ภาพรวม Azure Cognitive Services

Azure Cognitive Services ให้บริการ AI/ML แบบพร้อมใช้งาน รวมถึง Computer Vision, Form Recognizer สำหรับ OCR และการอ่านใบเสร็จภาษาไทย

```csharp
using Azure.AI.Vision.ImageAnalysis;
using Azure.AI.FormRecognizer.DocumentAnalysis;

// =====================================================
// Step 978: Azure Cognitive Services
// =====================================================

// Computer Vision Service
public class ComputerVisionService
{
    private readonly ImageAnalysisClient _client;
    private readonly ILogger<ComputerVisionService> _logger;

    public ComputerVisionService(
        string endpoint,
        string apiKey,
        ILogger<ComputerVisionService> logger)
    {
        _client = new ImageAnalysisClient(
            new Uri(endpoint),
            new Azure.AzureKeyCredential(apiKey));
        _logger = logger;
    }

    // วิเคราะห์รูปภาพ
    public async Task<ImageAnalysisResult> AnalyzeImageAsync(
        Uri imageUrl,
        CancellationToken cancellationToken = default)
    {
        var result = await _client.AnalyzeAsync(
            imageUrl,
            VisualFeatures.Caption |
            VisualFeatures.Objects |
            VisualFeatures.Tags |
            VisualFeatures.People |
            VisualFeatures.Read,
            new ImageAnalysisOptions
            {
                Language = "th", // ภาษาไทย
                GenderNeutralCaption = true
            },
            cancellationToken);

        return result.Value;
    }

    // วิเคราะห์รูปภาพจาก Stream
    public async Task<ImageAnalysisResult> AnalyzeImageFromStreamAsync(
        Stream imageStream,
        CancellationToken cancellationToken = default)
    {
        var result = await _client.AnalyzeAsync(
            BinaryData.FromStream(imageStream),
            VisualFeatures.Caption |
            VisualFeatures.Objects |
            VisualFeatures.Tags |
            VisualFeatures.Read,
            cancellationToken: cancellationToken);

        return result.Value;
    }

    // อ่านข้อความจากรูปภาพ (OCR)
    public async Task<string> ExtractTextFromImageAsync(
        Uri imageUrl,
        CancellationToken cancellationToken = default)
    {
        var result = await _client.AnalyzeAsync(
            imageUrl,
            VisualFeatures.Read,
            cancellationToken: cancellationToken);

        if (result.Value.Read?.Blocks == null)
        {
            return string.Empty;
        }

        var textBuilder = new System.Text.StringBuilder();

        foreach (var block in result.Value.Read.Blocks)
        {
            foreach (var line in block.Lines)
            {
                textBuilder.AppendLine(line.Text);
            }
        }

        return textBuilder.ToString();
    }
}

// Form Recognizer / Document Intelligence - สำหรับใบเสร็จภาษาไทย
public class ReceiptAnalysisService
{
    private readonly DocumentAnalysisClient _client;
    private readonly ILogger<ReceiptAnalysisService> _logger;

    public ReceiptAnalysisService(
        string endpoint,
        string apiKey,
        ILogger<ReceiptAnalysisService> logger)
    {
        _client = new DocumentAnalysisClient(
            new Uri(endpoint),
            new Azure.AzureKeyCredential(apiKey));
        _logger = logger;
    }

    // วิเคราะห์ใบเสร็จ (Thai Receipt OCR)
    public async Task<ThaiReceiptData> AnalyzeThaiReceiptAsync(
        Uri receiptUrl,
        CancellationToken cancellationToken = default)
    {
        _logger.LogInformation("กำลังวิเคราะห์ใบเสร็จ: {Url}", receiptUrl);

        // ใช้ prebuilt-receipt model
        var operation = await _client.AnalyzeDocumentFromUriAsync(
            WaitUntil.Completed,
            "prebuilt-receipt",
            receiptUrl,
            new AnalyzeDocumentOptions
            {
                // รองรับหลายภาษา
                Locale = "th-TH"
            },
            cancellationToken);

        var result = operation.Value;
        var receipt = new ThaiReceiptData();

        if (result.Documents.Count > 0)
        {
            var document = result.Documents[0];

            // ชื่อร้านค้า
            if (document.Fields.TryGetValue("MerchantName", out var merchantName))
            {
                receipt.MerchantName = merchantName.Content;
            }

            // ที่อยู่ร้านค้า
            if (document.Fields.TryGetValue("MerchantAddress", out var merchantAddress))
            {
                receipt.MerchantAddress = merchantAddress.Content;
            }

            // เบอร์โทร
            if (document.Fields.TryGetValue("MerchantPhoneNumber", out var phone))
            {
                receipt.PhoneNumber = phone.Content;
            }

            // วันที่
            if (document.Fields.TryGetValue("TransactionDate", out var transDate))
            {
                if (transDate.Value is DateTimeOffset dateValue)
                {
                    receipt.TransactionDate = dateValue.DateTime;
                }
            }

            // ยอดรวม
            if (document.Fields.TryGetValue("Total", out var total))
            {
                if (total.Value is double totalValue)
                {
                    receipt.Total = (decimal)totalValue;
                }
            }

            // VAT
            if (document.Fields.TryGetValue("TotalTax", out var tax))
            {
                if (tax.Value is double taxValue)
                {
                    receipt.Vat = (decimal)taxValue;
                }
            }

            // รายการสินค้า
            if (document.Fields.TryGetValue("Items", out var items) &&
                items.Value is IReadOnlyList<DocumentField> itemList)
            {
                foreach (var item in itemList)
                {
                    if (item.Value is IReadOnlyDictionary<string, DocumentField> itemFields)
                    {
                        var receiptItem = new ThaiReceiptItem();

                        if (itemFields.TryGetValue("Description", out var desc))
                        {
                            receiptItem.Description = desc.Content ?? "";
                        }

                        if (itemFields.TryGetValue("TotalPrice", out var price) &&
                            price.Value is double priceValue)
                        {
                            receiptItem.TotalPrice = (decimal)priceValue;
                        }

                        if (itemFields.TryGetValue("Quantity", out var qty) &&
                            qty.Value is double qtyValue)
                        {
                            receiptItem.Quantity = (int)qtyValue;
                        }

                        receipt.Items.Add(receiptItem);
                    }
                }
            }
        }

        _logger.LogInformation("วิเคราะห์ใบเสร็จสำเร็จ: ร้าน {Merchant}, ยอดรวม {Total}",
            receipt.MerchantName, receipt.Total);

        return receipt;
    }

    // วิเคราะห์ใบเสร็จจาก Stream
    public async Task<ThaiReceiptData> AnalyzeThaiReceiptFromStreamAsync(
        Stream receiptStream,
        string contentType = "image/jpeg",
        CancellationToken cancellationToken = default)
    {
        var operation = await _client.AnalyzeDocumentAsync(
            WaitUntil.Completed,
            "prebuilt-receipt",
            receiptStream,
            new AnalyzeDocumentOptions { Locale = "th-TH" },
            cancellationToken);

        // ใช้ logic เดียวกับ URL version
        var result = operation.Value;
        return new ThaiReceiptData
        {
            MerchantName = result.Documents.FirstOrDefault()?.Fields
                .GetValueOrDefault("MerchantName")?.Content ?? "ไม่ระบุ"
        };
    }

    // Custom Model - สำหรับเอกสารที่มีรูปแบบเฉพาะ
    public async Task<AnalyzeResult> AnalyzeWithCustomModelAsync(
        string modelId,
        Uri documentUrl,
        CancellationToken cancellationToken = default)
    {
        var operation = await _client.AnalyzeDocumentFromUriAsync(
            WaitUntil.Completed,
            modelId,
            documentUrl,
            cancellationToken: cancellationToken);

        return operation.Value;
    }
}

// Models สำหรับใบเสร็จไทย
public class ThaiReceiptData
{
    public string MerchantName { get; set; } = string.Empty;
    public string MerchantAddress { get; set; } = string.Empty;
    public string PhoneNumber { get; set; } = string.Empty;
    public DateTime? TransactionDate { get; set; }
    public decimal Total { get; set; }
    public decimal Vat { get; set; }
    public decimal SubTotal => Total - Vat;
    public List<ThaiReceiptItem> Items { get; set; } = new();
}

public class ThaiReceiptItem
{
    public string Description { get; set; } = string.Empty;
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
    public decimal TotalPrice { get; set; }
}
```

---

## ขั้นตอนที่ 979: Cost Optimization บน Azure

### ภาพรวม Cost Optimization

การจัดการต้นทุนบน Azure เป็นเรื่องสำคัญสำหรับองค์กร เราต้องเลือกใช้บริการที่เหมาะสมกับ workload และปรับแต่งการใช้งานให้คุ้มค่า

```csharp
// =====================================================
// Step 979: Cost Optimization Strategies
// =====================================================

// 1. Autoscaling Configuration
public class AutoscalingConfiguration
{
    // Scale Rules สำหรับ Container Apps
    public static ScaleConfiguration GetProductionScaleConfig()
    {
        return new ScaleConfiguration
        {
            MinReplicas = 2,        // ขั้นต่ำ 2 replicas เพื่อ High Availability
            MaxReplicas = 20,       // ขั้นสูงสุด
            ScaleOutThreshold = 80, // Scale out เมื่อ CPU > 80%
            ScaleInThreshold = 30,  // Scale in เมื่อ CPU < 30%
            CooldownPeriod = TimeSpan.FromMinutes(5)
        };
    }

    // Scale Rules สำหรับ Non-production
    public static ScaleConfiguration GetDevelopmentScaleConfig()
    {
        return new ScaleConfiguration
        {
            MinReplicas = 0,        // Scale to zero ใน Dev
            MaxReplicas = 3,
            ScaleOutThreshold = 60,
            ScaleInThreshold = 10,
            CooldownPeriod = TimeSpan.FromMinutes(2)
        };
    }
}

public class ScaleConfiguration
{
    public int MinReplicas { get; set; }
    public int MaxReplicas { get; set; }
    public int ScaleOutThreshold { get; set; }
    public int ScaleInThreshold { get; set; }
    public TimeSpan CooldownPeriod { get; set; }
}

// 2. Connection Pooling - ลด overhead ของ connection
public class OptimizedDbContext : DbContext
{
    public OptimizedDbContext(DbContextOptions<OptimizedDbContext> options)
        : base(options) { }

    public DbSet<Order> Orders => Set<Order>();
}

public static class DatabaseServiceExtensions
{
    public static IServiceCollection AddOptimizedDatabase(
        this IServiceCollection services,
        string connectionString)
    {
        services.AddDbContextPool<OptimizedDbContext>(
            options => options.UseSqlServer(connectionString, sqlOptions =>
            {
                // Connection Resiliency
                sqlOptions.EnableRetryOnFailure(
                    maxRetryCount: 3,
                    maxRetryDelay: TimeSpan.FromSeconds(30),
                    errorNumbersToAdd: null);

                // Command Timeout
                sqlOptions.CommandTimeout(30);
            }),
            poolSize: 128  // Connection Pool Size
        );

        return services;
    }
}

// 3. Caching Strategy - ลดการเรียก Database/API ซ้ำๆ
public class CostOptimizedCacheService
{
    private readonly IMemoryCache _memoryCache;
    private readonly IDistributedCache _distributedCache;
    private readonly ILogger<CostOptimizedCacheService> _logger;

    public CostOptimizedCacheService(
        IMemoryCache memoryCache,
        IDistributedCache distributedCache,
        ILogger<CostOptimizedCacheService> logger)
    {
        _memoryCache = memoryCache;
        _distributedCache = distributedCache;
        _logger = logger;
    }

    // Multi-level Cache: Memory -> Redis -> Database
    public async Task<T?> GetOrSetAsync<T>(
        string key,
        Func<Task<T>> factory,
        TimeSpan? localCacheDuration = null,
        TimeSpan? distributedCacheDuration = null,
        CancellationToken cancellationToken = default) where T : class
    {
        // ลองดึงจาก Memory Cache ก่อน (เร็วที่สุด ไม่มีต้นทุน)
        if (_memoryCache.TryGetValue<T>(key, out var memoryCached) && memoryCached != null)
        {
            _logger.LogDebug("Cache hit (memory): {Key}", key);
            return memoryCached;
        }

        // ลองดึงจาก Distributed Cache (Redis)
        var distributedCached = await _distributedCache.GetStringAsync(key, cancellationToken);
        if (distributedCached != null)
        {
            _logger.LogDebug("Cache hit (distributed): {Key}", key);
            var deserialized = JsonSerializer.Deserialize<T>(distributedCached);

            // เก็บใน Memory Cache ด้วย
            var localDuration = localCacheDuration ?? TimeSpan.FromMinutes(5);
            _memoryCache.Set(key, deserialized, localDuration);

            return deserialized;
        }

        // ดึงจาก Source
        _logger.LogDebug("Cache miss: {Key}", key);
        var value = await factory();

        if (value != null)
        {
            var json = JsonSerializer.Serialize(value);

            // เก็บใน Distributed Cache
            var distDuration = distributedCacheDuration ?? TimeSpan.FromHours(1);
            await _distributedCache.SetStringAsync(
                key,
                json,
                new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = distDuration
                },
                cancellationToken);

            // เก็บใน Memory Cache
            var localDuration = localCacheDuration ?? TimeSpan.FromMinutes(5);
            _memoryCache.Set(key, value, localDuration);
        }

        return value;
    }
}

// 4. Batch Processing - ประมวลผลงานเป็น Batch แทนทีละรายการ
public class BatchProcessingService
{
    private readonly ILogger<BatchProcessingService> _logger;

    public BatchProcessingService(ILogger<BatchProcessingService> logger)
    {
        _logger = logger;
    }

    public async Task<List<TResult>> ProcessInBatchesAsync<TInput, TResult>(
        IEnumerable<TInput> items,
        Func<IEnumerable<TInput>, Task<IEnumerable<TResult>>> batchProcessor,
        int batchSize = 100,
        int maxConcurrency = 5,
        CancellationToken cancellationToken = default)
    {
        var allItems = items.ToList();
        var batches = allItems
            .Select((item, index) => new { item, index })
            .GroupBy(x => x.index / batchSize)
            .Select(g => g.Select(x => x.item).ToList())
            .ToList();

        _logger.LogInformation(
            "ประมวลผล {TotalItems} รายการ ใน {BatchCount} batches",
            allItems.Count, batches.Count);

        var semaphore = new SemaphoreSlim(maxConcurrency);
        var results = new List<TResult>();
        var tasks = new List<Task<IEnumerable<TResult>>>();

        foreach (var batch in batches)
        {
            await semaphore.WaitAsync(cancellationToken);

            var task = Task.Run(async () =>
            {
                try
                {
                    return await batchProcessor(batch);
                }
                finally
                {
                    semaphore.Release();
                }
            }, cancellationToken);

            tasks.Add(task);
        }

        var batchResults = await Task.WhenAll(tasks);
        results.AddRange(batchResults.SelectMany(r => r));

        _logger.LogInformation("ประมวลผลเสร็จสิ้น: {ResultCount} ผลลัพธ์", results.Count);
        return results;
    }
}

// 5. Right-sizing ด้วยการ Monitor Resource Usage
public class ResourceMonitoringService
{
    private readonly TelemetryClient _telemetryClient;

    public ResourceMonitoringService(TelemetryClient telemetryClient)
    {
        _telemetryClient = telemetryClient;
    }

    // ติดตาม Memory Usage
    public void TrackMemoryUsage()
    {
        var process = System.Diagnostics.Process.GetCurrentProcess();
        var memoryMb = process.WorkingSet64 / (1024 * 1024);

        _telemetryClient.TrackMetric("MemoryUsageMB", memoryMb);
    }

    // ติดตาม Thread Pool Usage
    public void TrackThreadPoolUsage()
    {
        System.Threading.ThreadPool.GetAvailableThreads(
            out var workerThreads, out var completionPortThreads);
        System.Threading.ThreadPool.GetMaxThreads(
            out var maxWorkerThreads, out var maxCompletionThreads);

        var usedWorkerThreads = maxWorkerThreads - workerThreads;

        _telemetryClient.TrackMetric("ThreadPoolUsage",
            (double)usedWorkerThreads / maxWorkerThreads * 100);
    }
}

// Cost-Aware Storage Tier Selection
public enum StorageTier
{
    Hot,    // เข้าถึงบ่อย - ต้นทุนการเก็บสูง แต่เข้าถึงถูก
    Cool,   // เข้าถึงน้อยครั้ง - เก็บถูกกว่า แต่เข้าถึงแพงกว่า
    Cold,   // เก็บระยะยาว - เก็บถูกมาก
    Archive // Archive - เก็บถูกที่สุด แต่ต้องรอเวลาในการเข้าถึง
}

public class StorageTierOptimizer
{
    public static StorageTier RecommendTier(
        DateTime lastAccessedDate,
        int monthlyAccessCount)
    {
        var daysSinceLastAccess = (DateTime.UtcNow - lastAccessedDate).TotalDays;

        return (daysSinceLastAccess, monthlyAccessCount) switch
        {
            (< 30, > 10) => StorageTier.Hot,
            (< 90, > 1) => StorageTier.Cool,
            (< 365, _) => StorageTier.Cold,
            _ => StorageTier.Archive
        };
    }
}
```

---

## ขั้นตอนที่ 980: Multi-Cloud Considerations

### ภาพรวม Multi-Cloud Strategy

การออกแบบระบบสำหรับ Multi-Cloud ช่วยลด vendor lock-in และเพิ่มความยืดหยุ่น แต่ต้องมีการออกแบบ abstraction layers ที่ดี

```csharp
// =====================================================
// Step 980: Multi-Cloud Abstraction Layers
// =====================================================

// 1. Storage Abstraction Layer
public interface IObjectStorageService
{
    Task<string> UploadAsync(
        string bucketOrContainer,
        string objectKey,
        Stream content,
        string contentType,
        CancellationToken cancellationToken = default);

    Task<Stream> DownloadAsync(
        string bucketOrContainer,
        string objectKey,
        CancellationToken cancellationToken = default);

    Task DeleteAsync(
        string bucketOrContainer,
        string objectKey,
        CancellationToken cancellationToken = default);

    Task<bool> ExistsAsync(
        string bucketOrContainer,
        string objectKey,
        CancellationToken cancellationToken = default);

    Task<Uri> GetPreSignedUrlAsync(
        string bucketOrContainer,
        string objectKey,
        TimeSpan validity,
        CancellationToken cancellationToken = default);
}

// Azure Blob Storage Implementation
public class AzureBlobStorageService : IObjectStorageService
{
    private readonly BlobServiceClient _client;
    private readonly ILogger<AzureBlobStorageService> _logger;

    public AzureBlobStorageService(
        BlobServiceClient client,
        ILogger<AzureBlobStorageService> logger)
    {
        _client = client;
        _logger = logger;
    }

    public async Task<string> UploadAsync(
        string container,
        string blobName,
        Stream content,
        string contentType,
        CancellationToken cancellationToken = default)
    {
        var containerClient = _client.GetBlobContainerClient(container);
        await containerClient.CreateIfNotExistsAsync(cancellationToken: cancellationToken);

        var blobClient = containerClient.GetBlobClient(blobName);
        await blobClient.UploadAsync(content,
            new BlobHttpHeaders { ContentType = contentType },
            cancellationToken: cancellationToken);

        return blobClient.Uri.ToString();
    }

    public async Task<Stream> DownloadAsync(
        string container,
        string blobName,
        CancellationToken cancellationToken = default)
    {
        var blobClient = _client.GetBlobContainerClient(container).GetBlobClient(blobName);
        var response = await blobClient.DownloadStreamingAsync(cancellationToken: cancellationToken);
        return response.Value.Content;
    }

    public async Task DeleteAsync(
        string container,
        string blobName,
        CancellationToken cancellationToken = default)
    {
        var blobClient = _client.GetBlobContainerClient(container).GetBlobClient(blobName);
        await blobClient.DeleteIfExistsAsync(cancellationToken: cancellationToken);
    }

    public async Task<bool> ExistsAsync(
        string container,
        string blobName,
        CancellationToken cancellationToken = default)
    {
        var blobClient = _client.GetBlobContainerClient(container).GetBlobClient(blobName);
        var response = await blobClient.ExistsAsync(cancellationToken);
        return response.Value;
    }

    public Task<Uri> GetPreSignedUrlAsync(
        string container,
        string blobName,
        TimeSpan validity,
        CancellationToken cancellationToken = default)
    {
        var blobClient = _client.GetBlobContainerClient(container).GetBlobClient(blobName);
        var sasUri = blobClient.GenerateSasUri(new BlobSasBuilder
        {
            BlobContainerName = container,
            BlobName = blobName,
            Resource = "b",
            ExpiresOn = DateTimeOffset.UtcNow.Add(validity)
        });

        return Task.FromResult(sasUri);
    }
}

// AWS S3 Implementation (Stub - ต้องติดตั้ง AWSSDK.S3)
public class AwsS3StorageService : IObjectStorageService
{
    // Implementation สำหรับ AWS S3
    // ใช้ AWSSDK.S3 package

    public Task<string> UploadAsync(
        string bucket,
        string key,
        Stream content,
        string contentType,
        CancellationToken cancellationToken = default)
    {
        // Amazon S3 implementation
        // var request = new PutObjectRequest { BucketName = bucket, Key = key, ... };
        throw new NotImplementedException("AWS S3 implementation");
    }

    public Task<Stream> DownloadAsync(
        string bucket,
        string key,
        CancellationToken cancellationToken = default)
    {
        throw new NotImplementedException("AWS S3 implementation");
    }

    public Task DeleteAsync(
        string bucket,
        string key,
        CancellationToken cancellationToken = default)
    {
        throw new NotImplementedException("AWS S3 implementation");
    }

    public Task<bool> ExistsAsync(
        string bucket,
        string key,
        CancellationToken cancellationToken = default)
    {
        throw new NotImplementedException("AWS S3 implementation");
    }

    public Task<Uri> GetPreSignedUrlAsync(
        string bucket,
        string key,
        TimeSpan validity,
        CancellationToken cancellationToken = default)
    {
        throw new NotImplementedException("AWS S3 implementation");
    }
}

// 2. Message Queue Abstraction
public interface IMessageQueueService
{
    Task SendAsync<T>(
        string queueName,
        T message,
        CancellationToken cancellationToken = default) where T : class;

    Task<IMessageHandler<T>> SubscribeAsync<T>(
        string queueName,
        Func<T, CancellationToken, Task> handler,
        CancellationToken cancellationToken = default) where T : class;
}

public interface IMessageHandler<T> : IAsyncDisposable
{
    Task StartAsync(CancellationToken cancellationToken = default);
    Task StopAsync(CancellationToken cancellationToken = default);
}

// Azure Service Bus Implementation
public class AzureServiceBusQueueService : IMessageQueueService
{
    private readonly ServiceBusClient _client;

    public AzureServiceBusQueueService(ServiceBusClient client)
    {
        _client = client;
    }

    public async Task SendAsync<T>(
        string queueName,
        T message,
        CancellationToken cancellationToken = default) where T : class
    {
        var sender = _client.CreateSender(queueName);
        var json = JsonSerializer.Serialize(message);
        await sender.SendMessageAsync(
            new ServiceBusMessage(json) { ContentType = "application/json" },
            cancellationToken);
        await sender.DisposeAsync();
    }

    public Task<IMessageHandler<T>> SubscribeAsync<T>(
        string queueName,
        Func<T, CancellationToken, Task> handler,
        CancellationToken cancellationToken = default) where T : class
    {
        var processor = _client.CreateProcessor(queueName);
        var messageHandler = new ServiceBusMessageHandler<T>(processor, handler);
        return Task.FromResult<IMessageHandler<T>>(messageHandler);
    }
}

public class ServiceBusMessageHandler<T> : IMessageHandler<T> where T : class
{
    private readonly ServiceBusProcessor _processor;
    private readonly Func<T, CancellationToken, Task> _handler;

    public ServiceBusMessageHandler(
        ServiceBusProcessor processor,
        Func<T, CancellationToken, Task> handler)
    {
        _processor = processor;
        _handler = handler;

        _processor.ProcessMessageAsync += HandleMessageAsync;
        _processor.ProcessErrorAsync += HandleErrorAsync;
    }

    public async Task StartAsync(CancellationToken cancellationToken = default)
    {
        await _processor.StartProcessingAsync(cancellationToken);
    }

    public async Task StopAsync(CancellationToken cancellationToken = default)
    {
        await _processor.StopProcessingAsync(cancellationToken);
    }

    private async Task HandleMessageAsync(ProcessMessageEventArgs args)
    {
        var body = args.Message.Body.ToString();
        var message = JsonSerializer.Deserialize<T>(body);
        if (message != null)
        {
            await _handler(message, args.CancellationToken);
            await args.CompleteMessageAsync(args.Message);
        }
    }

    private Task HandleErrorAsync(ProcessErrorEventArgs args)
    {
        return Task.CompletedTask;
    }

    public async ValueTask DisposeAsync()
    {
        await _processor.DisposeAsync();
    }
}

// 3. Secret Management Abstraction
public interface ISecretManager
{
    Task<string> GetSecretAsync(string secretName, CancellationToken cancellationToken = default);
    Task SetSecretAsync(string secretName, string value, CancellationToken cancellationToken = default);
}

// Azure Key Vault Implementation
public class AzureKeyVaultSecretManager : ISecretManager
{
    private readonly SecretClient _secretClient;

    public AzureKeyVaultSecretManager(SecretClient secretClient)
    {
        _secretClient = secretClient;
    }

    public async Task<string> GetSecretAsync(
        string secretName,
        CancellationToken cancellationToken = default)
    {
        var secret = await _secretClient.GetSecretAsync(secretName, cancellationToken: cancellationToken);
        return secret.Value.Value;
    }

    public async Task SetSecretAsync(
        string secretName,
        string value,
        CancellationToken cancellationToken = default)
    {
        await _secretClient.SetSecretAsync(secretName, value, cancellationToken);
    }
}

// Environment Variable Implementation (สำหรับ Local Development)
public class EnvironmentVariableSecretManager : ISecretManager
{
    public Task<string> GetSecretAsync(
        string secretName,
        CancellationToken cancellationToken = default)
    {
        var value = Environment.GetEnvironmentVariable(secretName)
            ?? throw new KeyNotFoundException($"ไม่พบ secret: {secretName}");
        return Task.FromResult(value);
    }

    public Task SetSecretAsync(
        string secretName,
        string value,
        CancellationToken cancellationToken = default)
    {
        Environment.SetEnvironmentVariable(secretName, value);
        return Task.CompletedTask;
    }
}

// 4. Registration Helper สำหรับ Multi-Cloud
public static class MultiCloudServiceRegistration
{
    public enum CloudProvider
    {
        Azure,
        AWS,
        GCP,
        Local
    }

    public static IServiceCollection AddCloudServices(
        this IServiceCollection services,
        IConfiguration configuration,
        CloudProvider provider = CloudProvider.Azure)
    {
        switch (provider)
        {
            case CloudProvider.Azure:
                services.AddAzureCloudServices(configuration);
                break;
            case CloudProvider.Local:
                services.AddLocalCloudServices(configuration);
                break;
            default:
                throw new NotSupportedException($"ยังไม่รองรับ: {provider}");
        }

        return services;
    }

    private static IServiceCollection AddAzureCloudServices(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        var credential = new DefaultAzureCredential();

        services.AddSingleton<TokenCredential>(credential);

        services.AddSingleton(sp =>
            new BlobServiceClient(
                new Uri(configuration["Azure:StorageAccountUrl"]!),
                credential));

        services.AddSingleton(sp =>
            new ServiceBusClient(
                configuration["Azure:ServiceBusConnectionString"]!));

        services.AddSingleton<SecretClient>(sp =>
            new SecretClient(
                new Uri(configuration["Azure:KeyVaultUrl"]!),
                credential));

        services.AddScoped<IObjectStorageService, AzureBlobStorageService>();
        services.AddScoped<IMessageQueueService, AzureServiceBusQueueService>();
        services.AddScoped<ISecretManager, AzureKeyVaultSecretManager>();

        return services;
    }

    private static IServiceCollection AddLocalCloudServices(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        // ใช้ local implementations สำหรับ Development
        services.AddScoped<ISecretManager, EnvironmentVariableSecretManager>();
        // เพิ่ม local storage implementation...
        return services;
    }
}

// 5. Configuration สำหรับ Multi-Cloud
// appsettings.json ตัวอย่าง
/*
{
    "CloudProvider": "Azure",
    "Azure": {
        "StorageAccountUrl": "https://myaccount.blob.core.windows.net",
        "ServiceBusConnectionString": "Endpoint=sb://mybus.servicebus.windows.net/...",
        "KeyVaultUrl": "https://mykeyvault.vault.azure.net",
        "ManagedIdentityClientId": "00000000-0000-0000-0000-000000000000"
    }
}
*/

// 6. Health Check สำหรับ Cloud Services
public class AzureHealthChecks
{
    public static IHealthChecksBuilder AddAzureHealthChecks(
        this IHealthChecksBuilder builder,
        IConfiguration configuration)
    {
        // ตรวจสอบ Blob Storage
        builder.AddAzureBlobStorage(
            configuration["Azure:StorageConnectionString"]!,
            name: "azure-blob-storage",
            failureStatus: Microsoft.Extensions.Diagnostics.HealthChecks.HealthStatus.Degraded);

        // ตรวจสอบ Service Bus
        builder.AddAzureServiceBusQueue(
            configuration["Azure:ServiceBusConnectionString"]!,
            queueName: "orders-queue",
            name: "azure-service-bus");

        return builder;
    }
}
```

---

## สรุปขั้นตอนที่ 971-980

| ขั้นตอน | หัวข้อ | เทคโนโลยีหลัก |
|---------|--------|----------------|
| 971 | Azure SDK Overview | DefaultAzureCredential, Managed Identity |
| 972 | Azure Blob Storage | Upload, Download, SAS Tokens, Stream |
| 973 | Azure Service Bus | Queue, Topic/Subscription, Dead Letter |
| 974 | Azure Functions v4 | HTTP, Timer, Service Bus Triggers |
| 975 | Application Insights | Custom Metrics, Events, Dependencies |
| 976 | Azure Key Vault | Secrets, Certificates, Keys |
| 977 | Azure Container Apps | Dapr Integration, Scale Rules |
| 978 | Cognitive Services | Computer Vision, Form Recognizer, OCR ภาษาไทย |
| 979 | Cost Optimization | Autoscaling, Caching, Batch Processing |
| 980 | Multi-Cloud | Abstraction Layers, Vendor Lock-in Avoidance |

### แนวทางปฏิบัติที่ดีที่สุด (Best Practices)

1. **Authentication** - ใช้ `DefaultAzureCredential` เสมอ หลีกเลี่ยงการเก็บ credentials ใน code
2. **Managed Identity** - ใช้ System-Assigned หรือ User-Assigned Managed Identity สำหรับ Azure Services
3. **Retry Policy** - กำหนด retry strategy ที่เหมาะสมสำหรับ transient failures
4. **Connection Pooling** - ใช้ singleton clients สำหรับ Azure SDK clients
5. **Structured Logging** - ใช้ Application Insights สำหรับ monitoring แบบครบวงจร
6. **Secret Rotation** - ตั้งค่า automatic rotation สำหรับ Key Vault secrets
7. **Cost Monitoring** - ตั้ง budget alerts และ review cost reports เป็นประจำ
8. **Abstraction** - ออกแบบ interfaces ที่ไม่ผูกกับ provider เฉพาะ

---

## แบบฝึกหัด

1. สร้าง Blob Storage service ที่รองรับการ upload รูปภาพพร้อม resize thumbnail
2. ออกแบบ Service Bus topology สำหรับ Order Management System ด้วย Topics และ Subscriptions
3. สร้าง Azure Function ที่รับ webhook จาก payment gateway และบันทึกลง Table Storage
4. ติดตั้ง Application Insights และสร้าง custom dashboard แสดง business metrics
5. สร้างระบบ OCR สำหรับอ่านใบเสร็จภาษาไทยและบันทึกข้อมูลลง database
6. ออกแบบ abstraction layer ที่รองรับทั้ง Azure Blob Storage และ AWS S3

---

## อ้างอิง

- [Azure SDK for .NET Documentation](https://learn.microsoft.com/en-us/dotnet/azure/)
- [Azure Blob Storage .NET SDK](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blob-dotnet-get-started)
- [Azure Service Bus .NET SDK](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-dotnet-get-started-with-queues)
- [Azure Functions .NET Worker](https://learn.microsoft.com/en-us/azure/azure-functions/dotnet-isolated-process-guide)
- [Application Insights for ASP.NET Core](https://learn.microsoft.com/en-us/azure/azure-monitor/app/asp-net-core)
- [Azure Key Vault .NET SDK](https://learn.microsoft.com/en-us/azure/key-vault/secrets/quick-create-net)
- [Azure Container Apps with Dapr](https://learn.microsoft.com/en-us/azure/container-apps/dapr-overview)
- [Azure AI Document Intelligence](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/)

---

## การนำทาง

- [← Part 97: Advanced Patterns & Architecture](./part97-advanced-patterns.md)
- [→ Part 99: DevOps & CI/CD Pipeline](./part99-devops-cicd.md)
- [↑ กลับไปยังสารบัญหลัก](../README.md)
