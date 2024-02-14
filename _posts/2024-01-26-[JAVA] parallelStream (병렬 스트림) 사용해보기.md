---
title: "[JAVA] - parallelStream (병렬 스트림) 사용해보기"
aliases: 
tags:
  - java
  - 성능
categories:
  - etc
toc: true
toc_label: 목차
date: 2024-01-26
last_modified_at: 2024-02-14
---
Java 8 이상 사용 가능한 Stream을 병렬로 사용할 수 있는 메서드.
회사에서 마이그레이션 돌릴 려고 데이터 꺼내서 살짝 가공 후 넣는 코드를 작성했는데
너무 느려서 개선해봄.

```java
public static void main(String[] args) {  
    logger.info("vod syncronized start");  
    List<Map<String, Object>> vodList = null;  
    try {  
       vodList = SyncDAO.vodList();  
    } catch (SQLException e) {  
       logger.warn("error ", e);  
    }  
    logger.info("vod sync list size"+vodList.size());  
      
    Map<String, Object> vodInfo = new HashMap<>();  
  
       for (Map<String, Object> stringObjectMap : vodList) {  
  
           String path = stringObjectMap.get("PATH").toString();  
           if (path.startsWith("/vod/encoding/") && path.matches("/vod/encoding/\\d{4}/\\d{2}/.*")) {  
               path = path.replaceFirst("/vod", "/oldUpload");  
           }  
			  ... 
           vodInfo.put("CROP_YN", "N");  
           vodInfo.put("GIF_FILE_NAME", null);  
  
           try {  
               SyncDAO.vodInsert(vodInfo);  
           } catch (SQLException e) {  
               logger.warn("error ", e);  
           } finally {  
               logger.info("vod sync insert success");  
           }  
  
           vodInfo.put("ENCODING_FILE_NAME", stringObjectMap.get("ENCODING_FILE_NAME").toString().replace("_media1.mp4", "_thumb1.png"));  
			... 
           vodInfo.put("MEDIA_USE", "Y");  
           vodInfo.put("REG_USERID", "tgadmin");  
			...
           try {  
               SyncDAO.vodThumbInsert(vodInfo);  
           } catch (SQLException e) {  
               logger.warn("error ", e);  
           } finally {  
               logger.info("vod sync thumb insert success");  
           }  
       }  
}
```
대충 forEach 로 List 에 담긴 데이터 2만건을 두 테이블에 insert 하는 로직

foreEach 는 약 5분 정도 걸렸다.

![](https://i.imgur.com/NNBeppV.png)

![](https://i.imgur.com/hcRA9ly.png)

이걸 parallelStream 를 사용하여 바꿔 보면 코드는

```java
public static void main(String[] args) {  
    logger.info("vod syncronized  parallelStream start");  
  
    try {  
        List<Map<String, Object>> vodList = SyncDAO.vodList();  
  
        vodList.parallelStream()  
                .forEach(SyncActionTest::insertNewDb);  
    } catch (Exception e) {  
        logger.warn("error ", e);  
    }  
  
}


private static void insertNewDb(Map<String, Object> stringObjectMap) {  
    String path = stringObjectMap.get("PATH").toString();  
    if (path.startsWith("/vod/encoding/") && path.matches("/vod/encoding/\\d{4}/\\d{2}/.*")) {  
        path = path.replaceFirst("/vod", "/oldUpload");  
    }  
    Map<String, Object> vodInfo = new HashMap<>();  
  
    logger.info("vod sync inser mediaid : " + stringObjectMap.get("MEDIA_ID"));  
    vodInfo.put("MEDIA_ID", stringObjectMap.get("MEDIA_ID"));  
    vodInfo.put("ORIGINAL_FILE_PATH", stringObjectMap.get("ORIGINAL_FILE_PATH"));  
    ... 

	SyncDAO.vodInsert(vodInfo);
}
```
메소드 분리 하였고


```java
vodList.parallelStream()  
                .forEach(SyncActionTest::insertNewDb);  
```
다음과 같이 리스트를 parallelStream 으로 만들고  forEach 를 통해 각각 데이터에 
insertNewDb 메소드를 실행해 줬다.

이 결과

![](https://i.imgur.com/dsNWECQ.png)
![](https://i.imgur.com/vvCVKWi.png)

약 1분 30초로 대폭 빨라졌다.

물론 parallelStream 는 병렬 처리를 하기 때문에 순서나, 여러 쓰레드에서 접근 가능한 변수 등이 중요하면 사용할 때 고민해봐야 할 것 같다.


> 참고 : 그냥 Stream은 5분 걸렸다.
