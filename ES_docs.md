# Elasticsearch documentation

### Get Elasticsearch
Download elasticsearch/kibana/logstash/x-pack at elastic.co/downloads
Most tutorials use the tarball, so if you are on a UNIX based OS I would choose that.

### Elastic products
Elasticsearch is the core package, basically a document DB with awesome (near real time) search functions and a RESTful API. 
Kibana is a tool taht visualizes your elasticsearch cluster, e.g. I/O, status, logs etc. It also has a developer console which can be used instead of using CURL.
Logstash is a "server-side data processing pipline". It can be used to recieve data from elsewhere. It can recieve ES queries, filestreams, STDIN, websocket, http etc.
The full list is at https://www.elastic.co/guide/en/logstash/current/input-plugins.html.

### Elastic terms
Explanation of the basic terms of ES. Adapted from https://www.elastic.co/guide/en/elasticsearch/reference/current/_basic_concepts.html
- Cluster: A collection of one or more nodes that work together. The cluster holds all of the data. All nodes with the same cluster name will form a cluster (as long as the can access each other)
- Node: A single server. It is part of a cluster
- Index: a collection of documents that have simmilar characteristics (i.e. a Collection in MongoDB)
- Type: Within an index you can have one or more types. If we were to move Mijn Kwik to ES, mijn-kwik.experiments would be a type.
- Document: Same thing as in any Document DB. It is the basic unit of data, in JSON format
- Shards and replicas: It is possible to horizontally split an index. The parts of teh index are then called shards. Shards are nice because they allow to split content volume and to distribute and paralellize operations.
Replicas are nice because they are basically a backup of your data that can be used to search the same content in parallel.

### ES actions
All the actions are in CURL format
#### Create Index
`curl -XPUT "localhost:9200/customer?pretty"`
Creates the index `customer`, and displays the output in JSON pretty format.

#### Put something in the index
`curl -XPUT 'localhost:9200/customer/external/1?pretty&pretty' -H 'Content-Type: application/json' -d '{"name": "John Doe"}'`
Creates the 'external' type, and puts a document with ID 1 in it.

#### Delete an index
`curl -XDELETE 'localhost:9200/customer?pretty&pretty'`
Works the same for documents etc.

#### Update document
`curl -XPOST 'localhost:9200/customer/external/1/_update?pretty&pretty' -H 'Content-Type: application/json' -d '{"doc": { "name": "Jane Doe", "age": 20}}'`
Updates the ID 1 document, changing the name field, and inserting an age.

`curl -XPOST 'localhost:9200/customer/external/1/_update?pretty&pretty' -H 'Content-Type: application/json' -d '{"script" : "ctx._source.age += 5"}'`
Updates age with a script

#### Using the bulk API
`'''curl -XPOST 'localhost:9200/customer/external/_bulk?pretty&pretty' -H 'Content-Type: application/json' -d'
{"index":{"_id":"1"}}
{"name": "John Doe" }
{"index":{"_id":"2"}}
{"name": "Jane Doe" }
' '''`
#### Loading a dataset
`curl -H "Content-Type: application/json" -XPOST 'localhost:9200/bank/account/_bulk?pretty&refresh' --data-binary "@accounts.json"`
Creates the bank index, with the account type and load the data from accounts.json

#### Define a mapping
"""PUT /example
{
  "mappings": {
    "tracking": {
      "properties": {
        "plugins": {
          "type": "nested", 
          "properties": {
            "name": { "type": "keyword"},
            "url": { "type": "keyword"},
            "version": { "type": "keyword"},
            "author": { 
              "properties": {
                "name": {"type": "keyword"},
                "url": {"type": "keyword"}
              }
            }
          }
        }
      }
    }
  }
}"""

#### Sample queries
##### `match_all`
curl -XGET 'localhost:9200/bank/_search?pretty' -H 'Content-Type: application/json' -d'
{
  "query": { "match_all": {} },
  "size": 20,
  "from": 11,
  "sort": [
    { "account_number": "asc" }
  ]
}
'
Match all query with ascending sort, returning 20 results (10 is default), starting from the 12th document. 
##### SQL SELECT * FROM plugins.name;

GET /example/_search
{
  "query": { "match_all": {} },
  "_source": ["plugins.name"]
}

##### Bool
curl -XGET 'localhost:9200/bank/_search?pretty' -H 'Content-Type: application/json' -d'
{
  "query": {
    "bool": {
      "must": [
        { "match": { "age": "40" } }
      ],
      "must_not": [
        { "match": { "state": "ID" } }
      ]
    }
  }
}
'
In SQL: SELECT * FROM bank WHERE age = 40 AND state != "ID"

##### filter
curl -XGET 'localhost:9200/bank/_search?pretty' -H 'Content-Type: application/json' -d'
{
  "query": {
    "bool": {
      "must": { "match_all": {} },
      "filter": {
        "range": {
          "balance": {
            "gte": 20000,
            "lte": 30000
          }
        }
      }
    }
  }
}
'

##### Aggregations

curl -XGET 'localhost:9200/bank/_search?pretty' -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "aggs": {
    "group_by_age": {
      "range": {
        "field": "age",
        "ranges": [
          {
            "from": 20,
            "to": 30
          },
          {
            "from": 30,
            "to": 40
          },
          {
            "from": 40,
            "to": 50
          }
        ]
      },
      "aggs": {
        "group_by_gender": {
          "terms": {
            "field": "gender.keyword"
          },
          "aggs": {
            "average_balance": {
              "avg": {
                "field": "balance"
              }
            }
          }
        }
      }
    }
  }
}
'

Size 0 lets you output only teh results from teh aggregation, if size > 0: it will also output results from the query (in this example there is no query)

#### Snapshots
Snapshot guide at elastic.co:
https://www.elastic.co/guide/en/elasticsearch/reference/master/modules-snapshots.html

Domain management of AWS ES service:
http://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/es-managedomains.html
Features docs for setting up and restoring manual snapshots
Snapshots at AWS ES:
http://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/es-createupdatedomains.html#es-createdomain-configure-snapshots

Create snapshot:
`curl -XPUT 'http://localhost:9200/_snapshot/es-snapshot/20170725'`

Check snapshot:
`curl -XGET 'http://localhost:9200/_snapshot/es-snapshot/_all?pretty'`

#### Import/export dashboards/visualisations
https://discuss.elastic.co/t/how-to-save-dashboard-as-json-file/24561/3

Go to management tab > saved objects
Here you can Export or import objects

