---
title: "About"
date: 2021-03-18T22:40:02+08:00
draft: false
---

# 关于我
成都东软学院信息与软件工程系的本科在读学生，喜欢coding，生活，音乐，电影，爱捣鼓网站上的一些奇怪的东西，拥有轻度选择恐惧症。

主攻方向：Go后端开发
``` golang
type PersonalInformation struct {
	Name        string
	Age         int64
	Address     string
	Email       string
	Description string
}
type Skills struct {
	Skill []string
}
func Me() (PersonalInformation, Skills) {
	information := PersonalInformation{
		Name:        "YangZemiao",
		Age:         21,
		Address:     "SiChuan",
		Email:       "myxy99@foxmail.com",
		Description: "Like you like liking coding",
	}
	skill := Skills{
		Skill: []string{"Golang", "php", "python", "JS"},
	}
	return information, skill
}

```

致敬我的开始！

```golang
func main()  {
	fmt.Println("Hello, World!")
	return
}

```
