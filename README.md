# Overview
This repo is my ***ELK*** (Elasticsearch, Logstash, Kibana) practice: use ***ELK*** to visualize data form 공공데이터 open API (data.go.kr)

![Picture4](https://user-images.githubusercontent.com/79763166/186461600-36f67472-2085-4c07-8c5f-a2b7e6fafe71.png)

data.go.kr supports open API that can use for free after resgistration

**Enviroment**:
- Azure Virtual Machine: Linux (ubuntu 20.04)
- Elasticsearch: version 7.17.5 http://20.249.85.198:9200/

# Data
[Data](https://www.data.go.kr/data/15098931/openapi.do): Ministry of Agriculture, Food and Rural Affairs, Agriculture, Forestry and Livestock Quarantine Headquarters - Animal Protection Management System Abandoned Animal Information - Korea (농림축산식품부 농림축산검역본부 - 동물보호관리시스템 유기동물 정보)

 1. Input and format data
 
 For each query from open API, we can get 1 page of 1000 rows data in one time. Our code download one by one page and append data into our MySQL database `Abandoment`


 ***Input***
 
 We use `sqlalchemy` library in Python to connect with our AWS server.
  
 ```python
db_connection_str = 'mysql+pymysql://admin:i4GSOM8GCjRfDyV@vteam6.cbr4uubmqr4e.ap-northeast-2.rds.amazonaws.com/Abandoment'
db_connection = create_engine(db_connection_str)
conn = db_connection.connect()
 ```
 
 ***Format***
 
 In the step of input, we format data before input into MySQL database. Use the format as in the instruction file (data.go.kr)
 ```python
 dtypesql = {'desertionNo':sqlalchemy.types.VARCHAR(15),
            'filename':sqlalchemy.types.VARCHAR(100), 
            'happenDt':sqlalchemy.Date(), #VARCHAR(8)
            'kindCd':sqlalchemy.types.VARCHAR(50),
            'colorCd':sqlalchemy.types.VARCHAR(30), 
            'age':sqlalchemy.types.VARCHAR(30), 
            'weight':sqlalchemy.types.VARCHAR(30),
            'noticeNo':sqlalchemy.types.VARCHAR(30),
            'noticeSdt':sqlalchemy.types.Date(), #VARCHAR(8)
            'noticeEdt':sqlalchemy.types.Date(), #VARCHAR(8)
            'popfile':sqlalchemy.types.VARCHAR(100),
            'processState':sqlalchemy.types.VARCHAR(10),
            'sexCd':sqlalchemy.types.VARCHAR(1),
            'neuterYn':sqlalchemy.types.VARCHAR(1),
            'specialMark':sqlalchemy.types.VARCHAR(200),
            'careNm':sqlalchemy.types.VARCHAR(50),
            'careTel':sqlalchemy.types.VARCHAR(14),
            'careAddr':sqlalchemy.types.VARCHAR(200),
            'orgNm':sqlalchemy.types.VARCHAR(50),
            'chargeNm':sqlalchemy.types.VARCHAR(20),
            'officetel':sqlalchemy.types.VARCHAR(14),
            'noticeComment':sqlalchemy.types.VARCHAR(200),
}
df.to_sql(name='aug', con=db_connection, if_exists='append', index=False,dtype=dtypesql)
 ```
*MySQL table:*

<img width="1080" alt="Screenshot 2022-09-15 034841" src="https://user-images.githubusercontent.com/79763166/190238822-fbd36e3e-4209-46da-a333-f95f0946847b.png" style="width:800px;"/>

> This data supplies the information on abandoned animals of the animal protection management system, and provides city and county inquiry API, city and county inquiry API, shelter inquiry API, and breed inquiry API.

In this repo, I use only the breed inquiry API.

`json` format example of breed inquiry API:
```python
"body" : {
      "items" : {
        "item" : [ {
          "desertionNo" : "446485202200334",
          "filename" : "http://www.animal.go.kr/files/shelter/2022/08/202208161408827_s.jpg",
          "happenDt" : "20220730",
          "happenPlace" : "고서면 동운리 287-12",
          "kindCd" : "[개] 믹스견",
          "colorCd" : "흰색",
          "age" : "2019(년생)",
          "weight" : "18(Kg)",
          "noticeNo" : "전남-담양-2022-00219",
          "noticeSdt" : "20220816",
          "noticeEdt" : "20220830",
          "popfile" : "http://www.animal.go.kr/files/shelter/2022/08/202208161408827.jpg",
          "processState" : "보호중",
          "sexCd" : "M",
          "neuterYn" : "N",
          "specialMark" : "온순함, 파보 음성, 피부에 상처",
          "careNm" : "담양군 동물보호센터",
          "careTel" : "010-3604-1338",
          "careAddr" : "전라남도 담양군 용면 시암골로 280-57 (용면) :용면 두장리 21번지",
          "orgNm" : "전라남도 담양군",
          "chargeNm" : "송문성",
          "officetel" : "061-380-2737",
```
## Input - Filter - Ouput with Logstash
```python
input {
  http_poller {
    urls => {
      url => "http://apis.data.go.kr/1543061/abandonmentPublicSrvc/abandonmentPublic?bgnde=20220701&endde=20220730&pageNo=1&numOfRows=1000&_type=json&serviceKey=YWRncUmVrCMs8Nm7xSfh8Q2j0ao587l3WP2M%2Fl4uSYyNrN16%2BLo65V66u%2BIP1QUCWNki6e%2B6ejjwEaUJ%2BIyaew%3D%3D"
    }
    request_timeout => 60
    schedule => { every } => "30s" #test with 30s, this period should be 24h
    codec => "json"
  }
}
```
`bgnde`: start date of query (July 1st);
`endde`: end date of query (July 30th);
`numOfRows=1000`1000 is maximum rows for each query
```python
filter {
  mutate {
     remove_field => [ "[response][body][items][item][popfile]" ] #remove unnesscery field
  }
  date {
     match => { "[response][body][items][item][happlenDt]", "yyyyMMdd" } #format as date
  }
}
```
**Somehow, filter does not work, I could not delete and format these fields but Logstash runs and transfers data to Kibana normaly**
```python
output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "realtime-subway"
  }
  stdout{
    codec => rubydebug
  }
}
```
Run .conf file in Virtual Machine
```python
sudo /usr/share/logstash/bin/logstash -f abandonment.conf
```

## Visualization with Kibana
Dev Tools confirm
```python
get abadoment/_search
```
![Screenshot](https://user-images.githubusercontent.com/79763166/186496665-b442f9b2-6fbf-40d4-8582-c071ace20227.jpg)

Kibana Dashboard http://20.249.85.198:5601/app/dashboards#/view/137d9280-23db-11ed-82db-a3cc495457dd?_g=(filters:!(),refreshInterval:(pause:!t,value:0),time:(from:now-15h,to:'2022-08-24T18:32:39.155Z'))

***Here, because we can take maximum of 1000 rows and 4 pages of data, so it's not enough to show the difference in chart.***
