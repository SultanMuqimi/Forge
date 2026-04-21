# CLAUDE.md — ASP.NET Core Backend (Clean Architecture)
### Forge Stack Rules | ASP.NET Core for Enterprise Applications

---

This file extends the universal CLAUDE.md. Read that first.

---

## WHEN TO USE THIS STACK

- **Enterprise applications** with complex business logic
- **Government** or regulated industries requiring Microsoft stack
- Teams with existing .NET expertise
- High-security contexts (Microsoft's compliance story is strong)
- Integration with Microsoft ecosystem (Azure AD, SQL Server, Power BI)

Not the default for small apps or non-developers — too much ceremony.

---

## TECHNOLOGY STANDARDS

### Required
- **.NET 8+ LTS** (or .NET 9 if released)
- **ASP.NET Core Web API**
- **C#** with nullable reference types enabled
- **Entity Framework Core** or **Dapper** (EF Core default, Dapper for performance-critical)
- **PostgreSQL** (default) or **SQL Server** (enterprise preference)

### Required Libraries
- **FluentValidation** for request validation
- **MediatR** for CQRS/handler pattern
- **Serilog** for structured logging
- **AutoMapper** or **Mapster** for DTO mapping
- **Polly** for resilience (retries, circuit breakers)

---

## FOLDER STRUCTURE — CLEAN ARCHITECTURE

```
solution-root/
├── CLAUDE.md
├── .env.example
├── YourApp.sln
├── src/
│   ├── YourApp.API/                       (presentation layer)
│   │   ├── Controllers/
│   │   ├── Middleware/
│   │   ├── Filters/
│   │   ├── Program.cs
│   │   └── appsettings.json
│   ├── YourApp.Core/                      (domain layer - no dependencies)
│   │   ├── Entities/
│   │   ├── ValueObjects/
│   │   ├── Enums/
│   │   ├── Exceptions/
│   │   └── Interfaces/                    (repository interfaces)
│   ├── YourApp.Application/               (business logic)
│   │   ├── Features/                      (organized by feature)
│   │   │   └── Users/
│   │   │       ├── Commands/
│   │   │       │   ├── CreateUser/
│   │   │       │   │   ├── CreateUserCommand.cs
│   │   │       │   │   ├── CreateUserHandler.cs
│   │   │       │   │   └── CreateUserValidator.cs
│   │   │       └── Queries/
│   │   │           └── GetUser/
│   │   ├── Common/
│   │   │   ├── Behaviors/                 (MediatR pipeline behaviors)
│   │   │   └── Mappings/
│   │   └── Interfaces/                    (application-level interfaces)
│   └── YourApp.Infrastructure/            (data access, external services)
│       ├── Persistence/
│       │   ├── AppDbContext.cs
│       │   ├── Configurations/            (EF entity configurations)
│       │   ├── Repositories/
│       │   └── Migrations/
│       ├── Services/                      (email, storage, etc.)
│       └── DependencyInjection.cs
└── tests/
    ├── YourApp.UnitTests/
    ├── YourApp.IntegrationTests/
    └── YourApp.ArchitectureTests/
```

### Dependency Rule
- **Core** depends on nothing
- **Application** depends on Core
- **Infrastructure** depends on Core and Application
- **API** depends on all of them (entry point)
- **Never** reverse this direction

---

## CQRS PATTERN WITH MEDIATR

### Command Example
```csharp
// CreateUserCommand.cs
public record CreateUserCommand(string Email, string Name, string Password) 
    : IRequest<UserDto>;

// CreateUserValidator.cs
public class CreateUserValidator : AbstractValidator<CreateUserCommand>
{
    public CreateUserValidator()
    {
        RuleFor(x => x.Email).NotEmpty().EmailAddress();
        RuleFor(x => x.Name).NotEmpty().MaximumLength(100);
        RuleFor(x => x.Password).NotEmpty().MinimumLength(8);
    }
}

// CreateUserHandler.cs
public class CreateUserHandler : IRequestHandler<CreateUserCommand, UserDto>
{
    private readonly IUserRepository _repository;
    private readonly IPasswordHasher _hasher;
    
    public CreateUserHandler(IUserRepository repository, IPasswordHasher hasher)
    {
        _repository = repository;
        _hasher = hasher;
    }
    
    public async Task<UserDto> Handle(CreateUserCommand request, CancellationToken ct)
    {
        var existing = await _repository.FindByEmailAsync(request.Email, ct);
        if (existing is not null)
            throw new ConflictException("Email already registered");
        
        var user = new User
        {
            Email = request.Email,
            Name = request.Name,
            PasswordHash = _hasher.Hash(request.Password),
        };
        
        await _repository.AddAsync(user, ct);
        return user.ToDto();
    }
}
```

### Controller (Thin)
```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController(ISender mediator) : ControllerBase
{
    [HttpPost]
    public async Task<ActionResult<UserDto>> Create(
        CreateUserCommand command, 
        CancellationToken ct)
    {
        var result = await mediator.Send(command, ct);
        return CreatedAtAction(nameof(Get), new { id = result.Id }, result);
    }
}
```

**Controllers stay thin.** They only translate HTTP to commands/queries. Never put business logic in controllers.

---

## EXCEPTION HANDLING — GLOBAL MIDDLEWARE

**Never put try-catch in controllers.** Use `ExceptionHandlingMiddleware`:

```csharp
public class ExceptionHandlingMiddleware
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
        catch (Exception ex)
        {
            await HandleExceptionAsync(context, ex);
        }
    }
    
    private async Task HandleExceptionAsync(HttpContext context, Exception ex)
    {
        var (statusCode, message) = ex switch
        {
            ValidationException validationEx => (400, validationEx.Message),
            NotFoundException => (404, ex.Message),
            ConflictException => (409, ex.Message),
            UnauthorizedException => (401, ex.Message),
            _ => (500, "An internal error occurred"),
        };
        
        if (statusCode == 500)
            _logger.LogError(ex, "Unhandled exception");
        
        context.Response.StatusCode = statusCode;
        context.Response.ContentType = "application/json";
        await context.Response.WriteAsJsonAsync(new { error = message });
    }
}
```

Register in `Program.cs`:
```csharp
app.UseMiddleware<ExceptionHandlingMiddleware>();
```

---

## AUTHENTICATION

### JWT Setup
```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Secret"]!)),
        };
    });
```

### Role-Based Authorization
```csharp
[Authorize(Roles = "Admin")]
[HttpDelete("{id}")]
public async Task<IActionResult> Delete(Guid id, CancellationToken ct) { /* ... */ }
```

### Password Hashing
Use ASP.NET Core Identity's `PasswordHasher<T>` or BCrypt.Net-Next:
```csharp
public class PasswordHasher : IPasswordHasher
{
    public string Hash(string password) => BCrypt.Net.BCrypt.HashPassword(password, 12);
    public bool Verify(string password, string hash) => BCrypt.Net.BCrypt.Verify(password, hash);
}
```

---

## ENTITY FRAMEWORK CORE

### DbContext
```csharp
public class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    public DbSet<User> Users => Set<User>();
    
    protected override void OnModelCreating(ModelBuilder builder)
    {
        builder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}
```

### Entity Configuration (Fluent API)
```csharp
public class UserConfiguration : IEntityTypeConfiguration<User>
{
    public void Configure(EntityTypeBuilder<User> builder)
    {
        builder.HasKey(u => u.Id);
        builder.Property(u => u.Email).IsRequired().HasMaxLength(255);
        builder.HasIndex(u => u.Email).IsUnique();
        builder.Property(u => u.Name).IsRequired().HasMaxLength(100);
    }
}
```

### Migrations
- Create: `dotnet ef migrations add <Name> -p YourApp.Infrastructure -s YourApp.API`
- Apply: `dotnet ef database update -p YourApp.Infrastructure -s YourApp.API`
- Never edit applied migrations — create new ones

### Query Rules
- Use `AsNoTracking()` for read-only queries (performance)
- Use `Include()` explicitly — no lazy loading
- Use projections (`.Select()`) to avoid loading unused columns
- Use `CancellationToken` on every async query

---

## VALIDATION

### FluentValidation + MediatR Pipeline
Create a validation behavior that runs automatically before every handler:

```csharp
public class ValidationBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;
    
    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
    {
        _validators = validators;
    }
    
    public async Task<TResponse> Handle(
        TRequest request, 
        RequestHandlerDelegate<TResponse> next, 
        CancellationToken ct)
    {
        if (!_validators.Any()) return await next();
        
        var context = new ValidationContext<TRequest>(request);
        var failures = _validators
            .Select(v => v.Validate(context))
            .SelectMany(r => r.Errors)
            .Where(f => f != null)
            .ToList();
        
        if (failures.Any())
            throw new ValidationException(failures);
        
        return await next();
    }
}
```

---

## LOGGING — SERILOG

```csharp
// Program.cs
builder.Host.UseSerilog((context, configuration) =>
{
    configuration
        .ReadFrom.Configuration(context.Configuration)
        .Enrich.FromLogContext()
        .WriteTo.Console()
        .WriteTo.File("logs/app-.log", rollingInterval: RollingInterval.Day);
});
```

- Structured logging with properties, not string concatenation
- Never log passwords, tokens, or PII
- Use `LogInformation`, `LogWarning`, `LogError` appropriately

---

## CONFIGURATION

### `appsettings.json` vs Environment Variables
- Non-sensitive config in `appsettings.json`
- Sensitive values (secrets, connection strings with passwords) in environment variables or user secrets
- Override per environment with `appsettings.Development.json`, `appsettings.Production.json`

### Strongly-Typed Config
```csharp
public class JwtOptions
{
    public string Secret { get; set; } = null!;
    public string Issuer { get; set; } = null!;
    public string Audience { get; set; } = null!;
    public int AccessTokenMinutes { get; set; } = 15;
}

// Program.cs
builder.Services.Configure<JwtOptions>(builder.Configuration.GetSection("Jwt"));

// Usage
public class TokenService(IOptions<JwtOptions> options)
{
    private readonly JwtOptions _options = options.Value;
}
```

---

## TESTING

### Stack
- **xUnit** or **NUnit** (xUnit preferred)
- **FluentAssertions** for readable assertions
- **Moq** or **NSubstitute** for mocking
- **Testcontainers** for integration tests with real PostgreSQL
- **WebApplicationFactory** for API integration tests

### Test Projects
- `YourApp.UnitTests` — tests for handlers, services, validators
- `YourApp.IntegrationTests` — tests the full API with a test database
- `YourApp.ArchitectureTests` — enforces Clean Architecture rules with NetArchTest

---

## SECURITY CHECKLIST

- [ ] HTTPS enforced with `app.UseHttpsRedirection()`
- [ ] Security headers via `NWebsec` or custom middleware
- [ ] CORS configured with specific origins
- [ ] Rate limiting with `AspNetCoreRateLimit` or built-in rate limiter
- [ ] Password hashing with BCrypt (cost 12) or Identity
- [ ] JWT configuration validated on startup
- [ ] Authorization policies defined and applied
- [ ] Input validated via FluentValidation
- [ ] SQL injection prevented via EF Core parametrized queries
- [ ] Secrets in environment variables or Azure Key Vault
- [ ] Dependency scan (`dotnet list package --vulnerable`)

---

## ENVIRONMENT VARIABLES

```
ASPNETCORE_ENVIRONMENT=Development
ConnectionStrings__DefaultConnection=Host=localhost;Database=yourapp;Username=user;Password=pass
Jwt__Secret=<32+ bytes>
Jwt__Issuer=YourApp
Jwt__Audience=YourApp.Users
Jwt__AccessTokenMinutes=15
Cors__AllowedOrigins__0=http://localhost:5173
```

---

## SPECIFIC DO NOTS

- Never put business logic in controllers
- Never use EF Core without `AsNoTracking()` for reads
- Never disable nullable reference types
- Never use `var` when the type isn't obvious from the right side
- Never catch `Exception` and swallow it
- Never use `DateTime.Now` — use `DateTime.UtcNow` (or `TimeProvider` in .NET 8+)
- Never return entity types from controllers — return DTOs
- Never hardcode connection strings or secrets
- Never use `async void` except for event handlers
- Never forget `ConfigureAwait(false)` in library code (not needed in ASP.NET Core app code)

---

## SESSION COMPLETION CHECKLIST — ASP.NET CORE SPECIFIC

- [ ] `dotnet build` succeeds with zero warnings
- [ ] All handlers have validators
- [ ] All protected endpoints have `[Authorize]`
- [ ] Global exception middleware registered
- [ ] EF migrations are current
- [ ] Architecture tests pass
- [ ] Unit and integration tests pass
- [ ] No analyzer warnings
- [ ] Swagger/OpenAPI docs generated and accessible
- [ ] `appsettings.json` has no secrets

---

*This file is part of Forge by Sultan Al-Muqimi / NQTH LLC.*
