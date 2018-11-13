---
title: What is new in EF Core 2.2 - EF Core
author: divega
ms.date: 11/14/2018
ms.assetid: 998C04F3-676A-4FCF-8450-CFB0457B4198
uid: core/what-is-new/ef-core-2.2
---

# New features in EF Core 2.2

## Spatial data support

Spatial data can be used to represent the physical location and the shape of objects.
Many databases provide support for this type of data so it can be indexed and queried alongside other data.
Common scenarios include querying for objects within a given distance from a location, or selecting the object whose border contains a given location.
EF Core 2.2 supports mapping spatial data in multiple database to types defined by the [NetTopologySuite](https://github.com/NetTopologySuite/NetTopologySuite) (NTS) spatial library.

In order to take advantage of this new feature, you install a provider-specific extension pacakge that contributes the necessary type mappings and translations for common methods in spatial types.
Such provider extensions are now available for [SQL Server](https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.SqlServer.NetTopologySuite/), [SQLite](https://www.nuget.org/packages/Microsoft.EntityFrameworkCore.Sqlite.NetTopologySuite/), and [PostgreSQL](https://www.nuget.org/packages/Npgsql.EntityFrameworkCore.PostgreSQL.NetTopologySuite/) (from the [Npgsql project](http://www.npgsql.org/)).
Spatial types can be used directly with the [EF Core in-memory provider](https://docs.microsoft.com/en-us/ef/core/providers/in-memory/) without additional extensions.

Once you have installed the provider extension, you can add properties of supported types to your entities, e.g.:

``` csharp
using NetTopologySuite.Geometries;

namespace MyApp
{
  public class Friend
  {
    [Key]
    public string Name { get; set; }
  
    [Required]
    public Point Location { get; set; }
  }
}
``` 

You can then persist entities containing spatial data:

``` csharp
using (var context = new MyDbContext())
{
    context.Add(
        new Friend
        {
            Name = "Bill",
            Location = new Point(-122.34877, 47.6233355) {SRID = 4326 }
        });
    context.SaveChanges();
}
```
And you can execute database queries based on spatial data and operations:

``` csharp
  var nearestFriends =
      (from f in context.Friends
      orderby f.Location.Distance(myLocation) descending
      select f).Take(5).ToList();
```

For more information on this feature, see the [spatial types documentation](xref:core/tbd/spatial). 

## Collections of owned entities

EF Core 2.2 extends the ability to express ownership relationships to one-to-many associations.
This helps constraining how entities in an owned collection can be manipulated (for example, they cannot be tracked without an owner) and triggers automatic behaviors such as implicit eager loading.
In relational databases, owned collections are mapped to separate tables from the owner, just like regular one-to-many associations.
But in document-oriented databases, we plan to nest owned entities (in owned collections or references) within the same document as the owner.

You can use the feature by invoking the new OwnsMany() API:

``` csharp
modelBuilder.Entity<Customer>().OwnsMany(c => c.Addresses);
```

For more information about collections of owned entities, see the [updated documentation page](xref:core/modeling/owned-entities#collections-of-owned-types).

## Query tags

This feature is designed to facilitate the correlation of LINQ queries in code with the corresponding generated SQL output captured in logs.

To take advantage of the feature, you annotate a query using the new TagWith() API in a LINQ query. Using the spatial query from a previous example:

``` csharp
  var nearestFriends =
      (from f in context.Friends.TagWith(@"This is my spatial query!")
      orderby f.Location.Distance(myLocation) descending
      select f).Take(5).ToList();
```

This will produce the following SQL output:

``` sql
-- This is my spatial query!

SELECT TOP(@__p_1) [f].[Name], [f].[Location]
FROM [Friends] AS [f]
ORDER BY [f].[Location].STDistance(@__myLocation_0) DESC
```

For more information, see the [Query Tags documentation](xref:core/querying/tags). 