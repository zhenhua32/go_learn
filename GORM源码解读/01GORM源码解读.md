<!-- TOC -->

- [简介](#简介)
- [起步](#起步)
- [数据库连接](#数据库连接)
- [gorm.DB](#gormdb)
- [事务实现](#事务实现)
- [总结](#总结)

<!-- /TOC -->

## 简介

GORM 源码解读, 基于 [v1.9.11](https://github.com/jinzhu/gorm/tree/v1.9.11) 版本.

## 起步

官方文档上入门的例子如下:

```go
package main

import (
  "github.com/jinzhu/gorm"
  _ "github.com/jinzhu/gorm/dialects/sqlite"
)

type Product struct {
  gorm.Model
  Code string
  Price uint
}

func main() {
  db, err := gorm.Open("sqlite3", "test.db")
  if err != nil {
    panic("failed to connect database")
  }
  defer db.Close()

  // Migrate the schema
  db.AutoMigrate(&Product{})

  // 创建
  db.Create(&Product{Code: "L1212", Price: 1000})

  // 读取
  var product Product
  db.First(&product, 1) // 查询id为1的product
  db.First(&product, "code = ?", "L1212") // 查询code为l1212的product

  // 更新 - 更新product的price为2000
  db.Model(&product).Update("Price", 2000)

  // 删除 - 删除product
  db.Delete(&product)
}
```

## 数据库连接

从 `gorm.Open` 开始看起吧, 看数据库是怎么连接的:

```go
// Open initialize a new db connection, need to import driver first, e.g:
//
//     import _ "github.com/go-sql-driver/mysql"
//     func main() {
//       db, err := gorm.Open("mysql", "user:password@/dbname?charset=utf8&parseTime=True&loc=Local")
//     }
// GORM has wrapped some drivers, for easier to remember driver's import path, so you could import the mysql driver with
//    import _ "github.com/jinzhu/gorm/dialects/mysql"
//    // import _ "github.com/jinzhu/gorm/dialects/postgres"
//    // import _ "github.com/jinzhu/gorm/dialects/sqlite"
//    // import _ "github.com/jinzhu/gorm/dialects/mssql"
func Open(dialect string, args ...interface{}) (db *DB, err error) {
	if len(args) == 0 {
		err = errors.New("invalid database source")
		return nil, err
	}
	var source string
	var dbSQL SQLCommon
	var ownDbSQL bool

	switch value := args[0].(type) {
	case string:
		var driver = dialect
		if len(args) == 1 {
			source = value
		} else if len(args) >= 2 {
			driver = value
			source = args[1].(string)
		}
		dbSQL, err = sql.Open(driver, source)
		ownDbSQL = true
	case SQLCommon:
		dbSQL = value
		ownDbSQL = false
	default:
		return nil, fmt.Errorf("invalid database source: %v is not a valid type", value)
	}

	db = &DB{
		db:        dbSQL,
		logger:    defaultLogger,
		callbacks: DefaultCallback,
		dialect:   newDialect(dialect, dbSQL),
	}
	db.parent = db
	if err != nil {
		return
	}
	// Send a ping to make sure the database connection is alive.
	if d, ok := dbSQL.(*sql.DB); ok {
		if err = d.Ping(); err != nil && ownDbSQL {
			d.Close()
		}
	}
	return
}
```

`gorm.Open` 有两个参数, 一个是数据库名称, 其余是连接参数.

从 `switch` 语句中, 可以发现如果第一个参数是 string 类型, 实际上是通过 Golang 中的 `sql` 模块连接:

```go
dbSQL, err = sql.Open(driver, source)
```

也可以直接传递一个实现了 `SQLCommon` 接口的实例.

然后初始化了一个 `gorm.DB` 实例, 并在最后执行了一次 ping 请求, 测试数据库连接是否正常.

看一下 `gorm.DB` 结构体:

```go
// DB contains information for current db connection
type DB struct {
	sync.RWMutex
	Value        interface{}
	Error        error
	RowsAffected int64

	// single db
	db                SQLCommon
	blockGlobalUpdate bool
	logMode           logModeValue
	logger            logger
	search            *search
	values            sync.Map

	// global db
	parent        *DB
	callbacks     *Callback
	dialect       Dialect
	singularTable bool

	// function to be used to override the creating of a new timestamp
	nowFuncOverride func() time.Time
}
```

`gorm.DB` 扩展自 `sync.RWMutex` 读写互斥锁.

## gorm.DB

上面已经看过了 `gorm.DB` 结构体的定义了, 从入门的示例代码中可以看出, 所有的操作都是围绕它来进行的,
所以 `gorm.DB` 是核心的结构体. 看下它具体实现了哪些方法.

```go
// New clone a new db connection without search conditions
func (s *DB) New() *DB {
	clone := s.clone()
	clone.search = nil
	clone.Value = nil
	return clone
}

type closer interface {
	Close() error
}

// Close close current db connection.  If database connection is not an io.Closer, returns an error.
func (s *DB) Close() error {
	if db, ok := s.parent.db.(closer); ok {
		return db.Close()
	}
	return errors.New("can't close current db")
}
```

克隆数据库的连接和关闭数据库连接. `New` 方法内部使用到了 `s.clone()`,

```go
func (s *DB) clone() *DB {
	db := &DB{
		db:                s.db,
		parent:            s.parent,
		logger:            s.logger,
		logMode:           s.logMode,
		Value:             s.Value,
		Error:             s.Error,
		blockGlobalUpdate: s.blockGlobalUpdate,
		dialect:           newDialect(s.dialect.GetName(), s.db),
		nowFuncOverride:   s.nowFuncOverride,
	}

	s.values.Range(func(k, v interface{}) bool {
		db.values.Store(k, v)
		return true
	})

	if s.search == nil {
		db.search = &search{limit: -1, offset: -1}
	} else {
		db.search = s.search.clone()
	}

	db.search.db = db
	return db
}
```

略过一些简单的 get/set 方法, 接着看

```go
// NewScope create a scope for current operation
func (s *DB) NewScope(value interface{}) *Scope {
	dbClone := s.clone()
	dbClone.Value = value
	scope := &Scope{db: dbClone, Value: value}
	if s.search != nil {
		scope.Search = s.search.clone()
	} else {
		scope.Search = &search{}
	}
	return scope
}
```

`NewScope` 会为当前的操作创建一个新的 scope (作用域).

```go
// QueryExpr returns the query as expr object
func (s *DB) QueryExpr() *expr {
	scope := s.NewScope(s.Value)
	scope.InstanceSet("skip_bindvar", true)
	scope.prepareQuerySQL()

	return Expr(scope.SQL, scope.SQLVars...)
}

// SubQuery returns the query as sub query
func (s *DB) SubQuery() *expr {
	scope := s.NewScope(s.Value)
	scope.InstanceSet("skip_bindvar", true)
	scope.prepareQuerySQL()

	return Expr(fmt.Sprintf("(%v)", scope.SQL), scope.SQLVars...)
}
```

`QueryExpr` 和 `SubQuery` 都用到了 `NewScope`, 在当前的作用域下获取查询表达式和进行子查询.

接着是很多查询方法, 类似 `Where`,

```go
// Where return a new relation, filter records with given conditions, accepts `map`, `struct` or `string` as conditions, refer http://jinzhu.github.io/gorm/crud.html#query
func (s *DB) Where(query interface{}, args ...interface{}) *DB {
	return s.clone().search.Where(query, args...).db
}
```

跳过这些方法, 等后面探究查询表达式的时候再详细研究.

```go
// Scopes pass current database connection to arguments `func(*DB) *DB`, which could be used to add conditions dynamically
//     func AmountGreaterThan1000(db *gorm.DB) *gorm.DB {
//         return db.Where("amount > ?", 1000)
//     }
//
//     func OrderStatus(status []string) func (db *gorm.DB) *gorm.DB {
//         return func (db *gorm.DB) *gorm.DB {
//             return db.Scopes(AmountGreaterThan1000).Where("status in (?)", status)
//         }
//     }
//
//     db.Scopes(AmountGreaterThan1000, OrderStatus([]string{"paid", "shipped"})).Find(&orders)
// Refer https://jinzhu.github.io/gorm/crud.html#scopes
func (s *DB) Scopes(funcs ...func(*DB) *DB) *DB {
	for _, f := range funcs {
		s = f(s)
	}
	return s
}
```

`Scopes` 是一个钩子函数, 用于动态添加查询条件, 这在函数是一等公民的语言里是一个常见的模式.

## 事务实现

看一下事务是如何实现的:

```go
// Begin begins a transaction
func (s *DB) Begin() *DB {
	return s.BeginTx(context.Background(), &sql.TxOptions{})
}

// BeginTx begins a transaction with options
func (s *DB) BeginTx(ctx context.Context, opts *sql.TxOptions) *DB {
	c := s.clone()
	if db, ok := c.db.(sqlDb); ok && db != nil {
		tx, err := db.BeginTx(ctx, opts)
		c.db = interface{}(tx).(SQLCommon)

		c.dialect.SetDB(c.db)
		c.AddError(err)
	} else {
		c.AddError(ErrCantStartTransaction)
	}
	return c
}
```

这一部分是开始事务时的操作, 实际上是 `c.db` 实现了 `sqlDb` 接口, 调用了 `BeginTx` 方法.

```go
type sqlDb interface {
	Begin() (*sql.Tx, error)
	BeginTx(ctx context.Context, opts *sql.TxOptions) (*sql.Tx, error)
}
```

接着看如何提交事务:

```go
// Commit commit a transaction
func (s *DB) Commit() *DB {
	var emptySQLTx *sql.Tx
	if db, ok := s.db.(sqlTx); ok && db != nil && db != emptySQLTx {
		s.AddError(db.Commit())
	} else {
		s.AddError(ErrInvalidTransaction)
	}
	return s
}
```

和开始事务类似, `s.db` 实现了 `sqlTx` 接口, 调用了 `Commit` 方法.

```go
type sqlTx interface {
	Commit() error
	Rollback() error
}
```

`sqlTx` 接口里还有个 `Rollback` 方法, 所以回滚操作也是类似的:

```go
// Rollback rollback a transaction
func (s *DB) Rollback() *DB {
	var emptySQLTx *sql.Tx
	if db, ok := s.db.(sqlTx); ok && db != nil && db != emptySQLTx {
		if err := db.Rollback(); err != nil && err != sql.ErrTxDone {
			s.AddError(err)
		}
	} else {
		s.AddError(ErrInvalidTransaction)
	}
	return s
}

// RollbackUnlessCommitted rollback a transaction if it has not yet been
// committed.
func (s *DB) RollbackUnlessCommitted() *DB {
	var emptySQLTx *sql.Tx
	if db, ok := s.db.(sqlTx); ok && db != nil && db != emptySQLTx {
		err := db.Rollback()
		// Ignore the error indicating that the transaction has already
		// been committed.
		if err != sql.ErrTxDone {
			s.AddError(err)
		}
	} else {
		s.AddError(ErrInvalidTransaction)
	}
	return s
}
```

`RollbackUnlessCommitted` 和 `Rollback` 的区别在于前者少了一个 `err != nil` 的判断,
**看了半天还是难以理解这有什么差别**.

`RollbackUnlessCommitted` 作者给出的例子如下:

```go
func doTransaction(DB *gorm.DB) error {
  tx := DB.Begin()
  defer tx.RollbackUnlessCommitted()

  u := User{Name: "test"}
  if err != tx.Save(&User).Error; err != nil {
    return err
  }
  return tx.Commit().Error
}
```

相比较而言, 官方文档上事务的例子如下:

```go
func CreateAnimals(db *gorm.DB) error {
  // 请注意，事务一旦开始，你就应该使用 tx 作为数据库句柄
  tx := db.Begin()
  defer func() {
    if r := recover(); r != nil {
      tx.Rollback()
    }
  }()

  if err := tx.Error; err != nil {
    return err
  }

  if err := tx.Create(&Animal{Name: "Giraffe"}).Error; err != nil {
    tx.Rollback()
    return err
  }

  if err := tx.Create(&Animal{Name: "Lion"}).Error; err != nil {
    tx.Rollback()
    return err
  }

  return tx.Commit().Error
}
```

## 总结

暂时就看到这里吧, `gorm.DB` 还有很多方法等待后续发掘.

主要看了数据库的连接过程, 基本是通过 `dbSQL, err = sql.Open(driver, source)` 实现的.

也看了事务部分, 主要是要实现两个接口:

```go
type sqlDb interface {
	Begin() (*sql.Tx, error)
	BeginTx(ctx context.Context, opts *sql.TxOptions) (*sql.Tx, error)
}

type sqlTx interface {
	Commit() error
	Rollback() error
}
```

当然, 对于其中的 `RollbackUnlessCommitted` 和 `Rollback` 有点疑惑, 因为我想不明白到底有什么不同.

既然是 ORM, 模型定义应该是重中之重, 后续将探索 Model 实现.
