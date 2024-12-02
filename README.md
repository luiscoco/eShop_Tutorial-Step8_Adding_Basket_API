# Building 'eShop' from Zero to Hero: Adding Basket.API project

This sample scope of work is focus on adding the **Basket Service** in the **eShop** solution

We can review the architecture picture


## 1. We Download the Solution from Github repo

The starting point for this sample is based on the following github repo:

https://github.com/luiscoco/eShop_Tutorial-Step7_Provision_AI_in_AppHost

## 2. We Rename the Solution



## 3. We Add a New Project Basket.API

We right click on the Solution name and select the menu option **Add New Project**



We select the **ASP.NET Core gRPC Service** project template and press on the Next button



We input the project name and location and press on the Next button



We select the **.NET 9** Framework and press the Create button



We review the Solution folders structure after adding the new API



## 4. We Load the Nuget packages and we create the GlobalUsings.cs file (Basket.API project)



**Aspire.StackExchange.Redis**: This package is typically an extension or a customization of the popular StackExchange.Redis library, which is widely used for working with Redis, a high-performance in-memory data store

Use Cases:

Provides advanced features or abstractions for integrating Redis into .NET applications

Likely tailored for scenarios requiring distributed caching, pub/sub messaging, or session storage with Redis

May include additional features like connection management, serialization helpers, or specific configurations for easier integration with Redis in Aspire-related projects

Dependencies: It builds upon StackExchange.Redis, which is the primary library for Redis in .NET

**Grpc.AspNetCore**: This package provides tools for building gRPC services in ASP.NET Core applications

Use Cases:

Enables server and client implementations of gRPC (Google Remote Procedure Call), which is a high-performance, open-source, and language-neutral framework for remote procedure calls

Facilitates communication between microservices in distributed systems

Provides support for HTTP/2, which is essential for gRPC communication

Includes middleware and tools for defining, hosting, and consuming gRPC services in ASP.NET Core

Dependencies: It integrates tightly with the ASP.NET Core pipeline and leverages the Protobuf serialization format for defining service contracts and data exchange

## 5. We Delete Protos and Services folders



After deleting the folders we have this project structure



## 6. We Create the Folders structure (Basket.API project)



## 7. We Create the Data Model (Basket.API project)

We have to add two new files for defining the data model:



**BasketItem** represents an item in a shopping basket, including properties to describe the product, its pricing, quantity, and picture URL

It implements the **IValidatableObject** interface to include custom validation logic

This class is be used in an **e-commerce** system's basket management component

Before processing or persisting **BasketItem** objects, **validation** ensures that they meet the required **business rules** (e.g., Quantity must be positive)

If a user attempts to add a basket item with a Quantity of 0, the Validate method would catch the issue and return a validation error: "Invalid number of units"

This helps maintain consistent and valid data across the application

**BasketItem.cs**

```csharp
using System.ComponentModel.DataAnnotations;

namespace eShop.Basket.API.Model;

public class BasketItem : IValidatableObject
{
    public string Id { get; set; }
    public int ProductId { get; set; }
    public string ProductName { get; set; }
    public decimal UnitPrice { get; set; }
    public decimal OldUnitPrice { get; set; }
    public int Quantity { get; set; }
    public string PictureUrl { get; set; }

    public IEnumerable<ValidationResult> Validate(ValidationContext validationContext)
    {
        var results = new List<ValidationResult>();

        if (Quantity < 1)
        {
            results.Add(new ValidationResult("Invalid number of units", new[] { "Quantity" }));
        }

        return results;
    }
}
```

The **CustomerBasket** class represents a shopping basket associated with a specific customer

It contains the customer ID and a collection of items in the basket

**CustomerBasket.cs**

```csharp
namespace eShop.Basket.API.Model;

public class CustomerBasket
{
    public string BuyerId { get; set; }

    public List<BasketItem> Items { get; set; } = [];

    public CustomerBasket() { }

    public CustomerBasket(string customerId)
    {
        BuyerId = customerId;
    }
}
```

## 8. We Create Redis Repository (Basket.API project)

We have to add two files for defining the Repository:



The interface outlines the contract for a repository that **manages** operations related to **customer baskets** (shopping carts)

**IBasketRepository.cs**

```csharp
using eShop.Basket.API.Model;

namespace eShop.Basket.API.Repositories;

public interface IBasketRepository
{
    Task<CustomerBasket> GetBasketAsync(string customerId);
    Task<CustomerBasket> UpdateBasketAsync(CustomerBasket basket);
    Task<bool> DeleteBasketAsync(string id);
}
```

This C# code represents a repository implementation for **managing customer baskets** in a **Redis**-backed e-commerce application

This implementation is an efficient and scalable way to manage user-specific data in Redis, leveraging modern C# features like source generators and memory-efficient APIs for serialization and deserialization

**Redis** is chosen for storing baskets in e-commerce applications because it offers unmatched speed, simplicity, and scalability for handling ephemeral, high-traffic data. These qualities align well with the dynamic nature of shopping baskets, ensuring users experience real-time responsiveness and reliability



**RedisBasketRepository.cs**

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;
using eShop.Basket.API.Model;
using StackExchange.Redis;

namespace eShop.Basket.API.Repositories;

public class RedisBasketRepository(ILogger<RedisBasketRepository> logger, IConnectionMultiplexer redis) : IBasketRepository
{
    private readonly IDatabase _database = redis.GetDatabase();

    // implementation:

    // - /basket/{id} "string" per unique basket
    private static RedisKey BasketKeyPrefix = "/basket/"u8.ToArray();
    // note on UTF8 here: library limitation (to be fixed) - prefixes are more efficient as blobs

    private static RedisKey GetBasketKey(string userId) => BasketKeyPrefix.Append(userId);

    public async Task<bool> DeleteBasketAsync(string id)
    {
        return await _database.KeyDeleteAsync(GetBasketKey(id));
    }

    public async Task<CustomerBasket> GetBasketAsync(string customerId)
    {
        using var data = await _database.StringGetLeaseAsync(GetBasketKey(customerId));

        if (data is null || data.Length == 0)
        {
            return null;
        }
        return JsonSerializer.Deserialize(data.Span, BasketSerializationContext.Default.CustomerBasket);
    }

    public async Task<CustomerBasket> UpdateBasketAsync(CustomerBasket basket)
    {
        var json = JsonSerializer.SerializeToUtf8Bytes(basket, BasketSerializationContext.Default.CustomerBasket);
        var created = await _database.StringSetAsync(GetBasketKey(basket.BuyerId), json);

        if (!created)
        {
            logger.LogInformation("Problem occurred persisting the item.");
            return null;
        }


        logger.LogInformation("Basket item persisted successfully.");
        return await GetBasketAsync(basket.BuyerId);
    }
}

[JsonSerializable(typeof(CustomerBasket))]
[JsonSourceGenerationOptions(PropertyNameCaseInsensitive = true)]
public partial class BasketSerializationContext : JsonSerializerContext
{

}
```

## 9. We Create the Service and Proto file (Basket.API project)

We create a new Service file **BasketService.cs**



We review the **BasketService.cs**

```csharp
using System.Diagnostics.CodeAnalysis;
using eShop.Basket.API.Repositories;
using eShop.Basket.API.Extensions;
using eShop.Basket.API.Model;
using Microsoft.AspNetCore.Authorization;
using Grpc.Core;

namespace eShop.Basket.API.Grpc;

public class BasketService(
    IBasketRepository repository,
    ILogger<BasketService> logger) : Basket.BasketBase
{
    [AllowAnonymous]
    public override async Task<CustomerBasketResponse> GetBasket(GetBasketRequest request, ServerCallContext context)
    {
        var userId = context.GetUserIdentity();
        if (string.IsNullOrEmpty(userId))
        {
            return new();
        }

        if (logger.IsEnabled(LogLevel.Debug))
        {
            logger.LogDebug("Begin GetBasketById call from method {Method} for basket id {Id}", context.Method, userId);
        }

        var data = await repository.GetBasketAsync(userId);

        if (data is not null)
        {
            return MapToCustomerBasketResponse(data);
        }

        return new();
    }

    public override async Task<CustomerBasketResponse> UpdateBasket(UpdateBasketRequest request, ServerCallContext context)
    {
        var userId = context.GetUserIdentity();
        if (string.IsNullOrEmpty(userId))
        {
            ThrowNotAuthenticated();
        }

        if (logger.IsEnabled(LogLevel.Debug))
        {
            logger.LogDebug("Begin UpdateBasket call from method {Method} for basket id {Id}", context.Method, userId);
        }

        var customerBasket = MapToCustomerBasket(userId, request);
        var response = await repository.UpdateBasketAsync(customerBasket);
        if (response is null)
        {
            ThrowBasketDoesNotExist(userId);
        }

        return MapToCustomerBasketResponse(response);
    }

    public override async Task<DeleteBasketResponse> DeleteBasket(DeleteBasketRequest request, ServerCallContext context)
    {
        var userId = context.GetUserIdentity();
        if (string.IsNullOrEmpty(userId))
        {
            ThrowNotAuthenticated();
        }

        await repository.DeleteBasketAsync(userId);
        return new();
    }

    [DoesNotReturn]
    private static void ThrowNotAuthenticated() => throw new RpcException(new Status(StatusCode.Unauthenticated, "The caller is not authenticated."));

    [DoesNotReturn]
    private static void ThrowBasketDoesNotExist(string userId) => throw new RpcException(new Status(StatusCode.NotFound, $"Basket with buyer id {userId} does not exist"));

    private static CustomerBasketResponse MapToCustomerBasketResponse(CustomerBasket customerBasket)
    {
        var response = new CustomerBasketResponse();

        foreach (var item in customerBasket.Items)
        {
            response.Items.Add(new BasketItem()
            {
                ProductId = item.ProductId,
                Quantity = item.Quantity,
            });
        }

        return response;
    }

    private static CustomerBasket MapToCustomerBasket(string userId, UpdateBasketRequest customerBasketRequest)
    {
        var response = new CustomerBasket
        {
            BuyerId = userId
        };

        foreach (var item in customerBasketRequest.Items)
        {
            response.Items.Add(new()
            {
                ProductId = item.ProductId,
                Quantity = item.Quantity,
            });
        }

        return response;
    }
}
```

We also add the proto file **basket.proto**



We have to right click on the **Proto** folder and add a new **basket.proto** file



We also review the **basket.proto** source code:

```csharp
syntax = "proto3";

option csharp_namespace = "eShop.Basket.API.Grpc";

package BasketApi;

service Basket {
    rpc GetBasket(GetBasketRequest) returns (CustomerBasketResponse) {}
    rpc UpdateBasket(UpdateBasketRequest) returns (CustomerBasketResponse) {}
    rpc DeleteBasket(DeleteBasketRequest) returns (DeleteBasketResponse) {}
}

message GetBasketRequest {
}

message CustomerBasketResponse {
    repeated BasketItem items = 1;
}

message BasketItem {
    int32 product_id = 2;
    int32 quantity = 6;
}

message UpdateBasketRequest {
    repeated BasketItem items = 2;
}

message DeleteBasketRequest {
}

message DeleteBasketResponse {
}
```

## 10. We Define the Middleware and the Extensions files (Basket.API project)

We have to add the Extensions files: **Extensions.cs** and **ServerCallContextIdentityExtensions.cs**



We review the  **Extensions.cs** source code:

```csharp
using System.Text.Json.Serialization;
using eShop.Basket.API.Repositories;
using eShop.ServiceDefaults;

namespace eShop.Basket.API.Extensions;

public static class Extensions
{
    public static void AddApplicationServices(this IHostApplicationBuilder builder)
    {
        builder.AddDefaultAuthentication();

        builder.AddRedisClient("redis");

        builder.Services.AddSingleton<IBasketRepository, RedisBasketRepository>();
    }
}
```

We also review the **ServerCallContextIdentityExtensions.cs** code:

```csharp
#nullable enable
using Grpc.Core;
namespace eShop.Basket.API.Extensions;

internal static class ServerCallContextIdentityExtensions
{
    public static string? GetUserIdentity(this ServerCallContext context) => context.GetHttpContext().User.FindFirst("sub")?.Value;
    public static string? GetUserName(this ServerCallContext context) => context.GetHttpContext().User.FindFirst(x => x.Type == ClaimTypes.Name)?.Value;
}
```

We also define the middleware

**Program.cs**

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.AddServiceDefaults();

builder.AddApplicationServices();

builder.Services.AddGrpc();

var app = builder.Build();

app.MapDefaultEndpoints();

app.MapGrpcService<BasketService>();

app.Run();
```

## 11. We add orchestrator support in the Basket.API project

We right click on the Basket.API project and we select the menu option **Add .NET Aspire Orchestrator Support...**



We configure the Proto file



We confirm we also include the **eShop.ServicesDefault** project as a new refernce in the **Basket.API** project



We right click on the Basket.API project and Set As StartUp project

We buidl the Basket.API project for generating the Proto files



We can also review the generated code



## 12. We Modify the Middleware (eShop.AppHost project)






## 13. We Add the Basket Services files (WebApp project)







## 14. We Add the Cart Icon (WebApp project)







## 15. We Add the CartMenu razor component in the HeaderBar razor component (WebApp project)







## 16. We Add the CartPage razor component (WebApp project)







## 17. We Modify the Extensions Middleware (WebApp project)







## 18. We Run the Application and verify the results






























