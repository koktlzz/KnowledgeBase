---
title: "使用 MySQL 数据库"
date: 2020-11-04T09:19:42+01:00
lastmod: 2020-11-04T09:19:42+01:00
draft: false
menu:
  go:
    parent: "Go Web"
weight: 400
---

## 连接数据库

```go
package main

import (
    "context"
    "database/sql"
    "fmt"
    "log"

    _ "github.com/mikespook/mymysql"
)

func main() {
    var db *sql.DB
    // 第二个参数的格式为：user:password@tcp(ip: port)/dbname
    db, _ = sql.Open("mysql", "******")
    err := db.Ping()
    if err != nil {
        log.Fatalln(err.Error())
    }
    fmt.Println("CONNECTION SUCCESSFUL")
}
```

### 数据库驱动

想要连接到 SQL 数据库，那么首先需要加载目标数据库的驱动，驱动中包含了与数据库交互的逻辑。正常获取数据库驱动的方法是调用 sql.Register() 函数进行注册：

```go
var (
    driversMu sync.RWMutex
    // 使用一个 map 来储存用户定义的数据库驱动
    drivers   = make(map[string]driver.Driver)
)

func Register(name string, driver driver.Driver) {
    driversMu.Lock()
    defer driversMu.Unlock()
    if driver == nil {
        panic("sql: Register driver is nil")
    }
    if _, dup := drivers[name]; dup {
        panic("sql: Register called twice for driver " + name)
    }
    // 将用户定义的数据库驱动添加到 map 中
    drivers[name] = driver
}
```

它的第一个参数是数据库驱动的名称，在本例中为 mysql。第二个参数是一个结构体，类型为 driver.Driver：

```go
type Driver interface {
    // Open returns a new connection to the database.
    // The name is a string in a driver-specific format.
    // Open may return a cached connection (one previously closed), but doing so is unnecessary; 
    // the sql package maintains a pool of idle connections for efficient re-use.
    // The returned connection is only used by one goroutine at a time.
    Open(name string) (Conn, error)
}
```

Driver 的底层分析详见：[https://github.com/astaxie/build-web-application-with-golang/blob/master/zh/05.1.md](https://github.com/astaxie/build-web-application-with-golang/blob/master/zh/05.1.md)

而我们在注册数据库时却并没有调用 Register() 函数，而是引入了一个数据库驱动的包：*github.com/mikespook/mymysql*：

```go
// go get 命令安装数据库驱动包
_ "github.com/mikespook/mymysql"

// mymysql 包内部：
var d = Driver{proto: "tcp", raddr: "127.0.0.1:3306"}
func init() {
    Register("SET NAMES utf8")
    sql.Register("mymysql", &d)
}
```

由于 Go 引入 package 时，会自动调用自身的 init() 函数。因此只要我们引入数据库驱动包，就通过它自身的 init() 函数完成了数据库驱动的注册。所有第三方的数据库驱动包都实现 database/sql/driver 中定义的接口。

> 新手都会被这个`_`所迷惑，其实这个就是 Go 设计的巧妙之处，我们在变量赋值的时候经常看到这个符号，它是用来忽略变量赋值的占位符，那么包引入用到这个符号也是相似的作用，这儿使用`_`的意思是引入后面的包名而不直接使用这个包中定义的函数，变量等资源。

### 调用 sql.Open()

```go
func Open(driverName, dataSourceName string) (*DB, error) {
    driversMu.RLock()
    driveri, ok := drivers[driverName]
    driversMu.RUnlock()
    if !ok {
        return nil, fmt.Errorf("sql: unknown driver %q (forgotten import?)", driverName)
    }

    if driverCtx, ok := driveri.(driver.DriverContext); ok {
        connector, err := driverCtx.OpenConnector(dataSourceName)
        if err != nil {
            return nil, err
        }
        return OpenDB(connector), nil
    }

    return OpenDB(dsnConnector{dsn: dataSourceName, driver: driveri}), nil
}
```

第一个参数 driverName 为数据库驱动名称，在本例中为 mysql。第二个参数为数据源名称，它支持如下格式：

- user@unix(/path/to/socket)/dbname?charset=utf8
- user:password@tcp(localhost:5555)/dbname?charset=utf8
- user:password@/dbname
- user:password@tcp([de:ad:be:ef::ca:fe]:80)/dbname

该方法返回一个指向 sql.DB 结构体类型的指针。

Open() 函数并不会真正的连接数据库，甚至不会验证其参数。它只是把后续连接数据库所需的 sql.DB 设置好，而真正的连接是在后续操作数据库时才建立的。

### sql.DB

```go
type DB struct {
    connector driver.Connector
    mu           sync.Mutex // protects following fields
    freeConn     []*driverConn
    // Used to signal the need for new connections
    // a goroutine running connectionOpener() reads on this chan and
    // maybeOpenNewConnections sends on the chan (one send per needed connection)
    // It is closed during db.Close(). The close tells the connectionOpener
    // goroutine to exit.
    openerCh          chan struct{
        closed            bool
        stop func() // stop cancels the connection opener and the session resetter.
        ...
}
```

sql.DB 可以操作数据库，它代表了 0 个或 1 个数据库连接池，这些连接由 sql 包进行维护。sql 包会自动的创建和释放这些连接，而且它对于多个 goroutine 并发的使用的安全的。

最后通过 sql.DB 的 Ping() 方法检查数据库连接是否成功。

```go
// Ping 检查与数据库的连接是否仍有效，如果需要会创建连接。
func (db *DB) Ping() error
```

## 操作数据库

在数据库 kokt 中新建一张表 userinfo 并添加一些记录：

```sql
CREATE TABLE `userinfo` (
    `uid` INT(10) NOT NULL AUTO_INCREMENT,
    `username` VARCHAR(64) NULL DEFAULT NULL,
    `department` VARCHAR(64) NULL DEFAULT NULL,
    `created` DATE NULL DEFAULT NULL,
    PRIMARY KEY (`uid`)
);
```

![20210108031143](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20210108031143.png)

### 查询

sql.DB 的 Query() 方法可以执行一次 select 命令，传入 SQL 语句，返回多行结果（即结构体类型 Rows）。第一个参数 query 表示 SQL 语句，第二个参数 args 表示 query 中的占位参数。

```go
func (db *DB) Query(query string, args ...interface{}) (*Rows, error)
```

Rows 拥有的方法主要有：

- func (rs *Rows) Columns() ([]string, error)；Columns 返回列名。如果 Rows 已经关闭会返回错误。
- func (rs *Rows) Scan(dest ...interface{}) error；Scan 将当前行各列结果填充进 dest 指定的各个值中。
- func (rs *Rows) Next() bool；Next 准备用于 Scan 方法的下一行结果。如果成功会返回真，如果没有下一行或者出现错误会返回假。
- func (rs *Rows) Close() error；Close 关闭 Rows，阻止对其更多的列举。
- func (rs *Rows) Err() error；Err 返回可能的、在迭代时出现的错误。Err 需在显式或隐式调用 Close 方法后调用。

查询表中 departmen 字段为 marin 的记录，并将结果填充到一个结构体切片中：

```go
type result struct {
    uid        int
    username   string
    department string
    created    string
}
var results []result
department := "marin"
// 使用？传递参数
rw, err := db.Query("select * from userinfo where department = ?", department)
defer rw.Close()
// 当读取数据完毕后，rw.Next() 返回 false，退出循环
for rw.Next() {
    // temp 代表查询得到的一条记录
    temp := result{} 
    err = rw.Scan(&temp.uid, &temp.username, &temp.department, &temp.created)
    if err != nil {
        log.Fatal(err)
    }
    // 将查询记录添加到结果切片中
    results = append(results, temp)
}
fmt.Println(results)
```

上述程序输出结果为：

```go
[{4 luffy marin 2020-01-31} {5 ace marin 2020-01-01}]
```

### 更新/插入/删除

sql.DB 的 Exec() 方法可以执行一次命令（包括查询、删除、更新、插入等），传入 SQL 语句。返回值类型为 Result（对已执行的 SQL 命令的总结），参数 args 表示 query 中的占位参数。

```go
func (db *DB) Exec(query string, args ...interface{}) (Result, error)
```

以更新命令为例，将表中 username 字段为 luffy 的记录中的 department 字段更新为 pirate：

```go
department := "pirate"
username := "luffy"
_, err = db.Exec("update userinfo set department = ? where username = ?",department, username)
```

运行程序后可以发现 userinfo 表中的字段发生了改变：

![20210108031040](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20210108031040.png)

### 模板

如果我们需要对一个 SQL 语句进行重复调用，例如在不同的 goroutine 中使用相同的 SQL 语句，那么声明一个模板变量是一个明智的抉择：

```go
type Stmt struct {
    // 内含隐藏或非导出字段
}

func (s *Stmt) Exec(args ...interface{}) (Result, error)

func (*Stmt) Query

func (s *Stmt) Close() error
```

一般先调用 sql.DB 的 Prepare() 方法创建一个 Stmt 结构体作为模板：

```go
func (db *DB) Prepare(query string) (*Stmt, error)
```

然后调用 Stmt 的 Exec() 方法传入占位参数，执行模板中的 SQL 语句。以插入命令为例，向表 userinfo 中插入一条新的记录：

```go
username := "joker"
department := "movie"
created := "2020-02-14"
stmt, _ := db.Prepare("insert into userinfo set username=?,department=?,created=?")
defer stmt.Close()
// func (s *Stmt) Exec(args ...interface{}) (Result, error)
stmt.Exec(username, department, created)
```

运行程序后可以发现 userinfo 表中插入的新记录：

![20210108025449](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/20210108025449.png)
