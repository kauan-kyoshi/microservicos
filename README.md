# Microserviços

---

### **Plano de Implementação Detalhado**

#### **1. Estrutura Geral do Projeto (Solução no Visual Studio)**

Uma estrutura organizada é fundamental. Recomendo a seguinte disposição:

```
ECommerce.sln
|
├─── src
│    ├─── ApiGateways
│    │    └── OcelotApiGw/              # Projeto ASP.NET Core vazio com Ocelot
│    │
│    ├─── BuildingBlocks
│    │    └── EventBus.Messages/        # Biblioteca de classes com os eventos (contratos)
│    │
│    └─── Services
│         ├─── Identity.API/           # Microserviço para autenticação e geração de JWT
│         ├─── Product.API/            # Microserviço de Gestão de Estoque
│         └─── Sales.API/              # Microserviço de Gestão de Vendas
│
└─── docker-compose.yml                 # Arquivo para orquestrar os contêineres
```

**Por que um `Identity.API`?** Embora não explicitamente pedido, é uma boa prática separar a responsabilidade de autenticação em seu próprio serviço. Ele será simples: receberá credenciais e retornará um token JWT.

**Por que `EventBus.Messages`?** Para evitar acoplamento, os eventos que trafegam no RabbitMQ (como `PedidoRealizadoEvent`) devem estar em uma biblioteca compartilhada, servindo como um contrato entre o publicador (Sales.API) e o consumidor (Product.API).

---

### **2. Microserviço 1: Gestão de Estoque (Product.API)**

**Responsabilidades:** CRUD de produtos e atualização de estoque via RabbitMQ.

**a) Modelo (`Product.cs`) e DbContext:**

```csharp
// Product.API/Entities/Product.cs
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Description { get; set; }
    public decimal Price { get; set; }
    public int StockQuantity { get; set; }
}

// Product.API/Data/ProductDbContext.cs
public class ProductDbContext : DbContext
{
    public ProductDbContext(DbContextOptions<ProductDbContext> options) : base(options) { }
    public DbSet<Product> Products { get; set; }
}
```

**b) Controller (`ProductsController.cs`):**

Endpoints RESTful para gerenciar produtos.

```csharp
// Product.API/Controllers/ProductsController.cs
[ApiController]
[Route("api/[controller]")]
[Authorize] // Protege todos os endpoints
public class ProductsController : ControllerBase
{
    private readonly ProductDbContext _context;

    public ProductsController(ProductDbContext context)
    {
        _context = context;
    }

    [HttpGet]
    public async Task<IActionResult> GetProducts() 
    {
        return Ok(await _context.Products.ToListAsync());
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> GetProductById(int id)
    {
        var product = await _context.Products.FindAsync(id);
        if (product == null) return NotFound();
        // Verificação importante para o serviço de vendas
        return Ok(new { product.Id, product.StockQuantity });
    }

    [HttpPost]
    public async Task<IActionResult> CreateProduct([FromBody] Product product)
    {
        _context.Products.Add(product);
        await _context.SaveChangesAsync();
        return CreatedAtAction(nameof(GetProductById), new { id = product.Id }, product);
    }
}
```

**c) Consumidor de Eventos do RabbitMQ:**

Esta é a peça chave da comunicação assíncrona. Usaremos a biblioteca `MassTransit` para simplificar a integração com o RabbitMQ.

Primeiro, o evento no projeto `EventBus.Messages`:

```csharp
// EventBus.Messages/Events/OrderCreatedEvent.cs
public record OrderCreatedEvent
{
    public Guid OrderId { get; init; }
    public List<OrderItemMessage> OrderItems { get; init; }
}

public record OrderItemMessage
{
    public int ProductId { get; init; }
    public int Quantity { get; init; }
}
```

Agora, o consumidor no `Product.API`:

```csharp
// Product.API/EventBusConsumer/OrderCreatedConsumer.cs
using MassTransit;
using EventBus.Messages.Events;

public class OrderCreatedConsumer : IConsumer<OrderCreatedEvent>
{
    private readonly ProductDbContext _context;
    private readonly ILogger<OrderCreatedConsumer> _logger;

    public OrderCreatedConsumer(ProductDbContext context, ILogger<OrderCreatedConsumer> logger)
    {
        _context = context;
        _logger = logger;
    }

    public async Task Consume(ConsumeContext<OrderCreatedEvent> context)
    {
        var orderEvent = context.Message;
        _logger.LogInformation("Recebido evento OrderCreatedEvent para OrderId: {OrderId}", orderEvent.OrderId);

        foreach (var item in orderEvent.OrderItems)
        {
            var product = await _context.Products.FindAsync(item.ProductId);
            if (product != null && product.StockQuantity >= item.Quantity)
            {
                product.StockQuantity -= item.Quantity;
            }
            else
            {
                // Lógica de compensação (Saga Pattern) seria iniciada aqui
                _logger.LogWarning("Falha ao atualizar estoque para o produto {ProductId}. Estoque insuficiente.", item.ProductId);
                // Lançar exceção para o MassTransit tentar reprocessar ou mover para uma fila de erro.
                throw new InvalidOperationException("Estoque insuficiente ou produto não encontrado.");
            }
        }
        
        await _context.SaveChangesAsync();
        _logger.LogInformation("Estoque atualizado com sucesso para OrderId: {OrderId}", orderEvent.OrderId);
    }
}
```

**d) Configuração no `Program.cs` do `Product.API`:**

```csharp
// Adicionar MassTransit com RabbitMQ
builder.Services.AddMassTransit(config => {
    config.AddConsumer<OrderCreatedConsumer>();
    config.UsingRabbitMq((ctx, cfg) => {
        cfg.Host(builder.Configuration["EventBusSettings:HostAddress"]);
        cfg.ReceiveEndpoint("order-created-queue", c => {
            c.ConfigureConsumer<OrderCreatedConsumer>(ctx);
        });
    });
});

// Adicionar Autenticação JWT
builder.Services.AddAuthentication("Bearer")
    .AddJwtBearer(options =>
    {
        options.Authority = "http://identity.api"; // URL do serviço de identidade
        options.RequireHttpsMetadata = false;
        options.Audience = "product_api";
    });
```

---

### **3. Microserviço 2: Gestão de Vendas (Sales.API)**

**Responsabilidades:** Criar e consultar pedidos, validando o estoque de forma síncrona e publicando o evento de venda.

**a) Modelo (`Order.cs`) e DbContext:**

```csharp
// Sales.API/Entities/Order.cs
public class Order
{
    public Guid Id { get; set; }
    public DateTime OrderDate { get; set; }
    public string Status { get; set; } // "Processing", "Confirmed", "Cancelled"
    public List<OrderItem> OrderItems { get; set; }
}

public class OrderItem
{
    public int Id { get; set; }
    public int ProductId { get; set; }
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
}
```

**b) Controller (`OrdersController.cs`):**

```csharp
// Sales.API/Controllers/OrdersController.cs
[ApiController]
[Route("api/[controller]")]
[Authorize]
public class OrdersController : ControllerBase
{
    private readonly SalesDbContext _context;
    private readonly IHttpClientFactory _httpClientFactory;
    private readonly IPublishEndpoint _publishEndpoint; // MassTransit

    public OrdersController(SalesDbContext context, IHttpClientFactory httpClientFactory, IPublishEndpoint publishEndpoint)
    {
        _context = context;
        _httpClientFactory = httpClientFactory;
        _publishEndpoint = publishEndpoint;
    }

    [HttpPost]
    public async Task<IActionResult> CreateOrder([FromBody] CreateOrderDto createOrderDto)
    {
        // 1. Validação do Estoque (Comunicação Síncrona)
        var httpClient = _httpClientFactory.CreateClient("ProductAPI");
        foreach (var item in createOrderDto.OrderItems)
        {
            var response = await httpClient.GetAsync($"/api/products/{item.ProductId}");
            if (!response.IsSuccessStatusCode)
            {
                return BadRequest($"Produto {item.ProductId} não encontrado.");
            }
            var productInfo = await response.Content.ReadFromJsonAsync<ProductStockDto>();
            if (productInfo.StockQuantity < item.Quantity)
            {
                return BadRequest($"Estoque insuficiente para o produto {item.ProductId}.");
            }
        }

        // 2. Criar o Pedido
        var order = new Order { /* ... mapear do DTO ... */ };
        _context.Orders.Add(order);
        await _context.SaveChangesAsync();

        // 3. Publicar o evento para o RabbitMQ (Comunicação Assíncrona)
        var orderCreatedEvent = new OrderCreatedEvent
        {
            OrderId = order.Id,
            OrderItems = order.OrderItems.Select(oi => new OrderItemMessage { ProductId = oi.ProductId, Quantity = oi.Quantity }).ToList()
        };
        await _publishEndpoint.Publish(orderCreatedEvent);
        
        return CreatedAtAction(nameof(GetOrderById), new { id = order.Id }, order);
    }
    
    // ... outros endpoints como GetOrderById ...
}

// DTOs para comunicação
public record ProductStockDto(int Id, int StockQuantity);
```

**c) Configuração no `Program.cs` do `Sales.API`:**

```csharp
// Adicionar HttpClient para comunicação com Product.API
builder.Services.AddHttpClient("ProductAPI", client =>
{
    // A URL base deve ser o endereço do serviço, que pode vir da configuração
    // Em um ambiente de produção, seria o endereço do serviço no cluster (ex: http://product.api)
    client.BaseAddress = new Uri(builder.Configuration["ApiSettings:ProductApiUrl"]);
});

// Adicionar MassTransit (apenas para publicar)
builder.Services.AddMassTransit(config => {
    config.UsingRabbitMq((ctx, cfg) => {
        cfg.Host(builder.Configuration["EventBusSettings:HostAddress"]);
    });
});

// Adicionar Autenticação JWT (similar ao Product.API)
// ...
```

---

### **4. Serviço de Identidade (Identity.API)**

Um serviço simples para gerar tokens. Em um cenário real, usaria `IdentityServer` ou `ASP.NET Core Identity`.

```csharp
// Identity.API/Controllers/AuthController.cs
[ApiController]
[Route("api/[controller]")]
public class AuthController : ControllerBase
{
    private readonly IConfiguration _config;
    public AuthController(IConfiguration config) { _config = config; }

    [HttpPost("login")]
    public IActionResult Login([FromBody] UserLoginDto login)
    {
        // Lógica de validação de usuário (mockada aqui)
        if (login.Username == "user" && login.Password == "password")
        {
            var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_config["Jwt:Key"]));
            var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
            
            var claims = new[]
            {
                new Claim(JwtRegisteredClaimNames.Sub, login.Username),
                new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString())
            };

            var token = new JwtSecurityToken(
                issuer: _config["Jwt:Issuer"],
                audience: "api_gateway", // O token é para o gateway
                claims: claims,
                expires: DateTime.Now.AddMinutes(30),
                signingCredentials: creds
            );

            return Ok(new { token = new JwtSecurityTokenHandler().WriteToken(token) });
        }
        return Unauthorized();
    }
}
```

---

### **5. API Gateway (Ocelot)**

O ponto de entrada único. A configuração é feita no arquivo `ocelot.json`.

```json
// OcelotApiGw/ocelot.json
{
  "Routes": [
    // Rota para o serviço de Produtos
    {
      "DownstreamPathTemplate": "/api/products/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [ { "Host": "product.api", "Port": 80 } ],
      "UpstreamPathTemplate": "/product-api/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT" ],
      "AuthenticationOptions": {
        "AuthenticationProviderKey": "Bearer",
        "AllowedScopes": []
      }
    },
    // Rota para o serviço de Vendas
    {
      "DownstreamPathTemplate": "/api/orders/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [ { "Host": "sales.api", "Port": 80 } ],
      "UpstreamPathTemplate": "/sales-api/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST" ],
      "AuthenticationOptions": {
        "AuthenticationProviderKey": "Bearer",
        "AllowedScopes": []
      }
    },
    // Rota para o serviço de Identidade
    {
      "DownstreamPathTemplate": "/api/auth/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [ { "Host": "identity.api", "Port": 80 } ],
      "UpstreamPathTemplate": "/identity-api/{everything}",
      "UpstreamHttpMethod": [ "POST" ]
    }
  ],
  "GlobalConfiguration": {
    "BaseUrl": "http://localhost:5000"
  }
}
```

**Configuração no `Program.cs` do `OcelotApiGw`:**

```csharp
builder.Configuration.AddJsonFile("ocelot.json");
builder.Services.AddOcelot(builder.Configuration);

// Configuração do JWT para o Gateway validar o token
builder.Services.AddAuthentication("Bearer")
    .AddJwtBearer(options =>
    {
        options.Authority = "http://identity.api"; // Endereço do nosso serviço de identidade
        options.RequireHttpsMetadata = false;
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateAudience = false // O audience será validado nos serviços de destino
        };
    });

// ...
app.UseOcelot().Wait();
```

---

### **6. Orquestração com Docker Compose**

O `docker-compose.yml` na raiz da solução irá construir e executar todos os contêineres juntos.

```yaml
version: '3.8'

services:
  rabbitmq:
    image: rabbitmq:3-management-alpine
    container_name: rabbitmq
    ports:
      - "5672:5672"  # Porta para as aplicações
      - "15672:15672" # Interface de gerenciamento web

  productdb:
    image: mcr.microsoft.com/mssql/server:2019-latest
    # ... configuração do SQL Server

  salesdb:
    image: mcr.microsoft.com/mssql/server:2019-latest
    # ... configuração do SQL Server

  identity.api:
    image: ${DOCKER_REGISTRY-}identityapi
    build:
      context: .
      dockerfile: src/Services/Identity.API/Dockerfile
    ports:
      - "8000:80"

  product.api:
    image: ${DOCKER_REGISTRY-}productapi
    build:
      context: .
      dockerfile: src/Services/Product.API/Dockerfile
    ports:
      - "8001:80"
    depends_on:
      - rabbitmq
      - productdb

  sales.api:
    image: ${DOCKER_REGISTRY-}salesapi
    build:
      context: .
      dockerfile: src/Services/Sales.API/Dockerfile
    ports:
      - "8002:80"
    depends_on:
      - rabbitmq
      - salesdb

  ocelot.apigw:
    image: ${DOCKER_REGISTRY-}ocelotapigw
    build:
      context: .
      dockerfile: src/ApiGateways/OcelotApiGw/Dockerfile
    ports:
      - "5000:80"
    depends_on:
      - identity.api
      - product.api
      - sales.api
```

---

### **7. Extras: Testes e Logs**

*   **Testes Unitários:** Use `xUnit` ou `NUnit`. No `Sales.API`, você pode usar `Moq` para mockar o `IHttpClientFactory` e o `IPublishEndpoint`, garantindo que a lógica de validação de estoque e a publicação do evento são chamadas corretamente.
*   **Logs e Monitoramento:** Integre o `Serilog` em todos os microserviços. Configure-o para enriquecer os logs com um `CorrelationId`. Este ID pode ser gerado no API Gateway e propagado através dos cabeçalhos HTTP e das mensagens do RabbitMQ, permitindo rastrear uma requisição completa através de todos os serviços.

### **Conclusão**

Esta arquitetura cumpre todos os requisitos definidos. Ela estabelece uma clara separação de responsabilidades, utiliza comunicação síncrona para validações críticas (verificação de estoque) e comunicação assíncrona para operações que podem ser processadas em segundo plano (atualização de estoque), garantindo resiliência e escalabilidade. A segurança é centralizada no API Gateway com JWT, e a estrutura do projeto está preparada para crescer com a adição de novos microserviços.
