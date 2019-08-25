---
layout: post
title: Editor上传文件至阿里云OSS
categories: ThinkPHP
description: 在thinkphp中使用ueditor上传文件时，存储至阿里云oss
keywords: 阿里云,OSS,UEditor
---

# UEditor上传文件至阿里云OSS

## 1、使用环境

`ThinkPHP 5.1` + `UEditor`

## 2、准备工作

### 2.1 创建项目 ThinkPHP 5.1

使用 `composer` 安装，其它安装方法：<https://www.kancloud.cn/manual/thinkphp5_1/353948>

> composer create-project topthink/think=5.1.* tp5

### 2.2 安装阿里云OSS扩展

其它安装详细方法：<https://help.aliyun.com/document_detail/85580.html?spm=a2c4g.11186623.6.1049.2e8d33bcRmnO4o>


> composer require aliyuncs/oss-sdk-php

### 2.3 准备阿里云相关信息

创建阿里云账号

在控制台创建OSS仓库

创建秘钥

### 2.4 下载UEditor

下载地址：<https://ueditor.baidu.com/website/download.html>

下载 `utf8版本` 后将文件解压至：`your/app/path/public/static/ueditor` 

## 3、修改上传配置文件

### 3.1 增加项目配置文件

增加项目中的配置文件，主要是因为项目上传文件的不只有 `ueditor`，还有其它地方也需要用到，方便统一管理。

```shell
# 移动至项目配置路径
cd your/app/path/config
# 新建一个文件
touch diy.php
# 编辑
vim diy.php
```

写入以下内容并保存

```php
<?php

return [
    // 上传类型
    'upload' => [
        'type' => 'oss', // local, oss
    ],
    // 阿里云oss配置
    'oss' => [
        'accessKeyId'     => 'accessKeyId',
        'accessKeySecret' => 'accessKeySecret',
        'endPoint'        => 'oss-cn-shenzhen.aliyuncs.com',
        'bucket'          => 'go-file',
        'OssImageUrl'     => 'go-file.oss-cn-shenzhen.aliyuncs.com'
    ],
];
```

### 3.2 修改配置项

```shell
cd your/app/path/public/static/ueditor/php

vim config.json
```

主要修改各项中 `访问路径前缀` 以及 `上传保存路径`

```json
 {
    ...
   
 		"imageUrlPrefix": "http://go-file.oss-cn-shenzhen.aliyuncs.com/", /* 图片访问路径前缀，改地址可以在阿里云oss仓库获取 */
 		"imagePathFormat": "/uploads/image/{yyyy}{mm}{dd}/{time}{rand:6}", /* 上传保存路径,可以自定义保存路径和文件名格式 */
   
   ...
 }
```

### 3.3 引用配置文件及自动加载

自动加载 不可以使用 `tp` 自带的 `autoload.php`，需要引用 `aliyuncs/oss-sdk-php` 库里的 `autoload.php`

```shell
cd your/app/path/public/static/ueditor/php

vim action_upload.php
```

跟新并写入以下内容：

```php
<?php

// 引入自动加载
require_once realpath(dirname(__FILE__) . '/../../../../') . '/vendor/aliyuncs/oss-sdk-php/autoload.php';
// 引入配置文件
$diy = require realpath(dirname(__FILE__) . '/../../../../') . '/../../../../config/diy.php';

...
  
// 加入配置项
$config['diy'] = $diy;
/* 生成上传实例对象并完成上传 */
$up = new Uploader($fieldName, $config, $base64);
```

### 3.4 更改文件上传类

```shell
cd your/app/path/public/static/ueditor/php

vim Uploader.class.php
```

找到类中的方法：`upFile`

修改文件上传处

```php
...

private function upFile()
{
	...
	
	//检查是否不允许的文件格式
    if (!$this->checkType()) {
    	$this->stateInfo = $this->getStateInfo("ERROR_TYPE_NOT_ALLOWED");
    	return;
    }
    
    // 设置默认配置
    if (!isset($this->config['diy'])) {
    	$this->config['diy']['upload']['type'] = 'local';
    }
    $diy = $this->config['diy'];
    $upload_type = strtolower($diy['upload']['type']);
    // 本地上传
    if ($upload_type == 'local') {
        //创建目录失败
        if (!file_exists($dirname) && !mkdir($dirname, 0777, true)) {
            $this->stateInfo = $this->getStateInfo("ERROR_CREATE_DIR");
            return;
        } else if (!is_writeable($dirname)) {
            $this->stateInfo = $this->getStateInfo("ERROR_DIR_NOT_WRITEABLE");
            return;
    	}
		//移动文件
        if (!(move_uploaded_file($file["tmp_name"], $this->filePath) && file_exists($this->filePath))) { //移动失败
        	$this->stateInfo = $this->getStateInfo("ERROR_FILE_MOVE");
        } else { //移动成功
        	$this->stateInfo = $this->stateMap[0];
        }
	} elseif($upload_type == 'oss') {
        // 是否加载oss配置
        if (!isset($diy['oss'])) {
            $this->stateInfo = $this->getStateInfo("ERROR_NOT_FOUND_OSS_CONFIG");
            return;
        }
        $oss = $diy['oss'];
        try {
            $ossClient = new \OSS\OssClient($oss['accessKeyId'], $oss['accessKeySecret'], $oss['endPoint']);
            $result = $ossClient->uploadFile($oss['bucket'], ltrim($this->fullName, '/'), $file['tmp_name']);
            // 如果在config.json中没有配置 路径前缀，可更改该值，但是不推荐，后期若更换域名就会很麻烦
            // $this->fullName = $result['info']['url'];
            $this->stateInfo = $this->stateMap[0];
        } catch (\Exception $e) {
            $this->stateInfo = $this->getStateInfo($e->getMessage());
            return;
        }
    } else {
        $this->stateInfo = $this->getStateInfo("ERROR_UNKNOW_UPLOAD_TYPE");
        return;
    }
}

...
```

## 4、完成

接下来创建html并引入相关js访问即可。

> 在上传时使用的路径推荐使用由 `ueditor` 自行配置的，方便后续可自行打包传并更换使用位置。