# AzureSynapseNotes

https://docs.microsoft.com/zh-cn/azure/synapse-analytics/sql-data-warehouse/sql-data-warehouse-tables-distribute


# 选择数据均衡分布的分布列
为了获得最佳性能，所有分布区都应当具有大致相同的行数。 当一个或多个分布区的行数不相称时，某些分布区会先于其他分布区完成其并行查询部分。 由于必须等到所有分布区都完成处理，才能完成查询，因此，每个查询的速度取决于最慢分布区的速度。

数据倾斜意味着数据未均衡分布在分布区中
处理倾斜意味着在运行并行查询时，某些分布区所用的时间比其他分布区长。 数据倾斜时可能会出现这种情况。
若要平衡并行处理，请选择符合以下条件的分布列：

- 具有许多唯一值。 该列可以有重复值。 具有相同值的所有行都会分配到同一分布区。 由于有 60 个分布区，因此一些分布区可能有多个唯一值，而其他分布区可能以零值结尾。
- 没有 NULL 值，或者只有几个 NULL 值。 在极端示例中，如果列中的所有值均为 NULL，则所有行都分配到相同的分布区。 因此，查询处理会向一个分布区倾斜，无法从并行处理中受益。
- 不是日期列。 同一日期的所有数据都落在相同的分布区。 如果多个用户都筛选同一个日期，则会由 60 个分布区中的 1 个分布区单独执行所有处理工作。
- 选择能最大程度减少数据移动的分布列
为了获取正确的查询结果，查询可能将数据从一个计算节点移至另一个计算节点。 当查询对分布式表执行联接和聚合操作时，通常会发生数据移动。 选择一个能最大程度减少数据移动的分布列，这是优化专用 SQL 池性能的最重要策略之一。

### 若要最大程度减少数据移动，请选择符合以下条件的分布列：

- 用于 JOIN、GROUP BY、DISTINCT、OVER 和 HAVING 子句。 当两个大型事实数据表频繁联接时，如果将这两个表分布在某个联接列上，查询性能将得到提升。 如果某个表不进行联接操作，则考虑将该表分布在经常出现在 GROUP BY 子句中的列上。
- 不 用于 WHERE 子句。 这可以缩小查询范围，从而不必在所有分布区上运行查询。
- 不 是日期列。 WHERE 子句通常按日期进行筛选。 在这种情况下，可能会在少数几个分布区上运行所有处理。


## 循环表与复制的表的查询性能对比示例

复制的表不要求为联接移动任何数据，因为整个表已存在于每个计算节点上。 如果维度表是循环分布式的，则联接会将维度表整个复制到每个计算节点。 为了移动数据，查询计划包含了一个名为 BroadcastMoveOperation 的操作。 此类数据移动操作会降低查询性能，使用复制的表可以避免。 若要查看查询计划步骤，请使用 sys.dm_pdw_request_steps 系统目录视图。

例如，在针对 AdventureWorks 架构的以下查询中，FactInternetSales 表是哈希分布表。 DimDate 和 DimSalesTerritory 表是较小的维度表。 此查询返回 2004 会计年度在北美的总销售额：

以下查询使用 sys.pdw_replicated_table_cache_state DMV 列出已修改但未重新生成的复制的表。
```sql
SELECT [ReplicatedTable] = t.[name]
  FROM sys.tables t  
  JOIN sys.pdw_replicated_table_cache_state c  
    ON c.object_id = t.object_id
  JOIN sys.pdw_table_distribution_properties p
    ON p.object_id = t.object_id
  WHERE c.[state] = 'NotReady'
    AND p.[distribution_policy_desc] = 'REPLICATE'

```


## 选择能最大程度减少数据移动的分布列

为了获取正确的查询结果，查询可能将数据从一个计算节点移至另一个计算节点。 当查询对分布式表执行联接和聚合操作时，通常会发生数据移动。 选择一个能最大程度减少数据移动的分布列，这是优化专用 SQL 池性能的最重要策略之一。

若要最大程度减少数据移动，请选择符合以下条件的分布列：

- 用于 JOIN、GROUP BY、DISTINCT、OVER 和 HAVING 子句。 当两个大型事实数据表频繁联接时，如果将这两个表分布在某个联接列上，查询性能将得到提升。 如果某个表不进行联接操作，则考虑将该表分布在经常出现在 GROUP BY 子句中的列上。
- 不 用于 WHERE 子句。 这可以缩小查询范围，从而不必在所有分布区上运行查询。
- 不 是日期列。 WHERE 子句通常按日期进行筛选。 在这种情况下，可能会在少数几个分布区上运行所有处理。