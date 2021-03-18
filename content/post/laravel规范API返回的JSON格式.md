+++
tags = ["php", "laravel"]
categories = ["php"]
description = ""
menu = ""
banner = ""
images = []
title = "laravel规范API返回的JSON格式"
date = "2020-04-10T22:01:59+08:00"
+++


# laravel 规范API返回的JSON格式

**在最近开发项目的时候使用Laravel框架，对api进行返回数据的时候，在团队开发的时候太随意了，缺少一个统一的结构来包装返回值，所以就需要进行返回格式的统一**

**这里我就介绍两个方法：**

# ServiceProvider方法

## 创建一个ServiceProvider

命令行输入：

```sh
php artisan make:provider ResponseServiceProvider
```

## 在boot绑定response响应宏

这一步主要就是给自带的response方法添加一些自己的方法：

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/16/20200716220732.png)

```php
use Illuminate\Support\Facades\Response;
Response::macro('success', function ($code = 200, $msg = 'success', $data = '',$status = 200) {
    $content = array('code' => $code, 'msg' => $msg, 'data' => $data);
    return response()->json($content,$status);
});
Response::macro('fail', function ($code = 100, $msg = 'fail', $data = '',$status = 200) {
    $content = array('code' => $code, 'msg' => $msg, 'data' => $data);
    return response()->json($content,$status);
});

```

## 在config/app.php文件中注册

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/16/20200716220817.png)


## 控制器中的使用

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/16/20200716220846.png)


# composer全局绑定

## 定义返回函数

新建一个文件：json.php（这里的publicEncrypt是我进行数据加密的方法，你们直接删除掉就行）

```php
if (!function_exists('json_response')) {
    /**
     * 返回给页面json的信息，ajax操作时返回的提示信息
     * @param $code
     * @param $msg
     * @param null $data
     * @param int $status
     * @return \Illuminate\Http\JsonResponse
     */
    function json_response($code, $msg, $data = null, $status = 200)
    {
        return response()->json(array('code' => $code, 'msg' => $msg, 'data' => $data == null ? null : publicEncrypt($data)), $status);
    }
}
if (!function_exists('json_success')) {
    function json_success($msg = '操作成功!', $data = null)
    {
        return json_response(0, $msg, $data);
    }
}
if (!function_exists('json_fail')) {
    function json_fail($msg = '操作失败!', $data = null)
    {
        return json_response(1, $msg, $data);
    }
}

```
![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/16/20200716221718.png)


新建funtions.php（因为我这里还导入了其他的函数所以就有下面很多，主要的还是导入我们上面写的文件）

```php
/* 引入json相关辅助函数 */
require_once "json.php";

```

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/16/20200716221853.png)


## composer全局导入

打开composer.json文件,添加我们的funtions.php

```json
"files": [
    "app/Helpers/functions.php" //你的functions文件路径
]

```

然后只需要重新composer 更新一下就行：

```php
composer update

```

## 使用

控制器中的使用：

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/16/20200716222336.png)