# 3.4 엘라스틱서치 분석기

## 3.4.1 텍스트 분석 개요
엘라스틱 서치는 루씬을 기반으로 구축된 텍스트 기반 검색엔진이다.

루씬은 내부적으로 다양한 분석기를 제공하는데, 엘라스틱서치는 루씬이 제공하는 분석기를 그대로 활용한다.

엘라스틱서치는 문서를 색인하기 전에 해당 문서의 필드 타입이 무엇인지 확인하고, 텍스트 타입이면 분석기를 이용해 분석한다.

텍스트 분석은 언어별로 조금씩 다르게 동작하기 때문에, 엘라스틱서치는 각각 다른 언어의 형태소를 분석할 수 있도록 언어별 분석기를 제공한다.

만약 원하는 분석기가 없다면 직접 개발하거나 커뮤니티에서 개발한 Custom Analyzer를 설치해서 사용할 수도 있다.

## 3.4.2 역색인 구조

루씬의 색인은 역색인이라는 특수한 방식으로 구조화돼 있다. (색인한다는 것은 역색인 파일을 만든다는 것이다.)

#### 역색인 구조
* 모든 문서가 가지는 단어의 고유 단어 목록
* 해당 단어가 어떤 문서에 속해 있는지에 대한 정보
* 전체 문서에 각 단어가 몇 개 들어있는지에 대한정보
* 하나의 문서에 단어가 몇 번씩 출현했는지에 대한 빈도

## 3.4.3 분석기의 구조

#### 분석기 동작 프로세스
1. 문장을 특정한 규칙에 의해 수정한다
> CHARACTER FILTER : 
> 문장을 분석하기 전에 입력 텍스트에 대해 특정한 단어를 변경하거나 HTML과 같은 태그를 제거하는 역할을 하는 필터로 텍스트를 개별 토큰화하기 전의 전처리 과정이다. ReplaceAll() 함수처럼 패턴으로 텍스트를 변경하거 사용자가 정의한 필터를 적용할 수 있다.
2. 수정한 문장을 개별 토큰으로 분리한다.
> TOKENIZER FILTER :
> Tokenizer Filter는 분석기를 구성할 때 하나만 사용할 수 있으며 텍스트를 어떻게 나눌 것인지를 정의한다. 한글을 분해할 때는 한글 형태소 분석기의 Tokenizer를 사용하고, 영문을 분석할 때는 영문 형태소 분석기의 Tokenizer를 사용하는 등 상황에 맞게 적절한 Tokenizer를 사용하면 된다.
3. 개별 토큰을 특정한 규칙에 의해 변경한다.
> TOKEN FILTER : 토큰화된 단어를 하나씩 필터링해서 사용자가 원하는 토큰으로 변환한다. 예를 들어, 불필요한 단어를 제거하거나 동의어 사전을 만들어 단어를 추가하거나 영문 단어를 소문자로 변환하는 등의 작업을 수행할 수 있다. Token Filter는 여러 단계가 순차적으로 이뤄지며 순서를 어떻게 지정하느냐에 따라 검색의 질이 달라질 수 있다.

### 3.4.3.1 분석기 사용법
엘라스틱서치에서는 형태소가 어떻게 분석되는지를 확인할 수 있는 _analyze API를 제공한다.

#### 분석기를 이용한 분석
> 다음과 같이 설정하면 지정한 (미리 정의된)분석기를 적용했을 때 어떻게 토큰이 분리되는지 확인할 수 있다.
```json
POST _analyze
{ 
   "analyzer":"standard",
   "text":"_analyze API를 이용"
}

>>>

{ 
   "tokens":[ 
      { 
         "token":"_analyze",
         "start_offset":0,
         "end_offset":8,
         "type":"<ALPHANUM>",
         "position":0
      },
      { 
         "token":"api를",
         "start_offset":9,
         "end_offset":13,
         "type":"<ALPHANUM>",
         "position":1
      },
      { 
         "token":"이용",
         "start_offset":14,
         "end_offset":16,
         "type":"<HANGUL>",
         "position":2
      }
   ]
}

POST _analyze
{ 
   "tokenizer":"whitespace",
   "filter":[ 
      "lowercase",
      { 
         "type":"stop",
         "stopwords":[ 
            "a",
            "is"
         ]
      }
   ],
   "char_filter" : ["html_strip"],
   "text":"this is a <b>test</b>"
}

>>>

{ 
   "tokens":[ 
      { 
         "token":"this",
         "start_offset":0,
         "end_offset":4,
         "type":"word",
         "position":0
      },
      { 
         "token":"test",
         "start_offset":13,
         "end_offset":21,
         "type":"word",
         "position":3
      }
   ]
}
```

#### 필드를 이용한 분석
> 인덱스를 설정할 때, 다양한 옵션과 필터를 적용해 분석기를 직접 설정할 수 있으며, 설정한 분석기를 매핑 설정을 통해 칼럼에 지정할 수 있다.
```json
PUT index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_custom_analyzer": {
          "type": "custom", 
          "tokenizer": "standard",
          "char_filter": [
            "html_strip"
          ],
          "filter": [
            "lowercase",
            "asciifolding"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "contents": { 
        "type": "text",
        "analyzer": "my_custom_analyzer"
      }
    }
  }
}

```

> _analyze API는 다음과 같이 필드를 직접 지정해 사용할 수 있다.
```json
POST index/_analyze
{
  "analyzer": "my_custom_analyzer",
  "text": "Is this <b>déjà vu</b>?"
}

{
  "field": "contents",
  "text": "Is this <b>déjà vu</b>?"
}

>>>

{ 
   "tokens":[ 
      { 
         "token":"is",
         "start_offset":0,
         "end_offset":2,
         "type":"<ALPHANUM>",
         "position":0
      },
      { 
         "token":"this",
         "start_offset":3,
         "end_offset":7,
         "type":"<ALPHANUM>",
         "position":1
      },
      { 
         "token":"deja",
         "start_offset":11,
         "end_offset":15,
         "type":"<ALPHANUM>",
         "position":2
      },
      { 
         "token":"vu",
         "start_offset":16,
         "end_offset":22,
         "type":"<ALPHANUM>",
         "position":3
      }
   ]
}

```

> 색인 파일에 들어갈 토큰만 변경되어 저장되고, 실제 문서의 내용은 변함없이 저장된다.
```json
PUT index/_doc/1
{ 
  "contents" : "Is this <b>déjà vu</b>?"
}

>>>

GET index/_search

{ 
   "took":4,
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
      "max_score":1.0,
      "hits":[ 
         { 
            "_index":"index",
            "_type":"_doc",
            "_id":"1",
            "_score":1.0,
            "_source":{ 
               "contents":"Is this <b>déjà vu</b>?" 
            }
         }
      ]
   }
}

```

```json
PUT index
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_custom_analyzer": { 
          "type": "custom",
          "char_filter": [
            "emoticons"
          ],
          "tokenizer": "punctuation",
          "filter": [
            "lowercase",
            "english_stop"
          ]
        }
      },
      "tokenizer": {
        "punctuation": { 
          "type": "pattern",
          "pattern": "[ .,!?]"
        }
      },
      "char_filter": {
        "emoticons": { 
          "type": "mapping",
          "mappings": [
            ":) => _happy_",
            ":( => _sad_"
          ]
        }
      },
      "filter": {
        "english_stop": { 
          "type": "stop",
          "stopwords": "_english_"
        }
      }
    }
  }
}

POST index/_analyze
{
  "analyzer": "my_custom_analyzer",
  "text": "I'm a :) person, and you?"
}

>>>

{ 
   "tokens":[ 
      { 
         "token":"i'm",
         "start_offset":0,
         "end_offset":3,
         "type":"word",
         "position":0
      },
      { 
         "token":"_happy_",
         "start_offset":6,
         "end_offset":8,
         "type":"word",
         "position":2
      },
      { 
         "token":"person",
         "start_offset":9,
         "end_offset":15,
         "type":"word",
         "position":3
      },
      { 
         "token":"you",
         "start_offset":21,
         "end_offset":24,
         "type":"word",
         "position":5
      }
   ]
}
```


#### 색인과 검색 시 분석기를 각각 설정

> 분석기는 색인할 떄 사용되는 Index Analyzer와 검색할 때 사용되는 Search Analyzer로 구분해서 구성할 수도 있다.

```json
PUT index
{
  "settings": {
    "analysis": {
      "filter": {
        "autocomplete_filter": {
          "type": "edge_ngram",
          "min_gram": 1,
          "max_gram": 20
        }
      },
      "analyzer": {
        "autocomplete": { 
          "type": "custom",
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "autocomplete_filter"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "text": {
        "type": "text",
        "analyzer": "autocomplete", 
        "search_analyzer": "standard" 
      }
    }
  }
}

PUT index/_doc/1
{
  "text": "Quick Brown Fox" 
}

POST index/_analyze
{
  "analyzer": "autocomplete",
  "text": "Quick Brown Fox"
}

>>>

{ 
   "tokens":[ 
      { 
         "token":"q",
         "start_offset":0,
         "end_offset":5,
         "type":"<ALPHANUM>",
         "position":0
      },
      { 
         "token":"qu",
         "start_offset":0,
         "end_offset":5,
         "type":"<ALPHANUM>",
         "position":0
      },
      { 
         "token":"qui",
         "start_offset":0,
         "end_offset":5,
         "type":"<ALPHANUM>",
         "position":0
      },
      { 
         "token":"quic",
         "start_offset":0,
         "end_offset":5,
         "type":"<ALPHANUM>",
         "position":0
      },
      { 
         "token":"quick",
         "start_offset":0,
         "end_offset":5,
         "type":"<ALPHANUM>",
         "position":0
      },
      { 
         "token":"b",
         "start_offset":6,
         "end_offset":11,
         "type":"<ALPHANUM>",
         "position":1
      },
      { 
         "token":"br",
         "start_offset":6,
         "end_offset":11,
         "type":"<ALPHANUM>",
         "position":1
      },
      { 
         "token":"bro",
         "start_offset":6,
         "end_offset":11,
         "type":"<ALPHANUM>",
         "position":1
      },
      { 
         "token":"brow",
         "start_offset":6,
         "end_offset":11,
         "type":"<ALPHANUM>",
         "position":1
      },
      { 
         "token":"brown",
         "start_offset":6,
         "end_offset":11,
         "type":"<ALPHANUM>",
         "position":1
      },
      { 
         "token":"f",
         "start_offset":12,
         "end_offset":15,
         "type":"<ALPHANUM>",
         "position":2
      },
      { 
         "token":"fo",
         "start_offset":12,
         "end_offset":15,
         "type":"<ALPHANUM>",
         "position":2
      },
      { 
         "token":"fox",
         "start_offset":12,
         "end_offset":15,
         "type":"<ALPHANUM>",
         "position":2
      }
   ]
}

GET index/_search
{
  "query": {
    "match": {
      "text": {
        "query": "Quick Br", 
        "operator": "and"
      }
    }
  }
}

>>>

{ 
   "took":34,
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
      "max_score":0.839562,
      "hits":[ 
         { 
            "_index":"index",
            "_type":"_doc",
            "_id":"1",
            "_score":0.839562,
            "_source":{ 
               "text":"Quick Brown Fox"
            }
         }
      ]
   }
}
```

분석기를 매핑할 때 기본적으로 "analyzer"라는 항목을 이용해 설정하게 되는데, 이는 색인 시점과 검색 시점에 모두 동일한 분석기를 사용한다는 의미다. 만약 각 시점에 서로 다른 분석기를 사용하려면 "search_analyzer" 항목을 이용해 검색 시점의 분석기를 재정의해야 한다.

대개 Index Analyzer와 Search Analyzer는 같은 분석기를 사용한다. 그러나 위 예제처럼 autocomplete 기능을 위해 edge_ngram tokenizer를 사용하는 경우 검색시 다른 분석기를 사용하는 것이 합리적이다.

PUT mapping API(PUT /<index>/_mapping)를 이용하여 기존 필드의 search_analyzer 설정 업데이트가 가능하다.

### 3.4.3.2 대표적인 분석기

#### Standard Analyzer
인덱스를 생성할 때 setting에 analyzer를 정의하지 않고, 필드 데이터 타입을 Text 데이터 타입으로 사용한다면 기본적으로 Standard Analyzer를 사용한다.

공백 혹은 특수 기호를 기준으로 토큰을 분리하고 모든 문자를 소문자로 변경하는 토큰 필터를 사용한다.

#### Whitespace
공백 문자열을 기준으로 토큰을 분리

#### Keyword 분석기

전체 입력 문자열을 하나의 키워드처럼 처리한다. 토큰화 작업을 하지 않는다.

## 3.4.4 전처리 필터

토크나이저 내부에서도 일종의 전처리가 가능하기 때문에 전처리 필터는 상대적으로 활용도가 많이 떨어진다.

전처리 필터의 활용도가 높지 않기 때문에 엘라스틱서치에서 공식적으로 제공하는 전처리 필터의 종류도 많지 않다.

### Html strip char 필터
문장에서 HTML을 제거하는 전처리 필터다.

### 3.4.5 토크나이저 필터

분석기에 어떠한 토크나이저를 사용하느냐에 따라 분석기의 전체적인 성격이 결정된다.

#### Standard 토크나이저
엘라스틱서치에서 일반적으로 사용하는 토크나이저롯 대부분의 기호를 만나면 토큰으로 나눈다.

#### WHITESPACE TOKENIZER
공백을 만나면 텍스트를 토큰화한다.

#### Ngram 토크나이저
Ngram은 기본적으로 한 글자씩 토큰화한다. Ngram에 특정한 문자를 지정할 수도 있으며, 이 경우 지정된 문자의 목록 중 하나를 만날 때마다 단어를 자른다.

```json
POST _analyze
{
  "tokenizer": "ngram",
  "text": "Quick Fox"
}

>>>

{ 
   "tokens":[ 
      { 
         "token":"Q",
         "start_offset":0,
         "end_offset":1,
         "type":"word",
         "position":0
      },
      { 
         "token":"Qu",
         "start_offset":0,
         "end_offset":2,
         "type":"word",
         "position":1
      },
      { 
         "token":"u",
         "start_offset":1,
         "end_offset":2,
         "type":"word",
         "position":2
      },
      { 
         "token":"ui",
         "start_offset":1,
         "end_offset":3,
         "type":"word",
         "position":3
      },
      { 
         "token":"i",
         "start_offset":2,
         "end_offset":3,
         "type":"word",
         "position":4
      },
      { 
         "token":"ic",
         "start_offset":2,
         "end_offset":4,
         "type":"word",
         "position":5
      },
      { 
         "token":"c",
         "start_offset":3,
         "end_offset":4,
         "type":"word",
         "position":6
      },
      { 
         "token":"ck",
         "start_offset":3,
         "end_offset":5,
         "type":"word",
         "position":7
      },
      { 
         "token":"k",
         "start_offset":4,
         "end_offset":5,
         "type":"word",
         "position":8
      },
      { 
         "token":"k ",
         "start_offset":4,
         "end_offset":6,
         "type":"word",
         "position":9
      },
      { 
         "token":" ",
         "start_offset":5,
         "end_offset":6,
         "type":"word",
         "position":10
      },
      { 
         "token":" F",
         "start_offset":5,
         "end_offset":7,
         "type":"word",
         "position":11
      },
      { 
         "token":"F",
         "start_offset":6,
         "end_offset":7,
         "type":"word",
         "position":12
      },
      { 
         "token":"Fo",
         "start_offset":6,
         "end_offset":8,
         "type":"word",
         "position":13
      },
      { 
         "token":"o",
         "start_offset":7,
         "end_offset":8,
         "type":"word",
         "position":14
      },
      { 
         "token":"ox",
         "start_offset":7,
         "end_offset":9,
         "type":"word",
         "position":15
      },
      { 
         "token":"x",
         "start_offset":8,
         "end_offset":9,
         "type":"word",
         "position":16
      }
   ]
}
```

#### Edge Ngram 토크나이저
지정된 문자의 목록 중 하나를 만날 때마다 시작 부분을 고정시켜 단어를 자른다.

```json
POST _analyze
{
  "tokenizer": "edge_ngram",
  "text": "Quick Fox"
}

>>>

{ 
   "tokens":[ 
      { 
         "token":"Q",
         "start_offset":0,
         "end_offset":1,
         "type":"word",
         "position":0
      },
      { 
         "token":"Qu",
         "start_offset":0,
         "end_offset":2,
         "type":"word",
         "position":1
      }
   ]
}

```

#### Keyword 토크나이저
텍스트를 하나의 토큰으로 만든다.

### 3.4.6 토큰필터

토큰 필터는 토크나이저에서 분리된 토큰들을 변형하거나 추가, 삭제할 때 사용하는 필터다.

토크나이저에 의해 토큰이 모두 분리되면 분리된 토큰은 배열 형태로 토큰 필터로 전달된다. 토크나이저에 의해 토큰이 모두 분리돼야 비로소 동작하기 때문에 독립적으로는 사용할 수 없다.

#### Ascii Folding 토큰 필터
아스키 코드에 해당하지 않는 토큰의 문자를 ASCII 요소로 변경한다.

#### Lowercase 토큰 필터
토큰을 구성하는 전체 문자열을 소문자로 변환한다.

#### Uppercase 토큰 필터
토큰을 구성하는 전체 문자열을 대문자로 변환한다.

#### Stop 토큰 필터
불용어로 등록할 사전을 구축해서 사용하는 필터를 의미한다. 인덱스로 만들고 싶지 않거나 검색되지 않게 하고 싶은 단어를 등록해서 해당 단어에 대한 불용어 사전을 구축한다.

#### Stemmer 토큰 필터
Stemming 알고리즘을 사용해 토큰을 변형하는 필터.

Stemming : 어형이 변형된 단어로부터 접사 등을 제거하고 그 단어의 어간을 분리해 내는 것을 의미한다.

어간 : 문법에서 어형 변화의 기초가 되는 부분을 말한다.

#### Synonym 토큰 필터
동의어를 처리할 수 있는 필터

#### Trim 토큰 필터
앞뒤 공백을 제거하는 토큰 필터다.

### 3.4.7 동의어 사전
* 동의어를 매핑 설정 정보에 미리 파라미터로 등록하는 방식 -> 운영 중에는 동의어를 변경하기가 시실상 어렵다.
* 특정 파일을 별도로 생성해서 관리하는 방식

동의어를 관리하기 위해 모아둔 파일들을 칭할 때 일반적으로 "동의어 사전"이라는 용어를 사용한다.

#### 동의어 추가
동의어를 추가할 때 단어를 쉼표(,)로 분리해 등록한다.
```
Elasticsearch, 엘라스틱서치
```

#### 동의어 치환
동의어를 치환하면 원본 토큰이 제거되고 변경될 새로운 토큰이 추가된다. 동의어 치환은 동의어 추가와 구분하기 위해 화살표(=>)로 표시한다.
```
Elasticsearch => 엘라스틱서치
```

---

수정된 동의어를 적용하고 싶다면, 해당 동의어 사전을 사용하고 있는 인덱스를 Reload해야 한다. (동의어 사전의 모든 데이터는 Config 형태로 메모리에서 관리되는데 인덱스를 Reload해야만 이 정보가 갱신된다.)
```
POST <index>/_close
POST <index>/_open
```

동의어 사전의 내용이 변경되더라도, 기존 색인은 변경되지 않는다. 이 경우에는 기존 색인을 모두 삭제하고 색인을 다시 생성해야만 변경된 사전의 내용이 적용된다.

동의어 사전이 검색 시점에 사용된다면, 사전의 내용이 변경되더라도 해당 내용이 반영된다.
