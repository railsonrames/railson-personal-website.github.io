---
layout: post
title: OData in ASP.NET Core Simplifying the Creation of Flexible APIs
date: 2025-04-20 10:00:00
description: this is what included tabs in a post could look like
tags: dotnet core odata csharp sqlite api
categories: api
tabs: true
---

{% tabs post-content %}

{% tab post-content EN 🇺🇸 %}

In this article, I’ll walk you through how an API can leverage OData features such as pagination, sorting, and more — essential tools in many corporate scenarios. To demonstrate this in practice, I created a proof of concept with real code examples, which I’ll explain in detail throughout the post.

This tutorial will guide you through creating a simple ASP.NET Core Web API that uses SQLite for data storage and exposes product information using the Open Data Protocol (OData).

**Prerequisites:**

* .NET SDK 8.0 or later installed.
* Visual Studio Code (VS Code) or another suitable code editor.
* The NuGet Package Manager extension for VS Code (if using VS Code).
* Postman or a similar HTTP client for testing.

**Step 1: Create a New ASP.NET Core Web API Project**

1.  Open your terminal or command prompt.
2.  Navigate to the directory where you want to create your project.
3.  Run the following command to create a new Web API project:

    ```bash
    dotnet new webapi -n OdataExploreConcepts
    cd OdataExploreConcepts
    ```

**Step 2: Install Necessary NuGet Packages**

Open the integrated terminal in VS Code (Ctrl+\`) and run the following commands to add the required packages:

```bash
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
dotnet add package Microsoft.EntityFrameworkCore.Tools
dotnet add package Microsoft.AspNetCore.OData
```

* `Microsoft.EntityFrameworkCore.Sqlite`: Provides the SQLite database provider for Entity Framework Core.
* `Microsoft.EntityFrameworkCore.Tools`: Contains tools for EF Core, such as migrations.
* `Microsoft.AspNetCore.OData`: Enables OData support in ASP.NET Core.

**Step 3: Create the Product Model**

1.  Create a new folder named `Models` in your project.
2.  Inside the `Models` folder, create a new C# file named `Product.cs` with the following content:

    ```csharp
    namespace OdataExploreConcepts.Models
    {
        public class Product
        {
            public int Id { get; set; }
            public string Name { get; set; }
            public decimal Price { get; set; }
            public int Stock { get; set; }
        }
    }
    ```

**Step 4: Create the Data Context (ProductContext)**

1.  Create a new folder named `Data` in your project.
2.  Inside the `Data` folder, create a new C# file named `ProductContext.cs` with the following content:

    ```csharp
    using Microsoft.EntityFrameworkCore;
    using OdataExploreConcepts.Models;

    namespace OdataExploreConcepts.Data
    {
        public class ProductContext : DbContext
        {
            public ProductContext(DbContextOptions<ProductContext> options) : base(options)
            {
            }

            public DbSet<Product> Products { get; set; } = null!;

            protected override void OnModelCreating(ModelBuilder modelBuilder)
            {
                modelBuilder.Entity<Product>().HasKey(p => p.Id);
                modelBuilder.Entity<Product>().Property(p => p.Name).IsRequired();
                modelBuilder.Entity<Product>().Property(p => p.Price).HasColumnType("decimal(18,2)");
                modelBuilder.Entity<Product>().Property(p => p.Stock).IsRequired();
                modelBuilder.Entity<Product>().HasData(ProductSeedData.InitialProducts);
            }
        }
    }
    ```

**Step 5: Create Seed Data (ProductSeedData)**

1.  Inside the `Data` folder, create a new C# file named `ProductSeedData.cs` with the following content:

    ```csharp
    using OdataExploreConcepts.Models;

    namespace OdataExploreConcepts.Data
    {
        public static class ProductSeedData
        {
            public static Product[] InitialProducts = new Product[]
            {
                new Product { Id = 1, Name = "Produto 001", Price = 10.99m, Stock = 100 },
                new Product { Id = 2, Name = "Produto 002", Price = 20.50m, Stock = 50 },
                new Product { Id = 3, Name = "Produto 003", Price = 35.75m, Stock = 75 },
                new Product { Id = 4, Name = "Produto 004", Price = 55.20m, Stock = 200 },
                new Product { Id = 5, Name = "Produto 005", Price = 80.00m, Stock = 120 },
                new Product { Id = 6, Name = "Produto 006 Barato", Price = 5.99m, Stock = 300 },
                // Add more products as needed (up to 120 for testing)
                // Example for 120 products:
                // ... (loop to generate 114 more products with unique IDs, names, prices, and stocks)
            };
        }
    }
    ```

    *(For brevity, the full list of 120 products is not shown here. You can generate them programmatically or add them manually).*

**Step 6: Configure Database Connection and OData in `Program.cs`**

Open the `Program.cs` file and update its content as follows:

```csharp
using Microsoft.AspNetCore.OData;
using Microsoft.EntityFrameworkCore;
using Microsoft.OData.Edm;
using Microsoft.OData.ModelBuilder;
using OdataExploreConcepts.Data;
using OdataExploreConcepts.Models;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddControllers().AddOData(opt =>
    opt.AddRouteComponents("odata", GetEdmModel())
       .EnableQueryFeatures(maxTop: 100)); // Enable OData query features with a max top of 100

builder.Services.AddDbContext<ProductContext>(options =>
    options.UseSqlite(builder.Configuration.GetConnectionString("DefaultConnection")));

// Learn more about configuring Swagger/OpenAPI at https://aka.ms/aspnetcore/swashbuckle
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// Configure the HTTP request pipeline.
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
    app.UseODataRouteDebug(); // Optional: for debugging OData routes
}

app.UseHttpsRedirection();

app.UseAuthorization();

app.MapControllers();
app.MapODataRoute("odata", "odata", GetEdmModel());
app.UseODataQueryRequest(); // Enable OData query request processing

app.Run();

static IEdmModel GetEdmModel()
{
    var builder = new ODataConventionModelBuilder();
    builder.EntitySet<Product>("Products");
    builder.EntityType<Product>().Filter().OrderBy().Expand().Select().Count(); // Enable query options for Product
    return builder.GetEdmModel();
}
```

**Step 7: Create the OData Controller (ProductsController)**

1.  Create a new folder named `Controllers` in your project.
2.  Inside the `Controllers` folder, create a new C# file named `ProductsController.cs` with the following content:

    ```csharp
    using Microsoft.AspNetCore.Mvc;
    using Microsoft.AspNetCore.OData.Query;
    using Microsoft.AspNetCore.OData.Routing.Controllers;
    using Microsoft.EntityFrameworkCore;
    using OdataExploreConcepts.Data;
    using OdataExploreConcepts.Models;
    using System.Linq;
    using System.Threading.Tasks;

    public class ProductsController : ODataController
    {
        private readonly ProductContext _context;

        public ProductsController(ProductContext context)
        {
            _context = context;
        }

        [EnableQuery(MaxTop = 100)] // Apply MaxTop here as well
        [HttpGet]
        public IQueryable<Product> Get()
        {
            return _context.Products;
        }

        [EnableQuery]
        [HttpGet("{key}")]
        public async Task<IActionResult> Get([FromRoute] int key)
        {
            var product = await _context.Products.FindAsync(key);
            if (product == null)
            {
                return NotFound();
            }
            return Ok(product);
        }

        [HttpPost]
        public async Task<IActionResult> Post([FromBody] Product product)
        {
            if (!ModelState.IsValid)
            {
                return BadRequest(ModelState);
            }
            _context.Products.Add(product);
            await _context.SaveChangesAsync();
            return Created(product);
        }

        [HttpPut("{key}")]
        public async Task<IActionResult> Put([FromRoute] int key, [FromBody] Product update)
        {
            if (!ModelState.IsValid)
            {
                return BadRequest(ModelState);
            }
            var product = await _context.Products.FindAsync(key);
            if (product == null)
            {
                return NotFound();
            }
            update.Id = key;
            _context.Entry(product).CurrentValues.SetValues(update);
            try
            {
                await _context.SaveChangesAsync();
            }
            catch (DbUpdateConcurrencyException)
            {
                if (!ProductExists(key))
                {
                    return NotFound();
                }
                else
                {
                    throw;
                }
            }
            return NoContent();
        }

        [HttpDelete("{key}")]
        public async Task<IActionResult> Delete([FromRoute] int key)
        {
            var product = await _context.Products.FindAsync(key);
            if (product == null)
            {
                return NotFound();
            }
            _context.Products.Remove(product);
            await _context.SaveChangesAsync();
            return NoContent();
        }

        private bool ProductExists(int key)
        {
            return _context.Products.Any(e => e.Id == key);
        }
    }
    ```

**Step 8: Create and Apply Migrations**

1.  Open the integrated terminal in VS Code.
2.  Run the following commands to create and apply the initial migration:

    ```bash
    dotnet ef migrations add InitialCreate
    dotnet ef database update
    ```

    This will create the `products.db` file and the `Products` table with the seed data.

**Step 9: Testing OData Queries with Postman**

Run your application (`dotnet run`) and open Postman to test the following queries (assuming your application runs on `http://localhost:5000` or `https://localhost:5001`):

1.  **Get all products:**
    `GET http://localhost:<port>/odata/Products`

2.  **Filter by price greater than 50:**
    `GET http://localhost:<port>/odata/Products?$filter=Price gt 50`

3.  **Filter by name containing 'Produto 1':**
    `GET http://localhost:<port>/odata/Products?$filter=contains(Name,'Produto 1')`

4.  **Order by price descending:**
    `GET http://localhost:<port>/odata/Products?$orderby=Price desc`

5.  **Select only name and price:**
    `GET http://localhost:<port>/odata/Products?$select=Name,Price`

6.  **Get the first 10 products (pagination):**
    `GET http://localhost:<port>/odata/Products?$top=10`

7.  **Skip the first 5 and get the next 5 (pagination):**
    `GET http://localhost:<port>/odata/Products?$skip=5&$top=5`

8.  **Get the total count of products:**
    `GET http://localhost:<port>/odata/Products?$count=true`

By following these steps, you have created a basic ASP.NET Core Web API that leverages OData for flexible data querying against an SQLite database seeded with product information. This tutorial covers the essential aspects discussed in our conversation, providing a practical guide for building OData-enabled APIs.

{% endtab %}

{% tab post-content PT 🇧🇷 %}

Neste artigo, vou mostrar como uma API pode se beneficiar dos recursos do OData, como paginação, ordenação e muito mais — funcionalidades essenciais em diversos cenários corporativos. Para demonstrar isso na prática, criei uma prova de conceito com exemplos de código real, que explicarei em detalhes ao longo do post.

Este tutorial irá guiá-lo na criação de uma API Web ASP.NET Core simples que usa o SQLite para armazenamento de dados e expõe informações de produtos usando o Open Data Protocol (OData).

**Pré-requisitos:**

* .NET SDK 8.0 ou superior instalado.
* Visual Studio Code (VS Code) ou outro editor de código adequado.
* A extensão NuGet Package Manager para VS Code (se estiver usando VS Code).
* Postman ou um cliente HTTP similar para testes.

**Passo 1: Criar um Novo Projeto de API Web ASP.NET Core**

1.  Abra seu terminal ou prompt de comando.
2.  Navegue até o diretório onde você deseja criar seu projeto.
3.  Execute o seguinte comando para criar um novo projeto de API Web:

    ```bash
    dotnet new webapi -n OdataExploreConcepts
    cd OdataExploreConcepts
    ```

**Passo 2: Instalar os Pacotes NuGet Necessários**

Abra o terminal integrado no VS Code (Ctrl+\`) e execute os seguintes comandos para adicionar os pacotes necessários:

```bash
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
dotnet add package Microsoft.EntityFrameworkCore.Tools
dotnet add package Microsoft.AspNetCore.OData
```

* `Microsoft.EntityFrameworkCore.Sqlite`: Fornece o provedor de banco de dados SQLite para Entity Framework Core.
* `Microsoft.EntityFrameworkCore.Tools`: Contém ferramentas para o EF Core, como migrations.
* `Microsoft.AspNetCore.OData`: Habilita o suporte ao OData no ASP.NET Core.

**Passo 3: Criar o Modelo de Produto**

1.  Crie uma nova pasta chamada `Models` no seu projeto.
2.  Dentro da pasta `Models`, crie um novo arquivo C# chamado `Product.cs` com o seguinte conteúdo:

    ```csharp
    namespace OdataExploreConcepts.Models
    {
        public class Product
        {
            public int Id { get; set; }
            public string Name { get; set; }
            public decimal Price { get; set; }
            public int Stock { get; set; }
        }
    }
    ```

**Passo 4: Criar o Contexto de Dados (ProductContext)**

1.  Crie uma nova pasta chamada `Data` no seu projeto.
2.  Dentro da pasta `Data`, crie um novo arquivo C# chamado `ProductContext.cs` com o seguinte conteúdo:

    ```csharp
    using Microsoft.EntityFrameworkCore;
    using OdataExploreConcepts.Models;

    namespace OdataExploreConcepts.Data
    {
        public class ProductContext : DbContext
        {
            public ProductContext(DbContextOptions<ProductContext> options) : base(options)
            {
            }

            public DbSet<Product> Products { get; set; } = null!;

            protected override void OnModelCreating(ModelBuilder modelBuilder)
            {
                modelBuilder.Entity<Product>().HasKey(p => p.Id);
                modelBuilder.Entity<Product>().Property(p => p.Name).IsRequired();
                modelBuilder.Entity<Product>().Property(p => p.Price).HasColumnType("decimal(18,2)");
                modelBuilder.Entity<Product>().Property(p => p.Stock).IsRequired();
                modelBuilder.Entity<Product>().HasData(ProductSeedData.InitialProducts);
            }
        }
    }
    ```

**Passo 5: Criar os Dados de Seed (ProductSeedData)**

1.  Dentro da pasta `Data`, crie um novo arquivo C# chamado `ProductSeedData.cs` com o seguinte conteúdo:

    ```csharp
    using OdataExploreConcepts.Models;

    namespace OdataExploreConcepts.Data
    {
        public static class ProductSeedData
        {
            public static Product[] InitialProducts = new Product[]
            {
                new Product { Id = 1, Name = "Produto 001", Price = 10.99m, Stock = 100 },
                new Product { Id = 2, Name = "Produto 002", Price = 20.50m, Stock = 50 },
                new Product { Id = 3, Name = "Produto 003", Price = 35.75m, Stock = 75 },
                new Product { Id = 4, Name = "Produto 004", Price = 55.20m, Stock = 200 },
                new Product { Id = 5, Name = "Produto 005", Price = 80.00m, Stock = 120 },
                new Product { Id = 6, Name = "Produto 006 Barato", Price = 5.99m, Stock = 300 },
                // Adicione mais produtos conforme necessário (até 120 para testes)
                // Exemplo para 120 produtos:
                // ... (loop para gerar mais 114 produtos com IDs, nomes, preços e estoques únicos)
            };
        }
    }
    ```

    *(Por brevidade, a lista completa de 120 produtos não é mostrada aqui. Você pode gerá-los programaticamente ou adicioná-los manualmente).*

**Passo 6: Configurar a Conexão do Banco de Dados e o OData em `Program.cs`**

Abra o arquivo `Program.cs` e atualize seu conteúdo da seguinte forma:

```csharp
using Microsoft.AspNetCore.OData;
using Microsoft.EntityFrameworkCore;
using Microsoft.OData.Edm;
using Microsoft.OData.ModelBuilder;
using OdataExploreConcepts.Data;
using OdataExploreConcepts.Models;

var builder = WebApplication.CreateBuilder(args);

// Adicionar serviços ao contêiner.
builder.Services.AddControllers().AddOData(opt =>
    opt.AddRouteComponents("odata", GetEdmModel())
       .EnableQueryFeatures(maxTop: 100)); // Habilitar recursos de consulta OData com um max top de 100

builder.Services.AddDbContext<ProductContext>(options =>
    options.UseSqlite(builder.Configuration.GetConnectionString("DefaultConnection")));

// Aprenda mais sobre como configurar o Swagger/OpenAPI em https://aka.ms/aspnetcore/swashbuckle
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// Configure o pipeline de requisição HTTP.
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
    app.UseODataRouteDebug(); // Opcional: para depurar rotas OData
}

app.UseHttpsRedirection();

app.UseAuthorization();

app.MapControllers();
app.MapODataRoute("odata", "odata", GetEdmModel());
app.UseODataQueryRequest(); // Habilitar o processamento de requisições de consulta OData

app.Run();

static IEdmModel GetEdmModel()
{
    var builder = new ODataConventionModelBuilder();
    builder.EntitySet<Product>("Products");
    builder.EntityType<Product>().Filter().OrderBy().Expand().Select().Count(); // Habilitar opções de consulta para Product
    return builder.GetEdmModel();
}
```

**Passo 7: Criar o Controller OData (ProductsController)**

1.  Crie uma nova pasta chamada `Controllers` no seu projeto.
2.  Dentro da pasta `Controllers`, crie um novo arquivo C# chamado `ProductsController.cs` com o seguinte conteúdo:

    ```csharp
    using Microsoft.AspNetCore.Mvc;
    using Microsoft.AspNetCore.OData.Query;
    using Microsoft.AspNetCore.OData.Routing.Controllers;
    using Microsoft.EntityFrameworkCore;
    using OdataExploreConcepts.Data;
    using OdataExploreConcepts.Models;
    using System.Linq;
    using System.Threading.Tasks;

    public class ProductsController : ODataController
    {
        private readonly ProductContext _context;

        public ProductsController(ProductContext context)
        {
            _context = context;
        }

        [EnableQuery(MaxTop = 100)] // Aplicar MaxTop aqui também
        [HttpGet]
        public IQueryable<Product> Get()
        {
            return _context.Products;
        }

        [EnableQuery]
        [HttpGet("{key}")]
        public async Task<IActionResult> Get([FromRoute] int key)
        {
            var product = await _context.Products.FindAsync(key);
            if (product == null)
            {
                return NotFound();
            }
            return Ok(product);
        }

        [HttpPost]
        public async Task<IActionResult> Post([FromBody] Product product)
        {
            if (!ModelState.IsValid)
            {
                return BadRequest(ModelState);
            }
            _context.Products.Add(product);
            await _context.SaveChangesAsync();
            return Created(product);
        }

        [HttpPut("{key}")]
        public async Task<IActionResult> Put([FromRoute] int key, [FromBody] Product update)
        {
            if (!ModelState.IsValid)
            {
                return BadRequest(ModelState);
            }
            var product = await _context.Products.FindAsync(key);
            if (product == null)
            {
                return NotFound();
            }
            update.Id = key;
            _context.Entry(product).CurrentValues.SetValues(update);
            try
            {
                await _context.SaveChangesAsync();
            }
            catch (DbUpdateConcurrencyException)
            {
                if (!ProductExists(key))
                {
                    return NotFound();
                }
                else
                {
                    throw;
                }
            }
            return NoContent();
        }

        [HttpDelete("{key}")]
        public async Task<IActionResult> Delete([FromRoute] int key)
        {
            var product = await _context.Products.FindAsync(key);
            if (product == null)
            {
                return NotFound();
            }
            _context.Products.Remove(product);
            await _context.SaveChangesAsync();
            return NoContent();
        }

        private bool ProductExists(int key)
        {
            return _context.Products.Any(e => e.Id == key);
        }
    }
    ```

**Passo 8: Criar e Aplicar Migrações**

1.  Abra o terminal integrado no VS Code.
2.  Execute os seguintes comandos para criar e aplicar a migração inicial:

    ```bash
    dotnet ef migrations add InitialCreate
    dotnet ef database update
    ```

    Isso criará o arquivo `products.db` e a tabela `Products` com os dados de seed.

**Passo 9: Testar as Consultas OData com o Postman**

Execute sua aplicação (`dotnet run`) e abra o Postman para testar as seguintes consultas (assumindo que sua aplicação rode em `http://localhost:5000` ou `https://localhost:5001`):

1.  **Obter todos os produtos:**
    `GET http://localhost:<porta>/odata/Products`

2.  **Filtrar por preço maior que 50:**
    `GET http://localhost:<porta>/odata/Products?$filter=Price gt 50`

3.  **Filtrar por nome contendo 'Produto 1':**
    `GET http://localhost:<porta>/odata/Products?$filter=contains(Name,'Produto 1')`

4.  **Ordenar por preço decrescente:**
    `GET http://localhost:<porta>/odata/Products?$orderby=Price desc`

5.  **Selecionar apenas nome e preço:**
    `GET http://localhost:<porta>/odata/Products?$select=Name,Price`

6.  **Obter os 10 primeiros produtos (paginação):**
    `GET http://localhost:<porta>/odata/Products?$top=10`

7.  **Pular os 5 primeiros e obter os próximos 5 (paginação):**
    `GET http://localhost:<porta>/odata/Products?$skip=5&$top=5`

8.  **Obter a contagem total de produtos:**
    `GET http://localhost:<porta>/odata/Products?$count=true`

Seguindo estes passos, você criou uma API Web ASP.NET Core básica que utiliza o OData para consultas de dados flexíveis contra um banco de dados SQLite preenchido com informações de produtos. Este tutorial cobre os aspectos essenciais discutidos em nossa conversa, fornecendo um guia prático para construir APIs habilitadas para OData.

{% endtab %}

{% endtabs %}
