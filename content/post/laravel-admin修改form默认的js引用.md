+++
tags = ["laravel", "laravel-admin", "php"]
categories = ["php"]
description = ""
menu = ""
banner = ""
images = []
title = "laravel-admin修改form默认的js引用"
date = "2020-05-22T10:33:36+08:00"
+++

# 解决laravel-admin中form表单下拉框不是中文问题
#### 最近在使用laravel-admin开发得时候。在做表单提交的时候发现默认的提醒为英文的这里我就想能不能设置为中文：

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/16/20200716103834.png)

#### 然后我就在线怎么操作能够改为中文，我这里通过了通过在开发工具中搜索，我发现这里的渲染为js进行的：

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/16/20200716104007.png)

#### 这里我发现在对应得文件夹中也有中文版本得js只是没有使用上：

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/16/20200716104113.png)

#### 所以这里我就在想怎么修改默认的js路径，于是我开始使用开发工具继续搜索这个js名

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/16/20200716104335.png)

#### 结果在这个文件中发现了js的设置，现在就要想怎么修改他了，直接修改他的源代码肯定是不行的，所以这里我想的就是继承这个类在进行重写变量赋值。

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/16/20200716104425.png)

#### 现在的问题就是怎么让我们的类生效了，查看了官方文档发现只需要在bootstrap中注册就行了

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/16/20200716104458.png)

#### 现在再查看下页面就变成中文了

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/16/20200716104532.png)

#### 这里其实就是一个小问题，但是主要是再发现问题得时候，怎么一步一步得找到问题，进行解决。记录一下这个思路

参考连接：[https://laravel-admin.org/docs/zh](https://laravel-admin.org/docs/zh)