+++
tags = ["golang"]
categories = ["golang"]
description = ""
menu = ""
banner = ""
images = []
title = "Golang使用jwt"
date = "2020-07-07T20:35:19+08:00"
+++

# Golang中使用JWT(json web token)

## 什么是jwt
什么是jwt这里就不多说了，官网有介绍。官网介绍：[https://jwt.io/introduction/](https://jwt.io/introduction/)

## 使用步骤

### 下载依赖包

```golang
go get -u github.com/dgrijalva/jwt-go

```

### 编写代码

#### 配置文件

```ini
[jwt]
# 盐
JwtSecret = jdnsakjbduiiudu
# 过期时间（天
ExpiresAt = 3
# 签发
Issuer = jwt

```

#### golang代码

```golang
package Utils

import (
	"errors"
	"github.com/dgrijalva/jwt-go"
	"time"
	"wPan/v1/Config" //加载配置
	"wPan/v1/Models" //加载模型
)

// JWT 签名结构
type JWT struct {
	SigningKey []byte
}

//JWT 中存储得数据结构体 这里可以根据需求自由添加
type UserInfo struct {
	Id       int    `json:"id"`
	Username string `json:"username"`
	Email    string `json:"email"`
	Status   int    `json:"status"`
}

// 生成JWT函数
func GenerateToken(user *Models.User) (string, error) {
	claim := jwt.MapClaims{
		//这里为自定义
		"username": user.UserName,
		"id":       user.ID,
		"email":    user.Email,
		"status":   user.Status,
		//到这里，后面都是必须的
		"nbf": time.Now().Unix(),
		"iat": time.Now().Unix(),
		//设置过期时间
		"exp": time.Now().Unix() + Config.JWTSetting.ExpiresAt*60*60,
		//签名方
		"iss": Config.JWTSetting.Issuer,
	}
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claim)
	tokens, err := token.SignedString([]byte(Config.JWTSetting.JwtSecret))
	return tokens, err
}

//设置盐
func secret() jwt.Keyfunc {
	return func(token *jwt.Token) (interface{}, error) {
		return []byte(Config.JWTSetting.JwtSecret), nil
	}
}

//token 验证
func ParseToken(tokens string) (user *UserInfo, err error) {
	user = &UserInfo{}
	token, err := jwt.Parse(tokens, secret())
	if err != nil {
		return
	}
	claim, ok := token.Claims.(jwt.MapClaims)
	if !ok {
		err = errors.New("cannot convert claim to mapclaim")
		return
	}
	//验证token，如果token被修改过则为false
	if !token.Valid {
		err = errors.New("token is invalid")
		return
	}
	//提取当初我们生成时候存储的数据
	user.Id = int(claim["id"].(float64))
	user.Username = claim["username"].(string)
	user.Email = claim["email"].(string)
	user.Status = int(claim["status"].(float64))

	return
}

```

Service中的使用

```golang
func GetJwt(u *Models.User) (string, bool) {
	token, err := Utils.GenerateToken(u)
	if err != nil {
		return "", false
	}
	return token, true
}

```

#### 编写验证中间件
我这里使用的gin框架，其他的框架也类似
```golang
func Auth() gin.HandlerFunc {
	return func(c *gin.Context) {
		token := c.GetHeader("Authorization")
		userInfo, err := Utils.ParseToken(token[7:])
		if err != nil {
			R.Response(c, http.StatusUnauthorized, R.AUTH_ERROR, nil, http.StatusUnauthorized)
			c.Abort()
			return
		}
		c.Set("userInfo", userInfo)
		c.Next()
		return
	}
}

```

#### 路由中使用
```golang
authApi.GET("/info", Middleware.Auth(), Controllers.Info)

```

这样用户再访问需要验证的请求时，就需要请求头中带上``Authorization``，value为token。

参考连接：[https://github.com/dgrijalva/jwt-go](https://github.com/dgrijalva/jwt-go)