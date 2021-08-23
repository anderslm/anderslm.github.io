---
layout: post
title:  "Domain modelling with records in C# 9"
date:   2021-08-23 21:40:00 +0200
categories: C#
---
# Domain modelling with records in C# 9
In C# 9 a new shorthand syntax for creating records was introduced. 
It is called [positional record] and removes a lot of boilerplate when
creating new types.

Compared with how one would normally create a new type:

```c#
public class Product
{
    public string Name { get; set; }

    public decimal Price { get; set; }
}
```

this is now much better:

```c#
public record Product(string Name, decimal Price);
```

I won't go into the details of what the difference is between a `class` and a `record`,
but only focus on what this does for domain modelling.
## The domain
Consider creating a domain model of a product catalog in the utilities sector. In this
imaginary company they sell **electricity**, **internet** and **gas** and the pricing differs.
They all have a monthly fee, but:
- Gas and electricity has a basis usage cost per kWh plus the market price per kWh
- Gas has a price cap on the total usage cost per kWh

All the products are stored in a **database** and the market price is available through a third
party **http api**.

## Using class
When creating a **domain model** using `class`'s it's a common practice to create a new file for each new type.

It could look like this:

```
Products
|-- Product.cs
|-- GasProduct.cs
|-- ElectricityProduct.cs
|-- InternetProduct.cs
|-- MonthlyPrice.cs
|-- UsagePrice.cs
|-- CappedUsagePrice.cs
|-- IProductsRepository.cs
|-- IMarketPriceService.cs
```
In order to get an overview of all these `class`'s you would have to go into each file and remember
it's contents. 

Because of the syntactic overhead when working with `class`'s it doesn't help much
to put multiple `class`'s in a single file, as that would just make that file very big and full of 
boilerplate such as getters, setters and curly braces.

## Using record
It wouldn't make sense to adopt the 'one file per class' practice when working with records as a
`record` often fits on a single line. Instead of putting each new type in a file each, it's sufficient
to use a single file:
```
Products
|-- Model.cs
|-- IProductsRepository.cs
|-- IMarketPriceService.cs
```
Now this begs the question of the contents of `Model.cs`. 

Here it is:
```c#
namespace Products
{
    public abstract record Product(int Id, string Name, MonthlyPrice MonthlyPrice);
    
    public record GasProduct(int Id, string Name, UsagePrice MarketPrice, CappedUsagePrice UsagePrice) 
        : Product(Id, Name, new MonthlyPrice(100));
    
    public record ElectricityProduct(int Id, string Name, UsagePrice MarketPrice, UsagePrice UsagePrice)
        : Product(Id, Name, new MonthlyPrice(50));
    
    public record InternetProduct(int Id, string Name) 
        : Product(Id, Name, new MonthlyPrice(400)); 
    
    public record MonthlyPrice(decimal Price);
    
    public record UsagePrice(decimal Price);
    
    public record CappedUsagePrice(decimal Price, decimal PriceCap) : UsagePrice(Price);
}
```
Now this is much easier to get your head around than navigating between files and remembering the implementation.

## Size matters
The point is that size matters. Using `positional record`'s makes creating new types extremely lightweight,
and as such one tends to create more of them.

And we should create more types. The example above suffers from '[primitive obsession]' and
we could easily fix this:
```c#
public record ProductId(int Id);

public record ProductName(string Name);

public abstract record Product(ProductId Id, ProductName Name, MonthlyPrice MonthlyPrice);
    
public record GasProduct(ProductId Id, ProductName Name, UsagePrice MarketPrice, CappedUsagePrice UsagePrice) 
    : Product(Id, Name, new MonthlyPrice(100));

public record ElectricityProduct(ProductId Id, ProductName Name, UsagePrice MarketPrice, UsagePrice UsagePrice)
    : Product(Id, Name, new MonthlyPrice(50));

public record InternetProduct(ProductId Id, ProductName Name) 
    : Product(Id, Name, new MonthlyPrice(400)); 

public record Price(decimal dollars)

public record MonthlyPrice(Price Price);

public record UsagePrice(Price Price);

public record CappedUsagePrice(Price Price, Price PriceCap) : UsagePrice(Price);
```
Now you might disagree with the details of this model and that is the point. 
On just 20 lines we can read and reflect on the design of the model.

[primitive obsession]: https://refactoring.com/catalog/replacePrimitiveWithObject.html
[positional record]: https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/record#positional-syntax-for-property-definition
