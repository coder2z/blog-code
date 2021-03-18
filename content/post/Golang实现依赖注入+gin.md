+++
tags = ["golang", "依赖注入", "inject", "gin"]
categories = ["golang"]
description = ""
menu = ""
banner = ""
images = []
title = "Golang实现依赖注入+gin"
date = "2020-07-05T10:06:47+08:00"
+++


# Go实现依赖注入

最近在使用go开发的时候，发现构建系统依赖树非常繁琐，需要手动去new很多对象，又手工代码将它们拼接起来，写了一堆非常冗繁的代码。之前在laravel的使用中有其强大的ioc，都没有这方面的困扰。就在想golang中有没有好用的依赖注入方案，查询资料，发现了facebook团队开源的inject。GitHub地址：[http://github.com/facebookgo/inject](http://github.com/facebookgo/inject)


没有依赖注入的系统：
![](https://gitee.com/coder2m/pic/raw/master/img/20200709104638.png)

在代码上的表现：
```golang
func NewUserRepository() UserRepositoryImp {
	return &UserManagerRepository{
		Db: models.MysqlHandler,
	}
}

func NewUserServices(repository repositories.UserRepositoryImp) UserServiceImp {
	return &UserService{repository}
}

func NewUserController(userServices services.UserServiceImp) UserImp {
	return &UserController{UserServices: userServices}
}

repository := repositories.NewUserRepository()
userServices := services.NewUserServices(repository)
controller := controllers.NewUserController(userServices)

```

这里也能看出口想当的难受。而且重复代码很多。

## 使用的依赖
```golang
go get github.com/facebookgo/inject

```

## 简单的使用例子
```golang
type DBEngine struct {
	Name string
}

type UserDB struct {
	Db *DBEngine `inject:""`
}

type UserService struct {
	Db *UserDB `inject:""`
}

type App struct {
	Name string
	User *UserService `inject:""`
}

func (a *App) Create() string {
	return "create app, in db name:" + a.User.Db.Db.Name + " app name :" + a.Name
}

type Object struct {
	App *App
}

func Init() *Object {
	var g inject.Graph

	// 不适用依赖注入
	//a := DBEngine{Name: "db1"}
	//b := UserDB{&a}
	//c := UserService{&b}
	//app := App{Name: "go-app", User: &c}

	app := App{Name: "go-app"}

	_ = g.Provide(
		&inject.Object{Value: &DBEngine{Name: "db1"}},
		&inject.Object{Value: &app},
	)
	_ = g.Populate()

	return &Object{
		App: &app,
	}

}
func TestMains(t *testing.T) {
	obj := Init()
	fmt.Println(obj.App.Create())
}

```

这样很简单就实现了golang的依赖注入，看这个开源库的源码发现，整个类库的实现才500多行代码。这是多么轻量级的一个类库，只不过代码这么短，功能也不会太多，相比laravel的依赖注入而言，它的功能就单一太多了。不过没关系，相比Guice而言这些缺失的功能不是必须的，能帮我们省掉很多代码它已经做得很好了，这就足够了。


所以最上面的代码就可以修改为：
```golang
	var userController controllers.UserController
	var injector inject.Graph
	_ = injector.Provide(
		&inject.Object{Value: &repositories.UserManagerRepository{Db: models.MysqlHandler}},
		&inject.Object{Value: &services.UserService{}},
		&inject.Object{Value: &userController},
	)
	_ = injector.Populate()

```

## 在gin使用依赖注入

```golang
func TestInject(t *testing.T) {
	models.Init()
	models.MysqlHandler.AutoMigrate(models.User{})
	//使用 Inject New方法就不用写了
	var userController controllers.UserController
	var injector inject.Graph
	_ = injector.Provide(
		&inject.Object{Value: &repositories.UserManagerRepository{Db: models.MysqlHandler}},
		&inject.Object{Value: &services.UserService{}},
		&inject.Object{Value: &userController},
	)
	_ = injector.Populate()

	app := gin.Default()
	api := app.Group("/api")
	{
		api.POST("/login", userController.Login)
		api.POST("/register", userController.Register)
		api.GET("/me", middleware.Auth(), userController.Info)
	}
	_ = app.Run(":8080")
}

func TestNoInject(t *testing.T) {
	models.Init()
	models.MysqlHandler.AutoMigrate(models.User{})
	//不使用 Inject
	repository := repositories.NewUserRepository()
	userServices := services.NewUserServices(repository)
	controller := controllers.NewUserController(userServices)

	app := gin.Default()
	api := app.Group("/api")
	{
		api.POST("/login", controller.Login)
		api.POST("/register", controller.Register)
		api.GET("/me", middleware.Auth(), controller.Info)
	}
	_ = app.Run(":8080")
}

```

源码：[https://github.com/coder2m/shopping](https://github.com/coder2m/shopping)
