+++
tags = ["php", "laravel"]
categories = ["php"]
description = ""
menu = ""
banner = ""
images = []
title = "laravel实现定时任务"
date = "2020-05-25T10:11:13+08:00"
+++



# laravel实现定时任务

**原理是通过Cron**

## Cron简介

Cron 是 UNIX、SOLARIS、LINUX 下的一个十分有用的工具，通过 Cron 脚本能使计划任务定期地在系统后台自动运行。这种计划任务在 UNIX、SOLARIS、LINUX下术语为 Cron Jobs。Crontab 则是用来记录在特定时间运行的 Cron 的一个脚本文件，Crontab 文件的每一行均遵守特定的格式： 

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/16/20200716110047.png)

## 使用

￼ 我们可以在服务器上通过 crontab -e 来新增或编辑 Cron 条目，通过 crontab -l 查看已存在的 Cron 条目。 在以前，开发者需要为每一个需要调度的任务编写一个 Cron 条目，这是很让人头疼的事。你的任务调度不在源码控制中，你必须使用 SSH 登录到服务器然后添加这些 Cron 条目。 Laravel 命令调度器允许你流式而又不失优雅地在 Laravel 中定义命令调度，并且服务器上只需要一个 Cron 条目即可。任务调度定义在 app/Console/Kernel.php 文件的 schedule 方法中，该方法中已经包含了一个示例。
使用：

### 创建定时任务

> ```crontab -e```

在尾部添加代码

> ```* * * * * 《php路径》 《laravel项目路径》/artisan schedule:run >> /dev/null 2>&1```

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/16/20200716110351.png)

### 定义调度

**在App\Console\Commands下创建EmailTiming.php**

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/16/20200716110455.png)


**编辑 app/Console/Kernel.php 文件，将新生成的类进行注册：**

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/16/20200716110537.png)

常用：

```
->cron('* * * * *');    在自定义Cron调度上运行任务
->everyMinute();    每分钟运行一次任务
->everyFiveMinutes();   每五分钟运行一次任务
->everyTenMinutes();    每十分钟运行一次任务
->everyThirtyMinutes(); 每三十分钟运行一次任务
->hourly(); 每小时运行一次任务
->daily();  每天凌晨零点运行任务
->dailyAt('13:00'); 每天13:00运行任务
->twiceDaily(1, 13);    每天1:00 & 13:00运行任务
->weekly(); 每周运行一次任务
->monthly();    每月运行一次任务
->monthlyOn(4, '15:00');    每月4号15:00运行一次任务
->quarterly();  每个季度运行一次
->yearly(); 每年运行一次
->timezone('America/New_York'); 设置时区

```