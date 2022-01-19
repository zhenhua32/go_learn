<!-- TOC -->

- [简介](#简介)
- [模型交互](#模型交互)
  - [AutoMigrate](#automigrate)
  - [createTable](#createtable)
- [callbacks](#callbacks)
  - [实际注册流程](#实际注册流程)
  - [createCallback](#createcallback)
- [总结](#总结)

<!-- /TOC -->

## 简介

GORM 源码解读, 基于 [v1.9.11](https://github.com/jinzhu/gorm/tree/v1.9.11) 版本.

## 模型交互

前面已经研究过模型是如何定义并被解析的了, 这次看一下模型是如何和数据库交互的.

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

### AutoMigrate

当定义好模型之后, 第一步是使用 `AutoMigrate` 合并模型:

```go
db.AutoMigrate(&Product{})
```

看一下它的源码:

```go
// AutoMigrate run auto migration for given models, will only add missing fields, won't delete/change current data
func (s *DB) AutoMigrate(values ...interface{}) *DB {
	db := s.Unscoped()
	for _, value := range values {
		db = db.NewScope(value).autoMigrate().db
	}
	return db
}
```

内部是对每个传递的参数调用了 `db.NewScope(value).autoMigrate()`.

那具体是如何合并的呢?

```go
func (scope *Scope) autoMigrate() *Scope {
	tableName := scope.TableName()
	quotedTableName := scope.QuotedTableName()

	if !scope.Dialect().HasTable(tableName) {
		scope.createTable()
	} else {
		for _, field := range scope.GetModelStruct().StructFields {
			if !scope.Dialect().HasColumn(tableName, field.DBName) {
				if field.IsNormal {
					sqlTag := scope.Dialect().DataTypeOf(field)
					scope.Raw(fmt.Sprintf("ALTER TABLE %v ADD %v %v;", quotedTableName, scope.Quote(field.DBName), sqlTag)).Exec()
				}
			}
			scope.createJoinTable(field)
		}
		scope.autoIndex()
	}
	return scope
}
```

中间的 if 部分的代码展示了两条路径. 如果表还没有创建, 直接创建就行了.

否则就需要对模型中的每个字段进行操作, 如果列名不存在, 就需要变更表新增字段了.

```go
scope.Raw(fmt.Sprintf("ALTER TABLE %v ADD %v %v;", quotedTableName, scope.Quote(field.DBName), sqlTag)).Exec()
```

SQL 语句是如何执行的, 先暂时不理会, 但从代码的形式上看算是挺简洁的, 直接使用 Raw 构造语句, Exec 执行.

同时, 对于模型中的每个字段, 还要更新一遍连接表, `scope.createJoinTable(field)`.

在 for 循环处理完模型中的所有字段后, 再更新一遍索引, `scope.autoIndex()`.

总结起来, 自动合并主要做了这么几件事: 创建表, 添加新增的字段, 更新表的关系, 更新索引.

### createTable

前面省略了创建表的具体过程, 来仔细看看表是如何创建的.

```go
func (scope *Scope) createTable() *Scope {
	var tags []string
	var primaryKeys []string
	var primaryKeyInColumnType = false
	for _, field := range scope.GetModelStruct().StructFields {
		if field.IsNormal {
			sqlTag := scope.Dialect().DataTypeOf(field)

			// Check if the primary key constraint was specified as
			// part of the column type. If so, we can only support
			// one column as the primary key.
			if strings.Contains(strings.ToLower(sqlTag), "primary key") {
				primaryKeyInColumnType = true
			}

			tags = append(tags, scope.Quote(field.DBName)+" "+sqlTag)
		}

		if field.IsPrimaryKey {
			primaryKeys = append(primaryKeys, scope.Quote(field.DBName))
		}
		scope.createJoinTable(field)
	}

	var primaryKeyStr string
	if len(primaryKeys) > 0 && !primaryKeyInColumnType {
		primaryKeyStr = fmt.Sprintf(", PRIMARY KEY (%v)", strings.Join(primaryKeys, ","))
	}

	scope.Raw(fmt.Sprintf("CREATE TABLE %v (%v %v)%s", scope.QuotedTableName(), strings.Join(tags, ","), primaryKeyStr, scope.getTableOptions())).Exec()

	scope.autoIndex()
	return scope
}
```

这就是构建 SQL 创建表的过程, 主要的过程是这行代码:

```go
scope.Raw(fmt.Sprintf("CREATE TABLE %v (%v %v)%s", scope.QuotedTableName(), strings.Join(tags, ","), primaryKeyStr, scope.getTableOptions())).Exec()
```

前面的过程主要是遍历模型的字段, 获取每个字段的 `sqlTag`, 并加入 tags 中:

```go
tags = append(tags, scope.Quote(field.DBName)+" "+sqlTag)
```

带有双引号的列名加上空格加上 `sqlTag`.

这个过程中还涉及到了主键的判断, 不过感觉这部分有点坑, 因为
`sqlTag := scope.Dialect().DataTypeOf(field)` 的实现取决于每种数据库对 `DataTypeOf` 的具体实现.

[issues 2270](https://github.com/jinzhu/gorm/issues/2270) 显示出现多个 `primary key`,
使用的是如下的模型定义, 数据库使用了 sqlite3:

```go
type Permission struct {
	ID   int64  `gorm:"AUTO_INCREMENT;column:id;primary_key"`
	Name string `gorm:"column:name;type:varchar;unique;not null"`
	Idx  int64  `gorm:"AUTO_INCREMENT"`
}
```

虽然这个模型定义中只指定了一个 `primary_key`, 但结果 `Idx` 也变成了 `primary_key`:

```go
[2019-01-19 19:40:30]  table "permission" has more than one primary key

[2019-01-19 19:40:30]  [0.14ms]  CREATE TABLE "permission" ("id" integer primary key autoincrement,"name" varchar NOT NULL UNIQUE,"idx" integer primary key autoincrement )
[0 rows affected or returned ]
```

原因只有一个, 它使用了 `AUTO_INCREMENT` 选项, 而在 sqlite3 的 `DataTypeOf` 实现中:

```go
case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Uint, reflect.Uint8, reflect.Uint16, reflect.Uint32, reflect.Uintptr:
  if s.fieldCanAutoIncrement(field) {
    field.TagSettingsSet("AUTO_INCREMENT", "AUTO_INCREMENT")
    sqlType = "integer primary key autoincrement"
  } else {
    sqlType = "integer"
  }
case reflect.Int64, reflect.Uint64:
  if s.fieldCanAutoIncrement(field) {
    field.TagSettingsSet("AUTO_INCREMENT", "AUTO_INCREMENT")
    sqlType = "integer primary key autoincrement"
  } else {
    sqlType = "bigint"
  }
```

`AUTO_INCREMENT` 选项导致了返回的结果中存在 `primary key`.

我怀疑这是个 bug. 因为在后续有对是否是主键的判断, 并添加 `primaryKeyStr`.

```go
if field.IsPrimaryKey {
  primaryKeys = append(primaryKeys, scope.Quote(field.DBName))
}
```

```go
var primaryKeyStr string
if len(primaryKeys) > 0 && !primaryKeyInColumnType {
  primaryKeyStr = fmt.Sprintf(", PRIMARY KEY (%v)", strings.Join(primaryKeys, ","))
}
```

我觉得 `sqlType` 不应该返回关于 `primary key` 的信息.
要设置主键, 可以在后面的 `primaryKeyStr` 中进行.

好了, 对于主键的讨论就此告一段落了.

合并表和创建表的过程中都有 `createJoinTable`, 但因为关系实现还没有深入研究, 先忽略吧.

## callbacks

增删改查都和 DB 结构体中的 `callbacks` 有关:

```go
// DB contains information for current db connection
type DB struct {
  ...
	// global db
	parent        *DB
	callbacks     *Callback
	dialect       Dialect
	singularTable bool
  ...
}
```

看一下 Create 方法的代码:

```go
// Create insert the value into database
func (s *DB) Create(value interface{}) *DB {
	scope := s.NewScope(value)
	return scope.callCallbacks(s.parent.callbacks.creates).db
}
```

在新的 scope 中调用了 `callCallbacks` 方法, 里面的参数是 `s.parent.callbacks.creates`.
`parent` 的类型也是 `*DB`, 算是继承.

继续挖掘 `callCallbacks`:

```go
func (scope *Scope) callCallbacks(funcs []*func(s *Scope)) *Scope {
	defer func() {
		if err := recover(); err != nil {
			if db, ok := scope.db.db.(sqlTx); ok {
				db.Rollback()
			}
			panic(err)
		}
	}()
	for _, f := range funcs {
		(*f)(scope)
		if scope.skipLeft {
			break
		}
	}
	return scope
}
```

使用了 defer 下的 recover 模式, 以前介绍过这个模式, 不再深入.

`callCallbacks` 的参数其实是个函数的切片, 然后依次调用所有的函数, 除非 `scope.skipLeft` 为 true.

看过了调用的方式, 让我们来看看 `Callback` 到底是什么.

```go
// Callback is a struct that contains all CRUD callbacks
//   Field `creates` contains callbacks will be call when creating object
//   Field `updates` contains callbacks will be call when updating object
//   Field `deletes` contains callbacks will be call when deleting object
//   Field `queries` contains callbacks will be call when querying object with query methods like Find, First, Related, Association...
//   Field `rowQueries` contains callbacks will be call when querying object with Row, Rows...
//   Field `processors` contains all callback processors, will be used to generate above callbacks in order
type Callback struct {
	logger     logger
	creates    []*func(scope *Scope)
	updates    []*func(scope *Scope)
	deletes    []*func(scope *Scope)
	queries    []*func(scope *Scope)
	rowQueries []*func(scope *Scope)
	processors []*CallbackProcessor
}
```

`Callback` 里包含了很多的函数切片, 用于增删改查. 注释已经解释的很清楚了.

关注一下 `CallbackProcessor`, 这是用于按序生成所有 callbacks 的.

```go
// CallbackProcessor contains callback informations
type CallbackProcessor struct {
	logger    logger
	name      string              // current callback's name
	before    string              // register current callback before a callback
	after     string              // register current callback after a callback
	replace   bool                // replace callbacks with same name
	remove    bool                // delete callbacks with same name
	kind      string              // callback type: create, update, delete, query, row_query
	processor *func(scope *Scope) // callback handler
	parent    *Callback
}
```

```go
// Create could be used to register callbacks for creating object
//     db.Callback().Create().After("gorm:create").Register("plugin:run_after_create", func(*Scope) {
//       // business logic
//       ...
//
//       // set error if some thing wrong happened, will rollback the creating
//       scope.Err(errors.New("error"))
//     })
func (c *Callback) Create() *CallbackProcessor {
	return &CallbackProcessor{logger: c.logger, kind: "create", parent: c}
}

// Update could be used to register callbacks for updating object, refer `Create` for usage
func (c *Callback) Update() *CallbackProcessor {
	return &CallbackProcessor{logger: c.logger, kind: "update", parent: c}
}

// Delete could be used to register callbacks for deleting object, refer `Create` for usage
func (c *Callback) Delete() *CallbackProcessor {
	return &CallbackProcessor{logger: c.logger, kind: "delete", parent: c}
}

// Query could be used to register callbacks for querying objects with query methods like `Find`, `First`, `Related`, `Association`...
// Refer `Create` for usage
func (c *Callback) Query() *CallbackProcessor {
	return &CallbackProcessor{logger: c.logger, kind: "query", parent: c}
}

// RowQuery could be used to register callbacks for querying objects with `Row`, `Rows`, refer `Create` for usage
func (c *Callback) RowQuery() *CallbackProcessor {
	return &CallbackProcessor{logger: c.logger, kind: "row_query", parent: c}
}
```

`Callback` 有各种方法来创建不同类型的 `CallbackProcessor`.

```go
// After insert a new callback after callback `callbackName`, refer `Callbacks.Create`
func (cp *CallbackProcessor) After(callbackName string) *CallbackProcessor {
	cp.after = callbackName
	return cp
}

// Before insert a new callback before callback `callbackName`, refer `Callbacks.Create`
func (cp *CallbackProcessor) Before(callbackName string) *CallbackProcessor {
	cp.before = callbackName
	return cp
}
```

`After` 和 `Before` 更新了 `CallbackProcessor` 上特定的属性, 用于后续计算 callback 调用顺序.

```go
db.Callback().Create().After("gorm:create").Register("plugin:run_after_create", func(*Scope) {
  // business logic
  ...

  // set error if some thing wrong happened, will rollback the creating
  scope.Err(errors.New("error"))
})
```

注释上的例子是这样的, 继续看 `Register` 方法.

```go
// Register a new callback, refer `Callbacks.Create`
func (cp *CallbackProcessor) Register(callbackName string, callback func(scope *Scope)) {
	if cp.kind == "row_query" {
		if cp.before == "" && cp.after == "" && callbackName != "gorm:row_query" {
			cp.logger.Print(fmt.Sprintf("Registering RowQuery callback %v without specify order with Before(), After(), applying Before('gorm:row_query') by default for compatibility...\n", callbackName))
			cp.before = "gorm:row_query"
		}
	}

	cp.name = callbackName
	cp.processor = &callback
	cp.parent.processors = append(cp.parent.processors, cp)
	cp.parent.reorder()
}
```

主要是设置了 cp 的 `processor` 属性, 并将该 cp 添加到了 `cp.parent.processors` 中.
然后调用 `cp.parent.reorder()` 进行了重新排序.

有注册方法, 当然也有对应的删除方法:

```go
// Remove a registered callback
//     db.Callback().Create().Remove("gorm:update_time_stamp_when_create")
func (cp *CallbackProcessor) Remove(callbackName string) {
	cp.logger.Print(fmt.Sprintf("[info] removing callback `%v` from %v\n", callbackName, fileWithLineNum()))
	cp.name = callbackName
	cp.remove = true
	cp.parent.processors = append(cp.parent.processors, cp)
	cp.parent.reorder()
}
```

设置 `remove` 属性为 true, 然后重新排序.

替换的方法也是类似:

```go
// Replace a registered callback with new callback
//     db.Callback().Create().Replace("gorm:update_time_stamp_when_create", func(*Scope) {
//		   scope.SetColumn("Created", now)
//		   scope.SetColumn("Updated", now)
//     })
func (cp *CallbackProcessor) Replace(callbackName string, callback func(scope *Scope)) {
	cp.logger.Print(fmt.Sprintf("[info] replacing callback `%v` from %v\n", callbackName, fileWithLineNum()))
	cp.name = callbackName
	cp.processor = &callback
	cp.replace = true
	cp.parent.processors = append(cp.parent.processors, cp)
	cp.parent.reorder()
}
```

还是看一下重新排序是如何进行的吧:

```go
// reorder all registered processors, and reset CRUD callbacks
func (c *Callback) reorder() {
	var creates, updates, deletes, queries, rowQueries []*CallbackProcessor

	for _, processor := range c.processors {
		if processor.name != "" {
			switch processor.kind {
			case "create":
				creates = append(creates, processor)
			case "update":
				updates = append(updates, processor)
			case "delete":
				deletes = append(deletes, processor)
			case "query":
				queries = append(queries, processor)
			case "row_query":
				rowQueries = append(rowQueries, processor)
			}
		}
	}

	c.creates = sortProcessors(creates)
	c.updates = sortProcessors(updates)
	c.deletes = sortProcessors(deletes)
	c.queries = sortProcessors(queries)
	c.rowQueries = sortProcessors(rowQueries)
}
```

上半部分只是分别归类, 具体还是要看 `sortProcessors`:

```go
// sortProcessors sort callback processors based on its before, after, remove, replace
func sortProcessors(cps []*CallbackProcessor) []*func(scope *Scope) {
	var (
		allNames, sortedNames []string
		sortCallbackProcessor func(c *CallbackProcessor)
	)

	for _, cp := range cps {
		// show warning message the callback name already exists
		if index := getRIndex(allNames, cp.name); index > -1 && !cp.replace && !cp.remove {
			cp.logger.Print(fmt.Sprintf("[warning] duplicated callback `%v` from %v\n", cp.name, fileWithLineNum()))
		}
		allNames = append(allNames, cp.name)
	}

	sortCallbackProcessor = func(c *CallbackProcessor) {
		if getRIndex(sortedNames, c.name) == -1 { // if not sorted
			if c.before != "" { // if defined before callback
				if index := getRIndex(sortedNames, c.before); index != -1 {
					// if before callback already sorted, append current callback just after it
					sortedNames = append(sortedNames[:index], append([]string{c.name}, sortedNames[index:]...)...)
				} else if index := getRIndex(allNames, c.before); index != -1 {
					// if before callback exists but haven't sorted, append current callback to last
					sortedNames = append(sortedNames, c.name)
					sortCallbackProcessor(cps[index])
				}
			}

			if c.after != "" { // if defined after callback
				if index := getRIndex(sortedNames, c.after); index != -1 {
					// if after callback already sorted, append current callback just before it
					sortedNames = append(sortedNames[:index+1], append([]string{c.name}, sortedNames[index+1:]...)...)
				} else if index := getRIndex(allNames, c.after); index != -1 {
					// if after callback exists but haven't sorted
					cp := cps[index]
					// set after callback's before callback to current callback
					if cp.before == "" {
						cp.before = c.name
					}
					sortCallbackProcessor(cp)
				}
			}

			// if current callback haven't been sorted, append it to last
			if getRIndex(sortedNames, c.name) == -1 {
				sortedNames = append(sortedNames, c.name)
			}
		}
	}

	for _, cp := range cps {
		sortCallbackProcessor(cp)
	}

	var sortedFuncs []*func(scope *Scope)
	for _, name := range sortedNames {
		if index := getRIndex(allNames, name); !cps[index].remove {
			sortedFuncs = append(sortedFuncs, cps[index].processor)
		}
	}

	return sortedFuncs
}
```

首先获取了所有 cp 的名字, 同时提示是否发现了重复. `sortedNames` 里保存排序好的名字.

```go
// getRIndex get right index from string slice
func getRIndex(strs []string, str string) int {
	for i := len(strs) - 1; i >= 0; i-- {
		if strs[i] == str {
			return i
		}
	}
	return -1
}
```

`getRIndex` 获取最右边的索引.

看一下 `sortCallbackProcessor` 函数到底在做什么.

里面有两个判断部分, 先看第一个部分:

```go
if c.before != "" { // if defined before callback
  if index := getRIndex(sortedNames, c.before); index != -1 {
    // if before callback already sorted, append current callback just after it
    sortedNames = append(sortedNames[:index], append([]string{c.name}, sortedNames[index:]...)...)
  } else if index := getRIndex(allNames, c.before); index != -1 {
    // if before callback exists but haven't sorted, append current callback to last
    sortedNames = append(sortedNames, c.name)
    sortCallbackProcessor(cps[index])
  }
}
```

分为两种情况, 如果 before callback 已经排序好了, 直接插在它的后面就行.

如果 before callback 确实存在, 但还没有被排序, 就将当前名字直接放在 `sortedNames` 的最后.
然后递归调用 `sortCallbackProcessor(cps[index])`, 这就是直接进入到 before callback 的排序中了.

再看第二个部分:

```go
if c.after != "" { // if defined after callback
  if index := getRIndex(sortedNames, c.after); index != -1 {
    // if after callback already sorted, append current callback just before it
    sortedNames = append(sortedNames[:index+1], append([]string{c.name}, sortedNames[index+1:]...)...)
  } else if index := getRIndex(allNames, c.after); index != -1 {
    // if after callback exists but haven't sorted
    cp := cps[index]
    // set after callback's before callback to current callback
    if cp.before == "" {
      cp.before = c.name
    }
    sortCallbackProcessor(cp)
  }
}
```

其实和前面的逻辑差不多, 如果 after callback 已经排序好了, 直接插在它的前面就行.

如果 after callback 确实存在, 会修改 after callback 的 before 属性, 设置为当前 callback.
然后递归调用 `sortCallbackProcessor(cp)`, 进入到 after callback 的排序中.

```go
// if current callback haven't been sorted, append it to last
if getRIndex(sortedNames, c.name) == -1 {
  sortedNames = append(sortedNames, c.name)
}
```

还没保存就直接放到最后. `sortCallbackProcessor` 的内容就是这样.

```go
for _, cp := range cps {
  sortCallbackProcessor(cp)
}
```

开始排序. 等排序完了之后, `sortedNames` 就完成了:

```go
var sortedFuncs []*func(scope *Scope)
for _, name := range sortedNames {
  if index := getRIndex(allNames, name); !cps[index].remove {
    sortedFuncs = append(sortedFuncs, cps[index].processor)
  }
}

return sortedFuncs
```

将那些不是 `remove` 状态的 callback, 依次添加到 `sortedFuncs` 中.

最后还有一个 Get 方法用于获取注册的回调:

```go
// Get registered callback
//    db.Callback().Create().Get("gorm:create")
func (cp *CallbackProcessor) Get(callbackName string) (callback func(scope *Scope)) {
	for _, p := range cp.parent.processors {
		if p.name == callbackName && p.kind == cp.kind {
			if p.remove {
				callback = nil
			} else {
				callback = *p.processor
			}
		}
	}
	return
}
```

现在, 我们应该已经清楚了回调函数是如何注册并排序的了, 以及如何按名称获取单个回调函数.

### 实际注册流程

前面只是讲解了理论上的定义, 看一下实际上是在哪里注册的.

DB 在初始化的时候, 即 `Open` 方法调用了如下的语句:

```go
db = &DB{
  db:        dbSQL,
  logger:    defaultLogger,
  callbacks: DefaultCallback,
  dialect:   newDialect(dialect, dbSQL),
}
```

这个 `DefaultCallback` 的定义如下:

```go
// DefaultCallback default callbacks defined by gorm
var DefaultCallback = &Callback{}
```

一开始我也是有点慌, 这只是个空定义, 肯定有地方初始化的. 扫了一眼目录就明白了.

在 `callback_create.go` 文件下定义了 create 方面的注册流程.

```go
// Define callbacks for creating
func init() {
	DefaultCallback.Create().Register("gorm:begin_transaction", beginTransactionCallback)
	DefaultCallback.Create().Register("gorm:before_create", beforeCreateCallback)
	DefaultCallback.Create().Register("gorm:save_before_associations", saveBeforeAssociationsCallback)
	DefaultCallback.Create().Register("gorm:update_time_stamp", updateTimeStampForCreateCallback)
	DefaultCallback.Create().Register("gorm:create", createCallback)
	DefaultCallback.Create().Register("gorm:force_reload_after_create", forceReloadAfterCreateCallback)
	DefaultCallback.Create().Register("gorm:save_after_associations", saveAfterAssociationsCallback)
	DefaultCallback.Create().Register("gorm:after_create", afterCreateCallback)
	DefaultCallback.Create().Register("gorm:commit_or_rollback_transaction", commitOrRollbackTransactionCallback)
}
```

结合[文档](https://gorm.io/zh_CN/docs/hooks.html),
看一下 `BeforeSave` 和 `BeforeCreate` 是如何实现的.

当你定义一个模型时, 可以在这个模型上实现 `BeforeSave` 和 `BeforeCreate` 之类的方法,
这些方法会在恰当的时候被调用.

```go
func (u *User) BeforeSave() (err error) {
  if !u.IsValid() {
    err = errors.New("can't save invalid data")
  }
  return
}

func (u *User) AfterCreate(scope *gorm.Scope) (err error) {
  if u.ID == 1 {
    scope.DB().Model(u).Update("role", "admin")
  }
  return
}
```

上面是官方文档上的例子. 在前面我们在注释中看到了如何手动注册一个回调函数,
类似于 `DefaultCallback.Create().Register("gorm:begin_transaction", beginTransactionCallback)`,
但如何实现调用模型上定义的方法呢?

看一下 `beforeCreateCallback` 函数:

```go
// beforeCreateCallback will invoke `BeforeSave`, `BeforeCreate` method before creating
func beforeCreateCallback(scope *Scope) {
	if !scope.HasError() {
		scope.CallMethod("BeforeSave")
	}
	if !scope.HasError() {
		scope.CallMethod("BeforeCreate")
	}
}
```

原来是通过 `scope.CallMethod` 方法实现的, 传递特定的方法名称就能调用该方法了.

```go
// CallMethod call scope value's method, if it is a slice, will call its element's method one by one
func (scope *Scope) CallMethod(methodName string) {
	if scope.Value == nil {
		return
	}

	if indirectScopeValue := scope.IndirectValue(); indirectScopeValue.Kind() == reflect.Slice {
		for i := 0; i < indirectScopeValue.Len(); i++ {
			scope.callMethod(methodName, indirectScopeValue.Index(i))
		}
	} else {
		scope.callMethod(methodName, indirectScopeValue)
	}
}
```

绕了一圈, 继续看 `callMethod` 的代码:

```go
func (scope *Scope) callMethod(methodName string, reflectValue reflect.Value) {
	// Only get address from non-pointer
	if reflectValue.CanAddr() && reflectValue.Kind() != reflect.Ptr {
		reflectValue = reflectValue.Addr()
	}

	if methodValue := reflectValue.MethodByName(methodName); methodValue.IsValid() {
		switch method := methodValue.Interface().(type) {
		case func():
			method()
		case func(*Scope):
			method(scope)
		case func(*DB):
			newDB := scope.NewDB()
			method(newDB)
			scope.Err(newDB.Error)
		case func() error:
			scope.Err(method())
		case func(*Scope) error:
			scope.Err(method(scope))
		case func(*DB) error:
			newDB := scope.NewDB()
			scope.Err(method(newDB))
			scope.Err(newDB.Error)
		default:
			scope.Err(fmt.Errorf("unsupported function %v", methodName))
		}
	}
}
```

这些灵活的方式都是靠反射实现的, 关键代码是 `methodValue := reflectValue.MethodByName(methodName)`.

从 `switch` 可以看到, 方法可以有不同的签名:

```go
switch method := methodValue.Interface().(type) {
case func():
  method()
case func(*Scope):
  method(scope)
case func(*DB):
  newDB := scope.NewDB()
  method(newDB)
  scope.Err(newDB.Error)
case func() error:
  scope.Err(method())
case func(*Scope) error:
  scope.Err(method(scope))
case func(*DB) error:
  newDB := scope.NewDB()
  scope.Err(method(newDB))
  scope.Err(newDB.Error)
default:
  scope.Err(fmt.Errorf("unsupported function %v", methodName))
}
```

所以, 实际上这都可以看作是 `reflect` 的大型示范使用例子.

### createCallback

其他的钩子函数不看了, 具体看一下当插入单条数据时都在干什么:

```go
// createCallback the callback used to insert data into database
func createCallback(scope *Scope) {
	if !scope.HasError() {
		defer scope.trace(scope.db.nowFunc())

		var (
			columns, placeholders        []string
			blankColumnsWithDefaultValue []string
		)

		for _, field := range scope.Fields() {
			if scope.changeableField(field) {
				if field.IsNormal && !field.IsIgnored {
					if field.IsBlank && field.HasDefaultValue {
						blankColumnsWithDefaultValue = append(blankColumnsWithDefaultValue, scope.Quote(field.DBName))
						scope.InstanceSet("gorm:blank_columns_with_default_value", blankColumnsWithDefaultValue)
					} else if !field.IsPrimaryKey || !field.IsBlank {
						columns = append(columns, scope.Quote(field.DBName))
						placeholders = append(placeholders, scope.AddToVars(field.Field.Interface()))
					}
				} else if field.Relationship != nil && field.Relationship.Kind == "belongs_to" {
					for _, foreignKey := range field.Relationship.ForeignDBNames {
						if foreignField, ok := scope.FieldByName(foreignKey); ok && !scope.changeableField(foreignField) {
							columns = append(columns, scope.Quote(foreignField.DBName))
							placeholders = append(placeholders, scope.AddToVars(foreignField.Field.Interface()))
						}
					}
				}
			}
		}

		var (
			returningColumn = "*"
			quotedTableName = scope.QuotedTableName()
			primaryField    = scope.PrimaryField()
			extraOption     string
			insertModifier  string
		)

		if str, ok := scope.Get("gorm:insert_option"); ok {
			extraOption = fmt.Sprint(str)
		}
		if str, ok := scope.Get("gorm:insert_modifier"); ok {
			insertModifier = strings.ToUpper(fmt.Sprint(str))
			if insertModifier == "INTO" {
				insertModifier = ""
			}
		}

		if primaryField != nil {
			returningColumn = scope.Quote(primaryField.DBName)
		}

		lastInsertIDReturningSuffix := scope.Dialect().LastInsertIDReturningSuffix(quotedTableName, returningColumn)

		if len(columns) == 0 {
			scope.Raw(fmt.Sprintf(
				"INSERT %v INTO %v %v%v%v",
				addExtraSpaceIfExist(insertModifier),
				quotedTableName,
				scope.Dialect().DefaultValueStr(),
				addExtraSpaceIfExist(extraOption),
				addExtraSpaceIfExist(lastInsertIDReturningSuffix),
			))
		} else {
			scope.Raw(fmt.Sprintf(
				"INSERT %v INTO %v (%v) VALUES (%v)%v%v",
				addExtraSpaceIfExist(insertModifier),
				scope.QuotedTableName(),
				strings.Join(columns, ","),
				strings.Join(placeholders, ","),
				addExtraSpaceIfExist(extraOption),
				addExtraSpaceIfExist(lastInsertIDReturningSuffix),
			))
		}

		// execute create sql
		if lastInsertIDReturningSuffix == "" || primaryField == nil {
			if result, err := scope.SQLDB().Exec(scope.SQL, scope.SQLVars...); scope.Err(err) == nil {
				// set rows affected count
				scope.db.RowsAffected, _ = result.RowsAffected()

				// set primary value to primary field
				if primaryField != nil && primaryField.IsBlank {
					if primaryValue, err := result.LastInsertId(); scope.Err(err) == nil {
						scope.Err(primaryField.Set(primaryValue))
					}
				}
			}
		} else {
			if primaryField.Field.CanAddr() {
				if err := scope.SQLDB().QueryRow(scope.SQL, scope.SQLVars...).Scan(primaryField.Field.Addr().Interface()); scope.Err(err) == nil {
					primaryField.IsBlank = false
					scope.db.RowsAffected = 1
				}
			} else {
				scope.Err(ErrUnaddressable)
			}
		}
	}
}
```

首先， 内部的第一个 for 循环遍历了所有的字段， 并更新了开头定义的三个切片.

```go
for _, field := range scope.Fields() {
  if scope.changeableField(field) {
    if field.IsNormal && !field.IsIgnored {
      if field.IsBlank && field.HasDefaultValue {
        blankColumnsWithDefaultValue = append(blankColumnsWithDefaultValue, scope.Quote(field.DBName))
        scope.InstanceSet("gorm:blank_columns_with_default_value", blankColumnsWithDefaultValue)
      } else if !field.IsPrimaryKey || !field.IsBlank {
        columns = append(columns, scope.Quote(field.DBName))
        placeholders = append(placeholders, scope.AddToVars(field.Field.Interface()))
      }
    } else if field.Relationship != nil && field.Relationship.Kind == "belongs_to" {
      for _, foreignKey := range field.Relationship.ForeignDBNames {
        if foreignField, ok := scope.FieldByName(foreignKey); ok && !scope.changeableField(foreignField) {
          columns = append(columns, scope.Quote(foreignField.DBName))
          placeholders = append(placeholders, scope.AddToVars(foreignField.Field.Interface()))
        }
      }
    }
  }
}
```

然后就是获取并设置一些信息:

```go
var (
  returningColumn = "*"
  quotedTableName = scope.QuotedTableName()
  primaryField    = scope.PrimaryField()
  extraOption     string
  insertModifier  string
)
```

等信息都获取完了, 就开始构造插入语句了:

```go
if len(columns) == 0 {
  scope.Raw(fmt.Sprintf(
    "INSERT %v INTO %v %v%v%v",
    addExtraSpaceIfExist(insertModifier),
    quotedTableName,
    scope.Dialect().DefaultValueStr(),
    addExtraSpaceIfExist(extraOption),
    addExtraSpaceIfExist(lastInsertIDReturningSuffix),
  ))
} else {
  scope.Raw(fmt.Sprintf(
    "INSERT %v INTO %v (%v) VALUES (%v)%v%v",
    addExtraSpaceIfExist(insertModifier),
    scope.QuotedTableName(),
    strings.Join(columns, ","),
    strings.Join(placeholders, ","),
    addExtraSpaceIfExist(extraOption),
    addExtraSpaceIfExist(lastInsertIDReturningSuffix),
  ))
}
```

最后执行 sql 语句:

```go
// execute create sql
if lastInsertIDReturningSuffix == "" || primaryField == nil {
  if result, err := scope.SQLDB().Exec(scope.SQL, scope.SQLVars...); scope.Err(err) == nil {
    // set rows affected count
    scope.db.RowsAffected, _ = result.RowsAffected()

    // set primary value to primary field
    if primaryField != nil && primaryField.IsBlank {
      if primaryValue, err := result.LastInsertId(); scope.Err(err) == nil {
        scope.Err(primaryField.Set(primaryValue))
      }
    }
  }
} else {
  if primaryField.Field.CanAddr() {
    if err := scope.SQLDB().QueryRow(scope.SQL, scope.SQLVars...).Scan(primaryField.Field.Addr().Interface()); scope.Err(err) == nil {
      primaryField.IsBlank = false
      scope.db.RowsAffected = 1
    }
  } else {
    scope.Err(ErrUnaddressable)
  }
}
```

这里的第一个判断条件是和 `lastInsertIDReturningSuffix` 有关的, 只有 PostgreSQL 会返回非空的字符串.

```go
var userid int
err := db.QueryRow(`INSERT INTO users(name, favorite_fruit, age)
	VALUES('beatrice', 'starfruit', 93) RETURNING id`).Scan(&userid)
```

PostgreSQL 中不支持 `LastInsertId()` 方法, 要获取 ID 需要像上面这样调用.
参考 [PostgreSQL Queries](https://godoc.org/github.com/lib/pq#hdr-Queries).

所以执行方式有所不同.

这样, `createCallback` 回调就看完了, 插入数据的过程也知道了.

## 总结

在这一部分里, 主要看了数据表是如何创建和合并的, 以及钩子函数是如何注册并排序的, 以及何时被调用的.
