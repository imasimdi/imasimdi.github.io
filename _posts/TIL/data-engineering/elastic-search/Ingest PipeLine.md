# Ingest PipeLine
https://danawalab.github.io/elastic/2020/09/04/ElasticSearch-IngestPipeLine.html

## Ingest PipeLine 란?
- 색인할 때 필요에 따라 데이터를 가공해야할 때 쓰인다.
- 예를 들어 HTML가 포함된 필드를 색인 할 때, 구분 값으로 나눠진 데이터를 집계할 때 데이터를 별도의 프로세스를 통해 가공했었음
- 매번 인덱스에 해당사는 프로세스 만들고 배포가 번거로움
- `pipline`의 `processor`을 통해 불편함 해결
    - 여기서 processor란?
        - 데이터를 가공하는 개별 작업 단위
        - 이 processor들이 순차적으로 연결되어 하나의 pipline을 구성한다.

        ```json
        PUT _ingest/pipeline/my_pipeline
        {
        "description": "A pipeline to process log data",
        "processors": [
            {
            "grok": {
                "field": "message",
                "patterns": ["%{COMBINEDAPACHELOG}"]
            }
            },
            {
            "date": {
                "field": "timestamp",
                "formats": ["dd/MMM/yyyy:HH:mm:ss Z"]
            }
            },
            {
            "set": {
                "field": "processed",
                "value": true
            }
            }
        ]
        }
        ```

        - 이 예시에서 grok 프로세서는 message 필드의 특정 패턴을 추출
        - data 프로세서는 timestamp 필드의 날짜 형식을
        - set 프로세서는 processed 필드에 true 값으로 설정

## 필터와 Ingest Pipeline의 차이
- Ingest Pipeline은 데이터가 Elasticsearch에 저장되기 전에 데이터를 가공하는 데 사용
- Filter는 저장된 데이터에서 특정 조건을 만족하는 문서를 검색하는 데 사용

## PipeLine 예시
- Pipeline을 설정하기 위해서는 Ingest Node가 활성화 필요

```json
PUT _ingest/pipline/{pipline name}
{
  "description" : "...",
  "processors" : [ ... ]
}
```

- description은 해당 pipeline의 설명 필드
- processor에는 pipeline에서 수행할 processors에 대한 정의를 작성하는 필드
- 프로세서는 **순서대로** 수행되고 여러 개의 프로세서를 등록 가능

## Split processor
- 정의된 필드의 데이터를 구분자로 분해
- 배열 형태로 만들어준다.

```json
PUT _ingest/pipeline/split_field
{
  "description": "Decompose the field by separator",
  "processors": [
    {
      "split": {
        "field": "productList",
        "separator": ","
      }
    }
  ]
}
```

- `_simulate` API를 통해서 생성할 processor를 확인할 수 있다.
- docs의 _source 필드에 값을 입력하면 결과를 미리 볼 수 있다.

```json
POST _ingest/pipeline/_simulate
{
  "pipeline": {
    "description": "Decompose the field by separator",
    "processors": [
      {
        "split": {
          "field": "productList",
          "separator": ","
        }
      }
    ]
  },
  "docs": [
    {
      "_source": {
        "productList": "1,2,3,4,5"
      }
    }
  ]
}
```

- 결과

```json
{
  "docs" : [
    {
      "doc" : {
        "_index" : "_index",
        "_type" : "_doc",
        "_id" : "_id",
        "_source" : {
          "productList" : [
            "1",
            "2",
            "3",
            "4",
            "5"
          ]
        },
        "_ingest" : {
          "timestamp" : "2020-09-04T02:23:40.241192Z"
        }
      }
    }
  ]
}
```

## HTML strip processor
- 필드의 HTML Tag를 제거해준다.

```json
PUT _ingest/pipeline/html_strip
{
  "description": "Remove Html Tag", 
  "processors": [
    {
      "html_strip": {
        "field": "contents"
      }
    }
  ]
}
```

- 시뮬레이터 결과

```json
POST _ingest/pipeline/_simulate
{
  "pipeline": {
    "description": "Remove Html Tag",
    "processors": [
      {
        "html_strip": {
          "field": "contents"
        }
      }
    ]
  },
  "docs": [
    {
      "_source": {
        "contents": "<em>다나와 ElasticSearch</em>"
      }
    }
  ]
}
```

```json
{
  "docs": [
    {
      "doc": {
        "_index": "_index",
        "_type": "_doc",
        "_id": "_id",
        "_source": {
          "contents": "다나와 ElasticSearch"
        },
        "_ingest": {
          "timestamp": "2024-06-23T00:00:00.000000Z"
        }
      }
    }
  ]
}
```

## Index 적용
- 정의한 Pipeline을 적용하여 색인할 때는 index API뒤에 `pipeline={pipeline_name}`를 붙여서 호출한다.

```json
## 인덱스 생성
PUT sample_index
{
  "mappings": {
    "properties": {
      "boardContents": {
        "type": "text"
      },
      "productSeqList": {
        "type": "keyword"
      }
    }
  }
}


## PipeLine 정의
PUT _ingest/pipeline/sample_pipeLine
{
  "description": "sample", 
  "processors": [
    {
      "html_strip": {
        "field": "boardContents"
      }
    },
    {
      "trim": {
        "field": "productSeqList"
      }
    },
    {
      "split": {
        "field": "productSeqList",
        "separator": ","
      }
    }
  ]
}

##색인
POST sample_index/_doc?pipeline=sample_pipeLine
{
  "boardContents" : "<em>sample 제목</em>",
  "productSeqList" : "001,002,003"
}
```

- 결과

```json
 {
        "_index" : "sample_index",
        "_type" : "_doc",
        "_id" : "RjRaV3QBZ9BimQhnrQr8",
        "_score" : 1.0,
        "_source" : {
          "boardContents" : "sample 제목",
          "productSeqList" : [
            "001",
            "002",
            "003"
          ]
        }
      }
```

## 결론
- 인덱스 마다 별도의 전처리 과정을 만들지 않아서 간편하다.
- 다양한 processor가 존재하고, 이를 통해 색인 시 다양한 데이터 처리가 가능하다.

## 더 찾아본 processor
https://www.elastic.co/guide/en/elasticsearch/reference/current/processors.html

- convert

```json
{
  "convert": {
    "field": "age",
    "type": "integer"
  }
}
```

- grok
    - 정규 표현식을 사용하여 필드의 값을 구조화된 데이터로 추출
    ```json
    {
    "grok": {
        "field": "message",
        "patterns": ["%{IP:client_ip}"]
    }
    }

    ```

- rename
    - 필드의 이름을 변경
    ```json
    {
    "rename": {
        "field": "old_field",
        "target_field": "new_field"
    }
    }
    ```

- set

```json
{
  "set": {
    "field": "status",
    "value": "active"
  }
}

```

- 그 외 파이썬 문법과 비슷한 동작을 하는 join, split등 이이 있고, JSON, CSV와 같은 데이터를 파싱도 가능하다.

- json
    ```json
    POST _ingest/pipeline/_simulate
    {
    "pipeline": {
        "description": "Parse JSON from message field",
        "processors": [
        {
            "json": {
            "field": "message",
            "target_field": "parsed_message"
            }
        }
        ]
    },
    "docs": [
        {
        "_source": {
            "message": "{\"user\":\"kim\", \"age\":30, \"city\":\"Seoul\"}"
        }
        }
    ]
    }

    ```

    - target_field에서 파싱된 데이터를 저장할 필드명을 정한다.
    
    ```json
        {
    "docs": [
        {
        "doc": {
            "_index": "_index",
            "_type": "_doc",
            "_id": "_id",
            "_source": {
            "message": "{\"user\":\"kim\", \"age\":30, \"city\":\"Seoul\"}",
            "parsed_message": {
                "user": "kim",
                "age": 30,
                "city": "Seoul"
            }
            },
            "_ingest": {
            "timestamp": "2024-06-23T00:00:00.000000Z"
            }
        }
        }
    ]
    }
    ```