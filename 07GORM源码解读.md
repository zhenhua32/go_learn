## 简介

GORM 源码解读, 基于 [v1.9.11](https://github.com/jinzhu/gorm/tree/v1.9.11) 版本.

## 关系的使用

前面看过了模型的定义中关系是如何解析的. 这次主要是研究一下和关系相关的几个方法.

## Related

使用 `Related` 可以查询关联的数据.

```go
db.Model(&user).Related(&profile)
//// SELECT * FROM profiles WHERE user_id = 111; // 111 is user's ID
```

上面是 `Belongs To` 文档中的例子, 从 `user` 中查询关联的 `profile`.

`Related` 是有其他参数的, 看一下 `Has One` 文档中的例子.

```go
var card CreditCard
db.Model(&user).Related(&card, "CreditCard")
//// SELECT * FROM credit_cards WHERE user_id = 123; // 123 is user's primary key
// CreditCard 是 users 的字段，其含义是，获取 user 的 CreditCard 并填充至 card 变量
// 如果字段名与 model 名相同，比如上面的例子，此时字段名可以省略不写，像这样：
db.Model(&user).Related(&card)
```

第二个参数用于指定外键.

直接看源代码吧:

```go
// Related get related associations
func (s *DB) Related(value interface{}, foreignKeys ...string) *DB {
	return s.NewScope(s.Value).related(value, foreignKeys...).db
}

func (scope *Scope) related(value interface{}, foreignKeys ...string) *Scope {
	toScope := scope.db.NewScope(value)
	tx := scope.db.Set("gorm:association:source", scope.Value)

	for _, foreignKey := range append(foreignKeys, toScope.typeName()+"Id", scope.typeName()+"Id") {
		fromField, _ := scope.FieldByName(foreignKey)
		toField, _ := toScope.FieldByName(foreignKey)

		if fromField != nil {
			if relationship := fromField.Relationship; relationship != nil {
				if relationship.Kind == "many_to_many" {
					joinTableHandler := relationship.JoinTableHandler
					scope.Err(joinTableHandler.JoinWith(joinTableHandler, tx, scope.Value).Find(value).Error)
				} else if relationship.Kind == "belongs_to" {
					for idx, foreignKey := range relationship.ForeignDBNames {
						if field, ok := scope.FieldByName(foreignKey); ok {
							tx = tx.Where(fmt.Sprintf("%v = ?", scope.Quote(relationship.AssociationForeignDBNames[idx])), field.Field.Interface())
						}
					}
					scope.Err(tx.Find(value).Error)
				} else if relationship.Kind == "has_many" || relationship.Kind == "has_one" {
					for idx, foreignKey := range relationship.ForeignDBNames {
						if field, ok := scope.FieldByName(relationship.AssociationForeignDBNames[idx]); ok {
							tx = tx.Where(fmt.Sprintf("%v = ?", scope.Quote(foreignKey)), field.Field.Interface())
						}
					}

					if relationship.PolymorphicType != "" {
						tx = tx.Where(fmt.Sprintf("%v = ?", scope.Quote(relationship.PolymorphicDBName)), relationship.PolymorphicValue)
					}
					scope.Err(tx.Find(value).Error)
				}
			} else {
				sql := fmt.Sprintf("%v = ?", scope.Quote(toScope.PrimaryKey()))
				scope.Err(tx.Where(sql, fromField.Field.Interface()).Find(value).Error)
			}
			return scope
		} else if toField != nil {
			sql := fmt.Sprintf("%v = ?", scope.Quote(toField.DBName))
			scope.Err(tx.Where(sql, scope.PrimaryKeyValue()).Find(value).Error)
			return scope
		}
	}

	scope.Err(fmt.Errorf("invalid association %v", foreignKeys))
	return scope
}
```

直接看 `related` 的代码, 最核心的是中间的 for 循环部分. 如果这个循环不返回, 就会直接设置错误.

```go
toScope := scope.db.NewScope(value)
tx := scope.db.Set("gorm:association:source", scope.Value)
```

定义了 `toScope` 变量. `tx` 是设置了 `gorm:association:source` 后的 `*DB` 类型.

接着看 for 循环部分, range 的是 `append(foreignKeys, toScope.typeName()+"Id", scope.typeName()+"Id")`.
所以至少有两个值, 这就是为什么前面使用 `Related` 方法的时候可以只使用一个参数.

```go
fromField, _ := scope.FieldByName(foreignKey)
toField, _ := toScope.FieldByName(foreignKey)
```

分别从不同的模型中获取了 `fromField` 和 `toField` 字段.

先看 `fromField != nil` 的情况:

```go
if relationship := fromField.Relationship; relationship != nil {
  if relationship.Kind == "many_to_many" {
    joinTableHandler := relationship.JoinTableHandler
    scope.Err(joinTableHandler.JoinWith(joinTableHandler, tx, scope.Value).Find(value).Error)
  } else if relationship.Kind == "belongs_to" {
    for idx, foreignKey := range relationship.ForeignDBNames {
      if field, ok := scope.FieldByName(foreignKey); ok {
        tx = tx.Where(fmt.Sprintf("%v = ?", scope.Quote(relationship.AssociationForeignDBNames[idx])), field.Field.Interface())
      }
    }
    scope.Err(tx.Find(value).Error)
  } else if relationship.Kind == "has_many" || relationship.Kind == "has_one" {
    for idx, foreignKey := range relationship.ForeignDBNames {
      if field, ok := scope.FieldByName(relationship.AssociationForeignDBNames[idx]); ok {
        tx = tx.Where(fmt.Sprintf("%v = ?", scope.Quote(foreignKey)), field.Field.Interface())
      }
    }

    if relationship.PolymorphicType != "" {
      tx = tx.Where(fmt.Sprintf("%v = ?", scope.Quote(relationship.PolymorphicDBName)), relationship.PolymorphicValue)
    }
    scope.Err(tx.Find(value).Error)
  }
} else {
  sql := fmt.Sprintf("%v = ?", scope.Quote(toScope.PrimaryKey()))
  scope.Err(tx.Where(sql, fromField.Field.Interface()).Find(value).Error)
}
return scope
```

如果 `fromField.Relationship` 不为空, 会进行关系类型的判断, 然后执行对应的查询.
如果 `fromField.Relationship` 为空, 查询使用的字段名是 `toScope` 的主键, 以及 `fromField` 对应的值.

再来看 `toField != nil` 的情况, 这个就会比较简单了:

```go
else if toField != nil {
  sql := fmt.Sprintf("%v = ?", scope.Quote(toField.DBName))
  scope.Err(tx.Where(sql, scope.PrimaryKeyValue()).Find(value).Error)
  return scope
}
```

使用 `toField` 的字段名, 以及 `scope.PrimaryKeyValue()` 的主键值.

## Association

使用 `Association` 方法可以进入关联模式.

```go
// 开始使用关联模式
var user User
db.Model(&user).Association("Languages")
// `user` 是源，必须包含主键
// `Languages` 是关系中的源的字段名
// 只有在满足上面两个条件时，关联模式才能正常工作，请注意检查错误：
// db.Model(&user).Association("Languages").Error
```
