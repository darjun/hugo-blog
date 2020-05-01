---
layout:    post
title:    "Go 每日一库之 sqlc"
subtitle: 	"每天学习一个 Go 库"
date:		2020-04-28T23:09:23
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/04/28/godailylib/sqlc"
categories: [
	"Go"
]
---

## 简介

在 Go 语言中编写数据库操作代码真的非常痛苦！`database/sql`标准库提供的都是比较底层的接口。我们需要编写大量重复的代码。大量的模板代码不仅写起来烦，而且还容易出错。有时候字段类型修改了一下，可能就需要改动很多地方；添加了一个新字段，之前使用`select *`查询语句的地方都要修改。如果有些地方有遗漏，可能就会造成运行时`panic`。即使使用 ORM 库，这些问题也不能完全解决！这时候，`sqlc`来了！`sqlc`可以根据我们编写的 SQL 语句生成类型安全的、地道的 Go 接口代码，我们要做的只是调用这些方法。

## 快速使用

先安装：

```golang
$ go get github.com/kyleconroy/sqlc/cmd/sqlc
```

当然还有对应的数据库驱动：

```golang
$ go get github.com/lib/pq
$ go get github.com/go-sql-driver/mysql
```

`sqlc`是一个命令行工具，上面代码会将可执行程序`sqlc`放到`$GOPATH/bin`目录下。我习惯把`$GOPATH/bin`目录加入到系统`PATH`中。所以可以执行使用这个命令。

因为`sqlc`用到了一个 linux 下的库，在 windows 上无法正常编译。在 windows 上我们可以使用 docker 镜像`kjconroy/sqlc`。docker 的安装就不介绍了，网上有很多教程。拉取`kjconroy/sqlc`镜像：

```golang
$ docker pull kjconroy/sqlc
```

然后，编写 SQL 语句。在`schema.sql`文件中编写建表语句：

```sql
CREATE TABLE authors (
  id   BIGSERIAL PRIMARY KEY,
  name TEXT NOT NULL,
  bio  TEXT
);
```

在`query.sql`文件中编写查询语句：

```sql
-- name: GetAuthor :one
SELECT * FROM authors
WHERE id = $1 LIMIT 1;

-- name: ListAuthors :many
SELECT * FROM authors
ORDER BY name;

-- name: CreateAuthor :exec
INSERT INTO authors (
  name, bio
) VALUES (
  $1, $2
)
RETURNING *;

-- name: DeleteAuthor :exec
DELETE FROM authors
WHERE id = $1;
```

`sqlc`支持 PostgreSQL 和 MySQL，**不过对 MySQL 的支持是实验性的**。期待后续完善对 MySQL 的支持，增加对其它数据库的支持。本文我们使用的是 PostgreSQL。编写数据库程序时，上面两个 sql 文件是少不了的。`sqlc`额外只需要一个小小的配置文件`sqlc.yaml`：

```golang
version: "1"
packages:
  - name: "db"
    path: "./db"
    queries: "./query.sql"
    schema: "./schema.sql"
```

* `version`：版本；
* `packages`：
	* `name`：生成的包名；
	* `path`：生成文件的路径；
	* `queries`：查询 SQL 文件；
	* `schema`：建表 SQL 文件。

在 windows 上执行下面的命令生成对应的 Go 代码：

```golang
docker run --rm -v CONFIG_PATH:/src -w /src kjconroy/sqlc generate
```

上面的`CONFIG_PATH`替换成配置所在目录，我的是`D:\code\golang\src\github.com\darjun\go-daily-lib\sqlc\get-started`。`sqlc`为我们在同级目录下生成了数据库操作代码，目录结构如下：

```golang
db
├── db.go
├── models.go
└── query.sql.go
```

`sqlc`根据我们`schema.sql`和`query.sql`生成了模型对象结构：

```golang
// models.go
type Author struct {
  ID   int64
  Name string
  Bio  sql.NullString
}
```

和操作接口：

```golang
// query.sql.go
func (q *Queries) CreateAuthor(ctx context.Context, arg CreateAuthorParams) (Author, error)
func (q *Queries) DeleteAuthor(ctx context.Context, id int64) error
func (q *Queries) GetAuthor(ctx context.Context, id int64) (Author, error)
func (q *Queries) ListAuthors(ctx context.Context) ([]Author, error)
```

其中`Queries`是`sqlc`封装的一个结构。

说了这么多，来看看如何使用：

```golang
package main

import (
  "database/sql"
  "fmt"
  "log"

  _ "github.com/lib/pq"
  "golang.org/x/net/context"

  "github.com/darjun/go-daily-lib/sqlc/get-started/db"
)

func main() {
  pq, err := sql.Open("postgres", "dbname=sqlc sslmode=disable")
  if err != nil {
    log.Fatal(err)
  }

  queries := db.New(pq)

  authors, err := queries.ListAuthors(context.Background())
  if err != nil {
    log.Fatal("ListAuthors error:", err)
  }
  fmt.Println(authors)

  insertedAuthor, err := queries.CreateAuthor(context.Background(), db.CreateAuthorParams{
    Name: "Brian Kernighan",
    Bio:  sql.NullString{String: "Co-author of The C Programming Language and The Go Programming Language", Valid: true},
  })
  if err != nil {
    log.Fatal("CreateAuthor error:", err)
  }
  fmt.Println(insertedAuthor)

  fetchedAuthor, err := queries.GetAuthor(context.Background(), insertedAuthor.ID)
  if err != nil {
    log.Fatal("GetAuthor error:", err)
  }
  fmt.Println(fetchedAuthor)

  err = queries.DeleteAuthor(context.Background(), insertedAuthor.ID)
  if err != nil {
    log.Fatal("DeleteAuthor error:", err)
  }
}
```

生成的代码在包`db`下（由`packages.name`选项指定），首先调用`db.New()`将`sql.Open()`的返回值`sql.DB`作为参数传入，得到`Queries`对象。我们对`authors`表的操作都需要通过该对象的方法。

上面程序要运行，还需要启动 PostgreSQL，创建数据库和表：

```cmd
$ createdb sqlc
$ psql -f schema.sql -d sqlc
```

上面第一条命令创建一个名为`sqlc`的数据库，第二条命令在数据库`sqlc`中执行`schema.sql`文件中的语句，即创建表。

最后运行程序（多文件程序不能用`go run main.go`）：

```cmd
$ go run .
[]
{1 Brian Kernighan {Co-author of The C Programming Language and The Go Programming Language true}}
{1 Brian Kernighan {Co-author of The C Programming Language and The Go Programming Language true}}
```

## 代码生成

除了 SQL 语句本身，`sqlc`需要我们在编写 SQL 语句的时候通过注释的方式为生成的程序提供一些基本信息。语法为：

```golang
-- name: <name> <cmd>
```

`name`为生成的方法名，如上面的`CreateAuthor/ListAuthors/GetAuthor/DeleteAuthor`等，`cmd`可以有以下取值：

* `:one`：表示 SQL 语句返回一个对象，生成的方法的返回值为`(对象类型, error)`，对象类型可以从表名得出；
* `:many`：表示 SQL 语句会返回多个对象，生成的方法的返回值为`([]对象类型, error)`；
* `:exec`：表示 SQL 语句不返回对象，只返回一个`error`；
* `:execrows`：表示 SQL 语句需要返回受影响的行数。

### `:one`

```sql
-- name: GetAuthor :one
SELECT id, name, bio FROM authors
WHERE id = $1 LIMIT 1
```

注释中`--name`指示生成方法`GetAuthor`，从表名得出返回的基础类型为`Author`。`:one`又表示只返回一个对象。故最终的返回值为`(Author, error)`：

```golang
// db/query.sql.go
const getAuthor = `-- name: GetAuthor :one
SELECT id, name, bio FROM authors
WHERE id = $1 LIMIT 1
`

func (q *Queries) GetAuthor(ctx context.Context, id int64) (Author, error) {
  row := q.db.QueryRowContext(ctx, getAuthor, id)
  var i Author
  err := row.Scan(&i.ID, &i.Name, &i.Bio)
  return i, err
}
```

### `:many`

```sql
-- name: ListAuthors :many
SELECT * FROM authors
ORDER BY name;
```

注释中`--name`指示生成方法`ListAuthors`，从表名`authors`得到返回的基础类型为`Author`。`:many`表示返回一个对象的切片。故最终的返回值为`([]Author, error)`：

```golang
// db/query.sql.go
const listAuthors = `-- name: ListAuthors :many
SELECT id, name, bio FROM authors
ORDER BY name
`

func (q *Queries) ListAuthors(ctx context.Context) ([]Author, error) {
  rows, err := q.db.QueryContext(ctx, listAuthors)
  if err != nil {
    return nil, err
  }
  defer rows.Close()
  var items []Author
  for rows.Next() {
    var i Author
    if err := rows.Scan(&i.ID, &i.Name, &i.Bio); err != nil {
      return nil, err
    }
    items = append(items, i)
  }
  if err := rows.Close(); err != nil {
    return nil, err
  }
  if err := rows.Err(); err != nil {
    return nil, err
  }
  return items, nil
}
```

这里注意一个细节，即使我们使用了`select *`，生成的代码中 SQL 语句被也改写成了具体的字段：

```sql
SELECT id, name, bio FROM authors
ORDER BY name
```

这样后续如果我们需要添加或删除字段，只要执行了`sqlc`命令，这个 SQL 语句和`ListAuthors()`方法就能保持一致！是不是很方便？

### `:exec`

```sql
-- name: DeleteAuthor :exec
DELETE FROM authors
WHERE id = $1
```

注释中`--name`指示生成方法`DeleteAuthor`，从表名`authors`得到返回的基础类型为`Author`。`:exec`表示不返回对象。故最终的返回值为`error`：

```golang
// db/query.sql.go
const deleteAuthor = `-- name: DeleteAuthor :exec
DELETE FROM authors
WHERE id = $1
`

func (q *Queries) DeleteAuthor(ctx context.Context, id int64) error {
  _, err := q.db.ExecContext(ctx, deleteAuthor, id)
  return err
}
```

### `:execrows`

```sql
-- name: DeleteAuthorN :execrows
DELETE FROM authors
WHERE id = $1
```

注释中`--name`指示生成方法`DeleteAuthorN`，从表名`authors`得到返回的基础类型为`Author`。`:exec`表示返回受影响的行数（即删除了多少行）。故最终的返回值为`(int64, error)`：

```golang
// db/query.sql.go
const deleteAuthorN = `-- name: DeleteAuthorN :execrows
DELETE FROM authors
WHERE id = $1
`

func (q *Queries) DeleteAuthorN(ctx context.Context, id int64) (int64, error) {
  result, err := q.db.ExecContext(ctx, deleteAuthorN, id)
  if err != nil {
    return 0, err
  }
  return result.RowsAffected()
}
```

不管编写的 SQL 多复杂，总是逃不过上面的规则。我们只需要在编写 SQL 语句时额外添加一行注释，`sqlc`就能为我们生成地道的 SQL 操作方法。生成的代码与我们自己手写的没什么不同，错误处理都很完善，而且了避免手写的麻烦与错误。

### 模型对象

`sqlc`为所有的建表语句生成对应的模型结构。结构名为表名的单数形式，且首字母大写。例如：

```sql
CREATE TABLE authors (
  id   SERIAL PRIMARY KEY,
  name text   NOT NULL
);
```

生成对应的结构：

```golang
type Author struct {
  ID   int
  Name string
}
```

而且`sqlc`可以解析`ALTER TABLE`语句，它会根据最终的表结构来生成模型对象的结构。例如：

```sql
CREATE TABLE authors (
  id          SERIAL PRIMARY KEY,
  birth_year  int    NOT NULL
);

ALTER TABLE authors ADD COLUMN bio text NOT NULL;
ALTER TABLE authors DROP COLUMN birth_year;
ALTER TABLE authors RENAME TO writers;
```

上面的 SQL 语句中，建表时有两列`id`和`birth_year`。第一条`ALTER TABLE`语句添加了一列`bio`，第二条删除了`birth_year`列，第三条将表名`authors`改为`writers`。`sqlc`依据最终的表名`writers`和表中的列`id`、`bio`生成代码：

```golang
package db

type Writer struct {
  ID  int
  Bio string
}
```

### 配置字段

`sqlc.yaml`文件中还可以设置其他的配置字段。

#### `emit_json_tags`

默认为`false`，设置该字段为`true`可以为生成的模型对象结构添加 JSON 标签。例如：

```sql
CREATE TABLE authors (
  id         SERIAL    PRIMARY KEY,
  created_at timestamp NOT NULL
);
```

生成：

```golang
package db

import (
  "time"
)

type Author struct {
  ID        int       `json:"id"`
  CreatedAt time.Time `json:"created_at"`
}
```

#### `emit_prepared_queries`

默认为`false`，设置该字段为`true`，会为 SQL 生成对应的`prepared statement`。例如，在**快速开始**的示例中设置这个选项，最终生成的结构`Queries`中会添加所有 SQL 对应的`prepared statement`对象：

```golang
type Queries struct {
  db                DBTX
  tx                *sql.Tx
  createAuthorStmt  *sql.Stmt
  deleteAuthorStmt  *sql.Stmt
  getAuthorStmt     *sql.Stmt
  listAuthorsStmt   *sql.Stmt
}
```

和一个`Prepare()`方法：

```golang
func Prepare(ctx context.Context, db DBTX) (*Queries, error) {
  q := Queries{db: db}
  var err error
  if q.createAuthorStmt, err = db.PrepareContext(ctx, createAuthor); err != nil {
    return nil, fmt.Errorf("error preparing query CreateAuthor: %w", err)
  }
  if q.deleteAuthorStmt, err = db.PrepareContext(ctx, deleteAuthor); err != nil {
    return nil, fmt.Errorf("error preparing query DeleteAuthor: %w", err)
  }
  if q.getAuthorStmt, err = db.PrepareContext(ctx, getAuthor); err != nil {
    return nil, fmt.Errorf("error preparing query GetAuthor: %w", err)
  }
  if q.listAuthorsStmt, err = db.PrepareContext(ctx, listAuthors); err != nil {
    return nil, fmt.Errorf("error preparing query ListAuthors: %w", err)
  }
  return &q, nil
}
```

生成的其它方法都使用了这些对象，而非直接使用 SQL 语句：

```golang
func (q *Queries) CreateAuthor(ctx context.Context, arg CreateAuthorParams) (Author, error) {
  row := q.queryRow(ctx, q.createAuthorStmt, createAuthor, arg.Name, arg.Bio)
  var i Author
  err := row.Scan(&i.ID, &i.Name, &i.Bio)
  return i, err
}
```

我们需要在程序初始化时调用这个`Prepare()`方法。

#### `emit_interface`

默认为`false`，设置该字段为`true`，会为查询结构生成一个接口。例如，在**快速开始**的示例中设置这个选项，最终生成的代码会多出一个文件`querier.go`：

```golang
// db/querier.go
type Querier interface {
  CreateAuthor(ctx context.Context, arg CreateAuthorParams) (Author, error)
  DeleteAuthor(ctx context.Context, id int64) error
  DeleteAuthorN(ctx context.Context, id int64) (int64, error)
  GetAuthor(ctx context.Context, id int64) (Author, error)
  ListAuthors(ctx context.Context) ([]Author, error)
}
```

### 覆写类型

`sqlc`在生成模型对象结构时会根据数据库表的字段类型推算出一个 Go 语言类型，例如`text`对应`string`。我们也可以在配置文件中指定这种类型映射。

```yaml
version: "1"
packages:
  - name: "db"
    path: "./db"
    queries: "./query.sql"
    schema: "./schema.sql"
overrides:
  - go_type: "github.com/uniplaces/carbon.Time"
    db_type: "pg_catalog.timestamp"
```

在`overrides`下`go_type`表示使用的 Go 类型。如果是非标准类型，必须指定**全限定类型**（即包路径 + 类型名）。`db_type`设置为要映射的数据库类型。`sqlc`会自动导入对应的标准包或第三方包。生成代码如下：

```golang
package db

import (
  "github.com/uniplaces/carbon"
)

type Author struct {
  ID       int32
  Name     string
  CreateAt carbon.Time
}
```

**需要注意的是`db_type`的表示，文档这里一笔带过，使用上还是有些晦涩。**我也是看源码才找到如何覆写`timestamp`类型的，需要将`db_type`设置为`pg_catalog.timestamp`。同理`timestamptz`、`timetz`等类型也需要加上这个前缀。**一般复杂类型都需要加上前缀，一般的基础类型可以加也可以不加。**遇到不确定的情况，可以去看看源码[gen.go#L634](https://github.com/kyleconroy/sqlc/blob/c7d3e4b067980323f5f33280a26a031cb743c92f/internal/dinosql/gen.go#L634)。

也可以设定某个字段的类型，例如我们要将创建时间字段`created_at`设置为使用`carbon.Time`：

```yaml
version: "1"
packages:
  - name: "db"
    path: "./db"
    queries: "./query.sql"
    schema: "./schema.sql"
overrides:
  - column: "authors.create_at"
    go_type: "github.com/uniplaces/carbon.Time"
```

生成代码如下：

```golang
// db/models.go
package db

import (
  "github.com/uniplaces/carbon"
)

type Author struct {
  ID       int32
  Name     string
  CreateAt carbon.Time
}
```

最后我们还可以给生成的结构字段命名：

```yaml
version: "1"
packages:
  - name: "db"
    path: "./db"
    queries: "./query.sql"
    schema: "./schema.sql"
rename:
    id: "Id"
    name: "UserName"
    create_at: "CreateTime"
```

上面配置为生成的结构设置字段名，生成代码：

```golang
package db

import (
  "time"
)

type Author struct {
  Id         int32
  UserName   string
  CreateTime time.Time
}
```

## 安装 PostgreSQL

我之前使用 MySQL 较多。由于`sqlc`对 MySQL 的支持不太好，在体验这个库的时候还是选择支持较好的  PostgreSQL。不得不说，在 win10 上，PostgreSQL 的**安装门槛**实在是太高了！我摸索了很久最后只能在[https://www.enterprisedb.com/download-postgresql-binaries](https://www.enterprisedb.com/download-postgresql-binaries)下载可执行文件。我选择了 10.12 版本，下载、解压、将文件夹中的`bin`加入系统`PATH`。创建一个`data`目录，然后执行下面的命令初始化数据：

```cmd
$ initdb data
```

注册 PostgreSQL 服务，这样每次系统重启后会自动启动：

```cmd
$ pg_ctl register -N "pgsql" -D D:\data
```

**这里的`data`目录就是上面创建的，并且一定要使用绝对路径！**

启动服务：

```cmd
$ sc start pgsql
```

最后使用我们前面介绍的命令创建数据库和表即可。

如果有使用`installer`成功安装的小伙伴，还请不吝赐教！

## 总结

虽然目前还有一些不完善的地方，例如对 MySQL 的支持是实验性的，`sqlc`工具的确能大大简化我们使用 Go 编写数据库代码的复杂度，提升我们的编码效率，减少我们出错的概率。使用 PostgreSQL 的小伙伴非常建议尝试一番！

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. sqlc GitHub：[https://github.com/kyleconroy/sqlc](https://github.com/kyleconroy/sqlc)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)