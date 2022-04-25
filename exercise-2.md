
## Goals

In this exercise, we will add a new attribute to our catalog items - weight - and display it

### Prerequisite - install .NET EF Global tools

From a command line, install the `dotnet-ef` global tool:

```
dotnet tool install --global dotnet-ef
```

### Step 1: Create migration for Weight

If you've already run the application, the Catalog database has existing data that we need to remove. Drop the database:


From Visual Studio:

- Navigate to Tools -> SQL Server Object Explorer
- Connect to .\5433, sa, Pass@word
- Drop the database

Similar method exists for SQL Server Management Studio if you have it installed.

Next, let's add the `Weight` property to our `CatalogItem` entity.

`src\Services\Catalog\Catalog.API\Model\CatalogItem.cs`:

```csharp
public decimal Weight { get; set; } 
```

Next, let's add the migration. From a command line at the repository root:

```
> cd src\Services\Catalog\Catalog.API

> dotnet ef migrations add AddCatalogItemWeight --context CatalogContext -o Infrastructure/CatalogMigrations
```

Our new migration will include this extra column.

### Step 2: Populate its data

We've already dropped the database, but we need to modify our seed data to include this extra value. Replace the contents of this file with below:

`src\Services\Catalog\Catalog.API\Setup\CatalogItems.csv`:

```
CatalogTypeName,CatalogBrandName,Description,Name,Price,PictureFileName,availablestock,onreorder,weight
T-Shirt,.NET,".NET Bot Black Hoodie, and more",.NET Bot Black Hoodie,19.5,1.png,100,false,10.2
Mug,.NET,.NET Black & White Mug,.NET Black & White Mug,8.50,2.png,89,true,20.5
T-Shirt,Other,Prism White T-Shirt,Prism White T-Shirt,12,3.png,56,false,11.1
T-Shirt,.NET,.NET Foundation T-shirt,.NET Foundation T-shirt,12,4.png,120,false,11.3
Pin,Other,Roslyn Red Pin,Roslyn Red Pin,8.5,5.png,55,false,15.0
T-Shirt,.NET,.NET Blue Hoodie,.NET Blue Hoodie,12,6.png,17,false,10.2
T-Shirt,Other,Roslyn Red T-Shirt,Roslyn Red T-Shirt,12,7.png,8,false,11.6
T-Shirt,Other,Kudu Purple Hoodie,Kudu Purple Hoodie,8.5,8.png,34,false,12.8
Mug,Other,Cup<T> White Mug,Cup<T> White Mug,12,9.png,76,false,21.7
Pin,.NET,.NET Foundation Pin,.NET Foundation Pin,12,10.png,11,false,32.8
Pin,.NET,Cup<T> Pin,Cup<T> Pin,8.5,11.png,3,false,30.1
T-Shirt,Other,Prism White TShirt,Prism White TShirt,12,12.png,0,false,13.7
Mug,.NET,Modern .NET Black & White Mug,Modern .NET Black & White Mug,8.50,13.png,89,true,12.2
Mug,Other,Modern Cup<T> White Mug,Modern Cup<T> White Mug,12,14.png,76,false,12.6
```

Next, modify the logic that reads this file.

`src\Services\Catalog\Catalog.API\Infrastructure\CatalogContextSeed.cs`,  CreateCatalogItem method line 215:

```csharp
string weightString = column[Array.IndexOf(headers, "weight")].Trim('"').Trim();
if (!Decimal.TryParse(weightString, NumberStyles.AllowDecimalPoint, CultureInfo.InvariantCulture, out Decimal weight))
{
    throw new Exception($"weight={weightString}is not a valid decimal number");
}

var catalogItem = new CatalogItem()
{
    CatalogTypeId = catalogTypeIdLookup[catalogTypeName],
    CatalogBrandId = catalogBrandIdLookup[catalogBrandName],
    Description = column[Array.IndexOf(headers, "description")].Trim('"').Trim(),
    Name = column[Array.IndexOf(headers, "name")].Trim('"').Trim(),
    Price = price,
    PictureFileName = column[Array.IndexOf(headers, "picturefilename")].Trim('"').Trim(),
    Weight = weight
};
```

Next time we run the application, the weight value will be populated.

### Step 3: Display the new value

We'll need to modify the ViewModel first:

`src\Web\WebMVC\ViewModels\CatalogItem.cs`:

```csharp
public decimal Weight { get; init; }
```

And finally the View:

`src\Web\WebMVC\Views\Catalog\_product.cshtml`:

```html
<div class="esh-catalog-weight">
    <span>@Model.Weight.ToString("N1")</span>
</div>
```

Finally, styling (don't judge):

`src\Web\WebMVC\wwwroot\css\catalog\catalog.component.css`

```css
.esh-catalog-weight {
  font-size: 24px;
  font-weight: 900;
  text-align: center;
}

.esh-catalog-weight::after {
  content: 'g';
}

```

### Step 4: Run the app

Run the app and after a brief startup pause, we should see the catalog item's Weight show up in the list of products.
