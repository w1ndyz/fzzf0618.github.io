---
layout: post
title:  “Elasticsearch scroll 游标的使用!”
date:   2019-03-22 19:36:59
author: Chris Wang
categories: original
tags: es elasticsearch scroll 游标 
---

#### 记一次elasticsearch Java client的分页问题
最近在写关于数据导出的工具，用到了`kafka`， `flume`， `mysql`， `oracle`， `sql_server`, `mongodb`。其中遇到了很多的小问题，总结一下，以防今后遇到。首先来谈一下es的分页问题:

##### 问题是如何产生的？

首先在查询es的时候是可以用from，size这种查询方式的。但是在查询结果大于1w条的时候，他就会不起作用了，这个时候我们要使用`scroll`。

##### 如何使用scroll？

`scroll`的原理就是，在查询es的时候，先固定一个时间片段的数据，然后再这个时间片段的数据中反复进行分页查询。查询需要定义一个size。在每一次查询的过程中，会返回一个`scroll_id`，是一串很长的加密后的字符串。在下一次查询的过程中，带上这个`scroll_id`，它就会继续上一次的查询往下查`size`条数据。
我们在官方的文档中可以看到scroll的用法是:

``````json
  POST /twitter/_search?scroll=1m
  {
      "size": 100,
      "query": {
          "match" : {
              "title" : "elasticsearch"
          }
      }
  }


  POST /_search/scroll [image:60C7D524-9CBC-40B1-84F9-166B27D9DF51-309-00008DA230D90CB7/1.png]
  {
      "scroll" : "1m", [image:85F4DBA7-6D48-46B6-98A4-681E1C150FE7-309-00008DA230A98BE4/2.png]
      "scroll_id" : "DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAAD4WYm9laVYtZndUQlNsdDcwakFMNjU1QQ==" [image:F7277840-004F-4F16-91DD-62F5D6BFF7D7-309-00008DA230591EE5/3.png]
  }
``````

其中`scroll = 1m`的意思是，本次查询的时间挂起的长度为1分钟，在这一分中的时间里，数据是固定的，不会改变的，即使有新写入的数据，也不会在这个数据段中。

##### Java Client的写法是什么样的？

直接上代码:
``````java
  List<Map<String, Object>> resultList = new ArrayList<>();
  private static RestHighLevelClient restHighLevelClient; #我这里用的是restClient,网上有很多用的client,prepare...()都可以。
  SearchResponse response = null;
  SearchRequest searchRequest = new SearchRequest(ES_INDEX_NAME); #查询的索引名称
  SearchSourceBuilder searchSourceBuilder = SearchSourceBuilder.searchSource();
  String[] includeFields = fieldString.split(","); #需要查询的字段["user_id", "user_name", "create_at"......]
  String[] excludeFields = new String[] {""};
  BoolQueryBuilder boolQueryBuilder = QueryBuilders.boolQuery();
  RangeQueryBuilder timeRange = QueryBuilders.rangeQuery(timeField);
  timeRange.from(start_time); #查询的开始时间
  timeRange.to(en_time); #查询的结束时间
  boolQueryBuilder.must(timeRange);
  searchSourceBuilder.fetchSource(includeFields, excludeFields);
  searchSourceBuilder.query(boolQueryBuilder);
  // 因为数据量过大（超过1w），需要用scroll游标
  final Scroll globalScroll = new Scroll(TimeValue.timeValueMinutes(1L));
  searchSourceBuilder.size((int) pullCount); #拉取数据的size
  searchRequest.scroll(globalScroll); #设置游标
  searchRequest.source(searchSourceBuilder);
  try {
      response = restHighLevelClient.search(searchRequest);
  } catch (IOException e) {
      e.printStackTrace();
  }
  parseSearchResponse(response); #处理分页的结果

  //获取总数量
  long totalCount = response.getHits().getTotalHits();
  int page= (int) ((int)totalCount/pullCount);
  String scrollId = "";
  for (int i = 0; i <= page; i++) {
      //再次发送请求,并使用上次搜索结果的ScrollId
      scrollId = response.getScrollId();
      System.out.println("scrollId" + scrollId);
      SearchScrollRequest scrollRequest = new SearchScrollRequest(scrollId);
      scrollRequest.scroll(globalScroll);
      response = restHighLevelClient.searchScroll(scrollRequest);
      parseSearchResponse(response); #处理分页的结果
  }
  // 清空scroll游标
  ClearScrollRequest clearScrollRequest = new ClearScrollRequest();
  clearScrollRequest.addScrollId(scrollId);
  ClearScrollResponse clearScrollResponse = restHighLevelClient.clearScroll(clearScrollRequest);
``````

下面是查询的方法:

``````java
// 处理分页结果
private void parseSearchResponse(SearchResponse response) {
    List<Map<String, Object>> resultList = new ArrayList<>();
    SearchHit[] searchHits = response.getHits().getHits();
    // 解析当前查询结果
    for(SearchHit thisHits : searchHits){
        Map<String, Object> map = thisHits.getSourceAsMap();
        resultList.add(map);
    }
  resultList;
}

``````

##### 总结:

 在游标查询的时候，我们可以先查询一次结果，然后分片(`page`)，每片遍历去查。直到page查询完毕。得到所有的结果。