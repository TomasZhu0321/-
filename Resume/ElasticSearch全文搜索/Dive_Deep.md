• Implemented Elasticsearch full-text search for government data on the homepage search box, enabling accurate keyword-based article queries with keyword highlighting, and significantly boosting search performance.

# What I do
## Elasticsearch 的具体搜索 + 初始化脚本
* Elasticsearch 的具体搜索功能开发：包括查询逻辑、分页、搜索高亮、分词器设置等内容。
    * 查询逻辑：定义如何在 Elasticsearch 中执行搜索，包括关键词匹配、多条件组合查询等。
    * 分页：实现分页逻辑，包括普通的 from + size 分页和深度分页 search_after 的实现。
    * 搜索高亮：在搜索结果中实现关键词高亮显示，提高用户体验。
    * 分词器设置的调用：确保在查询中使用适当的分词器（比如 IK 分词器），以适应不同语言和搜索需求。
* 和同事讨论了 脚本设计（分片数3/副本数1/分析器）
* 同步/增量+kafka
## 与同事对接
* 索引名称和字段映射：
    * 索引的名称（如 articles）以及各字段的名称和类型。
    * 确认字段类型（如 text、keyword、date 等），确保您的查询中使用正确的字段和数据类型。

# 技术
## 文件设计架构
```java
src/main/java/com/example/search/
├── config/                           // 配置类目录
│   ├── ElasticsearchConfig.java     // ES 配置
│   └── KafkaConfig.java             // Kafka 配置（用于数据同步）
│
├── model/                           // 数据模型目录
│   ├── entity/                      // 数据库实体
│   │   └── Article.java            // MySQL 实体类
│   ├── document/                    // ES 文档模型
│   │   └── ArticleDocument.java    // ES 文档类
│   └── dto/                        // 数据传输对象
│       ├── SearchRequest.java      // 搜索请求参数
│       └── SearchResponse.java     // 搜索响应结果
│
├── repository/                      // 数据访问层
│   ├── ArticleRepository.java      // MySQL repository
│   └── ArticleEsRepository.java    // ES repository
│
├── service/                        // 业务逻辑层
│   ├── SearchService.java         // 搜索服务接口
│   ├── ArticleService.java        // 文章服务接口
│   ├── impl/
│   │   ├── SearchServiceImpl.java // 搜索服务实现
│   │   └── ArticleServiceImpl.java// 文章服务实现
│   └── sync/                      // 数据同步相关
│       ├── DataSyncService.java   // 数据同步服务
│       └── DataSyncListener.java  // 数据变更监听器
│
└── controller/                    // 控制器层
    └── SearchController.java     // 搜索接口控制器
```

## Code
```java
import org.elasticsearch.action.search.SearchRequest;
import org.elasticsearch.action.search.SearchResponse;
import org.elasticsearch.client.RequestOptions;
import org.elasticsearch.client.RestHighLevelClient;
import org.elasticsearch.index.query.BoolQueryBuilder;
import org.elasticsearch.index.query.QueryBuilders;
import org.elasticsearch.search.SearchHit;
import org.elasticsearch.search.builder.SearchSourceBuilder;
import org.elasticsearch.search.sort.SortOrder;
import org.elasticsearch.search.sort.FieldSortBuilder;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

public class ElasticsearchPagination {

    private final RestHighLevelClient client;

    public ElasticsearchPagination(RestHighLevelClient client) {
        this.client = client;
    }

    // 使用 from 和 size 实现前 10 页的普通分页查询
    public SearchResponse normalPaginationSearch(String indexName, String keyword, int pageNumber, int pageSize) throws IOException {
        SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
        
        // 构建查询条件
        BoolQueryBuilder query = QueryBuilders.boolQuery()
                .should(QueryBuilders.matchQuery("title", keyword))
                .should(QueryBuilders.matchQuery("content", keyword));
        searchSourceBuilder.query(query);
        
        // 设置分页
        searchSourceBuilder.from((pageNumber - 1) * pageSize);
        searchSourceBuilder.size(pageSize);
        
        // 设置排序，确保唯一性
        searchSourceBuilder.sort(new FieldSortBuilder("publish_date").order(SortOrder.DESC));
        searchSourceBuilder.sort(new FieldSortBuilder("_id").order(SortOrder.ASC));
        
        // 构建搜索请求
        SearchRequest searchRequest = new SearchRequest(indexName);
        searchRequest.source(searchSourceBuilder);
        
        // 执行查询
        return client.search(searchRequest, RequestOptions.DEFAULT);
    }

    // 基于 search_after 的分页查询
    public SearchResponse searchAfterPagination(String indexName, String keyword, int pageSize, Object[] lastSortValues) throws IOException {
        SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
        
        // 构建查询条件
        BoolQueryBuilder query = QueryBuilders.boolQuery()
                .should(QueryBuilders.matchQuery("title", keyword))
                .should(QueryBuilders.matchQuery("content", keyword));
        searchSourceBuilder.query(query);
        
        // 设置分页大小
        searchSourceBuilder.size(pageSize);
        
        // 设置排序
        searchSourceBuilder.sort(new FieldSortBuilder("publish_date").order(SortOrder.DESC));
        searchSourceBuilder.sort(new FieldSortBuilder("_id").order(SortOrder.ASC));
        
        // 添加 search_after 参数
        searchSourceBuilder.searchAfter(lastSortValues);
        
        // 构建搜索请求
        SearchRequest searchRequest = new SearchRequest(indexName);
        searchRequest.source(searchSourceBuilder);
        
        // 执行查询
        return client.search(searchRequest, RequestOptions.DEFAULT);
    }

    // 解析搜索结果并获取最后一个 sort 值
    public Object[] getLastSortValues(SearchResponse searchResponse) {
        SearchHit[] hits = searchResponse.getHits().getHits();
        if (hits.length > 0) {
            return hits[hits.length - 1].getSortValues();
        }
        return null;
    }

    // 执行分页逻辑
    public void executePagination(String indexName, String keyword, int pageSize) throws IOException {
        int currentPage = 1;
        Object[] lastSortValues = null;

        while (currentPage <= 30) {  // 超过30页则停止查询
            SearchResponse response;

            if (currentPage <= 10) {
                // 前10页使用正常分页方式
                response = normalPaginationSearch(indexName, keyword, currentPage, pageSize);
            } else {
                // 10-30页使用 search_after 分页
                response = searchAfterPagination(indexName, keyword, pageSize, lastSortValues);
            }

            // 处理查询结果
            processResults(response);

            // 获取当前页的最后一个 sort 值，用于下次分页
            lastSortValues = getLastSortValues(response);
            
            // 增加页码
            currentPage++;
        }
    }

    // 处理搜索结果的方法
    public void processResults(SearchResponse response) {
        List<String> results = new ArrayList<>();
        for (SearchHit hit : response.getHits().getHits()) {
            results.add(hit.getSourceAsString()); // 处理每个文档
        }
        System.out.println("Results: " + results);
    }
}

```