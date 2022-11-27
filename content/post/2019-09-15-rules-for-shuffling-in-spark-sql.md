---
title: "Rules for shuffling in Spark SQL"
date: 2019-09-15
tags: ["programming", "big-data"]
---
Make your Spark jobs reliable and fast --- use shuffle correctly.

## Avoid shuffle

Moving data between cluster nodes is very expensive. So try to reduce number of shuffles to minimum. Eg. in some cases sort-merge join can be replaced by broadcast join. If the data you want to join are too big for single broadcast join (the limit for single broadcast join is 8 GB) you can use several consecutive broadcast joins.

Sometimes you can avoid shuffle when using multiple `groupBy`s or multiple windows with window functions. Eg. if you call `groupBy($"userId", $"hour")` and later `groupBy($"userId")` then Spark will do 2 shuffles because after the first `groupBy` records for certain user can be on multiple executors (so second `groupBy` needs to redistribute records to ensure that records for single user are on single executor). On the other hand if you do first `groupBy($"userId")` and later `groupBy($"userId", $"hour")` then 1 shuffle may be enough.

## Shuffle less data

If you need shuffle then shuffle only the data you need. Maybe some columns can be computed after shuffling. Eg. you already have column `timestamp` and you want to add columns `year`, `month`, `day` then do so after the shuffle.

Even if you already computed those columns and need to shuffle consider dropping the columns, shuffling and then computing them again.

Every column adds at least 8 bytes to your row (even if it is just `Boolean` Spark will store it in `UnsafeRow` where everything is aligned to 8 bytes). To shuffle the column Spark will compress its values (remember at least 8 bytes each), send them over the network and decompress the values. You should keep this in mind when evaluating cost of recomputation. Eg. recomputation of `year`, `month`, `day` from `timestamp` is certainly faster than compression + transfer over the network + decompression.

Additionally if some columns of the row won’t be needed after the shuffle then replace their values before shuffle with `null`. Eg. suppose you have column `url` which is used after shuffle but only in certain conditions. Then replace `url` with `null` before shuffle if you know that those conditions won’t be met. This is especially important for nested structures, strings, arrays and maps because it will reduce the size of your `UnsafeRow`s.

Generally before shuffle you need:

* filter as many rows as possible,
* drop as many columns as possible,
* replace big columns by small columns,
* replace as many values as possible by `null`s.

## When shuffling make your partitions evenly sized

When using `repartition(partitionExprs: Column*)` (or anything which translates to `RepartitionByExpression` in your logical plan) use `partitionExprs` which result in partitions which need similar time and similar amounts of memory for processing. This prevents your jobs from failing because a small bunch of tasks are constantly running out of memory.

## Conclusion

So these were 3 simple tips how to optimize your shuffling when using Spark SQL. Completely different approach is to forget about distributed computation and run your job on single machine without Spark. If your machine has enough memory it can actually be faster than Spark because you can avoid lots of IO.
