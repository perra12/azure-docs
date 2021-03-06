---
title: Design guidance for replicated tables - Azure SQL Data Warehouse | Microsoft Docs
description: Recommendations for designing replicated tables in your Azure SQL Data Warehouse schema. 
services: sql-data-warehouse
documentationcenter: NA
author: ronortloff
manager: jhubbard
editor: ''

ms.service: sql-data-warehouse
ms.devlang: NA
ms.topic: article
ms.tgt_pltfrm: NA
ms.workload: data-services
ms.custom: tables
ms.date: 07/14/2017
ms.author: rortloff;barbkess

---

# Design guidance for using replicated tables in Azure SQL Data Warehouse
This article gives recommendations for designing replicated tables in your SQL Data Warehouse schema. Use these recommendations to improve query performance by reducing data movement and query complexity.

> [!NOTE]
> The replicated table feature is currently in public preview. Some behaviors are subject to change.
> 

## Prerequisites
This article assumes you are familiar with data distribution and data movement concepts in SQL Data Warehouse.  For more information, see [Distributed data](sql-data-warehouse-distributed-data.md). 

As part of table design, understand as much as possible about your data and how the data is queried.  For example, consider these questions:

- How large is the table?   
- How often is the table refreshed?   
- Do I have fact and dimension tables in a data warehouse?   

## What is a replicated table?
A replicated table has a full copy of the table accessible on each Compute node. Replicating a table removes the need to transfer data among Compute nodes before a join or aggregation. Since the table has multiple copies, replicated tables work best when the table size is less than 2 GB compressed.

The following diagram shows a replicated table that is accessible on each Compute node. In SQL Data Warehouse, the replicated table is fully copied to a distribution database on each Compute node. 

![Replicated table](media/sql-data-warehouse-distributed-data/replicated-table.png "Replicated table")  

Replicated tables work well for small dimension tables in a star schema. Dimension tables are usually of a size that makes it feasible to store and maintain multiple copies. Dimensions store descriptive data that changes slowly, such as customer name and address, and product details. The slowly changing nature of the data leads to fewer rebuilds of the replicated table. 

Consider using a replicated table when:

- The table size on disk is less than 2 GB, regardless of the number of rows. To find the size of a table, you can use the [DBCC PDW_SHOWSPACEUSED](https://docs.microsoft.com/en-us/sql/t-sql/database-console-commands/dbcc-pdw-showspaceused-transact-sql) command: `DBCC PDW_SHOWSPACEUSED('ReplTableCandidate')`. 
- The table is used in joins that would otherwise require data movement. For example, a join on hash-distributed tables requires data movement when the joining columns are not the same distribution column. If one of the hash-distributed tables is small, consider a replicated table. A join on a round-robin table requires data movement. We recommend using replicated tables instead of round-robin tables in most cases. 


Consider converting an existing distributed table to a replicated table when:

- Query plans use data movement operations that broadcast the data to all the Compute nodes. The BroadcastMoveOperation is expensive and slows query performance. To view data movement operations in query plans, use [sys.dm_pdw_request_steps](https://docs.microsoft.com/sql/relational-databases/system-dynamic-management-views/sys-dm-pdw-request-steps-transact-sql).
 
Replicated tables may not yield the best query performance when:

- The table has frequent insert, update, and delete operations. These data manipulation language (DML) operations require a rebuild of the replicated table. Rebuilding frequently can cause slower performance.
- The data warehouse is scaled frequently. Scaling a data warehouse changes the number of Compute nodes, which incurs a rebuild.
- The table has a large number of columns, but data operations typically access only a small number of columns. In this scenario, instead of replicating the entire table, it might be more effective to hash distribute the table, and then create an index on the frequently accessed columns. When a query requires data movement, SQL Data Warehouse only moves data in the requested columns. 



## Use replicated tables with simple query predicates
Before you choose to distribute or replicate a table, think about the types of queries you plan to run against the table. Whenever possible,

- Use replicated tables for queries with simple query predicates, such as equality or inequality.
- Use distributed tables for queries with complex query predicates, such as LIKE or NOT LIKE.

CPU-intensive queries perform best when the work is distributed across all of the Compute nodes. For example, queries that run computations on each row of a table perform better on distributed tables than replicated tables. Since a replicated table is stored in full on each Compute node, a CPU-intensive query against a replicated table runs against the entire table on every Compute node. The extra computation can slow query performance.

For example, this query has a complex predicate.  It runs faster when supplier is a distributed table instead of a replicated table. In this example, supplier can be hash-distributed or round-robin distributed.

```sql

SELECT EnglishProductName 
FROM DimProduct 
WHERE EnglishDescription LIKE '%frame%comfortable%'

```

## Convert existing round-robin tables to replicated tables
If you already have round-robin tables, we recommend converting them to replicated tables if they meet with criteria outlined in this article. Replicated tables improve performance over round-robin tables because they eliminate the need for data movement.  A round-robin table always requires data movement for joins. 

This example uses [CTAS](https://docs.microsoft.com/sql/t-sql/statements/create-table-as-select-azure-sql-data-warehouse) to change the DimSalesTerritory table to a replicated table. This example works regardless of whether DimSalesTerritory is hash-distributed or round-robin.

```sql
CREATE TABLE [dbo].[DimSalesTerritory_REPLICATE]   
WITH   
  (   
    CLUSTERED COLUMNSTORE INDEX,  
    DISTRIBUTION = REPLICATE  
  )  
AS SELECT * FROM [dbo].[DimSalesTerritory]
OPTION  (LABEL  = 'CTAS : DimSalesTerritory_REPLICATE') 

--Create statistics on new table
CREATE STATISTICS [SalesTerritoryKey] ON [DimSalesTerritory_REPLICATE] ([SalesTerritoryKey]);
CREATE STATISTICS [SalesTerritoryAlternateKey] ON [DimSalesTerritory_REPLICATE] ([SalesTerritoryAlternateKey]);
CREATE STATISTICS [SalesTerritoryRegion] ON [DimSalesTerritory_REPLICATE] ([SalesTerritoryRegion]);
CREATE STATISTICS [SalesTerritoryCountry] ON [DimSalesTerritory_REPLICATE] ([SalesTerritoryCountry]);
CREATE STATISTICS [SalesTerritoryGroup] ON [DimSalesTerritory_REPLICATE] ([SalesTerritoryGroup]);

-- Switch table names
RENAME OBJECT [dbo].[DimSalesTerritory] to [DimSalesTerritory_old];
RENAME OBJECT [dbo].[DimSalesTerritory_REPLICATE] TO [DimSalesTerritory];

DROP TABLE [dbo].[DimSalesTerritory_old];
```  

### Query performance example for round-robin versus replicated 

A replicated table does not require any data movement for joins because the entire table is already present on each Compute node. If the dimension tables are round-robin distributed, a join copies the dimension table in full to each Compute node. To move the data, the query plan contains an operation called BroadcastMoveOperation. This type of data movement operation slows query performance and is eliminated by using replicated tables. To view query plan steps, use the [sys.dm_pdw_request_steps](https://docs.microsoft.com/sql/relational-databases/system-dynamic-management-views/sys-dm-pdw-request-steps-transact-sql) system catalog view. 

For example, in following query against the AdventureWorks schema, the ` FactInternetSales` table is hash-distributed. The `DimDate` and `DimSalesTerritory` tables are smaller dimension tables. This query returns the total sales in North America for fiscal year 2004:
 
```sql
SELECT [TotalSalesAmount] = SUM(SalesAmount)
FROM dbo.FactInternetSales s
INNER JOIN dbo.DimDate d
  ON d.DateKey = s.OrderDateKey
INNER JOIN dbo.DimSalesTerritory t
  ON t.SalesTerritoryKey = s.SalesTerritoryKey
WHERE d.FiscalYear = 2004
  AND t.SalesTerritoryGroup = 'North America'
```
We re-created `DimDate` and `DimSalesTerritory` as round-robin tables. As a result, the query showed the following query plan, which has multiple broadcast move operations: 
 
![Round-robin query plan](media/design-guidance-for-replicated-tables/round-robin-tables-query-plan.jpg "Round-robin query plan") 

We re-created `DimDate` and `DimSalesTerritory` as replicated tables, and ran the query again. The resulting query plan is much shorter and does not have any broadcast moves.

![Replicated query plan](media/design-guidance-for-replicated-tables/replicated-tables-query-plan.jpg "Round-robin query plan") 


## Performance considerations for modifying replicated tables
SQL Data Warehouse implements a replicated table by maintaining a master version of the table. It copies the master version to one distribution database on each Compute node. When there is a change, SQL Data Warehouse first updates the master table. Then it requires a rebuild of the tables on each Compute node. A rebuild of a replicated table includes copying the table to each Compute node and then rebuilding the indexes.

Rebuilds are required after:
- Data is loaded or modified
- The data warehouse is scaled to a different DWU setting
- Table definition is updated

Rebuilds are not required after:
- Pause operation
- Resume operation

The rebuild does not happen immediately after data is modified. Instead, the rebuild is triggered the first time a query selects from the table.  Within the initial select statement from the table are steps to rebuild the replicated table.  Because the rebuild is done within the query, the impact to the initial select statement could be significant depending on the size of the table.  If multiple replicated tables are involved that need a rebuild, each copy is rebuilt serially as steps within the statement.  To maintain data consistency during the rebuild of the replicated table an exclusive lock is taken on the table.  The lock prevents all access to the table for the duration of the rebuild. 

### Use indexes conservatively
Standard indexing practices apply to replicated tables. SQL Data Warehouse rebuilds each replicated table index as part of the rebuild. Only use indexes when the performance gain outweighs the cost of rebuilding the indexes.  
 
### Batch data loads
When loading data into replicated tables, try to minimize rebuilds by batching loads together. Perform all the batched loads before running select statements.

For example, this load pattern loads data from four sources and invokes four rebuilds. 

- Load from source 1.
- Select statement triggers rebuild 1.
- Load from source 2.
- Select statement triggers rebuild 2.
- Load from source 3.
- Select statement triggers rebuild 3.
- Load from source 4.
- Select statement triggers rebuild 4.

For example, this load pattern loads data from four sources, but only invokes one rebuild.

- Load from source 1.
- Load from source 2.
- Load from source 3.
- Load from source 4.
- Select statement triggers rebuild.


### Rebuild a replicated table after a batch load
To ensure consistent query execution times, we recommend forcing a refresh of the replicated tables after a batch load. Otherwise, the first query must wait for the tables to refresh, which includes rebuilding the indexes. Depending on the size and number of replicated tables affected, the performance impact can be significant.  

This query uses the [sys.pdw_replicated_table_cache_state](https://docs.microsoft.com/sql/relational-databases/system-catalog-views/sys-pdw-replicated-table-cache-state-transact-sql) DMV to list the replicated tables that have been modified, but not rebuilt.

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
 
To force a rebuild, run the following statement on each table in the preceding output. 

```sql
SELECT TOP 1 * FROM [ReplicatedTable]
``` 
 
## Next steps 
To create a replicated table, use one of these statements:

- [CREATE TABLE (Azure SQL Data Warehouse)](https://docs.microsoft.com/sql/t-sql/statements/create-table-azure-sql-data-warehouse)
- [CREATE TABLE AS SELECT (Azure SQL Data Warehouse](https://docs.microsoft.com/sql/t-sql/statements/create-table-as-select-azure-sql-data-warehouse)

For an overview of distributed tables, see [distributed tables](sql-data-warehouse-tables-distribute.md).



