# Part 60: Docker & Deployment
## ขั้นตอนที่ 591-600: Deploy .NET Apps ด้วย Docker

---

## 🎯 เป้าหมายของ Part นี้
- Docker fundamentals
- Dockerfile สำหรับ .NET apps
- Multi-stage builds
- docker-compose สำหรับ development
- Environment configuration
- CI/CD pipeline basics
- Deploy ไปยัง Azure/VPS

---

## ขั้นตอนที่ 591: Dockerfile for .NET

```dockerfile
# Dockerfile - Multi-stage build
# Stage 1: Build
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Copy csproj and restore (layer cache optimization)
COPY ["src/MyApp.Api/MyApp.Api.csproj", "src/MyApp.Api/"]
COPY ["src/MyApp.Application/MyApp.Application.csproj", "src/MyApp.Application/"]
COPY ["src/MyApp.Domain/MyApp.Domain.csproj", "src/MyApp.Domain/"]
COPY ["src/MyApp.Infrastructure/MyApp.Infrastructure.csproj", "src/MyApp.Infrastructure/"]
RUN dotnet restore "src/MyApp.Api/MyApp.Api.csproj"

# Copy everything and build
COPY . .
RUN dotnet build "src/MyApp.Api/MyApp.Api.csproj" -c Release -o /app/build

# Stage 2: Publish
FROM build AS publish
RUN dotnet publish "src/MyApp.Api/MyApp.Api.csproj" -c Release -o /app/publish \
    /p:UseAppHost=false \
    /p:PublishSingleFile=false \
    /p:PublishTrimmed=false

# Stage 3: Runtime (smallest possible image)
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app
EXPOSE 8080
EXPOSE 8081

# Non-root user for security
RUN adduser --disabled-password --no-create-home appuser
USER appuser

COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyApp.Api.dll"]

# .dockerignore (put alongside Dockerfile)
# bin/
# obj/
# **/.vs
# **/*.user
# **/node_modules
```

---

## ขั้นตอนที่ 592: docker-compose for Development

```yaml
# docker-compose.yml
version: '3.9'

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Host=db;Database=myapp;Username=postgres;Password=password
      - JWT__Secret=your-super-secret-key-here-minimum-32-chars
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - myapp-net

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - myapp-net

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    networks:
      - myapp-net

  seq:
    image: datalust/seq:latest
    ports:
      - "5341:80"
    environment:
      ACCEPT_EULA: Y
    networks:
      - myapp-net

volumes:
  pgdata:

networks:
  myapp-net:
    driver: bridge
```

```yaml
# docker-compose.override.yml (development overrides)
version: '3.9'

services:
  api:
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - Serilog__MinimumLevel__Default=Debug
    volumes:
      - ./src:/src  # hot reload for development
```

---

## ขั้นตอนที่ 593: Configuration & Secrets

```csharp
// appsettings.json - non-sensitive defaults
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "Cors": {
    "Origins": ["https://myapp.com"]
  }
}

// appsettings.Production.json - production settings (no secrets)
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning"
    }
  }
}

// Environment variables override appsettings
// Format: Section__Key = value
// Example: ConnectionStrings__DefaultConnection=...

// Program.cs - Configuration setup
var builder = WebApplication.CreateBuilder(args);

// Azure Key Vault for production secrets
if (builder.Environment.IsProduction())
{
    var keyVaultUri = builder.Configuration["KeyVaultUri"];
    if (!string.IsNullOrEmpty(keyVaultUri))
    {
        builder.Configuration.AddAzureKeyVault(
            new Uri(keyVaultUri),
            new DefaultAzureCredential());
    }
}

// Bind config to strongly-typed classes
builder.Services.Configure<JwtSettings>(builder.Configuration.GetSection("Jwt"));
builder.Services.Configure<SmtpSettings>(builder.Configuration.GetSection("Smtp"));

// Access config
var jwtSecret = builder.Configuration["Jwt:Secret"];

// Or inject IOptions<JwtSettings>
public class AuthService
{
    private readonly JwtSettings _jwt;
    public AuthService(IOptions<JwtSettings> jwt) => _jwt = jwt.Value;
}
```

---

## ขั้นตอนที่ 594: Health Checks & Observability

```csharp
// Health checks setup
builder.Services.AddHealthChecks()
    .AddNpgsql(connectionString, name: "database", tags: new[] { "db" })
    .AddRedis(redisConnStr, name: "redis", tags: new[] { "cache" })
    .AddUrlGroup(new Uri("https://api.stripe.com"), name: "stripe", tags: new[] { "external" })
    .AddCheck<CustomHealthCheck>("custom");

// Map health check endpoints
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("db"),
    ResponseWriter = (ctx, report) =>
    {
        ctx.Response.ContentType = "application/json";
        var result = JsonSerializer.Serialize(new
        {
            status = report.Status.ToString(),
            checks = report.Entries.Select(e => new { name = e.Key, status = e.Value.Status.ToString() })
        });
        return ctx.Response.WriteAsync(result);
    }
});

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false // Always healthy if app is up
});

// Serilog structured logging
builder.Host.UseSerilog((ctx, config) =>
{
    config
        .ReadFrom.Configuration(ctx.Configuration)
        .Enrich.FromLogContext()
        .Enrich.WithMachineName()
        .Enrich.WithEnvironmentName()
        .WriteTo.Console(new JsonFormatter())
        .WriteTo.Seq(ctx.Configuration["Seq:Url"] ?? "http://seq:80");
});
```

---

## ขั้นตอนที่ 595: GitHub Actions CI/CD

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'
      
      - name: Restore
        run: dotnet restore
      
      - name: Build
        run: dotnet build --no-restore --configuration Release
      
      - name: Test
        run: dotnet test --no-build --configuration Release \
          --collect:"XPlat Code Coverage" \
          --results-directory ./coverage
      
      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          directory: ./coverage
  
  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Log in to registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and push image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
  
  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - name: Deploy to server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: deploy
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /app
            docker compose pull
            docker compose up -d --remove-orphans
            docker system prune -f
```

---

## ขั้นตอนที่ 596: Production Readiness

```csharp
// Program.cs - Production-ready setup
var builder = WebApplication.CreateBuilder(args);

// Security headers
builder.Services.AddHsts(options =>
{
    options.Preload = true;
    options.IncludeSubDomains = true;
    options.MaxAge = TimeSpan.FromDays(365);
});

// Rate limiting
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("api", o =>
    {
        o.Window = TimeSpan.FromMinutes(1);
        o.PermitLimit = 100;
        o.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        o.QueueLimit = 10;
    });
    
    options.AddSlidingWindowLimiter("login", o =>
    {
        o.Window = TimeSpan.FromMinutes(15);
        o.PermitLimit = 5; // 5 login attempts per 15 min
        o.SegmentsPerWindow = 3;
    });
    
    options.OnRejected = async (context, token) =>
    {
        context.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
        await context.HttpContext.Response.WriteAsJsonAsync(new
        {
            error = "Too many requests. Please try again later."
        }, cancellationToken: token);
    };
});

var app = builder.Build();

if (app.Environment.IsProduction())
{
    app.UseHsts();
    app.UseHttpsRedirection();
}

app.UseRateLimiter();
app.UseSecurityHeaders(); // custom middleware
app.UseAuthentication();
app.UseAuthorization();

// Graceful shutdown
var lifetime = app.Lifetime;
lifetime.ApplicationStopping.Register(() =>
{
    Console.WriteLine("Application is stopping...");
});
```

---

## ขั้นตอนที่ 597-600: VPS Deployment Script

```bash
#!/bin/bash
# deploy.sh - Deploy script for VPS

set -e  # Exit on any error

echo "=== Starting deployment ==="

# Pull latest image
docker pull ghcr.io/myorg/myapp:latest

# Run new container alongside old
docker run -d \
  --name myapp-new \
  --network myapp-net \
  -e ASPNETCORE_ENVIRONMENT=Production \
  -e ConnectionStrings__DefaultConnection="${DB_CONNECTION}" \
  -e JWT__Secret="${JWT_SECRET}" \
  --health-cmd "curl -f http://localhost:8080/health/live || exit 1" \
  --health-interval 10s \
  --health-timeout 5s \
  --health-retries 3 \
  ghcr.io/myorg/myapp:latest

# Wait for new container to be healthy
echo "Waiting for health check..."
for i in {1..30}; do
  status=$(docker inspect --format='{{.State.Health.Status}}' myapp-new 2>/dev/null || echo "starting")
  if [ "$status" = "healthy" ]; then
    echo "New container is healthy!"
    break
  fi
  echo "Health status: $status (attempt $i/30)"
  sleep 3
done

# Swap nginx to point to new container
docker exec nginx nginx -s reload

# Stop and remove old container
docker stop myapp-old 2>/dev/null || true
docker rm myapp-old 2>/dev/null || true
docker rename myapp-new myapp-old 2>/dev/null || true

echo "=== Deployment complete ==="
```

---

## 📝 สรุป Part 51-60

| Part | หัวข้อ | Steps |
|------|--------|-------|
| 51 | Clean Architecture | 501-510 |
| 52 | Unit Testing | 511-520 |
| 53 | Performance | 521-530 |
| 54 | Security | 531-540 |
| 55 | Advanced Async | 541-550 |
| 56 | Advanced C# | 551-560 |
| 57 | Blazor | 561-570 |
| 58 | ASP.NET Core API | 571-580 |
| 59 | SignalR | 581-590 |
| 60 | Docker & Deployment | 591-600 |

---

**ก่อนหน้า → [Part 59: SignalR](part59-signalr.md)**  
**ต่อไป → [Part 61: Domain-Driven Design](../part61-70/part61-ddd-intro.md)**
