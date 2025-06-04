# opensearch

## Steps to run the project

```
git clone https://github.com/pavansachi/opensearch.git

change the OPENSEARCH_INITIAL_ADMIN_PASSWORD and set a strong password. 
It looks like security is enabled by default in latest versions.

docker-compose up -d

docker-compose down
```

## Dev tools
Dev tools can be used to run the REST queries on opensearch instance 
[Dev Tools](http://localhost:5601/app/dev_tools#/console)


## Index mapping

```
PUT /works
{
  "settings": {
    "analysis": {
      "filter": {
        "synonym_filter": {
          "type": "synonym",
          "synonyms": [
            "sticker, sticker pack",
            "classic t-shirt, t-shirt, tee",
            "coasters, coaster",
            "clock, timepiece",
            "pullover hoodie, hoodie, pullover, sweatshirt"
          ]
        }
      },
      "analyzer": {
        "synonym_analyzer": {
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "synonym_filter"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "work_id": { "type": "long" },
      "title": {
        "type": "text",
        "analyzer": "standard"
      },
      "tags": {
        "type": "text",
        "analyzer": "standard"
      },
      "products": {
        "type": "nested",
        "properties": {
          "product_id": {
            "type": "keyword"
          },
          "name": {
            "type": "text",
            "analyzer": "synonym_analyzer"
          },
          "stage": {
            "type": "keyword"
          },
          "enabled": {
            "type": "boolean"
          }
        }
      }
    }
  }
}
```

## Data

```
POST /works/_doc
{
    "work_id": 274839,
    "title": "Retro Sticker Collection",
    "tags": ["stickers", "vintage", "fun"],
    "products": [
      { "product_id": "sticker", "name": "sticker", "stage": "Active", "enabled": true },
      { "product_id": "coaster", "name": "coasters", "stage": "Active", "enabled": true }
    ]
  }
  
POST /works/_doc
{
    "work_id": 593210,
    "title": "Clockwork Jungle",
    "tags": ["clock", "jungle", "time"],
    "products": [
      { "product_id": "clock", "name": "clock", "stage": "Active", "enabled": true },
      { "product_id": "tshirt", "name": "classic t-shirt", "stage": "Active", "enabled": true },
      { "product_id": "hoodie", "name": "Pullover Hoodie", "stage": "Active", "enabled": true }
    ]
  }
  
POST /works/_doc
{
    "work_id": 862147,
    "title": "Monochrome Tee Drop",
    "tags": ["fashion", "shirt", "minimal"],
    "products": [
      { "product_id": "tshirt", "name": "classic t-shirt", "stage": "Active", "enabled": true },
      { "product_id": "sticker", "name": "sticker", "stage": "Active", "enabled": true },
      { "product_id": "hoodie", "name": "Pullover Hoodie", "stage": "Active", "enabled": true }
    ]
  }

```

## Opensearch query

```
GET /works/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "bool": {
            "should": [
              { "match": { "title": "jungle" }},
              { "match": { "tags": "jungle" }},
              {
                "nested": {
                  "path": "products",
                  "query": {
                    "bool": {
                      "must": [
                        { "match": { "products.name": "jungle" }},
                        { "term": { "products.enabled": true }}
                      ]
                    }
                  }
                }
              }
            ]
          }
        },
        {
          "bool": {
            "should": [
              { "match": { "title": "hoodie" }},
              { "match": { "tags": "hoodie" }},
              {
                "nested": {
                  "path": "products",
                  "query": {
                    "bool": {
                      "must": [
                        { "match": { "products.name": "hoodie" }},
                        { "term": { "products.enabled": true }}
                      ]
                    }
                  }
                }
              }
            ]
          }
        }
      ]
    }
  }
}

{
  "took": 8,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
      "value": 1,
      "relation": "eq"
    },
    "max_score": 6.7253904,
    "hits": [
      {
        "_index": "works",
        "_id": "PB7COJcBErKXZ2PbiRd-",
        "_score": 6.7253904,
        "_source": {
          "work_id": 593210,
          "title": "Clockwork Jungle",
          "tags": [
            "clock",
            "jungle",
            "time"
          ],
          "products": [
            {
              "product_id": "clock",
              "name": "clock",
              "stage": "Active",
              "enabled": true
            },
            {
              "product_id": "tshirt",
              "name": "classic t-shirt",
              "stage": "Active",
              "enabled": true
            },
            {
              "product_id": "hoodie",
              "name": "Pullover Hoodie",
              "stage": "Active",
              "enabled": true
            }
          ]
        }
      }
    ]
  }
}

```
`Note`: The /_search query provides all the metadata and its responsibility of the client code to filter the hits and total.
This query does not support partial match but can be improvised with the autocomplete filters.

## Solution

### implementing the token-level matching described.
- The approach is to split the search token “jungle hoodie” and try to match for each token, the specific fields like tags, title and products.name (nested). The inner bool ensures one of the tags matches and external bool ensures both token matches somewhere. This ensures a single document is checked for both tokens across the fields.

### Token matching logic and how synonyms are applied
- Elasticsearch analyzes text fields using an analyzer. it breaks the text into tokens, apply lowercase tokens, remove punctuations etc. At query time, the text is again analyzed and broken into tokens, and matched against documents.
- synonyms are configured in index mappings. At time of index, it indexes synonyms and tokens in the document

### Handling nested products and enabling filtering
- fields like `products` that contain arrays of objects should be mapped as nested types. to preserve the relationship between fields inside each product object. By default elastic flattens the data and we lose the relationship. So while indexing we should create a nested type and also efficient for filtering.

### How relevance scoring might work and possible improvements
- there are algorithms like TF-IDF , where it checks how often a word or term appears in a document and how rare is term across documents. if a term matches in a field like **title**, it could have higher score compared to a longer field like **description**.
- in nested types scoring only considers matches within the nested object to preserve relevance accuracy
- **improvements** - field level boosting can be used at query time for fields which have higher relevance . for example like **title**

### What would you improve on the next iteration of this index.
- use a synonym_path to specify a file to reference synonyms and keeps the index mapping clean. One drawback is it needs reindexing.
- use search analyzer at query time instead of applying synonym filter at index time. this also reduces index size and cost
- use keyword type for fields like **title** if exact matching is needed. full text search can be resource intensive and adds to index cost

### What tools/utilities did you use as part of your process?
- docker
- docker-compose
- docker image - **opensearchproject/opensearch:2.13.0**
- Dev tools from opensearch for rest operations
- https://docs.opensearch.org/docs/latest/api-reference

### What scale of system would this support and what would you monitor to know when it needed to be reworked?
- this should scale well for moderate data sizes with hundred and thousands of docs upto a million
- standard text analysers should scale well with right sized cluster
- boolean and keyword are light weight and scale well. they are useful for filtering but not scoring
- metrics like query latency, indexing throughput, index size, disk utilization, heap memory usage and GC pauses, cluster and node health needs to be monitored

### How would you scale the OS cluster if we had 1 million times documents or if each document was 10 x size
- Use dedicated master and stable data nodes in a autoscale mode
- Calculate scale factor based on document size and count, document size * count, for example; 1M * 10KB = 10GB ~ 15GB
- Start with a baseline node count and scale with Load
- Upgrade node resources: RAM, CPU, and Disk I/O
- Maintain optimal JVM Heap: Under 50% of RAM
- Horizontally scale on sustained high heap usage

### How would you scale to accommodate a spike requests
- Horizontally Scale with Auto-Scaling Policies
- Implement robust load balancing
- Leverage filter context for query caching
- Increase default search timeouts with care
- Optimize queries for performance
- Implement proactive monitoring and Alerting