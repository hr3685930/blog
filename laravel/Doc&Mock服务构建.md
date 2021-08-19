# 基于RAML的接口描述规范 及 Doc&Mock服务构建
- 示例
```
1. 主配置文件（./api.yaml）
#%RAML 1.0
title: API Document
baseUri: https://api.example.com
protocols: [ HTTPS ]
mediaType: application/json

pageable: !include ./traits/pageable.raml

types:
  PostBookJson:
    type: object
    properties:
      name:
        type: string
        description: 名字
        required: true
  BookJson:
    type: PostBookJson
    properties:
      id:
        type: integer
        description: 主键
        required: true

/books:
  displayName: book列表资源
  get:
    description: 获取book资源列表
    is: pageable
    queryParameters:
      name: string
    responses:
      200:
        body:
          application/json:
            type: BookJson[]
  post:
    description: 创建book资源
    body:
      application/json:
        type: PostBookJson
    responses:
      201:
        body:
          application/json:
            type: BookJson
      422:
        body:
          application/json:
            example:
              error: validation_failed
              error_description: name is required
  /{id}:
    displayName: book详情资源
    get:
      description: 获取book详情
      queryParameters:
        id: integer
        name: string
      responses:
        200:
          body:
            application/json:
              type: BookJson
    put:
      description: 修改book资源
      body:
        application/json:
          type: PostBookJson
      responses:
        200:
          body:
            application/json:
              type: BookJson
        404:
          body:
            application/json:
              example:
                error: resource not found
                error_description: The book you requested was not found
        422:
          body:
            application/json:
              example:
                error: valication failed
                error_description:  name must be filled
    delete:
      description: 删除book资源
      responses:
        204:
          body: !!null
        404:
          body:
            application/json:
              example:
                error: resource not found
                error_description: The book you requested was not found
1. 引用文件（./traits/pageable.raml）
description: common page trait, resource can be paged defaultly
queryParameters:
  page:
    description: Specify the page that you want to retrieve
    type: number
    required: false
    default: 1
    maximum: 1000
    example: 1
  per_page:
    description: Specify the amount of items that will be retrieved per page
    type: number
    required: false
    default: 20
    maximum: 100
    example: 20
  sort:
    description: Specify the sort order of the items list, + mean asc, - mean desc
    type: string
    required: false
    default: -id
    example: +name,-id

responses:
  200:
    headers:
      Link:
        description: The paginator link according to https://tools.ietf.org/html/rfc5988
        type: string
        example: |
          '<https://api.example.com/api/users?page=1&per_page=20>; rel="first", <https://api.example.com/api/users?page=2&per_page=20>; rel="prev", <https://api.example.com/api/users?page=4&per_page=20>; rel="next", <https://api.example.com/api/users?page=50&per_page=20>; rel="last"'
      X-Total-Count:
        description: 分页列表数据总计
        type: number
        example: 1000
```

- 基于RAML构建文档服务 
``` 
sudo cnpm install -g raml2html  

raml2html -i api.raml -o index.html 【-t /templates/自定义模板.nunjucks】  
# 或者
raml2html 【-t /templates/自定义模板.nunjucks】api.raml > index.html
基于RAML构建Mock服务
sudo cnpm install -g localapi

pkill -f /usr/local/bin/localapi
localapi run api.raml -p 8000 --no-examples <&- >&- 2>&- & disown
```