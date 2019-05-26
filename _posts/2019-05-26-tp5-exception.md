---
layout: post
title: TP5自定义全局异常处理
categories: ThinkPHP
description: 指定错误处理方式，减少繁琐或不必要操作
keywords: ThinkPHP, php
---


# TP5自定义全局异常处理

### 0、起因之前

本篇内容为参考视频课程[^1]，学习的内容总结。

[^1]: ThinkPHP5.0+小程序商城构建全栈应用 

### 1、起因

1、在写接口的时候，写了一个 `BaseController` 用于被继承，主要做权限控制，中间项处理等等。但是被继承后使用 `return` 返回错误记录时，只返回该方法处理接果，并不会直接抛出结果。若直接使用异常抛出，调试阶段还能返回错误展示，自己看看还好，对于人家请求你的接口你返回一堆的错误信息，会表示很懵逼o((⊙﹏⊙))o。如果调试模式关掉了，就会直接返回一个页面错误的提示。反正调用者就是拿不到想要的错误信息。。。  

2、对于每次创建、编辑、删除、更新时，都会有做参数校验的操作，好在，写了公共校验方法，但是但是每次都要处理返回结果(如下)，那~~么多的接口，每次都要重复一边，心态正在逐渐炸裂(•́へ•́╬)  

```php
// 添加内容验证
$result = $this->validateParams('add');
if ($result['error_code'] == 1) {
    return $result;
}
```

### 2、解决方案

关于上面的问题，解决方案如下

```php
$result = json_encode(['error' => 500, 'msg' => '信息错误']);
exit($result);	
// or
print_r($result);die;
```

你以为这就完事了吗，不存在的。  

虽然毛病倒是没毛病，但是不觉得很lowxxx吗(emmm~~在此之前我也是。。)，而且诶， `tp5` 自带的 `json()` 函数可以传入状态码，若是做标准的接口，状态码是必不可缺的。

### 3、替代方案

我们只要传入参数抛出异常就ok了，也不需要捕获，让他们自己处理去吧

```php
/*
 *@param string $msg 错误信息
 *@param int $error_code 错误码
 *@param int $code 状态码
 **/
throw new InvalidParams($msg, $errro_code, $code);
```

比前面的简单很多了不，而且也直观呀

### 4、动手

#### 动手前

关于系统内处理错误的产生，概括为两种情况，一种是外部原因影响的，一种是内部原因所影响的。  

**外部原因：**由于输入内容错误导致，需要返回错误信息。比如：用户输入了10位手机号码，但是手机号码必须11位，传入后台时，我们需要丢出错误，并提醒给用户  

**内部原因：**由于内在处理错误，无需返回错误信息。比如：代码写错了，导致流程执行失败，此时日志记录错误信息，无需返回给用户

#### 第一步：覆写 `TP` 框架自带异常处理机制

在 `application` 文件夹下创建一个 `exception` 文件夹(你爱取啥名取啥名，最好是让自己规范的来) ，也可以创建在模块下面。   

创建 `ExceptionHandle` 类文件，并继承 `Handler` 类，覆写 `render` 方法 ，最后代码如下

```php
<?php

namespace app\exception;

use think\exception\Handle;

class ExceptionHandle extends Handle
{
    public function render(\Exception $e)
    {
        
    }
}
```

#### 第二步：修改配置

打开 `config.php` 文件

开启调试模式，将 `app_debug` 的值更改为 `true` 

修改异常处理handle类，将 `exception_handle` 的值更改为你上一步所写 `Exception` 类的命名空间 `app\exception\ExceptionHandle`

```php
// 应用调试模式
'app_debug'        => true,
// 异常处理handle类 留空使用 \think\exception\Handle
'exception_handle' => 'app\exception\ExceptionHandle',
```

> **注意：**
>
> 1.有人可能会问，调试模式因为覆写了 ，开不开有什么用？是没什么用，不过优化的时候我们会用到啦 
>
> 2.测试是否已完成覆写，在 `render` 方法中随意输出点什么，再在控制器中丢出一个异常，如果能成功返回你写的内容，那么，继续下一步。NO ELSE,   NO WHY~

#### 第三步：重构

我们只是部分需要直观的抛出错误信息，而不影响正确的异常抛出，因此我们需要在 `app\exception\` 文件夹下建立一个 `BaseException` 类

```php
<?php

namespace app\exception;

use think\Exception;

class BaseException extends Exception
{

}
```

异常为 `BaseException` 类型时才做我们自己的处理，修改 `app\exception\ExceptionHandle`

```php
...
    public function render(\Exception $e)
    {
        // 指定处理
        if ($e instanceof BaseException) {

        }
        // 正常处理
        return parent::render($e);
    }
...
```

嗯嗯~ 基本结构写完了，接下来我们就要传入参数啦，方便在 `render` 方法中使用并丢出结果  

给 `app\exception\BaseException` 重新写入参数

```php
...
class BaseException extends Exception
{
    public $error_msg = '未知错误';
    public $error_code = '999';
    public $code = 500;

    public function __construct($message = "", $error_code = 0, $code = 0, Throwable $previous = null)
    {
        $this->error_msg  = $message;
        $this->error_code = $error_code;
        $this->code = $code;
    }
}
```

再在 `app\exception\ExceptionHandle` 中拿结果并处理

```php
...
    public function render(\Exception $e)
    {
        // 指定处理
        if ($e instanceof BaseException) {
            $result = [
                'error_code' => $e->error_code,
                'error_msg'  => $e->error_msg,
            ];
            return json($result, $e->code);
        }
        // 正常处理
        return parent::render($e);
    }
...
```

至此，常规的基本操作已经完成啦，先来小时试一波

```php
throw new BaseException('ahahah~~', 999, 500);
// 输出
{
  "error_code": 999,
  "error_msg": "ahahah~~"
}
```

好的吧~结果还不错，达到了预期值​

当然咯，这个只是外部错误的处理，内部错误的处理也要写上

```php
...
	public function render(\Exception $e)
    {
        ...
        // 如果未开启调试模式 并且 为异步请求的时候 未指定处理，则对外统一错误报告
        if (!config('app_debug') && request()->isAjax()) {
            // 记录错误日志，方便查看
            Log::record($e, 'error');
            $result = [
                'error_code' => 999,
                'error_msg'  => '未知错误'
            ];
            return json($result, 500);
        }
        // 正常处理
        return parent::render($e);
    }
...
```

#### 第四步：测试

**自己动手，丰衣足食。**

### 5、结尾

以上内容若存在错误，或有更优方案，欢迎在下方评论。

关于状态码的定义，自行百度啦，略~~