## 简介

GORM 源码解读, 基于 [v1.9.11](https://github.com/jinzhu/gorm/tree/v1.9.11) 版本.

## 全量更新

上一节讲过了查询, 这次就轮到更新了.

更新一个对象有多种方式, 先看一下全量更新.

```go
db.First(&user)

user.Name = "jinzhu 2"
user.Age = 100
db.Save(&user)

//// UPDATE users SET name='jinzhu 2', age=100, birthday='2016-01-01', updated_at = '2013-11-17 21:34:10' WHERE id=111;
```

上面的例子来自官方文档, 同时文档上解释了 `Save` 是全量更新, 会更新所有字段, 即使没有赋值.

```go
// Save update value in database, if the value doesn't have primary key, will insert it
func (s *DB) Save(value interface{}) *DB {
	scope := s.NewScope(value)
	if !scope.PrimaryKeyZero() {
		newDB := scope.callCallbacks(s.parent.callbacks.updates).db
		if newDB.Error == nil && newDB.RowsAffected == 0 {
			return s.New().Table(scope.TableName()).FirstOrCreate(value)
		}
		return newDB
	}
	return scope.callCallbacks(s.parent.callbacks.creates).db
}
```

如果没有主键, 将会调用 creates 回调插入数据. 否则就是调用 update 回调更新数据.
如果没有行数被影响, 会使用 `FirstOrCreate` 方法.

```go
// FirstOrCreate find first matched record or create a new one with given conditions (only works with struct, map conditions)
// https://jinzhu.github.io/gorm/crud.html#firstorcreate
func (s *DB) FirstOrCreate(out interface{}, where ...interface{}) *DB {
	c := s.clone()
	if result := s.First(out, where...); result.Error != nil {
		if !result.RecordNotFound() {
			return result
		}
		return c.NewScope(out).inlineCondition(where...).initialize().callCallbacks(c.parent.callbacks.creates).db
	} else if len(c.search.assignAttrs) > 0 {
		return c.NewScope(out).InstanceSet("gorm:update_interface", c.search.assignAttrs).callCallbacks(c.parent.callbacks.updates).db
	}
	return c
}
```

从代码上看, `FirstOrCreate` 会首先寻找匹配的记录, 如果没有找到, 也会插入数据.

`callbacks.creates` 在前面已经见识过了, 来看看 `callbacks.updates`.

```go
// Define callbacks for updating
func init() {
	DefaultCallback.Update().Register("gorm:assign_updating_attributes", assignUpdatingAttributesCallback)
	DefaultCallback.Update().Register("gorm:begin_transaction", beginTransactionCallback)
	DefaultCallback.Update().Register("gorm:before_update", beforeUpdateCallback)
	DefaultCallback.Update().Register("gorm:save_before_associations", saveBeforeAssociationsCallback)
	DefaultCallback.Update().Register("gorm:update_time_stamp", updateTimeStampForUpdateCallback)
	DefaultCallback.Update().Register("gorm:update", updateCallback)
	DefaultCallback.Update().Register("gorm:save_after_associations", saveAfterAssociationsCallback)
	DefaultCallback.Update().Register("gorm:after_update", afterUpdateCallback)
	DefaultCallback.Update().Register("gorm:commit_or_rollback_transaction", commitOrRollbackTransactionCallback)
}
```

`Update` 上注册了一大堆的回调函数. 直接看最关键的 `updateCallback`:

```go
// updateCallback the callback used to update data to database
func updateCallback(scope *Scope) {
	if !scope.HasError() {
		var sqls []string

		if updateAttrs, ok := scope.InstanceGet("gorm:update_attrs"); ok {
			// Sort the column names so that the generated SQL is the same every time.
			updateMap := updateAttrs.(map[string]interface{})
			var columns []string
			for c := range updateMap {
				columns = append(columns, c)
			}
			sort.Strings(columns)

			for _, column := range columns {
				value := updateMap[column]
				sqls = append(sqls, fmt.Sprintf("%v = %v", scope.Quote(column), scope.AddToVars(value)))
			}
		} else {
			for _, field := range scope.Fields() {
				if scope.changeableField(field) {
					if !field.IsPrimaryKey && field.IsNormal && (field.Name != "CreatedAt" || !field.IsBlank) {
						if !field.IsForeignKey || !field.IsBlank || !field.HasDefaultValue {
							sqls = append(sqls, fmt.Sprintf("%v = %v", scope.Quote(field.DBName), scope.AddToVars(field.Field.Interface())))
						}
					} else if relationship := field.Relationship; relationship != nil && relationship.Kind == "belongs_to" {
						for _, foreignKey := range relationship.ForeignDBNames {
							if foreignField, ok := scope.FieldByName(foreignKey); ok && !scope.changeableField(foreignField) {
								sqls = append(sqls,
									fmt.Sprintf("%v = %v", scope.Quote(foreignField.DBName), scope.AddToVars(foreignField.Field.Interface())))
							}
						}
					}
				}
			}
		}

		var extraOption string
		if str, ok := scope.Get("gorm:update_option"); ok {
			extraOption = fmt.Sprint(str)
		}

		if len(sqls) > 0 {
			scope.Raw(fmt.Sprintf(
				"UPDATE %v SET %v%v%v",
				scope.QuotedTableName(),
				strings.Join(sqls, ", "),
				addExtraSpaceIfExist(scope.CombinedConditionSql()),
				addExtraSpaceIfExist(extraOption),
			)).Exec()
		}
	}
}
```

第一个 if 判断是否前面已经有错误产生, 一旦有错误存在, 就表明前面还未处理, 所以直接就跳过更新了.

一开始就定义了 `var sqls []string` 来保存 sql 语句的片段.

然后进入到更新属性的 if 判断中, `updateAttrs, ok := scope.InstanceGet("gorm:update_attrs")`.

```go
if updateAttrs, ok := scope.InstanceGet("gorm:update_attrs"); ok {
  // Sort the column names so that the generated SQL is the same every time.
  updateMap := updateAttrs.(map[string]interface{})
  var columns []string
  for c := range updateMap {
    columns = append(columns, c)
  }
  sort.Strings(columns)

  for _, column := range columns {
    value := updateMap[column]
    sqls = append(sqls, fmt.Sprintf("%v = %v", scope.Quote(column), scope.AddToVars(value)))
  }
}
```

使用 `columns` 保存需要更新的列名, 同时还排序了一下. 然后遍历列名, 生成 sql 片段 `"%v = %v"`.

接着看 else 的部分, 这是在没有 `gorm:update_attrs` 时的选择.

```go
else {
  for _, field := range scope.Fields() {
    if scope.changeableField(field) {
      if !field.IsPrimaryKey && field.IsNormal && (field.Name != "CreatedAt" || !field.IsBlank) {
        if !field.IsForeignKey || !field.IsBlank || !field.HasDefaultValue {
          sqls = append(sqls, fmt.Sprintf("%v = %v", scope.Quote(field.DBName), scope.AddToVars(field.Field.Interface())))
        }
      } else if relationship := field.Relationship; relationship != nil && relationship.Kind == "belongs_to" {
        for _, foreignKey := range relationship.ForeignDBNames {
          if foreignField, ok := scope.FieldByName(foreignKey); ok && !scope.changeableField(foreignField) {
            sqls = append(sqls,
              fmt.Sprintf("%v = %v", scope.Quote(foreignField.DBName), scope.AddToVars(foreignField.Field.Interface())))
          }
        }
      }
    }
  }
}
```

主要是遍历 `scope.Fields(`, 并处理那些 `scope.changeableField(field)` 为 true 的字段.

```go
var extraOption string
if str, ok := scope.Get("gorm:update_option"); ok {
  extraOption = fmt.Sprint(str)
}

if len(sqls) > 0 {
  scope.Raw(fmt.Sprintf(
    "UPDATE %v SET %v%v%v",
    scope.QuotedTableName(),
    strings.Join(sqls, ", "),
    addExtraSpaceIfExist(scope.CombinedConditionSql()),
    addExtraSpaceIfExist(extraOption),
  )).Exec()
}
```

最后就是获取了 `gorm:update_option`, 然后和前面得到的 sql 片段一起组成了 `UPDATE` 语句.
更新的时候通常都会有限制条件, 所以这里也用到了 `CombinedConditionSql()` 方法, 这个方法已经在前面了解过了.

## 更新字段

要更新指定的字段, 可以使用 `Update` 或者 `Updates`.

```go
// 更新单个属性，如果它有变化
db.Model(&user).Update("name", "hello")
//// UPDATE users SET name='hello', updated_at='2013-11-17 21:34:10' WHERE id=111;

// 根据给定的条件更新单个属性
db.Model(&user).Where("active = ?", true).Update("name", "hello")
//// UPDATE users SET name='hello', updated_at='2013-11-17 21:34:10' WHERE id=111 AND active=true;

// 使用 map 更新多个属性，只会更新其中有变化的属性
db.Model(&user).Updates(map[string]interface{}{"name": "hello", "age": 18, "actived": false})
//// UPDATE users SET name='hello', age=18, actived=false, updated_at='2013-11-17 21:34:10' WHERE id=111;

// 使用 struct 更新多个属性，只会更新其中有变化且为非零值的字段
db.Model(&user).Updates(User{Name: "hello", Age: 18})
//// UPDATE users SET name='hello', age=18, updated_at = '2013-11-17 21:34:10' WHERE id = 111;

// 警告：当使用 struct 更新时，GORM只会更新那些非零值的字段
// 对于下面的操作，不会发生任何更新，"", 0, false 都是其类型的零值
db.Model(&user).Updates(User{Name: "", Age: 0, Actived: false})
```

上面的示例代码来自官方文档.

先来看看 `Update` 方法.

```go
// Update update attributes with callbacks, refer: https://jinzhu.github.io/gorm/crud.html#update
func (s *DB) Update(attrs ...interface{}) *DB {
	return s.Updates(toSearchableMap(attrs...), true)
}
```

原来内部也是使用了 `Updates` 方法, 那就直接看 `Updates` 吧.

```go
// Updates update attributes with callbacks, refer: https://jinzhu.github.io/gorm/crud.html#update
func (s *DB) Updates(values interface{}, ignoreProtectedAttrs ...bool) *DB {
	return s.NewScope(s.Value).
		Set("gorm:ignore_protected_attrs", len(ignoreProtectedAttrs) > 0).
		InstanceSet("gorm:update_interface", values).
		callCallbacks(s.parent.callbacks.updates).db
}
```

似乎也没有什么特别的, 实际上重点在 `InstanceSet("gorm:update_interface", values)` 上.

回想前面的 `updateCallback` 回调, 在一开始的时候遇到了的一个判断是

```go
if updateAttrs, ok := scope.InstanceGet("gorm:update_attrs"); ok {
```

也就是有没有这个 `orm:update_attrs` 对应的值. 那这又和 `gorm:update_interface` 有什么关系呢?

这更新数据的过程有个回调函数是 `assignUpdatingAttributesCallback`:

```go
// assignUpdatingAttributesCallback assign updating attributes to model
func assignUpdatingAttributesCallback(scope *Scope) {
	if attrs, ok := scope.InstanceGet("gorm:update_interface"); ok {
		if updateMaps, hasUpdate := scope.updatedAttrsWithValues(attrs); hasUpdate {
			scope.InstanceSet("gorm:update_attrs", updateMaps)
		} else {
			scope.SkipLeft()
		}
	}
}
```

这个函数用于将 `gorm:update_interface` 中的数据, 更新为 `gorm:update_attrs`.

这样, 就使得 `gorm:update_attrs` 上有值, 因此当运行 `updateCallback` 时候, if 判断的结果就是真.
所以, 这就会使得 Update 更新的时候只更新 `gorm:update_attrs` 对应的字段, 而不是更新所有的字段.

结合 `assignUpdatingAttributesCallback`, 我们对更新的过程更清晰了, 知道了为何 `Save` 是更新所有的字段, 而 `Update` 和 `Updates` 更新的是指定的字段.

回到 `assignUpdatingAttributesCallback` 函数上, 看一下它的关键过程 `scope.updatedAttrsWithValues(attrs)`.

```go
func (scope *Scope) updatedAttrsWithValues(value interface{}) (results map[string]interface{}, hasUpdate bool) {
	if scope.IndirectValue().Kind() != reflect.Struct {
		return convertInterfaceToMap(value, false, scope.db), true
	}

	results = map[string]interface{}{}

	for key, value := range convertInterfaceToMap(value, true, scope.db) {
		if field, ok := scope.FieldByName(key); ok && scope.changeableField(field) {
			if _, ok := value.(*expr); ok {
				hasUpdate = true
				results[field.DBName] = value
			} else {
				err := field.Set(value)
				if field.IsNormal && !field.IsIgnored {
					hasUpdate = true
					if err == ErrUnaddressable {
						results[field.DBName] = value
					} else {
						results[field.DBName] = field.Field.Interface()
					}
				}
			}
		}
	}
	return
}

func convertInterfaceToMap(values interface{}, withIgnoredField bool, db *DB) map[string]interface{} {
	var attrs = map[string]interface{}{}

	switch value := values.(type) {
	case map[string]interface{}:
		return value
	case []interface{}:
		for _, v := range value {
			for key, value := range convertInterfaceToMap(v, withIgnoredField, db) {
				attrs[key] = value
			}
		}
	case interface{}:
		reflectValue := reflect.ValueOf(values)

		switch reflectValue.Kind() {
		case reflect.Map:
			for _, key := range reflectValue.MapKeys() {
				attrs[ToColumnName(key.Interface().(string))] = reflectValue.MapIndex(key).Interface()
			}
		default:
			for _, field := range (&Scope{Value: values, db: db}).Fields() {
				if !field.IsBlank && (withIgnoredField || !field.IsIgnored) {
					attrs[field.DBName] = field.Field.Interface()
				}
			}
		}
	}
	return attrs
}
```

主要是先使用 `convertInterfaceToMap` 规范化数据成 map 类型, 然后遍历 key 和 value, 最后返回的 results 保存了需要更新的字段.

## 其他更新方式

更新的时候可以指定要更新或忽略的字段, 使用 `Select` 和 `Omit`.

```go
// Select specify fields that you want to retrieve from database when querying, by default, will select all fields;
// When creating/updating, specify fields that you want to save to database
func (s *DB) Select(query interface{}, args ...interface{}) *DB {
	return s.clone().search.Select(query, args...).db
}

// Omit specify fields that you want to ignore when saving to database for creating, updating
func (s *DB) Omit(columns ...string) *DB {
	return s.clone().search.Omit(columns...).db
}
```

对应的内部函数是

```go
func (s *search) Select(query interface{}, args ...interface{}) *search {
	s.selects = map[string]interface{}{"query": query, "args": args}
	return s
}

func (s *search) Omit(columns ...string) *search {
	s.omits = columns
	return s
}

// SelectAttrs return selected attributes
func (scope *Scope) SelectAttrs() []string {
	if scope.selectAttrs == nil {
		attrs := []string{}
		for _, value := range scope.Search.selects {
			if str, ok := value.(string); ok {
				attrs = append(attrs, str)
			} else if strs, ok := value.([]string); ok {
				attrs = append(attrs, strs...)
			} else if strs, ok := value.([]interface{}); ok {
				for _, str := range strs {
					attrs = append(attrs, fmt.Sprintf("%v", str))
				}
			}
		}
		scope.selectAttrs = &attrs
	}
	return *scope.selectAttrs
}

// OmitAttrs return omitted attributes
func (scope *Scope) OmitAttrs() []string {
	return scope.Search.omits
}
```

具体是在哪里判断的呢?

```go
func (scope *Scope) changeableField(field *Field) bool {
	if selectAttrs := scope.SelectAttrs(); len(selectAttrs) > 0 {
		for _, attr := range selectAttrs {
			if field.Name == attr || field.DBName == attr {
				return true
			}
		}
		return false
	}

	for _, attr := range scope.OmitAttrs() {
		if field.Name == attr || field.DBName == attr {
			return false
		}
	}

	return true
}
```

其实就是 `changeableField` 时判断是否要更新这个字段.

另一点是更新的时候会触发 Hook 函数, 要取消这些动作, 可以使用 `UpdateColumn` 和 `UpdateColumns` 方法.

```go
// UpdateColumn update attributes without callbacks, refer: https://jinzhu.github.io/gorm/crud.html#update
func (s *DB) UpdateColumn(attrs ...interface{}) *DB {
	return s.UpdateColumns(toSearchableMap(attrs...))
}

// UpdateColumns update attributes without callbacks, refer: https://jinzhu.github.io/gorm/crud.html#update
func (s *DB) UpdateColumns(values interface{}) *DB {
	return s.NewScope(s.Value).
		Set("gorm:update_column", true).
		Set("gorm:save_associations", false).
		InstanceSet("gorm:update_interface", values).
		callCallbacks(s.parent.callbacks.updates).db
}
```

关键点就是设置了两个选项, `gorm:update_column` 和 `gorm:save_associations`.

以 beforeUpdateCallback 为例:

```go
// beforeUpdateCallback will invoke `BeforeSave`, `BeforeUpdate` method before updating
func beforeUpdateCallback(scope *Scope) {
	if scope.DB().HasBlockGlobalUpdate() && !scope.hasConditions() {
		scope.Err(errors.New("missing WHERE clause while updating"))
		return
	}
	if _, ok := scope.Get("gorm:update_column"); !ok {
		if !scope.HasError() {
			scope.CallMethod("BeforeSave")
		}
		if !scope.HasError() {
			scope.CallMethod("BeforeUpdate")
		}
	}
}
```

只有当 `gorm:update_column` 为 false 的时候, 才会调用 `BeforeSave` 和 `BeforeUpdate` 方法.

## 删除

最后再来看下删除.

```go
// 删除现有记录
db.Delete(&email)
//// DELETE from emails where id=10;

// 为删除 SQL 添加额外的 SQL 操作
db.Set("gorm:delete_option", "OPTION (OPTIMIZE FOR UNKNOWN)").Delete(&email)
//// DELETE from emails where id=10 OPTION (OPTIMIZE FOR UNKNOWN);
```

直接看源码吧:

```go
// Delete delete value match given conditions, if the value has primary key, then will including the primary key as condition
func (s *DB) Delete(value interface{}, where ...interface{}) *DB {
	return s.NewScope(value).inlineCondition(where...).callCallbacks(s.parent.callbacks.deletes).db
}
```

从源码和注释中看, 其实第一个参数不是必须的, 如果第一个参数中有主键, 那么也会将主键作为删除的条件.

所以, 要想批量删除, 直接使用空的对象作为第一个参数, 然后将筛选条件作为其他参数即可.

```go
db.Where("email LIKE ?", "%jinzhu%").Delete(Email{})
//// DELETE from emails where email LIKE "%jinzhu%";

db.Delete(Email{}, "email LIKE ?", "%jinzhu%")
//// DELETE from emails where email LIKE "%jinzhu%";
```

直接看一下删除的回调函数吧.

```go
// Define callbacks for deleting
func init() {
	DefaultCallback.Delete().Register("gorm:begin_transaction", beginTransactionCallback)
	DefaultCallback.Delete().Register("gorm:before_delete", beforeDeleteCallback)
	DefaultCallback.Delete().Register("gorm:delete", deleteCallback)
	DefaultCallback.Delete().Register("gorm:after_delete", afterDeleteCallback)
	DefaultCallback.Delete().Register("gorm:commit_or_rollback_transaction", commitOrRollbackTransactionCallback)
}
```

看一下关键的 `deleteCallback` 函数.

```go
// deleteCallback used to delete data from database or set deleted_at to current time (when using with soft delete)
func deleteCallback(scope *Scope) {
	if !scope.HasError() {
		var extraOption string
		if str, ok := scope.Get("gorm:delete_option"); ok {
			extraOption = fmt.Sprint(str)
		}

		deletedAtField, hasDeletedAtField := scope.FieldByName("DeletedAt")

		if !scope.Search.Unscoped && hasDeletedAtField {
			scope.Raw(fmt.Sprintf(
				"UPDATE %v SET %v=%v%v%v",
				scope.QuotedTableName(),
				scope.Quote(deletedAtField.DBName),
				scope.AddToVars(scope.db.nowFunc()),
				addExtraSpaceIfExist(scope.CombinedConditionSql()),
				addExtraSpaceIfExist(extraOption),
			)).Exec()
		} else {
			scope.Raw(fmt.Sprintf(
				"DELETE FROM %v%v%v",
				scope.QuotedTableName(),
				addExtraSpaceIfExist(scope.CombinedConditionSql()),
				addExtraSpaceIfExist(extraOption),
			)).Exec()
		}
	}
}
```

代码并不多, 这里涉及到删除的几个功能点, 官方文档中也有说明.

第一个功能是软删除, 如果 model 中有字段 `DeletedAt`, 就会获得软删除的功能.
实际上就是将 `DeletedAt` 字段更新为当前的时间, 而不是直接从数据库中删除记录.

```go
if !scope.Search.Unscoped && hasDeletedAtField {
  scope.Raw(fmt.Sprintf(
    "UPDATE %v SET %v=%v%v%v",
    scope.QuotedTableName(),
    scope.Quote(deletedAtField.DBName),
    scope.AddToVars(scope.db.nowFunc()),
    addExtraSpaceIfExist(scope.CombinedConditionSql()),
    addExtraSpaceIfExist(extraOption),
  )).Exec()
}
```

既然有软删除, 自然有对应的物理删除, 也就是直接使用 `DELETE` 语句, 对应的是 else 部分.

```go
else {
  scope.Raw(fmt.Sprintf(
    "DELETE FROM %v%v%v",
    scope.QuotedTableName(),
    addExtraSpaceIfExist(scope.CombinedConditionSql()),
    addExtraSpaceIfExist(extraOption),
  )).Exec()
}
```

要想使用物理删除, 有多种方式, 第一个是在定义 model 时不要定义 `DeletedAt` 字段.
另一种方式是使用 `Unscoped` 方法设置标志位, if 判断的时候会检查 `!scope.Search.Unscoped`.

```go
// Unscoped 方法可以物理删除记录
db.Unscoped().Delete(&order)
//// DELETE FROM orders WHERE id=10;
```

对应的源码是

```go
// Unscoped return all record including deleted record, refer Soft Delete https://jinzhu.github.io/gorm/crud.html#soft-delete
func (s *DB) Unscoped() *DB {
	return s.clone().search.unscoped().db
}

func (s *search) unscoped() *search {
	s.Unscoped = true
	return s
}
```

## 总结

这部分主要查看了 gorm 中 Update 是如何发生的, 顺带也看了 Delete 的实现.
