# Part 54: Security in C# Applications
## ขั้นตอนที่ 531-540: การรักษาความปลอดภัยใน C#

---

## 🎯 เป้าหมายของ Part นี้
- Password Hashing ด้วย PBKDF2, Argon2, BCrypt
- JWT Authentication (symmetric & asymmetric)
- การเข้ารหัสข้อมูลด้วย AES-GCM และ RSA
- ป้องกัน SQL Injection และ Input Validation
- Secrets Management (ไม่ hardcode secrets)
- OWASP Top 10 Checklist สำหรับ C#
- Rate Limiting, CORS, Anti-forgery tokens

---

## ขั้นตอนที่ 531: Password Hashing

การเก็บรหัสผ่านเป็น plaintext ถือเป็นความผิดพลาดร้ายแรงที่สุดในงาน security  
เสมอต้อง hash รหัสผ่านก่อนเก็บ และใช้ algorithm ที่ออกแบบมาสำหรับงานนี้โดยเฉพาะ

```csharp
// ติดตั้ง packages:
// dotnet add package BCrypt.Net-Next
// dotnet add package Konscious.Security.Cryptography.Argon2

using System.Security.Cryptography;
using System.Text;
using Konscious.Security.Cryptography;

// ============================================================
// 1. PBKDF2 (Password-Based Key Derivation Function 2)
//    - built-in ใน .NET, ไม่ต้องติดตั้ง package
//    - NIST แนะนำให้ใช้อย่างน้อย 310,000 iterations สำหรับ SHA-256
// ============================================================

public class Pbkdf2PasswordHasher
{
    // ค่าตั้งต้น: 32 bytes salt, 32 bytes hash, 310,000 iterations
    private const int SaltSize = 32;
    private const int HashSize = 32;
    private const int Iterations = 310_000;
    private const HashAlgorithmName Algorithm = HashAlgorithmName.SHA256;

    // Hash รหัสผ่านใหม่ (ตอน register)
    public static string HashPassword(string password)
    {
        // สร้าง salt แบบ cryptographically secure
        byte[] salt = RandomNumberGenerator.GetBytes(SaltSize);

        // Hash รหัสผ่านด้วย PBKDF2
        byte[] hash = Rfc2898DeriveBytes.Pbkdf2(
            password: Encoding.UTF8.GetBytes(password),
            salt: salt,
            iterations: Iterations,
            hashAlgorithm: Algorithm,
            outputLength: HashSize
        );

        // เก็บ salt + hash รวมกัน แล้ว encode เป็น Base64
        byte[] combined = new byte[SaltSize + HashSize];
        Buffer.BlockCopy(salt, 0, combined, 0, SaltSize);
        Buffer.BlockCopy(hash, 0, combined, SaltSize, HashSize);
        return Convert.ToBase64String(combined);
    }

    // ตรวจสอบรหัสผ่าน (ตอน login) — ใช้ constant-time comparison
    public static bool VerifyPassword(string password, string storedHash)
    {
        byte[] combined = Convert.FromBase64String(storedHash);

        // แยก salt ออกจาก hash
        byte[] salt = combined[..SaltSize];
        byte[] storedHashBytes = combined[SaltSize..];

        // คำนวณ hash ใหม่ด้วย salt เดิม
        byte[] computedHash = Rfc2898DeriveBytes.Pbkdf2(
            password: Encoding.UTF8.GetBytes(password),
            salt: salt,
            iterations: Iterations,
            hashAlgorithm: Algorithm,
            outputLength: HashSize
        );

        // ใช้ CryptographicOperations.FixedTimeEquals เพื่อป้องกัน timing attack
        // อย่าใช้ == หรือ SequenceEqual เพราะจะหยุดเปรียบเทียบเมื่อเจอ byte ที่ต่างกัน
        return CryptographicOperations.FixedTimeEquals(computedHash, storedHashBytes);
    }
}

// ============================================================
// 2. Argon2 (ผ่าน Konscious.Security.Cryptography.Argon2)
//    - ชนะ Password Hashing Competition (PHC) ปี 2015
//    - ปรับ memory + CPU + parallelism ได้
//    - Argon2id แนะนำมากที่สุด (hybrid ของ Argon2i + Argon2d)
// ============================================================

public class Argon2PasswordHasher
{
    private const int SaltSize = 16;      // 16 bytes = 128 bits
    private const int HashSize = 32;      // 32 bytes = 256 bits
    private const int MemorySize = 65536; // 64 MB (เพิ่มต้นทุน GPU attack)
    private const int Iterations = 4;    // จำนวนรอบ
    private const int DegreeOfParallelism = 2; // threads

    public static string HashPassword(string password)
    {
        byte[] salt = RandomNumberGenerator.GetBytes(SaltSize);
        byte[] hash = ComputeHash(password, salt);

        // Format: Base64(salt):Base64(hash)
        return $"{Convert.ToBase64String(salt)}:{Convert.ToBase64String(hash)}";
    }

    public static bool VerifyPassword(string password, string storedHash)
    {
        var parts = storedHash.Split(':');
        if (parts.Length != 2) return false;

        byte[] salt = Convert.FromBase64String(parts[0]);
        byte[] storedHashBytes = Convert.FromBase64String(parts[1]);

        byte[] computedHash = ComputeHash(password, salt);
        return CryptographicOperations.FixedTimeEquals(computedHash, storedHashBytes);
    }

    private static byte[] ComputeHash(string password, byte[] salt)
    {
        using var argon2 = new Argon2id(Encoding.UTF8.GetBytes(password))
        {
            Salt = salt,
            MemorySize = MemorySize,
            Iterations = Iterations,
            DegreeOfParallelism = DegreeOfParallelism
        };
        return argon2.GetBytes(HashSize);
    }
}

// ============================================================
// 3. BCrypt (ผ่าน BCrypt.Net-Next)
//    - ออกแบบมาสำหรับ password hashing โดยเฉพาะ
//    - ใช้งานง่าย มี work factor ปรับได้
//    - work factor 12 = ~300ms ปัจจุบัน (ปรับขึ้นได้เมื่อ hardware เร็วขึ้น)
// ============================================================

using BCrypt.Net;

public class BcryptPasswordHasher
{
    // work factor: 12 คือ 2^12 = 4096 rounds
    // เพิ่มทีละ 1 = ช้าลง 2 เท่า (เพิ่มเป็น 13, 14 เมื่อ hardware เร็วขึ้น)
    private const int WorkFactor = 12;

    public static string HashPassword(string password)
    {
        // BCrypt จัดการ salt เองภายใน
        return BCrypt.Net.BCrypt.EnhancedHashPassword(password, WorkFactor);
    }

    public static bool VerifyPassword(string password, string storedHash)
    {
        // Enhanced ใช้ SHA-384 ก่อน เพื่อรองรับ password ที่ยาวเกิน 72 bytes
        return BCrypt.Net.BCrypt.EnhancedVerify(password, storedHash);
    }
}

// ============================================================
// ตัวอย่างการใช้งานจริง
// ============================================================

// ตอน Register
string password = "MySecureP@ssw0rd!";
string hashed = Pbkdf2PasswordHasher.HashPassword(password);
Console.WriteLine($"Stored hash: {hashed}");
// Output: Base64 string (ไม่เห็น password เดิมเลย)

// ตอน Login
bool isValid = Pbkdf2PasswordHasher.VerifyPassword("MySecureP@ssw0rd!", hashed);
bool isInvalid = Pbkdf2PasswordHasher.VerifyPassword("wrongpassword", hashed);
Console.WriteLine($"Correct: {isValid}");   // True
Console.WriteLine($"Wrong: {isInvalid}");   // False

// ❌ NEVER ทำแบบนี้:
// string storedPassword = password;  // เก็บ plaintext ห้ามทำ!
// if (storedPassword == inputPassword) // เปรียบเทียบ plaintext ห้ามทำ!

// ❌ NEVER ใช้ MD5 หรือ SHA-1 สำหรับ password:
// var md5Hash = MD5.HashData(Encoding.UTF8.GetBytes(password)); // ไม่ปลอดภัย!
// var sha1Hash = SHA1.HashData(Encoding.UTF8.GetBytes(password)); // ไม่ปลอดภัย!
```

---

## ขั้นตอนที่ 532: JWT Authentication

JWT (JSON Web Token) เป็นมาตรฐานสำหรับส่ง claims ระหว่าง parties  
ประกอบด้วย 3 ส่วน: Header.Payload.Signature แต่ละส่วน encode ด้วย Base64Url

```csharp
// ติดตั้ง packages:
// dotnet add package System.IdentityModel.Tokens.Jwt
// dotnet add package Microsoft.IdentityModel.Tokens

using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Security.Cryptography;
using Microsoft.IdentityModel.Tokens;

// ============================================================
// 1. สร้าง JWT ด้วย Symmetric Key (HMACSHA256)
//    - ทั้ง sign และ verify ใช้ key เดียวกัน
//    - เหมาะกับ single server หรือ microservices ที่ share secret
// ============================================================

public class JwtTokenService
{
    // secret ควรยาวอย่างน้อย 32 bytes (256 bits)
    private readonly string _secretKey;
    private readonly string _issuer;
    private readonly string _audience;

    public JwtTokenService(string secretKey, string issuer, string audience)
    {
        _secretKey = secretKey;
        _issuer = issuer;
        _audience = audience;
    }

    // สร้าง Access Token
    public string CreateAccessToken(string userId, string email, IEnumerable<string> roles)
    {
        var handler = new JwtSecurityTokenHandler();
        var keyBytes = Encoding.UTF8.GetBytes(_secretKey);
        var signingKey = new SymmetricSecurityKey(keyBytes);

        // Claims = ข้อมูลที่ฝังไว้ใน token
        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub, userId),          // subject (user id)
            new(JwtRegisteredClaimNames.Email, email),         // email
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()), // unique token id
            new(JwtRegisteredClaimNames.Iat,                   // issued at
                DateTimeOffset.UtcNow.ToUnixTimeSeconds().ToString(),
                ClaimValueTypes.Integer64),
        };

        // เพิ่ม roles เป็น claims หลายตัว
        foreach (var role in roles)
            claims.Add(new Claim(ClaimTypes.Role, role));

        // เพิ่ม custom claims
        claims.Add(new Claim("tenant_id", "tenant-001"));
        claims.Add(new Claim("plan", "premium"));

        var tokenDescriptor = new SecurityTokenDescriptor
        {
            Subject = new ClaimsIdentity(claims),
            Issuer = _issuer,
            Audience = _audience,
            Expires = DateTime.UtcNow.AddMinutes(15),  // Access token หมดอายุเร็ว (15 นาที)
            SigningCredentials = new SigningCredentials(
                signingKey,
                SecurityAlgorithms.HmacSha256
            )
        };

        var token = handler.CreateToken(tokenDescriptor);
        return handler.WriteToken(token);
    }

    // ตรวจสอบ token และดึง claims ออกมา
    public ClaimsPrincipal? ValidateToken(string token)
    {
        var handler = new JwtSecurityTokenHandler();
        var keyBytes = Encoding.UTF8.GetBytes(_secretKey);

        var validationParams = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidIssuer = _issuer,
            ValidateAudience = true,
            ValidAudience = _audience,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(keyBytes),
            ClockSkew = TimeSpan.FromSeconds(30) // ยืดหยุ่น 30 วินาที
        };

        try
        {
            var principal = handler.ValidateToken(token, validationParams, out var validatedToken);
            return principal;
        }
        catch (SecurityTokenException)
        {
            return null; // token ไม่ valid
        }
    }
}

// ============================================================
// 2. JWT ด้วย Asymmetric Key (RSA)
//    - ใช้ private key สำหรับ sign
//    - ใช้ public key สำหรับ verify
//    - เหมาะกับ multiple services: Auth server sign, API servers verify
// ============================================================

public class RsaJwtTokenService
{
    private readonly RSA _privateKey;
    private readonly RSA _publicKey;

    public RsaJwtTokenService()
    {
        // ใน production: โหลดจาก PEM file หรือ Key Vault
        _privateKey = RSA.Create(2048);
        _publicKey = RSA.Create();
        _publicKey.ImportRSAPublicKey(_privateKey.ExportRSAPublicKey(), out _);
    }

    public string CreateToken(string userId, string email)
    {
        var handler = new JwtSecurityTokenHandler();

        // Sign ด้วย private key
        var signingCredentials = new SigningCredentials(
            new RsaSecurityKey(_privateKey),
            SecurityAlgorithms.RsaSha256
        );

        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub, userId),
            new(JwtRegisteredClaimNames.Email, email),
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
        };

        var tokenDescriptor = new SecurityTokenDescriptor
        {
            Subject = new ClaimsIdentity(claims),
            Issuer = "https://auth.example.com",
            Audience = "https://api.example.com",
            Expires = DateTime.UtcNow.AddMinutes(15),
            SigningCredentials = signingCredentials
        };

        return handler.WriteToken(handler.CreateToken(tokenDescriptor));
    }

    public ClaimsPrincipal? ValidateToken(string token)
    {
        var handler = new JwtSecurityTokenHandler();

        // Verify ด้วย public key เท่านั้น (ไม่ต้องมี private key)
        var validationParams = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidIssuer = "https://auth.example.com",
            ValidateAudience = true,
            ValidAudience = "https://api.example.com",
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new RsaSecurityKey(_publicKey),
            ClockSkew = TimeSpan.Zero
        };

        try
        {
            return handler.ValidateToken(token, validationParams, out _);
        }
        catch
        {
            return null;
        }
    }
}

// ============================================================
// 3. Refresh Token Pattern
//    - Access token: อายุสั้น (15 นาที), เก็บใน memory
//    - Refresh token: อายุยาว (7 วัน), เก็บใน HttpOnly cookie
// ============================================================

public record TokenPair(string AccessToken, string RefreshToken, DateTime RefreshTokenExpiry);

public class RefreshTokenService
{
    // เก็บ refresh tokens (ใน production ใช้ DB)
    private readonly Dictionary<string, (string UserId, DateTime Expiry)> _refreshTokens = new();

    public TokenPair CreateTokenPair(string userId, string email, IEnumerable<string> roles)
    {
        // สร้าง access token (อายุ 15 นาที)
        var jwtService = new JwtTokenService("super-secret-key-min-32-chars!!", "issuer", "audience");
        string accessToken = jwtService.CreateAccessToken(userId, email, roles);

        // สร้าง refresh token (random, opaque)
        string refreshToken = Convert.ToBase64String(RandomNumberGenerator.GetBytes(64));
        var expiry = DateTime.UtcNow.AddDays(7);

        _refreshTokens[refreshToken] = (userId, expiry);

        return new TokenPair(accessToken, refreshToken, expiry);
    }

    public TokenPair? RefreshTokens(string refreshToken)
    {
        if (!_refreshTokens.TryGetValue(refreshToken, out var info))
            return null;

        if (info.Expiry < DateTime.UtcNow)
        {
            _refreshTokens.Remove(refreshToken); // ลบ token หมดอายุ
            return null;
        }

        // Rotate refresh token (invalidate เก่า, สร้างใหม่)
        _refreshTokens.Remove(refreshToken);
        return CreateTokenPair(info.UserId, "user@example.com", ["user"]);
    }

    public void RevokeRefreshToken(string refreshToken)
    {
        _refreshTokens.Remove(refreshToken); // Logout
    }
}

// ============================================================
// ตัวอย่าง decode JWT (ดู payload โดยไม่ต้อง verify)
// ============================================================

public static void DecodeJwtPayload(string token)
{
    var handler = new JwtSecurityTokenHandler();

    if (handler.CanReadToken(token))
    {
        var jwt = handler.ReadJwtToken(token);

        Console.WriteLine($"Subject: {jwt.Subject}");
        Console.WriteLine($"Issuer: {jwt.Issuer}");
        Console.WriteLine($"Expires: {jwt.ValidTo}");

        foreach (var claim in jwt.Claims)
            Console.WriteLine($"  Claim: {claim.Type} = {claim.Value}");
    }
}
```

---

## ขั้นตอนที่ 533: Encryption

```csharp
// ============================================================
// 1. AES-GCM (Authenticated Encryption)
//    - Symmetric: ใช้ key เดียวกัน encrypt/decrypt
//    - Authenticated: ตรวจสอบว่าข้อมูลไม่ถูกแก้ไขด้วย Tag
//    - GCM mode ดีกว่า CBC เพราะมี authentication ในตัว
// ============================================================

public class AesGcmEncryption
{
    private const int KeySize = 32;    // 256-bit key
    private const int NonceSize = 12;  // 96-bit nonce (recommended for GCM)
    private const int TagSize = 16;    // 128-bit authentication tag

    // เข้ารหัสข้อมูล
    public static byte[] Encrypt(byte[] plaintext, byte[] key)
    {
        // สร้าง nonce แบบ random ทุกครั้ง (อย่าใช้ nonce ซ้ำ!)
        byte[] nonce = RandomNumberGenerator.GetBytes(NonceSize);
        byte[] ciphertext = new byte[plaintext.Length];
        byte[] tag = new byte[TagSize];

        using var aesGcm = new AesGcm(key, TagSize);
        aesGcm.Encrypt(nonce, plaintext, ciphertext, tag);

        // รวม nonce + tag + ciphertext เพื่อส่งหรือเก็บ
        // ผู้รับต้องรู้ว่าแต่ละส่วนอยู่ที่ตำแหน่งไหน
        byte[] result = new byte[NonceSize + TagSize + ciphertext.Length];
        Buffer.BlockCopy(nonce, 0, result, 0, NonceSize);
        Buffer.BlockCopy(tag, 0, result, NonceSize, TagSize);
        Buffer.BlockCopy(ciphertext, 0, result, NonceSize + TagSize, ciphertext.Length);

        return result;
    }

    // ถอดรหัสข้อมูล
    public static byte[] Decrypt(byte[] encryptedData, byte[] key)
    {
        // แยกแต่ละส่วนออก
        byte[] nonce = encryptedData[..NonceSize];
        byte[] tag = encryptedData[NonceSize..(NonceSize + TagSize)];
        byte[] ciphertext = encryptedData[(NonceSize + TagSize)..];

        byte[] plaintext = new byte[ciphertext.Length];

        using var aesGcm = new AesGcm(key, TagSize);
        // จะ throw AuthenticationTagMismatchException ถ้าข้อมูลถูกแก้ไข
        aesGcm.Decrypt(nonce, ciphertext, tag, plaintext);

        return plaintext;
    }

    // สร้าง key จาก password (Key Derivation)
    public static byte[] DeriveKeyFromPassword(string password, byte[] salt)
    {
        return Rfc2898DeriveBytes.Pbkdf2(
            password: Encoding.UTF8.GetBytes(password),
            salt: salt,
            iterations: 100_000,
            hashAlgorithm: HashAlgorithmName.SHA256,
            outputLength: KeySize
        );
    }
}

// ============================================================
// 2. RSA Asymmetric Encryption
//    - ใช้ Public Key เข้ารหัส, Private Key ถอดรหัส
//    - เหมาะกับข้อมูลขนาดเล็ก (เช่น AES key, short messages)
//    - ไม่เหมาะกับข้อมูลขนาดใหญ่ (ช้ากว่า AES มาก)
// ============================================================

public class RsaEncryption
{
    // เข้ารหัสด้วย public key (ทุกคนมีได้)
    public static byte[] Encrypt(byte[] data, RSAParameters publicKey)
    {
        using var rsa = RSA.Create();
        rsa.ImportParameters(publicKey);
        // OAEP padding ปลอดภัยกว่า PKCS#1 v1.5
        return rsa.Encrypt(data, RSAEncryptionPadding.OaepSHA256);
    }

    // ถอดรหัสด้วย private key (เจ้าของเท่านั้น)
    public static byte[] Decrypt(byte[] encryptedData, RSAParameters privateKey)
    {
        using var rsa = RSA.Create();
        rsa.ImportParameters(privateKey);
        return rsa.Decrypt(encryptedData, RSAEncryptionPadding.OaepSHA256);
    }

    // Export/Import keys เป็น PEM format
    public static (string PublicKeyPem, string PrivateKeyPem) GenerateKeyPair()
    {
        using var rsa = RSA.Create(2048);
        string publicPem = rsa.ExportRSAPublicKeyPem();
        string privatePem = rsa.ExportRSAPrivateKeyPem();
        return (publicPem, privatePem);
    }
}

// ============================================================
// 3. Hybrid Encryption (Best Practice สำหรับข้อมูลขนาดใหญ่)
//    - เข้ารหัสข้อมูลด้วย AES (เร็ว)
//    - เข้ารหัส AES key ด้วย RSA (ปลอดภัย)
//    - ผสมข้อดีของทั้งสองแบบ
// ============================================================

public class HybridEncryption
{
    public static byte[] Encrypt(byte[] data, RSAParameters recipientPublicKey)
    {
        // สร้าง AES key แบบ random สำหรับข้อมูลชุดนี้
        byte[] aesKey = RandomNumberGenerator.GetBytes(32);

        // เข้ารหัสข้อมูลด้วย AES-GCM
        byte[] encryptedData = AesGcmEncryption.Encrypt(data, aesKey);

        // เข้ารหัส AES key ด้วย RSA public key ของผู้รับ
        byte[] encryptedAesKey = RsaEncryption.Encrypt(aesKey, recipientPublicKey);

        // ล้าง AES key ออกจาก memory
        CryptographicOperations.ZeroMemory(aesKey);

        // รวม: [4 bytes length of encrypted key] + [encrypted key] + [encrypted data]
        byte[] keyLengthBytes = BitConverter.GetBytes(encryptedAesKey.Length);
        byte[] result = new byte[4 + encryptedAesKey.Length + encryptedData.Length];
        Buffer.BlockCopy(keyLengthBytes, 0, result, 0, 4);
        Buffer.BlockCopy(encryptedAesKey, 0, result, 4, encryptedAesKey.Length);
        Buffer.BlockCopy(encryptedData, 0, result, 4 + encryptedAesKey.Length, encryptedData.Length);

        return result;
    }

    public static byte[] Decrypt(byte[] hybridEncrypted, RSAParameters recipientPrivateKey)
    {
        // แยกส่วน
        int keyLength = BitConverter.ToInt32(hybridEncrypted, 0);
        byte[] encryptedAesKey = hybridEncrypted[4..(4 + keyLength)];
        byte[] encryptedData = hybridEncrypted[(4 + keyLength)..];

        // ถอดรหัส AES key ด้วย RSA private key
        byte[] aesKey = RsaEncryption.Decrypt(encryptedAesKey, recipientPrivateKey);

        // ถอดรหัสข้อมูลด้วย AES key
        byte[] plaintext = AesGcmEncryption.Decrypt(encryptedData, aesKey);

        CryptographicOperations.ZeroMemory(aesKey);
        return plaintext;
    }
}

// ============================================================
// 4. X509Certificate2 สำหรับ Production Keys
// ============================================================

public class X509KeyManager
{
    // โหลด certificate จาก Windows Certificate Store
    public static X509Certificate2? LoadFromStore(string thumbprint)
    {
        using var store = new X509Store(StoreName.My, StoreLocation.LocalMachine);
        store.Open(OpenFlags.ReadOnly);

        var certs = store.Certificates.Find(
            X509FindType.FindByThumbprint,
            thumbprint,
            validOnly: true
        );

        return certs.Count > 0 ? certs[0] : null;
    }

    // โหลด certificate จากไฟล์ PFX
    public static X509Certificate2 LoadFromFile(string pfxPath, string password)
    {
        return new X509Certificate2(pfxPath, password,
            X509KeyStorageFlags.MachineKeySet | X509KeyStorageFlags.PersistKeySet);
    }

    // เข้ารหัสด้วย certificate (ใช้ public key ที่อยู่ใน cert)
    public static byte[] EncryptWithCertificate(byte[] data, X509Certificate2 cert)
    {
        using var rsa = cert.GetRSAPublicKey()!;
        return rsa.Encrypt(data, RSAEncryptionPadding.OaepSHA256);
    }
}

// ============================================================
// 5. Data Protection API (DPAPI) สำหรับ Windows
//    - เข้ารหัสด้วย Windows user/machine credentials
//    - ง่าย ไม่ต้องจัดการ key เอง
// ============================================================

using Microsoft.AspNetCore.DataProtection;

public class DataProtectionExample
{
    private readonly IDataProtector _protector;

    public DataProtectionExample(IDataProtectionProvider provider)
    {
        // "purpose" เป็น namespace ป้องกัน cross-purpose decryption
        _protector = provider.CreateProtector("MyApp.SensitiveData.v1");
    }

    public string Protect(string plaintext)
    {
        return _protector.Protect(plaintext);
    }

    public string Unprotect(string ciphertext)
    {
        return _protector.Unprotect(ciphertext);
    }

    // Time-limited protection
    public string ProtectWithExpiry(string data, TimeSpan lifetime)
    {
        var timeLimited = _protector.ToTimeLimitedDataProtector();
        return timeLimited.Protect(data, lifetime);
    }
}

// Setup ใน Program.cs
builder.Services.AddDataProtection()
    .PersistKeysToFileSystem(new DirectoryInfo(@"/var/keys"))  // Linux
    .SetApplicationName("MyApp")
    .SetDefaultKeyLifetime(TimeSpan.FromDays(90));
```

---

## ขั้นตอนที่ 534: Input Validation & SQL Injection Prevention

SQL Injection ยังติดอันดับ OWASP Top 10 มาตลอด  
การป้องกันที่ดีที่สุดคือ parameterized queries และ input validation

```csharp
// ============================================================
// 1. Parameterized Queries ด้วย EF Core
//    - EF Core ใช้ parameterized queries โดย default
//    - อย่าใช้ string interpolation กับ FromSqlRaw!
// ============================================================

using Microsoft.EntityFrameworkCore;

public class UserRepository
{
    private readonly AppDbContext _db;

    public UserRepository(AppDbContext db) => _db = db;

    // ✅ SAFE: EF Core สร้าง parameterized query ให้อัตโนมัติ
    public async Task<User?> GetUserSafeAsync(string username)
    {
        // EF แปลงเป็น: SELECT * FROM Users WHERE Username = @p0
        return await _db.Users
            .Where(u => u.Username == username)
            .FirstOrDefaultAsync();
    }

    // ✅ SAFE: ใช้ FromSqlInterpolated (แปลงเป็น parameters อัตโนมัติ)
    public async Task<List<User>> SearchUsersSafeAsync(string searchTerm)
    {
        return await _db.Users
            .FromSqlInterpolated($"SELECT * FROM Users WHERE Username LIKE {$"%{searchTerm}%"}")
            .ToListAsync();
    }

    // ❌ DANGEROUS: SQL Injection vulnerability!
    public async Task<User?> GetUserDangerousAsync(string username)
    {
        // อย่าทำแบบนี้! ถ้า username = "' OR '1'='1" จะได้ทุก user
        var sql = $"SELECT * FROM Users WHERE Username = '{username}'";
        return await _db.Users.FromSqlRaw(sql).FirstOrDefaultAsync();
    }

    // ✅ SAFE: ถ้าต้องใช้ FromSqlRaw ใช้ parameters แยก
    public async Task<User?> GetUserWithRawSafeAsync(string username)
    {
        return await _db.Users
            .FromSqlRaw("SELECT * FROM Users WHERE Username = {0}", username)
            .FirstOrDefaultAsync();
    }
}

// ============================================================
// 2. Parameterized Queries ด้วย Dapper
// ============================================================

using Dapper;
using System.Data;

public class DapperUserRepository
{
    private readonly IDbConnection _connection;

    public DapperUserRepository(IDbConnection connection) => _connection = connection;

    // ✅ SAFE: ใช้ anonymous object เป็น parameters
    public async Task<User?> GetUserAsync(string username)
    {
        const string sql = "SELECT * FROM Users WHERE Username = @Username";
        return await _connection.QueryFirstOrDefaultAsync<User>(sql, new { Username = username });
    }

    // ✅ SAFE: INSERT ด้วย Dapper
    public async Task<int> CreateUserAsync(User user)
    {
        const string sql = @"
            INSERT INTO Users (Username, Email, PasswordHash, CreatedAt)
            VALUES (@Username, @Email, @PasswordHash, @CreatedAt)";

        return await _connection.ExecuteAsync(sql, user);
    }

    // ✅ SAFE: WHERE IN ด้วย Dapper
    public async Task<IEnumerable<User>> GetUsersByIdsAsync(IEnumerable<int> ids)
    {
        const string sql = "SELECT * FROM Users WHERE Id IN @Ids";
        // Dapper จัดการ IN clause ให้อัตโนมัติ
        return await _connection.QueryAsync<User>(sql, new { Ids = ids });
    }
}

// ============================================================
// 3. FluentValidation สำหรับ Input Validation
//    - dotnet add package FluentValidation.AspNetCore
// ============================================================

using FluentValidation;

public class RegisterRequest
{
    public string Username { get; set; } = "";
    public string Email { get; set; } = "";
    public string Password { get; set; } = "";
    public int Age { get; set; }
}

public class RegisterRequestValidator : AbstractValidator<RegisterRequest>
{
    public RegisterRequestValidator()
    {
        RuleFor(x => x.Username)
            .NotEmpty().WithMessage("Username ห้ามว่าง")
            .Length(3, 50).WithMessage("Username ต้องมี 3-50 ตัวอักษร")
            // อนุญาตเฉพาะ alphanumeric และ underscore
            .Matches(@"^[a-zA-Z0-9_]+$").WithMessage("Username ใช้ได้เฉพาะ a-z, A-Z, 0-9, _")
            .Must(NotContainSqlKeywords).WithMessage("Username ไม่ถูกต้อง");

        RuleFor(x => x.Email)
            .NotEmpty()
            .EmailAddress().WithMessage("รูปแบบ Email ไม่ถูกต้อง")
            .MaximumLength(256);

        RuleFor(x => x.Password)
            .NotEmpty()
            .MinimumLength(8).WithMessage("Password ต้องมีอย่างน้อย 8 ตัวอักษร")
            .Matches(@"[A-Z]").WithMessage("ต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว")
            .Matches(@"[a-z]").WithMessage("ต้องมีตัวพิมพ์เล็กอย่างน้อย 1 ตัว")
            .Matches(@"[0-9]").WithMessage("ต้องมีตัวเลขอย่างน้อย 1 ตัว")
            .Matches(@"[!@#$%^&*]").WithMessage("ต้องมีอักขระพิเศษอย่างน้อย 1 ตัว (!@#$%^&*)");

        RuleFor(x => x.Age)
            .InclusiveBetween(13, 120).WithMessage("อายุต้องอยู่ระหว่าง 13-120 ปี");
    }

    private static bool NotContainSqlKeywords(string value)
    {
        // ป้องกัน SQL keywords ใน username (defense in depth)
        var sqlKeywords = new[] { "SELECT", "DROP", "INSERT", "UPDATE", "DELETE", "UNION", "--", "/*" };
        return !sqlKeywords.Any(kw => value.Contains(kw, StringComparison.OrdinalIgnoreCase));
    }
}

// ============================================================
// 4. HTML Encoding ป้องกัน XSS
// ============================================================

using System.Text.Encodings.Web;

public class HtmlSanitizer
{
    // Encode HTML entities ป้องกัน XSS
    public static string EncodeHtml(string userInput)
    {
        // แปลง < เป็น &lt;, > เป็น &gt;, " เป็น &quot; ฯลฯ
        return HtmlEncoder.Default.Encode(userInput);
    }

    // Encode สำหรับ URL query parameter
    public static string EncodeUrl(string userInput)
    {
        return UrlEncoder.Default.Encode(userInput);
    }

    // Encode สำหรับ JavaScript string
    public static string EncodeJavaScript(string userInput)
    {
        return JavaScriptEncoder.Default.Encode(userInput);
    }

    // ตัวอย่างการใช้งาน
    public static void Demo()
    {
        string maliciousInput = "<script>alert('XSS')</script>";

        Console.WriteLine(EncodeHtml(maliciousInput));
        // Output: &lt;script&gt;alert(&#x27;XSS&#x27;)&lt;/script&gt;
        // Browser จะแสดง text ไม่ execute script
    }
}

// ============================================================
// 5. Path Traversal Prevention
// ============================================================

public class SafeFileAccess
{
    private readonly string _baseDirectory;

    public SafeFileAccess(string baseDirectory)
    {
        // Normalize path
        _baseDirectory = Path.GetFullPath(baseDirectory);
    }

    public string ReadFile(string fileName)
    {
        // อย่า trust user input โดยตรง
        // ถ้า fileName = "../../etc/passwd" จะอ่าน system file ได้!

        // ป้องกัน path traversal
        string sanitizedName = Path.GetFileName(fileName); // เอาแค่ filename, ตัด path ทิ้ง
        string fullPath = Path.GetFullPath(Path.Combine(_baseDirectory, sanitizedName));

        // Double-check: ต้องอยู่ใน base directory เท่านั้น
        if (!fullPath.StartsWith(_baseDirectory, StringComparison.OrdinalIgnoreCase))
            throw new UnauthorizedAccessException("Access denied: path traversal detected");

        if (!File.Exists(fullPath))
            throw new FileNotFoundException($"File not found: {sanitizedName}");

        return File.ReadAllText(fullPath);
    }

    // Whitelist extension ที่ allowed
    private static readonly HashSet<string> AllowedExtensions = new(StringComparer.OrdinalIgnoreCase)
    {
        ".txt", ".pdf", ".png", ".jpg", ".jpeg"
    };

    public bool IsAllowedExtension(string fileName)
    {
        var ext = Path.GetExtension(fileName);
        return AllowedExtensions.Contains(ext);
    }
}

// ============================================================
// 6. Regex สำหรับ Safe Patterns
// ============================================================

using System.Text.RegularExpressions;

public static class InputValidationPatterns
{
    // Compile regex ล่วงหน้าเพื่อประสิทธิภาพ
    private static readonly Regex UsernameRegex =
        new(@"^[a-zA-Z0-9_]{3,50}$", RegexOptions.Compiled);

    private static readonly Regex PhoneRegex =
        new(@"^\+?[0-9\s\-\(\)]{7,15}$", RegexOptions.Compiled);

    private static readonly Regex UuidRegex =
        new(@"^[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}$",
            RegexOptions.Compiled);

    // ใช้ Timeout เพื่อป้องกัน ReDoS (Regular Expression Denial of Service)
    private static readonly Regex EmailRegex = new(
        @"^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$",
        RegexOptions.Compiled,
        TimeSpan.FromMilliseconds(100) // timeout 100ms
    );

    public static bool IsValidUsername(string input) =>
        UsernameRegex.IsMatch(input);

    public static bool IsValidEmail(string input)
    {
        try
        {
            return EmailRegex.IsMatch(input);
        }
        catch (RegexMatchTimeoutException)
        {
            // เกิด ReDoS attempt
            return false;
        }
    }

    public static bool IsValidUuid(string input) =>
        UuidRegex.IsMatch(input);
}
```

---

## ขั้นตอนที่ 535: Secrets Management

การ hardcode secrets ใน source code เป็นความเสี่ยงที่พบบ่อยมาก  
secrets เช่น connection strings, API keys, passwords ต้องไม่อยู่ใน codebase

```csharp
// ============================================================
// 1. Environment Variables — วิธีพื้นฐาน
// ============================================================

// ❌ NEVER hardcode secrets:
// var connectionString = "Server=prod;Password=MyP@ssw0rd!";
// var apiKey = "sk-1234567890abcdef";

// ✅ ใช้ Environment Variables แทน:
string? connectionString = Environment.GetEnvironmentVariable("DB_CONNECTION_STRING");
string? apiKey = Environment.GetEnvironmentVariable("THIRD_PARTY_API_KEY");

if (string.IsNullOrEmpty(connectionString))
    throw new InvalidOperationException("DB_CONNECTION_STRING environment variable not set");

// ============================================================
// 2. .NET Secret Manager (Development เท่านั้น)
//    - เก็บ secrets นอก project directory
//    - ไม่ถูก commit เข้า git
// ============================================================

// Setup:
// dotnet user-secrets init
// dotnet user-secrets set "Database:Password" "devpassword123"
// dotnet user-secrets set "Jwt:SecretKey" "dev-jwt-secret-key-min-32chars"

// ใน Program.cs: (เพิ่มอัตโนมัติเมื่อ Development environment)
builder.Configuration.AddUserSecrets<Program>(optional: true);

// ใช้ผ่าน IConfiguration:
public class DatabaseService
{
    private readonly string _connectionString;

    public DatabaseService(IConfiguration config)
    {
        // อ่านจาก user-secrets (dev) หรือ env var (prod)
        _connectionString = config.GetConnectionString("DefaultConnection")
            ?? throw new InvalidOperationException("Connection string not configured");
    }
}

// ============================================================
// 3. Azure Key Vault
//    - dotnet add package Azure.Extensions.AspNetCore.Configuration.Secrets
//    - dotnet add package Azure.Identity
// ============================================================

using Azure.Identity;
using Azure.Security.KeyVault.Secrets;

// ใน Program.cs
var keyVaultUri = new Uri($"https://{builder.Configuration["KeyVaultName"]}.vault.azure.net/");

builder.Configuration.AddAzureKeyVault(
    keyVaultUri,
    // DefaultAzureCredential ลองหลาย auth methods อัตโนมัติ:
    // 1. Environment vars (AZURE_CLIENT_ID, etc.)
    // 2. Managed Identity (ใน Azure VM/App Service)
    // 3. Azure CLI credentials (dev)
    // 4. Visual Studio credentials
    new DefaultAzureCredential()
);

// ตัวอย่าง read secret โดยตรง
public class KeyVaultSecretReader
{
    private readonly SecretClient _client;

    public KeyVaultSecretReader(string keyVaultUri, TokenCredential credential)
    {
        _client = new SecretClient(new Uri(keyVaultUri), credential);
    }

    public async Task<string> GetSecretAsync(string secretName)
    {
        var response = await _client.GetSecretAsync(secretName);
        return response.Value.Value;
    }

    public async Task SetSecretAsync(string secretName, string secretValue)
    {
        await _client.SetSecretAsync(secretName, secretValue);
    }

    // List secrets (ชื่อ เพื่อ audit)
    public async Task<List<string>> ListSecretNamesAsync()
    {
        var names = new List<string>();
        await foreach (var secret in _client.GetPropertiesOfSecretsAsync())
            names.Add(secret.Name);
        return names;
    }
}

// ============================================================
// 4. AWS Secrets Manager
//    - dotnet add package AWSSDK.SecretsManager
// ============================================================

using Amazon.SecretsManager;
using Amazon.SecretsManager.Model;
using System.Text.Json;

public class AwsSecretsManagerReader
{
    private readonly IAmazonSecretsManager _client;

    public AwsSecretsManagerReader(IAmazonSecretsManager client)
    {
        _client = client;
    }

    public async Task<string> GetSecretAsync(string secretName)
    {
        var request = new GetSecretValueRequest { SecretId = secretName };
        var response = await _client.GetSecretValueAsync(request);

        // Secret อาจเป็น string หรือ binary
        return response.SecretString ?? throw new Exception("Secret is binary");
    }

    // Secret เก็บเป็น JSON object (แนะนำ)
    public async Task<Dictionary<string, string>> GetSecretJsonAsync(string secretName)
    {
        var secretString = await GetSecretAsync(secretName);
        return JsonSerializer.Deserialize<Dictionary<string, string>>(secretString)
            ?? new Dictionary<string, string>();
    }
}

// ============================================================
// 5. IConfiguration Pattern — รวม sources ไว้ที่เดียว
// ============================================================

// appsettings.json (ไม่มี secrets!)
// {
//   "App": {
//     "Name": "MyApp",
//     "Version": "1.0"
//   },
//   "Database": {
//     "Host": "localhost",   // non-secret
//     "Port": 5432,          // non-secret
//     "Name": "mydb"         // non-secret
//     // "Password" ไม่อยู่ที่นี่!
//   }
// }

// Program.cs — ลำดับสำคัญ: ทีหลัง override ทีก่อน
builder.Configuration
    .AddJsonFile("appsettings.json", optional: false)
    .AddJsonFile($"appsettings.{builder.Environment.EnvironmentName}.json", optional: true)
    .AddUserSecrets<Program>(optional: true)        // dev: override ด้วย user secrets
    .AddEnvironmentVariables()                       // prod: override ด้วย env vars
    .AddAzureKeyVault(keyVaultUri, new DefaultAzureCredential()); // prod: Key Vault

// Strong-typed configuration
public class DatabaseOptions
{
    public const string SectionName = "Database";

    public string Host { get; set; } = "localhost";
    public int Port { get; set; } = 5432;
    public string Name { get; set; } = "";
    public string Password { get; set; } = ""; // มาจาก secrets, ไม่ใช่ appsettings.json
}

// Register
builder.Services.Configure<DatabaseOptions>(
    builder.Configuration.GetSection(DatabaseOptions.SectionName));

// ใช้
public class MyService
{
    private readonly DatabaseOptions _dbOptions;

    public MyService(IOptions<DatabaseOptions> options)
    {
        _dbOptions = options.Value;
        // _dbOptions.Password มาจาก Key Vault หรือ env var แล้ว
    }
}
```

---

## ขั้นตอนที่ 536: OWASP Top 10 Checklist สำหรับ C#

OWASP Top 10 (2021) คือรายการความเสี่ยงที่พบบ่อยที่สุดในเว็บแอปพลิเคชัน  
ตรวจสอบแต่ละข้อเพื่อให้แอปพลิเคชันของคุณปลอดภัย

```csharp
// ============================================================
// A01: Broken Access Control — ควบคุมการเข้าถึงผิดพลาด
// ============================================================

// ✅ ใช้ [Authorize] attribute บน controllers/actions
[ApiController]
[Route("api/[controller]")]
[Authorize] // ต้อง login ทุก action ใน controller นี้
public class OrdersController : ControllerBase
{
    private readonly IOrderService _orderService;

    public OrdersController(IOrderService orderService) => _orderService = orderService;

    [HttpGet("{orderId}")]
    public async Task<IActionResult> GetOrder(int orderId)
    {
        var userId = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;

        // ✅ Verify ว่า order เป็นของ user นี้จริงๆ (Insecure Direct Object Reference)
        var order = await _orderService.GetOrderAsync(orderId, userId!);
        if (order == null)
            return NotFound(); // ไม่บอกว่า order มีอยู่แต่ไม่มีสิทธิ์

        return Ok(order);
    }

    // ✅ Role-based authorization
    [HttpDelete("{orderId}")]
    [Authorize(Roles = "Admin,Manager")]
    public async Task<IActionResult> DeleteOrder(int orderId)
    {
        await _orderService.DeleteOrderAsync(orderId);
        return NoContent();
    }

    // ✅ Policy-based authorization
    [HttpPost("{orderId}/approve")]
    [Authorize(Policy = "CanApproveOrders")]
    public async Task<IActionResult> ApproveOrder(int orderId)
    {
        await _orderService.ApproveOrderAsync(orderId);
        return Ok();
    }
}

// Register policies ใน Program.cs
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("CanApproveOrders", policy =>
        policy.RequireAssertion(context =>
            context.User.IsInRole("Manager") ||
            (context.User.IsInRole("Supervisor") && context.User.HasClaim("department", "sales"))
        ));
});

// ============================================================
// A02: Cryptographic Failures — ใช้ crypto ผิด/อ่อนแอ
// ============================================================

// ✅ ใช้ HTTPS เสมอ (redirect HTTP → HTTPS)
// ใน Program.cs:
app.UseHttpsRedirection();
app.UseHsts(); // HTTP Strict Transport Security

// ✅ เข้ารหัส sensitive data ก่อนเก็บ
public class SensitiveDataRepository
{
    private readonly AesGcmEncryption _crypto;
    private readonly byte[] _encryptionKey;

    public SensitiveDataRepository(IOptions<CryptoOptions> options)
    {
        _encryptionKey = Convert.FromBase64String(options.Value.EncryptionKeyBase64);
    }

    public async Task SaveCreditCardAsync(string userId, string cardNumber)
    {
        // เก็บแค่ last 4 digits + เข้ารหัส full number
        byte[] encrypted = AesGcmEncryption.Encrypt(
            Encoding.UTF8.GetBytes(cardNumber),
            _encryptionKey
        );

        var record = new CreditCardRecord
        {
            UserId = userId,
            LastFourDigits = cardNumber[^4..],
            EncryptedNumber = Convert.ToBase64String(encrypted)
        };

        // save to DB...
    }
}

// ============================================================
// A03: Injection — SQL, LDAP, OS command injection
// ============================================================

// ✅ ใช้ parameterized queries (ดูใน Step 534)
// ✅ Validate และ sanitize input ทุกตัว
// ✅ ป้องกัน OS command injection:

public class DangerousExample
{
    // ❌ NEVER ทำแบบนี้:
    public void RunCommandDangerous(string userInput)
    {
        // ถ้า userInput = "test; rm -rf /" จะเป็นอันตราย!
        Process.Start("bash", $"-c echo {userInput}");
    }

    // ✅ Whitelist approach:
    private static readonly HashSet<string> AllowedCommands = new() { "ping", "tracert" };

    public void RunCommandSafe(string command, string targetIp)
    {
        if (!AllowedCommands.Contains(command))
            throw new ArgumentException("Command not allowed");

        // Validate IP format
        if (!System.Net.IPAddress.TryParse(targetIp, out _))
            throw new ArgumentException("Invalid IP address");

        // ใช้ argument list แทน shell string (ป้องกัน injection)
        var psi = new ProcessStartInfo(command)
        {
            ArgumentList = { targetIp },
            UseShellExecute = false,
            RedirectStandardOutput = true
        };
        Process.Start(psi);
    }
}
```

---

## ขั้นตอนที่ 537: HttpOnly Cookies, SameSite, HTTPS

```csharp
// ============================================================
// Secure Cookie Configuration
// ============================================================

// ใน Program.cs — Cookie Security settings
builder.Services.ConfigureApplicationCookie(options =>
{
    options.Cookie.HttpOnly = true;     // ป้องกัน JavaScript อ่าน cookie (XSS)
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;  // ส่งเฉพาะ HTTPS
    options.Cookie.SameSite = SameSiteMode.Strict;  // ป้องกัน CSRF
    options.Cookie.Name = "__Host-Session";  // __Host- prefix บังคับ Secure + no domain
    options.ExpireTimeSpan = TimeSpan.FromHours(8);
    options.SlidingExpiration = true;
});

// สร้าง cookie แบบ manual
public class CookieService
{
    public void SetSecureCookie(HttpResponse response, string name, string value, TimeSpan maxAge)
    {
        var cookieOptions = new CookieOptions
        {
            HttpOnly = true,
            Secure = true,
            SameSite = SameSiteMode.Strict,
            MaxAge = maxAge,
            Path = "/",
            // Domain ไม่ตั้งเพื่อให้ scope เฉพาะ host นั้น
        };

        response.Cookies.Append(name, value, cookieOptions);
    }

    // Refresh Token ใน HttpOnly Cookie
    public void SetRefreshTokenCookie(HttpResponse response, string refreshToken)
    {
        var cookieOptions = new CookieOptions
        {
            HttpOnly = true,        // ป้องกัน XSS อ่าน token
            Secure = true,          // HTTPS only
            SameSite = SameSiteMode.Strict,
            MaxAge = TimeSpan.FromDays(7),
            Path = "/api/auth/refresh"  // จำกัด path เพื่อไม่ส่ง cookie ทุก request
        };

        response.Cookies.Append("refresh_token", refreshToken, cookieOptions);
    }
}

// ============================================================
// Security Headers ใน Program.cs
// ============================================================

app.Use(async (context, next) =>
{
    var headers = context.Response.Headers;

    // ป้องกัน Clickjacking
    headers.Append("X-Frame-Options", "DENY");

    // ป้องกัน MIME type sniffing
    headers.Append("X-Content-Type-Options", "nosniff");

    // Referrer Policy
    headers.Append("Referrer-Policy", "strict-origin-when-cross-origin");

    // Content Security Policy (ป้องกัน XSS)
    headers.Append("Content-Security-Policy",
        "default-src 'self'; " +
        "script-src 'self'; " +
        "style-src 'self' 'unsafe-inline'; " +
        "img-src 'self' data: https:; " +
        "frame-ancestors 'none'");

    // Permissions Policy
    headers.Append("Permissions-Policy", "camera=(), microphone=(), geolocation=()");

    await next();
});

// HSTS (HTTP Strict Transport Security) ใน Program.cs
builder.Services.AddHsts(options =>
{
    options.Preload = true;
    options.IncludeSubDomains = true;
    options.MaxAge = TimeSpan.FromDays(365);
});

app.UseHsts();
app.UseHttpsRedirection();
```

---

## ขั้นตอนที่ 538: Rate Limiting ด้วย .NET 8

Rate Limiting ป้องกัน brute force attacks, DDoS, และการใช้ API เกินกว่าที่กำหนด

```csharp
// ใน Program.cs (.NET 8+)
using System.Threading.RateLimiting;

builder.Services.AddRateLimiter(options =>
{
    // 1. Fixed Window: 10 requests ต่อ 1 นาที
    options.AddFixedWindowLimiter("fixed", limiterOptions =>
    {
        limiterOptions.PermitLimit = 10;
        limiterOptions.Window = TimeSpan.FromMinutes(1);
        limiterOptions.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        limiterOptions.QueueLimit = 5;
    });

    // 2. Sliding Window: ละเอียดกว่า fixed window
    options.AddSlidingWindowLimiter("sliding", limiterOptions =>
    {
        limiterOptions.PermitLimit = 100;
        limiterOptions.Window = TimeSpan.FromMinutes(1);
        limiterOptions.SegmentsPerWindow = 6;  // แบ่ง 1 นาที เป็น 6 ส่วน (10 วินาทีต่อส่วน)
    });

    // 3. Token Bucket: รองรับ burst
    options.AddTokenBucketLimiter("token", limiterOptions =>
    {
        limiterOptions.TokenLimit = 100;          // bucket ขนาด 100 tokens
        limiterOptions.ReplenishmentPeriod = TimeSpan.FromSeconds(10);
        limiterOptions.TokensPerPeriod = 10;      // เติม 10 tokens ทุก 10 วินาที
        limiterOptions.AutoReplenishment = true;
    });

    // 4. Concurrency: จำกัด concurrent requests
    options.AddConcurrencyLimiter("concurrency", limiterOptions =>
    {
        limiterOptions.PermitLimit = 10;
        limiterOptions.QueueLimit = 50;
    });

    // Rate limit per IP address
    options.AddPolicy("per-ip", context =>
    {
        var ipAddress = context.Connection.RemoteIpAddress?.ToString() ?? "unknown";

        return RateLimitPartition.GetFixedWindowLimiter(ipAddress, _ => new FixedWindowRateLimiterOptions
        {
            PermitLimit = 50,
            Window = TimeSpan.FromMinutes(1)
        });
    });

    // Rate limit per user (authenticated)
    options.AddPolicy("per-user", context =>
    {
        var userId = context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;

        if (userId != null)
        {
            // Authenticated users: 1000 requests/minute
            return RateLimitPartition.GetFixedWindowLimiter(userId, _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 1000,
                Window = TimeSpan.FromMinutes(1)
            });
        }
        else
        {
            // Anonymous users: 20 requests/minute
            var ipAddress = context.Connection.RemoteIpAddress?.ToString() ?? "unknown";
            return RateLimitPartition.GetFixedWindowLimiter(ipAddress, _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 20,
                Window = TimeSpan.FromMinutes(1)
            });
        }
    });

    // Response เมื่อ rate limit เกิน
    options.OnRejected = async (context, cancellationToken) =>
    {
        context.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
        context.HttpContext.Response.Headers.Append("Retry-After", "60");
        await context.HttpContext.Response.WriteAsync("Rate limit exceeded. Try again later.", cancellationToken);
    };
});

app.UseRateLimiter();

// ใช้บน Controller/Action
[ApiController]
[Route("api/[controller]")]
public class AuthController : ControllerBase
{
    // ใช้ rate limiter ที่ strict กว่าสำหรับ login
    [HttpPost("login")]
    [EnableRateLimiting("per-ip")]   // จำกัดต่อ IP
    public async Task<IActionResult> Login([FromBody] LoginRequest request)
    {
        // ...
        return Ok();
    }

    // ไม่ rate limit สำหรับ health check
    [HttpGet("health")]
    [DisableRateLimiting]
    public IActionResult Health() => Ok("Healthy");
}
```

---

## ขั้นตอนที่ 539: CORS Configuration

```csharp
// ============================================================
// CORS (Cross-Origin Resource Sharing)
// ============================================================

// ใน Program.cs
builder.Services.AddCors(options =>
{
    // Policy สำหรับ Production
    options.AddPolicy("ProductionPolicy", policy =>
    {
        policy
            .WithOrigins(
                "https://app.example.com",
                "https://admin.example.com"
            )
            .WithMethods("GET", "POST", "PUT", "DELETE", "PATCH")
            .WithHeaders("Authorization", "Content-Type", "X-Request-ID")
            .AllowCredentials()         // อนุญาต cookies/authorization headers
            .SetPreflightMaxAge(TimeSpan.FromMinutes(10));  // cache preflight
    });

    // Policy สำหรับ Development (ผ่อนปรนกว่า)
    options.AddPolicy("DevelopmentPolicy", policy =>
    {
        policy
            .WithOrigins("http://localhost:3000", "http://localhost:5173")
            .AllowAnyMethod()
            .AllowAnyHeader()
            .AllowCredentials();
    });

    // ❌ อย่าทำแบบนี้ใน Production:
    // policy.AllowAnyOrigin().AllowCredentials(); // จะ throw exception
    // policy.AllowAnyOrigin(); // เปิดให้ทุก origin เข้าถึงได้
});

// ใช้ policy ที่ต่างกันตาม environment
if (app.Environment.IsDevelopment())
    app.UseCors("DevelopmentPolicy");
else
    app.UseCors("ProductionPolicy");

// ใช้ CORS บน specific controller
[ApiController]
[Route("api/[controller]")]
[EnableCors("ProductionPolicy")]
public class PublicApiController : ControllerBase
{
    // ...
}

// ============================================================
// CORS Security Considerations
// ============================================================

// ✅ ใช้ specific origins (ไม่ใช่ *)
// ✅ ใช้ specific methods และ headers
// ✅ ตรวจสอบ origin ใน server ด้วย (ไม่เชื่อ CORS header อย่างเดียว)
// ✅ อย่าอนุญาต credentials กับ AllowAnyOrigin

// Dynamic CORS validation (สำหรับ multi-tenant)
builder.Services.AddCors(options =>
{
    options.AddPolicy("DynamicPolicy", policy =>
    {
        policy.SetIsOriginAllowed(origin =>
        {
            if (Uri.TryCreate(origin, UriKind.Absolute, out var uri))
            {
                // อนุญาตเฉพาะ subdomains ของ example.com
                return uri.Host.EndsWith(".example.com", StringComparison.OrdinalIgnoreCase)
                    && uri.Scheme == "https";
            }
            return false;
        })
        .AllowAnyMethod()
        .AllowAnyHeader()
        .AllowCredentials();
    });
});
```

---

## ขั้นตอนที่ 540: Anti-Forgery Tokens

Anti-Forgery Token ป้องกัน CSRF (Cross-Site Request Forgery)  
เมื่อ browser ส่ง request โดยอัตโนมัติ (เช่น จาก malicious site) token จะไม่ตรงกัน

```csharp
// ============================================================
// 1. Anti-Forgery ใน ASP.NET Core MVC (Razor Pages/Views)
// ============================================================

// ใน Program.cs
builder.Services.AddAntiforgery(options =>
{
    options.HeaderName = "X-CSRF-TOKEN";  // Header name สำหรับ SPA
    options.Cookie.HttpOnly = true;
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
    options.Cookie.SameSite = SameSiteMode.Strict;
    options.Cookie.Name = "CSRF-TOKEN";
    options.SuppressXFrameOptionsHeader = false;
});

// Razor View — @Html.AntiForgeryToken() สร้าง hidden input
// <form method="post">
//     @Html.AntiForgeryToken()
//     <input type="text" name="email" />
//     <button type="submit">Submit</button>
// </form>

// Controller — [ValidateAntiForgeryToken] ตรวจสอบ token
[HttpPost]
[ValidateAntiForgeryToken]
public async Task<IActionResult> UpdateProfile([FromForm] UpdateProfileRequest request)
{
    // ถ้า token ไม่ตรง จะได้ 400 Bad Request อัตโนมัติ
    await _userService.UpdateProfileAsync(request);
    return RedirectToAction("Profile");
}

// ============================================================
// 2. Anti-Forgery สำหรับ SPA (Angular/React)
// ============================================================

// Angular ส่ง XSRF-TOKEN cookie และ X-XSRF-TOKEN header อัตโนมัติ
builder.Services.AddAntiforgery(options =>
{
    options.HeaderName = "X-XSRF-TOKEN";
    options.Cookie.Name = "XSRF-TOKEN";
    options.Cookie.HttpOnly = false; // ต้องให้ JavaScript อ่านได้เพื่อส่ง header
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
    options.Cookie.SameSite = SameSiteMode.Strict;
});

// Endpoint สำหรับ SPA ขอ CSRF token
[ApiController]
[Route("api/[controller]")]
public class CsrfController : ControllerBase
{
    private readonly IAntiforgery _antiforgery;

    public CsrfController(IAntiforgery antiforgery) => _antiforgery = antiforgery;

    [HttpGet("token")]
    public IActionResult GetCsrfToken()
    {
        var tokens = _antiforgery.GetAndStoreTokens(HttpContext);
        // ส่ง token ใน cookie และ response body
        Response.Cookies.Append("XSRF-TOKEN", tokens.RequestToken!, new CookieOptions
        {
            HttpOnly = false,
            Secure = true,
            SameSite = SameSiteMode.Strict
        });
        return Ok(new { csrfToken = tokens.RequestToken });
    }
}

// API Controller ที่ต้องการ CSRF token
[ApiController]
[Route("api/[controller]")]
[AutoValidateAntiforgeryToken]  // ตรวจสอบ POST/PUT/DELETE/PATCH อัตโนมัติ
public class TransactionController : ControllerBase
{
    [HttpPost("transfer")]
    public async Task<IActionResult> Transfer([FromBody] TransferRequest request)
    {
        // มาถึงตรงนี้ได้ = CSRF token valid แล้ว
        return Ok();
    }

    [HttpGet("history")]
    [IgnoreAntiforgeryToken]  // GET ไม่ต้อง CSRF token
    public async Task<IActionResult> GetHistory()
    {
        return Ok();
    }
}

// ============================================================
// 3. Security Audit Checklist สรุป
// ============================================================

/*
OWASP Top 10 Checklist for C# Applications:

[A01] Broken Access Control
  ✅ [Authorize] บนทุก endpoint ที่ต้อง auth
  ✅ Verify ownership ก่อน access resource (IDOR prevention)
  ✅ ไม่เปิด directory listing
  ✅ Deny by default: ถ้าไม่ได้ให้สิทธิ์ = ปฏิเสธ

[A02] Cryptographic Failures
  ✅ HTTPS ทุก endpoint + HSTS
  ✅ Hash passwords ด้วย PBKDF2/Argon2/BCrypt
  ✅ เข้ารหัส sensitive data ก่อนเก็บ (AES-256-GCM)
  ✅ ไม่ใช้ MD5/SHA1 สำหรับ security
  ✅ TLS 1.2+ เท่านั้น

[A03] Injection
  ✅ Parameterized queries (EF Core / Dapper)
  ✅ Input validation ทุก user input
  ✅ HTML encode output
  ✅ ไม่ใช้ shell execution กับ user input

[A04] Insecure Design
  ✅ Threat modeling ก่อน implement
  ✅ Secure by default (ไม่ต้อง opt-in เพื่อความปลอดภัย)
  ✅ Fail securely (error ไม่เปิดเผย internal details)

[A05] Security Misconfiguration
  ✅ ปิด default credentials
  ✅ ปิด directory listing
  ✅ Error handling ไม่เปิดเผย stack trace ให้ users
  ✅ Security headers ครบ (X-Frame-Options, CSP, etc.)
  ✅ ปิด debug mode ใน production

[A06] Vulnerable Components
  ✅ อัปเดต NuGet packages สม่ำเสมอ
  ✅ ใช้ dotnet list package --vulnerable
  ✅ Dependabot หรือ Snyk ตรวจสอบ CVE อัตโนมัติ

[A07] Authentication Failures
  ✅ Rate limit login attempts
  ✅ Account lockout หลัง N ครั้งผิด
  ✅ Secure session management
  ✅ JWT expiry สั้น + refresh token rotation
  ✅ Multi-factor authentication

[A08] Software and Data Integrity
  ✅ Verify NuGet packages (checksums)
  ✅ ไม่ deserialize untrusted data โดยไม่ validate
  ✅ CI/CD pipeline security

[A09] Security Logging & Monitoring
  ✅ Log authentication events (success/failure)
  ✅ Log authorization failures
  ✅ Log input validation failures
  ✅ Alert บน suspicious patterns

[A10] Server-Side Request Forgery (SSRF)
  ✅ Validate URLs ก่อน fetch
  ✅ Whitelist allowed hosts
  ✅ ไม่ส่ง response body จาก internal services โดยตรง
*/

// ============================================================
// ตรวจสอบ vulnerable packages
// ============================================================

// Terminal command:
// dotnet list package --vulnerable
// dotnet list package --outdated

// ตัวอย่าง code สำหรับ security logging
public class SecurityAuditLogger
{
    private readonly ILogger<SecurityAuditLogger> _logger;

    public SecurityAuditLogger(ILogger<SecurityAuditLogger> logger) => _logger = logger;

    public void LogLoginAttempt(string username, bool success, string ipAddress)
    {
        if (success)
        {
            _logger.LogInformation(
                "LOGIN_SUCCESS: User={Username} IP={IpAddress} Time={Time}",
                username, ipAddress, DateTime.UtcNow);
        }
        else
        {
            _logger.LogWarning(
                "LOGIN_FAILURE: User={Username} IP={IpAddress} Time={Time}",
                username, ipAddress, DateTime.UtcNow);
        }
    }

    public void LogUnauthorizedAccess(string userId, string resource, string action)
    {
        _logger.LogWarning(
            "UNAUTHORIZED_ACCESS: User={UserId} Resource={Resource} Action={Action} Time={Time}",
            userId, resource, action, DateTime.UtcNow);
    }

    public void LogSuspiciousActivity(string description, string ipAddress, Dictionary<string, object> context)
    {
        _logger.LogError(
            "SECURITY_ALERT: {Description} IP={IpAddress} Context={@Context}",
            description, ipAddress, context);
    }
}
```

---

## 📝 สรุป Part 54

| หัวข้อ | สิ่งที่ต้องทำ | ห้ามทำ |
|--------|--------------|--------|
| **Password** | ใช้ PBKDF2/Argon2/BCrypt | เก็บ plaintext, ใช้ MD5/SHA1 |
| **JWT** | อายุสั้น (15 นาที), rotate refresh token | เก็บ sensitive data ใน payload |
| **Encryption** | AES-256-GCM + RSA hybrid | DES, RC4, ECB mode |
| **SQL Injection** | Parameterized queries, EF Core | String interpolation ใน SQL |
| **Input Validation** | FluentValidation, whitelist | Blacklist เพียงอย่างเดียว |
| **Secrets** | Key Vault, env vars, user-secrets | Hardcode ใน source code |
| **Cookies** | HttpOnly + Secure + SameSite=Strict | JavaScript-readable auth cookies |
| **Rate Limiting** | Per-IP + per-user limiters | ไม่มี rate limit ใน auth endpoints |
| **CORS** | Specific origins, no wildcard+credentials | AllowAnyOrigin + AllowCredentials |
| **CSRF** | Anti-forgery tokens สำหรับ mutations | ไม่มี CSRF protection บน POST/PUT/DELETE |

---

### Quick Reference: NuGet Packages

```text
BCrypt.Net-Next                          — BCrypt password hashing
Konscious.Security.Cryptography.Argon2  — Argon2 password hashing
System.IdentityModel.Tokens.Jwt         — JWT creation/validation
Microsoft.IdentityModel.Tokens          — Token validation parameters
Azure.Extensions.AspNetCore.Configuration.Secrets — Azure Key Vault
Azure.Identity                          — Azure authentication
AWSSDK.SecretsManager                   — AWS Secrets Manager
FluentValidation.AspNetCore             — Input validation
Microsoft.AspNetCore.DataProtection     — DPAPI (built-in)
```

---

**ก่อนหน้า → [Part 53: Performance Optimization](part53-performance.md)**  
**ต่อไป → [Part 55: Async Advanced](part55-async-advanced.md)**
