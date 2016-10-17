[TOC]

# ELK(Elasticsearch-Logstash-Kibana)

## 실습 계정 & 디렉토리 생성

```bash
$ useradd elk
$ su elk
$ cd
```



## 지하철 승하차 승객 수 데이터 다운로드

*seoul-metro-2014-source.zip* 을 다운로드 받아 적절한 디렉토리 압축을 해제한다.

```bash
$ wget https://github.com/hozoon/practice-collecting/raw/master/seoul-metro-2014-source.zip
$ unzip seoul-metro-2014-source.zip
$ cd data
$ touch *
```

이 데이터는 공공데이터 중 "서울시 역코드로 지하철역 위치 조회", "서울시 지하철 다국어 (한글,한자,영어,중국어,일어) 역명 정보", "역별 시간대별 승하차 인원 현황(2014년)" 데이터를 결합하여 하나의 json 파일 형태로 변환한 것이다.

| 필드명              | 타입        | 설명                 |
| ---------------- | --------- | ------------------ |
| time_slot        | datetime  | 승/하차 시간. 1시간단위     |
| line_num         | string    | 호선                 |
| line_num_en      | string    | 호선(영문)             |
| station_name     | string    | 역 이름 : 잠실          |
| station_name_kr  | string    | 역 이름 전체 : 잠실(송파구청) |
| station_name_en  | string    | 역 이름 영문            |
| station_name_chc | string    | 역 이름 한자            |
| station_name_ch  | string    | 역 이름 중국어 간체        |
| station_name_jp  | string    | 역 이름 일본어           |
| station_geo      | geo_point | 역 좌표               |
| people_in        | integer   | 승차인원               |
| people_out       | integer   | 하차인원               |



## elasticsearch

### 다운로드 & 설치

```bash
$ wget https://download.elastic.co/elasticsearch/release/org/elasticsearch/distribution/tar/elasticsearch/2.4.0/elasticsearch-2.4.0.tar.gz
$ tar zxvf elasticsearch-2.4.0.tar.gz
```

### 설정

```bash
$ cd ~/elasticsearch-2.4.0
$ vi config/elasticsearch.yml

cluster.name: elk-bigdata
network.host: 0.0.0.0
```

### head plugin 설치

haed plugin 을 설치함으로써 elasticsearch 클러스트를 gui 형태로 확인할 수 있다.

```bash
$ bin/plugin install mobz/elasticsearch-head
```

### 실행

```bash
$ bin/elasticsearch -d
```

### 실행 상태 확인

 웹 브라우저를 실행한 후 *http://localhost:9200/_plugin/head* 페이지가 정상적으로 표시되는지 확인한다.

### index 생성 & mapping 등록

일종의 DB 스키마와 같은 것으로써 미리 인덱스의 스키마를(필드명, 데이터타입 등) 정의해놓을 수 있다. mapping 을 등록해놓지 않으면 처음 들어오는 데이터를 보고 elasticsearch가 자동으로 mapping 을 생성한다.

```bash
$ curl -XPUT 'localhost:9201/seoul-metro-2014' -d '
  {
    "mappings" : {
      "seoul-metro" : {
        "properties" : {
          "time_slot" : { "type" : "date" },
          "line_num" : { "type" : "string", "index" : "not_analyzed" },
          "line_num_en" : { "type" : "string", "index" : "not_analyzed" },
          "station_name" : { "type" : "string", "index" : "not_analyzed" },
          "station_name_kr" : { "type" : "string", "index" : "not_analyzed" },
          "station_name_en" : { "type" : "string", "index" : "not_analyzed" },
          "station_name_chc" : { "type" : "string", "index" : "not_analyzed" },
          "station_name_ch" : { "type" : "string", "index" : "not_analyzed" },
          "station_name_jp" : { "type" : "string", "index" : "not_analyzed" },
          "station_geo" : { "type" : "geo_point" },
          "people_in" : { "type" : "integer" },
          "people_out" : { "type" : "integer" }
        }
      }
    }
  }'
```



## logstash

### logstash 다운로드 & 설치

```bash
$ cd
$ wget https://download.elastic.co/logstash/logstash/logstash-2.4.0.tar.gz
$ tar zxvf logstash-2.4.0.tar.gz
```

### logstash 설정 파일 작성

```bash
$ cd logstash-2.4.0
$ mkdir conf
$ vi conf/subway.conf
```

```ruby
input {
  file {
    codec => json
    path => "/home/elk/data/*.log"
    start_position => "beginning"
  }
}

filter{
  mutate {
    remove_field => [ "@version", "@timestamp", "host", "path" ]
  }
}

output{
  elasticsearch{
    hosts => ["localhost:9201"]
    index => "seoul-metro-2014"
    document_type => "seoul-metro"
  }
}
```

### 설정 파일 테스트

```bash
$ bin/logstash -f conf/subway.conf -t
```

### 실행

```bash
$ bin/logstash -f conf/subway.conf
```



## kibana

### 다운로드 & 설치

``` bash
$ cd
$ wget https://download.elastic.co/kibana/kibana/kibana-4.6.0-linux-x86_64.tar.gz
$ tar zxvf kibana-4.6.0-linux-x64.tar.gz
```

### 설정

``` bash
$ vi config/kibana.yml

elasticsearch.url: "http://localhost:9201"
```

### 실행

``` bash
$ cd kibana-4.6.0-linux-x64
$ bin/kibana
```

### 접속

웹 브라우저를 통해 *http://localhost:5601* 페이지로 접속한다.

index name or pattern 입력란에 seoul-metro-2014 를 입력하면 Time-filed name 에 자동으로 time_slot 이 설정된다.

 ![kibana1](/Users/yeobi/Desktop/kibana1.png)



craete 버튼을 누르면 보고자 하는 index 가 kibana 에 등록된다.

### 데이터 탐색

상단 메뉴바에서 Discover 를 선택 후, 오른쪽 상단의 시간 조건을 2014/01/01 ~ 2014/12/31 으로 변경한다. (데이터가 2014년 데이터이므로) 

![kibana2](/Users/yeobi/Desktop/kibana2.png)

### visualization 만들기

상단 메뉴바에서 Visualize 메뉴를 선택 하면, 비주얼 컴포넌트를 생성할 수 있는 화면으로 이동된다.

이 메뉴에서는 데이터를 차트, 테이블, 통계정보, 지도 등의 형태로 시각화 할 수 있는 기능을 제공한다.

 ![kibana3](/Users/yeobi/Desktop/kibana3.png)

#### metric

시각화 도구 중, metric 을 선택한 후, From a new search 를 선택하고, 아래 그림과 같이 2014년에 지하철 승하차한 사용자의 각각의 합을 구하고 지표로 표현한다.

 ![kibana4](/Users/yeobi/Desktop/kibana4.png)

위와 같이 설정 후, save 버튼을 클릭하여 적절한 이름을 지정하고 저장한다.

(설저을 완료하고 나면, play 버튼을 클릭해주어야 실제 시각화 내용이 그려진다.)



#### line chart

시각화 도구 중, line chart 를 선택한 후, From a new search 를 선택하고, 아래 그림과 같이 지하철 승하차한 사용자의 각각의 합을 라인 차트로 표현한다.

 ![kibana5](/Users/yeobi/Desktop/kibana5.png)

위와 같이 설정 후, save 버튼을 클릭하여 적절한 이름을 지정하고 저장한다.

### pie chart ![kibana6](/Users/yeobi/Desktop/kibana6.png)

###  tile map

 ![kibana7](/Users/yeobi/Desktop/kibana7.png)





### dashboard 만들기

 상단 메뉴 모음의 Dashboard 를 클락하여 dashboard 페이지로 이동한다.

다음과 같이 적절히 배치하여 Dashboard를 꾸민 후, 저장한다. ![kibana8](/Users/yeobi/Desktop/kibana8.png)