# Building 'eShop' from Zero to Hero: Adding Basket.API project

This sample scope of work is focus on adding the **Basket Service** in the **eShop** solution

We can review the architecture picture

![image](https://github.com/user-attachments/assets/bb063b3b-d46a-4d5b-ac67-9010d48c4589)

## 1. We Download the Solution from Github repo

The starting point for this sample is based on the following github repo:

https://github.com/luiscoco/eShop_Tutorial-Step7_Provision_AI_in_AppHost

## 2. We Rename the Solution

![image](https://github.com/user-attachments/assets/bac06caa-d08a-4c1f-98d2-33efb163c830)

## 3. We Add a New Project Basket.API

We right click on the Solution name and select the menu option **Add New Project**

![image](https://github.com/user-attachments/assets/e05cea1f-c0ed-4deb-a243-3db983cd9119)

We select the **ASP.NET Core gRPC Service** project template and press on the Next button

![image](https://github.com/user-attachments/assets/39eee51b-1704-4794-8bc6-d1a4fe88af2a)

We input the project name and location and press on the Next button

![image](https://github.com/user-attachments/assets/113b03dd-1d86-45dc-883f-209ef686db8f)

We select the **.NET 9** Framework and press the Create button

![image](https://github.com/user-attachments/assets/0f2a79d4-7fe4-43e7-a520-eeafdf7a3909)

## 4. We Load the Nuget packages and we create the GlobalUsings.cs file (Basket.API project)

![image](https://github.com/user-attachments/assets/59ce7bb9-8ea3-411d-a3f7-1e384594062a)

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

## 6. We Create the Folders structure (Basket.API project)

![image](https://github.com/user-attachments/assets/02c6f4aa-aada-4e31-a415-e4114e2d1e1b)

## 7. We Create the Data Model (Basket.API project)

We have to add two new files for defining the data model:

![image](https://github.com/user-attachments/assets/cec328e4-9750-473c-a7ca-8fbaa012ca66)

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

![image](https://github.com/user-attachments/assets/85697374-df0e-456c-9fa2-ca9d16561416)

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

![image](https://github.com/user-attachments/assets/7d328efa-8c97-41c9-be1d-df6bc5891d2a)

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

![image](https://github.com/user-attachments/assets/e2fc179b-98b4-475b-8144-2b0d755c7e34)

We have to right click on the **Proto** folder and add a new **basket.proto** file

![image](https://github.com/user-attachments/assets/e9f0fff0-184f-4e3e-b8f0-2492a7bdf715)

![image](https://github.com/user-attachments/assets/6a9cb154-c1ed-45ca-948d-910f00874af6)

We have to configure the Proto file properties

![image](https://github.com/user-attachments/assets/17e6cd5b-de8c-48db-9989-30dbce3f66b9)

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

![image](https://github.com/user-attachments/assets/ad7d1cb1-bd45-4f7e-a2e1-3b5de6c7e4b5)

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

![image](https://github.com/user-attachments/assets/731c9275-62ff-4286-849e-58b01f94cbf0)

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

![image](https://github.com/user-attachments/assets/22e5d264-e097-4ef6-bab0-cfcb2c01c79d)

We confirm we also include the **eShop.ServicesDefault** project as a new refernce in the **Basket.API** project

![image](https://github.com/user-attachments/assets/fdc0053b-f9d0-4004-9689-12f696f85e67)

We right click on the Basket.API project and Set As StartUp project

We **build** the **Basket.API** project for generating the Proto files

![image](https://github.com/user-attachments/assets/674b5326-f271-41c8-a70e-55d6eb919618)

We can also review the generated code

![image](https://github.com/user-attachments/assets/8dfff05a-4d63-41e8-944e-70d9a3b9fb4f)

## 12. We confirm the Basket.API project reference was included in the eShop.AppHost project

![image](https://github.com/user-attachments/assets/baa12a59-1959-4565-ae05-0aed7bd75a2e)

## 13. We Add the Nuget packages (eShop.AppHost project)

![image](https://github.com/user-attachments/assets/d6a77ab3-95d9-4f66-90fc-08aa0626f6b8)

**Aspire.Hosting.Redis**: this NuGet package provides extension methods and resource definitions for configuring a Redis resource within a .NET Aspire AppHost

This integration enables seamless setup and management of Redis instances in your distributed applications

## 14. We Modify the appsettings.json (Basket.API project)

In authentication or token-based systems (like OAuth or JWT), the **audience** often refers to the entity (service or application) that the token is intended for

The value "**basket**" could signify a specific service, such as an **e-commerce basket service** or API, indicating that this configuration is relevant for it

We modify the **appsettings.json** for adding this code:

```json
  "Identity": {
    "Audience": "basket"
  }
```

## 15. We Modify the Middleware (eShop.AppHost project)

We add the **Redis** service reference

```csharp
var redis = builder.AddRedis("redis");
```

We also have to add the **Basket.API** service registration in the middleware

```csharp
var basketApi = builder.AddProject<Projects.Basket_API>("basket-api")
    .WithReference(redis)
    .WithEnvironment("Identity__Url", identityEndpoint);
```

We add also the **Basket.API** reference in the **WebAp**

```csharp
var webApp = builder.AddProject<Projects.WebApp>("webapp", launchProfileName)
    .WithExternalHttpEndpoints()
    .WithReference(basketApi)
    .WithReference(catalogApi)
    .WithEnvironment("IdentityUrl", identityEndpoint);
```

## 16. We Add the Cart Icon (WebApp project)







## 17. We Add the CartMenu razor component in the HeaderBar razor component (WebApp project)







## 17. We Add the CartPage razor component (WebApp project)







## 18. We Modify the Extensions Middleware (WebApp project)







## 19. We Run the Application and verify the results






