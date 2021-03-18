+++
tags = ["golang", "gin"]
categories = ["golang"]
description = ""
menu = ""
banner = ""
images = []
title = "Gin表单绑定验证器"
date = "2020-07-13T08:34:28+08:00"
+++


# gin中内置validator的基础使用

```golang
type UserLoginParam struct {
	Name     string `form:"name" json:"name" binding:"required,min=2,max=30"`
	Password string `form:"password" json:"password" binding:"required,min=8,max=40"`
}

func TestValidator(t *testing.T) {
	app := gin.New()
	app.POST("/login", func(context *gin.Context) {
		var userLoginService UserLoginParam
		err := context.ShouldBind(&userLoginService)
		if err != nil {
			context.JSON(http.StatusUnprocessableEntity, gin.H{
				"msg": err.Error(),
			})
		}else{
			context.JSON(http.StatusOK, gin.H{
				"msg": "通过验证",
			})
		}
		return
	})
	_ = app.Run(":8888")
}

```

我们使用postman请求测试下：

拒绝：
![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/13/20200713084519.png)

通过：
![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/13/20200713084911.png)

可以看到表单验证已经生效，但是错误提示的字段不是特别友好，我们首先需要整理成前端能够解析的样子，在考虑翻译的问题。

# 优化返回格式

我们这里可以看到所有的验证输出都是来之或者ShouldBind后的err，我们就先看看这个err在哪里定义的，这里可以通过fmt包查看他的类型，就知道他定义的位置。

```golang
fmt.Printf("%T \n", err)
      
```

这里发现他的类型为:

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/13/20200713085854.png)

下一步就是找到他的定义了。然后发现他是一个FieldError切片类型：

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/13/20200713090619.png)

然后我们继续看看FieldError是个什么东西：

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/13/20200713090827.png)

发现这里的FieldError是一个接口类型，里面的接口想必就是获取各种信息的。我们打印一下看看，所有我们需要改一下我们的代码，这里我就随便打印几个，其实我们看下官方在接口定义哪里的注释，也大概能明白什么意思。

```golang
		if err != nil {
			fmt.Printf("%T \n", err)
			errors := err.(validator.ValidationErrors)

			for _, value := range errors {
				fmt.Println(value.Kind())
				fmt.Println(value.Field())
				fmt.Println(value.ActualTag())
			}


			context.JSON(http.StatusUnprocessableEntity, gin.H{
				"msg": err.Error(),
			})
		} else {
			context.JSON(http.StatusOK, gin.H{
				"msg": "通过验证",
			})
		}
```

访问接口=>输出

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/13/20200713091434.png)

这里也就很明显了。我们现在就能拿到我们错误信息进行封装返回了。我们小改一下代码：

```golang
func getParamError(err validator.ValidationErrors) map[string]string {
	result := make(map[string]string, 0)
	for _, v := range err {
		result[v.Field()] = v.Tag()
	}
	return result
}

func TestValidator(t *testing.T) {
	app := gin.Default()
	app.POST("/login", func(context *gin.Context) {
		var userLoginService UserLoginParam
		err := context.ShouldBind(&userLoginService)
		if err != nil {
			fmt.Printf("%T \n", err)
			errors := err.(validator.ValidationErrors)
			context.JSON(http.StatusUnprocessableEntity, gin.H{
				"msg": getParamError(errors),
			})
		} else {
			context.JSON(http.StatusOK, gin.H{
				"msg": "通过验证",
			})
		}
		return
	})
	_ = app.Run(":8888")
}

```

现在访问一下再看看：

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/13/20200713092201.png)

但是这样对于前端，还是不好获取，所有我把他改为数组。前端只需要循环这个数组就行。

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/13/20200713101702.png)



# 自定义字段校验方法

```golang
type UserLoginParam struct {
	Name     string `form:"name" json:"name" binding:"required,min=2,max=30,bookableDate"`
	Password string `form:"password" json:"password" binding:"required,min=8,max=40"`
}

func getParamError(err validator.ValidationErrors) []string {
	result := make([]string, 0)
	for _, v := range err {
		result = append(result, fmt.Sprintf("%v错误：%v", v.Field(), v.Tag()))
	}
	return result
}

// customFunc 自定义字段级别校验方法
var bookableDate validator.Func = func(fl validator.FieldLevel) bool {
	date, ok := fl.Field().Interface().(time.Time)
	if ok {
		today := time.Now()
		if today.After(date) {
			return false
		}
	}
	return true
}

func TestValidator(t *testing.T) {

	if v, ok := binding.Validator.Engine().(*validator.Validate); ok {
		//注册
		_ = v.RegisterValidation("bookableDate", bookableDate)
	}

	app := gin.Default()
	app.POST("/login", func(context *gin.Context) {
		var userLoginService UserLoginParam
		err := context.ShouldBind(&userLoginService)
		if err != nil {
			errors := err.(validator.ValidationErrors)
			context.JSON(http.StatusUnprocessableEntity, gin.H{
				"msg": getParamError(errors),
			})

		} else {
			context.JSON(http.StatusOK, gin.H{
				"msg": "通过验证",
			})
		}
		return
	})
	_ = app.Run(":8888")
}

```
参考连接：[https://gin-gonic.com/zh-cn/docs/examples/custom-validators/](https://gin-gonic.com/zh-cn/docs/examples/custom-validators/)

参考连接：[https://github.com/go-playground/validator/](https://github.com/go-playground/validator/)

参考连接：[https://www.liwenzhou.com/posts/Go/validator_usages/](https://www.liwenzhou.com/posts/Go/validator_usages/)