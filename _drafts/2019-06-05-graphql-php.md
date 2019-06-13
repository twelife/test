# 快速使用GraphQL API开发接口

## 关于GraphQL

GraphQL一种用于 API 的查询语言，详情请查阅 [GraphQL中文官网](https://graphql.cn/) 。

## 准备工作

框架：`thinkphp5.1`，以下简称 `tp`

插件：[`smilecc/think-graphql`](https://github.com/smilecc/think-graphql)，目前所有 `graphql` 的插件都是基于 [`webonyx/graphql-php`](https://github.com/webonyx/graphql-php) 开发的

浏览器扩展：[`Altair GraphQL Client`](https://chrome.google.com/webstore/detail/altair-graphql-client/flnheeellpciglgpaodhkhmapeljopja)，用于快速请求使用，以下简称 `Altair`

数据表结构

```mysql
-- MySQL
-- 文章表
CREATE TABLE `posts` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `author` varchar(50) DEFAULT NULL COMMENT '作者',
  `title` varchar(50) DEFAULT NULL COMMENT '文章标题',
  `desc` varchar(255) DEFAULT NULL COMMENT '文章简介',
  `content` text COMMENT '文章内容',
  `create_time` int(11) DEFAULT NULL COMMENT '创建时间',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='文章表';

-- 评论表
CREATE TABLE `comment` (
  `id` int(11) NOT NULL,
  `posts_id` int(11) DEFAULT NULL COMMENT '文章id',
  `content` varchar(255) DEFAULT NULL COMMENT '评论内容',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='评论表';
```

分别创建 `Posts` 和 `Comment` 模型，并在 `Posts` 中创建一个评论的关联模型

```php
public function comments()
{
	$this->hasMany('Comment', 'posts_id', 'id');
}
```

## 安装配置插件 

在项目中使用 `composer` 命令安装 `smilecc/think-graphql` 插件

```composer
composer require smilecc/think-graphql:dev-master
```

安装完后，在 `application/command.php` 中添加一个指令

```php
<?php

return [
    'smilecc\think\GraphQLCommand'
];
```

再在根目录中使用命令进行初始化操作

```shell
php think graph init
```

初始化成功后，在 `config/` 下会生成一个 `graph.php` 的配置文件，以及在`application/http/graph`文件夹下生成出实例项目。

接下来，我们启动 `tp` 内置服务器

```shell
php think run
```

在浏览器中打开 `Altair` 扩展，输入请求地址 `http://127.0.0.1:8000/api/graph`，并写入请求内容

```graphql
{
	user(id: 1){
		id
		nickname
		created_time
    }
}
```

若能获得以下结果，那么，恭喜你，插件已经安装配置完成 :blush:

```raphql
{
  "data": {
    "user": {
      "id": "1",
      "nickname": "TestUser",
      "created_time": "1533028910"
    }
  }
}
```

否则，你可要好好仔细检查下前面的步骤啦~~

![]({{ site.url }}/assets/images/posts/201906/13-001-1001.png)

## GraphQL的配置文件

关于 `graphql` 的配置文件还是非常友好的，有注释 :smile:

初始内容如下

```php
<?php

return [
    // 类型注册表
    'types' => [
        'graph' => [
            'query' => \app\http\graph\QueryType::class
        ],
        'user' => [
            'query' => \app\http\graph\User\UserType::class
        ]
    ],
    // 入口类型
    'schema' => [
        'graph',
        'blog' => 'graph'
    ],
    // 中间件
    'middleware' => [],
    // 路由前缀
    'routePrefix' => 'api/'
];
```

### routePrefix

路由前缀，即：`http://127.0.0.1:8000/api/graph` 中的 `api/`

### middleware

中间件，在执行前，我们可以做一些数据拦截什么的操作

### schema

入口类型，即：`http://127.0.0.1:8000/api/graph` 中的 `graph` ，其 ` blog` 为 `graph` 的别名，也可以使用`http://127.0.0.1:8000/api/blog` 访问，与前者是一样的，需要与 `types` 中的键名一致

### types

类型注册表，我们所有编写的类型均需要在此进行注册，每个接口中的类型仅分为 `query` 查询和 `mutation` 编辑处理两种。若未指定类型，则默认为 `query`

```php
return [
	'types' => [
        'graph' => [
            'query' => \app\http\graph\QueryType::class,
            'mutation' => \app\http\graph\QueryType::class
        ],
        'user' => [
            'query' => \app\http\graph\User\UserType::class
        ],
        'hi' => \app\http\graph\User\HiType::class
    ]
];
```

## Hello World

