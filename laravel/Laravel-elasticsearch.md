# Laravel elasticsearch
elasticsearch官方包
所有参数（uri、document body）传递均使用索引数组

- 安装
```
composer require elasticsearch/elasticsearch:~2.0
```

- 使用
```
$client = Elasticsearch\ClientBuilder::create()->build();
```

## 创建索引
- 请求：
```
$response = $client->indices()->create([
    'index' => 'my_index',
    'body' => [
        'settings' => [
            'number_of_shards' => 2,
            'number_of_replicas' => 0
        ]
    ]
]);

#响应：

[
	'acknowledged' => 1
]
```

## 索引文档 
```
#请求：
$response = $client->index([
    'index' => 'my_index',
    'type' => 'my_type',
    'id' => 'my_id',
    'body' => ['welcome' => 'hello world']
]);
#响应：
[
	'_index' => 'my_index',
	'_type' => 'my_type',
	'_id' => 'my_id',
	'_version' => '1',
	'created' => 1,
]
```

## 检索文档 
```
#请求：
$response = $client->get([
    'index' => 'my_index',
    'type' => 'my_type',
    'id' => 'my_id'
]);
#响应：
[
	'_index' => 'my_index',
	'_type' => 'my_type',
	'_id' => 'my_id',
	'_version' => '1',
	'found' => 1,
	'_source' => [
		'welcome' => 'hello world'
	],
]
# 可调用方法getSource直接获取_source
```

## 查询文档
```
#请求：
$response = $client->search([
    'index' => 'my_index',
    'type' => 'my_type',
    'body' => [
        'query' => [
            'match' => [
                'welcome' => 'hello'
            ]
        ]
    ]
]);
#响应：
[
    'took' => 1,
    'timed_out' => '',
    '_shards' => [
        'total' => 5,
        'successful' => 5,
        'failed' => 0,
    ],

    'hits' => [
        'total' => 1,
        'max_score' => 0.30685282,
        'hits' => [
            [
                '_index' => my_index,
                '_type' => my_type,
                '_id' => my_id,
                '_score' => 0.30685282,
                '_source' => [
                    'welcome' => 'hello world',
                ],
            ],
        ],
    ]
]
```

## 删除文档
```
#请求：
$response = $client->delete([
    'index' => 'my_index',
    'type' => 'my_type',
    'id' => 'my_id'
]);
#响应：
[
    'found' => 1,
    '_index' => my_index,
    '_type' => my_type,
    '_id' => my_id,
    '_version' => 2,
]
```

## 删除索引
```
#请求：
$response = $client->indices()->delete([
    'index' => 'my_index'
]);
#响应：
[
	'acknowledged' => 1
]
```

Scout包
> 针对 Eloquent ORM 开发的基于驱动的简易全文检索系统  

- 安装
```
############################# Scout ###############################
composer require laravel/scout

# 注册providor至config/app.php
Laravel\Scout\ScoutServiceProvider::class,

# 发布配置
php artisan vendor:publish --provider="Laravel\Scout\ScoutServiceProvider"

# 队列支持（可选，提高应用响应速度）
1. 配置队列
2. 启用队列：config/scout.php 中设置 'queue' => true,

######################## ElasticSearch驱动 ##########################
composer require tamayo/laravel-scout-elastic

# 注册provider至config/app.php
ScoutEngines\Elasticsearch\ElasticsearchProvider::class,

# 配置Scout驱动
'elasticsearch' => [
    'index' => env('ELASTICSEARCH_INDEX', 'laravel'), #索引
    'hosts' => [ #集群节点
        env('ELASTICSEARCH_HOST', 'http://localhost'),
    ],
],
```

- 模型配置
```
    在需要搜索支持的Model中 use Laravel\Scout\Searchable;（注册模型观察者自动保持模型和索引的同步）
    索引名称默认与表名一致（snake模型名s），亦可通过searchableAs方法自定义
    默认索引数据$model->toArray()，亦可toSearchableArray方法自定义

索引全表 -

php artisan scout:import 模型名 #已有数据批量导入索引
php artisan scout:flush  模型名 #批量更新索引

索引结果集 - 默认的，模型在更新时自动toArray()同步到索引，亦可searchable方法手动索引查询结果

# 自动对结果分块并进行索引
$collection->->searchable();

索引删除 - 默认的，模型在删除时自动删除索引，亦可unsearchable方法手动删除查询结果的索引

# 自动对结果分块并删除索引
$collection->->unsearchable();

索引暂停 -

App\Order::withoutSyncingToSearch(function(){
    //模型所有更新操作将关停索引同步
});

索引查询 -

# where仅支持数字相等查询，用于处理范围查询
模型::search($keyword)->where()->get()
模型::search($keyword)->where()->paginate([$num])
```

- 思考
Scout搭建简单快捷，在数据和索引的自动同步上提供了极大的便利性；
但是Scout的搜索功能过于简单，无法彻底发挥ES的强大功能；
Elasticquent包映射Eloquent Model至Elasticsearch Type
由于Scout包诸多欠缺性， 要充分发挥ES的强大功能， 我们可以结合使用Elasticquent包做搜索增强，具体的：
    Elasticquent包负责索引查询
        配置Mapping
        进行DSL复合查询
    Scout包负责索引更新、删除
        数据和索引的自动同步
- 安装
```
composer require elasticquent/elasticquent:dev-master

# 注册providor、facad至config/app.php
Elasticquent\ElasticquentServiceProvider::class,
'Es' => Elasticquent\ElasticquentElasticsearchFacade::class,

# 发布配置
php artisan vendor:publish --provider="Elasticquent\ElasticquentServiceProvider"

模型配置

use Elasticquent\ElasticquentTrait;

# 自定义分析器
protected $indexSettings = [
    'analysis' => [
        'filter' => [
            'synonym' => [
                'type' => 'synonym', # ES内置的同义词过滤器
                'synonyms' => ['you,你', 'I,me=>我'],
                #'synonyms_path' => 'synonym.txt', 可选同义词文件（必须存在集群的每个节点上）
            ],
            'keyword_filter' => [
                'type' => 'ngram', # ES内置的汉语模型过滤器（用于生成部分匹配或者自动补全的词条）
                'min_gram' => 1,
                'max_gram' => 20,
            ],
        ],
        'analyzer' => [
            'default' => [
                'type' => 'custom', #或者不用自定义analyser，直接使用ik_max_word
                'char_filter' => ['html_strip'],
                'tokenizer' => 'ik_max_word',
                'filter' => [
                    'lowercase',
                    'synonym',
                    'keyword_filter',
                ],
            ],
        ],
    ],
];
```

- 自定义Mapping
```
protected $mappingProperties = array(
    'id' => ['type' => 'integer'],
    'title' => ['type' => 'string'],
    'tag' => ['type' => 'string', 'index' => 'not_analyzed'],
    'location' => ['type' => 'geo_point'],
    'childs' => [
        'type' => 'nested',
        'properties' => [
            'id' => ['type' => 'integer'],
        ],
    ],
);

# 自定义索引名（覆盖配置文件中默认索引名）
function getIndexName()
{
    return 索引名;
}

# 自定义类型名（覆盖默认的 snake模型名s）
function getTypeName()
{
    return 类型名;
}

# 自定义索引结构
# 和$mappingProperties属性搭配使用
function getIndexDocumentData()
{
    return array(
        'id'      => $this->id,
        'name'   => 'xxx',
        ...
    );
}
```

- 索引管理
```
    索引初始化

# 创建索引
模型::createIndex($shards = null, $replicas = null);

# 根据$mappingProperties属性设置mapping
重设类型：模型::putMapping($ignoreConflicts = true);
删  除：模型::deleteMapping();
重  建：模型::rebuildMapping();
存在性：模型::mappingExists();
读  取：模型::getMapping();

# 检查Type
模型::typeExists();

    索引全表

模型::addAllToIndex(); #不会自动分块表数据
模型::reindex(); #重建索引，直接使用ES中_source字段的原始文档数据来重建

    索引结果集

# 模型id即为索引id
$collection->->addToIndex();
$model->addToIndex();

    索引查询

# term查询精确值
$results = 模型::search($keyword);

# 基本搜索
$results = 模型::searchByQuery(
								array $query = null,
								$aggregations = null,
								$sourceFields = null,
								$limit = null,
								$offset = null,
								$sort = null
		)

```

- 全文搜索（DSL原生语法）
```
$results = 模型::complexSearch(array $query)->paginate($pageNum)

# 搜索结果
$results->totalHits(); #匹配文档总量
$results->getAggregations(); #匹配文档聚合信息
$results->shards();
$results->maxScore();
$results->timedOut();
$results->took();


# 搜索结果集的单元
$model->isDocument();
$model->documentScore();

同ES官方包协作

# 原生查询结果转为Eloquent集合
$client = new \Elasticsearch\Client();
$collection = 模型::hydrateElasticsearchResult($client->search($params));
```