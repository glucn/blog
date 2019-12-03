# Querying Data in an Array of Nested Structures

*Originally posted at https://vendasta-blog.appspot.com/blog/BL-GM52SWNK/*

Our data models often contain complex data types and nested structures. In this post, I will use an example to discuss how to query the data in an array of nested structures with Elasticsearch, BigQuery, and Cloud SQL (MySQL).

## Background
In the recent sprints, we created the `pending` state in the product activation process. One requirement in the story was an endpoint which can list an app's pending activations which were not approved or rejected.

The data of pending activations lives in the `Activations` model in accounts microservice. This model is keyed with `business_id` and `app_id`, and the field `pending_app_activations` is an array of nested structure `PendingAppActivation` which contains fields like `status`, `activation_id` etc.

The `Activations` model is like this (simplified):

```
{
  business_id: string,
  app_id: string,
  app_activations: [ {...}, {...} ... ],
  pending_app_activations: [
    {
      activation_id: string
      status: string enum ("PENDING", "APPROVED", "REJECTED")
      ...
    }, {...} ...
  ],
  ...
} 
```

To support that endpoint `ListPendingActivations`, we need to lookup the data with given `app_id` and with `PENDING` status, which is in an array of nested structure.

## Elasticsearch
In Elasticsearch, we can query the nested objects with [Nested Query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-nested-query.html). A nested query is executed against the nested objects as if they were indexed as separate docs (they are, internally) and resulting in the parent document. However, a simple nested query on `Activations` model will return the whole model, while we want to know which inner nested `PendingAppActivation` is in `PENDING` status. In this case, we need to define [Inner Hits](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-request-inner-hits.html) in the nested query, which returns the nested hits in addition to the search response. Taking a step further, we can use `_source` field to disable including the parent `Activations` model in the response and solely retrieve the inner hits, this can improve the performance of both data querying and transferring.

At the end of the day, the query ends up like this:
```
GET vstore-t-accou-activ-xxx-elastic/_search
{
  "_source": false, 
  "query": {
    "bool": {
      "filter": [
        {
          "term": {
            "app_id": "RM"
          }
        },
        {
          "nested": {
            "inner_hits": {},
            "path": "pending_app_activations",
            "query": {
              "term": {
                "pending_app_activations.status": "PENDING"
              }
            }
          }
        }
      ]
    }
  }
}
```

And the response will be like the following example. The only concern is that the data that we need hide in the deep layer of the JSON, we need to call `hits.hits.inner_hits.xxxx.hits.hits` to access them.
```
{
  "took": 12,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "failed": 0
  },
  "hits": {
    "total": 10,
    "max_score": 0,
    "hits": [
      {
        "_index": "vstore-t-accou-activ-xxx-elastic",
        "_type": "Activations",
        "_id": "AG-XVBCPWL45L\u001fRM",
        "_score": 0,
        "inner_hits": {
          "pending_app_activations": {
            "hits": {
              "total": 1,
              "max_score": 9.047254,
              "hits": [
                {
                  "_type": "Activations",
                  "_id": "AG-XVBCPWL45L\u001fRM",
                  "_nested": {
                    "field": "pending_app_activations",
                    "offset": 0
                  },
                  "_score": 9.047254,
                  "_source": {
                    "activation_id": "a4ac83ce-818b-43c4-8c64-a14e5b5b0fd2",
                    "status": "PENDING",
                    ...
                  }
                }
              ]
            }
          }
        }
      }
    ]
  }
}
```


## BigQuery
Can we query the data in the same nested structure with BigQuery? The answer is yes, and the query in BigQuery is much more straightforward than in Elasticsearch.


```
SELECT activations.business_id, activations.app_id, pa.activation_id, pa.status
FROM `repcore-prod.vstore_production_accounts.vstore_activations` as activations 
CROSS JOIN
      UNNEST(pending_app_activations) as pa
where pa.status = "PENDING" and activations.app_id = "RM" LIMIT 1000
```
The [UNNEST](https://cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax#unnest) operator takes an array and returns a table, with one row for each element in the array. In this query, we use the `CROSS JOIN` operator to join the table to the UNNEST output of that array column, and it can be ignored. The following query will return the same result:


```
SELECT activations.business_id, activations.app_id, pa.activation_id, pa.status
FROM `repcore-prod.vstore_production_accounts.vstore_activations` as activations,
      UNNEST(pending_app_activations) as pa
where pa.status = "PENDING" and activations.app_id = "RM" LIMIT 1000
```

## Cloud SQL
Cloud SQL is another option for the secondary indexing and vStore uses MySQL version of Cloud SQL. We can also query data in an array of nested structures, but we need to do it in a hacky way.

An array of nested structures is stored in MySQL column as a JSON string. In the version of MySQL supported by Cloud SQL (v5.5 - 5.7), we do not have a built-in function to flatten a JSON array to a table, we have to mimic that operation ourselves:

```
SELECT business_id, app_id, idx, 
	json_unquote(json_extract(pending_app_activations, concat('$[', idx, '].activation_id'))) as activation_id,
	json_unquote(json_extract(pending_app_activations, concat('$[', idx, '].status')) as status
FROM xxxxxx
JOIN ( -- build a integer sequence as long as we need
 select 0 as idx UNION
 select 1 as idx UNION
 select 2 as idx UNION
 select 3 as idx
 ...
) as indexes
WHERE app_id = 'RM'
  AND json_unquote(json_extract(pending_app_activations, concat('$[', idx, '].status')) = 'PENDING'
```

A recent version of MySQL (Community Server 8.0.4-rc) has the `JSON_TABLE()` function, which accepts JSON data and returns it as a relational table whose columns are as specified. But we cannot use that function in Cloud SQL until it supports the newer version MySQL.

## Comparision
All of these three secondary indexes can support our requirement of looking up pending activations in `PENDING` status, but which one is faster?

As `Activation` model is not on Cloud SQL, I chose `SalesOrder` model in sales-orders microservice to carry out a tiny experiment.

At the moment of my experiment, `SalesOrder` model had 373 entities on `TEST`. I tried to query the entities with `product_activations.activation_status = 0`, and there were 159 hits. The time range each secondary index took were:

* Elasticsearch: 12 ~ 200 ms

* BigQuery: 2.4 ~ 3.2 s

* Cloud SQL: 0.1 ~ 0.5 s

Apparently, the performance of Elasticsearch is the best, but on the other hand, Elasticsearch has to be considered unstable due to those outages. The query performance could be a factor we want to consider when we choose a secondary index. But no matter which one is chosen, I hope this post can provide us some idea of how to query data in a complex nested structure.



###### References
* [BigQuery: Working with Arrays](https://cloud.google.com/bigquery/docs/reference/standard-sql/arrays)
* [MySQL: MySQL Community Server 8.0.4-rc](https://lists.mysql.com/announce/1240)
