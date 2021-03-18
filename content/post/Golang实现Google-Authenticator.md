+++
tags = ["golang", "Google-Authenticator"]
categories = ["golang"]
description = ""
menu = ""
banner = ""
images = []
title = "Golang实现Google-Authenticator"
date = "2020-08-05T20:43:46+08:00"
+++


# 什么是Google-Authenticator

Google身份验证器是一款基于时间与哈希的一次性密码算法的两步验证软件令牌。也就是我们署成的TOTP（Time-based One-time Password）

通俗的说就是：***密钥+算法=code***

通过控制变量法，这里我们只需要手机上也设置一样的密钥使用一样的算法，就可以生成一样的code，从而达到二次验证。

# 使用Go实现生成密码算法

```go
/**
* @Author: myxy99 <myxy99@foxmail.com>
* @Date: 2020/8/4 16:56
 */
package utils

import (
	"crypto/hmac"
	"crypto/sha1"
	"encoding/base32"
	"fmt"
	"hash"
	"math/rand"
	"net/url"
	"strings"
	"time"
)

var (
	codes   = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
	codeLen = len(codes)
)

type GoogleAuth struct {
	user, issuer, secret string
}

func getCode(key []byte, value []byte) uint32 {
	var (
		hmacSha1         hash.Hash
		bytes, hashParts []byte
		offset           uint8
		number, pwd      uint32
	)
	hmacSha1 = hmac.New(sha1.New, key)
	hmacSha1.Write(value)
	bytes = hmacSha1.Sum(nil)
	offset = bytes[len(bytes)-1] & 0x0F
	hashParts = bytes[offset : offset+4]
	hashParts[0] = hashParts[0] & 0x7F
	number = toUint32(hashParts)
	pwd = number % 1000000
	return pwd
}

func toBytes(value int64) []byte {
	var (
		result []byte
		mask   int64
		shifts [8]uint16
	)
	mask = int64(0xFF)
	shifts = [8]uint16{56, 48, 40, 32, 24, 16, 8, 0}
	for _, shift := range shifts {
		result = append(result, byte((value>>shift)&mask))
	}
	return result
}

func toUint32(bytes []byte) uint32 {
	return (uint32(bytes[0]) << 24) + (uint32(bytes[1]) << 16) +
		(uint32(bytes[2]) << 8) + uint32(bytes[3])
}

func (g *GoogleAuth) getCode() (code string, err error) {
	var (
		key      []byte
		secret   string
		codeUI32 uint32
	)
	secret = strings.ToUpper(strings.Replace(g.secret, " ", "", -1))
	key, err = base32.StdEncoding.DecodeString(secret)
	if err != nil {
		return "", nil
	}
	codeUI32 = getCode(key, toBytes(time.Now().Unix()/30))
	code = fmt.Sprintf("%0*d", 6, codeUI32)
	return
}

//生成url
func (g *GoogleAuth) ProvisionURI() (codeUrl string) {
	auth := "totp/"
	q := make(url.Values)
	q.Add("secret", g.secret)
	if g.issuer != "" {
		q.Add("issuer", g.issuer)
		auth += g.issuer + ":"
	}
	return "otpauth://" + auth + g.user + "?" + q.Encode()
}

//生成验证code正确
func (g *GoogleAuth) Authenticate(code string) (ok bool, err error) {
	var (
		nowCode string
	)
	nowCode, err = g.getCode()
	return nowCode == code, err
}

//生成密钥
func (g *GoogleAuth) GenerateKey() string {
	data := make([]byte, 32)
	rand.Seed(time.Now().UnixNano())
	for i := 0; i < 32; i++ {
		idx := rand.Intn(codeLen)
		data[i] = byte(codes[idx])
	}
	g.secret = string(data)
	return string(data)
}

//设置密钥
func (g *GoogleAuth) SetKey(key string) {
	g.secret = key
	return
}

func NewGoogleAuth(user string, issuer ...string) *GoogleAuth {
	var iss string
	if len(issuer) == 1 {
		iss = issuer[0]
	}
	return &GoogleAuth{
		user:   user,
		issuer: iss,
	}
}

```

# 使用

## 服务端

```go
func TestGoogle(t *testing.T) {
	auth := utils.NewGoogleAuth("admin@myxy99.cn")
	//生成密钥
	//key := auth.GenerateKey()
	//fmt.Println(key)

	//已经有密钥了，设置密钥
	auth.SetKey("TsJjVwEIcMDXuREaXuQgEMdpcRvJXfBV")
	//url生成
	fmt.Println(auth.ProvisionURI())

	//验证code
	ok, _ := auth.Authenticate("890456")
	fmt.Println(ok)
}

```

## 客户端

客户端只需要在手机上下载**Authenticator**添加服务端通过**ProvisionURI**方法生成的url，就可以达到验证效果:

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/08/05/IMG_0610.PNG)


这样就可以很方便的在你的网站中添加google验证器了。