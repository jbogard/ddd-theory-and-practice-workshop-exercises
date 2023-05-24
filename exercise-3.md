
## Goals

The purpose of this exercise is to extend the newly added `Weight` attribute to the cart. Because of the distributed nature of this system, it's a bit trickier than it seems.

When modeling subdomains after business flows, it often comes with a side effect of "duplicating" data as the information flows from subdomain to subdomain. However, just because this data has the same value does not imply it has the same meaning.

### Step 1: Modify the CatalogItem models

The `CatalogItem` entity already contain the `Weight` field, but it's not exposed in the APIs used by Cart.

First, the model returned by the Backend-for-Frontend:

`src/ApiGateways/Web.Bff.Shopping/aggregator/Models/CatalogItem`:

```csharp
public decimal Weight { get; set; }
```

Next, the GRPC contract `CatalogItemResponse` for the Catalog service.

`src/Services/Catalog/Catalog.API/Proto/catalog.proto`, line 33:

```proto
double weight=15;
```

Save and build this project to have the C# contract code-generated.

### Step 2: Modify the Basket-related models

First, add a `Weight` property to the model returned by the Backend-for-Frontend:

`src/ApiGateways/Web.Bff.Shopping/aggregator/Models/BasketDataItem`:

```csharp
public decimal Weight { get; set; }
```

Next, add a `Weight` property to the model returned by the Basket API:

`src/Services/Basket/Basket.API/Model/BasketItem.cs`:

```csharp
public decimal Weight { get; set; }
```

Next, modify the GRPC contract for the Catalog service.

`src/Services/Basket/Basket.API/Proto/basket.proto`:

```proto
double weight = 8;
```

Build the solution so that the gRPC clients and contracts are regenerated with the new properties.

### Step 3: Record the Weight in the Basket

Now that the Weight property is on the public APIs, we can modify the cart logic to include the `Weight` property.

First, the aggregator in the BFF used when adding to cart:

`src/ApiGateways/Web.Bff.Shopping/aggregator/Services/CatalogService.cs`:

`MapToCatalogItemResponse` method:

```csharp
Weight = (decimal)catalogItemResponse.Weight,
```

And the backing GRPC service:

`src/Services/Catalog/Catalog.API/Grpc/CatalogService.cs`

`GetItemById` method:

```csharp
Weight = (double)item.Weight
```

`MapToResponse` method:

```csharp
Weight = (double)i.Weight
```


`src/ApiGateways/Web.Bff.Shopping/aggregator/Controllers/BasketController.cs` class:

In the `UpdateAllBasketAsync` method, around line 54:

```csharp
Weight = catalogItem.Weight,
```

And the `AddBasketItemAsync` method, around line 141:

```csharp
Weight = item.Weight,
```

And now for the Basket side, first with the aggregator service in the BFF:


`src/ApiGateways/Web.Bff.Shopping/aggregator/Services/BasketService.cs`:

`MapToCustomerBasketResponse` method, around line 53:

```csharp
Weight = (decimal)item.Weight,
```

`MapToCustomerBasket` method, around line 86:

```csharp
Weight = (double)item.Weight,
```

The backing GRPC service:

`src/Services/Basket/Basket.API/Grpc/BasketService.cs`:

`MapToCustomerBasketResponse` method:

```csharp
Weight = (double)item.Weight
```

`MapToCustomerBasket` method

```csharp
Weight = (decimal)item.Weight
```

### Step 4: Display the basket weight

First up, changing the ViewModel:


`src/Web/WebMVC/ViewModels/BasketItem.cs`:

```csharp
public decimal Weight { get; init; }
```

And adding some logic to calculate the total weight:

`src/Web/WebMVC/ViewModels/Basket.cs`

```csharp
public decimal Weight()
{
    return Math.Round(Items.Sum(x => x.Weight * x.Quantity), 1);
}
```

Finally, displaying the weight on the cart list:

`src/Web/WebMVC/Views/Shared/Components/CartList/Default.cshtml`

```html
<section class="esh-basket-title col-2">Weight</section>
```

```html
<section class="esh-basket-item esh-basket-item--middle esh-basket-item--mark col-2">@Math.Round(item.Quantity * item.Weight, 1).ToString("N1")g</section>
```

```html
<section class="esh-basket-title col-2">Weight</section>
```

```html
<section class="esh-basket-item esh-basket-item--mark col-2">@Model.Weight().ToString("N1") g</section>
```

### Step 5: Run the app

Run the app and after a brief startup pause, adding a few items to the cart should update the weight total for each line item and the weight for the entire cart.
