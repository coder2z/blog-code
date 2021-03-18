+++
tags = ["php", "laravel"]
categories = ["php"]
description = ""
menu = ""
banner = ""
images = []
title = "laravel中的路由模型绑定"
date = "2020-04-25T09:48:12+08:00"
+++


# laravel中的路由模型绑定

在使用laravel进行开发的时候，会发现我们的控制器中多次出现了 $article = Article::findOrFail($id); 这一段代码。如果我们的程序能够自动找到 ID 对应的记录，并让 Laravel 自动查询，这是不是非常的棒？这就是路由模型绑定会做到的事情。

查看程序目录，我们会发现一个 app/Providers 文件夹，这里面的文件定义了 Laravel 如何创建或者启动组件。app/Providers/RouteServiceProvider.php 就是定义程序路由如何启动的。

# 定义路由

这里我使用资源路由：

```php
Route::resource('articles', 'ArticlesController')
```

查看路由列表:

```sh
D:\wamp\www\laravel5>php artisan route:list

 articles                     | articles.index
 articles/create              | articles.create
 articles                     | articles.store
 articles/{articles}          | articles.show
 articles/{articles}/edit     | articles.edit
 articles/{articles}          | articles.update
 articles/{articles}          | articles.destroy

```

# RouteServiceProvider

然后我们修改RouteServiceProvider文件：

```php
public function boot(Router $router)
{
	parent::boot($router);
    $router->model('articles','App\Article::class');
}

```

其中 model() 的参数 articles 是路由中的 key 值。例如 routes.php 中定义了路由 Route::get('foo/{bar}', function(){}); ，则 key 值就是 bar。第二个参数是需要绑定的模型。Articles控制器中的 key 值为 articles，所以在 boot() 方法中我们写的就是 articles 。

# 使用

经过上面的修改，Articles 控制器中的 show($id) 方法的参数 $id 已经不再是文章的ID了，可以在修改 show() 方法来看一下：

```php
public function show($id){
    $article =Article::findOrFail($id);
 	dd($article);
    return view('articles.show', compact('article'));
}

```

现在随便打开一篇文章的详细页，可以看到输出了下面的文章类：

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/17/20200717100138.png)

同理 show() ，edit() ， update()

```php
public function show(Article $article){
          return view('articles.show', compact('article'));
}
public function edit(Article $article){
  return view('articles.edit', compact('article'));
}
public function update(Article $article,ArticleRequest $request){
  $article->update($request->all());
  return redirect('articles');
}

```


**如果我们想做一些更多的操作，如添加 where() 条件等等，则可以修改 boot() 中的内容，使用 bind() 方法来返回结果：**

```php
public function boot(Router $router)
{
  parent::boot($router);
  $router->bind('articles',function($id){
    return \App\Article::published()->findOrFail($id);
  });
//$router->model('articles', 'App\Article');
}

```