# CH3 데이터 모델링

엘라스틱서치에서는 색인할 때 문서의 데이터 유형에 따라 필드에 적절한 데이터 타입을 지정해야 하는데 이러한 과정을 매핑이라고 한다. **매핑은 인덱스에 추가되는 각 데이터 타입을 구체적으로 정의하는 일이다 (데이터 모델링)**

## 3.1 매핑 API 이해하기

문서에 존재하는 필드의 속성을 정의할 때, 각 필드 속성에는 데이터 타입과 메타데이터가 포함된다.
이를 통해 색인 과정에서 문서가 어떻게 역색인으로 변환되는지를 상세하게 정의할 수 있다.


```
* 스키마리스

명시적으로 필드를 정의하지 않아도, 데이터 유형에 따라 필드 데이터 타입에 대한 매핑 정보가 자동으로 생성되는 것.

실수로 잘못된 타입이 지정되는 경우 수정할 방법이 없다.

```
#### 책 P.62 예제 테스트
---
##### PUT movie/_doc/1
```json
{
  "movieCd" : "20173732",
  "movieNm" : "캡틴 아메리카"
}
```
첫 번째 문서를 매핑 설정 없이 색인하면 movieCd 필드는 숫자 타입으로 매핑되고, movieNm 필드는 문자 타입으로 매핑된다고 한다.

---

##### GET movie/_mapping
```json
{ 
   "movie":{ 
      "mappings":{ 
         "properties":{ 
            "movieCd":{ 
               "type":"text",
               "fields":{ 
                  "keyword":{ 
                     "type":"keyword",
                     "ignore_above":256
                  }
               }
            },
            "movieNm":{ 
               "type":"text",
               "fields":{ 
                  "keyword":{ 
                     "type":"keyword",
                     "ignore_above":256
                  }
               }
            }
         }
      }
   }
}     
```
실제로 실행시켜본 결과 movieCd 필드가 문자 타입으로 매핑되었다.

---
##### PUT movie/_doc/2
```json
{
  "movieCd" : "XT001",
  "movieNm" : "아이언맨"
}
```
movieCd 필드가 문자 타입으로 매핑되어서 두 번째 문서도 색인되었다.

---
##### DELETE movie
##### PUT movie/_doc/1
```json
{
  "movieCd" : 20173732,
  "movieNm" : "캡틴 아메리카"
}
```
첫 번째 문서에서 20173732를 감싸고 있던 "" 을 제거하고 색인을 하면, movieCd 필드가 long 타입으로 매핑된다.
> 필드가 숫자 타입인 경우, 항상 long 또는 double로 매핑된다. (가장 큰 유형을 할당)

---
##### PUT movie/_doc/2

다시 두번째 문서를 색인하면 
```json
{
    ...

    "type": "mapper_parsing_exception",
    "reason": "failed to parse field [movieCd] of type [long] in document with id '2'. Preview of field's value: 'XT001'",
    "caused_by":{
        "type": "illegal_argument_exception",
        "reason": "For input string: \"XT001\""
    }
}
```
parsing_exception 이 발생한다.

만약 문서의 movieCd 필드 값이 숫자 형식(20173732)으로 되어있고 ""로 감싸져 있다면 parsing이 되기 때문에 색인이 가능하다.

> JSON 포맷에서 timestamp 값은 long으로 표현하기도 하는데, Elasticsearch 에서는 long로 표현된 timestamp 값을 date 필드로 감지하지 못한다.

---

#### 매핑 정보 설정시 고민사항
* 문자열을 분석할 것인가?
* _source에 어떤 필드를 정의할 것인가?
* 날짜 필드를 가지는 필드는 무엇인가?
* 매핑에 정의되지 않고 유입되는 필드는 어떻게 처리할 것인가?


### 3.1.1 매핑 인덱스 만들기
PUT movie_search
```json
{
    "setting" : {
        "number_of_shards" : 5,
        "number_of_replicas" : 1
    },
    "mappings": {
        "_doc": {
            "properties": {
                "movieCd": {
                    "type": "keyword"
                },

                ...

            }
        }
    }
}
```
### 3.1.2 매핑 확인
GET movie_search/_mapping

### 3.1.3 매핑 파라미터
매핑 파라미터는 색인할 필드의 데이터를 어떻게 저장할지에 대한 다양한 옵션을 제공한다. 이러한 옵션은 필드에 매핑 정보를 설정할 때 유용하게 사용할 수 있다.

#### analyzer
색인과 검색 시 지정한 분석기로 해당 필드의 데이터를 형태소 분석하겠다는 의미의 파라미터.

text 데이터 타입의 필드는 analyzer 매핑 파라미터를 기본적으로 사용해야 한다. (별도의 분석기를 지정하지 않으면 Standard Analyzer로 형태소 분석을 수행)

```json
PUT /my_index
{
  "mappings": {
    "properties": {
      "text": { 
        "type": "text",
        "fields": {
          "english": { 
            "type":     "text",
            "analyzer": "english"
          }
        }
      }
    }
  }
}
```

#### normalizer

keyword 데이터 타입의 경우 원문을 기준으로 문서가 색인된다. 그래서 Cafe, cAfe, caFE 는 서로 다른 데이터로 인식된다.
하지만 해당 유형을 normalizer를 통해 분석기에 asciifolding과 같은 필터를 사용하면 같은 데이터로 인식되게 할 수 있다.

normalizer는 keyword를 색인하기 전과 match query 또는 term query를 통해 keyword 필드를 검색할 때 적용된다.


```json
PUT index
{
  "settings": {
    "analysis": {
      "normalizer": {
        "my_normalizer": {
          "type": "custom",
          "char_filter": [],
          "filter": ["lowercase", "asciifolding"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "foo": {
        "type": "keyword",
        "normalizer": "my_normalizer"
      }
    }
  }
}

PUT index/_doc/1
{
  "foo": "BÀR"
}

PUT index/_doc/2
{
  "foo": "bar"
}

PUT index/_doc/3
{
  "foo": "baz"
}

GET index/_search
{
  "query": {
    "term": {
      "foo": "BAR"
    }
  }
}

{ 
   "took":1,
   "timed_out":false,
   "_shards":{ 
      "total":1,
      "successful":1,
      "skipped":0,
      "failed":0
   },
   "hits":{ 
      "total":{ 
         "value":2,
         "relation":"eq"
      },
      "max_score":0.47000363,
      "hits":[ 
         { 
            "_index":"index",
            "_type":"_doc",
            "_id":"1",
            "_score":0.47000363,
            "_source":{ 
               "foo":"BÀR"
            }
         },
         { 
            "_index":"index",
            "_type":"_doc",
            "_id":"2",
            "_score":0.47000363,
            "_source":{ 
               "foo":"bar"
            }
         }
      ]
   }
}

GET index/_search
{
  "query": {
    "match": {
      "foo": "BAR"
    }
  }
}

{ 
   "took":1,
   "timed_out":false,
   "_shards":{ 
      "total":1,
      "successful":1,
      "skipped":0,
      "failed":0
   },
   "hits":{ 
      "total":{ 
         "value":2,
         "relation":"eq"
      },
      "max_score":0.47000363,
      "hits":[ 
         { 
            "_index":"index",
            "_type":"_doc",
            "_id":"1",
            "_score":0.47000363,
            "_source":{ 
               "foo":"BÀR"
            }
         },
         { 
            "_index":"index",
            "_type":"_doc",
            "_id":"2",
            "_score":0.47000363,
            "_source":{ 
               "foo":"bar"
            }
         }
      ]
   }
}
```

#### boost
필드에 가중치를 부여한다. 가중치에 따라 유사도 점수(_score)가 달라지므로 검색 결과 노출 순서에 영향을 준다.

최신 엘라스틱서치는 색인 시 boost 설정을 할 수 없도록 바뀌었다.

#### coerce
색인 시 자동 형변환을 허용할지 여부를 설정하는 파라미터다.

숫자 형태의 문자열 -> Integer 타입

#### copy_to
매핑 파라미터를 추가한 필드의 값을 지정한 필드로 복사한다.

예를들어 keyword 타입의 필드의 값을 다른 필드로 복사할 때, 복사된 필드에서는 text 타입을 지정해 형태소 분석을 할 수 있다. 

또는 여러 개의 필드 데이터를 하나의 필드에 모아서 전체 검색 용도로 사용하기도 한다.

```json
PUT index
{
  "mappings": {
    "properties": {
      "first_name": {
        "type": "text",
        "copy_to": "full_name" 
      },
      "last_name": {
        "type": "text",
        "copy_to": "full_name" 
      },
      "full_name": {
        "type": "text"
      }
    }
  }
}

PUT index/_doc/1
{
  "first_name": "John",
  "last_name": "Smith"
}

GET index/_search
{
  "query": {
    "match": {
      "full_name": { 
        "query": "John Smith",
        "operator": "and"
      }
    }
  }
}

{ 
   "took":16,
   "timed_out":false,
   "_shards":{ 
      "total":1,
      "successful":1,
      "skipped":0,
      "failed":0
   },
   "hits":{ 
      "total":{ 
         "value":1,
         "relation":"eq"
      },
      "max_score":0.5753642,
      "hits":[ 
         { 
            "_index":"index",
            "_type":"_doc",
            "_id":"1",
            "_score":0.5753642,
            "_source":{ 
               "first_name":"John",
               "last_name":"Smith"
            }
         }
      ]
   }
}
```

#### fielddata
엘라스틱 서치가 힙 공간에 생성하는 메모리 캐시.

반복적인 메모리 부족 현상과 잦은 GC로 현재는 거의 사용되지 않는다.

text 타입의 필드는 기본적으로 분석기에 의해 형태소 분석이 되기 때문에 집계나 정렬 등의 기능을 수행할 수 없음 -> fielddata를 사용하여 집계나 정렬 수행 가능 

#### doc_values
엘라스틱서치에서 사용하는 기본 캐시로 text 타입을 제외한 모든 타입에서 기본적으로 doc_values 캐시를 사용

운영체제의 파일 시스템 캐시를 통해 디스크에 있는 데이터에 빠르게 접근 가능

필드를 정렬, 집계할 필요가 없고 스크립트에서 필드 값에 액세스할 필요가 없다면 디스크 공간을 절약하기 위해 doc_values를 비활성화할 수도 있다.

#### dynamic
매핑에 필드를 추가할 때 동적으로 생성할지, 생성하지 않을지를 결정한다.

* true : 새로 추가되는 필드를 매핑에 추가한다.
* false : 새로 추가되는 필드를 무시한다. 해당 필드는 색인되지 않아 검색할 수 없지만 _source에는 표시된다.
* strict : 새로운 필드가 감지되면 예외가 발생하고 문서 자체가 색인되지 않는다.

```json
PUT my_index
{
  "mappings": {
    "dynamic": false, // type level에서 Dynamic mapping이 비활성화.
    "properties": {
      "user": { // user object는 type-level 설정을 상속
        "properties": {
          "name": {
            "type": "text"
          },
          "social_networks": { 
            "dynamic": true, //user.social_networks object는 dynamic mapping 이 가능하다.
            "properties": {}
          }
        }
      }
    }
  }
}
```

#### enabled
검색 결과에는 포함하지만 색인은 하고 싶지 않은 경우 사용

매핑 파라미터 중 enabled를 false로 설정하면 _source에는 검색이 되지만 색인은 하지 않는다.

#### format
엘라스틱 서치는 format을 이용하여 JSON 문서에 문자열 형식으로 표현된 날짜를 인식하고 파싱하여 UTC의 밀리초 단위로 변환해 저장한다.

```json
PUT my_index
{
  "mappings": {
    "properties": {
      "date": {
        "type":   "date",
        "format": "yyyy-MM-dd" // "format": "date"
      }
    }
  }
}
```

#### ignore_above
필드에 저장되는 문자열이 지정한 크기를 넘어서면 빈 값으로 색인한다.

_source에는 표시된다.
#### ignore_malformed
엘라스틱서치는 잘못된 데이터 타입을 색인하려고 하면 예외가 발생하고 해당 문서 전체가 색인되지 않는다.

ignore_malformed 매핑 파라미터를 사용하면 해당 필드만 무시하고 문서는 색인할 수 있다.

#### index
필드값을 색인할지를 결정한다. 기본값은 true 이며, false 로 변경하면 해달 필드를 색인하지 않는다.

_source에는 표시된다.

#### fields
다중 필드를 설정할 수 있는 옵션이다. 필드 안에 또 다른 필드의 정보를 추가할 수 있어 같은 string 값을 각각 다른 분석기로 처리하도록 설정할 수 있다.

```json
PUT my_index
{
  "mappings": {
    "properties": {
      "city": {
        "type": "text",
        "fields": {
          "raw": { 
            "type":  "keyword"
          }
        }
      }
    }
  }
}

GET my_index/_search
{
  "query": {
    "match": {
      "city": "york" 
    }
  },
  "sort": {
    "city.raw": "asc" 
  },
  "aggs": {
    "Cities": {
      "terms": {
        "field": "city.raw" 
      }
    }
  }
}
```

#### norms
문서의 _score 값 계산에 필요한 정규화 인수를 사용할지 여부를 설정한다. 기본값은 true다.

_source 계산이 필요없거나 단순 필터링 용도로 사용하는 필드는 비활성화해서 디스크 공간을 절약할 수 있다.

#### null_value
엘라스틱서치는 색인 시 문서에 필드가 없거나 필드의 값이 null이면 필드를 생성하지 않는다.

null_value를 설정하면 문서의 값이 null이더라도 필드를 생성하고 null_value에 설정된 값으로 저장한다.

#### position_increment_gap
배열 형태의 데이터를 색인할 때 검색의 정확도를 높이기 위해 제공하는 옵션이다. 필드 데이터 중 단어와 단어 사이의 간격(slop)을 허용할지를 설정한다.

#### properties
오브젝트 타입이나 중첩 타입의 스키마를 정의할 때 사용되는 옵션으로 필드의 타입을 매핑한다.

#### search_analyzer
색인과 검색 시 다른 분석기를 사용하고 싶은 경우 search_analyzer 설정해서 검색 시 사용할 분석기를 별도로 지정할 수 있다.

#### similarity
유사도 측정 알고리즘을 지정한다.

#### store
필드의 값을 저장해 검색 결과에 값을 포함하기 위한 매핑 파라미터.

기본적으로 엘라스틱서치에서는 _source에 색인된 문서가 저장된다.(실제 문서의 정보를 담고 있는 항목) 

하지만 store 매핑 파라미터를 사용하면 해당 필드를 자체적으로 저장할 수 있다.

```json
PUT my_index
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "store": true 
      },
      "date": {
        "type": "date",
        "store": true 
      },
      "content": {
        "type": "text"
      }
    }
  }
}

GET my_index/_search
{
  "stored_fields": [ "title", "date" ] 
}
```

#### term_vector
루씬에서 분석된 용어의 정보를 포함할지 여부를 결정하는 매핑 파라미터이다.

### 3.2 메타 필드

메타 필드는 엘라스틱서치에서 생성된 문서가 제공하는 특별한 필드이다.

#### _index 메타 필드
해당 문서가 속한 인덱스의 이름을 담고 있다.

이를 이용해 검색된 문서의 인덱스명을 알 수 있으며, 해당 인덱스에 몇 개의 문서가 있는지 확인할 수 있다.

#### _type 메타 필드
해당 문서가 속한 매핑의 타입 정보를 담고 있다.

#### _id 메타 필드
_id 메타 필드는 문서를 식별하는 유일한 키 값이다. 한 인덱스에서 색인된 문서들은 서로 다른 키 값을 가진다.

#### _uid 메타 필드
_uid 메타 필드는 특수한 목적의 식별키로 "#" 태그를 이용해 _type과 _id 값을 조합해 내부적으로만 사용한다.

#### _source 메타 필드
_source 메타 필드는 문서의 원본 데이터를 제공한다. 내부에는 색인 시 전달된 원본 JSON 문서의 본문이 저장돼 있다. 

일반적으로 JSON 문서를 검색 결과로 표시할 때 사용한다.

#### _all 메타 필드
_all 메타 필드는 색인에 사용된 모든 필드의 정보를 가진 메타 필드이다.

데이터의 크기를 너무 많이 차지하는 문제가 있어 엘라스틱서치 6.0 이상부터는 폐기되었다.

필드의 복사가 필요한 경우 copy_to 파라미터를 사용해야 한다.

#### _routing 메타 필드

_routing 메타 필드는 특정 문서를 특정 샤드에 저장하기 위해 사용자가 지정하는 메타 필드다.

기본적으로 색인을 하면 문서 id를 이용해 문서가 색인될 샤드를 결정한다.
`Hash (document_id) % num_of_shards`

_routing 메타 필드를 사용하면 샤드를 결정하는데 문서 ID 대신 _routing 값이 사용된다.

`Hash (_routing) % num_of_shards`

### 3.3 필드 데이터 타입
* keyword, text 같은 문자열 데이터 타입
* date, long, double, integer, boolean, ip 같은 일반적인 데이터 타입
* 객체 또는 중첩문과 같은 JSON 계층 특성의 데이터 타입
* geo_point, geo_shape 같은 특수한 데이터 타입


#### Keyword 데이터 타입

Keyword 타입을 사용할 경우 별도의 분석기를 거치지 않고 원문 그대로 색인하기 때문에 특정 코드나 키워드 등 정형화된 콘텐츠에 주로 사용된다.

엘라스틱서치의 일부 기능은 형태소 분석을 하지 않아야만 사용이 가능한데 이 경우에도 Keyword 데이터 타입이 사용된다.

Keyword 데이터 타입은
* 검색 시 필터링 되는 항목
* 정렬이 필요한 항목
* 집계해야 하는 항목

에 많이 사용된다.

#### Text 데이터 타입

Text 데이터 타입을 이용하면 색인 시 지정된 분석기가 칼럼의 데이터를 문자열 데이터로 인식하고 이를 분석한다. (default : Standard Analyzer)

Text 데이터 타입은 전문 검색이 가능한다는 것이 가장 큰 특징이다. (전체 텍스트가 토큰화되어 특정 단어르 검색하는 것이 가능해진다.)

Text 데이터 타입을 사용한 필드에 정렬이나 집계 연산을 사용해야 하는 경우 Text 타입과 keyword 타입을 동시에 갖도록 멀티 필드로 설정할 수 있다.

#### Array 데이터 타입

Array 타입은 문자열이나 숫자처럼 일반적인 값뿐 아니라 객체 형태의 값도 지정할 수 있다.

엘라스틱서치에서는 매핑 설정 시  Array 타입을 명시적으로 정의하지 않는다. 정의된 인덱스 필드에 단순히 배열 값을 입력하면 자동으로 Array 형태로 저장된다.

필드가 동적으로 추가될 때, 배열의 첫 번째 값이 필드의 데이터 타입을 결정하며, 이후의 데이터는 모두 같은 타입이어야 색인 시 오류가 발생하지 않는다.

#### Numeric 데이터 타입

숫자 데이터 타입은 여러 가지 종류가 제공되는데, 데이터의 크기에 알맞은 타입을 사용함으로 색인과 검색을 효율적으로 처리하기 위해서다.

#### Date 데이터 타입

날짜는 다양하게 표현될 수 있기 때문에, 올바른 구문 분석을 위해서는 날짜 문자열 형식을 명시적으로 설정해야 한다. 별도의 형식을 지정하지 않는 경우 기본 형식인 "yyyy-MM-ddTHH:mm:ssZ" 로 지정된다.

내부적으로 UTC의 밀리초 단위로 변환해 저장한다.

#### Range 데이터 타입
Range 데이터 타입은 범위가 있는 데이터를 저장할 떄 사용하는 데이터 타입으로 데이터의 시작과 끝만 정의해주면 된다.

숫자뿐 아니라 IP에 대한 범위도 Range 데이터 타입으로 정의할 수 있다.

#### Boolean 데이터 타입
true, false

참과 거짓값을 문자열로 표현하는 것도 가능하다.

#### Geo-Point 데이터 타입

위도, 경도 등 위치 정보를 담은 데이터를 저장할 때 사용한다.

위치 기반 쿼리를 이용해 반경 내 쿼리, 위치 기반 집계, 위치별 정렬 등을 사용할 수 있기 때문에 위치 기반 데이터를 색인하고 검색하는데 매우 유용하다.

#### IP 데이터 타입

IP 주소를 저장하는데 사용한다.  IPv4나 IPv6를 모두 지정할 수 있다.

#### Object 데이터 타입

필드의 값으로 문서를 가지는 데이터 타입을 Object 데이터 타입이라고 한다. Object 데이터 타입을 정의할 때는 단지 필드 값으로 다른 문서의 구조를 입력하면 된다.

#### Nested 데이터 타입

Object 객체 배열을 독립적으로 색인하고 질의하는 형태의 데이터 타입.

데이터가 배열 형태로 저장되면 한 필드 내의 검색은 기본적으로 OR 조건으로 검색된다. -> 데이터의 구조가 복잡해지면 모호한 상황이 발생함.

Nested 데이터 타입을 이용하면 검색할 때 일치하는(AND 조건) 문서만 정확하게 출력할 수 있다.