根据`query_string`查询某段时间的Index

```
查询2018-01-23这一天：
{
  "query": {
    "bool": {
      "must": [
        {
          "query_string": {
            "query": "..."
          }
        },
        {
          "range": {
            "@timestamp": {
              "gte": "2018-01-23T00:00:00Z",
              "lte": "2018-01-23T23:59:59Z"
            }
          }
        }
      ]
    }
  }
}

```