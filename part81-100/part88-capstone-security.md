# Part 88: Capstone - Advanced Security
## ขั้นตอนที่ 871-880: Security-First Development

---

## 🎯 เป้าหมายของ Part นี้
- Authentication: JWT + Refresh Token Rotation
- Authorization: Policy-based + Resource-based
- OWASP Top 10 mitigation
- Data encryption at rest and in transit
- Security headers
- Audit logging
- Penetration testing basics

---

## ขั้นตอนที่ 871: JWT + Refresh Token Rotation

```csharp
// Secure JWT implementation
public class JwtTokenService : ITokenService
{
    private readonly JwtOptions _options;
    private readonly IRefreshTokenRepository _refreshTokens;
    
    // Access token: short-lived (15 min)
    public string GenerateAccessToken(User user)
    {
        var claims = new[]
        {
            new Claim(JwtRegisteredClaimNames.Sub, user.Id.ToString()),
            new Claim(JwtRegisteredClaimNames.Email, user.Email),
            new Claim(ClaimTypes.Role, user.Role.ToString()),
            new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
        };
        
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_options.Secret));
        var token = new JwtSecurityToken(
            issuer: _options.Issuer,
            audience: _options.Audience,
            claims: claims,
            expires: DateTime.UtcNow.AddMinutes(15),
            signingCredentials: new SigningCredentials(key, SecurityAlgorithms.HmacSha256));
        
        return new JwtSecurityTokenHandler().WriteToken(token);
    }
    
    // Refresh token: long-lived (7 days), stored in DB
    public async Task<string> GenerateRefreshTokenAsync(Guid userId)
    {
        // Random token: cryptographically secure
        var tokenBytes = RandomNumberGenerator.GetBytes(64);
        var token = Convert.ToBase64String(tokenBytes);
        
        await _refreshTokens.SaveAsync(new RefreshToken
        {
            Token = HashToken(token), // store hash, not plaintext
            UserId = userId,
            ExpiresAt = DateTime.UtcNow.AddDays(7),
            CreatedByIp = _httpContextAccessor.HttpContext?.Connection.RemoteIpAddress?.ToString()
        });
        
        return token; // return plaintext to client (store in httpOnly cookie)
    }
    
    // Refresh: rotate token on each use (detect token theft)
    public async Task<(string AccessToken, string RefreshToken)?> RefreshAsync(string refreshToken)
    {
        var hash = HashToken(refreshToken);
        var stored = await _refreshTokens.GetByHashAsync(hash);
        
        if (stored == null || stored.ExpiresAt < DateTime.UtcNow || stored.RevokedAt != null)
        {
            // If token was already used (RevokedAt set), detect theft: revoke entire family
            if (stored?.RevokedAt != null)
                await _refreshTokens.RevokeAllForUserAsync(stored.UserId);
            return null;
        }
        
        // Revoke old token, issue new one
        stored.RevokedAt = DateTime.UtcNow;
        await _refreshTokens.UpdateAsync(stored);
        
        var user = await _users.GetByIdAsync(stored.UserId);
        return (GenerateAccessToken(user!), await GenerateRefreshTokenAsync(stored.UserId));
    }
    
    private static string HashToken(string token)
    {
        var bytes = SHA256.HashData(Encoding.UTF8.GetBytes(token));
        return Convert.ToBase64String(bytes);
    }
}

// Store refresh token in httpOnly cookie (XSS-resistant)
app.MapPost("/auth/refresh", async (HttpContext ctx, ITokenService tokens) =>
{
    var refreshToken = ctx.Request.Cookies["refresh_token"];
    if (string.IsNullOrEmpty(refreshToken)) return Results.Unauthorized();
    
    var result = await tokens.RefreshAsync(refreshToken);
    if (result == null) return Results.Unauthorized();
    
    ctx.Response.Cookies.Append("refresh_token", result.Value.RefreshToken, new CookieOptions
    {
        HttpOnly = true,    // JS cannot access
        Secure = true,      // HTTPS only
        SameSite = SameSiteMode.Strict, // CSRF protection
        Expires = DateTimeOffset.UtcNow.AddDays(7)
    });
    
    return Results.Ok(new { accessToken = result.Value.AccessToken });
});
```

---

## ขั้นตอนที่ 872: Policy-Based Authorization

```csharp
// Define policies
builder.Services.AddAuthorization(options =>
{
    // Simple role policy
    options.AddPolicy("AdminOnly", policy => 
        policy.RequireRole("Admin"));
    
    // Claim-based policy
    options.AddPolicy("CanManageProducts", policy =>
        policy.RequireClaim("permission", "products.write"));
    
    // Custom requirement
    options.AddPolicy("OwnerOrAdmin", policy =>
        policy.AddRequirements(new OwnerOrAdminRequirement()));
    
    // Minimum age
    options.AddPolicy("AdultContent", policy =>
        policy.RequireAssertion(ctx =>
        {
            var dobClaim = ctx.User.FindFirst("date_of_birth");
            if (dobClaim == null) return false;
            var dob = DateOnly.Parse(dobClaim.Value);
            return dob.AddYears(18) <= DateOnly.FromDateTime(DateTime.Today);
        }));
});

// Resource-based authorization
public class OwnerOrAdminRequirement : IAuthorizationRequirement { }

public class OrderAuthorizationHandler 
    : AuthorizationHandler<OwnerOrAdminRequirement, Order>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        OwnerOrAdminRequirement requirement,
        Order resource)
    {
        var userId = context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        
        if (context.User.IsInRole("Admin") || resource.UserId.ToString() == userId)
            context.Succeed(requirement);
        else
            context.Fail(new AuthorizationFailureReason(this, "Not the order owner"));
        
        return Task.CompletedTask;
    }
}

// Usage in endpoint
[HttpGet("{id}")]
public async Task<IActionResult> GetOrder(Guid id)
{
    var order = await _orders.GetByIdAsync(id);
    if (order == null) return NotFound();
    
    var authResult = await _authService.AuthorizeAsync(User, order, "OwnerOrAdmin");
    if (!authResult.Succeeded) return Forbid();
    
    return Ok(order);
}
```

---

## ขั้นตอนที่ 873: OWASP Top 10 Mitigation

```csharp
// OWASP Top 10 in .NET

// A01: Broken Access Control → Resource-based auth (see above)
// A02: Cryptographic Failures → strong hashing, TLS
// A03: Injection → parameterized queries (EF Core handles this)
// A05: Security Misconfiguration → remove default endpoints, secure headers
// A07: Identification/Auth Failures → JWT rotation (see above)
// A09: Security Logging → audit log

// A03: SQL Injection prevention
// ✅ EF Core: parameterized automatically
var user = await db.Users.FirstOrDefaultAsync(u => u.Email == email);

// ✅ Raw SQL with parameters (not string interpolation)
var users = await db.Database.SqlQuery<User>($"SELECT * FROM users WHERE email = {email}").ToListAsync();
// ❌ NEVER: $"SELECT * FROM users WHERE email = '{email}'"

// A02: Password hashing (Argon2id via ASP.NET Identity)
public class PasswordService
{
    private readonly IPasswordHasher<User> _hasher;
    
    public string Hash(string password) => _hasher.HashPassword(null!, password);
    
    public bool Verify(string hash, string password)
        => _hasher.VerifyHashedPassword(null!, hash, password) 
           != PasswordVerificationResult.Failed;
}

// A05: Security headers middleware
app.Use(async (ctx, next) =>
{
    ctx.Response.Headers["X-Content-Type-Options"] = "nosniff";
    ctx.Response.Headers["X-Frame-Options"] = "DENY";
    ctx.Response.Headers["X-XSS-Protection"] = "1; mode=block";
    ctx.Response.Headers["Referrer-Policy"] = "strict-origin-when-cross-origin";
    ctx.Response.Headers["Permissions-Policy"] = "camera=(), microphone=(), geolocation=()";
    ctx.Response.Headers["Content-Security-Policy"] = 
        "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:;";
    await next();
});

// A09: Audit logging
public class AuditLogService
{
    public async Task LogAsync(AuditAction action, string entityType, string entityId, 
        string userId, object? before = null, object? after = null)
    {
        await _db.AuditLogs.AddAsync(new AuditLog
        {
            Action = action,
            EntityType = entityType,
            EntityId = entityId,
            UserId = userId,
            Before = before != null ? JsonSerializer.Serialize(before) : null,
            After = after != null ? JsonSerializer.Serialize(after) : null,
            Timestamp = DateTime.UtcNow,
            IpAddress = _httpContextAccessor.HttpContext?.Connection.RemoteIpAddress?.ToString()
        });
        await _db.SaveChangesAsync();
    }
}
```

---

## ขั้นตอนที่ 874: Data Encryption

```csharp
// Encrypt sensitive data at rest
public class DataProtectionService : IDataProtectionService
{
    private readonly IDataProtector _protector;
    
    public DataProtectionService(IDataProtectionProvider provider)
    {
        // Purpose string must be stable across deployments
        _protector = provider.CreateProtector("ShopThai.SensitiveData.v1");
    }
    
    public string Encrypt(string plaintext) => _protector.Protect(plaintext);
    
    public string Decrypt(string ciphertext) => _protector.Unprotect(ciphertext);
}

// EF Core value converter for encrypted fields
public class EncryptedStringConverter : ValueConverter<string, string>
{
    public EncryptedStringConverter(IDataProtector protector)
        : base(
            v => protector.Protect(v),
            v => protector.Unprotect(v))
    { }
}

// Apply to sensitive columns
modelBuilder.Entity<PaymentMethod>()
    .Property(p => p.CardTokenLast4)
    .HasConversion(new EncryptedStringConverter(dataProtector));

// Setup: Key Ring in Azure Blob Storage + Key Vault
builder.Services.AddDataProtection()
    .PersistKeysToAzureBlobStorage(storageUri, new DefaultAzureCredential())
    .ProtectKeysWithAzureKeyVault(keyVaultKeyId, new DefaultAzureCredential())
    .SetApplicationName("ShopThai")
    .SetDefaultKeyLifetime(TimeSpan.FromDays(90));
```

---

## ขั้นตอนที่ 875: Rate Limiting & Brute Force Protection

```csharp
// Rate limiting in .NET 7+
builder.Services.AddRateLimiter(options =>
{
    // Global limit
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(ctx =>
        RateLimitPartition.GetFixedWindowLimiter(
            ctx.Connection.RemoteIpAddress?.ToString() ?? "anonymous",
            _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 100,
                Window = TimeSpan.FromMinutes(1)
            }));
    
    // Strict limit for auth endpoints
    options.AddPolicy("auth", ctx =>
        RateLimitPartition.GetSlidingWindowLimiter(
            ctx.Connection.RemoteIpAddress?.ToString() ?? "anonymous",
            _ => new SlidingWindowRateLimiterOptions
            {
                PermitLimit = 5,
                Window = TimeSpan.FromMinutes(15),
                SegmentsPerWindow = 3
            }));
    
    options.OnRejected = async (ctx, ct) =>
    {
        ctx.HttpContext.Response.StatusCode = 429;
        if (ctx.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter))
            ctx.HttpContext.Response.Headers.RetryAfter = retryAfter.TotalSeconds.ToString("0");
        await ctx.HttpContext.Response.WriteAsJsonAsync(
            new { error = "Too many requests" }, ct);
    };
});

// Apply strict limit to auth endpoints
app.MapPost("/auth/login", LoginHandler).RequireRateLimiting("auth");
app.MapPost("/auth/register", RegisterHandler).RequireRateLimiting("auth");
app.MapPost("/auth/forgot-password", ForgotPasswordHandler).RequireRateLimiting("auth");
```

---

## ขั้นตอนที่ 876-880: Security Testing

```csharp
// Security-focused tests
public class SecurityTests
{
    [Theory]
    [InlineData("' OR '1'='1")]
    [InlineData("; DROP TABLE users; --")]
    [InlineData("<script>alert('xss')</script>")]
    [InlineData("../../../etc/passwd")]
    public async Task Login_WithMaliciousInput_ReturnsValidationError(string maliciousInput)
    {
        var client = _factory.CreateClient();
        var response = await client.PostAsJsonAsync("/api/auth/login", new
        {
            email = maliciousInput,
            password = maliciousInput
        });
        
        // Should never return 500 (SQL error would indicate injection vulnerability)
        Assert.NotEqual(HttpStatusCode.InternalServerError, response.StatusCode);
    }
    
    [Fact]
    public async Task GetOrder_WithOtherUsersToken_ReturnsForbidden()
    {
        var owner = await CreateUserAsync("owner@test.com");
        var attacker = await CreateUserAsync("attacker@test.com");
        var order = await CreateOrderForUserAsync(owner.Id);
        
        var client = CreateClientWithToken(attacker.Token);
        var response = await client.GetAsync($"/api/orders/{order.Id}");
        
        Assert.Equal(HttpStatusCode.Forbidden, response.StatusCode);
    }
    
    [Fact]
    public async Task Login_WithExpiredToken_ReturnsUnauthorized()
    {
        // Create expired token manually
        var expiredToken = GenerateTokenWithExpiry(DateTime.UtcNow.AddMinutes(-5));
        var client = _factory.CreateClient();
        client.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", expiredToken);
        
        var response = await client.GetAsync("/api/orders");
        Assert.Equal(HttpStatusCode.Unauthorized, response.StatusCode);
    }
    
    [Fact]
    public void SecurityHeaders_ArePresent()
    {
        var response = _factory.CreateClient().GetAsync("/").Result;
        Assert.Contains("X-Content-Type-Options", response.Headers.Select(h => h.Key));
        Assert.Contains("X-Frame-Options", response.Headers.Select(h => h.Key));
    }
}
```

---

## 📝 สรุป Part 88 - Security Checklist

```
✅ Authentication
   □ JWT with 15-minute expiry
   □ Refresh token rotation (detect theft)
   □ httpOnly, Secure, SameSite cookies for refresh token
   □ Rate limiting on auth endpoints (5 req/15min)
   □ Account lockout after failed attempts

✅ Authorization
   □ Role-based access control
   □ Resource-based authorization (owner check)
   □ Policy-based complex rules
   □ Test: unauthorized access returns 403, not 404

✅ Data Protection
   □ Passwords: Argon2id/BCrypt (NEVER MD5/SHA1)
   □ Sensitive fields: AES-256 encrypted at rest
   □ Key rotation via Azure Data Protection
   □ No secrets in code or config files

✅ Input Validation
   □ Server-side validation always (never trust client)
   □ Parameterized queries (EF Core by default)
   □ Output encoding in Razor views (automatic)
   □ File upload: type check, size limit, scan

✅ Transport Security
   □ HTTPS everywhere (HSTS enabled)
   □ TLS 1.2+ minimum
   □ Security headers (CSP, X-Frame-Options, etc.)
   □ CORS: specific origins only

✅ Monitoring
   □ Audit log for sensitive operations
   □ Alert on suspicious patterns (multiple failures)
   □ Dependency vulnerability scan (dotnet audit)
```

---

**ก่อนหน้า → [Part 87: Capstone Performance](part87-capstone-performance.md)**  
**ต่อไป → [Part 89: Capstone - Event-Driven Architecture](part89-capstone-event-driven.md)**
