---
layout: post
title: Row-level Error Handling with Apache Spark SQL
comments: true
excerpt_separator: <!--more-->
---

If you're using [Apache Spark SQL](https://spark.apache.org/docs/latest/sql-programming-guide.html) for running ETL jobs and applying data transformations between different domain models, you might be wondering what's the best way to deal with errors if some of the values cannot be mapped according to the specified business rules. In this blog post I would like to share one approach that can be used to filter out successful records and send to the next layer while quarantining failed records in a quarantine table. I'll be using PySpark and `DataFrame`s but the same concepts should apply when using Scala and `DataSet`s.
<!--more-->

In the below example your task is to transform the input data based on data model A into the target model B. Let's assume your model A data lives in a delta lake area called Bronze and your model B data lives in the area called Silver. When applying transformations to the input data we can also validate it at the same time.

For this example first we need to define some imports:

```python
from pyspark.sql import *
from pyspark.sql.functions import *
```

Let's say you have the following input `DataFrame` created with PySpark (in real world we would source it from our Bronze table):

```python
input_df = spark.createDataFrame(
    [
        (1, "value_1", True),
        (2, "value_2", False),
        (3, "value_3", None)
    ],
    "id INTEGER, string_col STRING, bool_col BOOLEAN"
)

input_df.show()

+--+----------+--------+
|id|string_col|bool_col|
+--+----------+--------+
| 1|   value_1|    true|
| 2|   value_2|   false|
| 3|   value_3|    null|
+--+----------+--------+
```

Now assume we need to implement the following business logic in our ETL pipeline using Spark that looks like this:

```python
mapped_df = input_df.select(
    col("id").alias("MAPPED_ID"),
    (
        when(col("string_col") == "value_1", "NEW_VAL_1")
        .when(col("string_col") == "value_2", "NEW_VAL_2")
    ).alias("MAPPED_STRING_COL"),
    (
        when(col("bool_col") == True, "MAPPED_BOOL_COL_TRUE")
        .when(col("bool_col") == False, "MAPPED_BOOL_COL_FALSE")
    ).alias("MAPPED_BOOL_COL")
)

mapped_df.show()

+---------+-----------------+-----------------------+
|MAPPED_ID|MAPPED_STRING_COL|        MAPPED_BOOL_COL|
+---------+-----------------+-----------------------+
|        1|        NEW_VAL_1|   MAPPED_BOOL_COL_TRUE|
|        2|        NEW_VAL_2|  MAPPED_BOOL_COL_FALSE|
|        3|          value_3|                   null|
+---------+-----------------+-----------------------+
```

As you can see now we have a bit of a problem. We were supposed to map our data from domain model A to domain model B but ended up with a `DataFrame` that's a mix of both. Even worse, we let invalid values (see row #3) slip through to the next step of our pipeline, and as every seasoned software engineer knows, it's always best to catch errors early. So, what can we do?

One approach could be to create a quarantine table still in our Bronze layer (and thus based on our domain model A) but enhanced with one extra column `errors` where we would store our failed records. Only successfully mapped records should be allowed through to the next layer (Silver).

In order to achieve this we need to somehow mark failed records and then split the resulting `DataFrame`. For this we can wrap the results of the transformation into a generic `Success`/`Failure` type of structure which most Scala developers should be familiar with. For this to work we just need to create 2 auxiliary functions:

```python
def create_success(mapped_value: Column) -> Column:
    return struct(mapped_value.alias("success"), lit(None).alias("failure"))

def create_failure(error_value: Column) -> Column:
    return struct(lit(None).alias("success"), error_value.alias("failure"))
```

So what happens here? Depending on the actual result of the mapping we can indicate either a success and wrap the resulting value, or a failure case and provide an error description. For the example above it would look something like this:

```python
mapped_df = input_df.select(
    create_success(col("id")).alias("MAPPED_ID"),
    (
        when(col("string_col") == "value_1", create_success("NEW_VAL_1"))
        .when(col("string_col") == "value_2", create_success("NEW_VAL_2"))
        .otherwise(
            create_failure(
                concat(
                    lit("Unable to map input column string_col value "),
                    col("string_col"),
                    lit(" to MAPPED_STRING_COL")
                )
            )
        )
    ).alias("MAPPED_STRING_COL"),
    (
        when(col("bool_col") == True, create_success("MAPPED_BOOL_COL_TRUE"))
        .when(col("bool_col") == False, create_success("MAPPED_BOOL_COL_FALSE"))
        .otherwise(
            create_failure(
                lit("Unable to map input column bool_col value to MAPPED_BOOL_COL because it's NULL")
            )
        )
    ).alias("MAPPED_BOOL_COL")
)

mapped_df.show()

+---------+---------------------+-----------------------------+
|MAPPED_ID|    MAPPED_STRING_COL|              MAPPED_BOOL_COL|
+---------+---------------------+-----------------------------+
|{1, null}|    {NEW_VAL_1, null}| {MAPPED_BOOL_COL_TRUE, null}|
|{2, null}|    {NEW_VAL_2, null}|{MAPPED_BOOL_COL_FALSE, null}|
|{3, null}|{null, Unable to ...}|        {null, Unable to ...}|
+---------+---------------------+-----------------------------+
```

You can see that by wrapping each mapped value into a `StructType` we were able to capture about `Success` and `Failure` cases separately.

Now based on this information we can split our `DataFrame` into 2 sets of rows: those that didn't have any mapping errors (hopefully the majority) and those that have at least one column that failed to be mapped into the target domain. In order to achieve this let's define the filtering functions as follows:

```python
def filter_success(df: DataFrame) -> DataFrame:
    return df.filter(" AND ".join([f"{col_name}.failure IS NULL" for col_name in _mapped_col_names(df)]))
             .selectExpr(*[f"{col_name}.success AS {col_name}" for col_name in _mapped_col_names(df)])

def filter_failure(df: DataFrame, original_df: DataFrame) -> DataFrame:
    return df.filter(" OR ".join([f"{col_name}.failure IS NOT NULL" for col_name in _mapped_col_names(df)]))
             .withColumn("temp_arr", array(*[f"{col_name}.failure" for col_name in _mapped_col_names(df)]))
             .withColumn("errors", expr("FILTER(temp_arr, x -> x is not null)"))
             .select(original_df["*"], "errors")

def _mapped_col_names(df: DataFrame) -> List[str]:
    return [c for c in df.schema.fieldNames() if col_name.startswith("MAPPED_")]
```

Ok, this probably requires some explanation. In the function `filter_success()` first we filter for all rows that were successfully processed and then unwrap the `success` field of our `STRUCT` data type created earlier to flatten the resulting `DataFrame` that can then be persisted into the Silver area of our data lake for further processing. The function `filter_failure()` looks for all rows where at least one of the fields could not be mapped, then the two following `withColumn()` calls make sure that we collect all error messages into one `ARRAY` typed field called `errors`, and then finally we select all of the columns from the original `DataFrame` plus the additional `errors` column, which would be ready to persist into our quarantine table in Bronze. The helper function `_mapped_col_names()` simply iterates over all column names _not_ in the original `DataFrame`, i.e. those which start with the prefix `MAPPED_`.

Now when we execute both functions for our sample `DataFrame` that we received as output of our transformation step we should see the following:

```python
df_success = filter_success(mapped_df)

df_success.show()

+---------+---------------------+-----------------------------+
|MAPPED_ID|    MAPPED_STRING_COL|              MAPPED_BOOL_COL|
+---------+---------------------+-----------------------------+
|        1|            NEW_VAL_1|         MAPPED_BOOL_COL_TRUE|
|        2|            NEW_VAL_2|        MAPPED_BOOL_COL_FALSE|
+---------+---------------------+-----------------------------+

df_failure = filter_failure(mapped_df)

df_failure.show()

+--+----------+--------+------------------------------+
|id|string_col|bool_col|                        errors|
+--+----------+--------+------------------------------+
| 3|   value_3|    null|[Unable to ..., Unable to ...]|
+--+----------+--------+------------------------------+
```

As we've seen in the above example, row-level error handling with Spark SQL requires some manual effort but once the foundation is laid it's easy to build up on it by e.g. extracting it into a common module and reusing the same concept for all types of data and transformations. One of the next steps could be automated reprocessing of the records from the quarantine table e.g. after a bug fix. If you have any questions let me know in the comments section below!