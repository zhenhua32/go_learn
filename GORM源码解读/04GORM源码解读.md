<!-- TOC -->

- [简介](#简介)
- [查询](#查询)
  - [查询流程](#查询流程)
  - [构建查询 SQL 语句](#构建查询-sql-语句)
  - [条件语句](#条件语句)
  - [小结](#小结)
- [search 结构体](#search-结构体)
  - [search 的定义](#search-的定义)
  - [search 的方法](#search-的方法)
  - [小结](#小结-1)
- [总结](#总结)

<!-- /TOC -->

## 简介

GORM 源码解读, 基于 [v1.9.11](https://github.com/jinzhu/gorm/tree/v1.9.11) 版本.

## 查询

上一节中, 我们已经探究过了模型是如何定义的, 以及数据表是如何创建的.
这次, 看一下查询是如何实现的.

查询涉及到很大的一块内容, 因为要支持各种类型的方法.
先看一下官方文档中提供的最简单的几个查询方法.

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

以 `First` 方法为例, 看一下它的实现:

```go
// First find first record that match given conditions, order by primary key
func (s *DB) First(out interface{}, where ...interface{}) *DB {
	newScope := s.NewScope(out)
	newScope.Search.Limit(1)

	return newScope.Set("gorm:order_by_primary_key", "ASC").
		inlineCondition(where...).callCallbacks(s.parent.callbacks.queries).db
}
```

`First` 方法从数据库中获取第一条数据, 以 primary key 升序排序.

前面介绍过, 具体的数据库操作实现是依靠 callbacks 的. 这里用到了 `callbacks.queries`.

在默认的 callbacks 中, 注册了三个不同的 query 回调函数.

```go
// Define callbacks for querying
func init() {
	DefaultCallback.Query().Register("gorm:query", queryCallback)
	DefaultCallback.Query().Register("gorm:preload", preloadCallback)
	DefaultCallback.Query().Register("gorm:after_query", afterQueryCallback)
}
```

### 查询流程

先来看一下最主要的 `queryCallback` 函数.

```go
// queryCallback used to query data from database
func queryCallback(scope *Scope) {
	if _, skip := scope.InstanceGet("gorm:skip_query_callback"); skip {
		return
	}

	//we are only preloading relations, dont touch base model
	if _, skip := scope.InstanceGet("gorm:only_preload"); skip {
		return
	}

	defer scope.trace(scope.db.nowFunc())

	var (
		isSlice, isPtr bool
		resultType     reflect.Type
		results        = scope.IndirectValue()
	)

	if orderBy, ok := scope.Get("gorm:order_by_primary_key"); ok {
		if primaryField := scope.PrimaryField(); primaryField != nil {
			scope.Search.Order(fmt.Sprintf("%v.%v %v", scope.QuotedTableName(), scope.Quote(primaryField.DBName), orderBy))
		}
	}

	if value, ok := scope.Get("gorm:query_destination"); ok {
		results = indirect(reflect.ValueOf(value))
	}

	if kind := results.Kind(); kind == reflect.Slice {
		isSlice = true
		resultType = results.Type().Elem()
		results.Set(reflect.MakeSlice(results.Type(), 0, 0))

		if resultType.Kind() == reflect.Ptr {
			isPtr = true
			resultType = resultType.Elem()
		}
	} else if kind != reflect.Struct {
		scope.Err(errors.New("unsupported destination, should be slice or struct"))
		return
	}

	scope.prepareQuerySQL()

	if !scope.HasError() {
		scope.db.RowsAffected = 0
		if str, ok := scope.Get("gorm:query_option"); ok {
			scope.SQL += addExtraSpaceIfExist(fmt.Sprint(str))
		}

		if rows, err := scope.SQLDB().Query(scope.SQL, scope.SQLVars...); scope.Err(err) == nil {
			defer rows.Close()

			columns, _ := rows.Columns()
			for rows.Next() {
				scope.db.RowsAffected++

				elem := results
				if isSlice {
					elem = reflect.New(resultType).Elem()
				}

				scope.scan(rows, columns, scope.New(elem.Addr().Interface()).Fields())

				if isSlice {
					if isPtr {
						results.Set(reflect.Append(results, elem.Addr()))
					} else {
						results.Set(reflect.Append(results, elem))
					}
				}
			}

			if err := rows.Err(); err != nil {
				scope.Err(err)
			} else if scope.db.RowsAffected == 0 && !isSlice {
				scope.Err(ErrRecordNotFound)
			}
		}
	}
}
```

核心的步骤在于 `scope.prepareQuerySQL()` 构建 SQL 语句.
然后通过 `rows, err := scope.SQLDB().Query(scope.SQL, scope.SQLVars...)`, 执行了数据库查询.

那么查询到的结果是如何传递的, 传递给谁呢?

函数的开头定义了 `results = scope.IndirectValue()`, 这就是最终查询结果的归属地.

`results` 只能是结构体或者是结构体的切片.

```go
if kind := results.Kind(); kind == reflect.Slice {
  isSlice = true
  resultType = results.Type().Elem()
  results.Set(reflect.MakeSlice(results.Type(), 0, 0))

  if resultType.Kind() == reflect.Ptr {
    isPtr = true
    resultType = resultType.Elem()
  }
} else if kind != reflect.Struct {
  scope.Err(errors.New("unsupported destination, should be slice or struct"))
  return
}
```

具体如何处理查询到的结果是在下面这部分代码中:

```go
columns, _ := rows.Columns()
for rows.Next() {
  scope.db.RowsAffected++

  elem := results
  if isSlice {
    elem = reflect.New(resultType).Elem()
  }

  scope.scan(rows, columns, scope.New(elem.Addr().Interface()).Fields())

  if isSlice {
    if isPtr {
      results.Set(reflect.Append(results, elem.Addr()))
    } else {
      results.Set(reflect.Append(results, elem))
    }
  }
}
```

这部分代码的核心语句在于 `scope.scan`, 看一下这个方法的定义:

```go
func (scope *Scope) scan(rows *sql.Rows, columns []string, fields []*Field) {
	var (
		ignored            interface{}
		values             = make([]interface{}, len(columns))
		selectFields       []*Field
		selectedColumnsMap = map[string]int{}
		resetFields        = map[int]*Field{}
	)

	for index, column := range columns {
		values[index] = &ignored

		selectFields = fields
		offset := 0
		if idx, ok := selectedColumnsMap[column]; ok {
			offset = idx + 1
			selectFields = selectFields[offset:]
		}

		for fieldIndex, field := range selectFields {
			if field.DBName == column {
				if field.Field.Kind() == reflect.Ptr {
					values[index] = field.Field.Addr().Interface()
				} else {
					reflectValue := reflect.New(reflect.PtrTo(field.Struct.Type))
					reflectValue.Elem().Set(field.Field.Addr())
					values[index] = reflectValue.Interface()
					resetFields[index] = field
				}

				selectedColumnsMap[column] = offset + fieldIndex

				if field.IsNormal {
					break
				}
			}
		}
	}

	scope.Err(rows.Scan(values...))

	for index, field := range resetFields {
		if v := reflect.ValueOf(values[index]).Elem().Elem(); v.IsValid() {
			field.Field.Set(v)
		}
	}
}
```

就和它的名字暗示的那样, 实际上就是调用了 `rows.Scan(values...)`, 将查询到的数据复制到对应的字段中.

由此, 我们就了解了查询时的主要流程了.

前面专注于流程, 略过了构建 SQL 语句的细节, 来仔细看看 `prepareQuerySQL` 方法.

### 构建查询 SQL 语句

```go
func (scope *Scope) prepareQuerySQL() {
	if scope.Search.raw {
		scope.Raw(scope.CombinedConditionSql())
	} else {
		scope.Raw(fmt.Sprintf("SELECT %v FROM %v %v", scope.selectSQL(), scope.QuotedTableName(), scope.CombinedConditionSql()))
	}
	return
}
```

内部分支中都使用到了 `scope.Raw`, 看一下它的实现:

```go
// Raw set raw sql
func (scope *Scope) Raw(sql string) *Scope {
	scope.SQL = strings.Replace(sql, "$$$", "?", -1)
	return scope
}
```

它的作用是将获取到的 sql 语句赋值到 `scope.SQL` 字段上, 其中替换了所有的 `$$$` 为 `?`.

回到 `prepareQuerySQL` 上来, 重要的部分是其实是 `Raw` 的参数.
if 的后半部分更好理解点, 就是构建了 `SELECT` 表达式.

`SELECT` 表达式需要三个变量, 字段名, 表名, 条件.

将每个都看一下吧.

```go
func (scope *Scope) selectSQL() string {
	if len(scope.Search.selects) == 0 {
		if len(scope.Search.joinConditions) > 0 {
			return fmt.Sprintf("%v.*", scope.QuotedTableName())
		}
		return "*"
	}
	return scope.buildSelectQuery(scope.Search.selects)
}

func (scope *Scope) buildSelectQuery(clause map[string]interface{}) (str string) {
	switch value := clause["query"].(type) {
	case string:
		str = value
	case []string:
		str = strings.Join(value, ", ")
	}

	args := clause["args"].([]interface{})
	replacements := []string{}
	for _, arg := range args {
		switch reflect.ValueOf(arg).Kind() {
		case reflect.Slice:
			values := reflect.ValueOf(arg)
			var tempMarks []string
			for i := 0; i < values.Len(); i++ {
				tempMarks = append(tempMarks, scope.AddToVars(values.Index(i).Interface()))
			}
			replacements = append(replacements, strings.Join(tempMarks, ","))
		default:
			if valuer, ok := interface{}(arg).(driver.Valuer); ok {
				arg, _ = valuer.Value()
			}
			replacements = append(replacements, scope.AddToVars(arg))
		}
	}

	buff := bytes.NewBuffer([]byte{})
	i := 0
	for pos, char := range str {
		if str[pos] == '?' {
			buff.WriteString(replacements[i])
			i++
		} else {
			buff.WriteRune(char)
		}
	}

	str = buff.String()

	return
}
```

当 `scope.Search.selects` 为空的时候, 比较简单.
只要根据是否有连表查询, 返回 `table.*` 或 `*`.

`buildSelectQuery` 就是根据 `scope.Search.selects` 构建查询字段名.

前面半部分一看就明白.

```go
switch value := clause["query"].(type) {
case string:
  str = value
case []string:
  str = strings.Join(value, ", ")
}
```

重点是遇到参数时如何处理, 也就是后半段代码.

```go
args := clause["args"].([]interface{})
replacements := []string{}
for _, arg := range args {
  switch reflect.ValueOf(arg).Kind() {
  case reflect.Slice:
    values := reflect.ValueOf(arg)
    var tempMarks []string
    for i := 0; i < values.Len(); i++ {
      tempMarks = append(tempMarks, scope.AddToVars(values.Index(i).Interface()))
    }
    replacements = append(replacements, strings.Join(tempMarks, ","))
  default:
    if valuer, ok := interface{}(arg).(driver.Valuer); ok {
      arg, _ = valuer.Value()
    }
    replacements = append(replacements, scope.AddToVars(arg))
  }
}

buff := bytes.NewBuffer([]byte{})
i := 0
for pos, char := range str {
  if str[pos] == '?' {
    buff.WriteString(replacements[i])
    i++
  } else {
    buff.WriteRune(char)
  }
}
```

主要的过程是遍历 `args := clause["args"].([]interface{})`,
创建了一个 `replacements` 切片. 然后将 `str` 中所有的 `?`,
替换为了对应的字段.

到此, 构建 `SELECT` 字段的过程就结束了.

获取表名的过程相对简单, 直接展示代码吧:

```go
// QuotedTableName return quoted table name
func (scope *Scope) QuotedTableName() (name string) {
	if scope.search != nil && len(scope.Search.tableName) > 0 {
		if strings.Contains(scope.Search.tableName, " ") {
			return scope.Search.tableName
		}
		return scope.Quote(scope.Search.tableName)
	}

	return scope.Quote(scope.TableName())
}
```

### 条件语句

更多的关注点在于如何构建筛选条件, 即 `CombinedConditionSql` 方法.

```go
// CombinedConditionSql return combined condition sql
func (scope *Scope) CombinedConditionSql() string {
	joinSQL := scope.joinsSQL()
	whereSQL := scope.whereSQL()
	if scope.Search.raw {
		whereSQL = strings.TrimSuffix(strings.TrimPrefix(whereSQL, "WHERE ("), ")")
	}
	return joinSQL + whereSQL + scope.groupSQL() +
		scope.havingSQL() + scope.orderSQL() + scope.limitAndOffsetSQL()
}
```

短小的代码中是精简的逻辑, 条件语句有很多模块, 这里总共有 6 个子句.
都看一遍吧, 看完之后应该对如何构建条件语句不会陌生了.

```go
func (scope *Scope) joinsSQL() string {
	var joinConditions []string
	for _, clause := range scope.Search.joinConditions {
		if sql := scope.buildCondition(clause, true); sql != "" {
			joinConditions = append(joinConditions, strings.TrimSuffix(strings.TrimPrefix(sql, "("), ")"))
		}
	}

	return strings.Join(joinConditions, " ") + " "
}
```

创建 joinSQL 的过程中主要用到了 `buildCondition`, 继续深入:

```go
func (scope *Scope) buildCondition(clause map[string]interface{}, include bool) (str string) {
	var (
		quotedTableName  = scope.QuotedTableName()
		quotedPrimaryKey = scope.Quote(scope.PrimaryKey())
		equalSQL         = "="
		inSQL            = "IN"
	)

	// If building not conditions
	if !include {
		equalSQL = "<>"
		inSQL = "NOT IN"
	}

	switch value := clause["query"].(type) {
	case sql.NullInt64:
		return fmt.Sprintf("(%v.%v %s %v)", quotedTableName, quotedPrimaryKey, equalSQL, value.Int64)
	case int, int8, int16, int32, int64, uint, uint8, uint16, uint32, uint64:
		return fmt.Sprintf("(%v.%v %s %v)", quotedTableName, quotedPrimaryKey, equalSQL, value)
	case []int, []int8, []int16, []int32, []int64, []uint, []uint8, []uint16, []uint32, []uint64, []string, []interface{}:
		if !include && reflect.ValueOf(value).Len() == 0 {
			return
		}
		str = fmt.Sprintf("(%v.%v %s (?))", quotedTableName, quotedPrimaryKey, inSQL)
		clause["args"] = []interface{}{value}
	case string:
		if isNumberRegexp.MatchString(value) {
			return fmt.Sprintf("(%v.%v %s %v)", quotedTableName, quotedPrimaryKey, equalSQL, scope.AddToVars(value))
		}

		if value != "" {
			if !include {
				if comparisonRegexp.MatchString(value) {
					str = fmt.Sprintf("NOT (%v)", value)
				} else {
					str = fmt.Sprintf("(%v.%v NOT IN (?))", quotedTableName, scope.Quote(value))
				}
			} else {
				str = fmt.Sprintf("(%v)", value)
			}
		}
	case map[string]interface{}:
		var sqls []string
		for key, value := range value {
			if value != nil {
				sqls = append(sqls, fmt.Sprintf("(%v.%v %s %v)", quotedTableName, scope.Quote(key), equalSQL, scope.AddToVars(value)))
			} else {
				if !include {
					sqls = append(sqls, fmt.Sprintf("(%v.%v IS NOT NULL)", quotedTableName, scope.Quote(key)))
				} else {
					sqls = append(sqls, fmt.Sprintf("(%v.%v IS NULL)", quotedTableName, scope.Quote(key)))
				}
			}
		}
		return strings.Join(sqls, " AND ")
	case interface{}:
		var sqls []string
		newScope := scope.New(value)

		if len(newScope.Fields()) == 0 {
			scope.Err(fmt.Errorf("invalid query condition: %v", value))
			return
		}
		scopeQuotedTableName := newScope.QuotedTableName()
		for _, field := range newScope.Fields() {
			if !field.IsIgnored && !field.IsBlank {
				sqls = append(sqls, fmt.Sprintf("(%v.%v %s %v)", scopeQuotedTableName, scope.Quote(field.DBName), equalSQL, scope.AddToVars(field.Field.Interface())))
			}
		}
		return strings.Join(sqls, " AND ")
	default:
		scope.Err(fmt.Errorf("invalid query condition: %v", value))
		return
	}

	replacements := []string{}
	args := clause["args"].([]interface{})
	for _, arg := range args {
		var err error
		switch reflect.ValueOf(arg).Kind() {
		case reflect.Slice: // For where("id in (?)", []int64{1,2})
			if scanner, ok := interface{}(arg).(driver.Valuer); ok {
				arg, err = scanner.Value()
				replacements = append(replacements, scope.AddToVars(arg))
			} else if b, ok := arg.([]byte); ok {
				replacements = append(replacements, scope.AddToVars(b))
			} else if as, ok := arg.([][]interface{}); ok {
				var tempMarks []string
				for _, a := range as {
					var arrayMarks []string
					for _, v := range a {
						arrayMarks = append(arrayMarks, scope.AddToVars(v))
					}

					if len(arrayMarks) > 0 {
						tempMarks = append(tempMarks, fmt.Sprintf("(%v)", strings.Join(arrayMarks, ",")))
					}
				}

				if len(tempMarks) > 0 {
					replacements = append(replacements, strings.Join(tempMarks, ","))
				}
			} else if values := reflect.ValueOf(arg); values.Len() > 0 {
				var tempMarks []string
				for i := 0; i < values.Len(); i++ {
					tempMarks = append(tempMarks, scope.AddToVars(values.Index(i).Interface()))
				}
				replacements = append(replacements, strings.Join(tempMarks, ","))
			} else {
				replacements = append(replacements, scope.AddToVars(Expr("NULL")))
			}
		default:
			if valuer, ok := interface{}(arg).(driver.Valuer); ok {
				arg, err = valuer.Value()
			}

			replacements = append(replacements, scope.AddToVars(arg))
		}

		if err != nil {
			scope.Err(err)
		}
	}

	buff := bytes.NewBuffer([]byte{})
	i := 0
	for _, s := range str {
		if s == '?' && len(replacements) > i {
			buff.WriteString(replacements[i])
			i++
		} else {
			buff.WriteRune(s)
		}
	}

	str = buff.String()

	return
}
```

开头是一个精妙的选择, 基于 `include`, 实现了 not 条件.

```go
var (
  quotedTableName  = scope.QuotedTableName()
  quotedPrimaryKey = scope.Quote(scope.PrimaryKey())
  equalSQL         = "="
  inSQL            = "IN"
)

// If building not conditions
if !include {
  equalSQL = "<>"
  inSQL = "NOT IN"
}
```

中间是一个 `switch value := clause["query"].(type)` 选择.
在这个 switch 选择中, 大部分的条件都会直接返回.
剩余的部分, 则会构建 `str` 字符串变量.

而这会继续进入到结尾部分, 这部分的代码和我们上面看过的非常类似,
就是根据 `clause["args"]` 构建 `replacements` 切片,
用来替换 `str` 变量中的 `?`.

接着看下一个 `whereSQL` 方法.

```go
func (scope *Scope) whereSQL() (sql string) {
	var (
		quotedTableName                                = scope.QuotedTableName()
		deletedAtField, hasDeletedAtField              = scope.FieldByName("DeletedAt")
		primaryConditions, andConditions, orConditions []string
	)

	if !scope.Search.Unscoped && hasDeletedAtField {
		sql := fmt.Sprintf("%v.%v IS NULL", quotedTableName, scope.Quote(deletedAtField.DBName))
		primaryConditions = append(primaryConditions, sql)
	}

	if !scope.PrimaryKeyZero() {
		for _, field := range scope.PrimaryFields() {
			sql := fmt.Sprintf("%v.%v = %v", quotedTableName, scope.Quote(field.DBName), scope.AddToVars(field.Field.Interface()))
			primaryConditions = append(primaryConditions, sql)
		}
	}

	for _, clause := range scope.Search.whereConditions {
		if sql := scope.buildCondition(clause, true); sql != "" {
			andConditions = append(andConditions, sql)
		}
	}

	for _, clause := range scope.Search.orConditions {
		if sql := scope.buildCondition(clause, true); sql != "" {
			orConditions = append(orConditions, sql)
		}
	}

	for _, clause := range scope.Search.notConditions {
		if sql := scope.buildCondition(clause, false); sql != "" {
			andConditions = append(andConditions, sql)
		}
	}

	orSQL := strings.Join(orConditions, " OR ")
	combinedSQL := strings.Join(andConditions, " AND ")
	if len(combinedSQL) > 0 {
		if len(orSQL) > 0 {
			combinedSQL = combinedSQL + " OR " + orSQL
		}
	} else {
		combinedSQL = orSQL
	}

	if len(primaryConditions) > 0 {
		sql = "WHERE " + strings.Join(primaryConditions, " AND ")
		if len(combinedSQL) > 0 {
			sql = sql + " AND (" + combinedSQL + ")"
		}
	} else if len(combinedSQL) > 0 {
		sql = "WHERE " + combinedSQL
	}
	return
}
```

主要构建了三个部分, `primaryConditions, andConditions, orConditions`.

```go
if !scope.Search.Unscoped && hasDeletedAtField {
  sql := fmt.Sprintf("%v.%v IS NULL", quotedTableName, scope.Quote(deletedAtField.DBName))
  primaryConditions = append(primaryConditions, sql)
}

if !scope.PrimaryKeyZero() {
  for _, field := range scope.PrimaryFields() {
    sql := fmt.Sprintf("%v.%v = %v", quotedTableName, scope.Quote(field.DBName), scope.AddToVars(field.Field.Interface()))
    primaryConditions = append(primaryConditions, sql)
  }
}
```

前面两个 if 构建了 `primaryConditions` 条件.

```go
for _, clause := range scope.Search.whereConditions {
  if sql := scope.buildCondition(clause, true); sql != "" {
    andConditions = append(andConditions, sql)
  }
}

for _, clause := range scope.Search.orConditions {
  if sql := scope.buildCondition(clause, true); sql != "" {
    orConditions = append(orConditions, sql)
  }
}

for _, clause := range scope.Search.notConditions {
  if sql := scope.buildCondition(clause, false); sql != "" {
    andConditions = append(andConditions, sql)
  }
}
```

然后三个 for 循环都使用了 `buildCondition` 方法.
注意到 `scope.Search.notConditions` 是算在 `andConditions` 中的.

```go
orSQL := strings.Join(orConditions, " OR ")
combinedSQL := strings.Join(andConditions, " AND ")
if len(combinedSQL) > 0 {
  if len(orSQL) > 0 {
    combinedSQL = combinedSQL + " OR " + orSQL
  }
} else {
  combinedSQL = orSQL
}
```

结合 `orConditions` 和 `andConditions` 生成了条件语句.

```go
if len(primaryConditions) > 0 {
  sql = "WHERE " + strings.Join(primaryConditions, " AND ")
  if len(combinedSQL) > 0 {
    sql = sql + " AND (" + combinedSQL + ")"
  }
} else if len(combinedSQL) > 0 {
  sql = "WHERE " + combinedSQL
}
return
```

最后, 结合 `primaryConditions` 生成最终的 WHERE 子句.

接着看另一个:

```go
func (scope *Scope) groupSQL() string {
	if len(scope.Search.group) == 0 {
		return ""
	}
	return " GROUP BY " + scope.Search.group
}
```

GROUP BY 子句比较简单, 直接就能构建.

继续:

```go
func (scope *Scope) havingSQL() string {
	if len(scope.Search.havingConditions) == 0 {
		return ""
	}

	var andConditions []string
	for _, clause := range scope.Search.havingConditions {
		if sql := scope.buildCondition(clause, true); sql != "" {
			andConditions = append(andConditions, sql)
		}
	}

	combinedSQL := strings.Join(andConditions, " AND ")
	if len(combinedSQL) == 0 {
		return ""
	}

	return " HAVING " + combinedSQL
}
```

HAVING 子句也不算难, 构建完条件之后用 AND 连接, 然后在最前面加上 HAVING 就行了.

继续:

```go
func (scope *Scope) orderSQL() string {
	if len(scope.Search.orders) == 0 || scope.Search.ignoreOrderQuery {
		return ""
	}

	var orders []string
	for _, order := range scope.Search.orders {
		if str, ok := order.(string); ok {
			orders = append(orders, scope.quoteIfPossible(str))
		} else if expr, ok := order.(*expr); ok {
			exp := expr.expr
			for _, arg := range expr.args {
				exp = strings.Replace(exp, "?", scope.AddToVars(arg), 1)
			}
			orders = append(orders, exp)
		}
	}
	return " ORDER BY " + strings.Join(orders, ",")
}
```

结构也是类似, 遍历 `scope.Search.orders` 切片, `order` 有两种不同的类型, 字符串或者 `expr` 结构体.
后者用于处理带参数的情况.

最后还有一个 `limitAndOffsetSQL` 方法:

```go
func (scope *Scope) limitAndOffsetSQL() string {
	return scope.Dialect().LimitAndOffsetSQL(scope.Search.limit, scope.Search.offset)
}
```

这直接调用了具体数据库驱动中的 `LimitAndOffsetSQL` 方法.

看两个具体的实现, 一个是通用中的实现, 另一个是 mysql 中的实现.

```go
func (commonDialect) LimitAndOffsetSQL(limit, offset interface{}) (sql string) {
	if limit != nil {
		if parsedLimit, err := strconv.ParseInt(fmt.Sprint(limit), 0, 0); err == nil && parsedLimit >= 0 {
			sql += fmt.Sprintf(" LIMIT %d", parsedLimit)
		}
	}
	if offset != nil {
		if parsedOffset, err := strconv.ParseInt(fmt.Sprint(offset), 0, 0); err == nil && parsedOffset >= 0 {
			sql += fmt.Sprintf(" OFFSET %d", parsedOffset)
		}
	}
	return
}
```

直接将 limit 和 offset 解析为 int 类型, 然后连接对应的关键字即可.

接着看一下 mysql 中的实现:

```go
func (s mysql) LimitAndOffsetSQL(limit, offset interface{}) (sql string) {
	if limit != nil {
		if parsedLimit, err := strconv.ParseInt(fmt.Sprint(limit), 0, 0); err == nil && parsedLimit >= 0 {
			sql += fmt.Sprintf(" LIMIT %d", parsedLimit)

			if offset != nil {
				if parsedOffset, err := strconv.ParseInt(fmt.Sprint(offset), 0, 0); err == nil && parsedOffset >= 0 {
					sql += fmt.Sprintf(" OFFSET %d", parsedOffset)
				}
			}
		}
	}
	return
}
```

两者的区别在于 offset 的嵌套, mysql 中 offset 必须和 limit 一起使用.

就这样, `CombinedConditionSql` 中的所有子句都看完了.
说到底其实也没什么魔法, 不过是根据不同的条件, 构建不同的 SQL 语句.

### 小结

一路从 `First` 深入到查询的内部细节. 在了解了底层细节之后, 其他类似的方法也就不难理解了.

```go
// Take return a record that match given conditions, the order will depend on the database implementation
func (s *DB) Take(out interface{}, where ...interface{}) *DB {
	newScope := s.NewScope(out)
	newScope.Search.Limit(1)
	return newScope.inlineCondition(where...).callCallbacks(s.parent.callbacks.queries).db
}

// Last find last record that match given conditions, order by primary key
func (s *DB) Last(out interface{}, where ...interface{}) *DB {
	newScope := s.NewScope(out)
	newScope.Search.Limit(1)
	return newScope.Set("gorm:order_by_primary_key", "DESC").
		inlineCondition(where...).callCallbacks(s.parent.callbacks.queries).db
}

// Find find records that match given conditions
func (s *DB) Find(out interface{}, where ...interface{}) *DB {
	return s.NewScope(out).inlineCondition(where...).callCallbacks(s.parent.callbacks.queries).db
}
```

## search 结构体

前面的过程中, 我们只看到了最简单的查询是如何产生的.
在这个过程中, 没有仔细研究查询条件是如何存储的.

看一下如何使用 `Where` 方法添加查询条件.

```go
// Get first matched record
db.Where("name = ?", "jinzhu").First(&user)
//// SELECT * FROM users WHERE name = 'jinzhu' limit 1;

// Get all matched records
db.Where("name = ?", "jinzhu").Find(&users)
//// SELECT * FROM users WHERE name = 'jinzhu';
```

上面的例子来自于官方文档. GORM 使用链式调用的风格, 可以串联多个 Where 方法, 或是其他的查询条件.

```go
// Where return a new relation, filter records with given conditions, accepts `map`, `struct` or `string` as conditions, refer http://jinzhu.github.io/gorm/crud.html#query
func (s *DB) Where(query interface{}, args ...interface{}) *DB {
	return s.clone().search.Where(query, args...).db
}
```

上面是 `Where` 方法的代码, 在它的源码附近有很多类似的的方法.

```go
// Or filter records that match before conditions or this one, similar to `Where`
func (s *DB) Or(query interface{}, args ...interface{}) *DB {
	return s.clone().search.Or(query, args...).db
}

// Not filter records that don't match current conditions, similar to `Where`
func (s *DB) Not(query interface{}, args ...interface{}) *DB {
	return s.clone().search.Not(query, args...).db
}
```

可以很容易的发现, 这一切的源头都是 `search` 对象.

结构体 `DB` 定义的时候, 有个字段就是 `search`:

```go
search            *search
```

### search 的定义

这就是用于存储查询条件的地方. 它的定义如下:

```go
type search struct {
	db               *DB
	whereConditions  []map[string]interface{}
	orConditions     []map[string]interface{}
	notConditions    []map[string]interface{}
	havingConditions []map[string]interface{}
	joinConditions   []map[string]interface{}
	initAttrs        []interface{}
	assignAttrs      []interface{}
	selects          map[string]interface{}
	omits            []string
	orders           []interface{}
	preload          []searchPreload
	offset           interface{}
	limit            interface{}
	group            string
	tableName        string
	raw              bool
	Unscoped         bool
	ignoreOrderQuery bool
}

type searchPreload struct {
	schema     string
	conditions []interface{}
}
```

这里有很多类型为 `[]map[string]interface{}` 的字段, 结合前面关于条件查询的代码, 就能回忆起这就是存储各种条件的地方.

另一些字段比如 `offset` 和 `limit` 也很容易明白它的作用.

### search 的方法

search 下有很多方法, 虽然方法数量比较多, 但基本都很短, 总共也就一百行出头.

```go
func (s *search) clone() *search {
	clone := *s
	return &clone
}
```

这个克隆方法有点独特, 似乎什么也没做, 也可能是我见识少.

```go
func (s *search) Where(query interface{}, values ...interface{}) *search {
	s.whereConditions = append(s.whereConditions, map[string]interface{}{"query": query, "args": values})
	return s
}

func (s *search) Not(query interface{}, values ...interface{}) *search {
	s.notConditions = append(s.notConditions, map[string]interface{}{"query": query, "args": values})
	return s
}

func (s *search) Or(query interface{}, values ...interface{}) *search {
	s.orConditions = append(s.orConditions, map[string]interface{}{"query": query, "args": values})
	return s
}
```

上面这些方法都是用参数构建成一个 map 然后推入对应的切片中, 考虑到链式调用, 返回了本身.

```go
func (s *search) Attrs(attrs ...interface{}) *search {
	s.initAttrs = append(s.initAttrs, toSearchableMap(attrs...))
	return s
}

func (s *search) Assign(attrs ...interface{}) *search {
	s.assignAttrs = append(s.assignAttrs, toSearchableMap(attrs...))
	return s
}

func toSearchableMap(attrs ...interface{}) (result interface{}) {
	if len(attrs) > 1 {
		if str, ok := attrs[0].(string); ok {
			result = map[string]interface{}{str: attrs[1]}
		}
	} else if len(attrs) == 1 {
		if attr, ok := attrs[0].(map[string]interface{}); ok {
			result = attr
		}

		if attr, ok := attrs[0].(interface{}); ok {
			result = attr
		}
	}
	return
}
```

这两个方法也是类似, 并使用了 `toSearchableMap` 转换参数.

```go
func (s *search) Order(value interface{}, reorder ...bool) *search {
	if len(reorder) > 0 && reorder[0] {
		s.orders = []interface{}{}
	}

	if value != nil && value != "" {
		s.orders = append(s.orders, value)
	}
	return s
}
```

看到这个可能有点疑惑, 可以从文档和注释中获取解释.

```go
// Order specify order when retrieve records from database, set reorder to `true` to overwrite defined conditions
//     db.Order("name DESC")
//     db.Order("name DESC", true) // reorder
//     db.Order(gorm.Expr("name = ? DESC", "first")) // sql expression
func (s *DB) Order(value interface{}, reorder ...bool) *DB {
	return s.clone().search.Order(value, reorder...).db
}
```

第二个参数用于判断是否覆盖前面的排序条件.

可能有点奇怪的是为什么 `reorder` 是可变参数, 不知为了兼容或者是历史遗留.

另一点是不能理解 `[]interface{}{}`, 这其实可以分为两部分, `[]interface{}` 是类型, `{}` 构造了一个空的该类型实例.

```go
func (s *search) Select(query interface{}, args ...interface{}) *search {
	s.selects = map[string]interface{}{"query": query, "args": args}
	return s
}

func (s *search) Omit(columns ...string) *search {
	s.omits = columns
	return s
}

func (s *search) Limit(limit interface{}) *search {
	s.limit = limit
	return s
}

func (s *search) Offset(offset interface{}) *search {
	s.offset = offset
	return s
}
```

这几个就是替换型的了, 每次调用都只会保存最新值.

```go
func (s *search) Group(query string) *search {
	s.group = s.getInterfaceAsSQL(query)
	return s
}

func (s *search) getInterfaceAsSQL(value interface{}) (str string) {
	switch value.(type) {
	case string, int, int8, int16, int32, int64, uint, uint8, uint16, uint32, uint64:
		str = fmt.Sprintf("%v", value)
	default:
		s.db.AddError(ErrInvalidSQL)
	}

	if str == "-1" {
		return ""
	}
	return
}
```

`getInterfaceAsSQL` 的一个特性是使用 `-1` 会重置.

```go
func (s *search) Having(query interface{}, values ...interface{}) *search {
	if val, ok := query.(*expr); ok {
		s.havingConditions = append(s.havingConditions, map[string]interface{}{"query": val.expr, "args": val.args})
	} else {
		s.havingConditions = append(s.havingConditions, map[string]interface{}{"query": query, "args": values})
	}
	return s
}

func (s *search) Joins(query string, values ...interface{}) *search {
	s.joinConditions = append(s.joinConditions, map[string]interface{}{"query": query, "args": values})
	return s
}
```

这其实也比较类似前面看过的, 就不多解释了.

```go
func (s *search) Preload(schema string, values ...interface{}) *search {
	var preloads []searchPreload
	for _, preload := range s.preload {
		if preload.schema != schema {
			preloads = append(preloads, preload)
		}
	}
	preloads = append(preloads, searchPreload{schema, values})
	s.preload = preloads
	return s
}
```

`Preload` 需要防止重复, 所以开头会重新遍历一遍已经存在的 `schema`.

```go
func (s *search) Raw(b bool) *search {
	s.raw = b
	return s
}

func (s *search) unscoped() *search {
	s.Unscoped = true
	return s
}

func (s *search) Table(name string) *search {
	s.tableName = name
	return s
}
```

最后几个方法也没什么特殊的.

### 小结

search 结构体还是挺简单的, 定义加方法总共也就一百多行.
但用处却不小, 查询相关的条件都是存储在这里的.

## 总结

这部分主要查看了 SQL 查询是如何发生的, 并在这个过程中探索了各种查询子句是如何实现的. 同时, 也研究了一下 search 结构体和它的作用.
