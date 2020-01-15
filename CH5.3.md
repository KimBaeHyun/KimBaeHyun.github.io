## 5.3 버킷 집계

버킷 집계는 쿼리 결과 내에서 집계가 이루어지며, 집계된 결과 데이터 집합을 메모리에 저장한다.(버킷을 생성한다.) 

집계된 버킷은 또 다시 하위에서 집계를 한 번 더 수행해서 집계된 결과에 대해 중첩된 집계를 수행하는 것이 가능하다. 중첩되는 단계가 깊어질수록, 메모리 사용량은 점점 더 증가해서 성능에 악영향을 줄 수 있다.

```json
GET /_search
{
    "aggs" : {
        "price_ranges" : {
            "range" : {
                "field" : "price",
                "ranges" : [
                    { "to" : 100 },
                    { "from" : 100, "to" : 200 },
                    { "from" : 200 }
                ]
            },
            "aggs" : {
                "price_stats" : {
                    "stats" : { "field" : "price" }
                }
            }
        }
    }
}

>>>

{
  ...
  "aggregations": {
    "price_ranges": {
      "buckets": [
        {
          "key": "*-100.0",
          "to": 100.0,
          "doc_count": 2,
          "price_stats": {
            "count": 2,
            "min": 10.0,
            "max": 50.0,
            "avg": 30.0,
            "sum": 60.0
          }
        },
        {
          "key": "100.0-200.0",
          "from": 100.0,
          "to": 200.0,
          "doc_count": 2,
          "price_stats": {
            "count": 2,
            "min": 150.0,
            "max": 175.0,
            "avg": 162.5,
            "sum": 325.0
          }
        },
        {
          "key": "200.0-*",
          "from": 200.0,
          "doc_count": 3,
          "price_stats": {
            "count": 3,
            "min": 200.0,
            "max": 200.0,
            "avg": 200.0,
            "sum": 600.0
          }
        }
      ]
    }
  }
}

```

### 5.3.1 범위 집계

범위 집계는 사용자가 지정한 범위 내에서 집계를 수행하는 다중 버킷 집계다.

집계가 수행되면 추출된 문서가 범위에 해당하는지 검증하게 되고, 범위(from 부터 to 미만의 범위)에 해당하는 문서들에 대해서만 집계가 수행된다.

```json
GET /_search
{
    "aggs" : {
        "price_ranges" : {
            "range" : {
                "field" : "price",
                "ranges" : [
                    { "to" : 100.0 },
                    { "from" : 100.0, "to" : 200.0 },
                    { "from" : 200.0 }
                ]
            }
        }
    }
}

>>>

{
    ...
    "aggregations": {
        "price_ranges" : {
            "buckets": [
                {
                    "key": "*-100.0",
                    "to": 100.0,
                    "doc_count": 2
                },
                {
                    "key": "100.0-200.0",
                    "from": 100.0,
                    "to": 200.0,
                    "doc_count": 2
                },
                {
                    "key": "200.0-*",
                    "from": 200.0,
                    "doc_count": 3
                }
            ]
        }
    }
}
```

* keyed 옵션 : keyed 옵션을 true로 설정하면 고유한 문자열 키가 각 버킷과 연결되고 ranges가 배열이 아닌 해시로 반환된다.


```json
GET /_search
{
    "aggs" : {
        "price_ranges" : {
            "range" : {
                "field" : "price",
                "keyed" : true,
                "ranges" : [
                    { "key" : "cheap", "to" : 100 },
                    { "key" : "average", "from" : 100, "to" : 200 },
                    { "key" : "expensive", "from" : 200 }
                ]
            }
        }
    }
}

>>>

{
    ...
    "aggregations": {
        "price_ranges" : {
            "buckets": {
                "cheap": {
                    "to": 100.0,
                    "doc_count": 2
                },
                "average": {
                    "from": 100.0,
                    "to": 200.0,
                    "doc_count": 2
                },
                "expensive": {
                    "from": 200.0,
                    "doc_count": 3
                }
            }
        }
    }
}
```

### 5.3.2 날짜 범위 집계

날짜 범위 집계는 데이터 범위 집계와 유사하지만, 숫자 값을 범위로 사용했던 범위 집계와는 달리 날짜 값을 범위로 집계를 수행한다.

* 날짜 형식은 엘라스틱서치에서 지원하는 형식만 사용해야 한다.

```json
POST /sales/_search?size=0
{
    "aggs": {
        "range": {
            "date_range": {
                "field": "date",
                "format": "MM-yyy",
                "ranges": [
                    { "from": "01-2015",  "to": "03-2015", "key": "quarter_01" },
                    { "from": "03-2015", "to": "06-2015", "key": "quarter_02" }
                ],
                "keyed": true
            }
        }
    }
}

>>>

{
    ...
    "aggregations": {
        "range": {
            "buckets": {
                "quarter_01": {
                    "from": 1.4200704E12,
                    "from_as_string": "01-2015",
                    "to": 1.425168E12,
                    "to_as_string": "03-2015",
                    "doc_count": 5
                },
                "quarter_02": {
                    "from": 1.425168E12,
                    "from_as_string": "03-2015",
                    "to": 1.4331168E12,
                    "to_as_string": "06-2015",
                    "doc_count": 2
                }
            }
        }
    }
}
```

### 5.3.3 히스토그램 집계

히스토그램 집계는 지정한 수치가 간격을 나타내고, 이 간격의 범위 내에서 집계를 수행한다.

```json
POST /sales/_search?size=0
{
    "aggs" : {
        "prices" : {
            "histogram" : {
                "field" : "price",
                "interval" : 50
            }
        }
    }
}

>>>

{
    ...
    "aggregations": {
        "prices" : {
            "buckets": [
                {
                    "key": 0.0,
                    "doc_count": 1
                },
                {
                    "key": 50.0,
                    "doc_count": 1
                },
                {
                    "key": 100.0,
                    "doc_count": 0
                },
                {
                    "key": 150.0,
                    "doc_count": 2
                },
                {
                    "key": 200.0,
                    "doc_count": 3
                }
            ]
        }
    }
}
```
만약 결과가 존재하지 않는 구간은 필요 없다면, 최소 문서 수(min_doc_count)를 설정해서 해당 구간을 제외시킬 수 있다.

### 5.3.4 날짜 히스토그램 집계

날짜 히스토그램 집계는 분, 시간, 월 연도를 구간으로 집계를 수행할 수 있다.

---
#### Deprecated in 7.2. interval field is deprecated

##### Calendar intervals

* unit
```
minute (m, 1m)
hour (h, 1h)
day (d, 1d)
week (w, 1w)
month (M, 1M)
quarter (q, 1q)
year (y, 1y)
```

> 단수 unit만 사용 가능하다. 복수 unit은 지원되지 않는다.

```json
POST /sales/_search?size=0
{
    "aggs" : {
        "sales_over_time" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "month"
            }
        }
    }
}



POST /sales/_search?size=0
{
    "aggs" : {
        "sales_over_time" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "2d"
            }
        }
    }
}

>>>

{
  "error" : {
    "root_cause" : [...],
    "type" : "x_content_parse_exception",
    "reason" : "[1:82] [date_histogram] failed to parse field [calendar_interval]",
    "caused_by" : {
      "type" : "illegal_argument_exception",
      "reason" : "The supplied interval [2d] could not be parsed as a calendar interval.",
      "stack_trace" : "java.lang.IllegalArgumentException: The supplied interval [2d] could not be parsed as a calendar interval."
    }
  }
}
```

##### Fixed intervals

* unit
```
milliseconds (ms)
seconds (s)
minutes (m)
hours (h)
days (d)
```

> 월 또는 분기와 같은 Calendar intervals unit 을 사용할 수 없다. 

```json
POST /sales/_search?size=0
{
    "aggs" : {
        "sales_over_time" : {
            "date_histogram" : {
                "field" : "date",
                "fixed_interval" : "30d"
            }
        }
    }
}



POST /sales/_search?size=0
{
    "aggs" : {
        "sales_over_time" : {
            "date_histogram" : {
                "field" : "date",
                "fixed_interval" : "2w"
            }
        }
    }
}

>>>

{
  "error" : {
    "root_cause" : [...],
    "type" : "x_content_parse_exception",
    "reason" : "[1:82] [date_histogram] failed to parse field [fixed_interval]",
    "caused_by" : {
      "type" : "illegal_argument_exception",
      "reason" : "failed to parse setting [date_histogram.fixedInterval] with value [2w] as a time value: unit is missing or unrecognized",
      "stack_trace" : "java.lang.IllegalArgumentException: failed to parse setting [date_histogram.fixedInterval] with value [2w] as a time value: unit is missing or unrecognized"
    }
  }
}
```
---

offset을 사용하면 집계 기준이 되는 날짝 의 시작 일자를 조정할 수 있다.

### 5.3.5 텀즈 집계

텀즈 집계는 집계 시 지정한 필드에 대해 빈도수가 높은 텀의 순위로 결과가 반환된다.

텀즈 집계의 필드 값으로는 Keyword 데이터 타입의 필드를 사용해야 한다. Text 데이터 타입의 경우 형태소 분석기를 통해 분석하는 과정이 항상 동반되기 때문에 집계할 때는 형태소 분석이 필요 없는 Keyword 데이터 타입을 사용해야만 한다.

```json
GET /_search
{
    "aggs" : {
        "genres" : {
            "terms" : { "field" : "genre" } 
        }
    }
}

>>>

{
    ...
    "aggregations" : {
        "genres" : {
            "doc_count_error_upper_bound": 0, 
            "sum_other_doc_count": 0, 
            "buckets" : [ 
                {
                    "key" : "electronic",
                    "doc_count" : 6
                },
                {
                    "key" : "rock",
                    "doc_count" : 3
                },
                {
                    "key" : "jazz",
                    "doc_count" : 2
                }
            ]
        }
    }
}
```

집계를 수행할 떄는 각 샤드에 집계 요청을 전달하고, 각 샤드는 집계 결과에 대해 정렬을 수행한 자체 뷰를 갖게 된다.

이것들을 병합해서 최종 뷰를 만들기 때문에 포함되지 않은 문서가 존재하는 경우에는 집계 결과가 정확하지 않을 수 있다.

-> size 값을 늘리면 집계의 정확도가 높아지지만, 메모리 사용량과 결과를 계산하는데 드는 처리 비용 또한 증가한다.

## 5.4 파이프라인 집계

파이프라인 집계는 다른 집계와 달리 쿼리 조건에 부합하는 문서에 대해 집계를 수행하는 것이 아니라 다른 집계로 생성된 버킷을 참조해서 집계를 수행한다.

파이프라인 집계를 수행할 때는 buckets_path 파라미터를 사용해 참조할 집계의 경로를 지정함으로써 체인 형식으로 집계 간의 연산이 이뤄진다. 

파이프라인 집계는 하위 집계를 가질 수는 없지만 다른 파이프라인 집계와는 buckets_path를 통해 참조하도록 지정할 수 있다.

```
AGG_SEPARATOR       =  `>` ;
METRIC_SEPARATOR    =  `.` ;
AGG_NAME            =  <the name of the aggregation> ;
METRIC              =  <the name of the metric (in case of multi-value metrics aggregation)> ;
MULTIBUCKET_KEY     =  `[<KEY_NAME>]`
PATH                =  <AGG_NAME><MULTIBUCKET_KEY>? (<AGG_SEPARATOR>, <AGG_NAME> )* ( <METRIC_SEPARATOR>, <METRIC> ) ;
```

### 5.4.1 형제 집계

형제 집계는 동일 선상의 위치에서 수행되는 새 집계를 의미한다. 

즉, 형제 집계를 통해 수행되는 집계는 기존 버킷에 추가되는 형태가 아니라 동일 선상의 위치에 새 집계가 생성되는 파이프라인 집계다.

```
평균 버킷 집계
최대 버킷 집계
최소 버킷 집계
합계 버킷 집계
통계 버킷 집계
확장 통계 버킷 집계
백분위수 버킷 집계
이동 평균 집계
```

```json
POST /sales/_search
{
    "size": 0,
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "month"
            },
            "aggs": {
                "sales": {
                    "sum": {
                        "field": "price"
                    }
                }
            }
        },
        "max_monthly_sales": {
            "max_bucket": {
                "buckets_path": "sales_per_month>sales" 
            }
        }
    }
}

>>>

{
   "took": 11,
   "timed_out": false,
   "_shards": ...,
   "hits": ...,
   "aggregations": {
      "sales_per_month": {
         "buckets": [
            {
               "key_as_string": "2015/01/01 00:00:00",
               "key": 1420070400000,
               "doc_count": 3,
               "sales": {
                  "value": 550.0
               }
            },
            {
               "key_as_string": "2015/02/01 00:00:00",
               "key": 1422748800000,
               "doc_count": 2,
               "sales": {
                  "value": 60.0
               }
            },
            {
               "key_as_string": "2015/03/01 00:00:00",
               "key": 1425168000000,
               "doc_count": 2,
               "sales": {
                  "value": 375.0
               }
            }
         ]
      },
      "max_monthly_sales": {
          "keys": ["2015/01/01 00:00:00"], 
          "value": 550.0
      }
   }
}
```

## 5.4.2 부모 집계

부모 집계는 집계를 통해 생성된 버킷을 사용해 계산을 수행하고, 그 결과를 기존 집계에 반영한다.

```
파생 집계
누적 집계
버킷 스크립트 집계
버킷 셀렉터 집계
시계열 차분 집계
```

파생 집계는 부모 히스토그램 또는 날짜 히스토그램 집계의 측정 항목에 대해 작동하고, 집계에 의한 각 버킷의 집계 값을 비교해서 차이를 계산한다.

```json
POST /sales/_search
{
    "size": 0,
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "month"
            },
            "aggs": {
                "sales": {
                    "sum": {
                        "field": "price"
                    }
                },
                "sales_deriv": {
                    "derivative": {
                        "buckets_path": "sales" 
                    }
                }
            }
        }
    }
}

>>>

{
   "took": 11,
   "timed_out": false,
   "_shards": ...,
   "hits": ...,
   "aggregations": {
      "sales_per_month": {
         "buckets": [
            {
               "key_as_string": "2015/01/01 00:00:00",
               "key": 1420070400000,
               "doc_count": 3,
               "sales": {
                  "value": 550.0
               } 
            },
            {
               "key_as_string": "2015/02/01 00:00:00",
               "key": 1422748800000,
               "doc_count": 2,
               "sales": {
                  "value": 60.0
               },
               "sales_deriv": {
                  "value": -490.0 
               }
            },
            {
               "key_as_string": "2015/03/01 00:00:00",
               "key": 1425168000000,
               "doc_count": 2, 
               "sales": {
                  "value": 375.0
               },
               "sales_deriv": {
                  "value": 315.0
               }
            }
         ]
      }
   }
}
```

## 5.5 근사값으로 제공되는 집계 연산

대부분의 경우 전체 데이터를 대상으로 집계가 일어나기 때문에 결과도 매우 정확하게 제공된다. 하지만 일부 집계 연산의 경우에는 성능 문제로 근삿값을 기반으로 한 결과를 제공한다.

### 5.5.1 집계 연산과 정확도

```
카디널리티 집계
백분위 수 집계
백분위 수 랭크 집계
```
> 근삿값을 제공하는 집계 연산이다.

### 5.5.2 분산 환경에서 집계 연산의 어려움

집계는 요청이 들어올 경우 각 샤드 내에서 1차적으로 집계 작업이 수행되고, 중간 결과를 모두 취합해서 최종 결과를 만들어 제공하는 방식으로 동작한다.

검색 작업도 집계 잡업과 같은 방식으로 처리되기 때문에 한 번의 실행으로 검색 결과와 집계 결과를  제공받을 수 있어 성능 측면에서도 효율적이다.

일반 집계의 경우는각 샤드에서 중간 집계 결과로 계산 값만 코디네이터 노드로 전송하면 되는 반면 카디널리티 집계의 경우 중복을 제거한 데이터 리스트를 코디네이터 노드로 전송해야 한다.

이처럼 최종 결과를 도출하기 위해 일정 크기의 데이터를 전송해야만 하는 집계 연산의 경우 엘라스틱서치의 기본 동작 방식이 단점으로 작용한다.
