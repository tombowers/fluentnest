# FluentNest
[![Nuget Package](https://img.shields.io/nuget/v/fluentnest.svg)](https://www.nuget.org/packages/fluentnest)
[![Build status](https://ci.appveyor.com/api/projects/status/wrorpoekyw416hn1?svg=true)](https://ci.appveyor.com/project/hoonzis/fluentnest)

LINQ-like query language for ElasticSearch built on top of NEST.

NEST for querying ElasticSearch is great, but complex queries are hard to read and reason about. The same can be said about the basic JSON ElasticSearch query language. This library contains set of methods that give you more LINQ-like feeling. Currently mainly aggregations and filters are covered. More details are available [on my blog](http://www.hoonzis.com/fluent-interface-for-elastic-search/).

Statistics
----------
```Csharp
var result = client.Search<Car>(search => search.Aggregations(agg => agg
	.SumBy<Car>(x => x.Price)
	.CardinalityBy(x => x.EngineType)
);

var priceSum = result.Aggs.GetSum<Car, decimal>(x => x.Price);
var engines = result.Aggs.GetCardinality<Car>(x => x.EngineType);
```

Since all the queries are always done on the same entity type, one can also convert the result into a typed container:

```Csharp
var container = result.Aggs.AsContainer<Car>();
var priceSum = container.GetSum(x => x.Price);
var engines = container.GetCardinality(x => x.EngineType);
```

Conditional statistics
----------------------
Conditional sums can be quite complicated with NEST. One has to define a **Filter** aggregation with nested inner **Sum** or other aggregation. Here is quicker way with FluentNest:

```CSharp
var result = client.Search<Car>(search => search.Aggregations(aggs => aggs
	.SumBy(x=>x.Price, c => c.EngineType == EngineType.Diesel)
	.SumBy(x=>x.Sales, c => c.CarType == "Car1"))
);
```

Filtering && expressions to queries
----------------------------------
Filtering on multiple conditions might be complicated since you have to compose filters using *Or*, *And*, *Range* methods, often resulting in huge lambdas. **FluentNest** can compile small expressions into NEST query language. Examples:

```CSharp
client.Search<Car>(s => s.FilteredOn(f => f.Timestamp > startDate && f.Timestamp < endDate));
client.Search<Car>(s => s.FilteredOn(f=> f.Ranking.HasValue || f.IsAllowed);
client.Search<Car>(s => s.FilteredOn(f=> f.Ranking!=null || f.IsAllowed == true);
```
**HasValue** on a nullable as well as **!=null** are compiled into an **Exists** filter. Boolean values or expressions of style **==true** are compiled into bool filters. Note that the same expressions can be used for conditional statistics as well as for general filters which affect the whole query. Comparisons of values are compiled into **Terms** filters.

Grouped statistics
--------------------
Quite often you might want to calculate a sum per group. With FluentNest you can write:

```CSharp
var result = client.Search<Car>(search => search.Aggregations(agg => agg
	.SumBy<Car>(s => s.Price)
	.GroupBy(s => s.EngineType)
);
```

Just for a comparison, here is the same query using only NEST:

```Csharp
var result = client.Search<Car>(s => s
	.Aggregations(fstAgg => fstAgg
		.Terms("firstLevel", f => f
			.Field(z => z.CarType)
				.Aggregations(sums => sums
					.Sum("priceSum", son => son
					.Field(f4 => f4.Price)
				)
			)
		)
	)
);
```

Nested groups are very easy as well:

```Csharp
agg => agg
	.SumBy<Car>(s => s.Price)
	.GroupBy(s => s.CarType)
	.GroupBy(s => s.EngineType)
```

Helper methods are available to unwrap the groups from the ElasticSearch query result.

```Csharp
var carTypes = result.Aggs.GetGroupBy<Car>(x => x.CarType);
foreach (var carType in carTypes)
{
	var engineTypes = carType.GetGroupBy<Car>(x => x.EngineType);
	var priceSum = carType.GetGroupBy<Car>(x=>x.Price)
}
```

Dynamic nested grouping
-----------------------
In some cases you might need to group dynamically on multiple criteria specified at run-time. For such cases there is an overload of **GroupBy** which takes the name of the field for grouping. This overload can be used to obtain nested grouping on a list of fields:

```CSharp
var result = client.Search<Car>(search => search.Aggregations(x => agg
	.SumBy<Car>(s => s.Price)
	.GroupBy(new List<string> {"engineType", "carType"})
));
```

Hitograms
---------
Histogram is another useful aggregation supported by ElasticSearch. Here is a way to get a **Sum** by month.

```CSharp
var result = client.Search<Car>(s => s.Aggregations(a =>agg.
	.SumBy<Car>(x => x.Price)
	.IntoDateHistogram(date => date.Timestamp, DateInterval.Month)
);
var histogram = result.Aggs.GetDateHistogram<Car>(x => x.Timestamp);
```

Distinct values
---------------
Getting distinct values by certain property is translated into a **Terms** query. Like other operations it is chainable. A helper getter will return the value as the correct type. This works for enums as well - in the following example the second line gets all the *EngineType* enum values stored in ES.

```Csharp
var result = client.Search<Car>(search => search.Aggregations(x => agg
	.DistinctBy<Car>(x => x.CarType)
	.DistinctBy(x => x.EngineType)
));

var distinctCarTypes = result.Aggs.GetDistinct<Car, String>(x => x.CarType);
var engineTypes = result.Aggs.GetDistinct<Car, EngineType>(x => x.EngineType);
```
