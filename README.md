# Intersystems cache input plugin for Embulk

Cache input plugin for Embulk loads records from Intersystems cache database.

## Overview

* **Plugin type**: input
* **Resume supported**: yes

## Configuration

- **driver_path**: path to the jar file of cache JDBC driver. (string)
- **host**: database host name (string, required if url is not set)
- **port**: database port number (integer, default: 1972)
- **user**: database login user name (string, required)
- **password**: database login password (string, default: "")
- **namespace**: destination namespace name (string, required)
- **schema**: destination schema name (string, optional)
- **url**: URL of the JDBC connection (string, optional)
- If you write SQL directly,
  - **query**: SQL to run (string)
- If **query** is not set,
  - **table**: destination table name (string, required)
  - **select**: expression of select (e.g. `id, created_at`) (string, default: "*")
  - **where**: WHERE condition to filter the rows (string, default: no-condition)
  - **order_by**: expression of ORDER BY to sort rows (e.g. `created_at DESC, id ASC`) (string, default: not sorted)
- **fetch_rows**: number of rows to fetch one time (used for java.sql.Statement#setFetchSize) (integer, default: 10000)
- **connect_timeout**: timeout for the driver to connect. 0 means the default of SQL Server (15 by default). (integer (seconds), default: 300)
- **socket_timeout**: timeout for executing the query. 0 means no timeout. (integer (seconds), default: 1800)
- **options**: extra JDBC properties (hash, default: {})
- **incremental**: if true, enables incremental loading. See next section for details (boolean, default: false)
- **incremental_columns**: column names for incremental loading (array of strings, default: use primary keys)
- **last_record**: values of the last record for incremental loading (array of objects, default: load all records)
- **default_timezone**: If the sql type of a column is `date`/`time`/`datetime` and the embulk type is `string`, column values are formatted int this default_timezone. You can overwrite timezone for each columns using column_options option. (string, default: `UTC`)
- **column_options**: advanced: a key-value pairs where key is a column name and value is options for the column.
  - **value_type**: embulk get values from database as this value_type. Typically, the value_type determines `getXXX` method of `java.sql.PreparedStatement`.
  (string, default: depends on the sql type of the column. Available values options are: `long`, `double`, `float`, `decimal`, `boolean`, `string`, `json`, `date`, `time`, `timestamp`)
  - **type**: Column values are converted to this embulk type.
  Available values options are: `boolean`, `long`, `double`, `string`, `json`, `timestamp`).
  By default, the embulk type is determined according to the sql type of the column (or value_type if specified).
  - **timestamp_format**: If the sql type of the column is `date`/`time`/`datetime` and the embulk type is `string`, column values are formatted by this timestamp_format. And if the embulk type is `timestamp`, this timestamp_format may be used in the output plugin. For example, stdout plugin use the timestamp_format, but *csv formatter plugin doesn't use*. (string, default : `%Y-%m-%d` for `date`, `%H:%M:%S` for `time`, `%Y-%m-%d %H:%M:%S` for `timestamp`)
  - **timezone**: If the sql type of the column is `date`/`time`/`datetime` and the embulk type is `string`, column values are formatted in this timezone.
(string, value of default_timezone option is used by default)
- **after_select**: if set, this SQL will be executed after the SELECT query in the same transaction.


### Incremental loading

Incremental loading uses monotonically increasing unique columns (such as IDENTITY column) to load records inserted (or updated) after last execution.

First, if `incremental: true` is set, this plugin loads all records with additional ORDER BY. For example, if `incremental_columns: [updated_at, id]` option is set, query will be as following:

```
SELECT * FROM (
  ...original query is here...
)
ORDER BY updated_at, id
```

When bulk data loading finishes successfully, it outputs `last_record: ` paramater as config-diff so that next execution uses it.

At the next execution, when `last_record: ` is also set, this plugin generates additional WHERE conditions to load records larger than the last record. For example, if `last_record: ["2017-01-01 00:32:12", 5291]` is set,

```
SELECT * FROM (
  ...original query is here...
)
WHERE created_at > '2017-01-01 00:32:12' OR (created_at = '2017-01-01 00:32:12' AND id > 5291)
ORDER BY updated_at, id
```

Then, it updates `last_record: ` so that next execution uses the updated last_record.

**IMPORTANT**: If you set `incremental_columns: ` option, make sure that there is an index on the columns to avoid full table scan. For this example, following index should be created:

```
CREATE INDEX embulk_incremental_loading_index ON table (updated_at, id);
```

Recommended usage is to leave `incremental_columns` unset and let this plugin automatically finds an IDENTITY primary key. Currently, only strings and integers are supported as incremental_columns.


## Example

```yaml
in:
  type: intersystems_cache
  driver_path: /lib/cachejdbc.jar
  host: localhost
  user: _system
  password: ""
  namespace: SAMPLES
  schema: Sample
  table: Person
```

This configuration will generate following SQL:

```
SELECT * FROM "Sample"."Person"
```

Advanced configuration:

```yaml
in:
  type: intersystems_cache
  driver_path: /lib/cachejdbc.jar
  host: localhost
  user: myuser
  password: ""
  namespace: SERVER
  table: "my_table"
  select: "col1, col2, col3"
  where: "col4 != 'a'"
  column_options:
    col1: {type: long}
    col3: {type: string, timestamp_format: "%Y/%m/%d", timezone: "+0900"}
  after_select: "update my_table set col5 = '1' where col4 != 'a'"

```

## Build

```
$ ./gradlew gem
```

Running tests:

You need to add cachejdbc.jar to embulk-input-cache/lib
