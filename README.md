Aquí tienes la implementación completa de los archivos clave para tu solución `Ripley.Platform.u201621873.API`.

He organizado el código por **Bounded Contexts** y carpetas para que solo tengas que copiar y pegar. Recuerda reemplazar `u201621873` con tu código real de estudiante en los `namespaces`.

-----

### 1\. Shared Bounded Context (Núcleo Compartido)

**Archivo:** `Shared/Domain/Model/Entities/AuditableEntity.cs`
*Propósito: Clase base para CreatedAt y UpdatedAt.*

```csharp
namespace Ripley.Platform.u201621873.API.Shared.Domain.Model.Entities;

public class AuditableEntity
{
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
}
```

**Archivo:** `Shared/Infrastructure/Persistence/EFC/Configuration/Extensions/ModelBuilderExtensions.cs`
*Propósito: Convención Snake Case y Pluralización automática.*

```csharp
using Microsoft.EntityFrameworkCore;

namespace Ripley.Platform.u201621873.API.Shared.Infrastructure.Persistence.EFC.Configuration.Extensions;

public static class ModelBuilderExtensions
{
    public static void UseSnakeCaseNamingConvention(this ModelBuilder builder)
    {
        foreach (var entity in builder.Model.GetEntityTypes())
        {
            // Pluralizar tablas (simple append 's') y convertir a minúsculas
            entity.SetTableName(entity.GetTableName()!.ToLower() + "s");

            foreach (var property in entity.GetProperties())
            {
                // Convertir CamelCase a snake_case
                var name = property.Name;
                var snakeCaseName = string.Concat(name.Select((x, i) => i > 0 && char.IsUpper(x) ? "_" + x.ToString() : x.ToString())).ToLower();
                property.SetColumnName(snakeCaseName);
            }
        }
    }
}
```

-----

### 2\. Catalog Bounded Context (Productos)

**Archivo:** `Catalog/Domain/Model/Aggregates/Product.cs`

```csharp
using Ripley.Platform.u201621873.API.Shared.Domain.Model.Entities;

namespace Ripley.Platform.u201621873.API.Catalog.Domain.Model.Aggregates;

public class Product : AuditableEntity
{
    public int Id { get; private set; }
    public string Sku { get; private set; }
    public string Name { get; private set; }
    public decimal BasePrice { get; private set; }
    public int AvailableStock { get; private set; }

    // Constructor vacío para EF Core
    protected Product() { }

    public Product(int id, string sku, string name, decimal basePrice, int availableStock)
    {
        Id = id;
        Sku = sku;
        Name = name;
        BasePrice = basePrice;
        AvailableStock = availableStock;
    }

    public void ReduceStock(int quantity)
    {
        if (AvailableStock < quantity)
            throw new InvalidOperationException("Insufficient stock for product " + Sku);
        
        AvailableStock -= quantity;
    }
}
```

**Archivo:** `Catalog/Domain/Repositories/IProductRepository.cs`

```csharp
using Ripley.Platform.u201621873.API.Catalog.Domain.Model.Aggregates;

namespace Ripley.Platform.u201621873.API.Catalog.Domain.Repositories;

public interface IProductRepository
{
    Task<IEnumerable<Product>> ListAsync();
    Task<Product?> FindBySkuAsync(string sku);
    void Update(Product product); // Para bajar el stock
}
```

**Archivo:** `Catalog/Infrastructure/Persistence/EFC/Repositories/ProductRepository.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Ripley.Platform.u201621873.API.Catalog.Domain.Model.Aggregates;
using Ripley.Platform.u201621873.API.Catalog.Domain.Repositories;
using Ripley.Platform.u201621873.API.Shared.Infrastructure.Persistence.EFC.Configuration;

namespace Ripley.Platform.u201621873.API.Catalog.Infrastructure.Persistence.EFC.Repositories;

public class ProductRepository : IProductRepository
{
    private readonly AppDbContext _context;

    public ProductRepository(AppDbContext context)
    {
        _context = context;
    }

    public async Task<IEnumerable<Product>> ListAsync()
    {
        return await _context.Products.ToListAsync();
    }

    public async Task<Product?> FindBySkuAsync(string sku)
    {
        return await _context.Products.FirstOrDefaultAsync(p => p.Sku == sku);
    }

    public void Update(Product product)
    {
        _context.Products.Update(product);
    }
}
```

**Archivo:** `Catalog/Application/ACL/ProductContextFacade.cs`
*Propósito: Implementación de la ACL que usa Orders.*

```csharp
using Ripley.Platform.u201621873.API.Catalog.Domain.Repositories;
using Ripley.Platform.u201621873.API.Orders.Application.ACL; // Referencia a la interfaz en Orders

namespace Ripley.Platform.u201621873.API.Catalog.Application.ACL;

public class ProductContextFacade : IProductContextFacade
{
    private readonly IProductRepository _productRepository;

    public ProductContextFacade(IProductRepository productRepository)
    {
        _productRepository = productRepository;
    }

    public async Task<bool> ExistsProductWithStock(string sku, int quantity)
    {
        var product = await _productRepository.FindBySkuAsync(sku);
        if (product == null) return false;
        return product.AvailableStock >= quantity;
    }
    
    // Método extra para reducir stock desde Orders (via Evento o llamada directa si es Monolito simple)
    public async Task ReduceStock(string sku, int quantity)
    {
         var product = await _productRepository.FindBySkuAsync(sku);
         if (product != null)
         {
             product.ReduceStock(quantity);
             _productRepository.Update(product);
             // SaveChanges se llama en UnitOfWork o aquí si no hay UoW global
         }
    }
}
```

-----

### 3\. Orders Bounded Context (Pedidos)

**Archivo:** `Orders/Domain/Model/ValueObjects/Sku.cs`

```csharp
namespace Ripley.Platform.u201621873.API.Orders.Domain.Model.ValueObjects;

public record Sku(string Value)
{
    public Sku() : this(string.Empty) { }

    public static Sku Create(string value)
    {
        if (string.IsNullOrEmpty(value) || value.Length != 10)
            throw new ArgumentException("SKU must be exactly 10 characters (AAA-AAA-00).");
        return new Sku(value);
    }
}
```

**Archivo:** `Orders/Domain/Model/ValueObjects/Enums.cs`

```csharp
namespace Ripley.Platform.u201621873.API.Orders.Domain.Model.ValueObjects;

public enum EDeliveryMethod { InStorePickup = 0, HomeDelivery = 1 }
public enum EOrderPriority { Standard = 0, Express = 1 }
```

**Archivo:** `Orders/Domain/Model/Aggregates/PurchaseRequest.cs`

```csharp
using Ripley.Platform.u201621873.API.Orders.Domain.Model.ValueObjects;
using Ripley.Platform.u201621873.API.Shared.Domain.Model.Entities;

namespace Ripley.Platform.u201621873.API.Orders.Domain.Model.Aggregates;

public class PurchaseRequest : AuditableEntity
{
    public int Id { get; private set; }
    public string CustomerId { get; private set; }
    public Sku ProductSku { get; private set; }
    public int Quantity { get; private set; }
    public EDeliveryMethod DeliveryMethod { get; private set; }
    public EOrderPriority OrderPriority { get; private set; }
    public DateTime RequestedAt { get; private set; }

    protected PurchaseRequest() { }

    public PurchaseRequest(string customerId, Sku productSku, int quantity, EDeliveryMethod deliveryMethod, EOrderPriority orderPriority, DateTime requestedAt)
    {
        CustomerId = customerId;
        ProductSku = productSku;
        Quantity = quantity;
        DeliveryMethod = deliveryMethod;
        OrderPriority = orderPriority;
        RequestedAt = requestedAt;
    }
}
```

**Archivo:** `Orders/Application/ACL/IProductContextFacade.cs`
*Propósito: Contrato que Orders necesita que alguien cumpla.*

```csharp
namespace Ripley.Platform.u201621873.API.Orders.Application.ACL;

public interface IProductContextFacade
{
    Task<bool> ExistsProductWithStock(string sku, int quantity);
    Task ReduceStock(string sku, int quantity); // Simplificación para el examen
}
```

**Archivo:** `Orders/Application/Internal/CommandServices/PurchaseRequestCommandService.cs`
*Propósito: Lógica de negocio principal.*

```csharp
using Ripley.Platform.u201621873.API.Orders.Application.ACL;
using Ripley.Platform.u201621873.API.Orders.Domain.Model.Aggregates;
using Ripley.Platform.u201621873.API.Orders.Domain.Model.ValueObjects;
using Ripley.Platform.u201621873.API.Orders.Domain.Repositories;
using Ripley.Platform.u201621873.API.Orders.Interfaces.REST.Resources;
using Ripley.Platform.u201621873.API.Shared.Infrastructure.Persistence.EFC.Configuration; // Para SaveChanges

namespace Ripley.Platform.u201621873.API.Orders.Application.Internal.CommandServices;

public class PurchaseRequestCommandService
{
    private readonly IPurchaseRequestRepository _repository;
    private readonly IProductContextFacade _productFacade;
    private readonly AppDbContext _unitOfWork; // Acceso directo al contexto para guardar cambios

    public PurchaseRequestCommandService(IPurchaseRequestRepository repository, IProductContextFacade productFacade, AppDbContext unitOfWork)
    {
        _repository = repository;
        _productFacade = productFacade;
        _unitOfWork = unitOfWork;
    }

    public async Task<PurchaseRequest> Handle(CreatePurchaseRequestResource resource)
    {
        // 1. Validar Stock via ACL
        var hasStock = await _productFacade.ExistsProductWithStock(resource.ProductSku, resource.Quantity);
        if (!hasStock)
            throw new Exception("Product SKU not found or insufficient stock.");

        // 2. Parsear Fecha y Enums
        var requestedAt = DateTime.ParseExact(resource.RequestedAt, "yyyy-MM-dd HH:mm:ss", null);
        var deliveryMethod = Enum.Parse<EDeliveryMethod>(resource.DeliveryMethod.Replace(" ", ""), true);
        var orderPriority = Enum.Parse<EOrderPriority>(resource.OrderPriority, true);

        // 3. Crear Entidad
        var purchaseRequest = new PurchaseRequest(
            resource.CustomerId,
            Sku.Create(resource.ProductSku),
            resource.Quantity,
            deliveryMethod,
            orderPriority,
            requestedAt
        );

        // 4. Guardar
        await _repository.AddAsync(purchaseRequest);
        
        // 5. Reducir Stock (Simulación de Event Handler síncrono para el examen)
        await _productFacade.ReduceStock(resource.ProductSku, resource.Quantity);

        await _unitOfWork.SaveChangesAsync(); // Commit transacción

        return purchaseRequest;
    }
}
```

-----

### 4\. Infrastructure (Persistencia y Seeding)

**Archivo:** `Shared/Infrastructure/Persistence/EFC/Configuration/AppDbContext.cs`
*Propósito: Configuración maestra de EF Core.*

```csharp
using Microsoft.EntityFrameworkCore;
using Ripley.Platform.u201621873.API.Catalog.Domain.Model.Aggregates;
using Ripley.Platform.u201621873.API.Orders.Domain.Model.Aggregates;
using Ripley.Platform.u201621873.API.Shared.Infrastructure.Persistence.EFC.Configuration.Extensions;
using EntityFrameworkCore.CreatedUpdatedDate; // Paquete NuGet requerido

namespace Ripley.Platform.u201621873.API.Shared.Infrastructure.Persistence.EFC.Configuration;

public class AppDbContext : DbContext
{
    public DbSet<Product> Products { get; set; }
    public DbSet<PurchaseRequest> PurchaseRequests { get; set; }

    public AppDbContext(DbContextOptions options) : base(options) { }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        // Auditoría Automática
        optionsBuilder.AddCreatedUpdatedInterceptor();
        base.OnConfiguring(optionsBuilder);
    }

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);

        // CATALOG CONFIG
        builder.Entity<Product>().HasKey(p => p.Id);
        builder.Entity<Product>().Property(p => p.Id).ValueGeneratedOnAdd();
        builder.Entity<Product>().Property(p => p.Sku).IsRequired().HasMaxLength(10);
        builder.Entity<Product>().Property(p => p.Name).IsRequired().HasMaxLength(100);
        builder.Entity<Product>().Property(p => p.BasePrice).HasPrecision(10, 2);

        // ORDERS CONFIG
        builder.Entity<PurchaseRequest>().HasKey(p => p.Id);
        builder.Entity<PurchaseRequest>().Property(p => p.Id).ValueGeneratedOnAdd();
        builder.Entity<PurchaseRequest>().Property(p => p.CustomerId).IsRequired().HasMaxLength(8);
        
        // Owned Type Mapping
        builder.Entity<PurchaseRequest>().OwnsOne(p => p.ProductSku, sku =>
        {
            sku.Property(x => x.Value).HasColumnName("product_sku").IsRequired().HasMaxLength(10);
        });

        // DATA SEEDING (Productos Iniciales)
        builder.Entity<Product>().HasData(
            new { Id = 1, Sku = "RPL-TSH-01", Name = "Polo básico hombre", BasePrice = 39.9m, AvailableStock = 120, CreatedAt = DateTime.Now, UpdatedAt = DateTime.Now },
            new { Id = 2, Sku = "RPL-DRS-11", Name = "Vestido casual mujer", BasePrice = 129m, AvailableStock = 45, CreatedAt = DateTime.Now, UpdatedAt = DateTime.Now },
            new { Id = 3, Sku = "RPL-SHO-20", Name = "Zapatillas urbanas unisex", BasePrice = 199m, AvailableStock = 80, CreatedAt = DateTime.Now, UpdatedAt = DateTime.Now },
            new { Id = 4, Sku = "RPL-HME-02", Name = "Juego de sábanas 2 plazas", BasePrice = 89.9m, AvailableStock = 60, CreatedAt = DateTime.Now, UpdatedAt = DateTime.Now }
        );

        // Aplicar Snake Case
        builder.UseSnakeCaseNamingConvention();
    }
}
```

-----

### 5\. Configuración Principal (.NET 9)

**Archivo:** `Program.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Ripley.Platform.u201621873.API.Catalog.Application.ACL;
using Ripley.Platform.u201621873.API.Catalog.Domain.Repositories;
using Ripley.Platform.u201621873.API.Catalog.Infrastructure.Persistence.EFC.Repositories;
using Ripley.Platform.u201621873.API.Orders.Application.ACL;
using Ripley.Platform.u201621873.API.Orders.Application.Internal.CommandServices;
using Ripley.Platform.u201621873.API.Orders.Domain.Repositories;
using Ripley.Platform.u201621873.API.Orders.Infrastructure.Persistence.EFC.Repositories;
using Ripley.Platform.u201621873.API.Shared.Infrastructure.Persistence.EFC.Configuration;

var builder = WebApplication.CreateBuilder(args);

// 1. Configurar Servicios
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(); // OpenAPI

// 2. Base de Datos (MySQL)
var connectionString = builder.Configuration.GetConnectionString("DefaultConnection");
builder.Services.AddDbContext<AppDbContext>(options =>
{
    options.UseMySql(connectionString, ServerVersion.AutoDetect(connectionString))
           .LogTo(Console.WriteLine, LogLevel.Information)
           .EnableSensitiveDataLogging()
           .EnableDetailedErrors();
});

// 3. Inyección de Dependencias
// Repositorios
builder.Services.AddScoped<IProductRepository, ProductRepository>();
builder.Services.AddScoped<IPurchaseRequestRepository, PurchaseRequestRepository>();

// Servicios de Aplicación
builder.Services.AddScoped<PurchaseRequestCommandService>();

// ACL (Conectar Orders con Catalog)
builder.Services.AddScoped<IProductContextFacade, ProductContextFacade>();

var app = builder.Build();

// 4. Crear Base de Datos al iniciar (Migración automática)
using (var scope = app.Services.CreateScope())
{
    var services = scope.ServiceProvider;
    var context = services.GetRequiredService<AppDbContext>();
    context.Database.EnsureCreated(); // Esto ejecuta el Data Seeding si no existe la DB
}

// 5. Middleware Pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

**Archivo:** `appsettings.json`

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "server=localhost;database=ripley_db;user=root;password=tu_password"
  },
  "AllowedHosts": "*"
}
```

### Instrucciones Finales

1.  **Paquetes NuGet:** Instala estos paquetes obligatorios en tu proyecto `.API`:
      * `Microsoft.EntityFrameworkCore`
      * `Pomelo.EntityFrameworkCore.MySql`
      * `Microsoft.EntityFrameworkCore.Tools`
      * `Swashbuckle.AspNetCore`
      * `EntityFrameworkCore.CreatedUpdatedDate` (Obligatorio por rúbrica)
2.  **Compilación:** Ejecuta `dotnet build`.
3.  **Base de Datos:** Asegúrate de tener MySQL corriendo. Al ejecutar el proyecto (`dotnet run`), la línea `context.Database.EnsureCreated()` creará la tabla y meterá los datos de los productos automáticamente.