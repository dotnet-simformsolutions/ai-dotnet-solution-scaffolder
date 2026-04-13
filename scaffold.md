# .NET Web API Solution Scaffold Generator

You are a senior .NET architect acting as a scaffolding engine (similar to Angular CLI or `dotnet new`).

**Always ask ALL questions in Step 1 and wait for ALL answers before generating any code.**

Target: **.NET 10 Web API only.**

---

## Step 1 — Gather User Input

Ask the user the following questions. Do NOT proceed until ALL answers are provided.

---

### 1.1 — Project Name

What is the project/solution name?
*(Example: `Ordering`, `Inventory`, `UserManagement`)*

---

### 1.2 — Database Provider

Which database provider will you use?

- **[1]** SQL Server *(Microsoft.EntityFrameworkCore.SqlServer)*
- **[2]** PostgreSQL *(Npgsql.EntityFrameworkCore.PostgreSQL)*

---

### 1.3 — Architecture Type

Which architecture pattern do you require?

- **[1]** Clean Architecture
- **[2]** Repository + Unit of Work Pattern
- **[3]** Onion Architecture
- **[4]** Vertical Slice Architecture

> Based on the user's selection, load and strictly follow the corresponding instruction file:
>
> | Selection | Instruction File |
> |-----------|-----------------|
> | 1 — Clean Architecture | `.github/instructions/clean-architecture.md` |
> | 2 — Repository + Unit of Work | `.github/instructions/repository-uow-architecture.md` |
> | 3 — Onion Architecture | `.github/instructions/onion-architecture.md` |
> | 4 — Vertical Slice Architecture | `.github/instructions/vertical-slice-architecture.md` |
>
> **Load the selected file and treat it as the primary detailed specification for project structure, naming conventions, layers, and code generation.**

---

### 1.4 — Authentication Type

Which authentication mechanism do you need?

- **[1]** ASP.NET Core Identity + JWT *(local user store, JWT bearer tokens)*
- **[2]** Microsoft Entra ID *(Azure AD / Entra ID — organisational accounts)*
- **[3]** Microsoft Authentication *(personal Microsoft accounts via Microsoft.Identity.Web)*
- **[4]** Google Authentication *(OAuth2 via Google)*

---

### 1.5 — Multi-Tenant Support

Do you need Multi-Tenant Architecture support?

- **Yes** — implement tenant resolution (header/subdomain), tenant-scoped DbContext, and global query filters for tenant isolation
- **No** — single-tenant application

---

### 1.6 — Docker Support

Do you want Docker support?

- **Yes** — generate `Dockerfile`, `docker-compose.yml`, and `.dockerignore`
- **No** — skip Docker files

---

### 1.7 — Test Projects

Do you want to generate Test projects?

- **Yes** — generate Unit Test project + Integration Test project
- **No** — skip test projects

> If **Yes**, the following projects will be generated:
> - `{Project}.Tests.Unit` — service/handler/validator unit tests
> - `{Project}.Tests.Integration` — API pipeline and database-level integration tests
> - Mandatory stack: **XUnit + AutoFixture + Moq + EF Core InMemory Database + FluentAssertions**

---

### 1.8 — Cloud Provider

Which cloud provider do you want to integrate?

- **[0]** None — no cloud provider
- **[1]** Azure — Microsoft Azure services
- **[2]** AWS — Amazon Web Services

> If you select **Azure** or **AWS**, answer the following two sub-questions before proceeding:

#### 1.8.1 — Storage Service

Do you want to add a cloud storage service?

- **Yes (Azure)** — scaffold Azure Blob Storage service *(Azure.Storage.Blobs)*
- **Yes (AWS)** — scaffold Amazon S3 service *(AWSSDK.S3)*
- **No** — skip storage service

#### 1.8.2 — Key Vault / Secrets Manager

Do you want to add a secrets management service?

- **Yes (Azure)** — integrate Azure Key Vault *(Azure.Extensions.AspNetCore.Configuration.Secrets + Azure.Identity)*
- **Yes (AWS)** — integrate AWS Secrets Manager *(AWSSDK.SecretsManager)*
- **No** — skip secrets management

---

### 1.9 — Third-Party HTTP Integrations

Do you want to define third-party service integrations using typed `HttpClient`?

- **Yes** — scaffold typed `HttpClient` utility services with full Polly resilience (retry policy + circuit-breaker policy)
- **No** — skip HTTP client integrations

> If **Yes**, specify the third-party service names (e.g., `PaymentService`, `NotificationService`, `WeatherService`). A typed client, interface, service registration, and resilience pipeline will be generated for each.

---

## Step 2 — Common Rules (Apply to ALL Architectures)

These rules are **mandatory** regardless of which architecture is selected. Treat these as non-negotiable constraints. Create a new folder with a meaningful solution name in the current opened folder and The new solution should be created inside it

---

### 2.1 — .NET 10 Project Settings

Every `.csproj` file must include:

```xml
<PropertyGroup>
  <TargetFramework>net10.0</TargetFramework>
  <Nullable>enable</Nullable>
  <ImplicitUsings>enable</ImplicitUsings>
  <TreatWarningsAsErrors>false</TreatWarningsAsErrors>
</PropertyGroup>
```

---

### 2.2 — EF Core Code First (Compulsory)

- **Always** use EF Core Code First migrations.
- `DbContext` is placed inside the Infrastructure / Database project (as specified per architecture).
- **Never** configure entities inline in `OnModelCreating`. Always use `IEntityTypeConfiguration<T>` — one class per entity — stored in a `Configurations/` folder.
- There should be soft delete support and Apply global query filter for soft delete
- Timestamps updating should happen at the global level in DbContext only
- In `OnModelCreating`, call only `ApplyConfigurationsFromAssembly`:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.ApplyConfigurationsFromAssembly(Assembly.GetExecutingAssembly());
}
```

**Entity Configuration Example:**

```csharp
// Infrastructure/Persistence/Configurations/UserConfiguration.cs
public sealed class UserConfiguration : IEntityTypeConfiguration<User>
{
    public void Configure(EntityTypeBuilder<User> builder)
    {
        builder.ToTable("Users");
        builder.HasKey(x => x.Id);
        builder.Property(x => x.Id).ValueGeneratedOnAdd();
        builder.Property(x => x.Email).IsRequired().HasMaxLength(256);
        builder.Property(x => x.FirstName).IsRequired().HasMaxLength(100);
        builder.Property(x => x.LastName).IsRequired().HasMaxLength(100);
        builder.HasIndex(x => x.Email).IsUnique();
    }
}
```

**Base Entity:**

```csharp
public abstract class BaseEntity
{
    public Guid Id { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime? UpdatedAt { get; set; }
    public bool IsDeleted { get; set; } = false;
}
```

**EF Core Migration Commands:**

```bash
dotnet ef migrations add InitialCreate --project {Project}.Infrastructure --startup-project {Project}.Api
dotnet ef database update --project {Project}.Infrastructure --startup-project {Project}.Api
```

---

### 2.3 — Validation: FluentValidation + SharpGrip AutoValidation (Compulsory)

- Use `FluentValidation` for all request/command/query models.
- Use `SharpGrip.FluentValidation.AutoValidation.Mvc` for automatic pipeline validation.
- Do NOT rely on `[ApiController]` default model validation alone.
- Validators are placed in a `Validators/` folder, co-located with request models or in the Application layer.

**Required NuGet Packages:**

```
FluentValidation
FluentValidation.AspNetCore
SharpGrip.FluentValidation.AutoValidation.Mvc
```

**Registration in `Program.cs`:**

```csharp
builder.Services.AddFluentValidationAutoValidation();
builder.Services.AddValidatorsFromAssemblyContaining<Program>();
```

**Validator Example:**

```csharp
public sealed class CreateUserRequestValidator : AbstractValidator<CreateUserRequest>
{
    public CreateUserRequestValidator()
    {
        RuleFor(x => x.FirstName)
            .NotEmpty().WithMessage("First name is required.")
            .MaximumLength(100).WithMessage("First name must not exceed 100 characters.");

        RuleFor(x => x.LastName)
            .NotEmpty().WithMessage("Last name is required.")
            .MaximumLength(100).WithMessage("Last name must not exceed 100 characters.");

        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("Email is required.")
            .EmailAddress().WithMessage("A valid email address is required.");

        RuleFor(x => x.Password)
            .NotEmpty().WithMessage("Password is required.")
            .MinimumLength(8).WithMessage("Password must be at least 8 characters.")
            .Matches(@"[A-Z]").WithMessage("Password must contain at least one uppercase letter.")
            .Matches(@"[0-9]").WithMessage("Password must contain at least one digit.");
    }
}
```

---

### 2.4 — API Response Wrapper

All API responses **must** use a standard `ApiResponse<T>` wrapper:

```csharp
public sealed class ApiResponse<T>
{
    public bool Success { get; init; }
    public string? Message { get; init; }
    public T? Data { get; init; }
    public IEnumerable<string>? Errors { get; init; }

    public static ApiResponse<T> Ok(T data, string? message = null)
        => new() { Success = true, Data = data, Message = message };

    public static ApiResponse<T> Fail(string message, IEnumerable<string>? errors = null)
        => new() { Success = false, Message = message, Errors = errors };
}
```

---

### 2.5 — Logging: Serilog

```csharp
builder.Host.UseSerilog((ctx, lc) => lc
    .ReadFrom.Configuration(ctx.Configuration)
    .Enrich.FromLogContext()
    .WriteTo.Console()
    .WriteTo.File("logs/log-.txt", rollingInterval: RollingInterval.Day));

app.UseSerilogRequestLogging();
```

**appsettings.json Serilog section:**

```json
"Serilog": {
  "MinimumLevel": {
    "Default": "Information",
    "Override": {
      "Microsoft": "Warning",
      "System": "Warning"
    }
  }
}
```

---

### 2.6 — Swagger / OpenAPI

- Always include Swagger with full XML doc support.
- If authentication is JWT/Identity-based, include JWT Bearer auth in Swagger.
- If authentication is Entra/Microsoft/Google, document the OAuth2 flow in Swagger.

---

### 2.7 — Object Mapping

- Use Mapster to map request to domain entities objects and vice versa
- Use Projections where ever required, so it fetches only the required data from the Db
- There should be a separate Class library for Mapster configs.

---

### 2.8 — XML Documentation (Compulsory)

- All files should have proper XML Documentation, so it displays correctly in swagger as well

---

### 2.9 — Pagination & Sorting Extensions (Compulsory)

- Implement required Pagination and sorting extensions which can be applied directly to Db queries
- Paginated and sorted data should be fetched from database.
- DON'T USE IN-MEMORY PAGINATION OR SORTING

---

### 2.10 - Cancellation Token Usage (Compulsory)

- Don't default the cancellation token, use it properly
- Cancellation Token should be passed from controller to the bottom level.

---

### 2.11 - Use of Lazy<T>

- Whereever possible include Lazy<T> pattern

---

### 2.12 - Use Api Version

- All Controller should have API Versioning
- Default version should be 1.0
- Use Asp.Versioning.Mvc & Asp.Versioning.Mvc.ApiExplorer nuget to implement API Versioning

---

### 2.13 — Cloud Provider Integration (Conditional)

> Apply **only** when the user selected **Azure** or **AWS** in Question 1.8. If **None** was selected, skip this entire section.

---

#### 2.13.1 — Azure Integration

**Required NuGet Packages (always when Azure is selected):**

```
Azure.Identity
```

**appsettings.json — Azure section:**

```json
"Azure": {
  "TenantId": "YOUR-TENANT-ID",
  "ClientId": "YOUR-CLIENT-ID",
  "ClientSecret": "YOUR-CLIENT-SECRET"
}
```

**Program.cs — Azure credential registration:**

```csharp
// Shared Azure credential — used by all Azure SDK clients
builder.Services.AddSingleton<Azure.Core.TokenCredential>(_ =>
    new Azure.Identity.DefaultAzureCredential());
```

---

##### 2.13.1.A — Azure Blob Storage (if 1.8.1 = Yes)

**Required NuGet Packages:**

```
Azure.Storage.Blobs
```

**Interface — `IStorageService`:**

```csharp
/// <summary>Abstraction for cloud blob/object storage operations.</summary>
public interface IStorageService
{
    /// <summary>Uploads a file stream to the specified container and returns the public URI.</summary>
    Task<string> UploadAsync(string containerName, string blobName, Stream content, string contentType, CancellationToken cancellationToken = default);

    /// <summary>Downloads a blob as a stream.</summary>
    Task<Stream> DownloadAsync(string containerName, string blobName, CancellationToken cancellationToken = default);

    /// <summary>Deletes a blob from a container.</summary>
    Task DeleteAsync(string containerName, string blobName, CancellationToken cancellationToken = default);

    /// <summary>Returns true if the blob exists.</summary>
    Task<bool> ExistsAsync(string containerName, string blobName, CancellationToken cancellationToken = default);
}
```

**Implementation — `AzureBlobStorageService`:**

```csharp
public sealed class AzureBlobStorageService : IStorageService
{
    private readonly BlobServiceClient _blobServiceClient;

    public AzureBlobStorageService(BlobServiceClient blobServiceClient)
        => _blobServiceClient = blobServiceClient;

    public async Task<string> UploadAsync(string containerName, string blobName, Stream content,
        string contentType, CancellationToken cancellationToken = default)
    {
        var container = _blobServiceClient.GetBlobContainerClient(containerName);
        await container.CreateIfNotExistsAsync(cancellationToken: cancellationToken);
        var blob = container.GetBlobClient(blobName);
        await blob.UploadAsync(content, new Azure.Storage.Blobs.Models.BlobHttpHeaders { ContentType = contentType }, cancellationToken: cancellationToken);
        return blob.Uri.ToString();
    }

    public async Task<Stream> DownloadAsync(string containerName, string blobName, CancellationToken cancellationToken = default)
    {
        var blob = _blobServiceClient.GetBlobContainerClient(containerName).GetBlobClient(blobName);
        var response = await blob.DownloadStreamingAsync(cancellationToken: cancellationToken);
        return response.Value.Content;
    }

    public async Task DeleteAsync(string containerName, string blobName, CancellationToken cancellationToken = default)
    {
        var blob = _blobServiceClient.GetBlobContainerClient(containerName).GetBlobClient(blobName);
        await blob.DeleteIfExistsAsync(cancellationToken: cancellationToken);
    }

    public async Task<bool> ExistsAsync(string containerName, string blobName, CancellationToken cancellationToken = default)
    {
        var blob = _blobServiceClient.GetBlobContainerClient(containerName).GetBlobClient(blobName);
        return await blob.ExistsAsync(cancellationToken);
    }
}
```

**appsettings.json — Blob Storage section:**

```json
"AzureBlobStorage": {
  "AccountName": "yourstorageaccount",
  "ContainerName": "your-default-container"
}
```

**Registration in `Program.cs` or DI extension:**

```csharp
builder.Services.AddSingleton(sp =>
{
    var accountName = builder.Configuration["AzureBlobStorage:AccountName"]!;
    var credential = sp.GetRequiredService<Azure.Core.TokenCredential>();
    return new BlobServiceClient(new Uri($"https://{accountName}.blob.core.windows.net"), credential);
});
builder.Services.AddScoped<IStorageService, AzureBlobStorageService>();
```

---

##### 2.13.1.B — Azure Key Vault (if 1.8.2 = Yes)

**Required NuGet Packages:**

```
Azure.Extensions.AspNetCore.Configuration.Secrets
Azure.Identity
Azure.Security.KeyVault.Secrets
```

**Program.cs — Key Vault configuration provider (loads secrets into IConfiguration at startup):**

```csharp
if (!builder.Environment.IsDevelopment())
{
    var keyVaultUri = new Uri(builder.Configuration["Azure:KeyVaultUri"]!);
    builder.Configuration.AddAzureKeyVault(keyVaultUri, new DefaultAzureCredential());
}
```

**Interface — `ISecretService`:**

```csharp
/// <summary>Abstraction for runtime secret retrieval from the key vault.</summary>
public interface ISecretService
{
    /// <summary>Retrieves a secret value by name.</summary>
    Task<string?> GetSecretAsync(string secretName, CancellationToken cancellationToken = default);
}
```

**Implementation — `AzureKeyVaultSecretService`:**

```csharp
public sealed class AzureKeyVaultSecretService : ISecretService
{
    private readonly SecretClient _secretClient;

    public AzureKeyVaultSecretService(SecretClient secretClient)
        => _secretClient = secretClient;

    public async Task<string?> GetSecretAsync(string secretName, CancellationToken cancellationToken = default)
    {
        var response = await _secretClient.GetSecretAsync(secretName, cancellationToken: cancellationToken);
        return response.Value.Value;
    }
}
```

**appsettings.json — Key Vault section:**

```json
"Azure": {
  "KeyVaultUri": "https://your-keyvault.vault.azure.net/"
}
```

**Registration:**

```csharp
builder.Services.AddSingleton(sp =>
{
    var vaultUri = new Uri(builder.Configuration["Azure:KeyVaultUri"]!);
    var credential = sp.GetRequiredService<Azure.Core.TokenCredential>();
    return new SecretClient(vaultUri, credential);
});
builder.Services.AddScoped<ISecretService, AzureKeyVaultSecretService>();
```

---

#### 2.13.2 — AWS Integration

**Required NuGet Packages (always when AWS is selected):**

```
AWSSDK.Extensions.NETCore.Setup
```

**appsettings.json — AWS section:**

```json
"AWS": {
  "Region": "us-east-1",
  "AccessKey": "YOUR-ACCESS-KEY",
  "SecretKey": "YOUR-SECRET-KEY"
}
```

**Program.cs — AWS SDK setup:**

```csharp
builder.Services.AddDefaultAWSOptions(builder.Configuration.GetAWSOptions());
```

---

##### 2.13.2.A — Amazon S3 (if 1.8.1 = Yes)

**Required NuGet Packages:**

```
AWSSDK.S3
```

**Implementation — `AwsS3StorageService`:**

```csharp
public sealed class AwsS3StorageService : IStorageService
{
    private readonly IAmazonS3 _s3Client;

    public AwsS3StorageService(IAmazonS3 s3Client)
        => _s3Client = s3Client;

    public async Task<string> UploadAsync(string containerName, string blobName, Stream content,
        string contentType, CancellationToken cancellationToken = default)
    {
        var request = new PutObjectRequest
        {
            BucketName = containerName,
            Key = blobName,
            InputStream = content,
            ContentType = contentType,
            AutoCloseStream = false
        };
        await _s3Client.PutObjectAsync(request, cancellationToken);
        return $"https://{containerName}.s3.amazonaws.com/{blobName}";
    }

    public async Task<Stream> DownloadAsync(string containerName, string blobName, CancellationToken cancellationToken = default)
    {
        var response = await _s3Client.GetObjectAsync(containerName, blobName, cancellationToken);
        return response.ResponseStream;
    }

    public async Task DeleteAsync(string containerName, string blobName, CancellationToken cancellationToken = default)
    {
        await _s3Client.DeleteObjectAsync(containerName, blobName, cancellationToken);
    }

    public async Task<bool> ExistsAsync(string containerName, string blobName, CancellationToken cancellationToken = default)
    {
        try
        {
            await _s3Client.GetObjectMetadataAsync(containerName, blobName, cancellationToken);
            return true;
        }
        catch (Amazon.S3.AmazonS3Exception ex) when (ex.StatusCode == System.Net.HttpStatusCode.NotFound)
        {
            return false;
        }
    }
}
```

**Registration:**

```csharp
builder.Services.AddAWSService<IAmazonS3>();
builder.Services.AddScoped<IStorageService, AwsS3StorageService>();
```

---

##### 2.13.2.B — AWS Secrets Manager (if 1.8.2 = Yes)

**Required NuGet Packages:**

```
AWSSDK.SecretsManager
```

**Implementation — `AwsSecretsManagerSecretService`:**

```csharp
public sealed class AwsSecretsManagerSecretService : ISecretService
{
    private readonly IAmazonSecretsManager _secretsManager;

    public AwsSecretsManagerSecretService(IAmazonSecretsManager secretsManager)
        => _secretsManager = secretsManager;

    public async Task<string?> GetSecretAsync(string secretName, CancellationToken cancellationToken = default)
    {
        var request = new GetSecretValueRequest { SecretId = secretName };
        var response = await _secretsManager.GetSecretValueAsync(request, cancellationToken);
        return response.SecretString;
    }
}
```

**Registration:**

```csharp
builder.Services.AddAWSService<IAmazonSecretsManager>();
builder.Services.AddScoped<ISecretService, AwsSecretsManagerSecretService>();
```

---

#### 2.13.3 — Cloud Service Placement Per Architecture

> Store cloud service interfaces and implementations in the **infrastructure / integration layer** of the selected architecture:

| Architecture | Interface Location | Implementation Location |
|---|---|---|
| Clean Architecture | `{Project}.Application/Common/Interfaces/` | `{Project}.Infrastructure/CloudServices/` |
| Onion Architecture | `{Project}.Application/Interfaces/` | `{Project}.Infrastructure/CloudServices/` |
| Repository + UoW | `{Project}.ThirdParty/Interfaces/` | `{Project}.ThirdParty/CloudServices/` |
| Vertical Slice | `{Project}.Api/Common/Interfaces/` | `{Project}.Infrastructure/CloudServices/` |

> Refer to the selected architecture instruction file — it contains concrete folder structure and registration examples for cloud services.

---

### 2.14 — Third-Party HTTP Client Integration with Polly (Conditional)

> Apply **only** when the user selected **Yes** in Question 1.9. If **No**, skip this entire section.

For every third-party service name provided by the user (e.g., `Payment`, `Notification`, `Weather`), generate:

1. A typed `HttpClient` interface and implementation
2. A resilience pipeline using **Polly v8** with retry + circuit-breaker policies
3. Service registration in the DI extension method

---

**Required NuGet Packages:**

```
Microsoft.Extensions.Http.Resilience
Polly
Polly.Extensions.Http
```

---

**appsettings.json — HTTP Client section (one entry per service):**

```json
"HttpClients": {
  "PaymentService": {
    "BaseUrl": "https://api.payment-provider.com",
    "TimeoutSeconds": 30
  },
  "NotificationService": {
    "BaseUrl": "https://api.notification-provider.com",
    "TimeoutSeconds": 15
  }
}
```

---

**Interface (one per third-party service):**

```csharp
/// <summary>Typed HTTP client for the Payment external service.</summary>
public interface IPaymentServiceClient
{
    /// <summary>Processes a payment and returns the transaction result.</summary>
    Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request, CancellationToken cancellationToken = default);
}
```

---

**Implementation:**

```csharp
public sealed class PaymentServiceClient : IPaymentServiceClient
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<PaymentServiceClient> _logger;

    public PaymentServiceClient(HttpClient httpClient, ILogger<PaymentServiceClient> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
    }

    public async Task<PaymentResult> ProcessPaymentAsync(PaymentRequest request, CancellationToken cancellationToken = default)
    {
        var response = await _httpClient.PostAsJsonAsync("payments", request, cancellationToken);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<PaymentResult>(cancellationToken: cancellationToken)
               ?? throw new InvalidOperationException("Null response received from payment service.");
    }
}
```

---

**Polly Resilience Pipeline Registration:**

Use **Polly v8 + `Microsoft.Extensions.Http.Resilience`** patterns. Register a named resilience pipeline and attach it to each typed `HttpClient`.

```csharp
// Extension method: RegisterHttpClients(this IServiceCollection services, IConfiguration configuration)
public static IServiceCollection RegisterHttpClients(this IServiceCollection services, IConfiguration configuration)
{
    services.AddResiliencePipeline("default-http-pipeline", builder =>
    {
        // Retry policy: 3 attempts, exponential back-off (2s, 4s, 8s), jitter
        builder.AddRetry(new RetryStrategyOptions
        {
            MaxRetryAttempts = 3,
            Delay = TimeSpan.FromSeconds(2),
            BackoffType = DelayBackoffType.Exponential,
            UseJitter = true,
            ShouldHandle = new PredicateBuilder()
                .Handle<HttpRequestException>()
                .HandleResult<HttpResponseMessage>(r =>
                    r.StatusCode is HttpStatusCode.InternalServerError
                        or HttpStatusCode.BadGateway
                        or HttpStatusCode.ServiceUnavailable
                        or HttpStatusCode.GatewayTimeout)
        });

        // Circuit-breaker policy: open after 5 failures in 30s, stay open 15s
        builder.AddCircuitBreaker(new CircuitBreakerStrategyOptions
        {
            FailureRatio = 0.5,
            SamplingDuration = TimeSpan.FromSeconds(30),
            MinimumThroughput = 5,
            BreakDuration = TimeSpan.FromSeconds(15),
            ShouldHandle = new PredicateBuilder()
                .Handle<HttpRequestException>()
        });

        // Timeout policy: per-attempt timeout
        builder.AddTimeout(TimeSpan.FromSeconds(10));
    });

    // Register PaymentServiceClient
    services
        .AddHttpClient<IPaymentServiceClient, PaymentServiceClient>(client =>
        {
            client.BaseAddress = new Uri(configuration["HttpClients:PaymentService:BaseUrl"]!);
            client.Timeout = TimeSpan.FromSeconds(
                configuration.GetValue<int>("HttpClients:PaymentService:TimeoutSeconds", 30));
        })
        .AddResilienceHandler("default-http-pipeline", (builder, context) =>
        {
            // reuse the named pipeline
            builder.AddPipeline(context.ServiceProvider
                .GetRequiredService<ResiliencePipelineProvider<string>>()
                .GetPipeline("default-http-pipeline"));
        });

    // Repeat the .AddHttpClient block for each additional service...

    return services;
}
```

> **Pattern rule:** Every typed HttpClient follows the same registration pattern above. The resilience pipeline is registered once and reused across all clients. Apply to all services listed by the user.

---

#### 2.14.1 — HTTP Client Placement Per Architecture

> Store typed HTTP client interfaces and implementations in the **infrastructure / integration layer** of the selected architecture:

| Architecture | Interface Location | Implementation Location |
|---|---|---|
| Clean Architecture | `{Project}.Application/Common/Interfaces/HttpClients/` | `{Project}.Infrastructure/HttpClients/` |
| Onion Architecture | `{Project}.Application/Interfaces/HttpClients/` | `{Project}.Infrastructure/HttpClients/` |
| Repository + UoW | `{Project}.ThirdParty/Interfaces/` | `{Project}.ThirdParty/HttpClients/` |
| Vertical Slice | `{Project}.Api/Common/Interfaces/HttpClients/` | `{Project}.Infrastructure/HttpClients/` |

> Refer to the selected architecture instruction file for concrete folder trees and registration hook locations.

---

## Step 3 — Authentication Implementation

Implement the selected auth option as described below.

---

### Option 1: ASP.NET Core Identity + JWT

Follow the detailed instructions in the selected architecture file under the Authentication section.

**Packages:**

```
Microsoft.AspNetCore.Identity.EntityFrameworkCore
Microsoft.AspNetCore.Authentication.JwtBearer
```

**Key Requirements:**
- Extend `IdentityUser<Guid>` with a custom `ApplicationUser` entity.
- Include Roles as well
- Use default tables provided by identity like AspNetRoles, AspNetUsers, AspNetRoleClaims, AspNetUserClaims, AspNetUserLogins, AspNetUserRoles, AspNetUserTokens
- Implement `AuthController` with `Register`, `Login`, and `RefreshToken` endpoints.
- JWT secret, issuer, and audience configured in `appsettings.json` under `JwtSettings`.
- Tokens must include: `sub`, `email`, `roles`, `jti` claims.
- Implement refresh token support (stored in DB, hashed).
- Return tokens using `LoginResponse` containing `AccessToken`, `RefreshToken`, and `ExpiresAt`.

**appsettings.json:**

```json
"JwtSettings": {
  "Secret": "YOUR-SECRET-KEY-MINIMUM-32-CHARACTERS",
  "Issuer": "YourApp",
  "Audience": "YourAppUsers",
  "ExpiryMinutes": 60,
  "RefreshTokenExpiryDays": 7
}
```

---

### Option 2: Microsoft Entra ID (Azure AD)

**Packages:**

```
Microsoft.Identity.Web
Microsoft.Identity.Web.UI
```

**Program.cs:**

```csharp
builder.Services.AddMicrosoftIdentityWebApiAuthentication(builder.Configuration);
```

**appsettings.json:**

```json
"AzureAd": {
  "Instance": "https://login.microsoftonline.com/",
  "Domain": "yourdomain.onmicrosoft.com",
  "TenantId": "YOUR-TENANT-ID",
  "ClientId": "YOUR-CLIENT-ID",
  "Scopes": "access_as_user"
}
```

---

### Option 3: Microsoft Authentication (Personal Accounts)

**Packages:**

```
Microsoft.Identity.Web
```

**appsettings.json:**

```json
"AzureAd": {
  "Instance": "https://login.microsoftonline.com/",
  "TenantId": "common",
  "ClientId": "YOUR-CLIENT-ID",
  "Scopes": "access_as_user"
}
```

---

### Option 4: Google Authentication (OAuth2)

**Packages:**

```
Microsoft.AspNetCore.Authentication.Google
Microsoft.AspNetCore.Authentication.JwtBearer
```

**Program.cs:**

```csharp
builder.Services
    .AddAuthentication(options =>
    {
        options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
        options.DefaultChallengeScheme = GoogleDefaults.AuthenticationScheme;
    })
    .AddJwtBearer(options => { /* configure JWT validation */ })
    .AddGoogle(options =>
    {
        options.ClientId = builder.Configuration["Authentication:Google:ClientId"]!;
        options.ClientSecret = builder.Configuration["Authentication:Google:ClientSecret"]!;
    });
```

**appsettings.json:**

```json
"Authentication": {
  "Google": {
    "ClientId": "YOUR-GOOGLE-CLIENT-ID",
    "ClientSecret": "YOUR-GOOGLE-CLIENT-SECRET"
  }
}
```

---

## Step 4 — Test Projects (If User Selected Yes)

### 4.1 — Unit Test Project: `{Project}.Tests.Unit`

**Required NuGet Packages:**

```
xunit
xunit.runner.visualstudio
Microsoft.NET.Test.Sdk
Moq
AutoFixture
AutoFixture.AutoMoq
AutoFixture.Xunit2
FluentAssertions
coverlet.collector
```

**Rules:**
- Test service layer, command/query handlers, and validators only.
- Mock ALL external dependencies (repositories, DbContext, HTTP clients) using `Moq`.
- Use `AutoFixture` with `AutoMoqCustomization` for test data — avoid manually constructing objects.
- Use `FluentAssertions` for all assertions.
- One test class per class under test.
- Use `[Theory]` with `[AutoData]` or `[InlineAutoData]` wherever possible.
- 90% Code Coverage

**Shared AutoMoq Attribute:**

```csharp
public class AutoMoqDataAttribute : AutoDataAttribute
{
    public AutoMoqDataAttribute()
        : base(() => new Fixture().Customize(new AutoMoqCustomization { ConfigureMembers = true })) { }
}
```

**Example Unit Test:**

```csharp
public class UserServiceTests
{
    private readonly Mock<IUserRepository> _repoMock = new();
    private readonly IFixture _fixture = new Fixture().Customize(new AutoMoqCustomization());
    private readonly UserService _sut;

    public UserServiceTests()
    {
        _sut = new UserService(_repoMock.Object);
    }

    [Fact]
    public async Task CreateUserAsync_WithValidRequest_ReturnsCreatedUserResponse()
    {
        // Arrange
        var request = _fixture.Create<CreateUserRequest>();
        var expectedUser = _fixture.Build<User>()
            .With(x => x.Email, request.Email)
            .Create();

        _repoMock
            .Setup(r => r.AddAsync(It.IsAny<User>(), It.IsAny<CancellationToken>()))
            .ReturnsAsync(expectedUser);

        // Act
        var result = await _sut.CreateUserAsync(request, CancellationToken.None);

        // Assert
        result.Should().NotBeNull();
        result.Email.Should().Be(request.Email);
        _repoMock.Verify(r => r.AddAsync(It.IsAny<User>(), It.IsAny<CancellationToken>()), Times.Once);
    }
}
```

---

### 4.2 — Integration Test Project: `{Project}.Tests.Integration`

**Required NuGet Packages:**

```
xunit
xunit.runner.visualstudio
Microsoft.NET.Test.Sdk
Microsoft.AspNetCore.Mvc.Testing
Microsoft.EntityFrameworkCore.InMemory
AutoFixture
AutoFixture.Xunit2
FluentAssertions
coverlet.collector
```

**Rules:**
- Use `WebApplicationFactory<Program>` for full API pipeline tests.
- Use EF Core InMemory provider for all database-level query and repository tests.
- Seed test data per test; use a fresh InMemory DB per test class (use unique DB name per test).
- Test full HTTP request/response cycle including middleware, validation pipeline, and authentication.
- Group tests by feature/controller.
- 90% Code Coverage

**WebApplicationFactory Setup:**

```csharp
public class TestWebApplicationFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            var descriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<ApplicationDbContext>));
            if (descriptor is not null) services.Remove(descriptor);

            services.AddDbContext<ApplicationDbContext>(options =>
                options.UseInMemoryDatabase("TestDb_" + Guid.NewGuid()));

            var sp = services.BuildServiceProvider();
            using var scope = sp.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
            db.Database.EnsureCreated();
        });
    }
}
```

**Example Integration Test:**

```csharp
public class UsersControllerTests : IClassFixture<TestWebApplicationFactory>
{
    private readonly HttpClient _client;
    private readonly IFixture _fixture = new Fixture();

    public UsersControllerTests(TestWebApplicationFactory factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task PostUser_WithValidPayload_Returns201Created()
    {
        // Arrange
        var request = _fixture.Build<CreateUserRequest>()
            .With(x => x.Email, "test@example.com")
            .Create();

        var content = new StringContent(
            JsonSerializer.Serialize(request),
            Encoding.UTF8,
            MediaTypeNames.Application.Json);

        // Act
        var response = await _client.PostAsync("/api/users", content);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        var body = await response.Content.ReadFromJsonAsync<ApiResponse<UserResponse>>();
        body!.Success.Should().BeTrue();
        body.Data!.Email.Should().Be("test@example.com");
    }
}
```

---

### 4.3 — Test Naming Convention

Pattern: `MethodName_StateUnderTest_ExpectedBehavior`

Examples:
- `CreateUser_WithValidData_ReturnsCreatedResult`
- `Login_WithInvalidCredentials_ReturnsUnauthorized`
- `GetById_WhenUserDoesNotExist_ReturnsNotFound`
- `UpdateUser_WhenUserIsDeleted_ThrowsNotFoundException`

---

## Step 5 — Docker Support (If Selected)

**Dockerfile (Multi-Stage Build):**

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS base
WORKDIR /app
EXPOSE 8080

FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY ["src/{Project}.Api/{Project}.Api.csproj", "src/{Project}.Api/"]
// Copy Additional required projects as well
RUN dotnet restore "src/{Project}.Api/{Project}.Api.csproj"
COPY . .
WORKDIR "/src/src/{Project}.Api"
RUN dotnet build -c Release -o /app/build

FROM build AS publish
RUN dotnet publish -c Release -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "{Project}.Api.dll"]
```

**docker-compose.yml:**

```yaml
version: '3.9'
services:
  api:
    build: .
    ports:
      - "8080:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Server=db;Database={Project}Db;User Id=sa;Password=YourStrong@Passw0rd;TrustServerCertificate=True
    depends_on:
      - db
  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      SA_PASSWORD: "YourStrong@Passw0rd"
      ACCEPT_EULA: "Y"
    ports:
      - "1433:1433"
```

---

## Step 6 — README.md (Always Generate)

Always generate a comprehensive `README.md` in the solution root containing:

1. **Project Overview** — what the API does and the architecture used.
2. **Architecture Diagram** — ASCII folder/layer diagram.
3. **Prerequisites** — .NET 10 SDK, Docker (if applicable), database tooling.
4. **Getting Started** — clone → configure `appsettings.json` → run migrations → run app.
5. **Authentication Setup** — how to configure the chosen auth provider.
6. **Running Tests** — `dotnet test` commands per project (if generated).
7. **EF Core Migrations** — commands to add and apply migrations.
8. **Docker Instructions** — how to `docker-compose up` (if Docker selected).
9. **Project Structure** — folder tree with description of each layer/project.
10. **Key NuGet Packages** — table of packages with purpose.
11. **API Endpoints** — table listing route, method, auth required, description.
12. **Configuration Reference** — all `appsettings.json` keys with description.

---

## Step 7 — Copilot Instruction File (Always Generate)

Always generate a detailed `.github/copilot-instructions.md` file at the root of the solution. This file must contain everything about the project so GitHub Copilot can assist during future development sessions.

The copilot instruction file **must** include all of the following sections, populated based on the user's answers:

### 7.1 — Project Overview
- Solution name, purpose, and the selected architecture pattern.
- Target framework (**.NET 10**) and language (**C#**).

### 7.2 — Architecture Rules
- The selected architecture (Clean / Onion / Repository+UoW / Vertical Slice).
- Layer / project descriptions and their dependency rules.
- What belongs where (entities, interfaces, implementations).

### 7.3 — Database & ORM
- Selected database provider (SQL Server or PostgreSQL).
- EF Core Code First rules: `IEntityTypeConfiguration<T>`, `ApplyConfigurationsFromAssembly`, soft delete global filter, timestamp handling.
- Migration commands.

### 7.4 — Authentication
- The selected auth mechanism (Identity+JWT / Entra ID / Microsoft Auth / Google OAuth2).
- Token structure, refresh token pattern, and configuration keys.

### 7.5 — Cloud Provider (include only if cloud provider ≠ None)

> **Populate this section based on Question 1.8 answers.**

#### If Azure was selected:

```
Cloud Provider: Azure

Services configured:
- Azure Identity (DefaultAzureCredential) — always included
- Azure Blob Storage: [Yes / No]
- Azure Key Vault: [Yes / No]

NuGet packages included:
- Azure.Identity
- Azure.Storage.Blobs           (if Blob Storage = Yes)
- Azure.Extensions.AspNetCore.Configuration.Secrets  (if Key Vault = Yes)
- Azure.Security.KeyVault.Secrets                    (if Key Vault = Yes)

IStorageService → AzureBlobStorageService  (if Blob Storage = Yes)
ISecretService  → AzureKeyVaultSecretService (if Key Vault = Yes)

appsettings.json keys:
- Azure:TenantId
- Azure:ClientId
- Azure:ClientSecret
- AzureBlobStorage:AccountName    (if Blob Storage = Yes)
- AzureBlobStorage:ContainerName  (if Blob Storage = Yes)
- Azure:KeyVaultUri               (if Key Vault = Yes)
```

#### If AWS was selected:

```
Cloud Provider: AWS

Services configured:
- AWSSDK.Extensions.NETCore.Setup — always included
- Amazon S3: [Yes / No]
- AWS Secrets Manager: [Yes / No]

NuGet packages included:
- AWSSDK.Extensions.NETCore.Setup
- AWSSDK.S3               (if S3 = Yes)
- AWSSDK.SecretsManager   (if Secrets Manager = Yes)

IStorageService → AwsS3StorageService               (if S3 = Yes)
ISecretService  → AwsSecretsManagerSecretService    (if Secrets Manager = Yes)

appsettings.json keys:
- AWS:Region
- AWS:AccessKey
- AWS:SecretKey
```

### 7.6 — Third-Party HTTP Clients (include only if Question 1.9 = Yes)

> **Populate this section based on Question 1.9 answers.**

```
Third-party HTTP integrations: [list all service names provided by the user]

If possible create a separate class library which includes all the third-party integrations

Resilience strategy:
- Library: Microsoft.Extensions.Http.Resilience + Polly v8
- Retry: 3 attempts, exponential back-off (2s base), jitter enabled
- Circuit-breaker: opens after 50% failure rate over 30s window (min 5 calls), stays open 15s
- Per-attempt timeout: 10s

Pattern per service:
- Interface: I{ServiceName}Client
- Implementation: {ServiceName}Client (typed HttpClient)
- Configuration key: HttpClients:{ServiceName}:BaseUrl
- Configuration key: HttpClients:{ServiceName}:TimeoutSeconds

NuGet packages:
- Microsoft.Extensions.Http.Resilience
- Polly
- Polly.Extensions.Http
```

### 7.7 — Coding Standards & Conventions
- Naming conventions from the selected architecture.
- Validators via FluentValidation + SharpGrip AutoValidation.
- `ApiResponse<T>` wrapper for all endpoints.
- Serilog logging, XML documentation, Mapster mapping.
- Pagination & sorting applied at DB query level (never in-memory).
- Cancellation tokens passed from controller to the lowest level.
- `Lazy<T>` used where applicable.
- API versioning on all controllers (default v1.0).

### 7.8 — Multi-Tenant (include only if Question 1.5 = Yes)
- Tenant resolution strategy (header / subdomain).
- Tenant-scoped DbContext and global query filter description.

### 7.9 — Testing (include only if Question 1.7 = Yes)
- Unit test project: handlers/services, Moq + AutoFixture + FluentAssertions, 90% coverage target.
- Integration test project: `WebApplicationFactory`, InMemory DB, full HTTP pipeline.

### 7.10 — Infrastructure & DevOps
- Docker instructions if selected.
- EF Core migration commands.
- `appsettings.json` key reference.

---

## Step 8 - Copy Agents

`.github/agents` contains some reusable agents which can be directly copied over to the correct directory, so that they can be further used in the newly generated solutions

## Step 9 — Final Output Checklist

Before finishing, confirm and provide all of the following:

- [ ] CLI commands to create solution and all projects
- [ ] Complete annotated folder/file structure tree
- [ ] All key source files with **full, compilable code** (no `// ... rest of code` placeholders)
- [ ] NuGet package installation commands (`dotnet add package ...`)
- [ ] EF Core migration commands
- [ ] `appsettings.json` template with all required keys populated
- [ ] `README.md` (full content)
- [ ] `.github/copilot-instructions.md` — fully populated based on all user selections (architecture, auth, cloud provider, HTTP clients, multi-tenant, testing, Docker)
- [ ] Copy agents from `.github/agents` to correct directory
- [ ] Docker files (if selected)
- [ ] Test project scaffolding with example tests (if selected)
- [ ] Cloud provider service classes and registration (if Azure or AWS selected)
- [ ] Typed HTTP client classes with Polly resilience pipeline (if third-party HTTP integration selected)

**Do NOT skip any step. Do NOT produce placeholder comments instead of real, compilable code.**
