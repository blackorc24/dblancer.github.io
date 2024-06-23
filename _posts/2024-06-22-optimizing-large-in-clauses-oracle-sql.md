---
title: 바인드 변수가 너무 많은 쿼리
author: me
date: 2024-06-21 16:53:00 +0800
categories: [Oracle,SQL개선]
tags: [파싱타임개선]
---

## 문제   
평소엔 1초도 안걸리는데 종종 쿼리가 엄청 느려진다는 요청을 받는다.   
어떤 쿼리인가 보니 바인드 변수 갯수가 그 때 그 때 마다 변하고     
추가되는 바인드 변수가 수십개에서 수천개가 사용되니 종종 다시 파싱하느라 오래 걸리는 것이다.    

## 원인   

필요한 대상을 선택하고 그 대상들에 대한 결과를 보여주는 화면이여서 선택된 대상의 키 데이터가 모두 바인드 변수로 들어와야했다.


## 해결

XML형태로 변수를 입력받아서 테이블 형태로 가공한다

``` 
WITH xmldata AS ( 
    SELECT XMLTYPE( TO_CLOB(:xml_bind_01)
                    || TO_CLOB(:xml_bind_02) ) xml_data 
      FROM DUAL
)
SELECT *
  FROM TAB1 A
 WHERE A.COL1 IN (
                 SELECT EXTRACTVALUE(COLUMN_VALUE,'/v')
                   FROM TABLE(
                              XMLSEQUENCE(
                                         EXTRACT(
                                                (SELECT xml_data
                                                   FROM xmldata)
                                               ,'/values/v')
                                         )
                              )
                 )
; 
```

변수를 xml_bind_01 와 xml_bind_02 두개를 썼는데 varchar타입의 길이 제한으로 인해   
더 길게 입력하기 위해서 두개를 썼다.   

바인드 변수는 ```<values>``` 로 시작해서 ```<v>데이터</v>``` 로 변수 데이터를 채우고   
마지막은 ```</values>```로 닫아주면 된다.   

```xml_bind_01``` 에는 ```<values><v>data1</v><v>data2</v>```       
```xml_bind_02``` 에는 ```<v>data3</v><v>data4</v></values>```    

이런식으로 채워서 하나의 문장으로 보고 태그를 잘 닫아주면 된다   
 

물론 변수를 한개만 쓰고 안쓰는건 NULL로 채워도 그만이다.  


변수를 여러개로 쓰고 싶지 않다면   

```ALTER SYSTEM SET max_string_size = EXTENDED SCOPE = SPFILE;```   

의 파라메터를 설정해서 VARCHAR2의 4000바이트 한계를 확장시키는 방법도 있다. 

