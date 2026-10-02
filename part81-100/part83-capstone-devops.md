# Part 83: Capstone - DevOps & CI/CD Pipeline
## ขั้นตอนที่ 821-830: DevOps สำหรับ ShopThai Platform

---

## 🎯 เป้าหมายของ Part นี้
- GitHub Actions CI/CD pipeline แบบสมบูรณ์
- Multi-environment deployment (Dev/Staging/Prod)
- Infrastructure as Code ด้วย Bicep / Terraform
- Blue-green deployment
- Automated rollback
- Secrets management
- Monitoring & alerting setup

---

## ขั้นตอนที่ 821: Complete GitHub Actions Pipeline

```yaml
# .github/workflows/main.yml
name: ShopThai CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}/shopthai-api
  DOTNET_VERSION: '8.0.x'

jobs:
  # ──────────────────────────────────────────────
  build-and-test:
    name: Build & Test
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: shopthai_test
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports: ["5432:5432"]
        options: --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5
      
      redis:
        image: redis:7-alpine
        ports: ["6379:6379"]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}
      
      - name: Cache NuGet packages
        uses: actions/cache@v4
        with:
          path: ~/.nuget/packages
          key: nuget-${{ runner.os }}-${{ hashFiles('**/*.csproj') }}
      
      - name: Restore
        run: dotnet restore
      
      - name: Build
        run: dotnet build --no-restore --configuration Release
      
      - name: Unit tests
        run: |
          dotnet test tests/ShopThai.Domain.Tests \
            --no-build --configuration Release \
            --collect:"XPlat Code Coverage" \
            --logger trx --results-directory ./TestResults
      
      - name: Integration tests
        run: |
          dotnet test tests/ShopThai.Integration.Tests \
            --no-build --configuration Release \
            --collect:"XPlat Code Coverage" \
            --logger trx --results-directory ./TestResults
        env:
          ConnectionStrings__DefaultConnection: "Host=localhost;Database=shopthai_test;Username=test;Password=test"
          ConnectionStrings__Redis: "localhost:6379"
      
      - name: Architecture tests
        run: dotnet test tests/ShopThai.Architecture.Tests --no-build --configuration Release
      
      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: ./TestResults
      
      - name: Code coverage report
        uses: codecov/codecov-action@v4
        with:
          files: ./TestResults/**/coverage.cobertura.xml
          fail_ci_if_error: true
          token: ${{ secrets.CODECOV_TOKEN }}

  # ──────────────────────────────────────────────
  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: build-and-test
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}
      
      - name: Check vulnerable packages
        run: dotnet list package --vulnerable --include-transitive 2>&1 | tee vuln-report.txt
      
      - name: Fail on vulnerabilities
        run: |
          if grep -q "has the following vulnerable packages" vuln-report.txt; then
            echo "❌ Vulnerable packages found!"
            cat vuln-report.txt
            exit 1
          fi
      
      - name: Trivy container scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'

  # ──────────────────────────────────────────────
  build-docker:
    name: Build & Push Docker Image
    runs-on: ubuntu-latest
    needs: [build-and-test, security-scan]
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop'
    
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Log in to registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=sha,prefix={{branch}}-
            type=raw,value=latest,enable={{is_default_branch}}
      
      - name: Build and push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            BUILD_VERSION=${{ github.sha }}

  # ──────────────────────────────────────────────
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: build-docker
    if: github.ref == 'refs/heads/main'
    environment: staging
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to staging
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: deploy
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            export IMAGE_TAG=${{ needs.build-docker.outputs.image-tag }}
            cd /app/shopthai-staging
            docker compose pull api
            docker compose up -d api
            docker system prune -f
      
      - name: Run smoke tests
        run: |
          sleep 30  # wait for startup
          curl -f https://staging.shopthai.com/health/ready || exit 1

  # ──────────────────────────────────────────────
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment: production  # requires manual approval
    
    steps:
      - name: Blue-green deploy
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.PROD_HOST }}
          username: deploy
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: /app/shopthai/scripts/blue-green-deploy.sh ${{ needs.build-docker.outputs.image-tag }}
```

---

## ขั้นตอนที่ 822: Blue-Green Deployment Script

```bash
#!/bin/bash
# /app/shopthai/scripts/blue-green-deploy.sh
set -euo pipefail

IMAGE_TAG="${1:?Image tag required}"
APP_DIR="/app/shopthai"
NGINX_CONF="/etc/nginx/sites-enabled/shopthai.conf"

echo "=== Blue-Green Deploy: $IMAGE_TAG ==="

# Determine current and next color
CURRENT=$(docker ps --filter "name=shopthai-" --filter "status=running" --format "{{.Names}}" | grep -oP '(blue|green)' | head -1 || echo "blue")
NEXT=$([ "$CURRENT" = "blue" ] && echo "green" || echo "blue")

echo "Current: $CURRENT → Deploying: $NEXT"

# Start new container
docker run -d \
  --name "shopthai-$NEXT" \
  --network shopthai-net \
  -e ASPNETCORE_ENVIRONMENT=Production \
  -e ConnectionStrings__Default="${DB_CONNECTION}" \
  --health-cmd="curl -f http://localhost:8080/health/live || exit 1" \
  --health-interval=10s \
  --health-retries=5 \
  "ghcr.io/myorg/shopthai-api:${IMAGE_TAG}"

# Wait for health
echo "Waiting for $NEXT to be healthy..."
RETRIES=30
while [ $RETRIES -gt 0 ]; do
  STATUS=$(docker inspect --format='{{.State.Health.Status}}' "shopthai-$NEXT" 2>/dev/null || echo "starting")
  [ "$STATUS" = "healthy" ] && break
  echo "Status: $STATUS ($RETRIES left)"
  sleep 3
  RETRIES=$((RETRIES - 1))
done

if [ $RETRIES -eq 0 ]; then
  echo "❌ Health check failed! Rolling back..."
  docker stop "shopthai-$NEXT" && docker rm "shopthai-$NEXT"
  exit 1
fi

# Switch nginx
sed -i "s/shopthai-$CURRENT/shopthai-$NEXT/g" "$NGINX_CONF"
nginx -s reload

echo "✅ Traffic switched to $NEXT"

# Cleanup old container
sleep 15  # drain connections
docker stop "shopthai-$CURRENT" && docker rm "shopthai-$CURRENT"
echo "=== Deploy complete ==="
```

---

## ขั้นตอนที่ 823: Infrastructure as Code (Bicep)

```bicep
// infrastructure/main.bicep - Azure resources for ShopThai
param location string = 'southeastasia'
param environmentName string

var prefix = 'shopthai-${environmentName}'

// PostgreSQL
resource postgres 'Microsoft.DBforPostgreSQL/flexibleServers@2023-06-01-preview' = {
  name: '${prefix}-db'
  location: location
  sku: {
    name: environmentName == 'prod' ? 'Standard_D4s_v3' : 'Standard_B1ms'
    tier: environmentName == 'prod' ? 'GeneralPurpose' : 'Burstable'
  }
  properties: {
    administratorLogin: 'pgadmin'
    administratorLoginPassword: postgresPassword
    version: '16'
    highAvailability: {
      mode: environmentName == 'prod' ? 'SameZone' : 'Disabled'
    }
    backup: {
      backupRetentionDays: environmentName == 'prod' ? 35 : 7
      geoRedundantBackup: environmentName == 'prod' ? 'Enabled' : 'Disabled'
    }
  }
}

// Redis Cache
resource redis 'Microsoft.Cache/Redis@2023-08-01' = {
  name: '${prefix}-redis'
  location: location
  properties: {
    sku: {
      name: environmentName == 'prod' ? 'Standard' : 'Basic'
      family: 'C'
      capacity: environmentName == 'prod' ? 1 : 0
    }
    enableNonSslPort: false
    minimumTlsVersion: '1.2'
  }
}

// Container App
resource containerApp 'Microsoft.App/containerApps@2023-05-01' = {
  name: '${prefix}-api'
  location: location
  properties: {
    configuration: {
      ingress: {
        external: true
        targetPort: 8080
        transport: 'http'
      }
      secrets: [
        { name: 'db-connection', value: 'Host=${postgres.properties.fullyQualifiedDomainName};...' }
      ]
    }
    template: {
      containers: [
        {
          name: 'api'
          image: 'ghcr.io/myorg/shopthai-api:latest'
          resources: {
            cpu: json(environmentName == 'prod' ? '1.0' : '0.5')
            memory: environmentName == 'prod' ? '2Gi' : '1Gi'
          }
          env: [
            { name: 'ASPNETCORE_ENVIRONMENT', value: environmentName == 'prod' ? 'Production' : 'Staging' }
            { name: 'ConnectionStrings__Default', secretRef: 'db-connection' }
          ]
          probes: [
            {
              type: 'Liveness'
              httpGet: { path: '/health/live', port: 8080 }
              initialDelaySeconds: 15
            }
            {
              type: 'Readiness'
              httpGet: { path: '/health/ready', port: 8080 }
              initialDelaySeconds: 5
            }
          ]
        }
      ]
      scale: {
        minReplicas: environmentName == 'prod' ? 2 : 1
        maxReplicas: environmentName == 'prod' ? 10 : 2
        rules: [
          {
            name: 'http-scaling'
            http: { metadata: { concurrentRequests: '100' } }
          }
        ]
      }
    }
  }
}

output apiUrl string = 'https://${containerApp.properties.configuration.ingress.fqdn}'
```

---

## ขั้นตอนที่ 824: Secrets Management

```csharp
// Azure Key Vault integration
// dotnet add package Azure.Extensions.AspNetCore.Configuration.Secrets
// dotnet add package Azure.Identity

// Program.cs
var keyVaultUri = builder.Configuration["KeyVault:Uri"];
if (!string.IsNullOrEmpty(keyVaultUri))
{
    builder.Configuration.AddAzureKeyVault(
        new Uri(keyVaultUri),
        new DefaultAzureCredential()); // Uses Managed Identity in Azure
}

// Local development: dotnet user-secrets
// dotnet user-secrets set "Jwt:Secret" "your-dev-secret"
// dotnet user-secrets set "ConnectionStrings:Default" "..."

// Key Vault naming convention (-- = :)
// Jwt--Secret         → Jwt:Secret
// ConnectionStrings--Default → ConnectionStrings:Default

// Managed Identity: no credentials in code!
// Azure Container App → System-assigned managed identity
// Grant "Key Vault Secrets User" role to identity
```

---

## ขั้นตอนที่ 825-830: Monitoring & Alerting

```yaml
# docker-compose.monitoring.yml
version: '3.9'

services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'
  
  grafana:
    image: grafana/grafana:latest
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./monitoring/grafana/datasources:/etc/grafana/provisioning/datasources
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD}
      GF_SMTP_ENABLED: "true"
      GF_SMTP_HOST: ${SMTP_HOST}
  
  alertmanager:
    image: prom/alertmanager:latest
    volumes:
      - ./monitoring/alertmanager.yml:/etc/alertmanager/alertmanager.yml
    ports:
      - "9093:9093"

volumes:
  prometheus_data:
  grafana_data:
```

```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alerts/*.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]

scrape_configs:
  - job_name: 'shopthai-api'
    static_configs:
      - targets: ['api:8080']
    metrics_path: '/metrics'
```

```yaml
# monitoring/alerts/api.yml
groups:
  - name: api_alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_server_requests_seconds_count{status=~"5.."}[5m]) > 0.1
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error rate on ShopThai API"
          description: "Error rate is {{ $value | humanizePercentage }} over last 5 minutes"
      
      - alert: SlowResponses
        expr: histogram_quantile(0.95, rate(http_server_requests_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "P95 response time > 2s"
      
      - alert: HighMemoryUsage
        expr: process_working_set_bytes / 1024 / 1024 > 512
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "API memory usage > 512MB"
```

---

## 📝 สรุป Part 83

| เรื่อง | เครื่องมือ |
|-------|-----------|
| CI/CD | GitHub Actions |
| Container Registry | GitHub Container Registry |
| Deployment | Blue-green deploy script |
| IaC | Azure Bicep |
| Secrets | Azure Key Vault + Managed Identity |
| Monitoring | Prometheus + Grafana |
| Alerting | AlertManager → Email/Slack/PagerDuty |

Pipeline stages: Build → Test → Security Scan → Docker Build → Deploy Staging → (Approval) → Deploy Prod

---

**ก่อนหน้า → [Part 82: Capstone Testing](part82-capstone-testing.md)**  
**ต่อไป → [Part 84: Capstone - Real-time Features](part84-capstone-realtime.md)**
