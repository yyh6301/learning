
orm框架，通常用来描述数据库和对象当中的映射，可以将程序中的对象持久化到数据库当中，利于用户的操作


## 映射关系

orm的核心是把面向对象语言当中的*对象*，翻译成数据库中的*表*，
对象的*字段*对应为表当中的*列*，
对象的*实例*，则对应表当中的*一行数据*


在golang当中，我们可以使用reflect，来获取一个结构体的所有属性，以及其标签等信息，随后再转换成对应的SQL语句，与数据库进行交互。


通过`Schema`来映射数据库当中的表，`Field`来映射数据库当中的字段。`Dialet`来映射数据库当中的数据结构
```go
// Field  对应数据库的字段
type Field struct {  
	Name string  
	Type string  
	Tag  string  
}  
  
// Schema 对应数据库当中的一张表 
type Schema struct {  
	Model      interface{}  
	Name       string  
	Fields     []*Field  
	FieldNames []string  
	fieldMap   map[string]*Field  
}

// Schema相关的Parse函数，将一个结构体，转换成Schema对象
func Parse(dest interface{}, d dialect.Dialect) *Schema {  
	modelType := reflect.Indirect(reflect.ValueOf(dest)).Type()  
	schema := &Schema{  
		Model:    dest,  
		Name:     modelType.Name(),  
		fieldMap: make(map[string]*Field),  
	}  
  
	for i := 0; i < modelType.NumField(); i++ {  
		p := modelType.Field(i)  
		if !p.Anonymous && ast.IsExported(p.Name) {  
			field := &Field{  
				Name: p.Name,  
				Type: d.DataTypeOf(reflect.Indirect(reflect.New(p.Type))),  
			}  
			if v, ok := p.Tag.Lookup("geeorm"); ok {  
				field.Tag = v  
			}  
			schema.Fields = append(schema.Fields, field)  
			schema.FieldNames = append(schema.FieldNames, p.Name)  
			schema.fieldMap[p.Name] = field  
		}  
	}  
	return schema  
}
```

dialet的实现，DataTypeOf来进行数据类型的转换，TableExistSQL用来判断表是否存在
```go
package dialect  
  
import "reflect"  
  
var dialectsMap = map[string]Dialect{}  
  
type Dialect interface {  
	DataTypeOf(typ reflect.Value) string  
	TableExistSQL(tableName string) (string, []interface{})  
}  
  
func RegisterDialect(name string, dialect Dialect) {  
	dialectsMap[name] = dialect  
}  
  
func GetDialect(name string) (dialect Dialect, ok bool) {  
	dialect, ok = dialectsMap[name]  
	return  
}


type sqlite3 struct{}  
  
var _ Dialect = (*sqlite3)(nil)  
  
func init() {  
	RegisterDialect("sqlite3", &sqlite3{})  
}  
  
func (s *sqlite3) DataTypeOf(typ reflect.Value) string {  
	switch typ.Kind() {  
	case reflect.Bool:  
		return "bool"  
	case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32,  
		reflect.Uint, reflect.Uint8, reflect.Uint16, reflect.Uint32, reflect.Uintptr:  
		return "integer"  
	case reflect.Int64, reflect.Uint64:  
		return "bigint"  
	case reflect.Float32, reflect.Float64:  
		return "real"  
	case reflect.String:  
		return "text"  
	case reflect.Array, reflect.Slice:  
		return "blob"  
	case reflect.Struct:  
		if _, ok := typ.Interface().(time.Time); ok {  
			return "datetime"  
		}  
	}  
	panic(fmt.Sprintf("invalid sql type %s (%s)", typ.Type().Name(), typ.Kind()))  
}  
  
func (s *sqlite3) TableExistSQL(tableName string) (string, []interface{}) {  
	args := []interface{}{tableName}  
	return "SELECT name FROM sqlite_master WHERE type='table' and name = ?", args  
}
```