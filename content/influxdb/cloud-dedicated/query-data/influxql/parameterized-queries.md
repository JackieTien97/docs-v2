---
title: Use parameterized queries with InfluxQL
description: >
  Use parameterized queries to prevent injection attacks and make queries more reusable.
weight: 104
menu:
  influxdb_cloud_dedicated:
    name: Parameterized queries
    parent: Query data
influxdb/cloud-dedicated/tags: [query, security, influxql]
---

Parameterized queries in {{% product-name %}} let you dynamically and safely change values in a query.
If your application code allows user input to customize values or expressions in a query, use a parameterized query to make sure untrusted input is processed strictly as data and not executed as code.

Parameterized queries:

- help prevent injection attacks, which can occur if input is executed as code
- help make queries more reusable

{{% note %}}
#### Prevent injection attacks

For more information on security and query parameterization,
see the [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html#defense-option-1-prepared-statements-with-parameterized-queries).
{{% /note %}}

In InfluxDB v3, a parameterized query is an InfluxQL or SQL query that contains one or more named parameter placeholders–variables that represent input data–for example, the following query contains a `$temp` parameter:

```sql
SELECT * FROM measurement WHERE temp > $temp
```

When executing a query, you specify parameter name-value pairs.
The value that you assign to a parameter must also be one of the [parameter data types](#parameter-data-types).

```go
{"temp": 22.0}
```

The InfluxDB Querier parses the query with the parameter placeholders, and then generates query plans that replace the placeholders with the values that you provide. This separation of query structure from input data ensures that input is treated as one of the allowed [data types](#parameter-data-types) and not as executable code.

## Parameter data types

In InfluxQL queries, you can use parameters in `WHERE` clause **predicate expressions**, in place of the following data types:

- Null
- Boolean
- Unsigned integer (`u_int64`)
- Integer (`int64`)
- Double (`float64`)
- String

### Time expressions

To parameterize time bounds, substitute a parameter for a timestamp literal--for example:

```sql
SELECT *
FROM home
WHERE time >= $min_time`
```

For the parameter value, specify the timestamp literal as a string--for example:

```go
// Assign a timestamp string literal to the min_time parameter.
parameters := influxdb3.QueryParameters{
    "min_time": "2024-03-18 00:00:00.00",
}
```

### Not supported

Some data types don't support parameters--for example:

- In InfluxQL queries, InfluxDB doesn't support parameter substitution for durations or inside of duration values--for example, the following won’t work:

  ```go
  query := `
      SELECT avg(temp) as temp, room,
        DATE_BIN(INTERVAL '2 hours', time, '1970-01-01T00:00:00Z'::TIMESTAMP) as _time
      FROM home
      WHERE time >= now() - $days
      AND room = $room
      GROUP BY _time, room`

  parameters := influxdb3.QueryParameters{
      "room": "Kitchen",
      "days": "7d",
  }
  ```

## Parametize an SQL query

{{% note %}}
#### Sample data

The following examples use the
[Get started home sensor data](/influxdb/cloud-dedicated/reference/sample-data/#get-started-home-sensor-data).
To run the example queries and return results,
[write the sample data](/influxdb/cloud-dedicated/reference/sample-data/#write-the-home-sensor-data-to-influxdb)
to your {{% product-name %}} database before running the example queries.
{{% /note %}}

To use a parameterized query, do the following:

1.  In your query text, use the `$parameter` syntax to reference a parameter name--for example,
the following InfluxQL query contains `$room` and `$min_temp` parameter placeholders:

```sql
SELECT *
FROM home
WHERE time > now() - 7d
AND temp >= $min_temp
AND room = $room
```

2.  Provide a value for each parameter name.
    If you don't assign a value for a parameter, InfluxDB returns an error.
    The syntax for providing parameter values depends on the client you use--for example:

<!-- Using code-tabs because I expect to add more client examples soon -->

    {{% code-tabs-wrapper %}}
{{% code-tabs %}}
[Go](#)
{{% code-tabs %}}

{{% code-tab-content %}}

```go
// Define a QueryParameters struct--a map of parameters to input values.
parameters := influxdb3.QueryParameters{
    "room": "Kitchen",
    "min_temp": 20.0,
}
```

{{% /code-tab-content %}}
    {{% /code-tabs-wrapper %}}

After InfluxDB receives your request and parses the SQL, it executes the query as

```sql
SELECT *
FROM home
WHERE time > now() - 7d
AND temp >= 20.0
AND room = 'Kitchen'
```

## Execute parameterized SQL queries

{{% note %}}
#### Sample data

The following examples use the
[Get started home sensor data](/influxdb/cloud-dedicated/reference/sample-data/#get-started-home-sensor-data).
To run the example queries and return results,
[write the sample data](/influxdb/cloud-dedicated/reference/sample-data/#write-the-home-sensor-data-to-influxdb)
to your {{% product-name %}} database before running the example queries.
{{% /note %}}

### Use InfluxDB Flight RPC clients

Using the InfluxDB v3 native Flight RPC protocol and supported clients, you can send a parameterized query and a list of parameter name-value pairs.
InfluxDB Flight clients that support parameterized queries pass the parameter name-value pairs in a Flight ticket `params` field.

The following examples show how to use client libraries to execute parameterized InfluxQL queries:

<!-- Using code-tabs because I expect to add more client examples soon -->

{{% code-tabs-wrapper %}}
{{% code-tabs %}}
[Go](#)
{{% code-tabs %}}

{{% code-tab-content %}}

```go
import (
    "context"
    "fmt"
    "io"
    "os"
    "text/tabwriter"
    "time"
    "github.com/apache/arrow/go/v14/arrow"
    "github.com/InfluxCommunity/influxdb3-go/influxdb3"
)

func Query(query string, parameters influxdb3.QueryParameters,
 options influxdb3.QueryOptions) error {
    url := os.Getenv("INFLUX_HOST")
    token := os.Getenv("INFLUX_TOKEN")
    database := os.Getenv("INFLUX_DATABASE")

    // Instantiate the influxdb3 client.
    client, err := influxdb3.New(influxdb3.ClientConfig{
        Host:     url,
        Token:    token,
        Database: database,
    })

    if err != nil {
        panic(err)
    }

    // Ensure the client is closed after the Query function finishes.
    defer func(client *influxdb3.Client) {
        err := client.Close()
        if err != nil {
            panic(err)
        }
    }(client)

    // Call the client's QueryWithParameters function.
    // Provide the query, parameters, and the InfluxQL QueryType option.
    iterator, err := client.QueryWithParameters(context.Background(), query,
    parameters, influxdb3.WithQueryType(options.QueryType))

    // Create a buffer for storing rows as you process them.
    w := tabwriter.NewWriter(io.Discard, 4, 4, 1, ' ', 0)
    w.Init(os.Stdout, 0, 8, 0, '\t', 0)

    fmt.Fprintf(w, "time\troom\tco\thum\ttemp\n")

    // Format and write each row to the buffer.
    // Process each row as key-value pairs.
    for iterator.Next() {
        row := iterator.Value()
        // Use Go arrow and time packages to format unix timestamp
        // as a time with timezone layout (RFC3339 format)
        time := (row["time"].(arrow.Timestamp)).
                ToTime(arrow.Nanosecond).Format(time.RFC3339)

        fmt.Fprintf(w, "%s\t%s\t%d\t%.1f\t%.1f\n",
            time, row["room"], row["co"], row["hum"], row["temp"])
    }
    w.Flush()

    return nil
}

func main() {
    // Use the $placeholder syntax in a query to reference parameter placeholders
    // for input data.
    // The following SQL query contains the placeholders $room and $min_temp.
    query := `
        SELECT *
        FROM home
        WHERE time > now() - 7d
        AND temp >= $min_temp
        AND room = $room`

    // Define a QueryParameters struct--a map that assigns placeholder names to input values.
    parameters := influxdb3.QueryParameters{
        "room": "Kitchen",
        "min_temp": 20.0,
    }

    Query(query, parameters, influxdb3.QueryOptions{
        QueryType: influxdb3.InfluxQL,
    })
}
```

{{% /code-tab-content %}}
{{% /code-tabs-wrapper %}}

## Client support for parameterized queries

- Not all [InfluxDB v3 Flight clients](/influxdb/cloud-serverless/reference/client-libraries/v3/) support parameterized queries.
- InfluxDB doesn't currently support parameterized queries or DataFusion prepared statements for Flight SQL   or Flight SQL clients.
- InfluxDB v3 SQL and InfluxQL parameterized queries aren’t supported in InfluxDB v1 and v2 clients.

## Not supported

Currently, parameterized queries in {{% product-name %}} don't provide the following:

- support for DataFusion prepared statements
- query caching, optimization, or performance benefits