---
title: Go Web下GORM框架使用（一）
date: 2023-06-01 21:40:32
tags: golang 前端 数据库
categories: Golang 前端
---

<!--more-->

## 〇、GORM
ORM库的目标是通过将对象和数据库表之间的映射关系定义为代码，从而提供一种更加面向对象的方式来处理数据库操作。

GORM提供了一组丰富的功能和API，使开发人员可以方便地进行数据库查询、插入、更新和删除操作，而无需直接编写SQL语句。它支持多种数据库，包括MySQL、PostgreSQL、SQLite等，并提供了事务处理、模型关联、预加载、自动迁移等功能。


## 一、连接数据库
通过框架提供的gorm.Open()函数连接数据库，前提是我们的数据库已经处于被选择的状态。

```go
	// 连接数据库
	// 默认端口为 3306
	db, err := gorm.Open("mysql", "root:password@tcp(127.0.0.1:3306)/dbname?charset=utf8mb4&parseTime=True&loc=Local")
	if err != nil {
		panic(err)
	}
	// 最后关闭数据库
	defer func(db *gorm.DB) {
		err := db.Close()
		if err != nil {
		}
	}(db)
```

## 二、创建
我们要在数据库中创建一个表单 table，可以使用迁移的方法，但是前提我们要定一个模型：

```go
type UserInfo struct {
	ID     uint
	Name   string
	Gender string
	Hobby  string
}
```
在这个模型定义好之后，我们直接可以用一个函数将该结构体映射成一个 table：

```go
db.AutoMigrate(&UserInfo{})
```
然后我们就可以向这个 table 中传入数据了：

```go
	u1 := UserInfo{1, "cxk", "男", "篮球"}
	u2 := UserInfo{2, "cxk", "男", "足球"}

	// 创建记录
	db.Create(&u1)
	db.Create(&u2)
```
当然，有时候我们希望表单中用默认数据：

```go
type User struct {
  ID   int64
  Name string `gorm:"default:'小王子'"`
  Age  int64
}
```

##  三、数据的查询
### （一）一般查询
```go
// 根据主键查询第一条记录
db.First(&user)
//// SELECT * FROM users ORDER BY id LIMIT 1;

// 随机获取一条记录
db.Take(&user)
//// SELECT * FROM users LIMIT 1;

// 根据主键查询最后一条记录
db.Last(&user)
//// SELECT * FROM users ORDER BY id DESC LIMIT 1;

// 查询所有的记录
db.Find(&users)
//// SELECT * FROM users;

// 查询指定的某条记录(仅当主键为整型时可用)
db.First(&user, 10)
//// SELECT * FROM users WHERE id = 10;
```
### （二）条件查询

```go
// Get first matched record
db.Where("name = ?", "jinzhu").First(&user)
//// SELECT * FROM users WHERE name = 'jinzhu' limit 1;

// Get all matched records
db.Where("name = ?", "jinzhu").Find(&users)
//// SELECT * FROM users WHERE name = 'jinzhu';

// <>
db.Where("name <> ?", "jinzhu").Find(&users)
//// SELECT * FROM users WHERE name <> 'jinzhu';

// IN
db.Where("name IN (?)", []string{"jinzhu", "jinzhu 2"}).Find(&users)
//// SELECT * FROM users WHERE name in ('jinzhu','jinzhu 2');

// LIKE
db.Where("name LIKE ?", "%jin%").Find(&users)
//// SELECT * FROM users WHERE name LIKE '%jin%';

// AND
db.Where("name = ? AND age >= ?", "jinzhu", "22").Find(&users)
//// SELECT * FROM users WHERE name = 'jinzhu' AND age >= 22;

// Time
db.Where("updated_at > ?", lastWeek).Find(&users)
//// SELECT * FROM users WHERE updated_at > '2000-01-01 00:00:00';

// BETWEEN
db.Where("created_at BETWEEN ? AND ?", lastWeek, today).Find(&users)
//// SELECT * FROM users WHERE created_at BETWEEN '2000-01-01 00:00:00' AND '2000-01-08 00:00:00';
```
除了以上查询，还有其它各种查询方法，不在累赘。

## 四、更新
`Save()`默认会更新该对象的所有字段，即使没有赋值：
```go
	db.First(&user)

	user.Name = "七米"
	user.Age = 99
	db.Save(&user)
```
## 五、删除数据

只得注意的是，删除记录时，请确保主键字段有值，GORM 会通过主键去删除记录，如果主键为空，GORM 会删除该 model 的所有记录。
```go
// 删除现有记录
db.Delete(&email)
//// DELETE from emails where id=10;

// 为删除 SQL 添加额外的 SQL 操作
db.Set("gorm:delete_option", "OPTION (OPTIMIZE FOR UNKNOWN)").Delete(&email)
//// DELETE from emails where id=10 OPTION (OPTIMIZE FOR UNKNOWN);
```
### （一）软删除
如果一个 model 有 `DeletedAt` 字段，他将自动获得软删除的功能！ 当调用 Delete 方法时， 记录不会真正的从数据库中被删除， 只会将`DeletedAt `字段的值会被设置为当前时间。

```go
db.Delete(&user)
//// UPDATE users SET deleted_at="2013-10-29 10:23" WHERE id = 111;

// 批量删除
db.Where("age = ?", 20).Delete(&User{})
//// UPDATE users SET deleted_at="2013-10-29 10:23" WHERE age = 20;

// 查询记录时会忽略被软删除的记录
db.Where("age = 20").Find(&user)
//// SELECT * FROM users WHERE age = 20 AND deleted_at IS NULL;

// Unscoped 方法可以查询被软删除的记录
db.Unscoped().Where("age = 20").Find(&users)
//// SELECT * FROM users WHERE age = 20;
```

*全文完，感谢阅读。*




