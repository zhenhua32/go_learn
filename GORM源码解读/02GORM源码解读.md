<!-- TOC -->

- [简介](#简介)
- [定义模型](#定义模型)
- [ModelStruct](#modelstruct)
  - [获取表名](#获取表名)
  - [StructField](#structfield)
  - [Relationship](#relationship)
  - [更多](#更多)
- [Scope](#scope)
  - [模型解析](#模型解析)
  - [字段解析](#字段解析)
  - [小结](#小结)
- [总结](#总结)

<!-- /TOC -->

## 简介

GORM 源码解读, 基于 [v1.9.11](https://github.com/jinzhu/gorm/tree/v1.9.11) 版本.

## 定义模型

GORM 是 ORM, 所以模型定义是最重要的部分, 这一次来探究下具体实现.

```go
type User struct {
  gorm.Model
  Name         string
  Age          sql.NullInt64
  Birthday     *time.Time
  Email        string  `gorm:"type:varchar(100);unique_index"`
  Role         string  `gorm:"size:255"` // 设置字段大小为255
  MemberNumber *string `gorm:"unique;not null"` // 设置会员号（member number）唯一并且不为空
  Num          int     `gorm:"AUTO_INCREMENT"` // 设置 num 为自增类型
  Address      string  `gorm:"index:addr"` // 给address字段创建名为addr的索引
  IgnoreMe     int     `gorm:"-"` // 忽略本字段
}
```

这是官方文档上的一个模型定义. 和普通的结构体类似, 但多了属于 gorm 的 tags.

所有的模型都*应该*包含 `gorm.Model`, 看一下它的定义:

```go
// Model base model definition, including fields `ID`, `CreatedAt`, `UpdatedAt`, `DeletedAt`, which could be embedded in your models
//    type User struct {
//      gorm.Model
//    }
type Model struct {
	ID        uint `gorm:"primary_key"`
	CreatedAt time.Time
	UpdatedAt time.Time
	DeletedAt *time.Time `sql:"index"`
}
```

当然, 这并不是强制要求, 也可以不包含 `gorm.Model`, 它只是定义了一些非常基础且实用的字段.

定义表的时候, 文档上介绍了很多预设, 比如默认 ID 是表的主键, 表名是结构体名称的复数等.

## ModelStruct

要深入了解模型定义, 要从 `ModelStruct` 开始:

```go
// ModelStruct model definition
type ModelStruct struct {
	PrimaryFields []*StructField
	StructFields  []*StructField
	ModelType     reflect.Type

	defaultTableName string
	l                sync.Mutex
}
```

`ModelStruct` 定义了模型结构体的轮廓, 包含主键字段的切片, 普通字段的切片, 模型类型, 默认表名.

### 获取表名

`ModelStruct` 有一个方法获取模型的表名, 看一下它的具体代码:

```go
// TableName returns model's table name
func (s *ModelStruct) TableName(db *DB) string {
	s.l.Lock()
	defer s.l.Unlock()

	if s.defaultTableName == "" && db != nil && s.ModelType != nil {
		// Set default table name
		if tabler, ok := reflect.New(s.ModelType).Interface().(tabler); ok {
			s.defaultTableName = tabler.TableName()
		} else {
			tableName := ToTableName(s.ModelType.Name())
			db.parent.RLock()
			if db == nil || (db.parent != nil && !db.parent.singularTable) {
				tableName = inflection.Plural(tableName)
			}
			db.parent.RUnlock()
			s.defaultTableName = tableName
		}
	}

	return DefaultTableNameHandler(db, s.defaultTableName)
}
```

首先, 使用反射检查是否实现了 `tabler` 接口, 如果实现了, 直接调用 `TableName()` 方法;
没有实现就使用 `ToTableName` 转换表名, 有条件地将表名转换为复数形式;
最后一步, 对于所有的表名使用 `DefaultTableNameHandler` 钩子函数进行再次转换.

看过源码之后, 就能更好的理解文档上关于表名的说明了.

### StructField

看一下 `StructField` 的定义, 即表中的字段是如何表示的:

```go
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

定义了很多字段, 从字段的名字中可以猜测出很多信息, 比如该字段是否是主键等.

注意到有个 `TagSettings` 字段, 以及配套的 `tagSettingsLock` 读写锁.

```go
// TagSettingsSet Sets a tag in the tag settings map
func (sf *StructField) TagSettingsSet(key, val string) {
	sf.tagSettingsLock.Lock()
	defer sf.tagSettingsLock.Unlock()
	sf.TagSettings[key] = val
}

// TagSettingsGet returns a tag from the tag settings
func (sf *StructField) TagSettingsGet(key string) (string, bool) {
	sf.tagSettingsLock.RLock()
	defer sf.tagSettingsLock.RUnlock()
	val, ok := sf.TagSettings[key]
	return val, ok
}

// TagSettingsDelete deletes a tag
func (sf *StructField) TagSettingsDelete(key string) {
	sf.tagSettingsLock.Lock()
	defer sf.tagSettingsLock.Unlock()
	delete(sf.TagSettings, key)
}
```

这些方法都是和 `TagSettings` 有关的, 也可以看作是读写锁 `sync.RWMutex` 的使用范例.

最后一个方法是关于复制结构体的.

```go
func (sf *StructField) clone() *StructField {
	clone := &StructField{
		DBName:          sf.DBName,
		Name:            sf.Name,
		Names:           sf.Names,
		IsPrimaryKey:    sf.IsPrimaryKey,
		IsNormal:        sf.IsNormal,
		IsIgnored:       sf.IsIgnored,
		IsScanner:       sf.IsScanner,
		HasDefaultValue: sf.HasDefaultValue,
		Tag:             sf.Tag,
		TagSettings:     map[string]string{},
		Struct:          sf.Struct,
		IsForeignKey:    sf.IsForeignKey,
	}

	if sf.Relationship != nil {
		relationship := *sf.Relationship
		clone.Relationship = &relationship
	}

	// copy the struct field tagSettings, they should be read-locked while they are copied
	sf.tagSettingsLock.Lock()
	defer sf.tagSettingsLock.Unlock()
	for key, value := range sf.TagSettings {
		clone.TagSettings[key] = value
	}

	return clone
}
```

复制 `tagSettingsLock` 中的字段时, 也用到了读锁.

### Relationship

结构体 `Relationship` 定义了关系类型.

```go
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

### 更多

在继续探索如何解析模型定义之前, 先来了解一下 `Scope` 结构体.

## Scope

```go
// Scope contain current operation's information when you perform any operation on the database
type Scope struct {
	Search          *search
	Value           interface{}
	SQL             string
	SQLVars         []interface{}
	db              *DB
	instanceID      string
	primaryKeyField *Field
	skipLeft        bool
	fields          *[]*Field
	selectAttrs     *[]string
}
```

`Scope` 是非常重要的一部分, 注释中写道, 当你在数据库上执行任何操作时, `Scope` 都会记录当前操作的信息.

```go
// IndirectValue return scope's reflect value's indirect value
func (scope *Scope) IndirectValue() reflect.Value {
	return indirect(reflect.ValueOf(scope.Value))
}

func indirect(reflectValue reflect.Value) reflect.Value {
	for reflectValue.Kind() == reflect.Ptr {
		reflectValue = reflectValue.Elem()
	}
	return reflectValue
}

// New create a new Scope without search information
func (scope *Scope) New(value interface{}) *Scope {
	return &Scope{db: scope.NewDB(), Search: &search{}, Value: value}
}

// NewDB create a new DB without search information
func (scope *Scope) NewDB() *DB {
	if scope.db != nil {
		db := scope.db.clone()
		db.search = nil
		db.Value = nil
		return db
	}
	return nil
}
```

`Scope` 下有很多方法, 先暂时不看. 对它的结构有所了解之后, 回到模型解析上来.

### 模型解析

用户定义模型之后, 就需要解析模型, 而这个工作是在 `Scope` 范围内完成的, 所以是其上的方法.

代码很长, 先略览它个大概, 感受一下整体结构.

```go
// GetModelStruct get value's model struct, relationships based on struct and tag definition
func (scope *Scope) GetModelStruct() *ModelStruct {
	var modelStruct ModelStruct
	// Scope value can't be nil
	if scope.Value == nil {
		return &modelStruct
	}

	reflectType := reflect.ValueOf(scope.Value).Type()
	for reflectType.Kind() == reflect.Slice || reflectType.Kind() == reflect.Ptr {
		reflectType = reflectType.Elem()
	}

	// Scope value need to be a struct
	if reflectType.Kind() != reflect.Struct {
		return &modelStruct
	}

	// Get Cached model struct
	isSingularTable := false
	if scope.db != nil && scope.db.parent != nil {
		scope.db.parent.RLock()
		isSingularTable = scope.db.parent.singularTable
		scope.db.parent.RUnlock()
	}

	hashKey := struct {
		singularTable bool
		reflectType   reflect.Type
	}{isSingularTable, reflectType}
	if value, ok := modelStructsMap.Load(hashKey); ok && value != nil {
		return value.(*ModelStruct)
	}

	modelStruct.ModelType = reflectType

	// Get all fields
	for i := 0; i < reflectType.NumField(); i++ {
		if fieldStruct := reflectType.Field(i); ast.IsExported(fieldStruct.Name) {
			field := &StructField{
				Struct:      fieldStruct,
				Name:        fieldStruct.Name,
				Names:       []string{fieldStruct.Name},
				Tag:         fieldStruct.Tag,
				TagSettings: parseTagSetting(fieldStruct.Tag),
			}

			// is ignored field
			if _, ok := field.TagSettingsGet("-"); ok {
				field.IsIgnored = true
			} else {
				if _, ok := field.TagSettingsGet("PRIMARY_KEY"); ok {
					field.IsPrimaryKey = true
					modelStruct.PrimaryFields = append(modelStruct.PrimaryFields, field)
				}

				if _, ok := field.TagSettingsGet("DEFAULT"); ok && !field.IsPrimaryKey {
					field.HasDefaultValue = true
				}

				if _, ok := field.TagSettingsGet("AUTO_INCREMENT"); ok && !field.IsPrimaryKey {
					field.HasDefaultValue = true
				}

				indirectType := fieldStruct.Type
				for indirectType.Kind() == reflect.Ptr {
					indirectType = indirectType.Elem()
				}

				fieldValue := reflect.New(indirectType).Interface()
				if _, isScanner := fieldValue.(sql.Scanner); isScanner {
					// is scanner
					field.IsScanner, field.IsNormal = true, true
					if indirectType.Kind() == reflect.Struct {
						for i := 0; i < indirectType.NumField(); i++ {
							for key, value := range parseTagSetting(indirectType.Field(i).Tag) {
								if _, ok := field.TagSettingsGet(key); !ok {
									field.TagSettingsSet(key, value)
								}
							}
						}
					}
				} else if _, isTime := fieldValue.(*time.Time); isTime {
					// is time
					field.IsNormal = true
				} else if _, ok := field.TagSettingsGet("EMBEDDED"); ok || fieldStruct.Anonymous {
					// is embedded struct
					for _, subField := range scope.New(fieldValue).GetModelStruct().StructFields {
						subField = subField.clone()
						subField.Names = append([]string{fieldStruct.Name}, subField.Names...)
						if prefix, ok := field.TagSettingsGet("EMBEDDED_PREFIX"); ok {
							subField.DBName = prefix + subField.DBName
						}

						if subField.IsPrimaryKey {
							if _, ok := subField.TagSettingsGet("PRIMARY_KEY"); ok {
								modelStruct.PrimaryFields = append(modelStruct.PrimaryFields, subField)
							} else {
								subField.IsPrimaryKey = false
							}
						}

						if subField.Relationship != nil && subField.Relationship.JoinTableHandler != nil {
							if joinTableHandler, ok := subField.Relationship.JoinTableHandler.(*JoinTableHandler); ok {
								newJoinTableHandler := &JoinTableHandler{}
								newJoinTableHandler.Setup(subField.Relationship, joinTableHandler.TableName, reflectType, joinTableHandler.Destination.ModelType)
								subField.Relationship.JoinTableHandler = newJoinTableHandler
							}
						}

						modelStruct.StructFields = append(modelStruct.StructFields, subField)
					}
					continue
				} else {
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
				}
			}

			// Even it is ignored, also possible to decode db value into the field
			if value, ok := field.TagSettingsGet("COLUMN"); ok {
				field.DBName = value
			} else {
				field.DBName = ToColumnName(fieldStruct.Name)
			}

			modelStruct.StructFields = append(modelStruct.StructFields, field)
		}
	}

	if len(modelStruct.PrimaryFields) == 0 {
		if field := getForeignField("id", modelStruct.StructFields); field != nil {
			field.IsPrimaryKey = true
			modelStruct.PrimaryFields = append(modelStruct.PrimaryFields, field)
		}
	}

	modelStructsMap.Store(hashKey, &modelStruct)

	return &modelStruct
}
```

其实首先折叠一下中间的 for 循环会好很多.

```go
// GetModelStruct get value's model struct, relationships based on struct and tag definition
func (scope *Scope) GetModelStruct() *ModelStruct {
	var modelStruct ModelStruct
	// Scope value can't be nil
	if scope.Value == nil {
		return &modelStruct
	}

	reflectType := reflect.ValueOf(scope.Value).Type()
	for reflectType.Kind() == reflect.Slice || reflectType.Kind() == reflect.Ptr {
		reflectType = reflectType.Elem()
	}

	// Scope value need to be a struct
	if reflectType.Kind() != reflect.Struct {
		return &modelStruct
	}

	// Get Cached model struct
	isSingularTable := false
	if scope.db != nil && scope.db.parent != nil {
		scope.db.parent.RLock()
		isSingularTable = scope.db.parent.singularTable
		scope.db.parent.RUnlock()
	}

	hashKey := struct {
		singularTable bool
		reflectType   reflect.Type
	}{isSingularTable, reflectType}
	if value, ok := modelStructsMap.Load(hashKey); ok && value != nil {
		return value.(*ModelStruct)
	}

  modelStruct.ModelType = reflectType

  // Get all fields
	for i := 0; i < reflectType.NumField(); i++ {
    ... // 折叠先不看
  }

  if len(modelStruct.PrimaryFields) == 0 {
		if field := getForeignField("id", modelStruct.StructFields); field != nil {
			field.IsPrimaryKey = true
			modelStruct.PrimaryFields = append(modelStruct.PrimaryFields, field)
		}
	}

	modelStructsMap.Store(hashKey, &modelStruct)

  return &modelStruct
}
```

开头初始化了 `var modelStruct ModelStruct`, 这也是最后要返回的结果.

一开始先判断了 `scope.Value` 不能为空, 否则就直接返回.

然后解析了 `scope.Value` 的具体类型, 对于切片或指针, 要看具

```go
reflectType := reflect.ValueOf(scope.Value).Type()
for reflectType.Kind() == reflect.Slice || reflectType.Kind() == reflect.Ptr {
  reflectType = reflectType.Elem()
}
```

如果 `scope.Value` 的具体类型不是 Struct, 也是直接返回.

然后, 判断是否有 `ModelStruct` 的缓存:

```go
// Get Cached model struct
isSingularTable := false
if scope.db != nil && scope.db.parent != nil {
  scope.db.parent.RLock()
  isSingularTable = scope.db.parent.singularTable
  scope.db.parent.RUnlock()
}

hashKey := struct {
  singularTable bool
  reflectType   reflect.Type
}{isSingularTable, reflectType}
if value, ok := modelStructsMap.Load(hashKey); ok && value != nil {
  return value.(*ModelStruct)
}
```

`modelStructsMap` 是定义在外部的, 用于共享缓存.

```go
var modelStructsMap sync.Map
```

如果可以从 `modelStructsMap` 找到, 就可以直接返回缓存.

```go
modelStruct.ModelType = reflectType
```

略过 `Get all fields` 部分, 直接看后面的部分.

```go
if len(modelStruct.PrimaryFields) == 0 {
  if field := getForeignField("id", modelStruct.StructFields); field != nil {
    field.IsPrimaryKey = true
    modelStruct.PrimaryFields = append(modelStruct.PrimaryFields, field)
  }
}
```

如果没有找到主键, 就会把 ID 作为主键.

```go
modelStructsMap.Store(hashKey, &modelStruct)

return &modelStruct
```

将解析好的结果保存到 `modelStructsMap`, 作为缓存, 加快后面解析的过程. 最后返回结果.

现在, 已经将整个解析流程看完了, 除了获取字段的过程不清楚, 其他都应该清楚了.

解析的过程中用到了缓存, 也是我们可以借鉴的地方, `sync.Map` 可以安全地用于 goroutine 中共享.
另一点是将结构体作为 key, 同时兼顾了单数形式的表名和复数形式的表名.

### 字段解析

前面的过程中省略了解析字段的过程, 这是非常重要的一部分. `GetModelStruct` 方法的大部分的代码都集中在这一部分中.

```go
for i := 0; i < reflectType.NumField(); i++ {
```

`reflectType.NumField()` 可以获取结构体中的字段总数.

```go
if fieldStruct := reflectType.Field(i); ast.IsExported(fieldStruct.Name) {
```

只解析可以导出的字段. 使用 `reflectType.Field(i)` 和索引 i, 可以获取到结构体中的字段.

```go
field := &StructField{
  Struct:      fieldStruct,
  Name:        fieldStruct.Name,
  Names:       []string{fieldStruct.Name},
  Tag:         fieldStruct.Tag,
  TagSettings: parseTagSetting(fieldStruct.Tag),
}
```

`StructField` 初始化, 可以看到很多信息都是从 `fieldStruct` 中获取的.

这一部分对于学习如何解析结构体中的 Tag 非常有帮助, 仔细看一下.

`fieldStruct.Tag` 可以获取字段中的 tag 部分, 比如:

```go
type Model struct {
  ID        uint `gorm:"primary_key"`
  CreatedAt time.Time
  UpdatedAt time.Time
  DeletedAt *time.Time
}
```

`ID` 字段中的 `gorm:"primary_key"` 部分.

`fieldStruct.Name` 可以获取字段的名字.

看一下具体是如何解析 Tag 字符串的.

```go
func parseTagSetting(tags reflect.StructTag) map[string]string {
	setting := map[string]string{}
	for _, str := range []string{tags.Get("sql"), tags.Get("gorm")} {
		if str == "" {
			continue
		}
		tags := strings.Split(str, ";")
		for _, value := range tags {
			v := strings.Split(value, ":")
			k := strings.TrimSpace(strings.ToUpper(v[0]))
			if len(v) >= 2 {
				setting[k] = strings.Join(v[1:], ":")
			} else {
				setting[k] = k
			}
		}
	}
	return setting
}
```

tags 的类型是 `reflect.StructTag`, 包含一些实用的方法, 比如 `Get` 方法可以获取特定的部分.
这里获取了 `sql` 和 `gorm` 部分.

每个 tag 部分中, 都是使用 `;` 分隔的选项. 每个选项又可能是 key/value 类型的, 由 `:` 分隔,
也可能是一个单独的值.

具体看一个例子:

```go
type User struct {
  gorm.Model
  Name         string
  Age          sql.NullInt64
  Birthday     *time.Time
  Email        string  `gorm:"type:varchar(100);unique_index"`
  Role         string  `gorm:"size:255"` // 设置字段大小为255
  MemberNumber *string `gorm:"unique;not null"` // 设置会员号（member number）唯一并且不为空
  Num          int     `gorm:"AUTO_INCREMENT"` // 设置 num 为自增类型
  Address      string  `gorm:"index:addr"` // 给address字段创建名为addr的索引
  IgnoreMe     int     `gorm:"-"` // 忽略本字段
}
```

比如 `Email` 字段的 tags 中 `gorm` 部分有两个选项, 一个是 `type:varchar(100)`, 另一个是 `unique_index`.

结合上面的 `parseTagSetting` 代码, 我们知道这个字段的 tags 是如何被解析的了.

对于导出的字段, 也有办法设置忽略该字段, 设置选项为 `-` 就行了.

```go
// is ignored field
if _, ok := field.TagSettingsGet("-"); ok {
  field.IsIgnored = true
}
```

然后就是解析每一个选项了. 主要的代码都在这里, 一点点看:

```go
if _, ok := field.TagSettingsGet("PRIMARY_KEY"); ok {
  field.IsPrimaryKey = true
  modelStruct.PrimaryFields = append(modelStruct.PrimaryFields, field)
}

if _, ok := field.TagSettingsGet("DEFAULT"); ok && !field.IsPrimaryKey {
  field.HasDefaultValue = true
}

if _, ok := field.TagSettingsGet("AUTO_INCREMENT"); ok && !field.IsPrimaryKey {
  field.HasDefaultValue = true
}
```

设置 `IsPrimaryKey` 和 `HasDefaultValue` 属性. 如果是主键的话, 还会添加到 `PrimaryFields` 中.

```go
indirectType := fieldStruct.Type
for indirectType.Kind() == reflect.Ptr {
  indirectType = indirectType.Elem()
}
```

获取字段的类型.

```go
fieldValue := reflect.New(indirectType).Interface()
```

获取字段对应的值.

然后是根据 fieldValue 的类型进行了一堆判断, 一个个看.

- 判断一

```go
if _, isScanner := fieldValue.(sql.Scanner); isScanner {
  // is scanner
  field.IsScanner, field.IsNormal = true, true
  if indirectType.Kind() == reflect.Struct {
    for i := 0; i < indirectType.NumField(); i++ {
      for key, value := range parseTagSetting(indirectType.Field(i).Tag) {
        if _, ok := field.TagSettingsGet(key); !ok {
          field.TagSettingsSet(key, value)
        }
      }
    }
  }
}
```

如果实现了 `sql.Scanner` 接口, 设置了两个属性为 true.

- 判断二

如果该字段是结构体, 将结构体中的每个 tag 设置都添加一遍.

```go
else if _, isTime := fieldValue.(*time.Time); isTime {
  // is time
  field.IsNormal = true
}
```

如果是 `*time.Time` 结构体, 设置 `IsNormal` 为 true.

```go
else if _, ok := field.TagSettingsGet("EMBEDDED"); ok || fieldStruct.Anonymous {
  // is embedded struct
  for _, subField := range scope.New(fieldValue).GetModelStruct().StructFields {
    subField = subField.clone()
    subField.Names = append([]string{fieldStruct.Name}, subField.Names...)
    if prefix, ok := field.TagSettingsGet("EMBEDDED_PREFIX"); ok {
      subField.DBName = prefix + subField.DBName
    }

    if subField.IsPrimaryKey {
      if _, ok := subField.TagSettingsGet("PRIMARY_KEY"); ok {
        modelStruct.PrimaryFields = append(modelStruct.PrimaryFields, subField)
      } else {
        subField.IsPrimaryKey = false
      }
    }

    if subField.Relationship != nil && subField.Relationship.JoinTableHandler != nil {
      if joinTableHandler, ok := subField.Relationship.JoinTableHandler.(*JoinTableHandler); ok {
        newJoinTableHandler := &JoinTableHandler{}
        newJoinTableHandler.Setup(subField.Relationship, joinTableHandler.TableName, reflectType, joinTableHandler.Destination.ModelType)
        subField.Relationship.JoinTableHandler = newJoinTableHandler
      }
    }

    modelStruct.StructFields = append(modelStruct.StructFields, subField)
  }
  continue
}
```

- 判断三

如果 tag 设置中有 `EMBEDDED` 字段, 表示是一个嵌入的结构体.

```go
for _, subField := range scope.New(fieldValue).GetModelStruct().StructFields {
```

遍历该字段对应的 ModelStruct 中的每个 `StructFields` 中的 `StructField`.

`subField = subField.clone()` 直接在副本上操作.

```go
subField.Names = append([]string{fieldStruct.Name}, subField.Names...)
if prefix, ok := field.TagSettingsGet("EMBEDDED_PREFIX"); ok {
  subField.DBName = prefix + subField.DBName
}
```

重新设置 `Names` 和 `DBName`.

```go
if subField.IsPrimaryKey {
  if _, ok := subField.TagSettingsGet("PRIMARY_KEY"); ok {
    modelStruct.PrimaryFields = append(modelStruct.PrimaryFields, subField)
  } else {
    subField.IsPrimaryKey = false
  }
}
```

如果 `subField` 是主键, 且有 `PRIMARY_KEY` tag 选项, 添加到 `modelStruct.PrimaryFields` 上去.
否则, 重置 `subField.IsPrimaryKey` 为 false.

```go
if subField.Relationship != nil && subField.Relationship.JoinTableHandler != nil {
  if joinTableHandler, ok := subField.Relationship.JoinTableHandler.(*JoinTableHandler); ok {
    newJoinTableHandler := &JoinTableHandler{}
    newJoinTableHandler.Setup(subField.Relationship, joinTableHandler.TableName, reflectType, joinTableHandler.Destination.ModelType)
    subField.Relationship.JoinTableHandler = newJoinTableHandler
  }
}
```

初始化了 `subField` 中的 `JoinTableHandler`.

```go
modelStruct.StructFields = append(modelStruct.StructFields, subField)
```

最后将 `subField` 添加到了 `modelStruct.StructFields` 中.

最后, 使用 `continue` 开始新的 for 循环. 因此, `field.TagSettingsGet("EMBEDDED")` 部分也结束了.

- 判断四

如果上面的三个判断都不满足, 就进入了最后的 else 判断了.

而这里面又是个 switch 判断, 真的是忧伤.

根据 `switch indirectType.Kind() {` 的类型, 主要是切片和结构体, 先看 `default` 部分:

```go
default:
  field.IsNormal = true
}
```

`case reflect.Slice:` 和 `case reflect.Struct:` 里都是一个 defer 函数.

```go
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
```

主要是处理了关系类型的 tag.

这一部分先跳过吧, 等具体研究关系的实现, 再继续深入.

等这整个 if 判断都结束后, 解析一下列名, 最后将 fields 添加到 `modelStruct.StructFields`:

```go
// Even it is ignored, also possible to decode db value into the field
if value, ok := field.TagSettingsGet("COLUMN"); ok {
  field.DBName = value
} else {
  field.DBName = ToColumnName(fieldStruct.Name)
}

modelStruct.StructFields = append(modelStruct.StructFields, field)
```

### 小结

所以, 整个模型解析的过程就是如此. 最耗时的部分在于解析每个字段, 解析 tag 以及字段间的关系.

## 总结

定义模型并解析模型的过程已经看完了, 但关于模型还有很多内容, 比如将模型转换为表插入数据库等.
