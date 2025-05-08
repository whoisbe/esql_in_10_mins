The Elasticsearch Query Language (ESQL) provides a powerful way to filter, transform, and analyze data stored in Elasticsearch. In this tutorial, we will learn how to use ESQL in under 10 minutes.

Prerequisites for following along on your own local development environment are
 - If you’re using Microsoft Windows, then install [Windows Subsystem for Linux](https://learn.microsoft.com/en-us/windows/wsl/install) (WSL).
 - If you don’t have Docker installed, download and install [Docker Desktop](https://www.docker.com/products/docker-desktop) for your operating system.

If your local machine meets these requirements then you can begin by cloning this repository, downloading the data file and  loading it into elasticsearch using the convenience script provided in the scripts directory.
```
curl -fsSL https://elastic.co/start-local | sh
```
This script creates an elastic-start-local folder containing configuration files and starts both Elasticsearch and Kibana using Docker.

After the script completes running successfully, it will print out the Kibana endpoint and login credentials for elastic superuser account. 
Access the Kibana endpoint at http://localhost:5601 and login with the user_id "elastic" and the password printed out in the terminal.


Next, download the mocked up webserver log dataset from [here](https://drive.google.com/file/d/1wLhAtIh4IWPRvA0z0Qb_UQ6Kzz5KqwXc/view?usp=drive_link)

In the home screen, Click on "Upload a file" and select the downloaded weblogs.ndjson file. Scroll down and click "Import". In the import data section, switch to the "Advanced" tab. Enter "weblogs" for the index name and check "Create data view". Replace the content of the "Mappings" section with the one below.


```json
{
    "properties": {
      "@timestamp": {
        "type": "date"
      },
      "client.geo.country_iso_code": {
        "type": "keyword"
      },
      "client.geo.location": {
        "type": "geo_point"
      },
      "client.ip": {
        "type": "ip"
      },
      "file.extension": {
        "type": "keyword"
      },
      "host.hostname": {
        "type": "keyword"
      },
      "http.request.referer": {
        "type": "keyword"
      },
      "http.response.bytes": {
        "type": "long"
      },
      "http.response.status_code": {
        "type": "keyword"
      },
      "machine.ram": {
        "type": "long"
      },
      "server.geo.country_iso_code": {
        "type": "keyword"
      },
      "server.ip": {
        "type": "ip"
      },
      "server.memory": {
        "type": "long"
      },
      "server.phpmemory": {
        "type": "long"
      },
      "tags": {
        "type": "keyword"
      },
      "timestamp": {
        "type": "date_nanos",
        "format": "iso8601"
      },
      "url.full": {
        "type": "keyword"
      },
      "url.path": {
        "type": "keyword"
      },
      "user_agent" : {
        "properties" : {
          "device" : {
            "properties" : {
              "name" : {
                "type" : "keyword"
              }
            }
          },
          "name" : {
            "type" : "keyword"
          },
          "original" : {
            "type" : "text",
            "fields" : {
              "keyword" : {
                "type" : "keyword",
                "ignore_above" : 256
              }
            }
          },
          "os" : {
            "properties" : {
              "full" : {
                "type" : "keyword"
              },
              "name" : {
                "type" : "keyword"
              },
              "version" : {
                "type" : "keyword"
              }
            }
          },
          "version" : {
            "type" : "keyword"
          }
        }
      }
    }
  }
```

Click import. After file upload is complete, click "View index in Discover". 

### Explore

This is Kibana's Discover App. It supports another query language called Kibana Query Language (KQL) by default. But we can switch to using ES|QL. But before we do that, let's make sure that we select the appropriate time frame for our dataset. select **weblogs** from the **Data view** drop down. This data set spans from *Dec 05, 2024* to *Feb 04, 2025*. Select this time frame using the time filter on the top right. Click on "Last 15 minutes" and switch to Absolute tab. Set the start datetime to *Dec 05, 2024 @ 00:00:00.000* and the end datetime (click on **Now > Absolute**) to *Feb 4, 2025 @ 00:00:00.000*. You should have a total of 50,009 documents. Now Click **Try ESQL** link up on top to switch to ESQL.

> You can either copy and paste ESQL snippets from here or type in the queries into Kibana's query bar in Discover and execute them (Command / Ctrl + Enter or click play).

You will get the following default query on the query bar.

```
from weblogs | LIMIT 10
```

Notice that there is no **Data view** dropdown anymore. FROM is a source command that can pull data from sources such as an elasticsearch [index](https://www.elastic.co/guide/en/elasticsearch/reference/current/documents-indices.html), [datastream](https://www.elastic.co/guide/en/elasticsearch/reference/current/data-streams.html) or [alias](https://www.elastic.co/guide/en/elasticsearch/reference/current/aliases.html).

 > Tip: You can use a combination of these sources using comma seperated values together with splat (e.g., `logs-*, metrics-*`) patterns.

 The output of the SOURCE command is piped to the LIMIT command. The LIMIT command limits the number of documents to the most recent 10 events. Remove the LIMIT and run the query again.

```
from weblogs
```

When LIMIT is omitted in a query, a default limit of 1000 will be applied by Kibana but you can use an explicit LIMIT command to limit the number of rows that are returned, up to a maximum of 10,000 rows. Notice that the output table contains the timestamp field and a summary field. The sidebar on the left contains list of fields in this data set. Explore this list and notice different icons that represent different data types.

We can pipe the output of the LIMIT command to the KEEP command and specify fields we want in the table.

```
FROM weblogs
| LIMIT 10
| KEEP @timestamp, url.full, client.geo.country_iso_code
```
> Tip: You can collapse the sidebar containing fields list to get more real estate for the output table.

KEEP command also supports splat patterns

```
FROM weblogs
| WHERE server.memory > 0
| LIMIT 10
| KEEP client*, server*, *geo.country_iso_code
```
You can drop one or more columns using the DROP processing command.
```
FROM weblogs
| LIMIT 10
| KEEP client*, server*, *geo.country_iso_code
| DROP server.memory, server.phpmemory
```
Data can be filtered using the WHERE command.
```
FROM weblogs
| WHERE user_agent.original.keyword LIKE "*Mozilla*"
| LIMIT 10
| KEEP client.ip, user_agent.original.keyword
```
There are several [operators](https://www.elastic.co/guide/en/elasticsearch/reference/current/esql-functions-operators.html#esql-operators) that can be used with the WHERE command. One such operator that WHERE command supports is the [LIKE](https://www.elastic.co/guide/en/elasticsearch/reference/current/esql-functions-operators.html#esql-like-operator) operator that you can use to filter data based on string patterns using wildcards. Both \* and \? can be used.

 > Tip: If you need to use a regular expressions instead, you can use the [RLIKE](https://www.elastic.co/guide/en/elasticsearch/reference/current/esql-functions-operators.html#esql-rlike-operator) operator.

Another operator that allows testing whether a field or expression equals an element in a list is the [IN](https://www.elastic.co/guide/en/elasticsearch/reference/current/esql-functions-operators.html#esql-in-operator) operator. If the value of the file.extension field is found in the list, then the row will qualify to be passed on to the next command.
```
FROM weblogs
| WHERE file.extension IN ("gz", "zip", "deb", "rpm")
| LIMIT 10
| KEEP @timestamp, url.full, client.geo.country_iso_code
```
Values can be calculated on the fly using the EVAL command. Here the EVAL command evaluates an expression that simply divides the http.response.bytes field by 1024 to get the kilobyte value.
```
FROM weblogs
| WHERE file.extension IN ("gz", "zip", "deb", "rpm")
| LIMIT 10
| EVAL kb = http.response.bytes/1024
| KEEP @timestamp, url.full, client.geo.country_iso_code, kb
```
Notice that the evaluated field is explicitly specified in the KEEP command. EVAL also supports various [functions](https://www.elastic.co/guide/en/elasticsearch/reference/current/esql-functions-operators.html#esql-functions) to work with the data. Here is an example that includes an EVAL command using a STARTS_WITH function to check if the http referer is a secure website or not.
```
FROM weblogs
| WHERE file.extension IN ("gz", "zip", "deb", "rpm")
| LIMIT 10
| EVAL kb = http.response.bytes/1024
| EVAL tls = STARTS_WITH(http.request.referer, "https")
| KEEP @timestamp, url.full, client.geo.country_iso_code, kb, tls
```
If you would rather extract the protocol (http or https) from the referer field, You can rewrite the newly added EVAL expression like this.
```
FROM weblogs
| WHERE file.extension IN ("gz", "zip", "deb", "rpm")
| LIMIT 10
| EVAL kb = http.response.bytes/1024
| EVAL protocol = MV_FIRST(SPLIT(http.request.referer, ":"))
| KEEP @timestamp, url.full, client.geo.country_iso_code, kb, protocol
```
This version uses a combination of SPLIT function that splits a string by a specified delimiter and extracts the first of the multi-value field generated using MV_FIRST. There are many [multi-value functions](https://www.elastic.co/guide/en/elasticsearch/reference/current/esql-functions-operators.html#esql-mv-functions) supported to work with multi-valued fields and outputs.

We can change the default sort order (time field) using the SORT command that will give us top 10 events with the most byte transfer.
```
FROM weblogs
| WHERE file.extension IN ("gz", "zip", "deb", "rpm")
| SORT http.response.bytes DESC
| LIMIT 10
| EVAL kb = http.response.bytes/1024
| EVAL tls = MV_FIRST(SPLIT(http.request.referer, "/")) == "https"
| KEEP @timestamp, url.full, client.geo.country_iso_code, kb, tls
```
Now we have events with most kilobyte transfers. Note that SORT is applied before the limit command. If the order is reversed, you would get a totally different result.

You can sort on multiple columns on ascending or descending values. By default, null values are treated as being larger than any other value.
```
FROM weblogs | KEEP host.hostname, server.memory | SORT server.memory DESC NULLS LAST
```
With an ascending sort order, null values are sorted last, and with a descending sort order, null values are sorted first. You can change that by providing NULLS FIRST or NULLS LAST.

### Statistics

We can also calculate statistics using the STATS command. This example calculates value for this new field avg_kb for the entire dataset in the specified time range.

```
FROM weblogs
| EVAL kb = http.response.bytes / 1024.0
| STATS avg_kb = AVG(kb)
```

The STATS command supports various other [aggregate functions](https://www.elastic.co/guide/en/elasticsearch/reference/current/esql-functions-operators.html#esql-agg-functions) like SUM, COUNT, COUNT_DISTINCT, MAX, MEDIAN and others.

This can also be rewritten like this to include the kilobyte calculation directly inside the AVG function
```
FROM weblogs
| STATS avg_kb = AVG(http.response.bytes / 1024.0)
```
STATS command supports calculating multiple values.
```
FROM weblogs
| STATS avg_kb = AVG(http.response.bytes / 1024.0), sum_kb = SUM(http.response.bytes / 1024.0)
```
You can use the ROUND function to round off the decimals to the specified number of places.
```
FROM weblogs
| STATS avg_kb = ROUND(AVG(http.response.bytes / 1024.0),2), sum_kb = ROUND(SUM(http.response.bytes / 1024.0), 2)
```
If you prefer to round the value to the nearest integer you can use the FLOOR or CEIL function.
You can group the calculated statistical values by categorical values in other columns using **BY**.
```
FROM weblogs
| STATS avg_kb = AVG(http.response.bytes / 1024.0), sum_kb = SUM(http.response.bytes / 1024.0) BY client.geo.country_iso_code
| SORT avg_kb DESC
| LIMIT 10
```
Note that we are sorting by the computed statistical value before we apply the limit. The default visualization above the results table keeps both calculated values on the same vertical axis. You can edit the visualization to have them in separate vertical axes. You can do this by clicking on the *edit* icon and adjusting [Lens](https://www.elastic.co/guide/en/kibana/current/lens.html) configuration.
If you'd like to group by values in a numeric field or a datetime field, you can use the BUCKET grouping function. Here the same values are being calculated for every week in the time series dataset.
```
FROM weblogs
| STATS avg_kb = AVG(http.response.bytes / 1024.0), sum_kb = SUM(http.response.bytes / 1024.0) BY week = BUCKET(@timestamp, 1 week)
| SORT avg_kb DESC
| LIMIT 10
```
BUCKET also supports a 4 parameter mode where you can specify the most number of groups you want and a range.
```
FROM weblogs
| STATS avg_kb = AVG(http.response.bytes / 1024.0), sum_kb = SUM(http.response.bytes / 1024.0) BY week = BUCKET(@timestamp, 15, "2024-12-01", "2025-01-30")
| SORT avg_kb DESC
| LIMIT 10
```
Here is another example of grouping by an evaluated numeric field using the BUCKET grouping function.
```
FROM weblogs
| EVAL kb = http.response.bytes / 1024
| STATS COUNT(*) BY BUCKET(kb, 2000.0)
| RENAME `COUNT(*)` AS event_count, `BUCKET(kb, 2000.0)` AS kb_range
| SORT kb_range DESC | KEEP kb_range, event_count
```
Once again the default visualization can be adjusted as needed. Notice the column name for the groups generated in the results is `BUCKET(kb, 2000.0)`. The default column name generated can be renamed using the RENAME command but it has to be surrounded by backticks when referenced because it contains special characters.
STATS can also be used with the WHERE clause to apply filters to the data that goes into the aggregation.
```
FROM weblogs
| EVAL kb = http.response.bytes / 1024
| STATS small = COUNT(*) WHERE kb < 5000,
medium = COUNT(*) WHERE kb >= 5000 AND kb < 10000,
large = COUNT(*) WHERE kb >= 10000
```
This gives you more control over the ranges in which data is aggregated for.

You can also produce the exact same results using a CASE function in an EVAL command and then using STATS to group by the values in the field.
```
FROM weblogs
| EVAL kb = http.response.bytes / 1024
| EVAL kb_range = CASE(kb < 5000, "small", kb >= 5000 AND kb < 10000, "medium", kb > 10000, "large")
| STATS COUNT(*) BY kb_range
```
### Processing
While the FROM source command is used to retrieve data from Elasticsearch, another source command ROW produces a row with one or more columns with values that are specified inline.
```
ROW @timestamp = "2099-11-15T13:12:00", message = "GET /search HTTP/1.1 200 1070000", user_id = "kimchy"
| KEEP @timestamp, message, user_id
```
This can be helpful for testing processing commands.

Suppose we have a row like this
```
ROW a = "2023-01-23T12:15:00.000Z 127.0.0.1 some.email@foo.com 42"
```
And we need to extract the timestamp, ip, email and a number that follows from this, we can use either the dissect or grok processing commands to do just that. Here is the row processed using the [DISSECT](https://www.elastic.co/guide/en/elasticsearch/reference/current/esql-commands.html#esql-dissect) processor.
```
ROW a = "2023-01-23T12:15:00.000Z 127.0.0.1 some.email@foo.com 42"
| DISSECT a "%{@timestamp} %{ip} %{email} %{number}"
| KEEP @timestamp, ip, email, number
```
DISSECT works by breaking up a string using a delimiter-based pattern. DISSECT works well when data is reliably repeated.

Here is the same row processed using [GROK](https://www.elastic.co/guide/en/elasticsearch/reference/current/esql-commands.html#esql-grok).
```
ROW a = "2023-01-23T12:15:00.000Z 127.0.0.1 some.email@foo.com 42"
| GROK a """%{TIMESTAMP_ISO8601:date} %{IP:ip} %{EMAILADDRESS:email} %{NUMBER:number}"""
| KEEP date, ip, email, number
```
GROK works similarly, but uses regular expressions under the hood. This makes GROK much more powerful, but generally also slower. If you want to learn more about ready made reusable patterns that are available in Grok, check out the GitHub link in the description.

You can also collect multiple values into the same field by specifying the field name multiple times.
```
ROW a = "2023-01-23T12:15:00.000Z 127.0.0.1 some.email@foo.com 42 43 44 45"
| GROK a """%{TIMESTAMP_ISO8601:date} %{IP:ip} %{EMAILADDRESS:email} %{NUMBER:number} %{NUMBER:number} %{NUMBER:number} %{NUMBER:number}"""
| KEEP date, ip, email, number
```
GROK will create a multi-valued column.


If you have lookup data prepared in elasticsearch using an enrich policy, you can perform a lookup based on field values in your source table. For example if we had an index with the following data we can use that to lookup the owner for hostnames in our logs

| hostname                       | owner         |
| -------------------------------| ------------- |
|www.elastic.co                  | Chirag Shetty |
|artifacts.elastic.co            | Aaron Chia    |
|cdn.elastic-elastic-elastic.org | Kim Astrup    |
|elastic-elastic-elastic.org.    | Mohammad Ahsan|

Let's create such an enrich index that we can use as a look up table. Click on Kibana's Main Menu and click on *Dev Tools*. Copy and paste the following and execute them (CMD/CTRL+ENTER).

First, we will create the schema for our contextual index.

```json
PUT servers
{
  "mappings": {
    "properties": {
      "hostname": {
        "type": "keyword"
      },
      "owner": {
        "type": "keyword"
      }
    }
  }
}
```
Next, we will index lookup data

```json
POST servers/_doc
{
  "hostname": "artifacts.elastic.co",
  "owner": "Aaron Chia"
}

POST servers/_doc
{
  "hostname": "cdn.elastic-elastic-elastic.org",
  "owner": "Kim Astrup"
}

POST servers/_doc
{
  "hostname": "www.elastic.co",
  "owner": "Chirag Shetty"
}

POST servers/_doc
{
  "hostname": "elastic-elastic-elastic.org",
  "owner": "Mohammad Ahsan"
}
```
This is how we create an enrich policy that specifies the index where the lookup data is, the field to lookup and the list of enrich fields.

```json
PUT _enrich/policy/server2owner_policy
{
  "match": {
    "indices": "servers",
    "match_field": "hostname",
    "enrich_fields": ["owner"]
  }
}
```
Finally, you have to execute the policy to be able to use the look up data in ES|QL using the ENRICH command

```bash
POST _enrich/policy/server2owner_policy/_execute
```

Each of the unique values for host.hostname field is matched up to a fictitious owner value. You can now enrich the weblogs data with the owner values in this lookup table using the [ENRICH](https://www.elastic.co/guide/en/elasticsearch/reference/current/esql-commands.html#esql-enrich) command.
```
FROM weblogs
| ENRICH server2owner_policy ON host.hostname WITH owner
| STATS COUNT(*) BY host.hostname , owner
```
You don't need to use the WITH operator unless you want to pick specific fields from the list of fields specified as enrich fields in the policy.

 > Preparing an enrich index that serves as a lookup table is performed outside the ES|QL context. Refer to the detailed documentation on [Data enrichment](https://www.elastic.co/guide/en/elasticsearch/reference/current/esql-enrich-data.html) for more information


Congratulations! You have learnt the most commonly used ES|QL syntax and query patterns that you can apply to your own data. Checkout Elasticsearch [ES|QL documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/esql.html) for an in-depth coverage of ES|QL syntax.
