# .NET Interview Questions & Answers 🎯

A comprehensive, open-source guide to .NET interview questions and answers, covering **easy to advanced topics** in backend development, architecture, and application flow.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![.NET](https://img.shields.io/badge/.NET-6.0%2B-512BD4)](https://dotnet.microsoft.com/)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](http://makeapullrequest.com)

## 📋 Table of Contents

- [Easy Questions (1-7)](#-easy-questions)
- [Intermediate Questions (8-14)](#-intermediate-questions)
- [Advanced Questions (15-25)](#-advanced-questions)
- [Contributing](#-contributing)
- [License](#-license)

---

## 🟢 Easy Questions

### 1. What is .NET, and how does it differ from .NET Framework?

**.NET** (formerly .NET Core) is a cross-platform, open-source framework for building modern applications. It's the evolution of .NET Framework designed to address its limitations.

**Key Differences:**

| Feature | .NET (Modern) | .NET Framework (Legacy) |
|---------|---------------|------------------------|
| **Platform** | Cross-platform (Windows, Linux, macOS) | Windows only |
| **Open Source** | Yes (MIT license) | Partially |
| **Deployment** | Side-by-side versioning, self-contained apps | System-wide installation |
| **Performance** | Significantly faster | Slower |
| **Modularity** | NuGet packages, lightweight | Monolithic |
| **CLI** | Built-in CLI tools | Limited |
| **Future** | Active development (.NET 6, 7, 8+) | Maintenance mode |
| **Current Version** | .NET 8 (LTS) | .NET Framework 4.8.1 (final) |

---

### 2. What is the role of the Startup.cs file in an ASP.NET Core application?

The `Startup.cs` file (in older versions, now often in `Program.cs` in .NET 6+) is responsible for **configuring services** and **the application's request pipeline**.

**Two main methods:**

```csharp
public class Startup
{
    // 1. ConfigureServices: Register services into DI container
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddControllers();
        services.AddDbContext<AppDbContext>();
        services.AddScoped<IUserService, UserService>();
    }

    // 2. Configure: Build the HTTP request pipeline (middleware)
    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }
        
        app.UseRouting();
        app.UseAuthentication();
        app.UseAuthorization();
        app.UseEndpoints(endpoints =>
        {
            endpoints.MapControllers();
        });
    }
}
```

**Note:** In .NET 6+, this is simplified using the minimal hosting model in `Program.cs`.

---

### 3. What are middleware components, and how do they work in the ASP.NET Core request pipeline?

**Middleware** are software components assembled into an application pipeline to handle requests and responses. Each component:
- Can process an incoming HTTP request
- Can pass the request to the next component
- Can process the outgoing HTTP response

**Flow:**
```
Request → Middleware 1 → Middleware 2 → Middleware 3 → Endpoint
                ↓            ↓            ↓
Response ← Middleware 1 ← Middleware 2 ← Middleware 3 ←
```

**Example:**

```csharp
public void Configure(IApplicationBuilder app)
{
    // Custom middleware
    app.Use(async (context, next) =>
    {
        Console.WriteLine("Before next middleware");
        await next(); // Call next middleware
        Console.WriteLine("After next middleware");
    });

    app.UseRouting();
    app.UseAuthentication();
    app.UseAuthorization();
    
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}

// Or create custom middleware class
public class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;

    public RequestLoggingMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        Console.WriteLine($"Request: {context.Request.Path}");
        await _next(context);
        Console.WriteLine($"Response: {context.Response.StatusCode}");
    }
}

// Usage: app.UseMiddleware<RequestLoggingMiddleware>();
```

---

### 4. How do you configure dependency injection in .NET?

.NET has **built-in DI container**. Configure in `ConfigureServices` method or `Program.cs`:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    // 1. Register built-in services
    services.AddControllers();
    services.AddDbContext<ApplicationDbContext>(options =>
        options.UseSqlServer(Configuration.GetConnectionString("Default")));

    // 2. Register custom services
    // Transient: New instance every time
    services.AddTransient<IEmailService, EmailService>();
    
    // Scoped: One instance per request
    services.AddScoped<IUserRepository, UserRepository>();
    
    // Singleton: One instance for application lifetime
    services.AddSingleton<ICacheService, CacheService>();

    // 3. Register with factory
    services.AddScoped<IOrderService>(provider =>
    {
        var logger = provider.GetRequiredService<ILogger<OrderService>>();
        return new OrderService(logger);
    });

    // 4. Register multiple implementations
    services.AddTransient<INotificationService, EmailNotificationService>();
    services.AddTransient<INotificationService, SmsNotificationService>();
}

// Usage in controller
public class UsersController : ControllerBase
{
    private readonly IUserRepository _userRepository;
    
    public UsersController(IUserRepository userRepository)
    {
        _userRepository = userRepository;
    }
}
```

**In .NET 6+ (Minimal API):**

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddScoped<IUserRepository, UserRepository>();
builder.Services.AddSingleton<ICacheService, CacheService>();

var app = builder.Build();
app.Run();
```

---

### 5. What is the difference between IConfiguration and IOptions in .NET?

**IConfiguration**: Direct access to configuration values  
**IOptions<T>**: Strongly-typed access with additional features

```csharp
// appsettings.json
{
    "ConnectionStrings": {
        "Default": "Server=..."
    },
    "EmailSettings": {
        "SmtpServer": "smtp.gmail.com",
        "Port": 587,
        "FromEmail": "noreply@example.com"
    }
}

// Using IConfiguration
public class EmailService
{
    private readonly IConfiguration _config;
    
    public EmailService(IConfiguration config)
    {
        _config = config;
    }
    
    public void Send()
    {
        var server = _config["EmailSettings:SmtpServer"]; // String-based, no type safety
        var port = _config.GetValue<int>("EmailSettings:Port");
    }
}

// Using IOptions<T> (Recommended)
public class EmailSettings
{
    public string SmtpServer { get; set; }
    public int Port { get; set; }
    public string FromEmail { get; set; }
}

// Register in ConfigureServices
services.Configure<EmailSettings>(Configuration.GetSection("EmailSettings"));

public class EmailService
{
    private readonly EmailSettings _settings;
    
    public EmailService(IOptions<EmailSettings> options)
    {
        _settings = options.Value; // Strongly-typed, validation support
    }
    
    public void Send()
    {
        var server = _settings.SmtpServer; // Type-safe
    }
}
```

**Key Differences:**
- `IConfiguration`: Direct, string-based access
- `IOptions<T>`: Strongly-typed, supports validation, change tracking
- `IOptionsSnapshot<T>`: Reloads per request (Scoped)
- `IOptionsMonitor<T>`: Reloads on config changes (Singleton)

---

### 6. How does the appsettings.json file work in .NET?

`appsettings.json` is the default configuration file for storing app settings.

```json
// appsettings.json (base configuration)
{
    "Logging": {
        "LogLevel": {
            "Default": "Information"
        }
    },
    "ConnectionStrings": {
        "DefaultConnection": "Server=localhost;Database=MyDb;"
    },
    "AppSettings": {
        "MaxUploadSize": 10485760,
        "AllowedHosts": ["example.com", "localhost"]
    }
}

// appsettings.Development.json (overrides for Development)
{
    "Logging": {
        "LogLevel": {
            "Default": "Debug"
        }
    },
    "ConnectionStrings": {
        "DefaultConnection": "Server=localhost;Database=MyDb_Dev;"
    }
}
```

**Loading Configuration:**

```csharp
// Program.cs (.NET 6+)
var builder = WebApplication.CreateBuilder(args);

// Automatically loads:
// 1. appsettings.json
// 2. appsettings.{Environment}.json
// 3. User secrets (in Development)
// 4. Environment variables
// 5. Command-line arguments

// Access configuration
var connectionString = builder.Configuration.GetConnectionString("DefaultConnection");
var maxSize = builder.Configuration.GetValue<int>("AppSettings:MaxUploadSize");

// Bind to object
builder.Services.Configure<AppSettings>(builder.Configuration.GetSection("AppSettings"));
```

**Hierarchy**: Later sources override earlier ones
```
appsettings.json < appsettings.{Environment}.json < Environment Variables < Command-line args
```

---

### 7. What is the significance of the Program.cs file in .NET applications?

`Program.cs` is the **entry point** of the application. It configures and runs the web host.

**.NET 5 and earlier:**

```csharp
public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
                webBuilder.UseKestrel(options =>
                {
                    options.Limits.MaxRequestBodySize = 10485760;
                });
            });
}
```

**.NET 6+ (Minimal Hosting Model):**

```csharp
var builder = WebApplication.CreateBuilder(args);

// Configure services
builder.Services.AddControllers();
builder.Services.AddDbContext<AppDbContext>();

var app = builder.Build();

// Configure middleware pipeline
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

**Key responsibilities:**
- Create and configure the host
- Set up dependency injection
- Configure the request pipeline
- Start the web server

---

## 🔵 Intermediate Questions

### 8. What is Kestrel, and why is it used in .NET?

**Kestrel** is a cross-platform web server built into ASP.NET Core. It's the default and recommended web server.

**Key Features:**
- High-performance, lightweight
- Cross-platform
- Supports HTTP/1.1, HTTP/2, HTTP/3
- Can be used standalone or behind a reverse proxy

**Architecture:**

```
Internet → Reverse Proxy (IIS/Nginx/Apache) → Kestrel → ASP.NET Core App
           ↑ (Optional but recommended in production)
```

**Configuration:**

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.WebHost.ConfigureKestrel(options =>
{
    // Listen on specific port
    options.ListenLocalhost(5000);
    options.ListenAnyIP(5001, listenOptions =>
    {
        listenOptions.UseHttps(); // HTTPS
    });

    // Configure limits
    options.Limits.MaxRequestBodySize = 10 * 1024 * 1024; // 10MB
    options.Limits.MaxConcurrentConnections = 100;
    options.Limits.MaxConcurrentUpgradedConnections = 100;
    options.Limits.RequestHeadersTimeout = TimeSpan.FromMinutes(1);

    // Enable HTTP/2
    options.ConfigureEndpointDefaults(listenOptions =>
    {
        listenOptions.Protocols = HttpProtocols.Http1AndHttp2;
    });
});
```

**Why use reverse proxy in production?**
- Load balancing
- SSL termination
- Security (hide internal infrastructure)
- Static file caching
- URL rewriting

---

### 9. Explain the concept of routing in ASP.NET Core.

**Routing** maps incoming HTTP requests to executable endpoints (actions).

**Two types:**

**1. Conventional Routing:**

```csharp
app.UseEndpoints(endpoints =>
{
    endpoints.MapControllerRoute(
        name: "default",
        pattern: "{controller=Home}/{action=Index}/{id?}");
    
    // Custom routes
    endpoints.MapControllerRoute(
        name: "blog",
        pattern: "blog/{year}/{month}/{day}/{slug}",
        defaults: new { controller = "Blog", action = "Post" });
});

// Matches:
// /Products/Details/5 → ProductsController.Details(5)
// /blog/2025/10/21/my-post → BlogController.Post(...)
```

**2. Attribute Routing (Recommended):**

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    // GET: api/products
    [HttpGet]
    public IActionResult GetAll() => Ok(products);

    // GET: api/products/5
    [HttpGet("{id}")]
    public IActionResult GetById(int id) => Ok(product);

    // GET: api/products/5/reviews
    [HttpGet("{id}/reviews")]
    public IActionResult GetReviews(int id) => Ok(reviews);

    // POST: api/products
    [HttpPost]
    public IActionResult Create([FromBody] Product product) => Created();

    // PUT: api/products/5
    [HttpPut("{id}")]
    public IActionResult Update(int id, [FromBody] Product product) => NoContent();

    // DELETE: api/products/5
    [HttpDelete("{id}")]
    public IActionResult Delete(int id) => NoContent();
}

// Advanced routing
[Route("api/users/{userId}/orders")]
public class OrdersController : ControllerBase
{
    [HttpGet("{orderId}")]
    public IActionResult Get(int userId, int orderId)
    {
        // GET: api/users/10/orders/5
    }
}

// Route constraints
[HttpGet("{id:int}")] // Must be integer
[HttpGet("{name:minlength(3)}")] // Min length 3
[HttpGet("{date:datetime}")] // Must be valid datetime
```

**Endpoint Routing (.NET Core 3.0+):**

```csharp
app.UseRouting(); // Match endpoint

// Middleware can inspect selected endpoint
app.Use(async (context, next) =>
{
    var endpoint = context.GetEndpoint();
    if (endpoint != null)
    {
        Console.WriteLine($"Endpoint: {endpoint.DisplayName}");
    }
    await next();
});

app.UseAuthentication();
app.UseAuthorization();

app.UseEndpoints(endpoints => // Execute endpoint
{
    endpoints.MapControllers();
    endpoints.MapGet("/health", () => "Healthy");
});
```

---

### 10. What is the purpose of ConfigureServices and Configure methods in the Startup class?

These methods define **what services are available** and **how the app processes requests**.

**ConfigureServices** - Register services (DI container):

```csharp
public void ConfigureServices(IServiceCollection services)
{
    // Called FIRST - runs at application startup
    // Purpose: Register services into DI container
    
    // Framework services
    services.AddControllers()
            .AddJsonOptions(options =>
            {
                options.JsonSerializerOptions.PropertyNamingPolicy = null;
            });
    
    // Database
    services.AddDbContext<AppDbContext>(options =>
        options.UseSqlServer(Configuration.GetConnectionString("Default")));
    
    // Authentication
    services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
            .AddJwtBearer(options => { /*...*/ });
    
    // Custom services
    services.AddScoped<IUserRepository, UserRepository>();
    services.AddSingleton<ICacheService, RedisCacheService>();
    
    // Configuration
    services.Configure<EmailSettings>(Configuration.GetSection("Email"));
    
    // HttpClient
    services.AddHttpClient<IExternalApiService, ExternalApiService>();
}
```

**Configure** - Build HTTP request pipeline (middleware):

```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    // Called SECOND - defines how app handles requests
    // Purpose: Configure middleware pipeline (ORDER MATTERS!)
    
    // Exception handling (must be first)
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }
    else
    {
        app.UseExceptionHandler("/error");
        app.UseHsts();
    }
    
    // Request processing
    app.UseHttpsRedirection();
    app.UseStaticFiles();
    
    // Routing
    app.UseRouting();
    
    // CORS (must be after UseRouting, before UseAuthorization)
    app.UseCors("AllowAll");
    
    // Auth (order matters: Authentication before Authorization)
    app.UseAuthentication();
    app.UseAuthorization();
    
    // Custom middleware
    app.UseMiddleware<RequestLoggingMiddleware>();
    
    // Endpoints (must be last)
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
        endpoints.MapHealthChecks("/health");
    });
}
```

**Key Difference:**
- `ConfigureServices`: **WHAT** services are available
- `Configure`: **HOW** requests are processed

---

### 11. How does ASP.NET Core handle cross-origin resource sharing (CORS)?

**CORS** allows controlled access to resources from different origins.

**Configuration:**

```csharp
// Startup.cs / Program.cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddCors(options =>
    {
        // 1. Named policy
        options.AddPolicy("AllowSpecificOrigin", builder =>
        {
            builder.WithOrigins("https://example.com", "https://app.example.com")
                   .AllowAnyMethod()
                   .AllowAnyHeader()
                   .AllowCredentials();
        });

        // 2. Default policy
        options.AddDefaultPolicy(builder =>
        {
            builder.AllowAnyOrigin()
                   .AllowAnyMethod()
                   .AllowAnyHeader();
        });

        // 3. Restrictive policy
        options.AddPolicy("RestrictivePolicy", builder =>
        {
            builder.WithOrigins("https://trusted.com")
                   .WithMethods("GET", "POST")
                   .WithHeaders("Content-Type", "Authorization")
                   .SetIsOriginAllowedToAllowWildcardSubdomains()
                   .WithExposedHeaders("X-Custom-Header");
        });
    });
}

public void Configure(IApplicationBuilder app)
{
    app.UseRouting();
    
    // Apply CORS (must be after UseRouting, before UseAuthorization)
    app.UseCors("AllowSpecificOrigin"); // Use named policy
    
    app.UseAuthorization();
    app.UseEndpoints(endpoints => endpoints.MapControllers());
}
```

**Apply CORS at different levels:**

```csharp
// 1. Controller level
[EnableCors("AllowSpecificOrigin")]
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    public IActionResult Get() => Ok();
}

// 2. Action level
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    [HttpGet]
    [EnableCors("RestrictivePolicy")]
    public IActionResult Get() => Ok();
    
    [HttpPost]
    [DisableCors] // Disable CORS for this action
    public IActionResult Create() => Ok();
}

// 3. Endpoint level (.NET 6+)
app.MapGet("/api/public", () => "Public")
   .RequireCors("AllowSpecificOrigin");
```

**How CORS works:**

```
1. Browser sends preflight request (OPTIONS):
   Origin: https://example.com
   Access-Control-Request-Method: POST

2. Server responds:
   Access-Control-Allow-Origin: https://example.com
   Access-Control-Allow-Methods: GET, POST
   
3. Browser sends actual request if allowed
```

---

### 12. What are the benefits of using Entity Framework Core in .NET applications?

**Entity Framework Core (EF Core)** is a modern ORM for .NET.

**Key Benefits:**

**1. Database Agnostic:**
```csharp
// Switch providers easily
services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString));     // SQL Server
    // options.UseNpgsql(connectionString);      // PostgreSQL
    // options.UseSqlite(connectionString);      // SQLite
    // options.UseInMemoryDatabase("TestDb");    // In-memory
```

**2. LINQ Support:**
```csharp
// Type-safe queries
var users = await _context.Users
    .Where(u => u.IsActive && u.Age > 18)
    .OrderBy(u => u.LastName)
    .Include(u => u.Orders)
    .ToListAsync();
```

**3. Change Tracking:**
```csharp
var user = await _context.Users.FindAsync(1);
user.Email = "newemail@example.com"; // Tracked automatically
await _context.SaveChangesAsync(); // Generates UPDATE
```

**4. Migrations:**
```bash
dotnet ef migrations add InitialCreate
dotnet ef database update
```

**5. Relationships:**
```csharp
public class Order
{
    public int Id { get; set; }
    public int UserId { get; set; }
    public User User { get; set; } // Navigation property
    public List<OrderItem> Items { get; set; }
}

// Eager loading
var orders = await _context.Orders
    .Include(o => o.User)
    .Include(o => o.Items)
        .ThenInclude(i => i.Product)
    .ToListAsync();

// Explicit loading
await _context.Entry(order)
    .Collection(o => o.Items)
    .LoadAsync();

// Lazy loading (requires Microsoft.EntityFrameworkCore.Proxies)
services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString)
           .UseLazyLoadingProxies());
```

**6. Performance Features:**
```csharp
// AsNoTracking for read-only queries
var users = await _context.Users
    .AsNoTracking()
    .ToListAsync();

// Compiled queries
private static readonly Func<AppDbContext, int, Task<User>> _getUserById =
    EF.CompileAsyncQuery((AppDbContext context, int id) =>
        context.Users.FirstOrDefault(u => u.Id == id));

// Split queries
var blogs = await _context.Blogs
    .Include(b => b.Posts)
    .AsSplitQuery() // Generates multiple SQL queries
    .ToListAsync();

// Bulk operations
_context.Users.RemoveRange(usersToDelete);
```

---

### 13. Explain the role of the IHostedService interface in .NET and provide an example

**IHostedService** runs background tasks in ASP.NET Core apps.

```csharp
public interface IHostedService
{
    Task StartAsync(CancellationToken cancellationToken);
    Task StopAsync(CancellationToken cancellationToken);
}
```

**Example: Email Queue Processor**

```csharp
public class EmailQueueHostedService : IHostedService, IDisposable
{
    private readonly ILogger<EmailQueueHostedService> _logger;
    private readonly IServiceProvider _services;
    private Timer _timer;

    public EmailQueueHostedService(
        ILogger<EmailQueueHostedService> logger,
        IServiceProvider services)
    {
        _logger = logger;
        _services = services;
    }

    public Task StartAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("Email Queue Service started");

        // Run every 30 seconds
        _timer = new Timer(ProcessQueue, null, TimeSpan.Zero, TimeSpan.FromSeconds(30));

        return Task.CompletedTask;
    }

    private async void ProcessQueue(object state)
    {
        _logger.LogInformation("Processing email queue...");

        // Create scope for scoped services
        using (var scope = _services.CreateScope())
        {
            var emailService = scope.ServiceProvider.GetRequiredService<IEmailService>();
            var queueService = scope.ServiceProvider.GetRequiredService<IQueueService>();

            var emails = await queueService.GetPendingEmailsAsync();
            
            foreach (var email in emails)
            {
                try
                {
                    await emailService.SendAsync(email);
                    await queueService.MarkAsSentAsync(email.Id);
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, $"Error sending email {email.Id}");
                }
            }
        }
    }

    public Task StopAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("Email Queue Service stopping");
        _timer?.Change(Timeout.Infinite, 0);
        return Task.CompletedTask;
    }

    public void Dispose()
    {
        _timer?.Dispose();
    }
}

// Registration
services.AddHostedService<EmailQueueHostedService>();
```

**Better approach: Use BackgroundService (see question 21)**

---

### 14. How does ASP.NET Core support asynchronous programming?

ASP.NET Core is **async by default** for better scalability.

**Key Features:**

**1. Async Controllers:**
```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly IUserRepository _repository;
    private readonly IEmailService _emailService;

    [HttpGet]
    public async Task<ActionResult<List<User>>> GetAllAsync()
    {
        var users = await _repository.GetAllAsync();
        return Ok(users);
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<User>> GetByIdAsync(int id)
    {
        var user = await _repository.GetByIdAsync(id);
        if (user == null)
            return NotFound();
        return Ok(user);
    }

    [HttpPost]
    public async Task<ActionResult<User>> CreateAsync([FromBody] User user)
    {
        var created = await _repository.AddAsync(user);
        await _emailService.SendWelcomeEmailAsync(user.Email);
        return CreatedAtAction(nameof(GetByIdAsync), new { id = created.Id }, created);
    }
}
```

**2. Async Middleware:**
```csharp
public class RequestTimingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestTimingMiddleware> _logger;

    public RequestTimingMiddleware(RequestDelegate next, ILogger<RequestTimingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var stopwatch = Stopwatch.StartNew();
        
        await _next(context); // Async call to next middleware
        
        stopwatch.Stop();
        _logger.LogInformation($"Request took {stopwatch.ElapsedMilliseconds}ms");
    }
}
```

**3. Parallel Operations:**
```csharp
public async Task<DashboardViewModel> GetDashboardAsync(int userId)
{
    // Execute multiple async operations in parallel
    var userTask = _userRepository.GetByIdAsync(userId);
    var ordersTask = _orderRepository.GetUserOrdersAsync(userId);
    var statsTask = _analyticsService.GetUserStatsAsync(userId);

    // Wait for all to complete
    await Task.WhenAll(userTask, ordersTask, statsTask);

    return new DashboardViewModel
    {
        User = await userTask,
        Orders = await ordersTask,
        Stats = await statsTask
    };
}
```

**4. Async Streams (IAsyncEnumerable):**
```csharp
[HttpGet("stream")]
public async IAsyncEnumerable<User> StreamUsersAsync()
{
    await foreach (var user in _repository.GetUsersStreamAsync())
    {
        yield return user;
    }
}

// Repository
public async IAsyncEnumerable<User> GetUsersStreamAsync()
{
    await using var connection = new SqlConnection(_connectionString);
    await connection.OpenAsync();

    await using var command = new SqlCommand("SELECT * FROM Users", connection);
    await using var reader = await command.ExecuteReaderAsync();

    while (await reader.ReadAsync())
    {
        yield return new User
        {
            Id = reader.GetInt32(0),
            Name = reader.GetString(1)
        };
    }
}
```

**Benefits:**
- Non-blocking I/O operations
- Better thread utilization
- Improved scalability (handles more concurrent requests)
- Responsive applications

---

## 🔴 Advanced Questions

### 15. How does the ASP.NET Core request processing pipeline work internally?

**Complete Pipeline Flow:**

```
1. HTTP Request arrives at Kestrel
   ↓
2. HttpContext created (represents request/response)
   ↓
3. Request passes through Middleware Pipeline (order matters!)
   │
   ├→ Exception Handling Middleware
   │   ├→ UseDeveloperExceptionPage() / UseExceptionHandler()
   │   └→ Catches exceptions from downstream middleware
   │
   ├→ HTTPS Redirection Middleware
   │   └→ UseHttpsRedirection()
   │
   ├→ Static Files Middleware
   │   ├→ UseStaticFiles()
   │   └→ Short-circuits if static file found
   │
   ├→ Routing Middleware
   │   ├→ UseRouting()
   │   └→ Selects endpoint (doesn't execute yet)
   │
   ├→ CORS Middleware
   │   └→ UseCors()
   │
   ├→ Authentication Middleware
   │   ├→ UseAuthentication()
   │   └→ Populates HttpContext.User
   │
   ├→ Authorization Middleware
   │   ├→ UseAuthorization()
   │   └→ Checks if user can access endpoint
   │
   ├→ Custom Middleware
   │   └→ Your custom logic
   │
   └→ Endpoint Middleware
       ├→ UseEndpoints() / MapControllers()
       └→ Executes selected endpoint (Controller Action)
           │
           ├→ Model Binding
           │   ├→ [FromRoute], [FromQuery], [FromBody], [FromHeader]
           │   └→ Binds request data to action parameters
           │
           ├→ Model Validation
           │   └→ Validates based on Data Annotations
           │
           ├→ Action Filters (in order)
           │   ├→ OnActionExecuting
           │   ├→ Action Method Execution
           │   └→ OnActionExecuted
           │
           ├→ Result Filters
           │   ├→ OnResultExecuting
           │   ├→ Result Execution (serialization)
           │   └→ OnResultExecuted
           │
           └→ Response generated
   
4. Response travels back through middleware (reverse order)
   ↓
5. Response sent to client via Kestrel
```

**Detailed Example:**

```csharp
public class Startup
{
    public void Configure(IApplicationBuilder app)
    {
        // 1. Exception handling (outermost)
        app.UseExceptionHandler("/error");
        
        // 2. Custom middleware - Request Logging
        app.Use(async (context, next) =>
        {
            Console.WriteLine($"[{DateTime.Now}] Request: {context.Request.Path}");
            await next(); // Pass to next middleware
            Console.WriteLine($"[{DateTime.Now}] Response: {context.Response.StatusCode}");
        });
        
        // 3. Static files (can short-circuit)
        app.UseStaticFiles();
        
        // 4. Routing (selects endpoint)
        app.UseRouting();
        
        // 5. CORS
        app.UseCors("AllowAll");
        
        // 6. Authentication (populates User)
        app.UseAuthentication();
        
        // 7. Authorization (checks permissions)
        app.UseAuthorization();
        
        // 8. Custom middleware - Performance monitoring
        app.Use(async (context, next) =>
        {
            var endpoint = context.GetEndpoint();
            if (endpoint != null)
            {
                var stopwatch = Stopwatch.StartNew();
                await next();
                stopwatch.Stop();
                context.Response.Headers.Add("X-Response-Time", $"{stopwatch.ElapsedMilliseconds}ms");
            }
            else
            {
                await next();
            }
        });
        
        // 9. Endpoints (executes matched endpoint)
        app.UseEndpoints(endpoints =>
        {
            endpoints.MapControllers();
            endpoints.MapFallback(() => Results.NotFound());
        });
    }
}
```

**Action Execution Pipeline:**

```csharp
[ApiController]
[Route("api/[controller]")]
[ServiceFilter(typeof(LoggingActionFilter))]
public class UsersController : ControllerBase
{
    [HttpPost]
    [Authorize(Roles = "Admin")]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Create([FromBody] CreateUserDto dto)
    {
        // 1. Authorization checked first
        // 2. Model binding: dto populated from request body
        // 3. Model validation: validates dto
        // 4. Action filters: OnActionExecuting
        // 5. THIS METHOD EXECUTES
        // 6. Action filters: OnActionExecuted
        // 7. Result filters: OnResultExecuting
        // 8. Result execution: serializes return value
        // 9. Result filters: OnResultExecuted
        
        var user = await _userService.CreateAsync(dto);
        return CreatedAtAction(nameof(GetById), new { id = user.Id }, user);
    }
}

public class LoggingActionFilter : IActionFilter
{
    public void OnActionExecuting(ActionExecutingContext context)
    {
        // Before action executes
        Console.WriteLine($"Executing: {context.ActionDescriptor.DisplayName}");
    }

    public void OnActionExecuted(ActionExecutedContext context)
    {
        // After action executes
        Console.WriteLine($"Executed: {context.ActionDescriptor.DisplayName}");
    }
}
```

---

### 16. How would you design and implement a microservices architecture using ASP.NET Core?

**Comprehensive Microservices Design:**

**Architecture Overview:**

```
API Gateway (Ocelot/YARP)
│
├─> User Service (Auth, Profile)
│   ├─> SQL Server
│   └─> Redis Cache
│
├─> Product Service (Catalog)
│   └─> PostgreSQL
│
├─> Order Service (Orders, Payments)
│   ├─> SQL Server
│   └─> RabbitMQ/Kafka
│
├─> Notification Service (Email, SMS)
│   └─> RabbitMQ/Kafka
│
└─> Service Discovery (Consul/Eureka)
```

**1. API Gateway (Ocelot):**

```json
// ocelot.json
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/users/{everything}",
      "DownstreamScheme": "https",
      "DownstreamHostAndPorts": [
        {
          "Host": "user-service",
          "Port": 5001
        }
      ],
      "UpstreamPathTemplate": "/users/{everything}",
      "UpstreamHttpMethod": [ "Get", "Post", "Put", "Delete" ],
      "AuthenticationOptions": {
        "AuthenticationProviderKey": "Bearer"
      },
      "RateLimitOptions": {
        "EnableRateLimiting": true,
        "Period": "1s",
        "Limit": 10
      }
    },
    {
      "DownstreamPathTemplate": "/api/products/{everything}",
      "DownstreamHostAndPorts": [
        {
          "Host": "product-service",
          "Port": 5002
        }
      ],
      "UpstreamPathTemplate": "/products/{everything}",
      "LoadBalancerOptions": {
        "Type": "RoundRobin"
      }
    }
  ],
  "GlobalConfiguration": {
    "ServiceDiscoveryProvider": {
      "Type": "Consul",
      "Host": "consul",
      "Port": 8500
    }
  }
}
```

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Configuration.AddJsonFile("ocelot.json", optional: false, reloadOnChange: true);
builder.Services.AddOcelot();

var app = builder.Build();
await app.UseOcelot();
app.Run();
```

**2. Individual Microservice (User Service):**

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Service Registration
builder.Services.AddControllers();
builder.Services.AddDbContext<UserDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("UserDb")));

// JWT Authentication
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]))
        };
    });

// Service Discovery (Consul)
builder.Services.AddConsul();
builder.Services.AddConsulServiceDiscovery();

// Health Checks
builder.Services.AddHealthChecks()
    .AddDbContextCheck<UserDbContext>()
    .AddRedis(builder.Configuration.GetConnectionString("Redis"));

// Distributed Tracing (OpenTelemetry)
builder.Services.AddOpenTelemetryTracing(tracerProviderBuilder =>
{
    tracerProviderBuilder
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddSqlClientInstrumentation()
        .AddJaegerExporter();
});

// Application Services
builder.Services.AddScoped<IUserRepository, UserRepository>();
builder.Services.AddScoped<IUserService, UserService>();

var app = builder.Build();

// Middleware
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
app.MapHealthChecks("/health");

// Register with Consul
var lifetime = app.Services.GetRequiredService<IHostApplicationLifetime>();
var consulClient = app.Services.GetRequiredService<IConsulClient>();

var registration = new AgentServiceRegistration
{
    ID = $"user-service-{Guid.NewGuid()}",
    Name = "user-service",
    Address = "localhost",
    Port = 5001,
    Check = new AgentServiceCheck
    {
        HTTP = "http://localhost:5001/health",
        Interval = TimeSpan.FromSeconds(10)
    }
};

lifetime.ApplicationStarted.Register(async () =>
{
    await consulClient.Agent.ServiceRegister(registration);
});

lifetime.ApplicationStopping.Register(async () =>
{
    await consulClient.Agent.ServiceDeregister(registration.ID);
});

app.Run();
```

**3. Inter-Service Communication:**

**a) Synchronous (HTTP via HttpClient):**

```csharp
public class ProductService : IProductService
{
    private readonly HttpClient _httpClient;

    public ProductService(IHttpClientFactory httpClientFactory)
    {
        _httpClient = httpClientFactory.CreateClient("UserService");
    }

    public async Task<ProductWithUserDto> GetProductWithUserAsync(int productId)
    {
        var product = await _repository.GetByIdAsync(productId);
        
        // Call User Service
        var response = await _httpClient.GetAsync($"/api/users/{product.CreatedBy}");
        response.EnsureSuccessStatusCode();
        
        var user = await response.Content.ReadFromJsonAsync<UserDto>();
        
        return new ProductWithUserDto
        {
            Product = product,
            CreatedByUser = user
        };
    }
}

// Configure in Program.cs
builder.Services.AddHttpClient("UserService", client =>
{
    client.BaseAddress = new Uri("http://user-service:5001");
    client.DefaultRequestHeaders.Add("Accept", "application/json");
})
.AddPolicyHandler(GetRetryPolicy())
.AddPolicyHandler(GetCircuitBreakerPolicy());

// Resilience with Polly
static IAsyncPolicy<HttpResponseMessage> GetRetryPolicy()
{
    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .WaitAndRetryAsync(3, retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)));
}

static IAsyncPolicy<HttpResponseMessage> GetCircuitBreakerPolicy()
{
    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .CircuitBreakerAsync(5, TimeSpan.FromSeconds(30));
}
```

**b) Asynchronous (Message Queue - RabbitMQ):**

```csharp
// Event
public class OrderCreatedEvent
{
    public int OrderId { get; set; }
    public int UserId { get; set; }
    public decimal TotalAmount { get; set; }
    public string UserEmail { get; set; }
}

// Publisher (Order Service)
public class OrderService : IOrderService
{
    private readonly IMessageBus _messageBus;

    public async Task<Order> CreateOrderAsync(CreateOrderDto dto)
    {
        var order = await _repository.AddAsync(new Order { /*...*/ });
        
        // Publish event
        await _messageBus.PublishAsync(new OrderCreatedEvent
        {
            OrderId = order.Id,
            UserId = order.UserId,
            TotalAmount = order.TotalAmount,
            UserEmail = order.User.Email
        });
        
        return order;
    }
}

// Consumer (Notification Service)
public class OrderCreatedEventHandler : IEventHandler<OrderCreatedEvent>
{
    private readonly IEmailService _emailService;

    public async Task HandleAsync(OrderCreatedEvent @event)
    {
        await _emailService.SendOrderConfirmationAsync(
            @event.UserEmail,
            @event.OrderId,
            @event.TotalAmount);
    }
}

// RabbitMQ Configuration
public class RabbitMQMessageBus : IMessageBus
{
    private readonly IConnection _connection;
    private readonly IModel _channel;

    public RabbitMQMessageBus(string hostname)
    {
        var factory = new ConnectionFactory { HostName = hostname };
        _connection = factory.CreateConnection();
        _channel = _connection.CreateModel();
    }

    public async Task PublishAsync<T>(T message) where T : class
    {
        var json = JsonSerializer.Serialize(message);
        var body = Encoding.UTF8.GetBytes(json);

        _channel.BasicPublish(
            exchange: "",
            routingKey: typeof(T).Name,
            basicProperties: null,
            body: body);
    }
}
```

**4. Database Per Service Pattern:**

```csharp
// User Service - UserDbContext
public class UserDbContext : DbContext
{
    public DbSet<User> Users { get; set; }
    public DbSet<UserProfile> UserProfiles { get; set; }
}

// Order Service - OrderDbContext
public class OrderDbContext : DbContext
{
    public DbSet<Order> Orders { get; set; }
    public DbSet<OrderItem> OrderItems { get; set; }
    // NO reference to User entity - uses UserId only
}
```

**5. Docker Compose:**

```yaml
version: '3.8'

services:
  api-gateway:
    build: ./ApiGateway
    ports:
      - "5000:80"
    depends_on:
      - user-service
      - product-service
      - order-service

  user-service:
    build: ./UserService
    environment:
      - ConnectionStrings__UserDb=Server=sql-server;Database=UserDb;
      - Consul__Host=consul
    depends_on:
      - sql-server
      - consul

  product-service:
    build: ./ProductService
    environment:
      - ConnectionStrings__ProductDb=Server=postgres;Database=ProductDb;
    depends_on:
      - postgres

  order-service:
    build: ./OrderService
    environment:
      - RabbitMQ__Host=rabbitmq
    depends_on:
      - rabbitmq

  notification-service:
    build: ./NotificationService
    depends_on:
      - rabbitmq

  sql-server:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=YourStrong@Passw0rd

  postgres:
    image: postgres:15
    environment:
      - POSTGRES_PASSWORD=postgres

  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "15672:15672"

  consul:
    image: consul:latest
    ports:
      - "8500:8500"

  redis:
    image: redis:alpine
```

**Key Patterns:**
- API Gateway for unified entry point
- Service Discovery for dynamic service location
- Circuit Breaker for resilience
- Event-Driven for loose coupling
- Health Checks for monitoring
- Distributed Tracing for debugging

---

### 17. What is the difference between IApplicationBuilder and IServiceCollection in .NET?

**IServiceCollection** - Dependency Injection Container (Services)  
**IApplicationBuilder** - Request Pipeline Builder (Middleware)

```csharp
public class Startup
{
    // IServiceCollection: Register SERVICES (WHAT is available)
    public void ConfigureServices(IServiceCollection services)
    {
        // Framework services
        services.AddControllers();
        services.AddAuthentication();
        services.AddAuthorization();
        
        // Database
        services.AddDbContext<AppDbContext>();
        
        // Custom services - Dependency Injection
        services.AddScoped<IUserService, UserService>();
        services.AddSingleton<ICacheService, CacheService>();
        services.AddTransient<IEmailService, EmailService>();
        
        // Configuration
        services.Configure<AppSettings>(Configuration.GetSection("AppSettings"));
        
        // HttpClient
        services.AddHttpClient<IExternalApi, ExternalApi>();
        
        // Returns: IServiceCollection (for chaining)
    }

    // IApplicationBuilder: Configure MIDDLEWARE (HOW requests are processed)
    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        // Request pipeline - ORDER MATTERS!
        app.UseExceptionHandler("/error");  // Error handling
        app.UseHttpsRedirection();          // Redirect HTTP to HTTPS
        app.UseStaticFiles();                // Serve static files
        app.UseRouting();                    // Route matching
        app.UseAuthentication();             // Identify user
        app.UseAuthorization();              // Check permissions
        
        // Custom middleware
        app.Use(async (context, next) =>
        {
            // Before
            await next();
            // After
        });
        
        app.UseEndpoints(endpoints =>        // Execute endpoint
        {
            endpoints.MapControllers();
        });
        
        // Returns: IApplicationBuilder (for chaining)
    }
}
```

**Key Differences:**

| Aspect | IServiceCollection | IApplicationBuilder |
|--------|-------------------|---------------------|
| **Purpose** | Register services for DI | Build request pipeline |
| **When** | Application startup (once) | Application startup (once) |
| **What** | Services, repositories, options | Middleware components |
| **Order** | Doesn't matter | **CRITICAL** - order matters |
| **Extension Methods** | Add* (AddControllers, AddDbContext) | Use* (UseRouting, UseAuthentication) |
| **Returns** | IServiceCollection | IApplicationBuilder |

**Extension Method Examples:**

```csharp
// IServiceCollection extensions
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddCustomServices(this IServiceCollection services)
    {
        services.AddScoped<IUserRepository, UserRepository>();
        services.AddScoped<IUserService, UserService>();
        return services;
    }
}

// Usage
services.AddCustomServices();

// IApplicationBuilder extensions
public static class ApplicationBuilderExtensions
{
    public static IApplicationBuilder UseCustomMiddleware(this IApplicationBuilder app)
    {
        app.UseMiddleware<RequestLoggingMiddleware>();
        app.UseMiddleware<PerformanceMiddleware>();
        return app;
    }
}

// Usage
app.UseCustomMiddleware();
```

---

### 18. How can you implement centralized logging and monitoring in a .NET application?

**Comprehensive Logging & Monitoring Solution:**

**1. Structured Logging with Serilog:**

```csharp
// Install: Serilog.AspNetCore, Serilog.Sinks.Console, Serilog.Sinks.File, Serilog.Sinks.Elasticsearch

// Program.cs
using Serilog;
using Serilog.Events;

var builder = WebApplication.CreateBuilder(args);

// Configure Serilog
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
    .MinimumLevel.Override("System", LogEventLevel.Warning)
    .Enrich.FromLogContext()
    .Enrich.WithMachineName()
    .Enrich.WithThreadId()
    .Enrich.WithProperty("Application", "MyApp")
    .Enrich.WithProperty("Environment", builder.Environment.EnvironmentName)
    .WriteTo.Console()
    .WriteTo.File(
        "logs/log-.txt",
        rollingInterval: RollingInterval.Day,
        outputTemplate: "{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level:u3}] {Message:lj}{NewLine}{Exception}")
    .WriteTo.Elasticsearch(new ElasticsearchSinkOptions(new Uri("http://elasticsearch:9200"))
    {
        AutoRegisterTemplate = true,
        IndexFormat = "myapp-logs-{0:yyyy.MM.dd}",
        NumberOfReplicas = 1,
        NumberOfShards = 2
    })
    .WriteTo.Seq("http://seq:5341") // Structured log viewer
    .CreateLogger();

builder.Host.UseSerilog();

var app = builder.Build();

// Request logging middleware
app.UseSerilogRequestLogging(options =>
{
    options.MessageTemplate = "HTTP {RequestMethod} {RequestPath} responded {StatusCode} in {Elapsed:0.0000}ms";
    options.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
    {
        diagnosticContext.Set("RequestHost", httpContext.Request.Host.Value);
        diagnosticContext.Set("UserAgent", httpContext.Request.Headers["User-Agent"]);
        diagnosticContext.Set("UserId", httpContext.User.FindFirst("sub")?.Value);
    };
});

app.MapControllers();
app.Run();
```

**2. Usage in Application:**

```csharp
public class UserService : IUserService
{
    private readonly ILogger<UserService> _logger;
    private readonly IUserRepository _repository;

    public UserService(ILogger<UserService> logger, IUserRepository repository)
    {
        _logger = logger;
        _repository = repository;
    }

    public async Task<User> CreateUserAsync(CreateUserDto dto)
    {
        // Structured logging with properties
        _logger.LogInformation("Creating user with email {Email}", dto.Email);

        try
        {
            var user = await _repository.AddAsync(new User
            {
                Email = dto.Email,
                Name = dto.Name
            });

            _logger.LogInformation(
                "User created successfully. UserId: {UserId}, Email: {Email}",
                user.Id,
                user.Email);

            return user;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to create user with email {Email}", dto.Email);
            throw;
        }
    }

    public async Task<User> GetByIdAsync(int id)
    {
        using (_logger.BeginScope(new Dictionary<string, object>
        {
            ["UserId"] = id,
            ["Operation"] = "GetUserById"
        }))
        {
            _logger.LogDebug("Fetching user from database");
            var user = await _repository.GetByIdAsync(id);

            if (user == null)
            {
                _logger.LogWarning("User not found");
                return null;
            }

            _logger.LogInformation("User retrieved successfully");
            return user;
        }
    }
}
```

**3. Application Performance Monitoring (APM) with Application Insights:**

```csharp
// Install: Microsoft.ApplicationInsights.AspNetCore

// Program.cs
builder.Services.AddApplicationInsightsTelemetry(options =>
{
    options.ConnectionString = builder.Configuration["ApplicationInsights:ConnectionString"];
    options.EnableAdaptiveSampling = true;
    options.EnablePerformanceCounterCollectionModule = true;
});

// Track custom metrics
public class OrderService : IOrderService
{
    private readonly TelemetryClient _telemetryClient;

    public OrderService(TelemetryClient telemetryClient)
    {
        _telemetryClient = telemetryClient;
    }

    public async Task<Order> CreateOrderAsync(CreateOrderDto dto)
    {
        var stopwatch = Stopwatch.StartNew();

        try
        {
            var order = await ProcessOrderAsync(dto);

            // Track custom metric
            _telemetryClient.TrackMetric("OrderValue", order.TotalAmount);
            _telemetryClient.TrackEvent("OrderCreated", new Dictionary<string, string>
            {
                ["OrderId"] = order.Id.ToString(),
                ["UserId"] = order.UserId.ToString()
            });

            return order;
        }
        catch (Exception ex)
        {
            _telemetryClient.TrackException(ex);
            throw;
        }
        finally
        {
            stopwatch.Stop();
            _telemetryClient.TrackMetric("OrderCreationTime", stopwatch.ElapsedMilliseconds);
        }
    }
}
```

**4. Health Checks & Monitoring Endpoints:**

```csharp
// Install: AspNetCore.HealthChecks.SqlServer, AspNetCore.HealthChecks.Redis

builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>("database")
    .AddRedis(builder.Configuration.GetConnectionString("Redis"), "redis")
    .AddUrlGroup(new Uri("https://external-api.com/health"), "external-api")
    .AddCheck<CustomHealthCheck>("custom-check");

// Custom health check
public class CustomHealthCheck : IHealthCheck
{
    private readonly IServiceProvider _serviceProvider;

    public CustomHealthCheck(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            // Check something
            var isHealthy = await CheckSomethingAsync();

            return isHealthy
                ? HealthCheckResult.Healthy("Service is healthy")
                : HealthCheckResult.Degraded("Service is degraded");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Service is unhealthy", ex);
        }
    }
}

// Configure endpoints
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = async (context, report) =>
    {
        context.Response.ContentType = "application/json";

        var result = JsonSerializer.Serialize(new
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
        });

        await context.Response.WriteAsync(result);
    }
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false // Only checks app is running
});
```

**5. ELK Stack Integration (Elasticsearch, Logstash, Kibana):**

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    depends_on:
      - elasticsearch
      - seq

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    ports:
      - "9200:9200"

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

  seq:
    image: datalust/seq:latest
    ports:
      - "5341:80"
    environment:
      - ACCEPT_EULA=Y
```

**6. Distributed Tracing with OpenTelemetry:**

```csharp
// Install: OpenTelemetry.Extensions.Hosting, OpenTelemetry.Instrumentation.AspNetCore

builder.Services.AddOpenTelemetryTracing(tracerProviderBuilder =>
{
    tracerProviderBuilder
        .AddSource("MyApp")
        .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("MyApp"))
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddSqlClientInstrumentation()
        .AddJaegerExporter(options =>
        {
            options.AgentHost = "jaeger";
            options.AgentPort = 6831;
        });
});
```

---

### 19. Explain the difference between transient, scoped, and singleton lifetimes in dependency injection.

**Service Lifetimes:**

```csharp
public void ConfigureServices(IServiceCollection services)
{
    // 1. TRANSIENT - New instance every time requested
    services.AddTransient<IEmailService, EmailService>();
    
    // 2. SCOPED - One instance per HTTP request
    services.AddScoped<IUserRepository, UserRepository>();
    
    // 3. SINGLETON - One instance for application lifetime
    services.AddSingleton<ICacheService, CacheService>();
}
```

**Detailed Comparison:**

| Lifetime | Creation | Lifetime | Use Cases | Thread Safety |
|----------|----------|----------|-----------|---------------|
| **Transient** | Every time requested | Very short | Lightweight, stateless services | Not required |
| **Scoped** | Once per request | HTTP request duration | Database contexts, repositories | Not required (single-threaded per request) |
| **Singleton** | Once per application | Application lifetime | Caching, configuration, logging | **MUST BE thread-safe** |

**Visual Example:**

```csharp
public interface IOperationTransient { Guid OperationId { get; } }
public interface IOperationScoped { Guid OperationId { get; } }
public interface IOperationSingleton { Guid OperationId { get; } }

public class Operation : IOperationTransient, IOperationScoped, IOperationSingleton
{
    public Guid OperationId { get; }
    public Operation() => OperationId = Guid.NewGuid();
}

// Register
services.AddTransient<IOperationTransient, Operation>();
services.AddScoped<IOperationScoped, Operation>();
services.AddSingleton<IOperationSingleton, Operation>();

// Controller
public class OperationsController : ControllerBase
{
    private readonly IOperationTransient _transient1;
    private readonly IOperationTransient _transient2;
    private readonly IOperationScoped _scoped1;
    private readonly IOperationScoped _scoped2;
    private readonly IOperationSingleton _singleton1;
    private readonly IOperationSingleton _singleton2;

    public OperationsController(
        IOperationTransient transient1,
        IOperationTransient transient2,
        IOperationScoped scoped1,
        IOperationScoped scoped2,
        IOperationSingleton singleton1,
        IOperationSingleton singleton2)
    {
        _transient1 = transient1;
        _transient2 = transient2;
        _scoped1 = scoped1;
        _scoped2 = scoped2;
        _singleton1 = singleton1;
        _singleton2 = singleton2;
    }

    [HttpGet]
    public IActionResult Get()
    {
        return Ok(new
        {
            // Different GUIDs - new instance each time
            Transient1 = _transient1.OperationId,  // abc-123
            Transient2 = _transient2.OperationId,  // def-456
            
            // Same GUID within same request
            Scoped1 = _scoped1.OperationId,        // ghi-789
            Scoped2 = _scoped2.OperationId,        // ghi-789
            
            // Same GUID across all requests (application lifetime)
            Singleton1 = _singleton1.OperationId,  // jkl-000
            Singleton2 = _singleton2.OperationId   // jkl-000
        });
    }
}
```

**Real-World Examples:**

```csharp
// TRANSIENT - Stateless, lightweight services
public interface IEmailService
{
    Task SendAsync(string to, string subject, string body);
}

public class EmailService : IEmailService
{
    // No shared state, can be recreated frequently
    public async Task SendAsync(string to, string subject, string body)
    {
        // Send email logic
    }
}

services.AddTransient<IEmailService, EmailService>();

// SCOPED - Database operations (one DbContext per request)
public class UserRepository : IUserRepository
{
    private readonly AppDbContext _context; // Should be scoped
    
    public UserRepository(AppDbContext context)
    {
        _context = context;
    }
    
    public async Task<User> GetByIdAsync(int id)
    {
        return await _context.Users.FindAsync(id);
    }
}

services.AddScoped<IUserRepository, UserRepository>();
services.AddDbContext<AppDbContext>(); // Scoped by default

// SINGLETON - Caching, Configuration (shared across entire app)
public class CacheService : ICacheService
{
    private readonly ConcurrentDictionary<string, object> _cache = new();
    
    public void Set(string key, object value)
    {
        _cache[key] = value; // Must be thread-safe!
    }
    
    public T Get<T>(string key)
    {
        return _cache.TryGetValue(key, out var value) ? (T)value : default;
    }
}

services.AddSingleton<ICacheService, CacheService>();
```

**Captive Dependency Problem:**

```csharp
// ❌ BAD: Singleton depends on Scoped
public class SingletonService // Singleton
{
    private readonly AppDbContext _context; // Scoped - PROBLEM!
    
    // This will cause issues - DbContext will be captured once
    // and used across all requests (connection issues, stale data)
}

// ✅ GOOD: Use IServiceProvider to resolve scoped services
public class SingletonService
{
    private readonly IServiceProvider _serviceProvider;
    
    public SingletonService(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }
    
    public async Task DoWorkAsync()
    {
        using var scope = _serviceProvider.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        // Use context
    }
}
```

**Best Practices:**

1. **Transient**: Lightweight, stateless services (helpers, utilities)
2. **Scoped**: Database contexts, repositories, request-specific data
3. **Singleton**: Expensive to create, thread-safe, stateless (caching, config)
4. **Never**: Inject Scoped into Singleton (captive dependency)

---

### 20. How would you implement health checks in an ASP.NET Core application?

```csharp
// Install: Microsoft.Extensions.Diagnostics.HealthChecks
//          AspNetCore.HealthChecks.SqlServer
//          AspNetCore.HealthChecks.Redis

// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddHealthChecks()
    // 1. Basic checks
    .AddCheck("self", () => HealthCheckResult.Healthy("API is running"))
    
    // 2. Database checks
    .AddSqlServer(
        connectionString: builder.Configuration.GetConnectionString("DefaultConnection"),
        name: "sql-server",
        failureStatus: HealthStatus.Degraded,
        tags: new[] { "db", "sql", "ready" })
    
    // 3. Redis cache check
    .AddRedis(
        redisConnectionString: builder.Configuration.GetConnectionString("Redis"),
        name: "redis",
        tags: new[] { "cache", "ready" })
    
    // 4. External API check
    .AddUrlGroup(
        new Uri("https://api.external.com/health"),
        name: "external-api",
        failureStatus: HealthStatus.Degraded,
        tags: new[] { "external" })
    
    // 5. Custom health checks
    .AddCheck<DatabaseHealthCheck>("database-custom")
    
    // 6. Memory check
    .AddCheck("memory", () =>
    {
        var allocated = GC.GetTotalMemory(forceFullCollection: false);
        var threshold = 1024L * 1024L * 1024L; // 1GB
        
        return allocated < threshold
            ? HealthCheckResult.Healthy($"Memory: {allocated / 1024 / 1024}MB")
            : HealthCheckResult.Degraded($"High memory usage: {allocated / 1024 / 1024}MB");
    }, tags: new[] { "memory" });

var app = builder.Build();

// Configure health check endpoints
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = async (context, report) =>
    {
        context.Response.ContentType = "application/json";
        
        var result = JsonSerializer.Serialize(new
        {
            status = report.Status.ToString(),
            timestamp = DateTime.UtcNow,
            duration = report.TotalDuration.TotalMilliseconds,
            checks = report.Entries.Select(e => new
            {
                name = e.Key,
                status = e.Value.Status.ToString(),
                description = e.Value.Description,
                duration = e.Value.Duration.TotalMilliseconds,
                exception = e.Value.Exception?.Message
            })
        }, new JsonSerializerOptions { WriteIndented = true });
        
        await context.Response.WriteAsync(result);
    }
});

// Kubernetes liveness probe (is app running?)
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false // No checks, just returns 200 if app is running
});

// Kubernetes readiness probe (is app ready to serve traffic?)
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});

app.Run();
```

**Custom Health Checks:**

```csharp
public class DatabaseHealthCheck : IHealthCheck
{
    private readonly AppDbContext _context;
    
    public DatabaseHealthCheck(AppDbContext context)
    {
        _context = context;
    }
    
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var stopwatch = Stopwatch.StartNew();
            await _context.Database.CanConnectAsync(cancellationToken);
            stopwatch.Stop();
            
            var responseTime = stopwatch.ElapsedMilliseconds;
            
            if (responseTime > 1000)
            {
                return HealthCheckResult.Degraded(
                    $"Database is slow: {responseTime}ms");
            }
            
            return HealthCheckResult.Healthy($"Database: {responseTime}ms");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Database unreachable", ex);
        }
    }
}
```

---

### 21. What is the purpose of the BackgroundService class, and how does it relate to IHostedService?

**BackgroundService** is an abstract base class that simplifies implementing **IHostedService**.

```csharp
// IHostedService (interface)
public interface IHostedService
{
    Task StartAsync(CancellationToken cancellationToken);
    Task StopAsync(CancellationToken cancellationToken);
}

// BackgroundService (abstract class implementing IHostedService)
public abstract class BackgroundService : IHostedService
{
    protected abstract Task ExecuteAsync(CancellationToken stoppingToken);
}
```

**Example:**

```csharp
public class DataCleanupService : BackgroundService
{
    private readonly ILogger<DataCleanupService> _logger;
    private readonly IServiceProvider _serviceProvider;
    
    public DataCleanupService(
        ILogger<DataCleanupService> logger,
        IServiceProvider serviceProvider)
    {
        _logger = logger;
        _serviceProvider = serviceProvider;
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Cleanup Service started");
        
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                using (var scope = _serviceProvider.CreateScope())
                {
                    var dbContext = scope.ServiceProvider.GetRequiredService<AppDbContext>();
                    
                    var cutoffDate = DateTime.UtcNow.AddDays(-30);
                    var oldRecords = await dbContext.Logs
                        .Where(l => l.CreatedAt < cutoffDate)
                        .ToListAsync(stoppingToken);
                    
                    dbContext.Logs.RemoveRange(oldRecords);
                    await dbContext.SaveChangesAsync(stoppingToken);
                    
                    _logger.LogInformation("Deleted {Count} old records", oldRecords.Count);
                }
                
                await Task.Delay(TimeSpan.FromHours(24), stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error in cleanup service");
                await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
            }
        }
    }
}

// Register
builder.Services.AddHostedService<DataCleanupService>();
```

---

### 22. How does the .NET Generic Host differ from the Web Host?

**Generic Host** (IHost) - for any .NET application  
**Web Host** (IWebHost) - legacy, web-only

```csharp
// ❌ OLD: Web Host (.NET Core 2.x)
public static IWebHostBuilder CreateWebHostBuilder(string[] args) =>
    WebHost.CreateDefaultBuilder(args)
        .UseStartup<Startup>();

// ✅ NEW: Generic Host (.NET 3.0+)
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder.UseStartup<Startup>();
        });

// ✅ LATEST: .NET 6+ Minimal API
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();
app.Run();
```

**Key Differences:**

| Feature | Web Host | Generic Host |
|---------|----------|--------------|
| **Purpose** | Web apps only | Any application type |
| **Background Services** | Limited | Full IHostedService |
| **Console Apps** | ❌ | ✅ |
| **Worker Services** | ❌ | ✅ |
| **Status** | Legacy | Current standard |

---

### 23. How would you handle global exception handling in ASP.NET Core?

**Custom Exception Middleware:**

```csharp
public class GlobalExceptionHandlerMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<GlobalExceptionHandlerMiddleware> _logger;
    
    public GlobalExceptionHandlerMiddleware(RequestDelegate next, ILogger<GlobalExceptionHandlerMiddleware> logger)
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
            _logger.LogError(ex, "Unhandled exception");
            await HandleExceptionAsync(context, ex);
        }
    }
    
    private static async Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        context.Response.ContentType = "application/json";
        
        var (statusCode, message) = exception switch
        {
            NotFoundException => (404, exception.Message),
            ValidationException => (400, exception.Message),
            UnauthorizedAccessException => (401, "Unauthorized"),
            _ => (500, "Internal server error")
        };
        
        context.Response.StatusCode = statusCode;
        
        await context.Response.WriteAsJsonAsync(new
        {
            statusCode,
            message,
            traceId = context.TraceIdentifier
        });
    }
}

// Register
app.UseMiddleware<GlobalExceptionHandlerMiddleware>();
```

**Custom Exceptions:**

```csharp
public class NotFoundException : Exception
{
    public NotFoundException(string message) : base(message) { }
    
    public NotFoundException(string entity, object key)
        : base($"{entity} with key '{key}' not found") { }
}

public class ValidationException : Exception
{
    public Dictionary<string, string[]> Errors { get; }
    
    public ValidationException(Dictionary<string, string[]> errors)
        : base("Validation failed")
    {
        Errors = errors;
    }
}
```

---

### 24. What is the purpose of DataProtection in .NET, and how would you use it?

**DataProtection** provides cryptographic APIs for protecting sensitive data.

**Configuration:**

```csharp
builder.Services.AddDataProtection()
    .SetApplicationName("MyApp")
    .PersistKeysToFileSystem(new DirectoryInfo(@"C:\keys"))
    .SetDefaultKeyLifetime(TimeSpan.FromDays(90));
```

**Usage:**

```csharp
public class SecureDataService
{
    private readonly IDataProtector _protector;
    
    public SecureDataService(IDataProtectionProvider provider)
    {
        _protector = provider.CreateProtector("SecureDataService");
    }
    
    public string Protect(string plainText)
    {
        return _protector.Protect(plainText);
    }
    
    public string Unprotect(string cipherText)
    {
        return _protector.Unprotect(cipherText);
    }
}

// Usage
var encrypted = secureService.Protect("sensitive-data");
var decrypted = secureService.Unprotect(encrypted);
```

**Time-Limited Protection:**

```csharp
public class TokenService
{
    private readonly ITimeLimitedDataProtector _protector;
    
    public TokenService(IDataProtectionProvider provider)
    {
        var baseProtector = provider.CreateProtector("Tokens");
        _protector = baseProtector.ToTimeLimitedDataProtector();
    }
    
    public string CreateToken(int userId)
    {
        return _protector.Protect($"user:{userId}", TimeSpan.FromHours(1));
    }
    
    public (bool isValid, int userId) ValidateToken(string token)
    {
        try
        {
            var data = _protector.Unprotect(token);
            var userId = int.Parse(data.Split(':')[1]);
            return (true, userId);
        }
        catch
        {
            return (false, 0);
        }
    }
}
```

---

### 25. How can you configure and manage multiple environments in ASP.NET Core?

**Environment Configuration:**

```csharp
// Set environment
// launchSettings.json: "ASPNETCORE_ENVIRONMENT": "Development"
// Command line: dotnet run --environment=Production

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
    app.UseSwagger();
}
else if (app.Environment.IsStaging())
{
    app.UseExceptionHandler("/error");
}
else if (app.Environment.IsProduction())
{
    app.UseExceptionHandler("/error");
    app.UseHsts();
}
```

**Configuration Files:**

```
appsettings.json                    ← Base
appsettings.Development.json        ← Development overrides
appsettings.Staging.json            ← Staging overrides
appsettings.Production.json         ← Production overrides
```

```json
// appsettings.json
{
  "ConnectionStrings": {
    "Default": "Server=localhost;Database=MyApp;"
  },
  "Features": {
    "EnableNewFeature": false
  }
}

// appsettings.Development.json
{
  "ConnectionStrings": {
    "Default": "Server=localhost;Database=MyApp_Dev;"
  },
  "Features": {
    "EnableNewFeature": true
  }
}
```

**Environment-Specific Services:**

```csharp
if (builder.Environment.IsDevelopment())
{
    builder.Services.AddScoped<IEmailService, FakeEmailService>();
}
else
{
    builder.Services.AddScoped<IEmailService, SendGridEmailService>();
}
```

**User Secrets (Development):**

```bash
# Initialize and set secrets
dotnet user-secrets init
dotnet user-secrets set "ApiKey" "my-secret-key"

# Access in code (auto-loaded in Development)
var apiKey = builder.Configuration["ApiKey"];
```

**Azure Key Vault (Production):**

```csharp
if (builder.Environment.IsProduction())
{
    var keyVaultEndpoint = new Uri(builder.Configuration["KeyVault:Endpoint"]);
    builder.Configuration.AddAzureKeyVault(
        keyVaultEndpoint,
        new DefaultAzureCredential());
}
```

---

## 🤝 Contributing

Contributions are welcome! Here's how you can help:

1. **Fork** the repository
2. **Create** a feature branch (`git checkout -b feature/AmazingFeature`)
3. **Commit** your changes (`git commit -m 'Add some AmazingFeature'`)
4. **Push** to the branch (`git push origin feature/AmazingFeature`)
5. **Open** a Pull Request

### Contribution Guidelines

- Add new questions with detailed answers
- Include code examples where applicable
- Ensure code examples are tested and working
- Follow existing formatting and structure
- Add references to official documentation when relevant

---

## 📚 Additional Resources

- [Official .NET Documentation](https://docs.microsoft.com/en-us/dotnet/)
- [ASP.NET Core Documentation](https://docs.microsoft.com/en-us/aspnet/core/)
- [.NET Blog](https://devblogs.microsoft.com/dotnet/)
- [C# Programming Guide](https://docs.microsoft.com/en-us/dotnet/csharp/)
- [Entity Framework Core Documentation](https://docs.microsoft.com/en-us/ef/core/)

---

## 👨‍💻 Author

**[@mbsagin](https://github.com/mbsagin)**

---

## 📧 Contact

Have questions or suggestions? Feel free to open an issue!

---

**Made with ❤️ for the .NET Community**

---

## 🔖 Tags

`dotnet` `csharp` `aspnetcore` `interview-questions` `backend` `software-engineering` `web-development` `microservices` `entity-framework` `dependency-injection` `middleware` `rest-api` `dotnet-core` `programming` `software-development`
