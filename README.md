# Elasticsearch Vector Scoring

A spike to demonstrate the use of the [Fast Cosine ES plugin from StaySense](https://github.com/StaySense/fast-cosine-similarity) on top of Elasticsearch 6.4.1. 

===================

####  1. Build the container
```
docker build -t es-knn .
```

####  2. Run the container
```
docker run -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" es-knn
```

####  3. Create index, apply mapping and settings
```
curl -H 'Content-Type: application/json' -X PUT "http://localhost:9200/products" -d '
{
  "settings":{                
    "index": {
      "similarity": {
        "default": {
          "type": "BM25"
        }
      },
      "analysis": {
        "analyzer": {
          "light_english": {
            "type": "custom",
            "tokenizer": "standard",
            "filter": [
              "standard",
              "lowercase",
              "english_stop",
              "apostrophe",
              "light_english"
            ]
          }
        },
        "filter": {
          "light_english": {
            "type": "stemmer",
            "name" : "light_english"
          },
          "english_stop": {
            "type":      "stop",
            "stopwords": "_english_"
          }
        }
      }
    }
  },
  "mappings": {
    "product": {
      "properties": {
        "embeddedVector": {
          "type": "binary",
          "doc_values": true
        },
        "product_id": {
          "type": "keyword"
        },
        "product_url": {
          "type": "keyword"
        },
        "title": {
          "type": "text",
          "analyzer": "light_english"
        },
        "gross_price": {
          "type": "integer"
        }
      }
    }
  }
}'
```

View the mapping:
```
http://localhost:9200/products/_mapping
```

View the settings:
```
http://localhost:9200/products/_settings
```


#### 4. Add products to the index

Use the 'seeding' script
```
chmod +x bin/seed
```

```
$ bin/seed
```

...or add an individual product.
```
curl -H 'Content-Type: application/json' -X POST 'http://localhost:9200/products/product' -d '
{
  "embeddedVector": "QNdxmRRuthhAyKm7bNW+okC/b6ErRxNeQMkVfl1L+edAytg+Xy4UekDJThkCWB33QIvCavUhc1BA3DO7m1HKOUDCYABjTPXGQNbxSnz6OdNAnL59l6QCP0DSeDY6ZOY8QMjSsHZw54hA24dDthdhH0DAsj5qOy/QQMdsC9vJbiJAyS55LjB5zUDZNqJhLjJEQNV9iRoTCBFA2E87SCpGiEDa3Kb5pdYMQJjOGUeMQmNAx/qBQ2fvVUDLX+EAIZHHQLDdvMCEwb9AvrO28HbudkDcdrRSbSG2QNkykLOxRxtA1CEEod/Q5EDS+JCvf5jLQJSZCPtNNYRA2CJ6PuO/S0DXxDLSaOSwQNXSoGf1LKBAwzrARzHWjkDNnLWo59MhQIzMzAVRx/xAtlasoaIv30DCcwYuscdrQHyFmW42GxVAuovNyYVxwUCgw+sQLDeyQMbvQPb7siJAn7I04SEFtkDaqyJyxfBaQNQstDznOZBA0dtS3q45D0DPL64FlVLrQNGUpecOi+5AxBeoDFZfhkC2nX94FV1iQNmNQT+96h1A2E4nSKoM10DXOb8b/T4WQNP/mp8HmexAyH8CEFraEEDP64uxZaekQLryyB7xVwJAo3+b0GxAcUC43Sl0cfQFQJg2zt51k15A08osTsCKOEC5wku9oCijQNAAyfuDZLtA0ReZzWHjvkDUZBtdoveyQJqDORZ0eLxAwyIEI/8lc0DOaQuCRlBoQNoaCh+ScKlA17PBGTeEAUDVo9h+pecmQMrVUsNyc8BA2wh1hj0ieEClxJNA++o7QNFvVxPApZlAtpTe4UJnJ0DRSCfQOwTNQK8v9yIFRgVAxRjAqHkEK0DZ9e+ICnZpQM59j5bxQRlAzR9OeArNHUDTCQwPy1CLQLtEeTF8rCdAtmrlkQWh/UC9S9xqJcYRQM4YvKjortpAyErUgyJVD0DW7ktjSS20QL8waRYLSihAy0O9Z2KlrkDWSOq2cko/QNdPZqY+8lBAsZmMKozbtEDUy39xliNqQNYI+TNnKMtAnka0FNpf60DYOQwmwXx6QNkqY+SwSE4=",
  "product_id": 1,
  "product_url": "site.com/1",
  "title": "Black Shoes",
  "gross_price": 99
}
'
```

View documents in the products index:
```
http://localhost:9200/products/_search
```

Specify a limit greater than the default of 10 
```
http://localhost:9200/products/_search?size=100
```

Get a count of documents in the products index:
```
http://localhost:9200/products/_count
```

####  5. Query the index using the cosine plugin.

This example nests a [bool query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-bool-query.html) within the a [function_score query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-function-score-query.html#query-dsl-function-score-query).

The ES docs state: Note that the score of the query is multiplied with the result of the script scoring. If you wish to inhibit this, set "boost_mode": "replace". We are using the plugin to define 'relevancy' and thus ordering so in this case the bool query is used as a way to filter the result set prior to running the script_score.

```
curl -H 'Content-Type: application/json' -X POST 'http://localhost:9200/products/_search' -d '
{
   "query":{
      "function_score":{
         "boost_mode":"replace",
         "query":{
            "bool":{
               "must":{
                  "term":{
                     "title":"black"
                  }
               },
               "must_not":{
                  "range":{
                     "price":{
                        "gte":20,
                        "lte":99
                     }
                  }
               }
            }
         },
         "functions":[
            {
               "script_score":{
                  "script":{
                     "source":"staysense",
                     "lang":"fast_cosine",
                     "params":{
                        "field":"embeddedVector",
                        "cosine":true,
                        "encoded_vector":"QMOFm6eRAThA0DEPnUCt6kDMVN4NykzgQMoGWYCl8mpA0aCnSEGuXUC92PFI6LF/QMqbEXVbAmpAylLA6A8L6UDS2R7g68g0QMr4NzIMjDpA2w4E7Hg6EEDXJ2q6oqi8QNM1JiQnR+JA1icGHrpshUDYvR9kjJjpQNIVLclFF6tA1Sy73i5oFUDUspOe2ZXTQNBGJxOwAg5Axqa4W+NFWEDIYphPOWPXQNoS4+OHx0FAsykWy30zc0DcA29XKB0pQMm5C2caw2BAr6/5e5f7rUC7ZrvaTr5sQNsf5Sgnuo1A3PHAEdnI1UDTqXqJFPW3QNl8hXFt+3FA2cZLlaO97kDbr/oEdlOHQNIm3D5sam1ApKiTCVzi8kBeaXBVOibSQNT9g8U2QLlAa+Ou+fV3XUCtkna1G4ErQNLuD64RL4pAwysRWsVAREDWOIPl7KJNQLUg7eMQv7lA2g/T1pczR0DdD7O4ACDOQLZns6NFBd9A3JIQhIAFd0DS8+TZ5qLXQNsoab6jQlNAs44VTfr3UUDNU+kPRkOlQMW1PK4R5/JA2kwc0Fiv0kCliEiUVG3bQNSBJhH3+y1AxXX5/EVp40DZcTnQaMQPQNYPLX+IjPFAyaVUT7hl4kDZHogeJbsQQNtP8vBk9nlA06fWtxqPbkDZUOpV5a82QNeKDmBxVyZA23aHx42mg0DC4Z4zcqpUQNPbxFkgEd5A0Kqj2PpkIkB/sXGXhIi6QNPmr6MbeLpAyS2dBgx6RUC2e9rNCizrQMXWYl1KvR9AzEFrYTyWIEDb+pPITTJmQMcnGGsN7GJAlhjfA6j9QEDasHzhum2sQJrNy5hKaD1A2LPvR7EbKEDYZpUsvubvQN05s5s2QshA3DnuNrgG/kDJfggKE3EcQNwA1sJZfDBA0Ra7NvA2N0DKDsNk+FlmQNX2hc1VSCFAy9p6O62J2kDZHQLP67kQQNkzPbSp2QtA3TUrUFI6x0DJ/F9QIjDTQN0O6Or/09RAs0mRpOe8fEDJ4XHgyWNZQM26DMZcz/JA1Nnv6aDOe0DchuHhEUOYQLlm0TFJWKw="
                     }
                  }
               }
            }
         ]
      }
   }
}
'
```

===================

#### Other useful commands:

- Delete all documents (retaining mapping/settings)
```
curl -H 'Content-Type: application/json' -X POST 'http://localhost:9200/products/_delete_by_query?conflicts=proceed' -d '{
    "query" : { 
        "match_all" : {}
    }
}'
```

- Close the index to apply new settings
```
curl -H 'Content-Type: application/json' -X POST 'localhost:9200/products/_close'
```


- Open the index after closing
```
curl -H 'Content-Type: application/json' -X POST 'localhost:9200/products/_open'
```


- Delete the index
```
curl -H 'Content-Type: application/json' -X DELETE  'http://localhost:9200/products'
```
