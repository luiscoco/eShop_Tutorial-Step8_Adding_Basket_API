# Building 'eShop' from Zero to Hero: Adding Basket.API project

This sample scope of work is focus on adding the **Basket Service** in the **eShop** solution

We can review the architecture picture

![image](https://github.com/user-attachments/assets/ad245c0d-2289-48eb-88db-ebc1208c673a)

## 1. We Download the Solution from Github repo

The starting point for this sample is based on the following github repo:

https://github.com/luiscoco/eShop_Tutorial-Step7_Provision_AI_in_AppHost

## 2. We Rename the Solution

![image](https://github.com/user-attachments/assets/523fed06-b2fa-44e9-b3cd-5ff10d67efd3)

## 3. We Add a New Project Basket.API

We right click on the Solution name and select the menu option **Add New Project**

![image](https://github.com/user-attachments/assets/b8a08d79-7d39-46de-94cb-0bada54246ef)

We select the **ASP.NET Core gRPC Service** project template and press on the Next button

![image](https://github.com/user-attachments/assets/0f43695b-7984-4d48-a49a-43905f310392)

We input the project name and location and press on the Next button

![image](https://github.com/user-attachments/assets/8e438b49-df5c-4431-87c8-472e66171bd4)

We select the **.NET 9** Framework and press the Create button

![image](https://github.com/user-attachments/assets/5c25c9a6-123a-49fb-9c67-ada87bb5c88a)

We review the Solution folders structure after adding the new API

![image](https://github.com/user-attachments/assets/ce8052ae-8a1e-4ad1-ac9e-d4e7f1e01224)

## 4. We Load the Nuget packages and we create the GlobalUsings.cs file (Basket.API project)

![image](https://github.com/user-attachments/assets/d5d1f833-c693-4461-9807-58166f169d5a)

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

## 5. We Create the Folders structure (Basket.API project)

![image](https://github.com/user-attachments/assets/9feb30bb-681f-4422-a526-b24589ee1430)

## 6. We Create the Data Model (Basket.API project)

We have to add two new files for defining the data model:

![image](https://github.com/user-attachments/assets/7ce8e9d4-99cb-409e-b805-e3f97873af4d)

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

## 7. We Create Redis Repository (Basket.API project)

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

![image](https://github.com/user-attachments/assets/c1adab5c-6e05-4b7a-97d7-0d811b383b9a)

![image](https://github.com/user-attachments/assets/f37fae43-c03f-4602-a2b1-89ca71abe3eb)

**RedisBasketRepository.cs**

```csharp
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

## 8. We Create the Service and Proto file (Basket.API project)

We first delete the **GreeterService.cs** file

![image](https://github.com/user-attachments/assets/e097cb36-f017-438a-a716-ad56c8035204)

We also delete the **greet.proto** file

![image](https://github.com/user-attachments/assets/1bfd2124-a236-4bf4-8782-0c9cf8e5a4ac)

We create a new file ****


## 9. We Define the Middleware and the Extensions files (Basket.API project)






## 10. We Modify the Middleware (eShop.AppHost project)






## 11. We Add the Basket Services files (WebApp project)







## 12. We Add the Cart Icon (WebApp project)







## 13. We Add the CartMenu razor component in the HeaderBar razor component (WebApp project)







## 14. We Add the CartPage razor component (WebApp project)







## 15. We Modify the Extensions Middleware (WebApp project)







## 16. We Run the Application and verify the results






























