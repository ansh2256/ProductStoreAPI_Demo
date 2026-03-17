# ProductStore API — Complete Demo Project
# ASP.NET Core Web API with EF Core, JWT, Swagger

## Project Structure
```
ProductStoreAPI/
├── Controllers/
│   ├── AuthController.cs
│   └── ProductsController.cs
├── Data/
│   └── AppDbContext.cs
├── Models/
│   ├── Product.cs
│   ├── LoginRequest.cs
│   └── LoginResponse.cs
├── Services/
│   └── JwtService.cs
├── Program.cs
└── appsettings.json
```

---

## STEP 1: Create Project

```bash
dotnet new webapi -n ProductStoreAPI
cd ProductStoreAPI

# Install packages
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Tools
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package Swashbuckle.AspNetCore
```

---

## STEP 2: Models/Product.cs

```csharp
using System.ComponentModel.DataAnnotations;

namespace ProductStoreAPI.Models;

public class Product
{
    public int Id { get; set; }

    [Required(ErrorMessage = "Product name is required")]
    [StringLength(100, ErrorMessage = "Name max 100 chars")]
    public string Name { get; set; } = string.Empty;

    [Required]
    [Range(0.01, 99999.99, ErrorMessage = "Price must be between 0.01 and 99999.99")]
    public decimal Price { get; set; }

    [StringLength(500)]
    public string? Description { get; set; }

    public string Category { get; set; } = "General";

    public int Stock { get; set; } = 0;

    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
}
```

---

## STEP 3: Models/LoginRequest.cs & LoginResponse.cs

```csharp
// LoginRequest.cs
namespace ProductStoreAPI.Models;

public class LoginRequest
{
    public string Username { get; set; } = string.Empty;
    public string Password { get; set; } = string.Empty;
}

// LoginResponse.cs
namespace ProductStoreAPI.Models;

public class LoginResponse
{
    public string Token { get; set; } = string.Empty;
    public string Username { get; set; } = string.Empty;
    public DateTime Expiry { get; set; }
}
```

---

## STEP 4: Data/AppDbContext.cs

```csharp
using Microsoft.EntityFrameworkCore;
using ProductStoreAPI.Models;

namespace ProductStoreAPI.Data;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options)
        : base(options) { }

    public DbSet<Product> Products { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Seed some test data
        modelBuilder.Entity<Product>().HasData(
            new Product { Id = 1, Name = "Apple iPhone 15", Price = 79999, Category = "Electronics", Stock = 50 },
            new Product { Id = 2, Name = "Samsung TV 55\"", Price = 45999, Category = "Electronics", Stock = 20 },
            new Product { Id = 3, Name = "Nike Air Max", Price = 8999, Category = "Footwear", Stock = 100 }
        );
    }
}
```

---

## STEP 5: Services/JwtService.cs

```csharp
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Text;
using Microsoft.IdentityModel.Tokens;

namespace ProductStoreAPI.Services;

public class JwtService
{
    private readonly IConfiguration _config;

    public JwtService(IConfiguration config)
    {
        _config = config;
    }

    public string GenerateToken(string username, string role)
    {
        var jwtSettings = _config.GetSection("JwtSettings");
        var secretKey = jwtSettings["SecretKey"]!;
        var issuer    = jwtSettings["Issuer"]!;
        var audience  = jwtSettings["Audience"]!;
        var expireMin = int.Parse(jwtSettings["ExpiryMinutes"]!);

        var key   = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(secretKey));
        var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var claims = new[]
        {
            new Claim(ClaimTypes.Name, username),
            new Claim(ClaimTypes.Role, role),
            new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
        };

        var token = new JwtSecurityToken(
            issuer:    issuer,
            audience:  audience,
            claims:    claims,
            expires:   DateTime.UtcNow.AddMinutes(expireMin),
            signingCredentials: creds
        );

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

---

## STEP 6: Controllers/AuthController.cs

```csharp
using Microsoft.AspNetCore.Mvc;
using ProductStoreAPI.Models;
using ProductStoreAPI.Services;

namespace ProductStoreAPI.Controllers;

[ApiController]
[Route("api/[controller]")]
public class AuthController : ControllerBase
{
    private readonly JwtService _jwtService;

    public AuthController(JwtService jwtService)
    {
        _jwtService = jwtService;
    }

    /// <summary>Login to get JWT token</summary>
    [HttpPost("login")]
    public IActionResult Login([FromBody] LoginRequest request)
    {
        // 🔴 Demo only — in real app, verify against DB!
        var validUsers = new Dictionary<string, (string Password, string Role)>
        {
            { "admin",   ("admin123",   "Admin") },
            { "user",    ("user123",    "User")  },
        };

        if (!validUsers.TryGetValue(request.Username, out var userInfo)
            || userInfo.Password != request.Password)
        {
            return Unauthorized(new { message = "Invalid username or password" });
        }

        var token  = _jwtService.GenerateToken(request.Username, userInfo.Role);
        var expiry = DateTime.UtcNow.AddMinutes(60);

        return Ok(new LoginResponse
        {
            Token    = token,
            Username = request.Username,
            Expiry   = expiry
        });
    }
}
```

---

## STEP 7: Controllers/ProductsController.cs

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using ProductStoreAPI.Data;
using ProductStoreAPI.Models;

namespace ProductStoreAPI.Controllers;

[ApiController]
[Route("api/[controller]")]
[Authorize]           // 🔒 All endpoints require JWT
public class ProductsController : ControllerBase
{
    private readonly AppDbContext _db;

    public ProductsController(AppDbContext db)
    {
        _db = db;
    }

    // ─────────────────────────────────
    // GET /api/products
    // ─────────────────────────────────
    /// <summary>Get all products</summary>
    [HttpGet]
    [AllowAnonymous]   // Public — no auth needed to browse
    public async Task<ActionResult<IEnumerable<Product>>> GetAll()
    {
        var products = await _db.Products.ToListAsync();
        return Ok(products);
    }

    // ─────────────────────────────────
    // GET /api/products/{id}
    // ─────────────────────────────────
    /// <summary>Get a product by ID</summary>
    [HttpGet("{id}")]
    [AllowAnonymous]
    public async Task<ActionResult<Product>> GetById(int id)
    {
        var product = await _db.Products.FindAsync(id);

        if (product == null)
            return NotFound(new { message = $"Product with ID {id} not found" });

        return Ok(product);
    }

    // ─────────────────────────────────
    // POST /api/products
    // ─────────────────────────────────
    /// <summary>Create a new product</summary>
    [HttpPost]
    [Authorize(Roles = "Admin")]   // Only Admin can create
    public async Task<ActionResult<Product>> Create([FromBody] Product product)
    {
        if (!ModelState.IsValid)
            return BadRequest(ModelState);

        product.CreatedAt = DateTime.UtcNow;
        _db.Products.Add(product);
        await _db.SaveChangesAsync();

        return CreatedAtAction(nameof(GetById), new { id = product.Id }, product);
    }

    // ─────────────────────────────────
    // PUT /api/products/{id}
    // ─────────────────────────────────
    /// <summary>Replace a product completely</summary>
    [HttpPut("{id}")]
    [Authorize(Roles = "Admin")]
    public async Task<IActionResult> Update(int id, [FromBody] Product product)
    {
        if (id != product.Id)
            return BadRequest(new { message = "ID in URL must match ID in body" });

        if (!ModelState.IsValid)
            return BadRequest(ModelState);

        var exists = await _db.Products.AnyAsync(p => p.Id == id);
        if (!exists)
            return NotFound(new { message = $"Product {id} not found" });

        _db.Entry(product).State = EntityState.Modified;
        await _db.SaveChangesAsync();

        return NoContent();
    }

    // ─────────────────────────────────
    // DELETE /api/products/{id}
    // ─────────────────────────────────
    /// <summary>Delete a product</summary>
    [HttpDelete("{id}")]
    [Authorize(Roles = "Admin")]
    public async Task<IActionResult> Delete(int id)
    {
        var product = await _db.Products.FindAsync(id);

        if (product == null)
            return NotFound(new { message = $"Product {id} not found" });

        _db.Products.Remove(product);
        await _db.SaveChangesAsync();

        return NoContent();
    }
}
```

---

## STEP 8: appsettings.json

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=ProductStoreDB;Trusted_Connection=True;"
  },
  "JwtSettings": {
    "SecretKey": "ThisIsMyVerySecretKey123!@#MustBe256Bits",
    "Issuer": "ProductStoreAPI",
    "Audience": "ProductStoreClients",
    "ExpiryMinutes": "60"
  },
  "Logging": {
    "LogLevel": { "Default": "Information" }
  },
  "AllowedHosts": "*"
}
```

---

## STEP 9: Program.cs (Complete)

```csharp
using System.Text;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.EntityFrameworkCore;
using Microsoft.IdentityModel.Tokens;
using Microsoft.OpenApi.Models;
using ProductStoreAPI.Data;
using ProductStoreAPI.Services;

var builder = WebApplication.CreateBuilder(args);

// ── 1. Database ───────────────────────────────────────────
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(
        builder.Configuration.GetConnectionString("DefaultConnection")));

// ── 2. Services ───────────────────────────────────────────
builder.Services.AddScoped<JwtService>();

// ── 3. Controllers + Validation ───────────────────────────
builder.Services.AddControllers();

// ── 4. JWT Authentication ─────────────────────────────────
var jwtSettings = builder.Configuration.GetSection("JwtSettings");
var secretKey   = jwtSettings["SecretKey"]!;

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer           = true,
            ValidateAudience         = true,
            ValidateLifetime         = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer              = jwtSettings["Issuer"],
            ValidAudience            = jwtSettings["Audience"],
            IssuerSigningKey         = new SymmetricSecurityKey(
                                           Encoding.UTF8.GetBytes(secretKey))
        };
    });

builder.Services.AddAuthorization();

// ── 5. Swagger with JWT Support ───────────────────────────
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new OpenApiInfo
    {
        Title       = "ProductStore API",
        Version     = "v1",
        Description = "A simple Product CRUD API with JWT Authentication"
    });

    // Add JWT button in Swagger UI
    c.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Name         = "Authorization",
        Type         = SecuritySchemeType.ApiKey,
        Scheme       = "Bearer",
        BearerFormat = "JWT",
        In           = ParameterLocation.Header,
        Description  = "Enter: Bearer {your JWT token here}"
    });

    c.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference
                {
                    Type = ReferenceType.SecurityScheme,
                    Id   = "Bearer"
                }
            },
            Array.Empty<string>()
        }
    });
});

var app = builder.Build();

// ── 6. Middleware Pipeline ────────────────────────────────
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI(c =>
    {
        c.SwaggerEndpoint("/swagger/v1/swagger.json", "ProductStore API v1");
        c.RoutePrefix = string.Empty;  // Swagger at root URL
    });
}

app.UseHttpsRedirection();
app.UseAuthentication();   // Must come before Authorization!
app.UseAuthorization();
app.MapControllers();

// ── 7. Auto-migrate on startup ────────────────────────────
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    db.Database.Migrate();
}

app.Run();
```

---

## STEP 10: Run Migrations & Start

```bash
# Create migration
dotnet ef migrations add InitialCreate

# Apply to database
dotnet ef database update

# Run the app
dotnet run

# Open browser at: https://localhost:{port}
# Swagger UI will be at the root
```

---

## API Quick Reference

| Method | URL | Auth | Description |
|--------|-----|------|-------------|
| GET | /api/products | None | List all products |
| GET | /api/products/{id} | None | Get one product |
| POST | /api/auth/login | None | Get JWT token |
| POST | /api/products | Admin JWT | Create product |
| PUT | /api/products/{id} | Admin JWT | Update product |
| DELETE | /api/products/{id} | Admin JWT | Delete product |

## Test Credentials (Demo only!)
- Admin: username=`admin`, password=`admin123`
- User: username=`user`, password=`user123`
