+++
tags = ["php", "laravel"]
categories = ["php"]
description = ""
menu = ""
banner = ""
images = []
title = "laravel通过观察者监听模型事件"
date = "2020-05-20T10:08:28+08:00"
+++


# 所有支持的模型事件

在 Eloquent 模型类上进行查询、插入、更新、删除操作时，会触发相应的模型事件（关于事件我们后面会单独讲），不管你有没有监听它们。这些事件包括：

|  方法名   | 说明  |
|  ----  | ----  |
| retrieved  | 获取到模型实例后触发 |
| creating  | 插入到数据库前触发 |
|created |插入到数据库后触发|
|updating |更新到数据库前触发|
|updated |更新到数据库后触发|
|saving |保存到数据库前触发（插入/更新之前，无论插入还是更新都会触发）|
|saved |保存到数据库后触发（插入/更新之后，无论插入还是更新都会触发）|
|deleting |从数据库删除记录前触发|
|deleted |从数据库删除记录后触发|
|restoring |恢复软删除记录前触发|
|restored |恢复软删除记录后触发|

# 通过观察者监听模型事件

针对模型事件这种特殊的事件类型，Laravel 还为我们提供了观察者类来处理模型事件的监听。观察者可以看作是上述订阅者处理模型事件的简化版本，我们不需要自定义事件类，不需要建立映射关系，只需要在观察者类中将需要监听的事件定义为同名方法，并在相应方法中编写业务处理代码即可。当某个模型事件触发时，Eloquent 底层会去该模型上注册的观察者类中通过反射查找是否定义了对应的方法，如果定义了则执行相应的逻辑，否则忽略。

## 创建观察者

首先，我们通过 Artisan命令初始化针对 User 模型的观察者：（laravel 5.8以上）

```sh
php artisan make:observer UserObserver --model=User

```

## 编写处理代码

默认生成的 UserObserver 会为 created、 updated、deleted、restored、forceDeleted（强制删除） 事件定义一个空方法。

你可以把前面定义的retrived、deleting、deleted 事件监听代码迁移过来，也可以将不需监听的事件方法移除，这里我们将编写保存模型时涉及的模型事件，包括 saving、creating、updating、updated、created、saved：

```php
namespace App\Observers; 
use App\User;
use Illuminate\Support\Facades\Log;
classUserObserver{ 
    public function saving(User $user){
       Log::info('即将保存用户到数据库['.$user->id.']'.$user->name);
       }
    public function creating(User $user){
       Log::info('即将插入用户到数据库['.$user->id.']'.$user->name);
       }
    public function updating(User $user){
      Log::info('即将更新用户到数据库['.$user->id.']'.$user->name);
      }
    public function updated(User $user){
      Log::info('已经更新用户到数据库['.$user->id.']'.$user->name);
      }
    public function created(User $user){
     Log::info('已经插入用户到数据库['.$user->id.']'.$user->name);
     }
    public function saved(User $user){
     Log::info('已经保存用户到数据库['.$user->id.']'.$user->name);
     }
 }

```

## 注册

编写好观察者后，需要将其注册到 User 模型上才能生效，我们可以在 EventServiceProvider 的 boot 方法中完成该工作：

```php
public function boot(){ 
    parent::boot();    
    User::observe(UserObserver::class);
}

```

## 测试

测试观察者监听模型事件。我们先编写一段添加新用户的代码：

```php
$user=new User();
$user->name='test1';
$user->email='239999@qq.com';
$user->password=bcrypt('123456');
$user->save();

```

执行上述代码后，即可在日志文件中看到相应的日志记录：

```log
[2019-11-13 10:30:00] local.INFO: 即将保存用户到数据库[]test1  
[2019-11-13 10:30:00] local.INFO: 即将插入用户到数据库[]test1  
[2019-11-13 10:30:00] local.INFO: 已经插入用户到数据库[19]test1  
[2019-11-13 10:30:00] local.INFO: 已经保存用户到数据库[19]test1  

```

编写一段更新已存在用户的代码：

```php
$user= User::findOrFail(15);
$user->name='test2';
$user->save();

```

执行上述代码，对应的日志记录如下：

```log
[2019-11-13 10:33:00] local.INFO: 从模型中获取用户[15]:Miss Lucile Langosh V  
[2019-11-13 10:33:00] local.INFO: 即将保存用户到数据库[15]test2 
[2019-11-13 10:33:00] local.INFO: 即将更新用户到数据库[15]test2  
[2019-11-13 10:33:00] local.INFO: 已经更新用户到数据库[15]test2 
[2019-11-13 10:33:00] local.INFO: 已经保存用户到数据库[15]test2

```