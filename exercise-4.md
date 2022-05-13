## Goals

In this exercise we will enforce the invariant that an order cannot be too heavy. Since there is not a "Shipping" or "Fulfillment" service, the "Order" service will have to suffice.

### Step 1: Modify OrderItem entity to keep track of weight

In order to enforce our invariant that heavy orders should be rejected, we first need to record the weight for items added.

First, we'll add the weight value to the `OrderItem` class, which requires the value in the constructor:

`/src/Services/Ordering/Ordering.Domain/AggregatesModel/OrderAggregate/OrderItem.cs`:

```csharp
public OrderItem(int productId, string productName, decimal unitPrice, decimal discount, string PictureUrl,
        decimal weight, int units = 1)
```

Create a private field for the weight:

```csharp
private readonly decimal _weight;
```

Set this value in the `OrderItem` constructor:

```csharp
_weight = weight;
```

Also add a method to expose this value:

```csharp
public decimal GetWeight() => _weight;
```

### Step 2: Modify Order aggregate to set weight value

Now that our `OrderItem` entity requires a weight, we'll need to include this value in our aggregate root's methods.

First, modify the `AddOrderItem` method to include a weight:

`src/Services/Ordering/Ordering.Domain/AggregatesModel/OrderAggregate/Order.cs`:

```csharp
public void AddOrderItem(int productId, string productName, decimal unitPrice, decimal discount, string pictureUrl,
        decimal weight, 
        int units = 1)
```

Next, pass this value through to the `OrderItem` constructor later in the method:

```csharp
var orderItem = new OrderItem(productId, productName, unitPrice, discount, pictureUrl, weight, units);
```

### Step 3: Modify commands to include weight

Orders are created through command objects, so we'll need to modify our command and handler to pass this value through to our aggregate.

First, we'll modify the command, in the `OrderItemDTO` class:

`src/Services/Ordering/Ordering.API/Application/Commands/CreateOrderCommand.cs`:

```csharp
public decimal Weight { get; init; }
```

Next, the handler:

`src/Services/Ordering/Ordering.API/Application/Commands/CreateOrderCommandHandler.cs`:

```csharp
order.AddOrderItem(item.ProductId, item.ProductName, item.UnitPrice, item.Discount, item.PictureUrl, item.Weight, item.Units);
```

We'll need to modify the Draft order handler as well:

`src/Services/Ordering/Ordering.API/Application/Commands/CreateOrderDraftCommandHandler.cs`:

```csharp
order.AddOrderItem(item.ProductId, item.ProductName, item.UnitPrice, item.Discount, item.PictureUrl, item.Weight, item.Units);
```

And finally building up the OrderDraftDTO at the end of the file in the `OrderDraftDTO.FromOrder` method:

```csharp
ProductName = oi.GetOrderItemProductName(),
Weight = oi.GetWeight()
```

### Step 4: Pass the weight from the Basket to our command

Now that we have our command handler ready to accept the weight, we need to pass this value from our basket. 

First, we need to modify the `BasketItem` contract in the `Ordering` service:

`src/Services/Ordering/Ordering.API/Application/Models/BasketItem.cs`

```csharp
public decimal Weight { get; init; }
```

Next, pass this value through in our method that build up this command DTO in the `ToOrderItemDTO` method:

`/src/Services/Ordering/Ordering.API/Extensions/BasketItemExtensions.cs`:

```csharp
Units = item.Quantity,
Weight = item.Weight
```

### Step 5: Fix unit tests

The unit tests use a builder pattern that hasn't been fixed for the new data, so let's do that. First, the builder:

`src/Services/Ordering/Ordering.UnitTests/Builders.cs`:

```csharp
public OrderBuilder AddOne(int productId,
         string productName,
         decimal unitPrice,
         decimal discount,
         string pictureUrl,
         decimal weight,
         int units = 1)
     {
        order.AddOrderItem(productId, productName, unitPrice, discount, pictureUrl, weight, units);
        return this;
     }
```

And finally our unit test:

`src/Services/Ordering/Ordering.UnitTests/Domain/OrderAggregateTest.cs`:

`when_add_two_times_on_the_same_item_then_the_total_of_order_should_be_the_sum_of_the_two_items` test:

```csharp
 var order = new OrderBuilder(address)
            .AddOne(1, "cup", 10.0m, 0, string.Empty, 5m)
            .AddOne(1, "cup", 10.0m, 0, string.Empty, 5m)
            .Build();
```

Other tests should be updated too, but we'll leave those out for now to get things compiling.


### Step 5: Enforce weight restriction

Now that our order has a weight, we can enforce a weight restriction when updating the shipping status (since there isn't an actual shipping service).

First, we'll need to calculate the weight of an order item:

`src/Services/Ordering/Ordering.Domain/AggregatesModel/OrderAggregate/OrderItem.cs`:

```csharp
public decimal GetTotalWeight()
{
    return _weight * _units;
}
```

Next, we'll need to calculate the total order weight:

`src/Services/Ordering/Ordering.Domain/AggregatesModel/OrderAggregate/Order.cs`:

```csharp
public decimal GetTotalWeight()
{
    return OrderItems.Sum(oi => oi.GetTotalWeight());
}
```

Finally, when we calculate shipping status, we can check the weight in the `SetShippedStatus` method:

```csharp
public void SetShippedStatus()
{
    if (GetTotalWeight() > 1000)
    {
        throw new OrderingDomainException("Too big.")
    }
```

Now we can run the application, try to add too many items with too much weight, and our order should not ship.