---
layout: post
title: 快速使用GraphQL API开发接口
categories: GraphQL
description: GraphQL一种用于 API 的查询语言
keywords: GraphQL, php, api
---

# 快速使用GraphQL API开发接口

## 关于GraphQL

GraphQL一种用于 API 的查询语言，详情请查阅GraphQL官网 [中文](https://graphql.cn/)/[English](https://graphql.org/) 。

## 准备工作

框架：`thinkphp5.1`，以下简称 `tp`

插件：[`smilecc/think-graphql`](https://github.com/smilecc/think-graphql)，目前所有 `graphql` 的插件都是基于 [`webonyx/graphql-php`](https://github.com/webonyx/graphql-php) 开发的

浏览器扩展：[`Altair GraphQL Client`](https://chrome.google.com/webstore/detail/altair-graphql-client/flnheeellpciglgpaodhkhmapeljopja)，用于快速模拟请求使用，以下简称 `Altair`

> **注：**若是无法进入谷歌应用，请直接在 [Altair GraphQL](https://altair.sirmuel.design/#download) 官网下载客户端使用。

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
query{
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

否则，你可要好好仔细检查前面的步骤啦~~

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

中间件，在执行前，我们可以做一些数据拦截什么的操作，在 `route/graph.php` 中已经生成了一些默认配置

### schema

入口类型，即：`http://127.0.0.1:8000/api/graph` 中的 `graph` ，需要与 `types` 中的键名一致， `'blog' => 'graph'` 中的 `blog` 为 `graph` 的别名，也可以使用 `http://127.0.0.1:8000/api/blog` 访问，与前者是一样的。

### types

类型注册表，我们所有编写的类型均需要在此进行注册，每个接口中的类型仅分为 `query` 查询和 `mutation` 编辑处理两种。若未指定类型，则默认为 `query`

> 两种操作类型主要是为了区分查询和处理，其操作编写方法一致

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

删除 `application/http`

清除 `config/graph.php` 中的 `types`

开始我们的第一个编写吧~

在 `application/graphql/` 下创建一个 `Query.php` 的类，并继承 `ObjectType` [对象类型](https://smilecc.github.io/think-graphql/type/object/)

```php
<?php

namespace app\graphql;

use smilecc\think\Support\ObjectType;

class Query extends ObjectType
{

}
```

> 有关该类型中的方法以及相关字段描述可[点击查阅](https://smilecc.github.io/think-graphql/type/object/)

类已经创建完了我们首先对类型进行描述，并命名，主要用于接口文档的查阅

```php
...
	public function attrs()
    {
        return [
            'name' => 'QueryType',
            'desc' => '我写的第一次查询类型'
        ];
    }
...
```

然后呢，就需要定义我们要查询的字段 `hello`

> **字段类型主要分为：`自定义类型` 、`Types::string` 字符串、`Types::int` 整型、`Types::boolean` 布尔、`Types::float` 浮点、`Types::id` 唯一**

```php
...
    public function fields()
    {
        return [
            'hello' => [
                'type' => Types::string(),
                'desc' => '第一个方法哦~'
            ],
            // 仅指定字段类型可以按照下列写法，类型指定为必须，不可忽略
            // 'hello' => Types::string()
        ];
    }
...
```

字段好了，我们还需要给定结果值即返回值，结果值可以为我们处理的结果或者一个默认值，所有字段的处理方法名都需要以 `resolve` 开头，后面接上字段的驼峰命名法 `resolveHello` ， 若是字段以下划线链接那就这样子这样子然后再这样子:smile: `create_time => resolveCreateTime`

```php
...
    /**
     * @param $val 表示输入的值，在自定类型时使用
     * @param $args 参数，若存在参数传递，则可通过该值获取
     * @return string
     */
	public function resolveHello($val, $args)
    {
        return 'hello world';
    }
...
```

好啦，打开 `Altair` ，输入我们访问的网址 `http://127.0.0.1/api/graph`，输入查询内容

```graphql
query{
  hello
}
```

若是得到以下结果，恭喜你，已经get了 `graphql` 的基本技能啦~

```json
{
  "data": {
    "hello": "hello world"
  }
}
```

![]({{ site.url }}/assets/images/posts/201906/13-001-1002.png)

## 小试牛刀

基本操作已经会了，下面我们来加大一点点的难度

通过 `mutation` 类型修改用户姓名及年龄，再通过 `query` 类型获取用户的最新信息

首先定义一个用户信息类型类 `application/graphql/user/User.php`

```php
<?php

namespace app\graphql\user;

use smilecc\think\Support\ObjectType;
use smilecc\think\Support\Types;

class User extends ObjectType
{
    public function attrs()
    {
        return [
            'name' => 'UserTypes',
            'desc' => '用户信息'
        ];
    }

    public function fields()
    {
        return [
            'name' => [
                'type' => Types::string(),
                'desc' => '用户姓名'
            ],
            'age'  => [
                'type' => Types::int(),
                'desc' => '用户年龄'
            ],
            'edit_time' => [
                'type' => Types::string(),
                'desc' => '编辑时间',
            ]
        ];
    }
}
```

时间存储的时候用的整型，取出的时候我们需要转换下，不处理则返回默认值

```php
...
	public function resolveEditTime($val, $args)
    {
        return date('Y-m-d H:i:s', $val['edit_time']);
    }
...
```

类型类定义完后，需要在 `config/graph.php` 中进行注册

```php
<?php

return [
    // 类型注册表
    'types' => [
        'user' => \app\graphql\user\User::class
    ],
    ...
]
```

接下来我们建一个处理类 `application/graphql/Mutation.php`

```php
<?php

namespace app\graphql;

use smilecc\think\Support\ObjectType;
use smilecc\think\Support\Types;
use think\facade\Cache;

class Mutation extends ObjectType
{
    public function attrs()
    {
        return [
            'name' => 'MutationType',
            'desc' => '第一个处理类型'
        ];
    }

    public function fields()
    {
        return [
            'user' => [
                'type' => Types::string(),
                'desc' => '需要更新的内容',
                'args' => [ // 需要传入的参数
                    'name' => Types::string(),
                    'age'  => Types::nonNull(Types::int()) // Types::nonNull 表示该字段为非空必填
                ]
            ]
        ];
    }
}
```

我们把传入的参数存储起来，以便读取

```php
...
	public function resolveUser($val, $args)
    {
        Cache::set('user', [
            'name' => $args['name'],
            'age'  => $args['age'],
            'edit_time' => time()
        ]);
        return '处理成功';
    }
...
```

处理类完了后，我们就需要读取啦，定义一个读取类型类 `application/graphql/Query.php`

```php
<?php

namespace app\graphql;

use smilecc\think\Support\ObjectType;
use smilecc\think\Support\Types;
use think\facade\Cache;

class Query extends ObjectType
{
    public function attrs()
    {
        return [
            'name' => 'Query',
            'desc' => '我写的第一次查询类型'
        ];
    }

    public function fields()
    {
        return [
            'user' => [
                'type' => Types::user(), // 使用自定义类型
                'desc' => '获取用户信息'
            ]
        ];
    }
}
```

> **注意：**在这里我们user使用的自定义类型，`Types::user()` 已经在前面的注册表注册过了哟~ 若是 `Types::user()` 未注明 `mutation` 类型（如：`Types::user('mutation')`），那么将默认使用 `query` 类型

同样的套路，接下来处理读取信息

```php
...
	public function resolveUser($val, $args)
    {
        $info = Cache::get('user');
        if (empty($info)) {
            return '信息未获取到';
        }
        $info['name'] = empty($info['name']) ? '未知' : $info['name'];
        return $info;
    }
...
```

> **注意：**返回的字段需要与自定义类型中字段一致。

然后我们把 `mutation` 和 `query` 注册

```php
<?php

return [
    // 类型注册表
    'types' => [
        'user' => \app\graphql\user\User::class,
        'graph' => [
            'query' => \app\graphql\Query::class,
            'mutation' => \app\graphql\Mutation::class,
        ]
    ],
]
...
```

一切准备完毕，打开 `Altair` 开始请求，先进行处理操作

```
输入网址
http://127.0.0.1:8000/api/graph

请求内容
mutation{
  user(name: "smile的微笑", age: 18)
}

成功返回
{
  "data": {
    "user": "处理成功"
  }
}
```

再接着进行查询操作

```
输入网址
http://127.0.0.1:8000/api/graph

请求内容
query{
  user{
    name
    age
    edit_time
  }
}

成功返回
{
  "data": {
    "user": {
      "name": "smile的微笑",
      "age": 18,
      "edit_time": "2019-06-13 14:07:18"
    }
  }
}
```

若是能得到相应结果，那就说明处理成功了，恭喜啦~

> **注意：**请求内容中存在子字段时，必须写入至少一个查询字段

## 完结

一定要多写多试多练多探索~