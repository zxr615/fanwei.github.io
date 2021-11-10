> 前段时间写需求写到搜索模块，伴着条件越来越多，各种条件组合的也越来越复杂，后期不好维护，所以改进了一下以前的搜索写法，业余时间写了一个小案例，和大家一起探讨探讨。



## 完整案例

[https://github.com/zxr615/go-escase](https://github.com/zxr615/go-escase)

- 基于 es7.14 开发，使用 [ik_smart ](https://github.com/medcl/elasticsearch-analysis-ik)分词

- 使用 [olivere/elastic](https://github.com/olivere/elastic) go-es扩展

- 使用到 `gin` 路由

- 批量生成 [测试数据](https://github.com/brianvoe/gofakeit)

- 批量导入数据到es [case](https://github.com/zxr615/go-escase/blob/master/cmd/article_reload/reload.go)

- v1改进前代码&v2 改进后代码

  

## 请求结构

SearchRequest 搜索请求结构

```go
// SearchRequest 搜索请求结构
type SearchRequest struct {
	Keyword    string `form:"keyword"`                                // 关键词
	CategoryId uint8  `form:"category_id"`                            // 分类
	Sort       uint8  `form:"sort" binding:"omitempty,oneof=1 2 3"`   // 排序 1=浏览量；2=收藏；3=点赞；
	IsSolve    uint8  `form:"is_solve" binding:"omitempty,oneof=1 2"` // 是否解决
	Page       int    `form:"page,default=1"`                         // 页数
	PageSize   int    `form:"page_size,default=10"`                   // 每页数量
}
```


## 改进前

Search() 文章搜索和 Recommend() 文章推荐的代码几乎一样，只是条件有所不同，重复代码太多，也不好维护。

例：Search() 处理搜索请求

```go
func (a ArticleV1) Search(c *gin.Context) {
	req := new(model.SearchRequest)
	if err := c.ShouldBind(req); err != nil {
		c.JSON(400, err.Error())
		return
	}

	// 构建搜索
	builder := es.Client.Search().Index(model.ArticleEsAlias)

	bq := elastic.NewBoolQuery()
	// 标题
	if req.Keyword != "" {
		builder.Query(bq.Must(elastic.NewMatchQuery("title", req.Keyword)))
	}

	// 分类
	if req.CategoryId != 0 {
		builder.Query(bq.Filter(elastic.NewTermQuery("category_id", req.CategoryId)))
	}

	// 是否解决
	if req.IsSolve != 0 {
		builder.Query(bq.Filter(elastic.NewTermQuery("is_solve", req.IsSolve)))
	}

	// 排序
	switch req.Sort {
	case SortBrowseDesc:
		builder.Sort("brows_num", false)
	case SortUpvoteDesc:
		builder.Sort("upvote_num", false)
	case SortCollectDesc:
		builder.Sort("collect_num", false)
	default:
		builder.Sort("created_at", false)
	}

	// 分页
	from := (req.Page - 1) * req.PageSize
	// 指定查询字段
	include := []string{"id", "category_id", "title", "brows_num", "collect_num", "upvote_num", "is_recommend", "is_solve", "created_at"}
	builder.
		FetchSourceContext(
			elastic.NewFetchSourceContext(true).Include(include...),
		).
		From(from).
		Size(req.PageSize)

	// 执行查询
	do, err := builder.Do(context.Background())
	if err != nil {
		c.JSON(500, err.Error())
		return
	}

	// 获取匹配到的数量
	total := do.TotalHits()

	// 序列化数据
	list := make([]model.SearchResponse, len(do.Hits.Hits))
	for i, raw := range do.Hits.Hits {
		tmpArticle := model.SearchResponse{}
		if err := json.Unmarshal(raw.Source, &tmpArticle); err != nil {
			log.Println(err)
		}

		list[i] = tmpArticle
	}

	c.JSON(http.StatusOK, gin.H{
		"code": 200,
		"data": gin.H{
			"total": total,
			"list":  list,
		},
	})
	return
}
```

请求测试一下

```shell
curl GET '127.0.0.1:8080/article/v1/search?keyword=茴香豆&page=1&page_size=2&sort=1'
```
<details>
  <summary> 点这里查看返回结果 </summary>

```json
{
    "code": 200,
    "data": {
        "list": [
            {
                "id": 8912,
                "category_id": 238,
                "title": "茴香豆上账；又好笑，",
                "brows_num": 253,
                "collect_num": 34,
                "upvote_num": 203,
                "is_recommend": 2,
                "is_solve": 1,
                "created_at": "2021-02-01 15:19:36"
            }
             ...
        ],
        "total": 157
    }
}
```
</details>

Recommend() 处理推荐请求

<details>
   <summary> func (a ArticleV1) Recommend(c *gin.Context) {} </summary>

  ```go
  // Recommend 文章推荐
func (a ArticleV1) Recommend(c *gin.Context) {
	// 构建搜索
	builder := es.Client.Search().Index(model.ArticleEsAlias)

	bq := elastic.NewBoolQuery()

	builder.Query(bq.Filter(
		// 推荐文章
		elastic.NewTermQuery("category_id", model.ArticleIsRecommendYes),
		// 已解决
		elastic.NewTermQuery("is_solve", model.ArticleIsSolveYes),
	))

	// 浏览量排序
	builder.Sort("brows_num", false)

	do, err := builder.From(0).Size(10).Do(context.Background())
	if err != nil {
		return
	}

	// 序列化数据
	...

  ```
</details>



## 改进后

先看结果

> 把所有查询的条件都拆分开来，像查询数据库一样查询 es，多方便呢，只需要组合需要的条件就可以得到想要的结果

```go
// Search 文章搜索
func (a ArticleV2) Search(c *gin.Context) {
	req := new(model.SearchRequest)
	if err := c.ShouldBind(req); err != nil {
		c.JSON(400, err.Error())
		return
	}

  // 像查数据库一样方便的添加条件即可查询
	list, total, err := service.NewArticle().
		WhereKeyword(req.Keyword).
		WhereCategoryId(req.CategoryId).
		WhereIsSolve(req.IsSolve).
		Sort(req.Sort).
		Paginate(req.Page, req.PageSize).
		DecodeSearch()

	if err != nil {
		c.JSON(400, err.Error())
		return
	}

	c.JSON(http.StatusOK, gin.H{
		"code": 200,
		"data": gin.H{
			"total": total,
			"list":  list,
		},
	})
	return
}

// Recommend 文章推荐
func (a ArticleV2) Recommend(c *gin.Context) {
  // 像查数据库一样方便的添加条件即可查询
	list, _, err := service.NewArticle().
		WhereCategoryId(model.ArticleIsRecommendYes).
		WhereIsSolve(model.ArticleIsSolveYes).
		OrderByDesc("brows_num").
		PageSize(5).
		DecodeRecommend()

	if err != nil {
		c.JSON(400, err.Error())
		return
	}

	c.JSON(http.StatusOK, gin.H{
		"code": 200,
		"data": gin.H{
			"total": len(list),
			"list":  list,
		},
	})
	return
}

```

## 怎么做到的呢

> 既然只是条件不同，那用组合的方式，把需要的条件组装起来再执行查询

internal/service/article.go

本案例中涉及到了 `es` 的 `must` `filter` `sort` 条件，所以我们要先构建一个 `struct` 保存要组合的条件

```go
type article struct {
	must   []elastic.Query
	filter []elastic.Query
	sort   []elastic.Sorter
	from   int
	size   int
}
```

构造函数

```go
func NewArticle() *article {
	return &article{
		must:   make([]elastic.Query, 0),
		filter: make([]elastic.Query, 0),
		sort:   make([]elastic.Sorter, 0),
		from:   0,
		size:   10,
	}
}
```

添加组合的条件的方法

> 刚开始用过 `AddKeyword()` `WithKeyword` 方法名，但都感觉不太好，后来想到 `Laravel` 中有快捷的查询条件的方法，如`whereId(1)` 会生成 sql ：`xxx where id = 1` 所以就仿照了 `Laravel ` 的方式命名了
>

```go
// WhereKeyword 关键词
func (a article) WhereKeyword(keyword string) article {
	if keyword != "" {
		a.must = append(a.must, elastic.NewMatchQuery("title", keyword))
	}
	return a
}

// WhereCategoryId 分类
func (a article) WhereCategoryId(categoryId uint8) article {
	if categoryId != 0 {
		a.filter = append(a.filter, elastic.NewTermQuery("category_id", categoryId))
	}
	return a
}

// WhereIsSolve 是否已解决
func (a article) WhereIsSolve(isSolve uint8) article {
	if isSolve != 0 {
		a.filter = append(a.filter, elastic.NewTermQuery("is_solve", isSolve))
	}
	return a
}

// Sort 排序
func (a article) Sort(sort uint8) article {
	switch sort {
	case SortBrowseDesc:
		return a.OrderByDesc("brows_num")
	case SortUpvoteDesc:
		return a.OrderByDesc("upvote_num")
	case SortCollectDesc:
		return a.OrderByDesc("collect_num")
	}
	return a
}

// OrderByDesc 通过字段倒序排序
func (a article) OrderByDesc(field string) article {
	a.sort = append(a.sort, elastic.SortInfo{Field: field, Ascending: false})
	return a
}

// OrderByAsc 通过字段正序排序
func (a article) OrderByAsc(field string) article {
	a.sort = append(a.sort, elastic.SortInfo{Field: field, Ascending: true})
	return a
}

// Paginate 分页
// page 当前页码
// pageSize 每页数量
func (a article) Paginate(page, pageSize int) article {
	a.from = (page - 1) * pageSize
	a.size = pageSize
	return a
}

```

到这里已经把需要的全部条件已经构建好了，现在条件有了，需要执行最后的搜索了

```go
// Searcher 执行查询
func (a article) Searcher(include ...interface{}) ([]json.RawMessage, int64, error) {
	builder := es.Client.Search().Index(model.ArticleEsAlias)

	// 查询的字段
	includeKeys := make([]string, 0)
	if len(include) > 0 {
		includeKeys = structer.Keys(include[0])
	}

	// 构建查询
	builder.Query(
		// 构建 bool query 条件
		elastic.NewBoolQuery().Must(a.must...).Filter(a.filter...),
	)

	// 执行查询
	do, err := builder.
		FetchSourceContext(elastic.NewFetchSourceContext(true).Include(includeKeys...)).
		From(a.from).
		Size(a.size).
		SortBy(a.sort...).
		Do(context.Background())

	if err != nil {
		return nil, 0, err
	}

	total := do.TotalHits()
	list := make([]json.RawMessage, len(do.Hits.Hits))
	for i, hit := range do.Hits.Hits {
		list[i] = hit.Source
	}

	return list, total, nil
}
```

到这里就可以使用上面构建好的组合条件的模式针对不同的接口条件组合不同的条件获取结果

```go
list, _, err := service.NewArticle().
		WhereCategoryId(model.ArticleIsRecommendYes).
		WhereIsSolve(model.ArticleIsSolveYes).
		OrderByDesc("brows_num").
		PageSize(5).
		Searcher()
```

但这个时候还有个问题，这里的 `list` 返回的是一个 `[]json.RawMessage` 的数组，每一条纪录都是一条`json`字符串，有时候我们需要对查询出来的数据进行进一步的处理，这个时候就需要把内容序列化成对应的 `struct` 了，我们可以再写一个`DecodeXxxx()` 的方法，来针对不同的用途序列化成对应的 `struct`



> 推荐结果的结构体，这样返回一个 `[]model.RecommendResponse` 可用的结构体，调用时就不直接调用 `Searcher()` 调用 `DecodeRecommend()` 就好了。

```go
func (a article) DecodeRecommend() ([]model.RecommendResponse, int64, error) {
	rawList, total, err := a.Searcher(new(model.RecommendResponse))
	if err != nil {
		return nil, total, err
	}

	list := make([]model.RecommendResponse, len(rawList))
	for i, raw := range rawList {
		tmp := model.RecommendResponse{}
		if err := json.Unmarshal(raw, &tmp); err != nil {
			log.Println(err)
			continue
		}

		list[i] = tmp
	}

	return list, total, nil
}
```



## 总结

到此已经完成了搜索的简单封装，完整的案例可以去 [https://github.com/zxr615/go-escase](https://github.com/zxr615/go-escase) 交流交流
