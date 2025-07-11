---
navigation_title: Get started using the API
mapped_pages:
  - https://www.elastic.co/guide/en/elasticsearch/reference/current/rollup-getting-started.html
products:
  - id: elasticsearch
---

# Get started with rollups using the API

::::{admonition} Deprecated in 8.11.0.
:class: warning

Rollups will be removed in a future version. [Migrate](migrating-from-rollup-to-downsampling.md) to [downsampling](/manage-data/data-store/data-streams/downsampling-time-series-data-stream.md) instead.
::::

::::{warning}
From 8.15.0 invoking the put job API in a cluster with no rollup usage will fail with a message about Rollup’s deprecation and planned removal. A cluster either needs to contain a rollup job or a rollup index in order for the put job API to be allowed to execute.
::::


To use the Rollup feature, you need to create one or more "Rollup Jobs". These jobs run continuously in the background and rollup the index or indices that you specify, placing the rolled documents in a secondary index (also of your choosing).

Imagine you have a series of daily indices that hold sensor data (`sensor-2017-01-01`, `sensor-2017-01-02`, etc). A sample document might look like this:

```js
{
  "timestamp": 1516729294000,
  "temperature": 200,
  "voltage": 5.2,
  "node": "a"
}
```


## Creating a rollup job [_creating_a_rollup_job]

We’d like to rollup these documents into hourly summaries, which will allow us to generate reports and dashboards with any time interval one hour or greater. A rollup job might look like this:

```console
PUT _rollup/job/sensor
{
  "index_pattern": "sensor-*",
  "rollup_index": "sensor_rollup",
  "cron": "*/30 * * * * ?",
  "page_size": 1000,
  "groups": {
    "date_histogram": {
      "field": "timestamp",
      "fixed_interval": "60m"
    },
    "terms": {
      "fields": [ "node" ]
    }
  },
  "metrics": [
    {
      "field": "temperature",
      "metrics": [ "min", "max", "sum" ]
    },
    {
      "field": "voltage",
      "metrics": [ "avg" ]
    }
  ]
}
```

We give the job the ID of "sensor" (in the url: `PUT _rollup/job/sensor`), and tell it to rollup the index pattern `"sensor-*"`. This job will find and rollup any index that matches that pattern. Rollup summaries are then stored in the `"sensor_rollup"` index.

The `cron` parameter controls when and how often the job activates. When a rollup job’s cron schedule triggers, it will begin rolling up from where it left off after the last activation. So if you configure the cron to run every 30 seconds, the job will process the last 30 seconds worth of data that was indexed into the `sensor-*` indices.

If instead the cron was configured to run once a day at midnight, the job would process the last 24 hours worth of data. The choice is largely preference, based on how "realtime" you want the rollups, and if you wish to process continuously or move it to off-peak hours.

Next, we define a set of `groups`. Essentially, we are defining the dimensions that we wish to pivot on at a later date when querying the data. The grouping in this job allows us to use `date_histogram` aggregations on the `timestamp` field, rolled up at hourly intervals. It also allows us to run terms aggregations on the `node` field.

::::{admonition} Date histogram interval vs cron schedule
You’ll note that the job’s cron is configured to run every 30 seconds, but the date_histogram is configured to rollup at 60 minute intervals. How do these relate?

The date_histogram controls the granularity of the saved data. Data will be rolled up into hourly intervals, and you will be unable to query with finer granularity. The cron simply controls when the process looks for new data to rollup. Every 30 seconds it will see if there is a new hour’s worth of data and roll it up. If not, the job goes back to sleep.

Often, it doesn’t make sense to define such a small cron (30s) on a large interval (1h), because the majority of the activations will simply go back to sleep. But there’s nothing wrong with it either, the job will do the right thing.

::::


After defining which groups should be generated for the data, you next configure which metrics should be collected. By default, only the `doc_counts` are collected for each group. To make rollup useful, you will often add metrics like averages, mins, maxes, etc. In this example, the metrics are fairly straightforward: we want to save the min/max/sum of the `temperature` field, and the average of the `voltage` field.

::::{admonition} Averages aren’t composable?!
If you’ve worked with rollups before, you may be cautious around averages. If an average is saved for a 10 minute interval, it usually isn’t useful for larger intervals. You cannot average six 10-minute averages to find a hourly average; the average of averages is not equal to the total average.

For this reason, other systems tend to either omit the ability to average or store the average at multiple intervals to support more flexible querying.

Instead, the {{rollup-features}} save the `count` and `sum` for the defined time interval. This allows us to reconstruct the average at any interval greater-than or equal to the defined interval. This gives maximum flexibility for minimal storage costs… and you don’t have to worry about average accuracies (no average of averages here!)

::::


For more details about the job syntax, see [Create {{rollup-jobs}}](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-rollup-put-job).

After you execute the above command and create the job, you’ll receive the following response:

```console-result
{
  "acknowledged": true
}
```


## Starting the job [_starting_the_job]

After the job is created, it will be sitting in an inactive state. Jobs need to be started before they begin processing data (this allows you to stop them later as a way to temporarily pause, without deleting the configuration).

To start the job, execute this command:

```console
POST _rollup/job/sensor/_start
```


## Searching the rolled results [_searching_the_rolled_results]

After the job has run and processed some data, we can use the [Rollup search](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-rollup-rollup-search) endpoint to do some searching. The Rollup feature is designed so that you can use the same Query DSL syntax that you are accustomed to… it just happens to run on the rolled up data instead.

For example, take this query:

```console
GET /sensor_rollup/_rollup_search
{
  "size": 0,
  "aggregations": {
    "max_temperature": {
      "max": {
        "field": "temperature"
      }
    }
  }
}
```

It’s a simple aggregation that calculates the maximum of the `temperature` field. But you’ll notice that it is being sent to the `sensor_rollup` index instead of the raw `sensor-*` indices. And you’ll also notice that it is using the `_rollup_search` endpoint. Otherwise the syntax is exactly as you’d expect.

If you were to execute that query, you’d receive a result that looks like a normal aggregation response:

```console-result
{
  "took" : 102,
  "timed_out" : false,
  "terminated_early" : false,
  "_shards" : ... ,
  "hits" : {
    "total" : {
        "value": 0,
        "relation": "eq"
    },
    "max_score" : 0.0,
    "hits" : [ ]
  },
  "aggregations" : {
    "max_temperature" : {
      "value" : 202.0
    }
  }
}
```

The only notable difference is that Rollup search results have zero `hits`, because we aren’t really searching the original, live data any more. Otherwise it’s identical syntax.

There are a few interesting takeaways here. Firstly, even though the data was rolled up with hourly intervals and partitioned by node name, the query we ran is just calculating the max temperature across all documents. The `groups` that were configured in the job are not mandatory elements of a query, they are just extra dimensions you can partition on. Second, the request and response syntax is nearly identical to normal DSL, making it easy to integrate into dashboards and applications.

Finally, we can use those grouping fields we defined to construct a more complicated query:

```console
GET /sensor_rollup/_rollup_search
{
  "size": 0,
  "aggregations": {
    "timeline": {
      "date_histogram": {
        "field": "timestamp",
        "fixed_interval": "7d"
      },
      "aggs": {
        "nodes": {
          "terms": {
            "field": "node"
          },
          "aggs": {
            "max_temperature": {
              "max": {
                "field": "temperature"
              }
            },
            "avg_voltage": {
              "avg": {
                "field": "voltage"
              }
            }
          }
        }
      }
    }
  }
}
```

Which returns a corresponding response:

```console-result
{
   "took" : 93,
   "timed_out" : false,
   "terminated_early" : false,
   "_shards" : ... ,
   "hits" : {
     "total" : {
        "value": 0,
        "relation": "eq"
     },
     "max_score" : 0.0,
     "hits" : [ ]
   },
   "aggregations" : {
     "timeline" : {
       "buckets" : [
         {
           "key_as_string" : "2018-01-18T00:00:00.000Z",
           "key" : 1516233600000,
           "doc_count" : 6,
           "nodes" : {
             "doc_count_error_upper_bound" : 0,
             "sum_other_doc_count" : 0,
             "buckets" : [
               {
                 "key" : "a",
                 "doc_count" : 2,
                 "max_temperature" : {
                   "value" : 202.0
                 },
                 "avg_voltage" : {
                   "value" : 5.1499998569488525
                 }
               },
               {
                 "key" : "b",
                 "doc_count" : 2,
                 "max_temperature" : {
                   "value" : 201.0
                 },
                 "avg_voltage" : {
                   "value" : 5.700000047683716
                 }
               },
               {
                 "key" : "c",
                 "doc_count" : 2,
                 "max_temperature" : {
                   "value" : 202.0
                 },
                 "avg_voltage" : {
                   "value" : 4.099999904632568
                 }
               }
             ]
           }
         }
       ]
     }
   }
}
```

In addition to being more complicated (date histogram and a terms aggregation, plus an additional average metric), you’ll notice the date_histogram uses a `7d` interval instead of `60m`.

This quickstart should have provided a concise overview of the core functionality that Rollup exposes. There are more tips and things to consider when setting up Rollups, which you can find throughout the rest of this section. You may also explore the [REST API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-rollup-get-jobs) for an overview of what is available.

## Historical-only search example

Suppose you have an index named `sensor-1` that contains raw data, and you've created a {{rollup-job}} with the following configuration:

```console
PUT _rollup/job/sensor
{
  "index_pattern": "sensor-*",
  "rollup_index": "sensor_rollup",
  "cron": "*/30 * * * * ?",
  "page_size": 1000,
  "groups": {
    "date_histogram": {
      "field": "timestamp",
      "fixed_interval": "1h",
      "delay": "7d"
    },
    "terms": {
      "fields": [ "node" ]
    }
  },
  "metrics": [
    {
      "field": "temperature",
      "metrics": [ "min", "max", "sum" ]
    },
    {
      "field": "voltage",
      "metrics": [ "avg" ]
    }
  ]
}
```
% TEST[setup:sensor_index]

This rolls up the `sensor-*` pattern and stores the results in `sensor_rollup`.
To search this rolled up data, use the `_rollup_search` endpoint.
You can use Query DSL to search the rolled-up data:

```console
GET /sensor_rollup/_rollup_search
{
  "size": 0,
  "aggregations": {
    "max_temperature": {
      "max": {
        "field": "temperature"
      }
    }
  }
}
```
% TEST[setup:sensor_prefab_data]
% TEST[s/_rollup_search/_rollup_search?filter_path=took,timed_out,terminated_early,_shards,hits,aggregations/]

The query is targeting the `sensor_rollup` data, since this contains the rollup
data as configured in the job. A `max` aggregation has been used on the
`temperature` field, yielding the following response:

```console-result
GET /sensor_rollup/_rollup_search
{
  "size": 0,
  "aggregations": {
    "max_temperature": {
      "max": {
        "field": "temperature"
      }
    }
  }
}
```
% TESTRESPONSE[s/"took" : 102/"took" : $body.$_path/]
% TESTRESPONSE[s/"_shards" : \.\.\. /"_shards" : $body.$_path/]

The response follows the same structure as a standard query with aggregations: it includes metadata about the request (`took`, `_shards`, etc.), an empty hits section (as rollup searches do not return individual documents), and the aggregation results.

Rollup searches are limited to the functionality defined in the {{rollup-job}} configuration. For example, if the `avg` metric was not configured for the `temperature` field, calculating the average temperature is not possible. Running such a query results in an error:

```console
GET sensor_rollup/_rollup_search
{
  "size": 0,
  "aggregations": {
    "avg_temperature": {
      "avg": {
        "field": "temperature"
      }
    }
  }
}
```
% TEST[continued]
% TEST[catch:/illegal_argument_exception/]

```console-result
{
  "error": {
    "root_cause": [
      {
        "type": "illegal_argument_exception",
        "reason": "There is not a rollup job that has a [avg] agg with name [avg_temperature] which also satisfies all requirements of query.",
        "stack_trace": ...
      }
    ],
    "type": "illegal_argument_exception",
    "reason": "There is not a rollup job that has a [avg] agg with name [avg_temperature] which also satisfies all requirements of query.",
    "stack_trace": ...
  },
  "status": 400
}
```
% TESTRESPONSE[s/"stack_trace": \.\.\./"stack_trace": $body.$_path/]

## Searching both historical rollup and non-rollup data

The rollup search API has the capability to search across both live
non-rollup data and the aggregated rollup data. This is done by adding
the live indices to the URI:

```console
GET sensor-1,sensor_rollup/_rollup_search 
{
  "size": 0, 
  "aggregations": {
    "max_temperature": {
      "max": {
        "field": "temperature"
      }
    }
  }
}
```
% TEST[continued]
% TEST[s/_rollup_search/_rollup_search?filter_path=took,timed_out,terminated_early,_shards,hits,aggregations/]

Note the URI now searches `sensor-1` and `sensor_rollup` at the same time.

When the search is executed, the rollup search endpoint does two things:

1. The original request is sent to the non-rollup index unaltered.
2. A rewritten version of the original request is sent to the rollup index.

When the two responses are received, the endpoint rewrites the rollup response
and merges the two together. During the merging process, if there is any overlap
in buckets between the two responses, the buckets from the non-rollup index are
used.

The response to the above query looks as expected, despite spanning rollup and
non-rollup indices:

```console-result
{
  "took" : 102,
  "timed_out" : false,
  "terminated_early" : false,
  "_shards" : ... ,
  "hits" : {
    "total" : {
        "value": 0,
        "relation": "eq"
    },
    "max_score" : 0.0,
    "hits" : [ ]
  },
  "aggregations" : {
    "max_temperature" : {
      "value" : 202.0
    }
  }
}
```
% TESTRESPONSE[s/"took" : 102/"took" : $body.$_path/]
% TESTRESPONSE[s/"_shards" : \.\.\. /"_shards" : $body.$_path/]
