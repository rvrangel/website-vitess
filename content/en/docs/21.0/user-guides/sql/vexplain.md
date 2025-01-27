---
title: Analyzing a SQL statement using VEXPLAIN
weight: 1
aliases: ['/docs/user-guides/vtexplain/']
---

# Introduction

To see which queries are run on your behalf on the MySQL instances when you execute a query on vtgate, you can use `vexplain [ALL|PLAN|QUERIES|TRACE|KEYS]`.

# `QUERIES` Type

The `QUERIES` format returns an output similar to what the command line application [`vtexplain`](../../../reference/programs/vtexplain) returns - a list of the queries that have been run on MySQL, and against which shards they were issued.

## How it works

Unlike normal `EXPLAIN` queries, `VEXPLAIN QUERIES` actually runs your query, and logs the interactions with the tablets.
After running your query using this extra logging, the result you get is a table with all the interactions listed.

## How to read the output

The output has four columns:
* The first column, `#` groups queries that were sent in a single call together.
* Keyspace - which keyspace was this query sent to.
* Shard - for sharded keyspaces, this column will show which shard a query is sent to.
* Query - the actual query used.

### Example 1:
```mysql
mysql> vexplain queries select * from user where id = 4;
+------+----------+-------+-----------------------------------------------------------+
| #    | keyspace | shard | query                                                     |
+------+----------+-------+-----------------------------------------------------------+
|    0 | ks       | c0-   | select id, lookup, lookup_unique from `user` where id = 4 |
+------+----------+-------+-----------------------------------------------------------+
1 row in set (0.00 sec)
```

Here we have a query where the planner can immediately see which shard to send the query to.

### Example 2:
```mysql
mysql> vexplain queries select * from user where lookup = 'apa';
+------+----------+-------+-------------------------------------------------------------------+
| #    | keyspace | shard | query                                                             |
+------+----------+-------+-------------------------------------------------------------------+
|    0 | ks       | -40   | select lookup, keyspace_id from lookup where lookup in ('apa')    |
|    1 | ks       | c0-   | select id, lookup, lookup_unique from `user` where lookup = 'apa' |
|    2 | ks       | 40-80 | select id, lookup, lookup_unique from `user` where lookup = 'apa' |
+------+----------+-------+-------------------------------------------------------------------+
3 rows in set (0.02 sec)
```

This is a query where the planner has to do a vindex lookup to find which shard the data might live on.

# `PLAN` Type

The `PLAN` format returns the vtgate plan for the given query.
It does so without actually running any queries - it just plans the given query and presents the plan.

## How to read the output

The output contains a scalar output having a JSON description of the plan that vtgate will use for the query.

### Example:
```mysql
mysql> vexplain plan select * from corder join commerce.product as prod on corder.sku = prod.sku;
```

```json
{
  "OperatorType": "Join",
  "Variant": "Join",
  "JoinColumnIndexes": "L:1,L:2,L:3,L:4,R:0,R:1,R:2",
  "JoinVars": {
    "corder_sku": 0
  },
  "TableName": "corder_product",
  "Inputs": [
    {
      "OperatorType": "Route",
      "Variant": "Scatter",
      "Keyspace": {
        "Name": "customer",
        "Sharded": true
      },
      "FieldQuery": "select corder.sku, corder.order_id as order_id, corder.customer_id as customer_id, corder.sku as sku, corder.price as price from corder where 1 != 1",
      "Query": "select corder.sku, corder.order_id as order_id, corder.customer_id as customer_id, corder.sku as sku, corder.price as price from corder",
      "Table": "corder"
    },
    {
      "OperatorType": "Route",
      "Variant": "Unsharded",
      "Keyspace": {
        "Name": "commerce",
        "Sharded": false
      },
      "FieldQuery": "select prod.sku as sku, prod.description as description, prod.price as price from product as prod where 1 != 1",
      "Query": "select prod.sku as sku, prod.description as description, prod.price as price from product as prod where prod.sku = :corder_sku",
      "Table": "product"
    }
  ]
} 
```

In this example, we are executing a cross-keyspace join between two tables. The `corder` table living in the `customer` keyspace and `product` table living in the `commerce` keyspace.

# `ALL` Type

The `ALL` format returns the vtgate plan along with the MySQL explain output for the executed queries.

## How to read the output

The output contains a scalar output having a JSON description of the plan that vtgate will use for the query annotated with the explain output from mysql for these queries.

### Example:

```mysql
mysql> vexplain all select * from corder join commerce.product as prod on corder.sku = prod.sku;
```

```json
{
  "OperatorType": "Join",
  "Variant": "Join",
  "JoinColumnIndexes": "L:1,L:2,L:3,L:4,R:0,R:1,R:2",
  "JoinVars": {
    "corder_sku": 0
  },
  "TableName": "corder_product",
  "Inputs": [
    {
      "OperatorType": "Route",
      "Variant": "Scatter",
      "Keyspace": {
        "Name": "customer",
        "Sharded": true
      },
      "FieldQuery": "select corder.sku, corder.order_id as order_id, corder.customer_id as customer_id, corder.sku as sku, corder.price as price from corder where 1 != 1",
      "Query": "select corder.sku, corder.order_id as order_id, corder.customer_id as customer_id, corder.sku as sku, corder.price as price from corder",
      "Table": "corder",
      "mysql_explain_json": {
        "query_block": {
          "select_id": 1,
          "cost_info": {
            "query_cost": "0.65"
          },
          "table": {
            "table_name": "corder",
            "access_type": "ALL",
            "rows_examined_per_scan": 4,
            "rows_produced_per_join": 4,
            "filtered": "100.00",
            "cost_info": {
              "read_cost": "0.25",
              "eval_cost": "0.40",
              "prefix_cost": "0.65",
              "data_read_per_join": "640"
            },
            "used_columns": [
              "order_id",
              "customer_id",
              "sku",
              "price"
            ]
          }
        }
      }
    },
    {
      "OperatorType": "Route",
      "Variant": "Unsharded",
      "Keyspace": {
        "Name": "commerce",
        "Sharded": false
      },
      "FieldQuery": "select prod.sku as sku, prod.description as description, prod.price as price from product as prod where 1 != 1",
      "Query": "select prod.sku as sku, prod.description as description, prod.price as price from product as prod where prod.sku = :corder_sku",
      "Table": "product",
      "mysql_explain_json": {
        "query_block": {
          "select_id": 1,
          "cost_info": {
            "query_cost": "1.00"
          },
          "table": {
            "table_name": "prod",
            "access_type": "const",
            "possible_keys": [
              "PRIMARY"
            ],
            "key": "PRIMARY",
            "used_key_parts": [
              "sku"
            ],
            "key_length": "130",
            "ref": [
              "const"
            ],
            "rows_examined_per_scan": 1,
            "rows_produced_per_join": 1,
            "filtered": "100.00",
            "cost_info": {
              "read_cost": "0.00",
              "eval_cost": "0.10",
              "prefix_cost": "0.00",
              "data_read_per_join": "272"
            },
            "used_columns": [
              "sku",
              "description",
              "price"
            ]
          }
        }
      }
    }
  ]
}
```

This example uses the same query as the previous ones. For all the Route operators, we are annotating them with the MySQL explain output for the query that the route is executing.

# `TRACE` Type

The `TRACE` format provides a detailed execution trace of the query, showing how it's processed through various operators and interactions with tablets.

## How it works

`VEXPLAIN TRACE` runs the query and logs all interactions between the operators and the tablets. VExplain returns a single row with a single column containing the execution plan in JSON format, including all the interactions between the operators and the tablets.
## How to read the output

The output is a JSON representation of the query execution plan, with additional fields for each operator:

- `NoOfCalls`: Number of times the operator was invoked
- `AvgNumberOfRows`: Average number of rows processed by the operator
- `MedianNumberOfRows`: Median number of rows processed by the operator
- `ShardsQueried`: (For operators interacting with vttablets) Number of shards queried

### Example:

```mysql
mysql> vexplain trace select ... ;
```

```json
{
  "OperatorType": "Projection",
  "NoOfCalls": 1,
  "AvgNumberOfRows": 5,
  "MedianNumberOfRows": 5,
  "Inputs": [
    {
      "OperatorType": "Aggregate",
      "Variant": "Ordered",
      "NoOfCalls": 1,
      "AvgNumberOfRows": 5,
      "MedianNumberOfRows": 5,
      "Inputs": [
        {
          "OperatorType": "Route",
          "Variant": "Scatter",
          "NoOfCalls": 1,
          "AvgNumberOfRows": 8,
          "MedianNumberOfRows": 8,
          "ShardsQueried": 2
        }
      ]
    }
  ]
}
```

This trace output shows the execution flow through different operators (Projection, Aggregate, Route) and provides statistics on the number of calls and rows processed at each step.

## When to use TRACE

Use `vexplain trace` when you need to thoroughly understand what a query is doing and where it's performing the most work. It's particularly useful for:

1. Identifying performance bottlenecks
2. Understanding query execution patterns
3. Optimizing complex queries
4. Debugging unexpected query behavior

By analyzing the trace output, you can gain insights into how your query is executed across different shards and operators, helping you make informed decisions about query optimization and database design.

# `KEYS` Type

The `KEYS` format provides a concise summary of the query structure, highlighting columns used in joins, filters, and grouping operations. This information is crucial for making informed decisions about sharding keys and optimizing query performance.

## How it works

`VEXPLAIN KEYS` analyzes the query structure without executing it. It identifies important columns that are potential candidates for sharding keys, including SARGable columns in the WHERE clause, join conditions, and grouping columns.

## How to read the output

The output is a JSON object containing the following key information:

- `groupingColumns`: Columns used in GROUP BY clauses
- `joinColumns`: Columns used in join conditions, not matter if they are on the WHERE or the JOIN ... ON clause
- `filterColumns`: Columns used in WHERE clauses that are potential candidates for indexes (or vindexes), primary keys, or sharding keys. These typically include columns used in equality comparisons or range conditions.
- `statementType`: The type of SQL statement (e.g., SELECT, INSERT, UPDATE, DELETE)

### Example:

```mysql
mysql> vexplain keys select u.foo, ue.bar, count(*) from user u join user_extra ue on u.id = ue.user_id where u.name = 'John Doe' group by 1, 2;
+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ColumnUsage                                                                                                                                                                                |
+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| {
        "groupingColumns": [
                "user.foo",
                "user_extra.bar"
        ],
        "joinColumns": [
                "user.id",
                "user_extra.user_id"
        ],
        "filterColumns": [
                "user.name"
        ],
        "statementType": "SELECT"
} |
+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
```

## When to use KEYS
Use vexplain keys when you need to:

* Identify potential sharding key candidates
* Optimize query performance by understanding which columns are frequently used in filters and joins
* Analyze query patterns across your application to inform database design decisions
* Quickly understand the structure of complex queries

By analyzing the KEYS output across multiple queries, you can make more informed decisions about sharding strategies, potentially improving query performance and data distribution in your Vitess deployment.

# Safety for DML

The normal behaviour for `VEXPLAIN` is to not actually run the query for DMLs — it usually only plans the query and presents the produced plan for the `PLAN` type.
Since `vexplain ALL|QUERIES` really runs your queries, you need to add a query directive to show that you are aware that your DML will actually run.

### Example:

```mysql
mysql> vexplain queries insert into customer(email) values('abc@xyz.com');
ERROR 1105 (HY000): VT09008: vexplain queries/all will actually run queries
```

This is the error you will get is you do not add the comment directive to your `VEXPLAIN` statement.

### Example:

```mysql
mysql> vexplain /*vt+ EXECUTE_DML_QUERIES */ queries insert into customer(email) values('abc@xyz.com');
+------+----------+-------+-----------------------------------------------------------------------+
| #    | keyspace | shard | query                                                                 |
+------+----------+-------+-----------------------------------------------------------------------+
|    0 | customer | 80-   | insert into customer(email, customer_id) values ('abc@xyz.com', 1001) |
+------+----------+-------+-----------------------------------------------------------------------+
1 row in set (0.00 sec)
```

Here we can see how vtgate will insert rows to the main table, but also to the two lookup vindexes declared for this table.

Note - MySQL client by default strips out the comments from the queries before it sends to the server.
So you'll need to run the client with `-c` flag to allow passing in comments.
