# Clean Architecture — Detailed Instruction File

> This file is loaded automatically when the user selects **Clean Architecture** in `scaffold.md`.
> Follow every section completely. Do not skip layers, rules, or code examples.

---

## 1. Architecture Overview

Clean Architecture enforces strict **dependency inversion**: outer layers depend on inner layers; inner layers know nothing about outer layers.

```
┌─────────────────────────────────────────────────────────┐
│                        API Layer                        │
│  Controllers, Middleware, Filters, DI Registration      │
├─────────────────────────────────────────────────────────┤
│                  Infrastructure Layer                   │
│  EF Core DbContext, Repositories, Identity, Services    │
├─────────────────────────────────────────────────────────┤
│                   Application Layer                     │
│  Commands, Queries, Handlers (MediatR), DTOs,           │
│  Validators, IRepository Interfaces, Mappings           │
├─────────────────────────────────────────────────────────┤
│                     Domain Layer                        │
│  Entities, Value Objects, Domain Events, Enums,         │
│  Domain Exceptions, IRepository Interfaces (contracts)  │
└─────────────────────────────────────────────────────────┘
       Dependencies point INWARD only (↑)
```

**Core Principle:** The Domain and Application layers must have zero knowledge of EF Core, HTTP, or any framework.

---

## 2. Solution Structure

### Solution Name: `{Project}`

```
{Project}.sln
│
├── src/
│   ├── {Project}.Api/
│   ├── {Project}.Application/
│   ├── {Project}.Domain/
│   ├── {Project}.Extensions/
│   └── {Project}.Infrastructure/
│
└── tests/  (if test projects selected)
    ├── {Project}.Tests.Unit/
    └── {Project}.Tests.Integration/
```

---

## 3. Project Descriptions

| Project | Type | Responsibility |
|---------|------|----------------|
| `{Project}.Api` | ASP.NET Core Web API | HTTP entry point, controllers, middleware, DI wiring |
| `{Project}.Application` | Class Library | Use cases as Commands/Queries, MediatR handlers, DTOs, validators, interfaces |
| `{Project}.Domain` | Class Library | Core business entities, value objects, domain events, domain exceptions, enums |
| `{Project}.Extensions` | Class Library | Dynamic service and repository registration via reflection and marker interfaces |
| `{Project}.Infrastructure` | Class Library | EF Core DbContext, repository implementations, Identity, external service integrations |

---

## 4. Project Reference Rules

```
{Project}.Api
    ├── references {Project}.Application
    ├── references {Project}.Infrastructure
    └── references {Project}.Extensions       ← only for DI registration

{Project}.Application
    └── references {Project}.Domain

{Project}.Infrastructure
    ├── references {Project}.Application      ← implements Application interfaces
    └── references {Project}.Domain

{Project}.Extensions
    ├── references {Project}.Application      ← resolves service/repository interfaces
    └── references {Project}.Infrastructure   ← scans concrete implementations

{Project}.Domain
    └── NO references (zero dependencies)
```

> **Critical Rule:** `{Project}.Api` must **not** reference `{Project}.Infrastructure` in business logic — only in `Program.cs` for DI registration.
> `{Project}.Domain` must **never** reference any other project.
> `{Project}.Extensions` is a pure registration helper — it must contain no business logic.

---

## 5. Naming Conventions

| Concept | Naming Pattern | Example |
|---------|---------------|---------|
| Solution/Project | `{Project}.{Layer}` | `Ordering.Api` |
| Namespaces | Match folder structure | `Ordering.Application.Features.Users.Commands` |
| Commands | `{Action}{Entity}Command` | `CreateUserCommand` |
| Command Handlers | `{Action}{Entity}CommandHandler` | `CreateUserCommandHandler` |
| Queries | `{Action}{Entity}Query` | `GetUserByIdQuery` |
| Query Handlers | `{Action}{Entity}QueryHandler` | `GetUserByIdQueryHandler` |
| Request DTOs | `{Action}{Entity}Request` | `CreateUserRequest` |
| Response DTOs | `{Entity}Response` | `UserResponse` |
| Validators | `{Request}Validator` | `CreateUserRequestValidator` |
| Repository Interface | `I{Entity}Repository` | `IUserRepository` |
| Repository Impl | `{Entity}Repository` | `UserRepository` |
| Entity Config | `{Entity}Configuration` | `UserConfiguration` |
| Domain Exception | `{Entity}{Issue}Exception` | `UserNotFoundException` |

---

## 6. Folder Structure

### `{Project}.Domain`

```
{Project}.Domain/
├── Entities/
│   ├── BaseEntity.cs
│   └── User.cs
├── Enums/
│   └── UserStatus.cs
├── Events/
│   └── UserCreatedEvent.cs
├── Exceptions/
│   ├── DomainException.cs
│   └── UserNotFoundException.cs
└── ValueObjects/
    └── Email.cs
```

### `{Project}.Application`

```
{Project}.Application/
├── Common/
│   ├── Behaviours/
│   │   ├── ValidationBehaviour.cs
│   │   └── LoggingBehaviour.cs
│   ├── Interfaces/
│   │   ├── IApplicationDbContext.cs
│   │   └── ITokenService.cs
│   └── Models/
│       └── ApiResponse.cs
├── Features/
│   └── Users/
│       ├── Commands/
│       │   ├── CreateUser/
│       │   │   ├── CreateUserCommand.cs
│       │   │   ├── CreateUserCommandHandler.cs
│       │   │   └── CreateUserCommandValidator.cs
│       │   └── DeleteUser/
│       │       ├── DeleteUserCommand.cs
│       │       └── DeleteUserCommandHandler.cs
│       ├── Queries/
│       │   ├── GetUserById/
│       │   │   ├── GetUserByIdQuery.cs
│       │   │   └── GetUserByIdQueryHandler.cs
│       │   └── GetAllUsers/
│       │       ├── GetAllUsersQuery.cs
│       │       └── GetAllUsersQueryHandler.cs
│       └── DTOs/
│           └── UserDto.cs
└── DependencyInjection.cs
```

### `{Project}.Infrastructure`

```
{Project}.Infrastructure/
├── Persistence/
│   ├── ApplicationDbContext.cs
│   ├── Configurations/
│   │   └── UserConfiguration.cs
│   └── Migrations/
├── Repositories/
│   ├── BaseRepository.cs
│   └── UserRepository.cs
├── Identity/
│   ├── ApplicationUser.cs
│   └── TokenService.cs
├── Services/
│   └── DateTimeService.cs
└── DependencyInjection.cs
```

### `{Project}.Extensions`

```
{Project}.Extensions/
├── Attributes/
│   └── IgnoreRegistrationAttribute.cs
├── Interfaces/
│   ├── ITransientService.cs
│   ├── IScopedService.cs
│   └── ISingletonService.cs
├── ServiceRegistrar.cs
├── RepositoryRegistrar.cs
└── DependencyInjection.cs
```

### `{Project}.Api`

```
{Project}.Api/
├── Controllers/
│   ├── BaseApiController.cs
│   ├── AuthController.cs
│   └── UsersController.cs
├── Middleware/
│   ├── ExceptionHandlingMiddleware.cs
│   └── TenantMiddleware.cs  ← only if multi-tenant
├── Filters/
│   └── ApiResponseFilter.cs
├── appsettings.json
├── appsettings.Development.json
└── Program.cs
```

---

## 7. Domain Layer — Implementation Details

### BaseEntity

```csharp
// Domain/Entities/BaseEntity.cs
namespace {Project}.Domain.Entities;

public abstract class BaseEntity
{
    public Guid Id { get; protected set; } = Guid.NewGuid();
    public DateTime CreatedAt { get; protected set; } = DateTime.UtcNow;
    public DateTime? UpdatedAt { get; protected set; }
    public bool IsDeleted { get; protected set; } = false;

    public void MarkAsDeleted() => IsDeleted = true;
    public void SetUpdated() => UpdatedAt = DateTime.UtcNow;
}
```

### Domain Entity Example

```csharp
// Domain/Entities/User.cs
namespace {Project}.Domain.Entities;

public sealed class User : BaseEntity
{
    public string FirstName { get; private set; } = string.Empty;
    public string LastName { get; private set; } = string.Empty;
    public string Email { get; private set; } = string.Empty;
    public UserStatus Status { get; private set; } = UserStatus.Active;

    private User() { } // EF Core

    public static User Create(string firstName, string lastName, string email)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(firstName);
        ArgumentException.ThrowIfNullOrWhiteSpace(lastName);
        ArgumentException.ThrowIfNullOrWhiteSpace(email);

        return new User
        {
            FirstName = firstName,
            LastName = lastName,
            Email = email.ToLowerInvariant()
        };
    }

    public void Update(string firstName, string lastName)
    {
        FirstName = firstName;
        LastName = lastName;
        SetUpdated();
    }
}
```

### Domain Exception

```csharp
// Domain/Exceptions/UserNotFoundException.cs
namespace {Project}.Domain.Exceptions;

public sealed class UserNotFoundException : DomainException
{
    public UserNotFoundException(Guid userId)
        : base($"User with ID '{userId}' was not found.") { }
}

public abstract class DomainException : Exception
{
    protected DomainException(string message) : base(message) { }
}
```

---

## 8. Application Layer — Implementation Details

### IApplicationDbContext Interface

```csharp
// Application/Common/Interfaces/IApplicationDbContext.cs
namespace {Project}.Application.Common.Interfaces;

public interface IApplicationDbContext
{
    DbSet<User> Users { get; }
    Task<int> SaveChangesAsync(CancellationToken cancellationToken);
}
```

### MediatR CQRS Pattern

```csharp
// Application/Features/Users/Commands/CreateUser/CreateUserCommand.cs
namespace {Project}.Application.Features.Users.Commands.CreateUser;

public sealed record CreateUserCommand(
    string FirstName,
    string LastName,
    string Email,
    string Password) : IRequest<UserDto>;
```

```csharp
// Application/Features/Users/Commands/CreateUser/CreateUserCommandHandler.cs
namespace {Project}.Application.Features.Users.Commands.CreateUser;

public sealed class CreateUserCommandHandler : IRequestHandler<CreateUserCommand, UserDto>
{
    private readonly IApplicationDbContext _context;

    public CreateUserCommandHandler(IApplicationDbContext context)
        => _context = context;

    public async Task<UserDto> Handle(CreateUserCommand request, CancellationToken cancellationToken)
    {
        var emailExists = await _context.Users
            .AnyAsync(u => u.Email == request.Email.ToLowerInvariant(), cancellationToken);

        if (emailExists)
            throw new InvalidOperationException($"Email '{request.Email}' is already registered.");

        var user = User.Create(request.FirstName, request.LastName, request.Email);
        _context.Users.Add(user);
        await _context.SaveChangesAsync(cancellationToken);

        return new UserDto(user.Id, user.FirstName, user.LastName, user.Email);
    }
}
```

```csharp
// Application/Features/Users/Commands/CreateUser/CreateUserCommandValidator.cs
namespace {Project}.Application.Features.Users.Commands.CreateUser;

public sealed class CreateUserCommandValidator : AbstractValidator<CreateUserCommand>
{
    public CreateUserCommandValidator()
    {
        RuleFor(x => x.FirstName).NotEmpty().MaximumLength(100);
        RuleFor(x => x.LastName).NotEmpty().MaximumLength(100);
        RuleFor(x => x.Email).NotEmpty().EmailAddress();
        RuleFor(x => x.Password).NotEmpty().MinimumLength(8);
    }
}
```

### MediatR Pipeline Behaviours

```csharp
// Application/Common/Behaviours/ValidationBehaviour.cs
namespace {Project}.Application.Common.Behaviours;

public sealed class ValidationBehaviour<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehaviour(IEnumerable<IValidator<TRequest>> validators)
        => _validators = validators;

    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        if (!_validators.Any()) return await next();

        var context = new ValidationContext<TRequest>(request);
        var failures = _validators
            .Select(v => v.Validate(context))
            .SelectMany(r => r.Errors)
            .Where(e => e is not null)
            .ToList();

        if (failures.Count != 0)
            throw new ValidationException(failures);

        return await next();
    }
}
```

### Application DependencyInjection

```csharp
// Application/DependencyInjection.cs
namespace {Project}.Application;

public static class DependencyInjection
{
    public static IServiceCollection AddApplication(this IServiceCollection services)
    {
        services.AddMediatR(cfg =>
        {
            cfg.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly());
            cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(ValidationBehaviour<,>));
            cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(LoggingBehaviour<,>));
        });

        services.AddValidatorsFromAssembly(Assembly.GetExecutingAssembly());
        services.AddFluentValidationAutoValidation();

        return services;
    }
}
```

---

## 9. Infrastructure Layer — Implementation Details

### ApplicationDbContext

```csharp
// Infrastructure/Persistence/ApplicationDbContext.cs
namespace {Project}.Infrastructure.Persistence;

public sealed class ApplicationDbContext : DbContext, IApplicationDbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options) : base(options) { }

    public DbSet<User> Users => Set<User>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(Assembly.GetExecutingAssembly());
        base.OnModelCreating(modelBuilder);
    }

    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        foreach (var entry in ChangeTracker.Entries<BaseEntity>())
        {
            if (entry.State == EntityState.Modified)
                entry.Entity.SetUpdated();
        }
        return await base.SaveChangesAsync(cancellationToken);
    }
}
```

### Infrastructure DependencyInjection

```csharp
// Infrastructure/DependencyInjection.cs
namespace {Project}.Infrastructure;

public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddDbContext<ApplicationDbContext>(options =>
            options.UseSqlServer(configuration.GetConnectionString("DefaultConnection"),
                b => b.MigrationsAssembly(typeof(ApplicationDbContext).Assembly.FullName)));

        services.AddScoped<IApplicationDbContext>(provider =>
            provider.GetRequiredService<ApplicationDbContext>());

        return services;
    }
}
```

---

## 10. Extensions Layer — Implementation Details

The Extensions project uses **reflection** to scan the Infrastructure assembly at startup and register services and repositories automatically — eliminating the need to manually wire each one in `DependencyInjection.cs`.

### Marker Interfaces

```csharp
// Extensions/Interfaces/ITransientService.cs
namespace {Project}.Extensions.Interfaces;

/// <summary>Marker — implementing classes are registered as Transient.</summary>
public interface ITransientService { }
```

```csharp
// Extensions/Interfaces/IScopedService.cs
namespace {Project}.Extensions.Interfaces;

/// <summary>Marker — implementing classes are registered as Scoped.</summary>
public interface IScopedService { }
```

```csharp
// Extensions/Interfaces/ISingletonService.cs
namespace {Project}.Extensions.Interfaces;

/// <summary>Marker — implementing classes are registered as Singleton.</summary>
public interface ISingletonService { }
```

### IgnoreRegistrationAttribute

```csharp
// Extensions/Attributes/IgnoreRegistrationAttribute.cs
namespace {Project}.Extensions.Attributes;

/// <summary>Opt-out of automatic registration for a specific class.</summary>
[AttributeUsage(AttributeTargets.Class, Inherited = false)]
public sealed class IgnoreRegistrationAttribute : Attribute { }
```

### ServiceRegistrar

```csharp
// Extensions/ServiceRegistrar.cs
namespace {Project}.Extensions;

public static class ServiceRegistrar
{
    /// <summary>
    /// Scans <paramref name="assembliesToScan"/> for classes that implement
    /// ITransientService, IScopedService, or ISingletonService and registers
    /// them against every non-marker interface they implement.
    /// Classes decorated with [IgnoreRegistration] are skipped.
    /// </summary>
    public static IServiceCollection AddServicesFromAssemblies(
        this IServiceCollection services,
        params Assembly[] assembliesToScan)
    {
        var markerInterfaces = new[]
        {
            typeof(ITransientService),
            typeof(IScopedService),
            typeof(ISingletonService)
        };

        var candidates = assembliesToScan
            .SelectMany(a => a.GetExportedTypes())
            .Where(t => t.IsClass
                     && !t.IsAbstract
                     && !t.IsDefined(typeof(IgnoreRegistrationAttribute), inherit: false)
                     && t.GetInterfaces().Any(i => markerInterfaces.Contains(i)));

        foreach (var impl in candidates)
        {
            var lifetime = impl.IsAssignableTo(typeof(ISingletonService)) ? ServiceLifetime.Singleton
                         : impl.IsAssignableTo(typeof(ITransientService))  ? ServiceLifetime.Transient
                         : ServiceLifetime.Scoped;

            // Register against every declared interface except the marker ones
            var serviceInterfaces = impl.GetInterfaces()
                .Where(i => !markerInterfaces.Contains(i));

            foreach (var serviceType in serviceInterfaces)
                services.Add(new ServiceDescriptor(serviceType, impl, lifetime));

            // Always register the concrete type itself too
            services.Add(new ServiceDescriptor(impl, impl, lifetime));
        }

        return services;
    }
}
```

### RepositoryRegistrar

```csharp
// Extensions/RepositoryRegistrar.cs
namespace {Project}.Extensions;

public static class RepositoryRegistrar
{
    /// <summary>
    /// Scans <paramref name="implementationAssembly"/> for classes whose name ends with
    /// "Repository" and automatically pairs them with the matching I{Name}Repository
    /// interface found in <paramref name="interfaceAssembly"/>.
    /// All repositories are registered as Scoped.
    /// </summary>
    public static IServiceCollection AddRepositoriesFromAssembly(
        this IServiceCollection services,
        Assembly implementationAssembly,
        Assembly interfaceAssembly)
    {
        var implementations = implementationAssembly
            .GetExportedTypes()
            .Where(t => t.IsClass
                     && !t.IsAbstract
                     && t.Name.EndsWith("Repository", StringComparison.Ordinal)
                     && !t.IsDefined(typeof(IgnoreRegistrationAttribute), inherit: false));

        var interfaceMap = interfaceAssembly
            .GetExportedTypes()
            .Where(t => t.IsInterface && t.Name.StartsWith('I') && t.Name.EndsWith("Repository"))
            .ToDictionary(t => t.Name); // "IUserRepository" → Type

        foreach (var impl in implementations)
        {
            var expectedInterface = $"I{impl.Name}"; // "UserRepository" → "IUserRepository"
            if (interfaceMap.TryGetValue(expectedInterface, out var serviceType))
                services.AddScoped(serviceType, impl);
            else
                services.AddScoped(impl); // no matching interface — register concrete only
        }

        return services;
    }
}
```

### Extensions DependencyInjection

```csharp
// Extensions/DependencyInjection.cs
namespace {Project}.Extensions;

public static class DependencyInjection
{
    /// <summary>
    /// Dynamically registers all services (via marker interfaces) and all repositories
    /// (via naming convention) found in the Infrastructure assembly.
    /// </summary>
    public static IServiceCollection AddExtensions(this IServiceCollection services)
    {
        var infrastructureAssembly = typeof({Project}.Infrastructure.AssemblyReference).Assembly;
        var applicationAssembly    = typeof({Project}.Application.AssemblyReference).Assembly;

        services.AddServicesFromAssemblies(infrastructureAssembly);
        services.AddRepositoriesFromAssembly(infrastructureAssembly, applicationAssembly);

        return services;
    }
}
```

> **Note:** Add a lightweight `AssemblyReference.cs` marker class to both `{Project}.Infrastructure` and `{Project}.Application` if they don't already expose a stable type for assembly discovery:
>
> ```csharp
> // Infrastructure/AssemblyReference.cs
> namespace {Project}.Infrastructure;
> public static class AssemblyReference { }
> ```
>
> ```csharp
> // Application/AssemblyReference.cs
> namespace {Project}.Application;
> public static class AssemblyReference { }
> ```

### Usage on Services and Repositories

Any Infrastructure service or repository simply implements the appropriate marker interface:

```csharp
// Infrastructure/Services/DateTimeService.cs
public sealed class DateTimeService : IDateTimeService, IScopedService
{
    public DateTime UtcNow => DateTime.UtcNow;
}
```

```csharp
// Infrastructure/Repositories/UserRepository.cs
// Automatically paired with IUserRepository from Application layer
public sealed class UserRepository : BaseRepository<User>, IUserRepository, IScopedService
{
    public UserRepository(ApplicationDbContext context) : base(context) { }
    // ...
}
```

> Repositories that implement `IScopedService` are picked up by **both** `ServiceRegistrar` and `RepositoryRegistrar`. Use `[IgnoreRegistration]` on the class if you want only one of the two registrars to handle it.

---

## 11. API Layer — Implementation Details

### Program.cs

```csharp
using {Project}.Application;
using {Project}.Infrastructure;
using Serilog;
using SharpGrip.FluentValidation.AutoValidation.Mvc.Extensions;

var builder = WebApplication.CreateBuilder(args);

builder.Host.UseSerilog((ctx, lc) => lc
    .ReadFrom.Configuration(ctx.Configuration)
    .Enrich.FromLogContext()
    .WriteTo.Console()
    .WriteTo.File("logs/log-.txt", rollingInterval: RollingInterval.Day));

builder.Services.AddApplication();
builder.Services.AddInfrastructure(builder.Configuration);
builder.Services.AddExtensions();   // dynamic service & repository registration

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new OpenApiInfo { Title = "{Project} API", Version = "v1" });
    c.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Name = "Authorization",
        Type = SecuritySchemeType.Http,
        Scheme = "Bearer",
        BearerFormat = "JWT"
    });
    c.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference { Type = ReferenceType.SecurityScheme, Id = "Bearer" }
            },
            Array.Empty<string>()
        }
    });
});

// JWT Auth (adjust based on auth selection)
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        var jwtSettings = builder.Configuration.GetSection("JwtSettings");
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = jwtSettings["Issuer"],
            ValidAudience = jwtSettings["Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(jwtSettings["Secret"]!))
        };
    });

builder.Services.AddAuthorization();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseSerilogRequestLogging();
app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();
app.UseMiddleware<ExceptionHandlingMiddleware>();
app.MapControllers();

app.Run();

public partial class Program { } // Required for WebApplicationFactory in integration tests
```

### BaseApiController

```csharp
// Controllers/BaseApiController.cs
namespace {Project}.Api.Controllers;

[ApiController]
[Route("api/[controller]")]
public abstract class BaseApiController : ControllerBase
{
    private ISender? _mediator;
    protected ISender Mediator => _mediator ??=
        HttpContext.RequestServices.GetRequiredService<ISender>();
}
```

### UsersController

```csharp
// Controllers/UsersController.cs
namespace {Project}.Api.Controllers;

public sealed class UsersController : BaseApiController
{
    [HttpGet("{id:guid}")]
    [Authorize]
    [ProducesResponseType(typeof(ApiResponse<UserDto>), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> GetById(Guid id, CancellationToken cancellationToken)
    {
        var result = await Mediator.Send(new GetUserByIdQuery(id), cancellationToken);
        return Ok(ApiResponse<UserDto>.Ok(result));
    }

    [HttpPost]
    [ProducesResponseType(typeof(ApiResponse<UserDto>), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> Create(
        [FromBody] CreateUserCommand command,
        CancellationToken cancellationToken)
    {
        var result = await Mediator.Send(command, cancellationToken);
        return CreatedAtAction(nameof(GetById), new { id = result.Id }, ApiResponse<UserDto>.Ok(result));
    }
}
```

### Global Exception Handling Middleware

```csharp
// Middleware/ExceptionHandlingMiddleware.cs
namespace {Project}.Api.Middleware;

public sealed class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;

    public ExceptionHandlingMiddleware(RequestDelegate next, ILogger<ExceptionHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (ValidationException ex)
        {
            _logger.LogWarning("Validation error: {Errors}", ex.Errors);
            context.Response.StatusCode = StatusCodes.Status400BadRequest;
            context.Response.ContentType = "application/json";
            var errors = ex.Errors.Select(e => e.ErrorMessage);
            await context.Response.WriteAsJsonAsync(
                ApiResponse<object>.Fail("Validation failed.", errors));
        }
        catch (DomainException ex)
        {
            _logger.LogWarning("Domain error: {Message}", ex.Message);
            context.Response.StatusCode = StatusCodes.Status400BadRequest;
            context.Response.ContentType = "application/json";
            await context.Response.WriteAsJsonAsync(
                ApiResponse<object>.Fail(ex.Message));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception");
            context.Response.StatusCode = StatusCodes.Status500InternalServerError;
            context.Response.ContentType = "application/json";
            await context.Response.WriteAsJsonAsync(
                ApiResponse<object>.Fail("An unexpected error occurred."));
        }
    }
}
```

---

## 12. Authentication (ASP.NET Core Identity + JWT)

When Option 1 (Identity + JWT) is selected:

1. Add `IdentityDbContext` to `ApplicationDbContext` by inheriting from `IdentityDbContext<ApplicationUser>`.
2. Create `ApplicationUser : IdentityUser` in `Infrastructure/Identity/`.
3. Implement `ITokenService` in `Application/Common/Interfaces/`.
4. Implement `TokenService` in `Infrastructure/Identity/`.
5. Add `AuthController` in `Api/Controllers/`.

```csharp
// Infrastructure/Identity/ApplicationUser.cs
public sealed class ApplicationUser : IdentityUser
{
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public string? RefreshToken { get; set; }
    public DateTime? RefreshTokenExpiry { get; set; }
}
```

```csharp
// Infrastructure/Identity/TokenService.cs
public sealed class TokenService : ITokenService
{
    private readonly JwtSettings _settings;

    public TokenService(IOptions<JwtSettings> settings) => _settings = settings.Value;

    public string GenerateAccessToken(ApplicationUser user, IList<string> roles)
    {
        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub, user.Id),
            new(JwtRegisteredClaimNames.Email, user.Email!),
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
        };
        claims.AddRange(roles.Select(r => new Claim(ClaimTypes.Role, r)));

        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_settings.Secret));
        var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
        var token = new JwtSecurityToken(
            issuer: _settings.Issuer,
            audience: _settings.Audience,
            claims: claims,
            expires: DateTime.UtcNow.AddMinutes(_settings.ExpiryMinutes),
            signingCredentials: creds);

        return new JwtSecurityTokenHandler().WriteToken(token);
    }

    public string GenerateRefreshToken() =>
        Convert.ToBase64String(RandomNumberGenerator.GetBytes(64));
}
```

---

## 13. NuGet Packages Per Project

### `{Project}.Api`

```bash
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package Microsoft.AspNetCore.OpenApi
dotnet add package Swashbuckle.AspNetCore
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.File
dotnet add package SharpGrip.FluentValidation.AutoValidation.Mvc
```

### `{Project}.Application`

```bash
dotnet add package MediatR
dotnet add package FluentValidation
dotnet add package FluentValidation.AspNetCore
dotnet add package Microsoft.Extensions.Logging.Abstractions
```

### `{Project}.Infrastructure`

```bash
dotnet add package Microsoft.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore.SqlServer         # or Npgsql.EntityFrameworkCore.PostgreSQL
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore
dotnet add package Microsoft.Extensions.Configuration.Abstractions
```

### `{Project}.Domain`

```bash
# No external NuGet packages — Domain must remain dependency-free
```

### `{Project}.Extensions`

```bash
dotnet add package Microsoft.Extensions.DependencyInjection.Abstractions
```

---

## 14. CLI Commands to Scaffold the Solution

```bash
# Create solution
dotnet new sln -n {Project}

# Create projects
dotnet new webapi  -o src/{Project}.Api           --framework net10.0
dotnet new classlib -o src/{Project}.Application   --framework net10.0
dotnet new classlib -o src/{Project}.Domain        --framework net10.0
dotnet new classlib -o src/{Project}.Extensions    --framework net10.0
dotnet new classlib -o src/{Project}.Infrastructure --framework net10.0

# Add projects to solution
dotnet sln add src/{Project}.Api/{Project}.Api.csproj
dotnet sln add src/{Project}.Application/{Project}.Application.csproj
dotnet sln add src/{Project}.Domain/{Project}.Domain.csproj
dotnet sln add src/{Project}.Extensions/{Project}.Extensions.csproj
dotnet sln add src/{Project}.Infrastructure/{Project}.Infrastructure.csproj

# Add project references
dotnet add src/{Project}.Api reference src/{Project}.Application
dotnet add src/{Project}.Api reference src/{Project}.Infrastructure
dotnet add src/{Project}.Api reference src/{Project}.Extensions
dotnet add src/{Project}.Application reference src/{Project}.Domain
dotnet add src/{Project}.Infrastructure reference src/{Project}.Application
dotnet add src/{Project}.Infrastructure reference src/{Project}.Domain
dotnet add src/{Project}.Extensions reference src/{Project}.Application
dotnet add src/{Project}.Extensions reference src/{Project}.Infrastructure

# Test projects (if selected)
dotnet new xunit -o tests/{Project}.Tests.Unit --framework net10.0
dotnet new xunit -o tests/{Project}.Tests.Integration --framework net10.0
dotnet sln add tests/{Project}.Tests.Unit/{Project}.Tests.Unit.csproj
dotnet sln add tests/{Project}.Tests.Integration/{Project}.Tests.Integration.csproj
dotnet add tests/{Project}.Tests.Unit reference src/{Project}.Application
dotnet add tests/{Project}.Tests.Unit reference src/{Project}.Domain
dotnet add tests/{Project}.Tests.Integration reference src/{Project}.Api
dotnet add tests/{Project}.Tests.Integration reference src/{Project}.Infrastructure
```

---

## 15. Multi-Tenant Support (If Selected)

When multi-tenant is enabled:
1. Add `ITenantService` to `Application/Common/Interfaces/`.
2. Implement `TenantService` resolving tenant from HTTP header `X-Tenant-Id`.
3. Add `TenantId` property to `BaseEntity`.
4. Apply a global query filter in `ApplicationDbContext.OnModelCreating`:

```csharp
modelBuilder.Entity<User>()
    .HasQueryFilter(u => u.TenantId == _tenantService.GetTenantId() && !u.IsDeleted);
```

5. Register `TenantMiddleware` in `Program.cs` before `UseAuthentication`.

---

## 16. Unit Testing Rules for Clean Architecture

- Test **Command Handlers** and **Query Handlers** in isolation.
- Mock `IApplicationDbContext` using Moq — never use a real DB in unit tests.
- Use `AutoFixture` to generate entities and commands.
- Test **Validators** separately — instantiate validator and call `.Validate()`.
- Test **Domain Entity** methods directly — no mocking needed for pure domain.

**Example Handler Unit Test:**

```csharp
public class CreateUserCommandHandlerTests
{
    private readonly Mock<IApplicationDbContext> _contextMock = new();
    private readonly CreateUserCommandHandler _sut;
    private readonly IFixture _fixture = new Fixture();

    public CreateUserCommandHandlerTests()
    {
        var usersDbSet = CreateMockDbSet(new List<User>());
        _contextMock.Setup(c => c.Users).Returns(usersDbSet);
        _contextMock.Setup(c => c.SaveChangesAsync(It.IsAny<CancellationToken>())).ReturnsAsync(1);
        _sut = new CreateUserCommandHandler(_contextMock.Object);
    }

    [Fact]
    public async Task Handle_WithValidCommand_AddsUserAndReturnsDto()
    {
        var command = new CreateUserCommand("John", "Doe", "john@example.com", "Password1!");

        var result = await _sut.Handle(command, CancellationToken.None);

        result.Should().NotBeNull();
        result.Email.Should().Be("john@example.com");
        _contextMock.Verify(c => c.SaveChangesAsync(It.IsAny<CancellationToken>()), Times.Once);
    }
}
```

---

## 17. Integration Testing Rules for Clean Architecture

- Use `WebApplicationFactory<Program>` with InMemory EF Core DB.
- Replace `IApplicationDbContext` with real `ApplicationDbContext` backed by InMemory database.
- Seed test data using the `ApplicationDbContext` directly in the factory.
- Test full request cycle: HTTP → Controller → MediatR → Handler → DB.

---

## 18. Cloud Provider Services (Conditional — apply only if cloud provider selected)

> Generate this section only when the user selected **Azure** or **AWS** in Question 1.8 of `scaffold.md`.

### 18.1 — Folder Structure

Add a `CloudServices/` folder inside `{Project}.Infrastructure`:

```
{Project}.Infrastructure/
├── CloudServices/
│   ├── Storage/
│   │   ├── AzureBlobStorageService.cs    ← Azure only
│   │   └── AwsS3StorageService.cs        ← AWS only
│   └── Secrets/
│       ├── AzureKeyVaultSecretService.cs ← Azure only
│       └── AwsSecretsManagerSecretService.cs ← AWS only
```

Add cloud service interfaces to `{Project}.Application`:

```
{Project}.Application/
├── Common/
│   └── Interfaces/
│       ├── IStorageService.cs    ← if storage selected
│       └── ISecretService.cs     ← if Key Vault / Secrets Manager selected
```

### 18.2 — Interface Registration (Infrastructure `DependencyInjection.cs`)

```csharp
// {Project}.Infrastructure/DependencyInjection.cs (add inside AddInfrastructure)

// Azure Blob Storage (if selected)
services.AddSingleton(sp =>
{
    var accountName = configuration["AzureBlobStorage:AccountName"]!;
    var credential = sp.GetRequiredService<Azure.Core.TokenCredential>();
    return new BlobServiceClient(new Uri($"https://{accountName}.blob.core.windows.net"), credential);
});
services.AddScoped<IStorageService, AzureBlobStorageService>();

// Azure Key Vault (if selected)
services.AddSingleton(sp =>
{
    var vaultUri = new Uri(configuration["Azure:KeyVaultUri"]!);
    var credential = sp.GetRequiredService<Azure.Core.TokenCredential>();
    return new SecretClient(vaultUri, credential);
});
services.AddScoped<ISecretService, AzureKeyVaultSecretService>();

// AWS S3 (if selected — replace Azure registrations above with these)
services.AddAWSService<IAmazonS3>();
services.AddScoped<IStorageService, AwsS3StorageService>();

// AWS Secrets Manager (if selected)
services.AddAWSService<IAmazonSecretsManager>();
services.AddScoped<ISecretService, AwsSecretsManagerSecretService>();
```

### 18.3 — Usage in Application Handlers

Inject `IStorageService` or `ISecretService` directly into MediatR command/query handlers via constructor injection. The Application layer depends only on the interface (defined in `{Project}.Application/Common/Interfaces/`), never on the concrete Azure/AWS implementation.

```csharp
// Example: UploadProfilePictureCommandHandler.cs
public sealed class UploadProfilePictureCommandHandler
    : IRequestHandler<UploadProfilePictureCommand, string>
{
    private readonly IStorageService _storageService;

    public UploadProfilePictureCommandHandler(IStorageService storageService)
        => _storageService = storageService;

    public async Task<string> Handle(UploadProfilePictureCommand request, CancellationToken cancellationToken)
    {
        var blobName = $"profiles/{request.UserId}/{Guid.NewGuid()}{Path.GetExtension(request.FileName)}";
        return await _storageService.UploadAsync("user-media", blobName, request.FileStream,
            request.ContentType, cancellationToken);
    }
}
```

---

## 19. Third-Party HTTP Client Integration with Polly (Conditional)

> Generate this section only when the user selected **Yes** in Question 1.9 of `scaffold.md`.

### 19.1 — Folder Structure

```
{Project}.Application/
├── Common/
│   └── Interfaces/
│       └── HttpClients/
│           └── I{ServiceName}Client.cs    ← one interface per third-party service

{Project}.Infrastructure/
├── HttpClients/
│   ├── {ServiceName}Client.cs             ← typed HttpClient implementation
│   └── HttpClientServiceExtensions.cs    ← DI registration with Polly
```

### 19.2 — Full Registration Example

```csharp
// {Project}.Infrastructure/HttpClients/HttpClientServiceExtensions.cs
public static class HttpClientServiceExtensions
{
    public static IServiceCollection AddHttpClients(
        this IServiceCollection services, IConfiguration configuration)
    {
        services.AddResiliencePipeline("default-http-pipeline", pipelineBuilder =>
        {
            pipelineBuilder.AddRetry(new RetryStrategyOptions
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

            pipelineBuilder.AddCircuitBreaker(new CircuitBreakerStrategyOptions
            {
                FailureRatio = 0.5,
                SamplingDuration = TimeSpan.FromSeconds(30),
                MinimumThroughput = 5,
                BreakDuration = TimeSpan.FromSeconds(15),
                ShouldHandle = new PredicateBuilder().Handle<HttpRequestException>()
            });

            pipelineBuilder.AddTimeout(TimeSpan.FromSeconds(10));
        });

        // Register each typed client — repeat this block per service
        services
            .AddHttpClient<I{ServiceName}Client, {ServiceName}Client>(client =>
            {
                client.BaseAddress = new Uri(configuration["HttpClients:{ServiceName}:BaseUrl"]!);
                client.Timeout = TimeSpan.FromSeconds(
                    configuration.GetValue<int>("HttpClients:{ServiceName}:TimeoutSeconds", 30));
            })
            .AddResilienceHandler("default-http-pipeline", (b, ctx) =>
            {
                b.AddPipeline(ctx.ServiceProvider
                    .GetRequiredService<ResiliencePipelineProvider<string>>()
                    .GetPipeline("default-http-pipeline"));
            });

        return services;
    }
}
```

Call `services.AddHttpClients(configuration)` inside `{Project}.Infrastructure/DependencyInjection.cs`.

### 19.3 — Usage in Application Handlers

Inject `I{ServiceName}Client` into MediatR handlers via constructor. The Application layer depends on the interface only.

```csharp
public sealed class ProcessPaymentCommandHandler
    : IRequestHandler<ProcessPaymentCommand, PaymentResult>
{
    private readonly IPaymentServiceClient _paymentClient;

    public ProcessPaymentCommandHandler(IPaymentServiceClient paymentClient)
        => _paymentClient = paymentClient;

    public async Task<PaymentResult> Handle(ProcessPaymentCommand request, CancellationToken cancellationToken)
        => await _paymentClient.ProcessPaymentAsync(request.ToPaymentRequest(), cancellationToken);
}
```
