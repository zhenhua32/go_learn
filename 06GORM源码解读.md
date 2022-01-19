## 简介

GORM 源码解读, 基于 [v1.9.11](https://github.com/jinzhu/gorm/tree/v1.9.11) 版本.

## 关系

SQL 中一个非常重要的点是关系. 和现实生活一样, 每一张表都不太可能是独立的, 表与表之间充满了纠缠的关系.

关系的类型有多种, 通常可以分为三种, 一对一, 一对多, 多对多.

官方文档上介绍了四种不同的关系, 其中一对一被分为了 `Belongs To` 和 `Has One`. 先来看一下这两种定义吧.

## `Belongs To` 和 `Has One`

官方文档的例子如下:

```go
type User struct {
  gorm.Model
  Name string
}

// `Profile` 属于 `User`， 外键是`UserID`
type Profile struct {
  gorm.Model
  UserID int
  User   User
  Name   string
}
```

上面是 `Belongs To` 的, 接着对比下 `Has One` 的:

```go
// User 只能有一张信用卡 (CreditCard), CreditCardID 是外键
type CreditCard struct {
  gorm.Model
  Number   string
  UserID   uint
}

type User struct {
  gorm.Model
  CreditCard   CreditCard
}
```

两者的差别在于语义, `Has One` 更偏向于包容, 一个模型的实例中包含了另一个模型的实例.
在这里的例子中, 就是每个 `User` 都会有一个 `CreditCard`.
而 `Belongs To` 倾向于从属, 这个 `Profile` 是属于某个 `User` 的.

定义的时候, ID(即主键)和模型实例所在的位置也不同.
`Belongs To` 在同一个模型中包含了另一个模型的 ID 和实例.
`Has One` 中实例和 ID 在不同的模型中, `CreditCard` 有 `User` 的主键 `UserID`,
而 `User` 中包含了模型 `CreditCard` 的实例.

`Has One` 和 `Has Many` 支持多态关联, 看一下官方文档上的例子:

```go
type Cat struct {
  ID    int
  Name  string
  Toy   Toy `gorm:"polymorphic:Owner;"`
}

type Dog struct {
  ID   int
  Name string
  Toy  Toy `gorm:"polymorphic:Owner;"`
}

type Toy struct {
  ID        int
  Name      string
  OwnerID   int
  OwnerType string
}
```

这是猫和狗以及玩具之间的关系. 这里猫和狗都有自己的玩具.
而当定义玩具的模型时, 使用 `OwnerID` 和 `OwnerType` 来唯一拥有者.

## `Has Many` 和 `Many To Many`

再来看看一对多和多对多关系.

`Has Many` 的官方文档例子如下:

```go
// User 可以有多张信用卡(CreditCards), UserID 是外键
type User struct {
  gorm.Model
  CreditCards []CreditCard
}

type CreditCard struct {
  gorm.Model
  Number   string
  UserID  uint
}
```

一个 `User` 拥有的 `CreditCard` 可以是零个或多个, 所以定义模型的时候使用了切片形式.

它的多态关联和 `Has One` 类似, 例子如下:

```go
type Cat struct {
  ID    int
  Name  string
  Toy   []Toy `gorm:"polymorphic:Owner;"`
}

type Dog struct {
  ID   int
  Name string
  Toy  []Toy `gorm:"polymorphic:Owner;"`
}

type Toy struct {
  ID        int
  Name      string
  OwnerID   int
  OwnerType string
}
```

`Many To Many` 和前面的都不一样, 因为前面的几种模式都可以依赖外键完成, 但 `Many To Many` 需要额外的表来存储关联数据.

```go
// User 拥有并属于多种 Language，使用 `user_languages` 连接表
type User struct {
  gorm.Model
  Languages         []Language `gorm:"many2many:user_languages;"`
}

type Language struct {
  gorm.Model
  Name string
}
```

对于自引用的多对多关系, 还需要指定一个不同的外键名:

```go
// GORM 会生成一个关联表，其外键为 user_id 和 friend_id，并用其保存自引用用户关系。
type User struct {
  gorm.Model
  Friends []*User `gorm:"many2many:friendships;association_jointable_foreignkey:friend_id"`
}
```

## 实现

前面介绍了 GORM 中的四种关系类型. 现在来看一下它的实现方式.

`createCallback` 回调中有这么一段处理关系的代码:

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

这段代码主要是处理 `belongs_to` 类型的关系的, 会在其中填充 `foreignField` 对应的值.
但更多的还是将关注点指向了 `field.Relationship`, 让我们去看看 `field.Relationship` 到底是什么.

在以前的分析中, 我们已经知道了模型是通过 `ModelStruct` 定义的, 而其中的字段是通过 `StructField` 定义的.

```go
// ModelStruct model definition
type ModelStruct struct {
	PrimaryFields []*StructField
	StructFields  []*StructField
	ModelType     reflect.Type

	defaultTableName string
	l                sync.Mutex
}

// StructField model field's struct definition
type StructField struct {
	DBName          string
	Name            string
	Names           []string
	IsPrimaryKey    bool
	IsNormal        bool
	IsIgnored       bool
	IsScanner       bool
	HasDefaultValue bool
	Tag             reflect.StructTag
	TagSettings     map[string]string
	Struct          reflect.StructField
	IsForeignKey    bool
	Relationship    *Relationship

	tagSettingsLock sync.RWMutex
}
```

`StructField` 其中有几个字段和关系有关, `IsForeignKey` 和 `Relationship`.
`IsForeignKey` 是布尔类型, 用于标识这个字段是否是外键.
更多的注意力应该集中在 `Relationship` 上, 来看看它的定义.

```go
// Relationship described the relationship between models
type Relationship struct {
	Kind                         string
	PolymorphicType              string
	PolymorphicDBName            string
	PolymorphicValue             string
	ForeignFieldNames            []string
	ForeignDBNames               []string
	AssociationForeignFieldNames []string
	AssociationForeignDBNames    []string
	JoinTableHandler             JoinTableHandlerInterface
}
```

以前我们也探索过了解析模型的方法 `GetModelStruct`, 但故意忽略了解析关系的部分.
整个方法就不放了, 直接看和关系有关的部分.

```go
// build relationships
switch indirectType.Kind() {
case reflect.Slice:
  defer func(field *StructField) {
    var (
      relationship           = &Relationship{}
      toScope                = scope.New(reflect.New(field.Struct.Type).Interface())
      foreignKeys            []string
      associationForeignKeys []string
      elemType               = field.Struct.Type
    )

    if foreignKey, _ := field.TagSettingsGet("FOREIGNKEY"); foreignKey != "" {
      foreignKeys = strings.Split(foreignKey, ",")
    }

    if foreignKey, _ := field.TagSettingsGet("ASSOCIATION_FOREIGNKEY"); foreignKey != "" {
      associationForeignKeys = strings.Split(foreignKey, ",")
    } else if foreignKey, _ := field.TagSettingsGet("ASSOCIATIONFOREIGNKEY"); foreignKey != "" {
      associationForeignKeys = strings.Split(foreignKey, ",")
    }

    for elemType.Kind() == reflect.Slice || elemType.Kind() == reflect.Ptr {
      elemType = elemType.Elem()
    }

    if elemType.Kind() == reflect.Struct {
      if many2many, _ := field.TagSettingsGet("MANY2MANY"); many2many != "" {
        relationship.Kind = "many_to_many"

        { // Foreign Keys for Source
          joinTableDBNames := []string{}

          if foreignKey, _ := field.TagSettingsGet("JOINTABLE_FOREIGNKEY"); foreignKey != "" {
            joinTableDBNames = strings.Split(foreignKey, ",")
          }

          // if no foreign keys defined with tag
          if len(foreignKeys) == 0 {
            for _, field := range modelStruct.PrimaryFields {
              foreignKeys = append(foreignKeys, field.DBName)
            }
          }

          for idx, foreignKey := range foreignKeys {
            if foreignField := getForeignField(foreignKey, modelStruct.StructFields); foreignField != nil {
              // source foreign keys (db names)
              relationship.ForeignFieldNames = append(relationship.ForeignFieldNames, foreignField.DBName)

              // setup join table foreign keys for source
              if len(joinTableDBNames) > idx {
                // if defined join table's foreign key
                relationship.ForeignDBNames = append(relationship.ForeignDBNames, joinTableDBNames[idx])
              } else {
                defaultJointableForeignKey := ToColumnName(reflectType.Name()) + "_" + foreignField.DBName
                relationship.ForeignDBNames = append(relationship.ForeignDBNames, defaultJointableForeignKey)
              }
            }
          }
        }

        { // Foreign Keys for Association (Destination)
          associationJoinTableDBNames := []string{}

          if foreignKey, _ := field.TagSettingsGet("ASSOCIATION_JOINTABLE_FOREIGNKEY"); foreignKey != "" {
            associationJoinTableDBNames = strings.Split(foreignKey, ",")
          }

          // if no association foreign keys defined with tag
          if len(associationForeignKeys) == 0 {
            for _, field := range toScope.PrimaryFields() {
              associationForeignKeys = append(associationForeignKeys, field.DBName)
            }
          }

          for idx, name := range associationForeignKeys {
            if field, ok := toScope.FieldByName(name); ok {
              // association foreign keys (db names)
              relationship.AssociationForeignFieldNames = append(relationship.AssociationForeignFieldNames, field.DBName)

              // setup join table foreign keys for association
              if len(associationJoinTableDBNames) > idx {
                relationship.AssociationForeignDBNames = append(relationship.AssociationForeignDBNames, associationJoinTableDBNames[idx])
              } else {
                // join table foreign keys for association
                joinTableDBName := ToColumnName(elemType.Name()) + "_" + field.DBName
                relationship.AssociationForeignDBNames = append(relationship.AssociationForeignDBNames, joinTableDBName)
              }
            }
          }
        }

        joinTableHandler := JoinTableHandler{}
        joinTableHandler.Setup(relationship, many2many, reflectType, elemType)
        relationship.JoinTableHandler = &joinTableHandler
        field.Relationship = relationship
      } else {
        // User has many comments, associationType is User, comment use UserID as foreign key
        var associationType = reflectType.Name()
        var toFields = toScope.GetStructFields()
        relationship.Kind = "has_many"

        if polymorphic, _ := field.TagSettingsGet("POLYMORPHIC"); polymorphic != "" {
          // Dog has many toys, tag polymorphic is Owner, then associationType is Owner
          // Toy use OwnerID, OwnerType ('dogs') as foreign key
          if polymorphicType := getForeignField(polymorphic+"Type", toFields); polymorphicType != nil {
            associationType = polymorphic
            relationship.PolymorphicType = polymorphicType.Name
            relationship.PolymorphicDBName = polymorphicType.DBName
            // if Dog has multiple set of toys set name of the set (instead of default 'dogs')
            if value, ok := field.TagSettingsGet("POLYMORPHIC_VALUE"); ok {
              relationship.PolymorphicValue = value
            } else {
              relationship.PolymorphicValue = scope.TableName()
            }
            polymorphicType.IsForeignKey = true
          }
        }

        // if no foreign keys defined with tag
        if len(foreignKeys) == 0 {
          // if no association foreign keys defined with tag
          if len(associationForeignKeys) == 0 {
            for _, field := range modelStruct.PrimaryFields {
              foreignKeys = append(foreignKeys, associationType+field.Name)
              associationForeignKeys = append(associationForeignKeys, field.Name)
            }
          } else {
            // generate foreign keys from defined association foreign keys
            for _, scopeFieldName := range associationForeignKeys {
              if foreignField := getForeignField(scopeFieldName, modelStruct.StructFields); foreignField != nil {
                foreignKeys = append(foreignKeys, associationType+foreignField.Name)
                associationForeignKeys = append(associationForeignKeys, foreignField.Name)
              }
            }
          }
        } else {
          // generate association foreign keys from foreign keys
          if len(associationForeignKeys) == 0 {
            for _, foreignKey := range foreignKeys {
              if strings.HasPrefix(foreignKey, associationType) {
                associationForeignKey := strings.TrimPrefix(foreignKey, associationType)
                if foreignField := getForeignField(associationForeignKey, modelStruct.StructFields); foreignField != nil {
                  associationForeignKeys = append(associationForeignKeys, associationForeignKey)
                }
              }
            }
            if len(associationForeignKeys) == 0 && len(foreignKeys) == 1 {
              associationForeignKeys = []string{scope.PrimaryKey()}
            }
          } else if len(foreignKeys) != len(associationForeignKeys) {
            scope.Err(errors.New("invalid foreign keys, should have same length"))
            return
          }
        }

        for idx, foreignKey := range foreignKeys {
          if foreignField := getForeignField(foreignKey, toFields); foreignField != nil {
            if associationField := getForeignField(associationForeignKeys[idx], modelStruct.StructFields); associationField != nil {
              // source foreign keys
              foreignField.IsForeignKey = true
              relationship.AssociationForeignFieldNames = append(relationship.AssociationForeignFieldNames, associationField.Name)
              relationship.AssociationForeignDBNames = append(relationship.AssociationForeignDBNames, associationField.DBName)

              // association foreign keys
              relationship.ForeignFieldNames = append(relationship.ForeignFieldNames, foreignField.Name)
              relationship.ForeignDBNames = append(relationship.ForeignDBNames, foreignField.DBName)
            }
          }
        }

        if len(relationship.ForeignFieldNames) != 0 {
          field.Relationship = relationship
        }
      }
    } else {
      field.IsNormal = true
    }
  }(field)
case reflect.Struct:
  defer func(field *StructField) {
    var (
      // user has one profile, associationType is User, profile use UserID as foreign key
      // user belongs to profile, associationType is Profile, user use ProfileID as foreign key
      associationType           = reflectType.Name()
      relationship              = &Relationship{}
      toScope                   = scope.New(reflect.New(field.Struct.Type).Interface())
      toFields                  = toScope.GetStructFields()
      tagForeignKeys            []string
      tagAssociationForeignKeys []string
    )

    if foreignKey, _ := field.TagSettingsGet("FOREIGNKEY"); foreignKey != "" {
      tagForeignKeys = strings.Split(foreignKey, ",")
    }

    if foreignKey, _ := field.TagSettingsGet("ASSOCIATION_FOREIGNKEY"); foreignKey != "" {
      tagAssociationForeignKeys = strings.Split(foreignKey, ",")
    } else if foreignKey, _ := field.TagSettingsGet("ASSOCIATIONFOREIGNKEY"); foreignKey != "" {
      tagAssociationForeignKeys = strings.Split(foreignKey, ",")
    }

    if polymorphic, _ := field.TagSettingsGet("POLYMORPHIC"); polymorphic != "" {
      // Cat has one toy, tag polymorphic is Owner, then associationType is Owner
      // Toy use OwnerID, OwnerType ('cats') as foreign key
      if polymorphicType := getForeignField(polymorphic+"Type", toFields); polymorphicType != nil {
        associationType = polymorphic
        relationship.PolymorphicType = polymorphicType.Name
        relationship.PolymorphicDBName = polymorphicType.DBName
        // if Cat has several different types of toys set name for each (instead of default 'cats')
        if value, ok := field.TagSettingsGet("POLYMORPHIC_VALUE"); ok {
          relationship.PolymorphicValue = value
        } else {
          relationship.PolymorphicValue = scope.TableName()
        }
        polymorphicType.IsForeignKey = true
      }
    }

    // Has One
    {
      var foreignKeys = tagForeignKeys
      var associationForeignKeys = tagAssociationForeignKeys
      // if no foreign keys defined with tag
      if len(foreignKeys) == 0 {
        // if no association foreign keys defined with tag
        if len(associationForeignKeys) == 0 {
          for _, primaryField := range modelStruct.PrimaryFields {
            foreignKeys = append(foreignKeys, associationType+primaryField.Name)
            associationForeignKeys = append(associationForeignKeys, primaryField.Name)
          }
        } else {
          // generate foreign keys form association foreign keys
          for _, associationForeignKey := range tagAssociationForeignKeys {
            if foreignField := getForeignField(associationForeignKey, modelStruct.StructFields); foreignField != nil {
              foreignKeys = append(foreignKeys, associationType+foreignField.Name)
              associationForeignKeys = append(associationForeignKeys, foreignField.Name)
            }
          }
        }
      } else {
        // generate association foreign keys from foreign keys
        if len(associationForeignKeys) == 0 {
          for _, foreignKey := range foreignKeys {
            if strings.HasPrefix(foreignKey, associationType) {
              associationForeignKey := strings.TrimPrefix(foreignKey, associationType)
              if foreignField := getForeignField(associationForeignKey, modelStruct.StructFields); foreignField != nil {
                associationForeignKeys = append(associationForeignKeys, associationForeignKey)
              }
            }
          }
          if len(associationForeignKeys) == 0 && len(foreignKeys) == 1 {
            associationForeignKeys = []string{scope.PrimaryKey()}
          }
        } else if len(foreignKeys) != len(associationForeignKeys) {
          scope.Err(errors.New("invalid foreign keys, should have same length"))
          return
        }
      }

      for idx, foreignKey := range foreignKeys {
        if foreignField := getForeignField(foreignKey, toFields); foreignField != nil {
          if scopeField := getForeignField(associationForeignKeys[idx], modelStruct.StructFields); scopeField != nil {
            foreignField.IsForeignKey = true
            // source foreign keys
            relationship.AssociationForeignFieldNames = append(relationship.AssociationForeignFieldNames, scopeField.Name)
            relationship.AssociationForeignDBNames = append(relationship.AssociationForeignDBNames, scopeField.DBName)

            // association foreign keys
            relationship.ForeignFieldNames = append(relationship.ForeignFieldNames, foreignField.Name)
            relationship.ForeignDBNames = append(relationship.ForeignDBNames, foreignField.DBName)
          }
        }
      }
    }

    if len(relationship.ForeignFieldNames) != 0 {
      relationship.Kind = "has_one"
      field.Relationship = relationship
    } else {
      var foreignKeys = tagForeignKeys
      var associationForeignKeys = tagAssociationForeignKeys

      if len(foreignKeys) == 0 {
        // generate foreign keys & association foreign keys
        if len(associationForeignKeys) == 0 {
          for _, primaryField := range toScope.PrimaryFields() {
            foreignKeys = append(foreignKeys, field.Name+primaryField.Name)
            associationForeignKeys = append(associationForeignKeys, primaryField.Name)
          }
        } else {
          // generate foreign keys with association foreign keys
          for _, associationForeignKey := range associationForeignKeys {
            if foreignField := getForeignField(associationForeignKey, toFields); foreignField != nil {
              foreignKeys = append(foreignKeys, field.Name+foreignField.Name)
              associationForeignKeys = append(associationForeignKeys, foreignField.Name)
            }
          }
        }
      } else {
        // generate foreign keys & association foreign keys
        if len(associationForeignKeys) == 0 {
          for _, foreignKey := range foreignKeys {
            if strings.HasPrefix(foreignKey, field.Name) {
              associationForeignKey := strings.TrimPrefix(foreignKey, field.Name)
              if foreignField := getForeignField(associationForeignKey, toFields); foreignField != nil {
                associationForeignKeys = append(associationForeignKeys, associationForeignKey)
              }
            }
          }
          if len(associationForeignKeys) == 0 && len(foreignKeys) == 1 {
            associationForeignKeys = []string{toScope.PrimaryKey()}
          }
        } else if len(foreignKeys) != len(associationForeignKeys) {
          scope.Err(errors.New("invalid foreign keys, should have same length"))
          return
        }
      }

      for idx, foreignKey := range foreignKeys {
        if foreignField := getForeignField(foreignKey, modelStruct.StructFields); foreignField != nil {
          if associationField := getForeignField(associationForeignKeys[idx], toFields); associationField != nil {
            foreignField.IsForeignKey = true

            // association foreign keys
            relationship.AssociationForeignFieldNames = append(relationship.AssociationForeignFieldNames, associationField.Name)
            relationship.AssociationForeignDBNames = append(relationship.AssociationForeignDBNames, associationField.DBName)

            // source foreign keys
            relationship.ForeignFieldNames = append(relationship.ForeignFieldNames, foreignField.Name)
            relationship.ForeignDBNames = append(relationship.ForeignDBNames, foreignField.DBName)
          }
        }
      }

      if len(relationship.ForeignFieldNames) != 0 {
        relationship.Kind = "belongs_to"
        field.Relationship = relationship
      }
    }
  }(field)
default:
  field.IsNormal = true
}
```

这部分代码的背景是在解析所有的字段的循环中, 主要是解析模型中的每一个字段, 这部分主要是处理字段的关系.
`field` 就是 `&StructField` 类型, 也就是最后解析后的结果.
`modelStruct` 是 `GetModelStruct` 方法最后的返回值.

```go
var modelStruct ModelStruct

field := &StructField{
  Struct:      fieldStruct,
  Name:        fieldStruct.Name,
  Names:       []string{fieldStruct.Name},
  Tag:         fieldStruct.Tag,
  TagSettings: parseTagSetting(fieldStruct.Tag),
}
```

光是这部分代码, 就已经够长了. 先整体来梳理下结构, 主体是个 `switch`, 根据 `indirectType.Kind()` 类型的不同分别处理不同的情况.
每个 case 下面都是一个 defer 函数.

### case reflect.Slice

先来看一个部分, 即 `case reflect.Slice:`.

```go
var (
  relationship           = &Relationship{}
  toScope                = scope.New(reflect.New(field.Struct.Type).Interface())
  foreignKeys            []string
  associationForeignKeys []string
  elemType               = field.Struct.Type
)
```

定义了所有需要的变量.

```go
if foreignKey, _ := field.TagSettingsGet("FOREIGNKEY"); foreignKey != "" {
  foreignKeys = strings.Split(foreignKey, ",")
}

if foreignKey, _ := field.TagSettingsGet("ASSOCIATION_FOREIGNKEY"); foreignKey != "" {
  associationForeignKeys = strings.Split(foreignKey, ",")
} else if foreignKey, _ := field.TagSettingsGet("ASSOCIATIONFOREIGNKEY"); foreignKey != "" {
  associationForeignKeys = strings.Split(foreignKey, ",")
}

for elemType.Kind() == reflect.Slice || elemType.Kind() == reflect.Ptr {
  elemType = elemType.Elem()
}
```

前面几行代码比较简单, 主要是从 `TagSettings` 中获取 `foreignKeys` 和 `associationForeignKeys`.

最后一段是 if 判断, `if elemType.Kind() == reflect.Struct {`. 如果为 true, 就说明遇到关系了.
否则就是一个普通的字段.

```go
else {
  field.IsNormal = true
}
```

现在来看下关系到底是怎么处理的.

里面也是一个 if 判断, `if many2many, _ := field.TagSettingsGet("MANY2MANY"); many2many != "" {`.
判断是否遇到了 `Many To Many` 关系.

先来看这个 if 为 true 的情况.

```go
relationship.Kind = "many_to_many"
```

设置了关系的类型为 `many_to_many`.

```go
{ // Foreign Keys for Source
  joinTableDBNames := []string{}

  if foreignKey, _ := field.TagSettingsGet("JOINTABLE_FOREIGNKEY"); foreignKey != "" {
    joinTableDBNames = strings.Split(foreignKey, ",")
  }

  // if no foreign keys defined with tag
  if len(foreignKeys) == 0 {
    for _, field := range modelStruct.PrimaryFields {
      foreignKeys = append(foreignKeys, field.DBName)
    }
  }

  for idx, foreignKey := range foreignKeys {
    if foreignField := getForeignField(foreignKey, modelStruct.StructFields); foreignField != nil {
      // source foreign keys (db names)
      relationship.ForeignFieldNames = append(relationship.ForeignFieldNames, foreignField.DBName)

      // setup join table foreign keys for source
      if len(joinTableDBNames) > idx {
        // if defined join table's foreign key
        relationship.ForeignDBNames = append(relationship.ForeignDBNames, joinTableDBNames[idx])
      } else {
        defaultJointableForeignKey := ToColumnName(reflectType.Name()) + "_" + foreignField.DBName
        relationship.ForeignDBNames = append(relationship.ForeignDBNames, defaultJointableForeignKey)
      }
    }
  }
}
```

这段代码主要是在遍历 `foreignKeys`, 将外键的名字添加到 `relationship.ForeignFieldNames` 中.
for 循环的后半部分是添加 `relationship.ForeignDBNames`, 这里有两种情况.
如果已经在 `JOINTABLE_FOREIGNKEY` struct tag 中定义, 就从这边取; 否则就使用默认的格式
`ToColumnName(reflectType.Name()) + "_" + foreignField.DBName`, 即 `表名_字段名`.

```go
func getForeignField(column string, fields []*StructField) *StructField {
	for _, field := range fields {
		if field.Name == column || field.DBName == column || field.DBName == ToColumnName(column) {
			return field
		}
	}
	return nil
}
```

接着看另一部分, 关联外键.

```go
{ // Foreign Keys for Association (Destination)
  associationJoinTableDBNames := []string{}

  if foreignKey, _ := field.TagSettingsGet("ASSOCIATION_JOINTABLE_FOREIGNKEY"); foreignKey != "" {
    associationJoinTableDBNames = strings.Split(foreignKey, ",")
  }

  // if no association foreign keys defined with tag
  if len(associationForeignKeys) == 0 {
    for _, field := range toScope.PrimaryFields() {
      associationForeignKeys = append(associationForeignKeys, field.DBName)
    }
  }

  for idx, name := range associationForeignKeys {
    if field, ok := toScope.FieldByName(name); ok {
      // association foreign keys (db names)
      relationship.AssociationForeignFieldNames = append(relationship.AssociationForeignFieldNames, field.DBName)

      // setup join table foreign keys for association
      if len(associationJoinTableDBNames) > idx {
        relationship.AssociationForeignDBNames = append(relationship.AssociationForeignDBNames, associationJoinTableDBNames[idx])
      } else {
        // join table foreign keys for association
        joinTableDBName := ToColumnName(elemType.Name()) + "_" + field.DBName
        relationship.AssociationForeignDBNames = append(relationship.AssociationForeignDBNames, joinTableDBName)
      }
    }
  }
}
```

这一段的结构和上面的类似, 只不过将 struct tag 从 `JOINTABLE_FOREIGNKEY` 换成了 `ASSOCIATION_JOINTABLE_FOREIGNKEY`.

```go
joinTableHandler := JoinTableHandler{}
joinTableHandler.Setup(relationship, many2many, reflectType, elemType)
relationship.JoinTableHandler = &joinTableHandler
field.Relationship = relationship
```

多对多关系是需要额外的表来存储关系的, 所以这几行代码就是设置连接表的.

看完了 `Many To Many` 关系的情况, 再来看看 else 部分, 即一对多的情况.

```go
// User has many comments, associationType is User, comment use UserID as foreign key
var associationType = reflectType.Name()
var toFields = toScope.GetStructFields()
relationship.Kind = "has_many"
```

关系的类型被设置为 `has_many`.

```go
if polymorphic, _ := field.TagSettingsGet("POLYMORPHIC"); polymorphic != "" {
  // Dog has many toys, tag polymorphic is Owner, then associationType is Owner
  // Toy use OwnerID, OwnerType ('dogs') as foreign key
  if polymorphicType := getForeignField(polymorphic+"Type", toFields); polymorphicType != nil {
    associationType = polymorphic
    relationship.PolymorphicType = polymorphicType.Name
    relationship.PolymorphicDBName = polymorphicType.DBName
    // if Dog has multiple set of toys set name of the set (instead of default 'dogs')
    if value, ok := field.TagSettingsGet("POLYMORPHIC_VALUE"); ok {
      relationship.PolymorphicValue = value
    } else {
      relationship.PolymorphicValue = scope.TableName()
    }
    polymorphicType.IsForeignKey = true
  }
}
```

`Has Many` 和 `Has One` 都是支持多态的, 多态需要两个额外的字段, 一个是 `NameID` 即外键, 另一个是 `NameType` 来区分不同模型.
注意 `associationType = polymorphic` 会将 `associationType` 替换为 `POLYMORPHIC` 中定义的名字.

```go
type Cat struct {
	ID       int
	Name     string
  MainToy  []Toy `gorm:"POLYMORPHIC:Owner;POLYMORPHIC_VALUE:cats_main;"`
  OtherToy []Toy `gorm:"POLYMORPHIC:Owner;POLYMORPHIC_VALUE:cats_other;"`
}

type Dog struct {
	ID   int
	Name string
	Toy  []Toy `gorm:"POLYMORPHIC:Owner;"`
}

type Toy struct {
	ID        int
	Name      string
	OwnerID   int
	OwnerType string
}
```

如果要在同一个模型中定义多个多态, 就需要使用 `POLYMORPHIC_VALUE` 区分, 示例代码如上.

```go
// if no foreign keys defined with tag
if len(foreignKeys) == 0 {
  // if no association foreign keys defined with tag
  if len(associationForeignKeys) == 0 {
    for _, field := range modelStruct.PrimaryFields {
      foreignKeys = append(foreignKeys, associationType+field.Name)
      associationForeignKeys = append(associationForeignKeys, field.Name)
    }
  } else {
    // generate foreign keys from defined association foreign keys
    for _, scopeFieldName := range associationForeignKeys {
      if foreignField := getForeignField(scopeFieldName, modelStruct.StructFields); foreignField != nil {
        foreignKeys = append(foreignKeys, associationType+foreignField.Name)
        associationForeignKeys = append(associationForeignKeys, foreignField.Name)
      }
    }
  }
} else {
  // generate association foreign keys from foreign keys
  if len(associationForeignKeys) == 0 {
    for _, foreignKey := range foreignKeys {
      if strings.HasPrefix(foreignKey, associationType) {
        associationForeignKey := strings.TrimPrefix(foreignKey, associationType)
        if foreignField := getForeignField(associationForeignKey, modelStruct.StructFields); foreignField != nil {
          associationForeignKeys = append(associationForeignKeys, associationForeignKey)
        }
      }
    }
    if len(associationForeignKeys) == 0 && len(foreignKeys) == 1 {
      associationForeignKeys = []string{scope.PrimaryKey()}
    }
  } else if len(foreignKeys) != len(associationForeignKeys) {
    scope.Err(errors.New("invalid foreign keys, should have same length"))
    return
  }
}
```

这部分代码主要是在获取 `foreignKeys` 和 `associationForeignKeys`.

```go
for idx, foreignKey := range foreignKeys {
  if foreignField := getForeignField(foreignKey, toFields); foreignField != nil {
    if associationField := getForeignField(associationForeignKeys[idx], modelStruct.StructFields); associationField != nil {
      // source foreign keys
      foreignField.IsForeignKey = true
      relationship.AssociationForeignFieldNames = append(relationship.AssociationForeignFieldNames, associationField.Name)
      relationship.AssociationForeignDBNames = append(relationship.AssociationForeignDBNames, associationField.DBName)

      // association foreign keys
      relationship.ForeignFieldNames = append(relationship.ForeignFieldNames, foreignField.Name)
      relationship.ForeignDBNames = append(relationship.ForeignDBNames, foreignField.DBName)
    }
  }
}

if len(relationship.ForeignFieldNames) != 0 {
  field.Relationship = relationship
}
```

前面获取好 `foreignKeys` 和 `associationForeignKeys` 之后, 这里就会将对应的字段设置到 `relationship` 上.
最后, 如果 `relationship.ForeignFieldNames` 不为空, 就会将 `relationship` 设置到 `field.Relationship` 上.

以上就是 `case reflect.Slice:` 下的情况, 主要是处理了 `Many To Many` 和 `Has Many` 两种情况.
这和模型定义是符合的, 定义这两种关系时都会使用结构体的切片.

接下来看看另外的两种关系类型.

### case reflect.Struct

进入到 `case reflect.Struct:` 中, 看看剩余的两种关系类型 `Belongs To` 和 `Has One` 是如何处理的.

```go
var (
  // user has one profile, associationType is User, profile use UserID as foreign key
  // user belongs to profile, associationType is Profile, user use ProfileID as foreign key
  associationType           = reflectType.Name()
  relationship              = &Relationship{}
  toScope                   = scope.New(reflect.New(field.Struct.Type).Interface())
  toFields                  = toScope.GetStructFields()
  tagForeignKeys            []string
  tagAssociationForeignKeys []string
)
```

`Belongs To` 和 `Has One` 的区别在于外键定义在哪个模型中. 前面的两句注释也很好的解释了这种情况.

```go
if foreignKey, _ := field.TagSettingsGet("FOREIGNKEY"); foreignKey != "" {
  tagForeignKeys = strings.Split(foreignKey, ",")
}

if foreignKey, _ := field.TagSettingsGet("ASSOCIATION_FOREIGNKEY"); foreignKey != "" {
  tagAssociationForeignKeys = strings.Split(foreignKey, ",")
} else if foreignKey, _ := field.TagSettingsGet("ASSOCIATIONFOREIGNKEY"); foreignKey != "" {
  tagAssociationForeignKeys = strings.Split(foreignKey, ",")
}
```

获取了 `tagForeignKeys` 和 `tagAssociationForeignKeys`.

```go
if polymorphic, _ := field.TagSettingsGet("POLYMORPHIC"); polymorphic != "" {
  // Cat has one toy, tag polymorphic is Owner, then associationType is Owner
  // Toy use OwnerID, OwnerType ('cats') as foreign key
  if polymorphicType := getForeignField(polymorphic+"Type", toFields); polymorphicType != nil {
    associationType = polymorphic
    relationship.PolymorphicType = polymorphicType.Name
    relationship.PolymorphicDBName = polymorphicType.DBName
    // if Cat has several different types of toys set name for each (instead of default 'cats')
    if value, ok := field.TagSettingsGet("POLYMORPHIC_VALUE"); ok {
      relationship.PolymorphicValue = value
    } else {
      relationship.PolymorphicValue = scope.TableName()
    }
    polymorphicType.IsForeignKey = true
  }
}
```

`Has One` 的多态和 `Has Many` 类似.

```go
// Has One
{
  var foreignKeys = tagForeignKeys
  var associationForeignKeys = tagAssociationForeignKeys
  // if no foreign keys defined with tag
  if len(foreignKeys) == 0 {
    // if no association foreign keys defined with tag
    if len(associationForeignKeys) == 0 {
      for _, primaryField := range modelStruct.PrimaryFields {
        foreignKeys = append(foreignKeys, associationType+primaryField.Name)
        associationForeignKeys = append(associationForeignKeys, primaryField.Name)
      }
    } else {
      // generate foreign keys form association foreign keys
      for _, associationForeignKey := range tagAssociationForeignKeys {
        if foreignField := getForeignField(associationForeignKey, modelStruct.StructFields); foreignField != nil {
          foreignKeys = append(foreignKeys, associationType+foreignField.Name)
          associationForeignKeys = append(associationForeignKeys, foreignField.Name)
        }
      }
    }
  } else {
    // generate association foreign keys from foreign keys
    if len(associationForeignKeys) == 0 {
      for _, foreignKey := range foreignKeys {
        if strings.HasPrefix(foreignKey, associationType) {
          associationForeignKey := strings.TrimPrefix(foreignKey, associationType)
          if foreignField := getForeignField(associationForeignKey, modelStruct.StructFields); foreignField != nil {
            associationForeignKeys = append(associationForeignKeys, associationForeignKey)
          }
        }
      }
      if len(associationForeignKeys) == 0 && len(foreignKeys) == 1 {
        associationForeignKeys = []string{scope.PrimaryKey()}
      }
    } else if len(foreignKeys) != len(associationForeignKeys) {
      scope.Err(errors.New("invalid foreign keys, should have same length"))
      return
    }
  }

  for idx, foreignKey := range foreignKeys {
    if foreignField := getForeignField(foreignKey, toFields); foreignField != nil {
      if scopeField := getForeignField(associationForeignKeys[idx], modelStruct.StructFields); scopeField != nil {
        foreignField.IsForeignKey = true
        // source foreign keys
        relationship.AssociationForeignFieldNames = append(relationship.AssociationForeignFieldNames, scopeField.Name)
        relationship.AssociationForeignDBNames = append(relationship.AssociationForeignDBNames, scopeField.DBName)

        // association foreign keys
        relationship.ForeignFieldNames = append(relationship.ForeignFieldNames, foreignField.Name)
        relationship.ForeignDBNames = append(relationship.ForeignDBNames, foreignField.DBName)
      }
    }
  }
}
```

这段代码被标记为 `Has One`. 主要也是在获取 `relationship` 的四个关键字段,
`AssociationForeignFieldNames`, `AssociationForeignDBNames`,
`ForeignFieldNames`, `ForeignDBNames`.

最后一段是个 if 判断, 主要用于区分是 `Has One` 还是 `Belongs To`.

```go
if len(relationship.ForeignFieldNames) != 0 {
  relationship.Kind = "has_one"
  field.Relationship = relationship
}
```

如果前面的那段被标记为 `Has One` 的代码起作用了, 即 `relationship.ForeignFieldNames` 不为空,
那么就会可以设置关系类型为 `has_one` 了.

否则就进入 else 部分.

```go
var foreignKeys = tagForeignKeys
var associationForeignKeys = tagAssociationForeignKeys

if len(foreignKeys) == 0 {
  // generate foreign keys & association foreign keys
  if len(associationForeignKeys) == 0 {
    for _, primaryField := range toScope.PrimaryFields() {
      foreignKeys = append(foreignKeys, field.Name+primaryField.Name)
      associationForeignKeys = append(associationForeignKeys, primaryField.Name)
    }
  } else {
    // generate foreign keys with association foreign keys
    for _, associationForeignKey := range associationForeignKeys {
      if foreignField := getForeignField(associationForeignKey, toFields); foreignField != nil {
        foreignKeys = append(foreignKeys, field.Name+foreignField.Name)
        associationForeignKeys = append(associationForeignKeys, foreignField.Name)
      }
    }
  }
} else {
  // generate foreign keys & association foreign keys
  if len(associationForeignKeys) == 0 {
    for _, foreignKey := range foreignKeys {
      if strings.HasPrefix(foreignKey, field.Name) {
        associationForeignKey := strings.TrimPrefix(foreignKey, field.Name)
        if foreignField := getForeignField(associationForeignKey, toFields); foreignField != nil {
          associationForeignKeys = append(associationForeignKeys, associationForeignKey)
        }
      }
    }
    if len(associationForeignKeys) == 0 && len(foreignKeys) == 1 {
      associationForeignKeys = []string{toScope.PrimaryKey()}
    }
  } else if len(foreignKeys) != len(associationForeignKeys) {
    scope.Err(errors.New("invalid foreign keys, should have same length"))
    return
  }
}

for idx, foreignKey := range foreignKeys {
  if foreignField := getForeignField(foreignKey, modelStruct.StructFields); foreignField != nil {
    if associationField := getForeignField(associationForeignKeys[idx], toFields); associationField != nil {
      foreignField.IsForeignKey = true

      // association foreign keys
      relationship.AssociationForeignFieldNames = append(relationship.AssociationForeignFieldNames, associationField.Name)
      relationship.AssociationForeignDBNames = append(relationship.AssociationForeignDBNames, associationField.DBName)

      // source foreign keys
      relationship.ForeignFieldNames = append(relationship.ForeignFieldNames, foreignField.Name)
      relationship.ForeignDBNames = append(relationship.ForeignDBNames, foreignField.DBName)
    }
  }
}

if len(relationship.ForeignFieldNames) != 0 {
  relationship.Kind = "belongs_to"
  field.Relationship = relationship
}
```

结构其实和前面的 `Has One` 部分是类似的, 主要的区别在于这里使用的是 `toScope`, 前面使用的是 `modelStruct`.

```go
// 来自 Belongs To 部分
if len(foreignKeys) == 0 {
  // generate foreign keys & association foreign keys
  if len(associationForeignKeys) == 0 {
    for _, primaryField := range toScope.PrimaryFields() {
      foreignKeys = append(foreignKeys, field.Name+primaryField.Name)
      associationForeignKeys = append(associationForeignKeys, primaryField.Name)
    }
  }

// 定义 Belongs To 关系
type User struct {
  gorm.Model
  Name string
}

// `Profile` 属于 `User`， 外键是`UserID`
type Profile struct {
  gorm.Model
  UserID int
  User   User
  Name   string
}
```

以 `Belongs To` 为例, 现在解析的是 `Profile` 模型的 `User` 字段,
循环的是 `toScope` 的主键, `toScope` 对应的是 `User` 字段的类型, 即 `User` 的主键.
所以 `foreignKeys` 中是 `"User" + "ID"` 即 `UserID`,
`associationForeignKeys` 中是 `ID`.

```go
// 来自 Has One 部分
if len(foreignKeys) == 0 {
  // if no association foreign keys defined with tag
  if len(associationForeignKeys) == 0 {
    for _, primaryField := range modelStruct.PrimaryFields {
      foreignKeys = append(foreignKeys, associationType+primaryField.Name)
      associationForeignKeys = append(associationForeignKeys, primaryField.Name)
    }
  }

// 定义 Has One 关系
// User 只能有一张信用卡 (CreditCard), CreditCardID 是外键
type CreditCard struct {
  gorm.Model
  Number   string
  UserID   uint
}

type User struct {
  gorm.Model
  CreditCard   CreditCard
}
```

以 `Has One` 为例, 解析的是 `User` 模型的 `CreditCard` 字段,
循环的是 `modelStruct` 的主键, `modelStruct` 对应的是当前的模型, 即 `User` 的主键.
所以 `foreignKeys` 中依然是 `"User" + "ID"` 即 `UserID`,
`associationForeignKeys` 中是 `ID`.

通过对比, 应该对 `Belongs To` 和 `Has One` 处理过程中的差别有了更深的理解.
`Belongs To` 解析的是当前字段对应的类型, 比如 `User User` 中的类型为 `User`.
`Has One` 解析的是当前模型, 所以包含 `CreditCard CreditCard` 的模型为 `User`.

### 小结

通过重温 `GetModelStruct` 方法, 我们对模型是如何解析关系有了更深的理解.

## JoinTableHandler

```go
joinTableHandler := JoinTableHandler{}
joinTableHandler.Setup(relationship, many2many, reflectType, elemType)
relationship.JoinTableHandler = &joinTableHandler
field.Relationship = relationship
```

前面看 `Many To Many` 关系解析的时候, 忽略了设置连接表的过程, 这里回顾一下.

```go
// JoinTableHandlerInterface is an interface for how to handle many2many relations
type JoinTableHandlerInterface interface {
	// initialize join table handler
	Setup(relationship *Relationship, tableName string, source reflect.Type, destination reflect.Type)
	// Table return join table's table name
	Table(db *DB) string
	// Add create relationship in join table for source and destination
	Add(handler JoinTableHandlerInterface, db *DB, source interface{}, destination interface{}) error
	// Delete delete relationship in join table for sources
	Delete(handler JoinTableHandlerInterface, db *DB, sources ...interface{}) error
	// JoinWith query with `Join` conditions
	JoinWith(handler JoinTableHandlerInterface, db *DB, source interface{}) *DB
	// SourceForeignKeys return source foreign keys
	SourceForeignKeys() []JoinTableForeignKey
	// DestinationForeignKeys return destination foreign keys
	DestinationForeignKeys() []JoinTableForeignKey
}

// JoinTableForeignKey join table foreign key struct
type JoinTableForeignKey struct {
	DBName            string
	AssociationDBName string
}

// JoinTableSource is a struct that contains model type and foreign keys
type JoinTableSource struct {
	ModelType   reflect.Type
	ForeignKeys []JoinTableForeignKey
}

// JoinTableHandler default join table handler
type JoinTableHandler struct {
	TableName   string          `sql:"-"`
	Source      JoinTableSource `sql:"-"`
	Destination JoinTableSource `sql:"-"`
}
```

相关的结构如上, 主要结构是 `JoinTableHandler`, 实现了 `JoinTableHandlerInterface` 接口.

看一下 `Setup` 方法:

```go
// Setup initialize a default join table handler
func (s *JoinTableHandler) Setup(relationship *Relationship, tableName string, source reflect.Type, destination reflect.Type) {
	s.TableName = tableName

	s.Source = JoinTableSource{ModelType: source}
	s.Source.ForeignKeys = []JoinTableForeignKey{}
	for idx, dbName := range relationship.ForeignFieldNames {
		s.Source.ForeignKeys = append(s.Source.ForeignKeys, JoinTableForeignKey{
			DBName:            relationship.ForeignDBNames[idx],
			AssociationDBName: dbName,
		})
	}

	s.Destination = JoinTableSource{ModelType: destination}
	s.Destination.ForeignKeys = []JoinTableForeignKey{}
	for idx, dbName := range relationship.AssociationForeignFieldNames {
		s.Destination.ForeignKeys = append(s.Destination.ForeignKeys, JoinTableForeignKey{
			DBName:            relationship.AssociationForeignDBNames[idx],
			AssociationDBName: dbName,
		})
	}
}
```

主要作用是初始化 `JoinTableHandler`, 基于 `relationship` 中的 `ForeignFieldNames` 和 `AssociationForeignFieldNames`.

顺带也看一下它的 `JoinWith` 方法.

```go
// JoinWith query with `Join` conditions
func (s JoinTableHandler) JoinWith(handler JoinTableHandlerInterface, db *DB, source interface{}) *DB {
	var (
		scope           = db.NewScope(source)
		tableName       = handler.Table(db)
		quotedTableName = scope.Quote(tableName)
		joinConditions  []string
		values          []interface{}
	)

	if s.Source.ModelType == scope.GetModelStruct().ModelType {
		destinationTableName := db.NewScope(reflect.New(s.Destination.ModelType).Interface()).QuotedTableName()
		for _, foreignKey := range s.Destination.ForeignKeys {
			joinConditions = append(joinConditions, fmt.Sprintf("%v.%v = %v.%v", quotedTableName, scope.Quote(foreignKey.DBName), destinationTableName, scope.Quote(foreignKey.AssociationDBName)))
		}

		var foreignDBNames []string
		var foreignFieldNames []string

		for _, foreignKey := range s.Source.ForeignKeys {
			foreignDBNames = append(foreignDBNames, foreignKey.DBName)
			if field, ok := scope.FieldByName(foreignKey.AssociationDBName); ok {
				foreignFieldNames = append(foreignFieldNames, field.Name)
			}
		}

		foreignFieldValues := scope.getColumnAsArray(foreignFieldNames, scope.Value)

		var condString string
		if len(foreignFieldValues) > 0 {
			var quotedForeignDBNames []string
			for _, dbName := range foreignDBNames {
				quotedForeignDBNames = append(quotedForeignDBNames, tableName+"."+dbName)
			}

			condString = fmt.Sprintf("%v IN (%v)", toQueryCondition(scope, quotedForeignDBNames), toQueryMarks(foreignFieldValues))

			keys := scope.getColumnAsArray(foreignFieldNames, scope.Value)
			values = append(values, toQueryValues(keys))
		} else {
			condString = fmt.Sprintf("1 <> 1")
		}

		return db.Joins(fmt.Sprintf("INNER JOIN %v ON %v", quotedTableName, strings.Join(joinConditions, " AND "))).
			Where(condString, toQueryValues(foreignFieldValues)...)
	}

	db.Error = errors.New("wrong source type for join table handler")
	return db
}
```

`JoinWith` 用来在查询的时候实现 `Join` 条件.

```go
var (
  scope           = db.NewScope(source)
  tableName       = handler.Table(db)
  quotedTableName = scope.Quote(tableName)
  joinConditions  []string
  values          []interface{}
)

if s.Source.ModelType == scope.GetModelStruct().ModelType {
  ...
}

db.Error = errors.New("wrong source type for join table handler")
return db
```

主要的内容在 if 中进行, 否则就是直接设置一个错误.

```go
destinationTableName := db.NewScope(reflect.New(s.Destination.ModelType).Interface()).QuotedTableName()
for _, foreignKey := range s.Destination.ForeignKeys {
  joinConditions = append(joinConditions, fmt.Sprintf("%v.%v = %v.%v", quotedTableName, scope.Quote(foreignKey.DBName), destinationTableName, scope.Quote(foreignKey.AssociationDBName)))
}
```

获取 `joinConditions`.

```go
var foreignDBNames []string
var foreignFieldNames []string

for _, foreignKey := range s.Source.ForeignKeys {
  foreignDBNames = append(foreignDBNames, foreignKey.DBName)
  if field, ok := scope.FieldByName(foreignKey.AssociationDBName); ok {
    foreignFieldNames = append(foreignFieldNames, field.Name)
  }
}
```

获取 `foreignDBNames` 和 `foreignFieldNames`.

```go
foreignFieldValues := scope.getColumnAsArray(foreignFieldNames, scope.Value)

var condString string
if len(foreignFieldValues) > 0 {
  var quotedForeignDBNames []string
  for _, dbName := range foreignDBNames {
    quotedForeignDBNames = append(quotedForeignDBNames, tableName+"."+dbName)
  }

  condString = fmt.Sprintf("%v IN (%v)", toQueryCondition(scope, quotedForeignDBNames), toQueryMarks(foreignFieldValues))

  keys := scope.getColumnAsArray(foreignFieldNames, scope.Value)
  values = append(values, toQueryValues(keys))
} else {
  condString = fmt.Sprintf("1 <> 1")
}
```

获取 `condString`.

```go
return db.Joins(fmt.Sprintf("INNER JOIN %v ON %v", quotedTableName, strings.Join(joinConditions, " AND "))).
  Where(condString, toQueryValues(foreignFieldValues)...)
```

最后就是拼接 SQL, 使用的是 `INNER JOIN`, 也需要使用 `Where` 带上条件.

## 总结

这部分主要探索了模型定义中关系是如何解析的.
