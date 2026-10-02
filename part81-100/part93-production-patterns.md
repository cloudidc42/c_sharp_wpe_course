# Part 93: Production Deployment Patterns
## ขั้นตอนที่ 921-930: การ Deploy สู่ Production

การ Deploy แอปพลิเคชัน .NET สู่ Production ต้องการความรู้ความเข้าใจในหลายด้าน ตั้งแต่การสร้าง Docker Image ที่มีประสิทธิภาพ การจัดการ Kubernetes Cluster การทำ GitOps ไปจนถึง Production Readiness Checklist ที่ครอบคลุม ใน Part นี้เราจะเรียนรู้ Pattern และ Best Practice ที่ใช้ใน Production จริง

---

## ขั้นตอนที่ 921: Multi-stage Dockerfile

### แนวคิด Multi-stage Docker Build

Multi-stage Dockerfile ช่วยให้เราสามารถแบ่งกระบวนการ Build แอปพลิเคชันออกเป็นหลายขั้นตอน (Stage) ซึ่งส่งผลให้ Docker Image ที่ได้มีขนาดเล็กลงและปลอดภัยมากขึ้น เนื่องจาก Final Image จะมีเฉพาะไฟล์ที่จำเป็นสำหรับการรันแอปพลิเคชันเท่านั้น

### ประโยชน์ของ Multi-stage Build

- **ขนาด Image เล็กลง**: ไม่มี SDK หรือ Build Tool ใน Final Image
- **ความปลอดภัยสูงขึ้น**: ลด Attack Surface โดยไม่มีเครื่องมือพัฒนาใน Production
- **Non-root User**: รันแอปพลิเคชันด้วย User ที่มีสิทธิ์จำกัด
- **Health Check**: ตรวจสอบสุขภาพของ Container อัตโนมัติ

### ตัวอย่าง Multi-stage Dockerfile สำหรับ ASP.NET Core

```dockerfile
# Stage 1: Build Stage - ใช้ SDK Image สำหรับการ Compile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Copy project files and restore dependencies
# แยก COPY ไฟล์ .csproj ออกมาเพื่อใช้ประโยชน์จาก Docker Layer Cache
COPY ["src/MyApp.Api/MyApp.Api.csproj", "src/MyApp.Api/"]
COPY ["src/MyApp.Core/MyApp.Core.csproj", "src/MyApp.Core/"]
COPY ["src/MyApp.Infrastructure/MyApp.Infrastructure.csproj", "src/MyApp.Infrastructure/"]

# Restore NuGet packages
RUN dotnet restore "src/MyApp.Api/MyApp.Api.csproj"

# Copy ไฟล์ Source Code ทั้งหมด
COPY . .

# Build โปรเจค
WORKDIR "/src/src/MyApp.Api"
RUN dotnet build "MyApp.Api.csproj" -c Release -o /app/build

# Stage 2: Publish Stage - สร้าง Publish Output
FROM build AS publish
RUN dotnet publish "MyApp.Api.csproj" \
    -c Release \
    -o /app/publish \
    /p:UseAppHost=false \
    /p:PublishTrimmed=false \
    /p:PublishSingleFile=false

# Stage 3: Runtime Stage - Final Image ที่จะใช้ใน Production
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final

# ตั้งค่า Working Directory
WORKDIR /app

# สร้าง Non-root User เพื่อความปลอดภัย
# ไม่ควรรัน Application ด้วย root user ใน Production
RUN groupadd --gid 1000 appgroup && \
    useradd --uid 1000 --gid appgroup --shell /bin/bash --create-home appuser

# Copy Publish Output จาก Stage ก่อนหน้า
COPY --from=publish /app/publish .

# ตั้งค่า Environment Variables
ENV ASPNETCORE_URLS=http://+:8080
ENV ASPNETCORE_ENVIRONMENT=Production
ENV DOTNET_RUNNING_IN_CONTAINER=true

# สร้าง Directory สำหรับ Logs และตั้งค่า Permission
RUN mkdir -p /app/logs && \
    chown -R appuser:appgroup /app

# เปลี่ยนไปใช้ Non-root User
USER appuser

# เปิด Port ที่แอปพลิเคชันใช้
EXPOSE 8080

# Health Check - ตรวจสอบว่าแอปพลิเคชันยังทำงานอยู่
HEALTHCHECK --interval=30s \
            --timeout=10s \
            --start-period=60s \
            --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

# Entry Point สำหรับรันแอปพลิเคชัน
ENTRYPOINT ["dotnet", "MyApp.Api.dll"]
```

### Health Check Endpoint ใน ASP.NET Core

```csharp
// Program.cs - การตั้งค่า Health Check
using Microsoft.Extensions.Diagnostics.HealthChecks;

var builder = WebApplication.CreateBuilder(args);

// เพิ่ม Health Check Services
builder.Services.AddHealthChecks()
    // ตรวจสอบ Database Connection
    .AddSqlServer(
        connectionString: builder.Configuration.GetConnectionString("DefaultConnection")!,
        healthQuery: "SELECT 1",
        name: "database",
        failureStatus: HealthStatus.Unhealthy,
        tags: new[] { "db", "sql", "sqlserver" })
    // ตรวจสอบ Redis Connection
    .AddRedis(
        redisConnectionString: builder.Configuration.GetConnectionString("Redis")!,
        name: "redis",
        failureStatus: HealthStatus.Degraded,
        tags: new[] { "cache", "redis" })
    // Custom Health Check
    .AddCheck<ExternalApiHealthCheck>("external-api", tags: new[] { "external" });

var app = builder.Build();

// Health Check Endpoints
// /health - สำหรับ Kubernetes Liveness Probe
app.MapHealthChecks("/health", new HealthCheckOptions
{
    Predicate = _ => true,
    ResponseWriter = async (context, report) =>
    {
        context.Response.ContentType = "application/json";
        var result = new
        {
            status = report.Status.ToString(),
            checks = report.Entries.Select(e => new
            {
                name = e.Key,
                status = e.Value.Status.ToString(),
                description = e.Value.Description,
                duration = e.Value.Duration.TotalMilliseconds
            }),
            totalDuration = report.TotalDuration.TotalMilliseconds
        };
        await context.Response.WriteAsJsonAsync(result);
    }
});

// /health/ready - สำหรับ Kubernetes Readiness Probe
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("db") || check.Tags.Contains("cache"),
    ResponseWriter = async (context, report) =>
    {
        context.Response.ContentType = "application/json";
        await context.Response.WriteAsJsonAsync(new
        {
            status = report.Status.ToString(),
            ready = report.Status == HealthStatus.Healthy
        });
    }
});

// /health/live - สำหรับ Basic Liveness
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false // ไม่ตรวจสอบ Dependency ใดๆ เพียงแค่ตอบ 200
});

app.Run();

// Custom Health Check Implementation
public class ExternalApiHealthCheck : IHealthCheck
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<ExternalApiHealthCheck> _logger;

    public ExternalApiHealthCheck(IHttpClientFactory factory, ILogger<ExternalApiHealthCheck> logger)
    {
        _httpClient = factory.CreateClient("external-api");
        _logger = logger;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var response = await _httpClient.GetAsync("/api/status", cancellationToken);
            
            if (response.IsSuccessStatusCode)
            {
                return HealthCheckResult.Healthy("External API is responding normally");
            }
            
            return HealthCheckResult.Degraded(
                $"External API returned status code: {response.StatusCode}");
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "External API health check failed");
            return HealthCheckResult.Unhealthy("External API is not reachable", ex);
        }
    }
}
```

### .dockerignore - ไฟล์ที่ควรสร้างควบคู่กัน

```
# .dockerignore
**/.dockerignore
**/.git
**/.gitignore
**/.vs
**/.vscode
**/bin
**/obj
**/out
**/publish
**/.editorconfig
**/Dockerfile*
**/docker-compose*
**/*.md
**/tests
```

---

## ขั้นตอนที่ 922: Kubernetes Deployment Manifests

### ภาพรวมของ Kubernetes Resources

Kubernetes ใช้ YAML Manifests ในการกำหนดสถานะที่ต้องการของแอปพลิเคชัน โดย Resources หลักที่จำเป็นสำหรับการ Deploy .NET Application ประกอบด้วย:

- **Deployment**: กำหนดจำนวน Replica และ Pod Template
- **Service**: เปิดให้เข้าถึง Pod จากภายนอกหรือภายใน Cluster
- **ConfigMap**: จัดเก็บ Configuration ที่ไม่เป็นความลับ
- **Secret**: จัดเก็บข้อมูลลับเช่น Password และ API Key
- **HorizontalPodAutoscaler (HPA)**: ปรับจำนวน Pod อัตโนมัติตาม Load

### Namespace - แยก Environment

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp-production
  labels:
    app.kubernetes.io/name: myapp
    environment: production
    managed-by: kubectl
```

### ConfigMap - Configuration ที่ไม่เป็นความลับ

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
  namespace: myapp-production
  labels:
    app.kubernetes.io/name: myapp
    app.kubernetes.io/component: config
data:
  # Application Settings
  ASPNETCORE_ENVIRONMENT: "Production"
  ASPNETCORE_URLS: "http://+:8080"
  
  # Logging Configuration
  Logging__LogLevel__Default: "Information"
  Logging__LogLevel__Microsoft: "Warning"
  Logging__LogLevel__Microsoft.Hosting.Lifetime: "Information"
  
  # Feature Flags
  FeatureFlags__EnableNewCheckout: "true"
  FeatureFlags__EnableAnalytics: "true"
  
  # Application Config
  App__Name: "MyApp"
  App__Version: "1.0.0"
  App__MaxRequestSize: "10485760"
  
  # appsettings.json ทั้งไฟล์สามารถใส่ใน ConfigMap ได้
  appsettings.Production.json: |
    {
      "AllowedHosts": "*",
      "Serilog": {
        "MinimumLevel": {
          "Default": "Information",
          "Override": {
            "Microsoft": "Warning",
            "System": "Warning"
          }
        },
        "WriteTo": [
          {
            "Name": "Console",
            "Args": {
              "outputTemplate": "{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level:u3}] {Message:lj}{NewLine}{Exception}"
            }
          }
        ]
      }
    }
```

### Secret - ข้อมูลลับ

```yaml
# secret.yaml
# หมายเหตุ: ใน Production จริงควรใช้ External Secret Manager เช่น Vault หรือ AWS Secrets Manager
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
  namespace: myapp-production
  labels:
    app.kubernetes.io/name: myapp
    app.kubernetes.io/component: secrets
type: Opaque
# ค่าใน data ต้อง Base64 Encode
# echo -n "your-value" | base64
data:
  ConnectionStrings__DefaultConnection: "U2VydmVyPXByb2QtZGI7RGF0YWJhc2U9TXlBcHA7VXNlciBJZD1hcHB1c2VyO1Bhc3N3b3JkPXN1cGVyc2VjcmV0"
  ConnectionStrings__Redis: "cmVkaXMtcHJvZC5teWFwcC5zdmM6NjM3OQ=="
  JwtSettings__SecretKey: "bXktc3VwZXItc2VjcmV0LWtleS1mb3ItcHJvZHVjdGlvbi11c2U="
  ExternalApi__ApiKey: "YXBpa2V5LXByb2R1Y3Rpb24tMTIzNDU2Nzg="
```

### Deployment - กำหนดการ Deploy

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-api
  namespace: myapp-production
  labels:
    app.kubernetes.io/name: myapp
    app.kubernetes.io/component: api
    app.kubernetes.io/version: "1.0.0"
  annotations:
    deployment.kubernetes.io/revision: "1"
spec:
  # จำนวน Replica เริ่มต้น
  replicas: 3
  
  # Strategy สำหรับการ Update
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # เพิ่ม Pod ได้สูงสุด 1 ตัวระหว่าง Update
      maxUnavailable: 0  # ไม่ให้มี Pod ที่ไม่พร้อมใช้งานระหว่าง Update
  
  # Selector สำหรับเลือก Pod
  selector:
    matchLabels:
      app.kubernetes.io/name: myapp
      app.kubernetes.io/component: api
  
  # Pod Template
  template:
    metadata:
      labels:
        app.kubernetes.io/name: myapp
        app.kubernetes.io/component: api
        app.kubernetes.io/version: "1.0.0"
      annotations:
        # Force Pod Restart เมื่อ ConfigMap หรือ Secret เปลี่ยนแปลง
        checksum/config: "{{ sha256sum .Values.configmap }}"
    
    spec:
      # Service Account สำหรับ Pod
      serviceAccountName: myapp-service-account
      
      # ไม่ให้ Mount Service Account Token อัตโนมัติ (หากไม่ต้องการ)
      automountServiceAccountToken: false
      
      # Security Context ระดับ Pod
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      
      # Anti-affinity เพื่อกระจาย Pod ไปยัง Node ต่างๆ
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app.kubernetes.io/name
                  operator: In
                  values:
                  - myapp
              topologyKey: kubernetes.io/hostname
      
      # Init Container สำหรับรอ Database พร้อม
      initContainers:
      - name: wait-for-db
        image: busybox:1.35
        command: ['sh', '-c', 
          'until nc -z prod-db 1433; do echo "Waiting for database..."; sleep 2; done']
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
      
      # Main Container
      containers:
      - name: myapp-api
        image: myregistry.azurecr.io/myapp-api:1.0.0
        imagePullPolicy: Always
        
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        
        # Security Context ระดับ Container
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        
        # Environment Variables จาก ConfigMap
        envFrom:
        - configMapRef:
            name: myapp-config
        
        # Environment Variables จาก Secret
        env:
        - name: ConnectionStrings__DefaultConnection
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: ConnectionStrings__DefaultConnection
        - name: ConnectionStrings__Redis
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: ConnectionStrings__Redis
        - name: JwtSettings__SecretKey
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: JwtSettings__SecretKey
        
        # Resource Limits และ Requests - สำคัญมากใน Production
        resources:
          requests:
            cpu: "100m"      # 0.1 CPU Core
            memory: "256Mi"  # 256 MB RAM
          limits:
            cpu: "500m"      # 0.5 CPU Core
            memory: "512Mi"  # 512 MB RAM
        
        # Liveness Probe - ตรวจสอบว่า Container ยังทำงานอยู่
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 15
          timeoutSeconds: 5
          failureThreshold: 3
        
        # Readiness Probe - ตรวจสอบว่า Container พร้อมรับ Traffic
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        # Startup Probe - สำหรับแอปที่ใช้เวลา Start นาน
        startupProbe:
          httpGet:
            path: /health/live
            port: 8080
          failureThreshold: 30  # รอได้สูงสุด 5 นาที (30 * 10s)
          periodSeconds: 10
        
        # Volume Mounts
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: app-logs
          mountPath: /app/logs
        - name: appsettings
          mountPath: /app/appsettings.Production.json
          subPath: appsettings.Production.json
          readOnly: true
      
      # Volumes
      volumes:
      - name: tmp
        emptyDir: {}
      - name: app-logs
        emptyDir: {}
      - name: appsettings
        configMap:
          name: myapp-config
      
      # Image Pull Secret
      imagePullSecrets:
      - name: registry-credentials
      
      # Termination Grace Period
      terminationGracePeriodSeconds: 60
      
      # DNS Config
      dnsConfig:
        options:
        - name: ndots
          value: "2"
```

### Service - เปิดให้เข้าถึง Pod

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-api
  namespace: myapp-production
  labels:
    app.kubernetes.io/name: myapp
    app.kubernetes.io/component: api
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: myapp
    app.kubernetes.io/component: api
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
  sessionAffinity: None

---
# Ingress - สำหรับรับ Traffic จากภายนอก
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: myapp-production
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - api.myapp.com
    secretName: myapp-tls
  rules:
  - host: api.myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-api
            port:
              number: 80
```

### HorizontalPodAutoscaler - Auto Scaling

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-api-hpa
  namespace: myapp-production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp-api
  
  minReplicas: 3   # จำนวน Pod น้อยสุด
  maxReplicas: 20  # จำนวน Pod มากสุด
  
  metrics:
  # CPU-based Scaling
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # Scale up เมื่อ CPU เกิน 70%
  
  # Memory-based Scaling
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80  # Scale up เมื่อ Memory เกิน 80%
  
  # Custom Metric (ต้องมี Metrics Server)
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
  
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60   # รอ 1 นาทีก่อน Scale Up
      policies:
      - type: Percent
        value: 100
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # รอ 5 นาทีก่อน Scale Down
      policies:
      - type: Percent
        value: 25
        periodSeconds: 60
```

---

## ขั้นตอนที่ 923: Kubernetes Liveness/Readiness Probes

### ความแตกต่างระหว่าง Probe แต่ละประเภท

| Probe | วัตถุประสงค์ | ผลเมื่อ Fail |
|-------|-------------|--------------|
| **Liveness** | ตรวจสอบว่า Container ยังทำงานอยู่ | Restart Container |
| **Readiness** | ตรวจสอบว่า Container พร้อมรับ Traffic | หยุดส่ง Traffic ไปยัง Pod |
| **Startup** | ตรวจสอบว่า Application เริ่มต้นสำเร็จ | Restart Container (ใช้แทน Liveness ระหว่าง Startup) |

### การ Implement Probe Endpoints ใน ASP.NET Core

```csharp
// HealthChecks/DatabaseHealthCheck.cs
using Microsoft.Extensions.Diagnostics.HealthChecks;
using Microsoft.Data.SqlClient;

public class DatabaseHealthCheck : IHealthCheck
{
    private readonly string _connectionString;
    private readonly ILogger<DatabaseHealthCheck> _logger;

    public DatabaseHealthCheck(IConfiguration configuration, ILogger<DatabaseHealthCheck> logger)
    {
        _connectionString = configuration.GetConnectionString("DefaultConnection")
            ?? throw new ArgumentNullException("DefaultConnection");
        _logger = logger;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            using var connection = new SqlConnection(_connectionString);
            await connection.OpenAsync(cancellationToken);
            
            using var command = connection.CreateCommand();
            command.CommandText = "SELECT 1";
            command.CommandTimeout = 5; // 5 วินาที Timeout
            
            await command.ExecuteScalarAsync(cancellationToken);
            
            return HealthCheckResult.Healthy(
                "Database connection is healthy",
                new Dictionary<string, object>
                {
                    { "database", connection.Database },
                    { "server", connection.DataSource }
                });
        }
        catch (SqlException ex)
        {
            _logger.LogError(ex, "Database health check failed");
            return HealthCheckResult.Unhealthy(
                "Database connection failed",
                ex,
                new Dictionary<string, object>
                {
                    { "errorCode", ex.Number },
                    { "errorMessage", ex.Message }
                });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unexpected error during database health check");
            return HealthCheckResult.Unhealthy("Unexpected error", ex);
        }
    }
}

// HealthChecks/RedisHealthCheck.cs
using StackExchange.Redis;

public class RedisHealthCheck : IHealthCheck
{
    private readonly IConnectionMultiplexer _redis;

    public RedisHealthCheck(IConnectionMultiplexer redis)
    {
        _redis = redis;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var db = _redis.GetDatabase();
            var pingResult = await db.PingAsync();
            
            if (pingResult < TimeSpan.FromSeconds(1))
            {
                return HealthCheckResult.Healthy(
                    $"Redis is responding normally. Ping: {pingResult.TotalMilliseconds:F1}ms");
            }
            
            return HealthCheckResult.Degraded(
                $"Redis is slow. Ping: {pingResult.TotalMilliseconds:F1}ms");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Redis connection failed", ex);
        }
    }
}

// Extensions/HealthCheckExtensions.cs
public static class HealthCheckExtensions
{
    public static IServiceCollection AddApplicationHealthChecks(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddHealthChecks()
            .AddCheck<DatabaseHealthCheck>(
                "database",
                failureStatus: HealthStatus.Unhealthy,
                tags: new[] { "ready", "db" })
            .AddCheck<RedisHealthCheck>(
                "redis",
                failureStatus: HealthStatus.Degraded,
                tags: new[] { "ready", "cache" })
            .AddUrlGroup(
                new Uri("https://external-payment-api.com/health"),
                name: "payment-gateway",
                failureStatus: HealthStatus.Degraded,
                tags: new[] { "external" });
        
        return services;
    }
    
    public static WebApplication MapApplicationHealthChecks(this WebApplication app)
    {
        // Liveness Probe - แค่ตรวจว่า Process ยังทำงานอยู่
        app.MapHealthChecks("/health/live", new HealthCheckOptions
        {
            Predicate = _ => false,
            ResponseWriter = WriteSimpleResponse
        });
        
        // Readiness Probe - ตรวจว่าพร้อมรับ Request
        app.MapHealthChecks("/health/ready", new HealthCheckOptions
        {
            Predicate = check => check.Tags.Contains("ready"),
            ResponseWriter = WriteDetailedResponse
        });
        
        // Full Health Check - สำหรับ Monitoring Dashboard
        app.MapHealthChecks("/health", new HealthCheckOptions
        {
            Predicate = _ => true,
            ResponseWriter = WriteDetailedResponse
        });
        
        return app;
    }
    
    private static Task WriteSimpleResponse(HttpContext context, HealthReport report)
    {
        context.Response.ContentType = "application/json";
        return context.Response.WriteAsJsonAsync(new
        {
            status = report.Status.ToString().ToLower(),
            timestamp = DateTime.UtcNow
        });
    }
    
    private static Task WriteDetailedResponse(HttpContext context, HealthReport report)
    {
        context.Response.ContentType = "application/json";
        return context.Response.WriteAsJsonAsync(new
        {
            status = report.Status.ToString().ToLower(),
            timestamp = DateTime.UtcNow,
            duration = report.TotalDuration.TotalMilliseconds,
            checks = report.Entries.Select(e => new
            {
                name = e.Key,
                status = e.Value.Status.ToString().ToLower(),
                description = e.Value.Description,
                duration = e.Value.Duration.TotalMilliseconds,
                tags = e.Value.Tags,
                data = e.Value.Data,
                exception = e.Value.Exception?.Message
            })
        });
    }
}
```

---

## ขั้นตอนที่ 924: Rolling Updates และ Rollback

### กลยุทธ์การ Update ใน Kubernetes

Rolling Update คือการค่อยๆ เปลี่ยน Pod เก่าเป็น Pod ใหม่ทีละตัว ทำให้แอปพลิเคชันยังคงให้บริการได้ระหว่างการ Update

```bash
# อัปเดต Image ของ Deployment
kubectl set image deployment/myapp-api \
  myapp-api=myregistry.azurecr.io/myapp-api:1.1.0 \
  -n myapp-production

# ดู Status การ Rollout
kubectl rollout status deployment/myapp-api -n myapp-production

# ดูประวัติการ Rollout
kubectl rollout history deployment/myapp-api -n myapp-production

# Rollback ไปยัง Version ก่อนหน้า
kubectl rollout undo deployment/myapp-api -n myapp-production

# Rollback ไปยัง Revision ที่ต้องการ
kubectl rollout undo deployment/myapp-api --to-revision=2 -n myapp-production

# หยุด Rollout ชั่วคราว
kubectl rollout pause deployment/myapp-api -n myapp-production

# ดำเนินการ Rollout ต่อ
kubectl rollout resume deployment/myapp-api -n myapp-production
```

### Graceful Shutdown ใน ASP.NET Core

```csharp
// Program.cs - ตั้งค่า Graceful Shutdown
var builder = WebApplication.CreateBuilder(args);

// ตั้งค่า Timeout สำหรับ Shutdown
builder.Services.Configure<HostOptions>(options =>
{
    options.ShutdownTimeout = TimeSpan.FromSeconds(30);
});

// เพิ่ม Graceful Shutdown Service
builder.Services.AddHostedService<GracefulShutdownService>();

var app = builder.Build();

// Handle Cancellation Token สำหรับ Request ที่กำลัง Process อยู่
app.Use(async (context, next) =>
{
    context.RequestAborted.Register(() =>
    {
        // Log เมื่อ Request ถูก Abort
        var logger = context.RequestServices.GetRequiredService<ILogger<Program>>();
        logger.LogInformation("Request {Path} was aborted", context.Request.Path);
    });
    
    await next();
});

app.Run();

// GracefulShutdownService.cs
public class GracefulShutdownService : IHostedService
{
    private readonly ILogger<GracefulShutdownService> _logger;
    private readonly IHostApplicationLifetime _lifetime;

    public GracefulShutdownService(
        ILogger<GracefulShutdownService> logger,
        IHostApplicationLifetime lifetime)
    {
        _logger = logger;
        _lifetime = lifetime;
    }

    public Task StartAsync(CancellationToken cancellationToken)
    {
        _lifetime.ApplicationStarted.Register(() =>
        {
            _logger.LogInformation("Application started at {Time}", DateTime.UtcNow);
        });
        
        _lifetime.ApplicationStopping.Register(() =>
        {
            _logger.LogInformation("Application is stopping. Finishing in-flight requests...");
            // รอให้ Request ที่กำลัง Process เสร็จสิ้น
        });
        
        _lifetime.ApplicationStopped.Register(() =>
        {
            _logger.LogInformation("Application has stopped at {Time}", DateTime.UtcNow);
        });
        
        return Task.CompletedTask;
    }

    public Task StopAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("GracefulShutdownService stopped");
        return Task.CompletedTask;
    }
}
```

### Deployment Strategy Manifest

```yaml
# deployment-strategy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-api
  namespace: myapp-production
  annotations:
    # เก็บประวัติ Revision ไว้ 10 Version
    kubernetes.io/change-cause: "Update to version 1.1.0 - Add payment feature"
spec:
  revisionHistoryLimit: 10  # เก็บประวัติไว้กี่ Version
  
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: "25%"        # เพิ่ม Pod ได้สูงสุด 25% ระหว่าง Update
      maxUnavailable: "0%"   # ไม่ยอมให้ Pod ไม่พร้อมใช้งาน
  
  template:
    spec:
      containers:
      - name: myapp-api
        image: myregistry.azurecr.io/myapp-api:1.1.0
        # Lifecycle Hooks
        lifecycle:
          preStop:
            exec:
              # รอ 15 วินาทีก่อน Stop เพื่อให้ Load Balancer อัปเดต
              command: ["/bin/sleep", "15"]
```

---

## ขั้นตอนที่ 925: Helm Chart Structure สำหรับ .NET Apps

### โครงสร้าง Helm Chart

Helm คือ Package Manager สำหรับ Kubernetes ที่ช่วยให้การ Deploy และจัดการแอปพลิเคชันง่ายขึ้น

```
myapp/
├── Chart.yaml                 # ข้อมูล Chart
├── values.yaml               # Default Values
├── values-staging.yaml       # Values สำหรับ Staging
├── values-production.yaml    # Values สำหรับ Production
├── templates/
│   ├── _helpers.tpl          # Template Helpers
│   ├── deployment.yaml       # Deployment Template
│   ├── service.yaml          # Service Template
│   ├── ingress.yaml          # Ingress Template
│   ├── configmap.yaml        # ConfigMap Template
│   ├── secret.yaml           # Secret Template (ถ้าจำเป็น)
│   ├── hpa.yaml              # HPA Template
│   ├── pdb.yaml              # PodDisruptionBudget Template
│   ├── serviceaccount.yaml   # ServiceAccount Template
│   ├── networkpolicy.yaml    # NetworkPolicy Template
│   ├── NOTES.txt             # ข้อความแสดงหลัง Install
│   └── tests/
│       └── test-connection.yaml
└── charts/                   # Dependency Charts
```

### Chart.yaml

```yaml
# Chart.yaml
apiVersion: v2
name: myapp
description: A Helm chart for MyApp .NET Application
type: application
version: 1.0.0
appVersion: "1.0.0"
keywords:
  - dotnet
  - aspnetcore
  - api
maintainers:
  - name: DevOps Team
    email: devops@myapp.com
dependencies:
  - name: redis
    version: "18.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
  - name: postgresql
    version: "13.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
```

### values.yaml - Default Configuration

```yaml
# values.yaml
# Global Settings
global:
  imageRegistry: ""
  imagePullSecrets: []

# Image Configuration
image:
  repository: myregistry.azurecr.io/myapp-api
  tag: "latest"
  pullPolicy: IfNotPresent

# Replica Count
replicaCount: 1

# Service Configuration
service:
  type: ClusterIP
  port: 80
  targetPort: 8080
  annotations: {}

# Ingress Configuration
ingress:
  enabled: false
  className: nginx
  annotations: {}
  hosts:
  - host: myapp.local
    paths:
    - path: /
      pathType: Prefix
  tls: []

# Resource Limits
resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

# Autoscaling
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

# Probes Configuration
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 15
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

# Application Configuration
config:
  environment: Production
  logLevel: Information
  featureFlags:
    newCheckout: "true"
    analytics: "true"

# Secrets (ใน Production ควรใช้ External Secrets)
secrets:
  create: false
  existingSecret: ""
  
# ServiceAccount
serviceAccount:
  create: true
  annotations: {}
  name: ""

# Pod Anti-Affinity
podAntiAffinity:
  enabled: true
  type: preferred  # preferred หรือ required

# PodDisruptionBudget
podDisruptionBudget:
  enabled: true
  minAvailable: 1

# Network Policy
networkPolicy:
  enabled: false

# Redis Dependency
redis:
  enabled: false

# PostgreSQL Dependency
postgresql:
  enabled: false
```

### _helpers.tpl - Template Helpers

```yaml
{{/*
# templates/_helpers.tpl
*/}}

{{/* ชื่อ Chart */}}
{{- define "myapp.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/* Full Name ของ Release */}}
{{- define "myapp.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/* Chart Label */}}
{{- define "myapp.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/* Common Labels */}}
{{- define "myapp.labels" -}}
helm.sh/chart: {{ include "myapp.chart" . }}
{{ include "myapp.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/* Selector Labels */}}
{{- define "myapp.selectorLabels" -}}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/* ServiceAccount Name */}}
{{- define "myapp.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "myapp.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

### Helm Commands

```bash
# Install Chart
helm install myapp ./myapp \
  -n myapp-production \
  --create-namespace \
  -f values-production.yaml

# Upgrade Release
helm upgrade myapp ./myapp \
  -n myapp-production \
  -f values-production.yaml \
  --set image.tag=1.1.0

# Rollback ไปยัง Version ก่อนหน้า
helm rollback myapp 1 -n myapp-production

# ดูรายการ Releases
helm list -n myapp-production

# ลบ Release
helm uninstall myapp -n myapp-production
```

---

## ขั้นตอนที่ 926: GitOps ด้วย ArgoCD

### แนวคิด GitOps

GitOps คือ Pattern ที่ใช้ Git เป็น Single Source of Truth สำหรับสถานะของ Infrastructure และแอปพลิเคชัน โดย ArgoCD จะ Monitor Git Repository และ Sync การเปลี่ยนแปลงไปยัง Kubernetes Cluster โดยอัตโนมัติ

### ข้อดีของ GitOps

- **ตรวจสอบได้**: ทุกการเปลี่ยนแปลงมีประวัติใน Git
- **Rollback ง่าย**: เพียงแค่ `git revert`
- **ความปลอดภัย**: ไม่ต้องให้ CI/CD มีสิทธิ์เข้าถึง Cluster โดยตรง
- **Self-healing**: ArgoCD จะแก้ไขความแตกต่างระหว่าง Desired State และ Actual State อัตโนมัติ

### ArgoCD Application Manifest

```yaml
# argocd/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-production
  namespace: argocd
  labels:
    app.kubernetes.io/name: myapp
    environment: production
  # Finalizer เพื่อ Cleanup Resources เมื่อลบ Application
  finalizers:
  - resources-finalizer.argocd.argoproj.io
  
spec:
  # Project ที่ Application นี้อยู่
  project: production
  
  # Source - Git Repository
  source:
    repoURL: https://github.com/myorg/myapp-gitops
    targetRevision: main  # Branch, Tag หรือ Commit SHA
    path: environments/production  # Path ใน Repository
    
    # หากใช้ Helm
    helm:
      valueFiles:
      - values-production.yaml
      parameters:
      - name: image.tag
        value: "1.0.0"
  
  # Destination - Kubernetes Cluster
  destination:
    server: https://kubernetes.default.svc  # Cluster URL
    namespace: myapp-production
  
  # Sync Policy
  syncPolicy:
    automated:
      prune: true      # ลบ Resources ที่ไม่มีใน Git
      selfHeal: true   # แก้ไข Drift โดยอัตโนมัติ
      allowEmpty: false
    syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    - PruneLast=true
    - ApplyOutOfSyncOnly=true
    
    # Retry เมื่อ Sync ล้มเหลว
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  
  # Health Check Override
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas  # ไม่ใช้ Replicas จาก Git (ให้ HPA จัดการ)
```

### ArgoCD Project - กำหนดสิทธิ์

```yaml
# argocd/project.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
  namespace: argocd
spec:
  description: Production environment project
  
  # Source Repositories ที่อนุญาต
  sourceRepos:
  - https://github.com/myorg/myapp-gitops
  - https://charts.bitnami.com/bitnami
  
  # Destination Clusters และ Namespaces ที่อนุญาต
  destinations:
  - namespace: myapp-production
    server: https://kubernetes.default.svc
  - namespace: monitoring
    server: https://kubernetes.default.svc
  
  # Cluster Resource Whitelist
  clusterResourceWhitelist:
  - group: ''
    kind: Namespace
  
  # Namespace Resource Blacklist
  namespaceResourceBlacklist:
  - group: ''
    kind: ResourceQuota
  
  # Roles
  roles:
  - name: deploy
    description: Deploy role for CI/CD
    policies:
    - p, proj:production:deploy, applications, sync, production/*, allow
    - p, proj:production:deploy, applications, get, production/*, allow
    jwtTokens:
    - iat: 1700000000
```

### Image Updater - อัปเดต Image Tag อัตโนมัติ

```yaml
# argocd/image-updater.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-production
  namespace: argocd
  annotations:
    # ArgoCD Image Updater annotations
    argocd-image-updater.argoproj.io/image-list: api=myregistry.azurecr.io/myapp-api
    argocd-image-updater.argoproj.io/api.update-strategy: semver
    argocd-image-updater.argoproj.io/api.allow-tags: regexp:^1\.[0-9]+\.[0-9]+$
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/git-branch: main
```

---

## ขั้นตอนที่ 927: Environment-specific Configuration ด้วย Kustomize

### แนวคิด Kustomize

Kustomize ช่วยให้เราสามารถจัดการ Configuration หลาย Environment ได้โดยไม่ต้อง Copy YAML Files ทั้งหมด โดยใช้ Base Configuration และ Overlays

### โครงสร้าง Directory

```
kustomize/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── hpa.yaml
├── overlays/
│   ├── development/
│   │   ├── kustomization.yaml
│   │   ├── deployment-patch.yaml
│   │   └── configmap-patch.yaml
│   ├── staging/
│   │   ├── kustomization.yaml
│   │   ├── deployment-patch.yaml
│   │   └── configmap-patch.yaml
│   └── production/
│       ├── kustomization.yaml
│       ├── deployment-patch.yaml
│       ├── configmap-patch.yaml
│       └── hpa-patch.yaml
```

### base/kustomization.yaml

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml
- configmap.yaml
- hpa.yaml

commonLabels:
  app.kubernetes.io/name: myapp
  app.kubernetes.io/managed-by: kustomize

images:
- name: myregistry.azurecr.io/myapp-api
  newTag: latest
```

### overlays/production/kustomization.yaml

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# อ้างอิง Base
resources:
- ../../base

# Namespace Override
namespace: myapp-production

# Prefix สำหรับ Resources
namePrefix: ""

# Common Labels สำหรับ Production
commonLabels:
  environment: production

# Common Annotations
commonAnnotations:
  managed-by: argocd
  environment: production

# Image Tag Override
images:
- name: myregistry.azurecr.io/myapp-api
  newTag: "1.0.0"

# Patches
patches:
# Deployment Patch - เพิ่ม Replica และปรับ Resources
- path: deployment-patch.yaml
  target:
    kind: Deployment
    name: myapp-api

# ConfigMap Patch - ปรับ Config สำหรับ Production
- path: configmap-patch.yaml
  target:
    kind: ConfigMap
    name: myapp-config

# Strategic Merge Patches
patchesStrategicMerge:
- hpa-patch.yaml

# Secret Generator - สร้าง Secret จากไฟล์
secretGenerator:
- name: myapp-db-secrets
  envs:
  - secrets.env
  type: Opaque
```

### overlays/production/deployment-patch.yaml

```yaml
# overlays/production/deployment-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-api
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: myapp-api
        resources:
          requests:
            cpu: "200m"
            memory: "512Mi"
          limits:
            cpu: "1000m"
            memory: "1Gi"
        env:
        - name: ASPNETCORE_ENVIRONMENT
          value: "Production"
        - name: Logging__LogLevel__Default
          value: "Warning"
```

### overlays/production/configmap-patch.yaml

```yaml
# overlays/production/configmap-patch.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  ASPNETCORE_ENVIRONMENT: "Production"
  App__EnableDetailedErrors: "false"
  App__EnableSwagger: "false"
  Cache__DefaultExpiry: "3600"
  RateLimit__RequestsPerMinute: "1000"
```

### Kustomize Commands

```bash
# ดู Output ที่จะ Apply
kubectl kustomize overlays/production

# Apply Configuration
kubectl apply -k overlays/production

# Diff ระหว่าง Current State และ Desired State
kubectl diff -k overlays/production
```

---

## ขั้นตอนที่ 928: Persistent Volume Claims สำหรับ Stateful Services

### ประเภทของ Storage ใน Kubernetes

```yaml
# storageclass.yaml - กำหนดประเภท Storage
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: premium-ssd
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS  # Azure Premium SSD
  cachingMode: ReadOnly
  kind: Managed
reclaimPolicy: Retain    # ไม่ลบ Volume เมื่อลบ PVC
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true

---
# PersistentVolumeClaim - ขอใช้ Storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myapp-data
  namespace: myapp-production
  labels:
    app.kubernetes.io/name: myapp
spec:
  accessModes:
  - ReadWriteOnce  # ReadWriteOnce, ReadOnlyMany, ReadWriteMany
  storageClassName: premium-ssd
  resources:
    requests:
      storage: 10Gi
```

### StatefulSet สำหรับแอปที่ต้องการ Persistent Storage

```yaml
# statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: myapp-worker
  namespace: myapp-production
spec:
  serviceName: myapp-worker
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: myapp-worker
  
  template:
    metadata:
      labels:
        app.kubernetes.io/name: myapp-worker
    spec:
      containers:
      - name: worker
        image: myregistry.azurecr.io/myapp-worker:1.0.0
        
        resources:
          requests:
            cpu: "200m"
            memory: "512Mi"
          limits:
            cpu: "1000m"
            memory: "1Gi"
        
        volumeMounts:
        - name: data
          mountPath: /app/data
        - name: config
          mountPath: /app/config
          readOnly: true
      
      volumes:
      - name: config
        configMap:
          name: myapp-worker-config
  
  # Volume Claim Templates - สร้าง PVC แยกสำหรับแต่ละ Pod
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes:
      - ReadWriteOnce
      storageClassName: premium-ssd
      resources:
        requests:
          storage: 5Gi

---
# Headless Service สำหรับ StatefulSet
apiVersion: v1
kind: Service
metadata:
  name: myapp-worker
  namespace: myapp-production
spec:
  clusterIP: None  # Headless Service
  selector:
    app.kubernetes.io/name: myapp-worker
  ports:
  - port: 8080
    name: http
```

### การใช้ PVC ใน .NET Application

```csharp
// FileStorageService.cs - การใช้งาน Persistent Storage
public class FileStorageService : IFileStorageService
{
    private readonly string _basePath;
    private readonly ILogger<FileStorageService> _logger;

    public FileStorageService(IConfiguration configuration, ILogger<FileStorageService> logger)
    {
        _basePath = configuration["Storage:BasePath"] ?? "/app/data";
        _logger = logger;
        
        // สร้าง Directory หากยังไม่มี
        Directory.CreateDirectory(_basePath);
    }

    public async Task<string> SaveFileAsync(
        Stream fileStream,
        string fileName,
        CancellationToken cancellationToken = default)
    {
        var safeFileName = Path.GetFileName(fileName); // ป้องกัน Path Traversal
        var filePath = Path.Combine(_basePath, safeFileName);
        
        _logger.LogInformation("Saving file {FileName} to {Path}", safeFileName, filePath);
        
        await using var fileOutput = File.Create(filePath);
        await fileStream.CopyToAsync(fileOutput, cancellationToken);
        
        return safeFileName;
    }

    public async Task<Stream> GetFileAsync(
        string fileName,
        CancellationToken cancellationToken = default)
    {
        var safeFileName = Path.GetFileName(fileName);
        var filePath = Path.Combine(_basePath, safeFileName);
        
        if (!File.Exists(filePath))
        {
            throw new FileNotFoundException($"File not found: {safeFileName}");
        }
        
        return File.OpenRead(filePath);
    }
}
```

---

## ขั้นตอนที่ 929: Network Policies สำหรับ Service Isolation

### แนวคิด Network Policy

Network Policy ช่วยควบคุมการสื่อสารระหว่าง Pod ใน Kubernetes Cluster ทำให้สามารถแยก Service ออกจากกันได้และเพิ่มความปลอดภัย

```yaml
# network-policy.yaml

# Default Deny All - ปฏิเสธ Traffic ทั้งหมดก่อน
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: myapp-production
spec:
  podSelector: {}  # ใช้กับทุก Pod ใน Namespace
  policyTypes:
  - Ingress
  - Egress

---
# Allow API to receive traffic from Ingress Controller
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-to-api
  namespace: myapp-production
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/component: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    # อนุญาตจาก Ingress Controller Namespace
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
    # หรือจาก Pod ที่มี Label ที่กำหนด
    - podSelector:
        matchLabels:
          app.kubernetes.io/name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080

---
# Allow API to access Database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-database
  namespace: myapp-production
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/component: api
  policyTypes:
  - Egress
  egress:
  # อนุญาตให้ API เข้าถึง Database
  - to:
    - podSelector:
        matchLabels:
          app.kubernetes.io/component: database
    ports:
    - protocol: TCP
      port: 1433
  
  # อนุญาตให้ API เข้าถึง Redis
  - to:
    - podSelector:
        matchLabels:
          app.kubernetes.io/component: redis
    ports:
    - protocol: TCP
      port: 6379
  
  # อนุญาต DNS Resolution
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
  
  # อนุญาต HTTPS ออกนอก Cluster
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 10.0.0.0/8
        - 172.16.0.0/12
        - 192.168.0.0/16
    ports:
    - protocol: TCP
      port: 443

---
# Allow Monitoring to scrape metrics
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scraping
  namespace: myapp-production
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: myapp
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: monitoring
      podSelector:
        matchLabels:
          app.kubernetes.io/name: prometheus
    ports:
    - protocol: TCP
      port: 8080
```

### การ Implement Metrics Endpoint ใน .NET

```csharp
// Metrics ด้วย Prometheus
// Install: dotnet add package prometheus-net.AspNetCore

// Program.cs
using Prometheus;

var builder = WebApplication.CreateBuilder(args);

// เพิ่ม Prometheus Metrics
builder.Services.AddMetricServer(options =>
{
    options.Port = 9090; // Metrics Port แยกจาก Application Port
});

var app = builder.Build();

// เพิ่ม Prometheus Middleware
app.UseHttpMetrics(options =>
{
    options.AddCustomLabel("version", context => 
        context.Response.Headers["X-App-Version"].ToString());
});

// Custom Metrics
var requestCounter = Metrics.CreateCounter(
    "myapp_requests_total",
    "Total number of HTTP requests",
    new CounterConfiguration
    {
        LabelNames = new[] { "method", "endpoint", "status" }
    });

var requestDuration = Metrics.CreateHistogram(
    "myapp_request_duration_seconds",
    "HTTP request duration in seconds",
    new HistogramConfiguration
    {
        LabelNames = new[] { "method", "endpoint" },
        Buckets = new[] { 0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0, 5.0 }
    });

// Metrics Endpoint
app.MapMetrics("/metrics");

app.Run();
```

---

## ขั้นตอนที่ 930: Production Readiness Checklist

### PodDisruptionBudget - รับประกัน Availability

```yaml
# pdb.yaml - PodDisruptionBudget
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-api-pdb
  namespace: myapp-production
spec:
  minAvailable: 2  # ต้องมี Pod พร้อมใช้งานอย่างน้อย 2 ตัวเสมอ
  # หรือใช้ maxUnavailable
  # maxUnavailable: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: myapp
      app.kubernetes.io/component: api
```

### ResourceQuota - จำกัด Resources ของ Namespace

```yaml
# resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: myapp-quota
  namespace: myapp-production
spec:
  hard:
    # Compute Resources
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    
    # Object Count
    pods: "50"
    services: "20"
    persistentvolumeclaims: "10"
    secrets: "50"
    configmaps: "50"
    
    # LoadBalancer Services
    services.loadbalancers: "2"
    services.nodeports: "0"  # ไม่อนุญาต NodePort

---
# LimitRange - กำหนด Default Resource Limits
apiVersion: v1
kind: LimitRange
metadata:
  name: myapp-limits
  namespace: myapp-production
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "256Mi"
    max:
      cpu: "2"
      memory: "2Gi"
    min:
      cpu: "50m"
      memory: "64Mi"
```

### Anti-Affinity - กระจาย Pod ไปยัง Node ต่างๆ

```yaml
# anti-affinity-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-api
  namespace: myapp-production
spec:
  template:
    spec:
      # Pod Anti-Affinity - ห้าม Pod อยู่ Node เดียวกัน
      affinity:
        podAntiAffinity:
          # Required: ต้องอยู่คนละ Node
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app.kubernetes.io/name
                operator: In
                values:
                - myapp
            topologyKey: kubernetes.io/hostname
          
          # Preferred: พยายามอยู่คนละ Availability Zone
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app.kubernetes.io/name
                  operator: In
                  values:
                  - myapp
              topologyKey: topology.kubernetes.io/zone
        
        # Node Affinity - รัน Pod บน Node ที่มี Label ที่กำหนด
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-role
                operator: In
                values:
                - application
          
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 50
            preference:
              matchExpressions:
              - key: node-type
                operator: In
                values:
                - high-memory
```

### Production Readiness ใน .NET Application

```csharp
// ProductionReadinessExtensions.cs
public static class ProductionReadinessExtensions
{
    public static WebApplicationBuilder AddProductionReadiness(
        this WebApplicationBuilder builder)
    {
        var services = builder.Services;
        var config = builder.Configuration;
        
        // 1. Structured Logging
        builder.Host.UseSerilog((context, services, configuration) =>
        {
            configuration
                .ReadFrom.Configuration(context.Configuration)
                .ReadFrom.Services(services)
                .Enrich.FromLogContext()
                .Enrich.WithEnvironmentName()
                .Enrich.WithMachineName()
                .Enrich.WithProperty("Application", "MyApp")
                .WriteTo.Console(new JsonFormatter())
                .WriteTo.OpenTelemetry(options =>
                {
                    options.Endpoint = config["OpenTelemetry:Endpoint"]!;
                });
        });
        
        // 2. OpenTelemetry Tracing
        services.AddOpenTelemetry()
            .WithTracing(tracing =>
            {
                tracing
                    .AddAspNetCoreInstrumentation()
                    .AddHttpClientInstrumentation()
                    .AddSqlClientInstrumentation()
                    .AddRedisInstrumentation()
                    .AddOtlpExporter(options =>
                    {
                        options.Endpoint = new Uri(config["OpenTelemetry:Endpoint"]!);
                    });
            })
            .WithMetrics(metrics =>
            {
                metrics
                    .AddAspNetCoreInstrumentation()
                    .AddHttpClientInstrumentation()
                    .AddRuntimeInstrumentation()
                    .AddPrometheusExporter();
            });
        
        // 3. Rate Limiting
        services.AddRateLimiter(options =>
        {
            options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(context =>
            {
                var ipAddress = context.Connection.RemoteIpAddress?.ToString() ?? "unknown";
                return RateLimitPartition.GetFixedWindowLimiter(ipAddress, _ =>
                    new FixedWindowRateLimiterOptions
                    {
                        PermitLimit = 1000,
                        Window = TimeSpan.FromMinutes(1),
                        QueueProcessingOrder = QueueProcessingOrder.OldestFirst,
                        QueueLimit = 100
                    });
            });
            
            options.OnRejected = async (context, token) =>
            {
                context.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
                await context.HttpContext.Response.WriteAsJsonAsync(new
                {
                    error = "Too many requests",
                    retryAfter = context.Lease.TryGetMetadata(
                        MetadataName.RetryAfter, out var retryAfter)
                        ? retryAfter.TotalSeconds
                        : 60
                }, token);
            };
        });
        
        // 4. Response Compression
        services.AddResponseCompression(options =>
        {
            options.EnableForHttps = true;
            options.Providers.Add<BrotliCompressionProvider>();
            options.Providers.Add<GzipCompressionProvider>();
        });
        
        // 5. Circuit Breaker ด้วย Polly
        services.AddHttpClient("external-api")
            .AddStandardResilienceHandler(options =>
            {
                options.CircuitBreaker.BreakDuration = TimeSpan.FromSeconds(30);
                options.CircuitBreaker.FailureRatio = 0.5;
                options.CircuitBreaker.MinimumThroughput = 10;
                options.TotalRequestTimeout.Timeout = TimeSpan.FromSeconds(30);
                options.Retry.MaxRetryAttempts = 3;
                options.Retry.Delay = TimeSpan.FromSeconds(1);
            });
        
        return builder;
    }
}

// Production Readiness Checklist Class
public class ProductionReadinessChecker
{
    private readonly ILogger<ProductionReadinessChecker> _logger;
    private readonly IConfiguration _configuration;

    public ProductionReadinessChecker(
        ILogger<ProductionReadinessChecker> logger,
        IConfiguration configuration)
    {
        _logger = logger;
        _configuration = configuration;
    }

    // ตรวจสอบ Production Readiness Items
    public IEnumerable<ReadinessItem> CheckReadiness()
    {
        var items = new List<ReadinessItem>();

        // ตรวจสอบ Resource Limits
        items.Add(new ReadinessItem(
            "Resource Limits",
            "Kubernetes resource limits are configured",
            CheckResourceLimits()));

        // ตรวจสอบ Health Checks
        items.Add(new ReadinessItem(
            "Health Checks",
            "Liveness and Readiness probes are configured",
            CheckHealthChecks()));

        // ตรวจสอบ Structured Logging
        items.Add(new ReadinessItem(
            "Structured Logging",
            "JSON structured logging is enabled",
            CheckLogging()));

        // ตรวจสอบ Secrets Management
        items.Add(new ReadinessItem(
            "Secrets Management",
            "Sensitive values are not hardcoded",
            CheckSecrets()));

        return items;
    }

    private bool CheckResourceLimits()
    {
        // ในแอปพลิเคชันจริง จะ Check จาก Kubernetes API
        return true;
    }

    private bool CheckHealthChecks()
    {
        // ตรวจสอบว่ามี Health Check Endpoints
        return true;
    }

    private bool CheckLogging()
    {
        // ตรวจสอบว่า Logging เป็น JSON Format
        return _configuration["Serilog:WriteTo:0:Name"] == "Console";
    }

    private bool CheckSecrets()
    {
        // ตรวจสอบว่าไม่มี Secret Hardcoded
        var connectionString = _configuration.GetConnectionString("DefaultConnection");
        return connectionString?.Contains("Password=") != true || 
               connectionString.Contains("$(");
    }
}

public record ReadinessItem(string Name, string Description, bool Passed);
```

### Production Checklist Manifest สำหรับ Kubernetes

```yaml
# complete-production-deployment.yaml
# รวม Resources ทั้งหมดสำหรับ Production Readiness

# 1. Namespace
---
apiVersion: v1
kind: Namespace
metadata:
  name: myapp-production
  labels:
    environment: production
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted

# 2. ServiceAccount
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-service-account
  namespace: myapp-production
  annotations:
    # Azure Workload Identity
    azure.workload.identity/client-id: "00000000-0000-0000-0000-000000000000"
automountServiceAccountToken: false

# 3. PodDisruptionBudget
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-api-pdb
  namespace: myapp-production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: myapp
      app.kubernetes.io/component: api

# 4. Deployment with all best practices
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-api
  namespace: myapp-production
  annotations:
    kubernetes.io/change-cause: "Version 1.0.0 initial deployment"
spec:
  replicas: 3
  revisionHistoryLimit: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app.kubernetes.io/name: myapp
      app.kubernetes.io/component: api
  template:
    metadata:
      labels:
        app.kubernetes.io/name: myapp
        app.kubernetes.io/component: api
    spec:
      serviceAccountName: myapp-service-account
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app.kubernetes.io/name
                operator: In
                values:
                - myapp
            topologyKey: kubernetes.io/hostname
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app.kubernetes.io/name: myapp
      containers:
      - name: myapp-api
        image: myregistry.azurecr.io/myapp-api:1.0.0
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
          name: http
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        resources:
          requests:
            cpu: "200m"
            memory: "512Mi"
          limits:
            cpu: "1000m"
            memory: "1Gi"
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 15
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        startupProbe:
          httpGet:
            path: /health/live
            port: 8080
          failureThreshold: 30
          periodSeconds: 10
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sleep", "15"]
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: app-logs
          mountPath: /app/logs
      volumes:
      - name: tmp
        emptyDir: {}
      - name: app-logs
        emptyDir: {}
      terminationGracePeriodSeconds: 60
```

### สรุป Production Readiness Checklist

```
✅ Production Readiness Checklist

🐳 Container
  □ Multi-stage Dockerfile (ขนาด Image เล็กที่สุด)
  □ Non-root User
  □ Read-only Root Filesystem
  □ Drop All Linux Capabilities
  □ Health Check ใน Dockerfile

🚀 Deployment
  □ Resource Requests และ Limits กำหนดครบ
  □ Liveness Probe
  □ Readiness Probe
  □ Startup Probe (สำหรับแอปที่ Start นาน)
  □ Rolling Update Strategy
  □ Revision History Limit
  □ preStop Hook สำหรับ Graceful Shutdown
  □ terminationGracePeriodSeconds

🔒 Security
  □ Pod Security Context (runAsNonRoot)
  □ Container Security Context
  □ ServiceAccount พร้อม Least Privilege
  □ Network Policy
  □ Secret Management (ไม่ใช้ Environment Variable ธรรมดา)
  □ Image Pull Policy: Always

📈 Availability
  □ Minimum 3 Replicas
  □ PodDisruptionBudget
  □ Pod Anti-Affinity (กระจาย Node)
  □ Topology Spread Constraints (กระจาย Zone)
  □ HorizontalPodAutoscaler

📊 Observability
  □ Structured Logging (JSON)
  □ Distributed Tracing (OpenTelemetry)
  □ Metrics (Prometheus)
  □ Health Check Endpoints
  □ Correlation ID Middleware

⚙️ Configuration
  □ Environment-specific Config
  □ Secrets ไม่ Hardcode
  □ ConfigMap สำหรับ Non-sensitive Config
  □ Secret สำหรับ Sensitive Data

🔄 GitOps
  □ Infrastructure as Code (Helm/Kustomize)
  □ ArgoCD Application Manifest
  □ Auto-sync และ Self-heal
  □ Image Updater สำหรับ Automated Deployment
```

---

## สรุป Part 93

ใน Part นี้เราได้เรียนรู้ Production Deployment Patterns ที่สำคัญสำหรับ .NET Application บน Kubernetes:

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| **Multi-stage Dockerfile** | สร้าง Image ที่เล็กและปลอดภัยด้วย Build/Publish/Runtime Stages |
| **Kubernetes Manifests** | Deployment, Service, ConfigMap, Secret, HPA |
| **Health Probes** | Liveness, Readiness, Startup Probes |
| **Rolling Updates** | Zero-downtime Deployment และ Rollback |
| **Helm Charts** | Package และ Deploy แอปพลิเคชันด้วย Template |
| **ArgoCD GitOps** | Declarative Deployment จาก Git Repository |
| **Kustomize** | Environment-specific Configuration |
| **Persistent Volumes** | จัดการ Storage สำหรับ Stateful Services |
| **Network Policies** | แยก Service และควบคุม Traffic |
| **Production Checklist** | Resource Limits, Anti-affinity, PodDisruptionBudget |

---

## แนวทางการศึกษาต่อ

หลังจาก Part นี้แนะนำให้ศึกษาเพิ่มเติมในหัวข้อ:

- **Service Mesh** (Istio, Linkerd) สำหรับ Advanced Traffic Management
- **Secrets Management** ด้วย HashiCorp Vault หรือ Azure Key Vault
- **Observability** ด้วย Prometheus, Grafana, Jaeger
- **Chaos Engineering** เพื่อทดสอบ Resilience ของระบบ

---

[← Part 92: Advanced Messaging Patterns](part92-advanced-messaging.md) | [Part 94: Observability and Monitoring →](part94-observability.md)
