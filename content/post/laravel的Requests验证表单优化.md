+++
tags = ["php", "laravel"]
categories = ["php"]
description = ""
menu = ""
banner = ""
images = []
title = "laravel的Requests验证表单优化"
date = "2020-05-10T11:10:25+08:00"
+++


# laravel的Requests验证表单优化

## 首先使用 artisan 建立 request 
```sh
 php artisan make:request 方法名

```

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/16/20200716111217.png)

就会在app/requests下生成对应文件名

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/16/20200716111253.png)

## 验证用户权限

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/16/20200716111322.png)

## 添加验证规则具体验证规则常考laravel手册

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/16/20200716111347.png)

## 设置返回提示内容

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/16/20200716111406.png)

## 返回json

这样就可以进行表单验证了，如果是前后分离，就需要返回的为json的数据给前台，就可以重写方法failedValidation

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/16/20200716111450.png)

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/16/20200716111522.png)

**返回示例：**

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/16/20200716111728.png)

## 完整代码

```php
<?php
namespace App\Http\Requests;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Contracts\Validation\Validator;
use Illuminate\Http\Exceptions\HttpResponseException;

class LoginPost extends FormRequest
{
    /**
     * Determine if the user is authorized to make this request.
     *
     * @return bool
     */
    //验证用户权限  true 为有权限
    public function authorize()
    {
        return true;
    }

    /**
     * Get the validation rules that apply to the request.
     *
     * @return array
     */
    public function rules()
    {
        return [
            'user_name'=>'required',
            'user_pwd'=>'required'
        ];
    }

    public function messages()
    {
        return [
            'user_name.required'=>'用户名不能为空',
            'user_pwd.required'=>'密码不能为空',
        ];
    }

    protected function failedValidation(Validator $validator)
    {
        throw (new HttpResponseException(response()->json([
            'code'=>100,
            $validator->errors(),
        ],422)));
    }
}

```

## 前台ajax 返回错误信息

```js
error: function (msg) {
    const json = JSON.parse(msg.responseText)[0];
    var text = "";
    let x;
    for (x in json) {
        text += json[x];
        text += '<br>';
    }
    layer.alert(text, {title: '提示', icon: 2});
}

```

## 优化

在后面的项目开发的时候，感觉前端的交互在错误渲染上，不太友好，前端交互不方便，这里我们处理了一下代码，现在得错误返回就是这样得：

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/16/20200716111908.png)

这样前端只需要直接渲染这个数组就行了，具体的代码实现也简单，只需要修改failedValidation方法就行：

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/16/20200716111957.png)
