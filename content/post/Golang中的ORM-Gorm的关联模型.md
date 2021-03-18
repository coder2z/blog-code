+++
tags = ["golang", "orm", "gorm"]
categories = ["golang"]
description = ""
menu = ""
banner = ""
images = []
title = "Golang中的ORM-Gorm的关联模型"
date = "2020-07-10T09:09:28+08:00"
+++


# Golang中的ORM-Gorm的关联模型

对于gorm的基础CRUD用法，这里就不论述了，这里主要说下关联模型的问题，因为我自己在查看官方文档进行关联模型操作的时候，总是感觉官方的例子很奇怪，用着很不明白。对于gorm的基础CRUD用法，不明白的可以看看官方文档：[https://gorm.io/zh_CN/docs/models.html](https://gorm.io/zh_CN/docs/models.html)


# 创建数据库层面的外键：
```golang
models.MysqlHandler.Model(&models.Order{}).AddForeignKey("user_id", "user(id)", "RESTRICT", "RESTRICT")
```

说说这里的RESTRICT，这里还可以填CASCADE、NO ACTION、RESTRICT、SET NULL。分别的意思是：

> CASCADE：父表delete、update的时候，子表会delete、update掉关联记录；
>
> SET NULL：父表delete、update的时候，子表会将关联记录的外键字段所在列设为null，所以注意在设计子表时外键不能设为not null；
> 
> RESTRICT：如果想要删除父表的记录时，而在子表中有关联该父表的记录，则不允许删除父表中的记录；
> 
> NO ACTION：同 RESTRICT，也是首先先检查外键；

# Belongs To

belongs to 关联建立一个和另一个模型的一对一连接，使得模型声明每个实例都```属于```另一个模型的一个实例 。

## 定义模型

例如，如果你的应用包含了用户和用户所属部门， 并且每一个用户所属只分配给一个用户。模型定义：
```golang
type User struct {
	gorm.Model
	Name string
}

// `Department` 属于 `User`， 外键是`UserID`
type Department struct {
	gorm.Model
	UserID uint
	User   User
	Name   string
}

```

生成的表结构：
![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/11/20200711101256.png)

## 外键
默认的外键使用所有者类型名称加上其主键。像上面的例子，为了声明一个模型属于 User，它的外键应该为 UserID。

GORM 提供了一个定制外键的方法，例如:

```golang
type User struct {
    gorm.Model
    Name string
}

type Department struct {
  gorm.Model
  Name      string
  User      User `gorm:"foreignkey:UserDepartment"` // 使用 UserRefer 作为外键
  UserDepartment uint
}

```

 > 后面其他连接方式也一样

## 关联外键
GORM 默认使用所有者的主键作为外键值，在上面的例子中，就是 User 的 ID。当你分配一个资料给一个用户， GORM 将保存用户表的 ID 值 到 用户资料表的 UserID 字段里。你可以通过改变标签 association_foreignkey 外键值：

```golang
type User struct {
    gorm.Model
    Department uint
    Name string
}

type Department struct {
  gorm.Model
  Name      string
  User      User `gorm:"association_foreignkey:Refer"` // use Department 作为关联外键
  UserDepartment uint
}

```
 > 后面其他连接方式也一样

## Belongs To 的使用

这里就是重点了，这里我直接放代码：

```golang
type User struct {
	gorm.Model
	Name string
}

// `Department` 属于 `User`， 外键是`UserID`
type Department struct {
	gorm.Model
	UserID uint
	User   User
	Name   string
}
//这里的MysqlHandler就是gorm中的数据库操作DB
func TestGormBelongsTo(t *testing.T) {
	MysqlHandler = mysqlBuild()
	MysqlHandler.AutoMigrate(User{})
	MysqlHandler.AutoMigrate(Department{})

	//链表添加数据
	//profile := Profile{
	//	User: User{
	//		Name: "张三",
	//	},
	//	Name: "项目组",
	//}
	//MysqlHandler.Create(&profile)

	//链表查询
	//var infoList []Department
	//var user User
	//MysqlHandler.Preload("User").Find(&infoList)
	//for _, value := range infoList {
	//	fmt.Println(value.Name)
	//	fmt.Println(value.User.Name)
	//}

	// 查询添加条件 这里的条件就是 Department id为2 对应的数据
	//var info Department
	//info.ID = 2
	//MysqlHandler.Debug().Preload("User").Find(&info)
	//fmt.Println(info.Name, info.User.Name)

	//USER name为张三A 对应的数据
	//var info Department
	//MysqlHandler.Debug().Preload("User", func(query *gorm.DB) *gorm.DB {
	//	return query.Where("name =? ", "张三A")
	//}).First(&info)
	//fmt.Println(info.Name, info.User.Name, info.User.ID)
	//	or
	//var infoList []Department
	//MysqlHandler.Debug().Preload("User", "name =?","张三A").First(&infoList)
	//for _, value := range infoList {
	//	fmt.Println(value.Name, value.User.Name, value.User.ID)
	//}

	// 使用 Related 查找 belongs to 关系
	var department Department               //需要查找总的结构体
	department.ID = 3                       //定义查询条件
	MysqlHandler.Debug().First(&department) //首先查询总表
	/// SELECT * FROM `department`  WHERE `department`.`deleted_at` IS NULL AND `department`.`id` = 3 ORDER BY `department`.`id` ASC LIMIT 1
	MysqlHandler.Debug().Model(&department).Related(&department.User) //查询子表进行赋值
	/// SELECT * FROM `user`  WHERE `user`.`deleted_at` IS NULL AND ((`id` = 2))
	fmt.Println(department.Name, "---", department.User.Name, "---", department.ID, "---", department.User.ID)
}

```

由于gorm支持链式操作，后续需要什么操作 再往上加就行。

# Has One

## 定义模型

```golang
//车
type Car struct {
	gorm.Model
	Host         string //车主人名字
	LicensePlate LicensePlate
}

//车牌
type LicensePlate struct {
	gorm.Model
	Number string //车牌号
	CarID  uint
}

```
## 操作

Has One的操作很Belongs To相同，可以自己实操一下。就明白了。


# Belongs To跟Has One的区别

同样是一对一 Belongs To跟Has One的区别是什么呢？从两个结构体不难看出差别。区别： 外键属性存在位置不同，foreignKey 指定源不同，targetKey 指定源不同

# Has Many

has many 关联就是创建和另一个模型的一对多关系， 不像 has one，所有者可以拥有0个或多个模型实例。

## 定义模型


如果你的业务数据库包含学校和专业， 并且每一个学校都拥有多门专业。

```golang
//学校
type School struct {
	gorm.Model
	Name       string
	Profession []Profession
}

//专业
type Profession struct {
	gorm.Model
	Name     string
	SchoolId uint
}

```

## 操作
```golang
func TestGormHasMany(t *testing.T) {
	MysqlHandler = mysqlBuild()
	MysqlHandler.AutoMigrate(School{})
	MysqlHandler.AutoMigrate(Profession{})

	//创建
	//profession1 := Profession{Name: "信息工程"}
	//profession2 := Profession{Name: "计算机科学"}
	//var professionList []Profession
	//professionList = append(professionList, profession1)
	//professionList = append(professionList, profession2)
	//school := School{
	//	Name:       "成都大学",
	//	Profession: professionList,
	//}
	//MysqlHandler.Save(&school) //or Create

	//查询单列
	//var school School
	//MysqlHandler.Preload("Profession").First(&school)
	//fmt.Println(school)

	//var school School
	//MysqlHandler.First(&school)
	//MysqlHandler.Model(&school).Related(&school.Profession)
	//fmt.Println(school)

	//查询多列
	var school []School
	//MysqlHandler.First(&school)
	MysqlHandler.Model(&school).Preload("Profession").Find(&school)
	for _, value := range school {
		fmt.Println(value.Name, value.Profession)
	}
}

```

# Many To Many

多对多为两个模型增加了一个中间表。


## 定义模型
例如，如果你的应用包含用户和语言， 一个用户会说多种语言，并且很多用户会说一种特定的语言。

```golang
// 用户拥有并属于多种语言， 使用  `user_languages` 作为中间表
type User struct {
    gorm.Model
    Languages         []Language `gorm:"many2many:user_languages;"`
}

type Language struct {
    gorm.Model
    Name string
    Users               []User     `gorm:"many2many:user_languages;"`
}

```

## 操作

```golang
func TestGormManyToMany(t *testing.T) {
	MysqlHandler = mysqlBuild()
	MysqlHandler.AutoMigrate(Users{})
	MysqlHandler.AutoMigrate(Language{})

	//	创建
	//langEN := Language{Name: "EN"}
	//langCN := Language{Name: "CN"}
	//u1 := &Users{
	//	Name: "user1",
	//	Languages: []Language{
	//		langEN,
	//		langCN,
	//	},
	//}
	//MysqlHandler.Create(u1)

	//LangCN := Language{}
	//MysqlHandler.Where("name=?","CN").First(&LangCN)
	//u2 := &Users{
	//	Name: "user2",
	//	Languages: []Language{
	//		LangCN,
	//	},
	//}
	//MysqlHandler.Create(u2)

	//查询
	//获取 用户id 为 3 的 user 的语言：
	//var users Users
	//MysqlHandler.Find(&users, 3)
	//MysqlHandler.Model(&users).Related(&users.Languages, "Languages")


	//查询
	//获取 使用语言 为 CN 的 user：
	//var language Language
	//MysqlHandler.Find(&language, "name=?","CN")
	//MysqlHandler.Model(&language).Related(&language.Users, "Users")
	//fmt.Println(language)
}

```
参考连接：[https://gorm.io/zh_CN/docs/](https://gorm.io/zh_CN/docs/)

文章代码地址：[https://github.com/coder2m/shopping/blob/master/test/gorm_test.go](https://github.com/coder2m/shopping/blob/master/test/gorm_test.go)