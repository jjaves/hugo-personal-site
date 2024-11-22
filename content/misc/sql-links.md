# 🗄️🔍 database design & SQL links

## SQL Online Learning Resources
 - [Kaggle's Intro to SQL](https://www.kaggle.com/learn/intro-to-sql)
 - [Kaggle's Advanced SQL](https://www.kaggle.com/learn/advanced-sql)
 - [Snowflake's Data Warehousing Workshop](https://learn.snowflake.com/en/courses/uni-essdww101/)
 - [Mode's SQL Tutorial](https://mode.com/sql-tutorial/)

## Snowflake Links
 - [Resource Optimization: Performance](https://quickstarts.snowflake.com/guide/resource_optimization_performance_optimization/index.html?index=..%2F..index#0)
 - [Resource Optimization: Usage Monitoring](https://quickstarts.snowflake.com/guide/resource_optimization_usage_monitoring/index.html?index=..%2F..index#0)
 - [Resource Optimization: Billing Metrics](https://quickstarts.snowflake.com/guide/resource_optimization_billing_metrics/index.html?index=..%2F..index#0)
 - [Clever way to join `information_schema` tables to `query_history`](https://stackoverflow.com/questions/70585936/how-to-get-a-list-of-tables-used-for-each-query-in-the-query-history-in-snowflak/70589299#70589299)

 - [Good article explaining Snowflake credit usage](https://medium.com/snowflake/mythbusting-snowflake-pricing-all-the-cool-stuff-you-get-with-1-credit-f3daad217a98)
 - [ODBC Configuration and Connection Parameters](https://docs.snowflake.com/en/developer-guide/odbc/odbc-parameters)
  
  ### Snowflake Cost Management Links
  - [Track Materialized Views](https://docs.snowflake.com/en/user-guide/views-materialized#label-materialized-views-maintenance-billing)
  - [Lower Auto-Suspend](https://docs.snowflake.com/en/user-guide/warehouses-overview#auto-suspension-and-auto-resumption)
  - [How Snowflake cache works](https://community.snowflake.com/s/article/Caching-in-Snowflake-Data-Warehouse) 
  - [Understand auto-suspension](https://docs.snowflake.com/en/user-guide/warehouses-overview#auto-suspension-and-auto-resumption)
  - [sample github snowflake monitoring queries](https://github.com/seanrm42/snowflake_scripts/tree/main/5_monitoring)
  - [Apply tags from dbt](https://docs.getdbt.com/reference/resource-configs/snowflake-configs#query-tags)

# Snowflake Cheat Sheet
### Functions

[NVL2](https://docs.snowflake.com/en/sql-reference/functions/nvl2.html) - Return values depending if first value is null.

[ARRAY_AGG](https://docs.snowflake.com/en/sql-reference/functions/array_agg.html) - Returns inputs pivoted into an array.

[PIVOT](https://docs.snowflake.com/en/sql-reference/constructs/pivot.html) - Transform narrow table to wider table by expanding dimensions to columns.

[VALUES](https://docs.snowflake.com/en/sql-reference/constructs/values.html) - Select from constant values

[GENERATOR](https://docs.snowflake.com/en/sql-reference/functions/generator.html) - Creates rows of data based either on a specified number of rows

[CONNECT BY](https://docs.snowflake.com/en/sql-reference/constructs/connect-by.html) - Joins a table to itself recursively

[ANY_VALUE](https://docs.snowflake.com/en/sql-reference/functions/any_value) - Returns non-deterministic value as an aggregate or window function

[OBJECT_KEYS](https://docs.snowflake.com/en/sql-reference/functions/object_keys) - Returns array of the top level of json object

[ENCRYPT](https://docs.snowflake.com/en/sql-reference/functions/encrypt) - Encrypts values with a passphrase

[DECRYPT](https://docs.snowflake.com/en/sql-reference/functions/decrypt) - Decrypts values with a passphrase, add `to_varchar([value], ‘utf-8’)`

[SYSTEM$CLUSTERING_INFORMATION](https://docs.snowflake.com/en/sql-reference/functions/system_clustering_information) - Returns clustering information

## useful examples

### Clustering

```sql
SYSTEM$CLUSTERING_INFORMATION( '<table_name>'
    [ , { '( <expr1> [ , <expr2> ... ] )' | <number_of_errors> } ] )
```

Example

```sql
select SYSTEM$CLUSTERING_INFORMATION('models.staging.transactions', '(date_trunc(day, CREATED_AT_LOCAL))')
```

#### Pivot without using pivot [example here](https://medium.com/snowflake/pivot-anything-in-snowflake-without-the-sql-pivot-function-ba030dce8dfc)

```sql
select
    parsed:project_name::string "project",
    parsed:"project mgr"::string "project mgr",
    parsed:"chief architect"::string "chief architect",
    parsed:"dba"::string "dba"
from
    (   select
            parse_json(proj_roles) parsed
        from
            (   select
                    concat('{', concat('"project_name','":"',project_name,'",', listagg(concat('"'
                    ,project_role, '":','"',person,'"'),',') ),'}')proj_roles
                from
                    (   select
                            project_name,
                            project_role,
                            person
                        from
                            "project_mgmt")
                group by
                    project_name));
```

#### Parsing email domains

```sql
SPLIT(email_address, '@')[SAFE_OFFSET(1)]
```

#### Convert date time to readable clock time

```sql
to_char({datetime_field}, 'HH12:MIPM')
```

#### Generator example to create 365 days into the future. Neat.

```sql
select
  dateadd(
    day,
    '-' || row_number() over (order by null),
    dateadd(day, '+1', current_date())
  ) as date
from table (generator(rowcount => 365))
```

#### How to overwrite Nulls using window function

```sql
select 
	c1, 
	coalesce(c2, 
		first_value(c2) over (partition by c1 order by c2 nulls last)
		) as c2
  from (values (1, 'one'), (1, null)) as v1 (c1, c2);
```

#### NVL2 example

```sql
NVL2( <expr1> , <expr2> , <expr3> )

If expr1 is NOT NULL, then NVL2 returns expr2.

If expr1 is NULL, then NVL2 returns expr3.
```

#### Convert UTC to Local

```sql
src.createdat::timestamp_ntz as created_at_utc,
convert_timezone('UTC', 'America/Los_Angeles', created_at_utc) as created_at_local,
```

#### Lateral Flatten

```sql
select id as "ID",
   f.value as "Contact",
   f1.value:type as "Type",
   f1.value:content as "Details"
 from persons p,
   lateral flatten(input => p.c, path => 'contact') f,
   lateral flatten(input => f.value:business) f1;
```