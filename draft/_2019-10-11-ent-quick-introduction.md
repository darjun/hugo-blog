---
layout:		post
title:		"Facebook ent框架 系列一：快速入门"
subtitle: 	"快速搭建一个 ent 项目结构"
date:		2019-10-11T12:21:00
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - ent
URL: "2019/10/11/ent-quick-introduction/"
categories: [ 
	"Go"
]
---

## 概述

一个稍微有点用的项目，必然会存在多个对象实体（`Entity`）。如何处理存储、处理这些实体之间的关系是非常需要花费精力的地方。接下来一系列文章将会介绍如何使用`ent`框架简化这些操作。

[ent](https://entgo.io/) 是 Faceboo 开源的一个 Go 语言开发框架。它具有以下特点：

* 易于对图结构数据建模。
* 以代码来描述模式（`Schema`）。
* 通过代码生成，使项目代码基于静态类型，无反射。
* 简化图的遍历。

`ent`还内置了对 MySQL 和 sqlite3 的支持，非常简单易用。

## 安装

使用下面的命令安装：

```bash
go get github.com/facebookincubator/ent/cmd/entc
```

安装完成之后，会在`$GOPATH/bin`中生成一个可执行文件`entc`，这个就是`ent`框架的代码生成器。为了便于使用，需要将`entc`放到`PATH`目录中。我习惯将`$GOPATH/bin`加入`PATH`了，所以不需要进行其它操作就可以使用`entc`命令。

如果执行命令出现`strings.ReplaceAll undefined`错误，请将 Go 版本更新到**1.13**。

## 创建项目

在`$GOPATH/src/github.com/<user>/ent-go-example`目录下，创建我们的项目目录。将其中的`<user>`替换为你的用户名，例如我的用户名为`darjun`，命令如下：

```bash
$ mkdir -p $GOPATH/src/github.com/darjun/ent-go-example
$ cd $GOPATH/src/github.com/darjun/ent-go-example
```

### 创建 Schema

使用`entc`来创建 Schema 非常方便（`Schema`我们可以理解为模板，可以用来生成对象，与数据库中的表结构对应）。

```bash
$ entc int User
```

该命令会在项目根目录下创建如下的目录结构和文件`user.go`：

```
ent
└── schema
    └── user.go
```

通过`entc`命令创建的 Schema 都存放在`ent/schema`中。接下来，我们来看一看`user.go`文件的内容：

```golang
package schema

import "github.com/facebookincubator/ent"

// User holds the schema definition for the User entity.
type User struct {
    ent.Schema
}

// Fields of the User.
func (User) Fields() []ent.Field {
    return nil
}

// Edges of the User.
func (User) Edges() []ent.Edge {
    return nil
}
```

其中`Fields()`方法中定义`User`有哪些属性，`Edges()`方法中定义`User`与其他对象的关系。我们先为`User`添加几个属性：

```golang
package schema

import (
	"github.com/facebookincubator/ent"
	"github.com/facebookincubator/ent/schema/field"
)

// Fields of the User.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.String("name").Default("unknown"),
		field.Int("age").Positive(),
	}
}
```

我们通过子包`field`中的方法为`User`添加了 2 个属性，`name`和`age`，很好理解。接下来，使用`entc`生成具体结构以及操作的方法。在项目根目录执行：

```bash
$ entc generate ./ent/schema
```

生成的文件如下：

```
ent
├── client.go
├── config.go
├── context.go
├── ent.go
├── example_test.go
├── migrate
│   ├── migrate.go
│   └── schema.go
├── predicate
│   └── predicate.go
├── schema
│   └── user.go
├── tx.go
├── user
│   ├── user.go
│   └── where.go
├── user.go
├── user_create.go
├── user_delete.go
├── user_query.go
└── user_update.go
```

通过文件名大概也能猜到作用，先简单了解一下，后续文章还会说明：

* `client.go`：操作接口。
* `config.go`: 配置，如配置数据库驱动，日志输出等。
* `context.go`: 定义上下文`key`。
* `ent.go`: 定义一些查询和聚合选项，如升序/降序和`Count()`方法。
* `migrate/`: 包含创建表和迁移表的代码。
* `schema/`: 定义模式，**唯一需要我们编辑的目录**。
    * `user.go`: 定义`User`的属性和与其他对象的关系。
* `tx.go`: 数据库事务相关操作。
* `user/`: 
    * `user.go`: 定义数据库表相关常量。
    * `where.go`: 对每个字段定义丰富的查询条件。
* `user.go`: `User`结构和几个通用方法。
* `user_create.go/user_delete.go/user_query.go/user_update.go`: 定义创建/删除/查询/更新对象，这些对象会辅助生成 sql 语句，类比构造器模式。

### 创建

所有的操作都是通过`client`完成的，包括保存、查询、更新、删除等。`client`对象通过调用`ent.Open`获得，操作方式与打开数据库连接相同：

```golang
package main

import (
	"context"
	"log"

	"github.com/darjun/ent-go-example/ent"

	_ "github.com/go-sql-driver/mysql"
)

func main() {
	client, err := ent.Open("mysql", "root:12345@tcp(localhost:3306)/test?charset=utf8")
	if err != nil {
		log.Fatal(err)
	}
	defer client.Close()

	if err := client.Schema.Create(context.Background()); err != nil {
		log.Fatal(err)
	}

    if _, err := CreateUser(context.Background(), client); err != nil {
		log.Fatal(err)
	}
}

func CreateUser(ctx context.Context, client *ent.Client) (*ent.User, error) {
	u, err := client.User.
		Create().
		SetName("darjun").
		SetAge(29).
		Save(ctx)

	if err != nil {
		return nil, fmt.Errorf("failed creating user: %v", err)
	}

	log.Println("user was created: ", u)
	return u, nil
}
```

* 我们使用了 MySQL，所以需要导入驱动包`github.com/go-sql-driver/mysql`。这里假设 MySQL 已安装并启动。用户名、密码、端口号和数据库名请自行修改。
* `client.Schema.Create()`会**自动创建数据库表**，极大地简化了开发流程！！！
* 创建`User`时，使用了链式调用，`Create()`返回创建对象，后续两个方法设置字段值，最后`Save()`方法将之前设置的字段保存到数据库。

`User`默认对应`users`表，查看表结构：

| Column | Type | Defalut Value | Nullable | Character Set | Collation | Privileges | Extra | Comments |
|--------|------|---------------|----------|---------------|-----------|------------|-------|----------|
| id | bigint(20) | | NO | | | | select,insert,update,references | auto_increment | |
| name | varchar(255) | **unknown** | NO | utf8mb4 | utf8mb4_bin | select,insert,update,references | | |
| age | bigint(20) | | NO | | | select,insert,update,references | | |

我们看到：

* `ent`会默认生成一个自增的`id`字段。
* `name`和`age`列是我们通过`field.Type`指定的，注意`Defalut()`方法为`age`设置的默认值。后续会介绍更多的操作方式。

### 查询

对象的查询也是通过`client`来完成的，使用方式基本类似：

```golang
func QueryUser(ctx context.Context, client *ent.Client) (*ent.User, error) {
    u, err := client.User.
        Query().
        Where(user.NameEQ("darjun")).
        Only(ctx)
    if err != nil {
        return nil, fmt.Errorf("failed to query user: %v", err)
    }

    log.Println("query user:", u)
    return u, nil
}
```

* `Query()`返回查询对象，辅助构造 sql 语句。
* `Where()`指定查询条件，可以有多个`Where`，`user.NameEQ()`方法是`entc generate`生成的，位于`ent/user/where.go`文件中。类似的还有`NameNEQ`（不等）、`NameIn`（在一组值中）等。
* `Only()`执行查询语句，如果有多个结果会返回错误。

### 更新

更新操作也大同小异：

```golang
func UpdateUser(ctx context.Context, client *ent.Client) error {
	err := client.User.
		Update().
		Where(user.NameEQ("darjun")).
		SetAge(30).
		Exec(ctx)
	if err != nil {
		return fmt.Errorf("failed to update user: %v", err)
	}

	log.Println("update user")
	return nil
}
```

* `Update()`返回更新对象，辅助构造 sql 语句。
* `Where()`指定查询条件。
* `SetAge()`设置更新值。
* `Exec()`执行更新语句。

### 删除

删除操作也是类似的：

```golang
func DeleteUser(ctx context.Context, client *ent.Client) error {
	n, err := client.User.
		Delete().
		Where(user.NameEQ("darjun")).
		Exec(ctx)
	if err != nil {
		return fmt.Errorf("failed to delete user: %v", err)
	}

	log.Println("delete user cnt:", n)
	return nil
}
```

* `Delete()`返回删除对象，辅助构造 sql 语句。
* `Where()`指定查询条件。
* `Exec()`执行删除语句。

## 约定

代码生成器按照一定的模式生成代码，稍微熟悉一下就能应用地得心应手，例如：

* 创建、查询、更新和删除，需要调用对应结构的`Create()/Query()/Update()/Delete()`生成相应的辅助对象，然后再附加查询条件、设置值等，最后通过`Only()/All()`方法查询，`Exec()`方法执行。
* 每一个查询/执行的方法都有一个加`X`后缀的对应方法，例如`Only()`和`OnlyX()`。有`X`后缀的方法出错时，直接`panic`，而无`X`后缀的方法将返回错误。
* 代码生成器生成了丰富的条件位于`ent/<entity>/where.go`文件中，建议熟悉。

## 代码

本文代码放在 github 的 ent-go-example 仓库中，每篇文章对应的代码会指定不同的`tag`。例如本篇文章的代码`tag`为`introduction`，[去看代码](https://github.com/darjun/ent-go-example/tree/introduction)。