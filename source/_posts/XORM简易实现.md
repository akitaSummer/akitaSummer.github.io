---
title: XORM 简易实现
date: 2022-02-01 11:37:35
tags:
  - golang
  - orm
  - 数据库
categories: 学习笔记
---

> 优良设计创造价值的速度，快于其增加成本的速度。

# 前言

面向对象编程把所有实体看成对象（object），关系型数据库则是采用实体之间的关系（relation）连接数据。很早就有人提出，关系也可以用对象表达，这样的话，就能使用面向对象编程，来操作关系型数据库。

![orm](/image/xorm/orm.png)

简单说，ORM 就是通过实例对象的语法，完成关系型数据库的操作的技术，是"对象-关系映射"（Object/Relational Mapping） 的缩写。

举个最简单的例子：

```sql
CREATE TABLE `User` (`Name` text, `Age` integer);
INSERT INTO `User` (`Name`, `Age`) VALUES ("Tom", 18);
SELECT * FROM `User`;
```

对应的 orm 应该为：

```golang
type User struct {
    Name string
    Age  int
}

orm.CreateTable(&User{})
orm.Save(&User{"Tom", 18})
var users []User
orm.Find(&users)
```

我最近阅读了[xorm](https://github.com/xormplus/xorm)的源码，实现了一个简易的 orm 框架，其主要实现了：

- 屏蔽不同数据库之间的差异
- 完成数据库中表结构和编程语言中的对象映射
- 支持链式调用
- 支持 hooks
- 事务的支持
- 简易数据库迁移

框架的架构图如下：

![architecture](/image/xorm/architecture.jpg)

简单的流程就是 go 代码 -> 得到表信息 -> 根据表信息和代码拼接 sql -> 去 db 查询返回结果。
有点乱？看不懂？没事的，接下来我会带你一步步实现这个简易的 orm。

# Session

从架构图中可以看出，session 是整个项目的核心，我们先来实现他与数据库最简单的原生交互：

![session](/image/xorm/session.jpg)

```golang
package session

import (
        "aktorm/log"
        "database/sql"
        "strings"
)

type Session struct {
        db      *sql.DB
        sql     strings.Builder // 用于拼接SQL语句
        sqlVars []interface{} // 用于记录SQL语句中占位符的对应值
}

func New(db *sql.DB) *Session {
        return &Session{db: db}
}

func (s *Session) Clear() {
        s.sql.Reset()
        s.sqlVars = nil
}

func (s *Session) DB() *sql.DB {
        return s.db
}

func (s *Session) Raw(sql string, values ...interface{}) *Session { // 改变这sql和sqlVars的值
        s.sql.WriteString(sql)
        s.sql.WriteString(" ")
        s.sqlVars = append(s.sqlVars, values...)
        return s
}
```

接下来我们实现 Exec，QueryRow，QueryRows 原生方法

```golang
func (s *Session) Exec() (result sql.Result, err error) {
        defer s.Clear()
        log.Info(s.sql.String(), s.sqlVars)
        if result, err = s.DB().Exec(s.sql.String(), s.sqlVars...); err != nil {
                log.Error(err)
        }
        return
}

func (s *Session) QueryRow() *sql.Row {
        defer s.Clear()
        log.Info(s.sql.String(), s.sqlVars)
        return s.DB().QueryRow(s.sql.String(), s.sqlVars...)
}

func (s *Session) QueryRows() (rows *sql.Rows, err error) {
        defer s.Clear()
        log.Info(s.sql.String(), s.sqlVars)
        if rows, err = s.DB().Query(s.sql.String(), s.sqlVars...); err != nil {
                log.Error(err)
        }
        return
}
```

Session 主要负责与数据库交互及交互前生成 sql 语句，而整个项目我们需要使用一个 Engine 封装来维护，作为用户的入口：

```golang
package aktorm

import (
        "database/sql"

        "aktorm/log"
        "aktorm/session"
)

type Engine struct {
        db *sql.DB
}

func NewEngine(driver, source string) (e *Engine, err error) {
        db, err := sql.Open(driver, source)
        if err != nil {
                log.Error(err)
                return
        }
        if err = db.Ping(); err != nil {
                log.Error(err)
                return
        }
        e = &Engine{db: db}
        log.Info("Connect database success")
        return
}

func (engine *Engine) Close() {
        if err := engine.db.Close(); err != nil {
                log.Error("Failed to close database")
        }
        log.Info("Close database success")
}

func (engine *Engine) NewSession() *session.Session {
        return session.New(engine.db)
}
```

这样，一个最基本的结构就形成了，我们写两个测试来测试一下

```golang
package session

import (
        "database/sql"
        "os"
        "testing"

        _ "github.com/mattn/go-sqlite3"
)

var TestDB *sql.DB

func TestMain(m *testing.M) {
        TestDB, _ = sql.Open("sqlite3", "../akt.db")
        code := m.Run()
        _ = TestDB.Close()
        os.Exit(code)
}

func NewSession() *Session {
        return New(TestDB)
}

func TestSession_Exec(t *testing.T) {
        s := NewSession()
        _, _ = s.Raw("DROP TABLE IF EXISTS User;").Exec()
        _, _ = s.Raw("CREATE TABLE User(Name text);").Exec()
        result, _ := s.Raw("INSERT INTO User(`Name`) values (?), (?)", "Tom", "Sam").Exec()
        if count, err := result.RowsAffected(); err != nil || count != 2 {
                t.Fatal("expect 2, but got", count)
        }
}

func TestSession_QueryRows(t *testing.T) {
        s := NewSession()
        _, _ = s.Raw("DROP TABLE IF EXISTS User;").Exec()
        _, _ = s.Raw("CREATE TABLE User(Name text);").Exec()
        row := s.Raw("SELECT count(*) FROM User").QueryRow()
        var count int
        if err := row.Scan(&count); err != nil || count != 0 {
                t.Fatal("failed to query db", err)
        }
}

package aktorm

import (
        _ "github.com/mattn/go-sqlite3"
        "testing"
)

func OpenDB(t *testing.T) *Engine {
        t.Helper()
        engine, err := NewEngine("sqlite3", "akt.db")
        if err != nil {
                t.Fatal("failed to connect", err)
        }
        return engine
}

func TestNewEngine(t *testing.T) {
        engine := OpenDB(t)
        defer engine.Close()
}
```

# Dialect

dialect  主要作用是根据要储存的数据库，将 go 语言的类型映射为数据库中的类型。

```golang
package dialect

import "reflect"

var dialectsMap = map[string]Dialect{}

type Dialect interface {
        DataTypeOf(reflect.Value) string                        // 用于将 Go 语言的类型转换为该数据库的数据类型
        TableExistSQL(tableName string) (string, []interface{}) // 返回某个表是否存在的 SQL 语句
}

func RegisterDialect(name string, dialect Dialect) {
        dialectsMap[name] = dialect
}

func GetDialect(name string) (Dialect, bool) {
        dialect, ok := dialectsMap[name]
        return dialect, ok
}
```

```golang
package dialect

import (
        "fmt"
        "reflect"
        "time"
)

type sqlite3 struct{}

// 确保sqlite3实现了Dialect接口的所有方法
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

在引擎创建时，应当指明 dialect 类型

```golang
import (
        "database/sql"

        "aktorm/dialect"
        "aktorm/log"
        "aktorm/session"
)

type Engine struct {
        db      *sql.DB
        // new
        dialect dialect.Dialect
}

func NewEngine(driver, source string) (e *Engine, err error) {
        db, err := sql.Open(driver, source)
        if err != nil {
                log.Error(err)
                return
        }
        if err = db.Ping(); err != nil {
                log.Error(err)
                return
        }
        // new
        dial, ok := dialect.GetDialect(driver)
        if !ok {
                log.Errorf("dialect %s Not Found", driver)
                return
        }
        e = &Engine{db: db, dialect: dial}
        log.Info("Connect database success")
        return
}
func (engine *Engine) Close() {
        if err := engine.db.Close(); err != nil {
                log.Error("Failed to close database")
        }
        log.Info("Close database success")
}

// new
func (engine *Engine) NewSession() *session.Session {
        return session.New(engine.db, engine.dialect)
}
```

Schema
有了 dialect 后，我们可以创建一个 schema 用于桥接 session 和 dialect，负责解析 go 代码获取相应的 table 信息：

```golang
package schema

import (
        "aktorm/dialect"
        "go/ast"
        "reflect"
)

type Field struct {
        Name string // 字段名
        Type string // 类型
        Tag  string // 约束条件
}

type Schema struct {
        Model      interface{} // 被映射的对象
        Name       string      // 表名
        Fields     []*Field    // 字段
        FieldNames []string    //列名
        fieldMap   map[string]*Field
}

func (schema *Schema) GetField(name string) *Field {
        return schema.fieldMap[name]
}

// 任意的对象解析为Schema实例

func Parse(dest interface{}, d dialect.Dialect) *Schema {
        // reflect.Indirect()获取指针指向的实例 reflect.ValueOf()获取值 reflect.Type()获取类型
        modelType := reflect.Indirect(reflect.ValueOf(dest)).Type()
        schema := &Schema{
                Model:    dest,
                Name:     modelType.Name(), // 获取到结构体的名称
                fieldMap: make(map[string]*Field),
        }
        // modelType.NumField()获取实例的字段的个数
        for i := 0; i < modelType.NumField(); i++ {
                p := modelType.Field(i) // 通过下标获取到特定字段
                if !p.Anonymous && ast.IsExported(p.Name) {
                        field := &Field{
                                Name: p.Name,
                                Type: d.DataTypeOf(reflect.Indirect(reflect.New(p.Type))),
                        }
                        if v, ok := p.Tag.Lookup("aktorm"); ok {
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

然后我们更新下 session：

```golang
package session

import (
        "aktorm/dialect"
        "aktorm/log"
        "aktorm/schema"
        "database/sql"
        "strings"
)

type Session struct {
        db       *sql.DB
        dialect  dialect.Dialect
        refTable *schema.Schema
        sql      strings.Builder
        sqlVars  []interface{}
}

func New(db *sql.DB, dialect dialect.Dialect) *Session {
        return &Session{
                db:      db,
                dialect: dialect,
        }
}
```

最后我们来实现一下建表语句：

```golang
package session

import (
        "aktorm/log"
        "aktorm/schema"
        "fmt"
        "reflect"
        "strings"
)

// 给 refTable 赋值
func (s *Session) Model(value interface{}) *Session {
        if s.refTable == nil || reflect.TypeOf(value) != reflect.TypeOf(s.refTable.Model) {
                s.refTable = schema.Parse(value, s.dialect)
        }
        return s
}

func (s *Session) RefTable() *schema.Schema {
        if s.refTable == nil {
                log.Error("Model is not set")
        }
        return s.refTable
}

func (s *Session) CreateTable() error {
        table := s.RefTable()
        columns := []string{}
        for _, field := range table.Fields {
                columns = append(columns, fmt.Sprintf("%s %s %s", field.Name, field.Type, field.Tag))
        }
        desc := strings.Join(columns, ",")
        _, err := s.Raw(fmt.Sprintf("CREATE TABLE %s (%s);", table.Name, desc)).Exec()
        return err
}

func (s *Session) DropTable() error {
        _, err := s.Raw(fmt.Sprintf("DROP TABLE IF EXISTS %s", s.RefTable().Name)).Exec()
        return err
}

func (s *Session) HasTable() bool {
        sql, values := s.dialect.TableExistSQL(s.RefTable().Name)
        row := s.Raw(sql, values...).QueryRow()
        var tmp string
        _ = row.Scan(&tmp)
        return tmp == s.RefTable().Name
}
```

我们来写下测试测试一下代码：

```golang
package schema

import (
        "aktorm/dialect"
        "testing"
)

type User struct {
        Name string `aktorm:"PRIMARY KEY"`
        Age  int
}

var TestDial, _ = dialect.GetDialect("sqlite3")

func TestParse(t *testing.T) {
        schema := Parse(&User{}, TestDial)
        if schema.Name != "User" || len(schema.Fields) != 2 {
                t.Fatal("failed to parse User struct")
        }
        if schema.GetField("Name").Tag != "PRIMARY KEY" {
                t.Fatal("failed to parse primary key")
        }
}

package session

import "testing"

type User struct {
        Name string `aktorm:"PRIMARY KEY"`
        Age  int
}

func TestSession_CreateTable(t *testing.T) {
        s := NewSession().Model(&User{})
        _ = s.DropTable()
        _ = s.CreateTable()
        if !s.HasTable() {
                t.Fatal("Failed to create table User")
        }
}
```

# Clause

现在我们来实现最后的 clause 作用主要是实现并拼接各个子句的生成规则。
generator 主要实现生成规则：

```golang
package clause

import (
        "fmt"
        "strings"
)

type generator func(values ...interface{}) (string, []interface{})

var generators map[Type]generator

func init() {
        generators = make(map[Type]generator)
        generators[INSERT] = _insert
        generators[VALUES] = _values
        generators[SELECT] = _select
        generators[LIMIT] = _limit
        generators[WHERE] = _where
        generators[ORDERBY] = _orderBy
}

func genBindVars(num int) string {
        var vars []string
        for i := 0; i < num; i++ {
                vars = append(vars, "?")
        }
        return strings.Join(vars, ", ")
}

func _insert(values ...interface{}) (string, []interface{}) { // INSERT INTO $tableName ($fields)
        tableName := values[0]
        fields := strings.Join(values[1].([]string), ",")
        return fmt.Sprintf("INSERT INTO %s (%v)", tableName, fields), []interface{}{}
}

func _values(values ...interface{}) (string, []interface{}) { // VALUES ($v1), ($v2), ...
        var bindStr string
        var sql strings.Builder
        var vars []interface{}
        sql.WriteString("VALUES ")
        for i, value := range values {
                v := value.([]interface{})
                if bindStr == "" {
                        bindStr = genBindVars(len(v))
                }
                sql.WriteString(fmt.Sprintf("(%v)", bindStr))
                if i+1 != len(values) {
                        sql.WriteString(", ")
                }
                vars = append(vars, v...)
        }
        return sql.String(), vars

}

func _select(values ...interface{}) (string, []interface{}) { // SELECT $fields FROM $tableName
        tableName := values[0]
        fields := strings.Join(values[1].([]string), ",")
        return fmt.Sprintf("SELECT %v FROM %s", fields, tableName), []interface{}{}
}

func _limit(values ...interface{}) (string, []interface{}) { // LIMIT $num
        return "LIMIT ?", values
}

func _where(values ...interface{}) (string, []interface{}) { // WHERE $desc
        desc, vars := values[0], values[1:]
        return fmt.Sprintf("WHERE %s", desc), vars
}

func _orderBy(values ...interface{}) (string, []interface{}) {
        return fmt.Sprintf("ORDER BY %s", values[0]), []interface{}{}
}
```

clause 主要用于拼接

```golang
package clause

import "strings"

type Clause struct {
        sql     map[Type]string
        sqlVars map[Type][]interface{}
}

type Type int

const (
        INSERT Type = iota
        VALUES
        SELECT
        LIMIT
        WHERE
        ORDERBY
)

func (c *Clause) Set(name Type, vars ...interface{}) {
        if c.sql == nil {
                c.sql = make(map[Type]string)
                c.sqlVars = make(map[Type][]interface{})
        }
        sql, vars := generators[name](vars...)
        c.sql[name] = sql
        c.sqlVars[name] = vars
}

func (c *Clause) Build(orders ...Type) (string, []interface{}) {
        var sqls []string
        var vars []interface{}
        for _, order := range orders {
                if sql, ok := c.sql[order]; ok {
                        sqls = append(sqls, sql)
                        vars = append(vars, c.sqlVars[order]...)
                }
        }
        return strings.Join(sqls, " "), vars
}
```

我们将 clause 添加到 session 上：

```golang
package session

import (
    // new
        "aktorm/clause"
        "aktorm/dialect"
        "aktorm/log"
        "aktorm/schema"
        )
type Session struct {
        db       *sql.DB
        dialect  dialect.Dialect
        refTable *schema.Schema
        // new
        clause   clause.Clause
        sql      strings.Builder
        sqlVars  []interface{}
}

func (s *Session) Clear() {
        s.sql.Reset()
        s.sqlVars = nil
        s.clause = clause.Clause{}
}
```

然后我们编写插入/查询方法：

```golang
package session

import (
        "aktorm/clause"
        "reflect"
)

// INSERT INTO table_name(col1, col2, col3, ...) VALUES
//     (A1, A2, A3, ...),
//     (B1, B2, B3, ...)

// s := aktorm.NewEngine("sqlite3", "akt.db").NewSession()
// u1 := &User{Name: "Tom", Age: 18}
// u2 := &User{Name: "Sam", Age: 25}
// s.Insert(u1, u2)
func (s *Session) Insert(values ...interface{}) (int64, error) {
        recordValues := make([]interface{}, 0)
        for _, value := range values {
                table := s.Model(value).RefTable()
                s.clause.Set(clause.INSERT, table.Name, table.FieldNames)
                recordValues = append(recordValues, table.RecordValues(value))
        }

        s.clause.Set(clause.VALUES, recordValues...)
        sql, vars := s.clause.Build(clause.INSERT, clause.VALUES)
        result, err := s.Raw(sql, vars...).Exec()
        if err != nil {
                return 0, err
        }

        return result.RowsAffected()
}

// s := aktorm.NewEngine("sqlite3", "akt.db").NewSession()
// var users []User
// s.Find(&users);
func (s *Session) Find(values interface{}) error {
        destSlice := reflect.Indirect(reflect.ValueOf(values))
        destType := destSlice.Type().Elem() // 获取切片的单个元素的类型
        table := s.Model(reflect.New(destType).Elem().Interface()).RefTable()

        s.clause.Set(clause.SELECT, table.Name, table.FieldNames)
        sql, vars := s.clause.Build(clause.SELECT, clause.WHERE, clause.ORDERBY, clause.LIMIT)
        rows, err := s.Raw(sql, vars...).QueryRows()
        if err != nil {
                return err
        }

        for rows.Next() {
                dest := reflect.New(destType).Elem()
                var values []interface{}
                for _, name := range table.FieldNames {
                        values = append(values, dest.FieldByName(name).Addr().Interface())
                }
                if err := rows.Scan(values...); err != nil {
                        return err
                }
                destSlice.Set(reflect.Append(destSlice, dest))
        }
        return rows.Close()
}
```

测试：

```golang
package clause

import (
        "reflect"
        "testing"
)

func testSelect(t *testing.T) {
        var clause Clause
        clause.Set(LIMIT, 3)
        clause.Set(SELECT, "User", []string{"*"})
        clause.Set(WHERE, "Name = ?", "Tom")
        clause.Set(ORDERBY, "Age ASC")
        sql, vars := clause.Build(SELECT, WHERE, ORDERBY, LIMIT)
        t.Log(sql, vars)
        if sql != "SELECT * FROM User WHERE Name = ? ORDER BY Age ASC LIMIT ?" {
                t.Fatal("failed to build SQL")
        }
        if !reflect.DeepEqual(vars, []interface{}{"Tom", 3}) {
                t.Fatal("failed to build SQLVars")
        }
}

func TestClause_Build(t *testing.T) {
        t.Run("select", func(t *testing.T) {
                testSelect(t)
        })
}

package session

import "testing"

var (
        user1 = &User{"Tom", 18}
        user2 = &User{"Sam", 25}
        user3 = &User{"Jack", 25}
)

func testRecordInit(t *testing.T) *Session {
        t.Helper()
        s := NewSession().Model(&User{})
        err1 := s.DropTable()
        err2 := s.CreateTable()
        _, err3 := s.Insert(user1, user2)
        if err1 != nil || err2 != nil || err3 != nil {
                t.Fatal("failed init test records")
        }
        return s
}

func TestSession_Insert(t *testing.T) {
        s := testRecordInit(t)
        affected, err := s.Insert(user3)
        if err != nil || affected != 1 {
                t.Fatal("failed to create record")
        }
}

func TestSession_Find(t *testing.T) {
        s := testRecordInit(t)
        var users []User
        if err := s.Find(&users); err != nil || len(users) != 2 {
                t.Fatal("failed to query all")
        }
}
```

# 链式调用

通过前 4 章的努力，我们已经将 orm 框架基本搭建了起来，接下来我们为其扩充更多的功能，首先是链式调用，update， delete 和 count 比较常用到，我们以这三个为例来编写：

```golang
package clause
import "strings"
type Clause struct {
        sql     map[Type]string
        sqlVars map[Type][]interface{}
}
type Type int
const (
        INSERT Type = iota
        VALUES
        SELECT
        LIMIT
        WHERE
        ORDERBY
        // new
        UPDATE
        DELETE
        COUNT
)
```

在 record 中注册方法，方法通过返回 session 从而实现链式调用：

```golang
func (s *Session) Update(kv ...interface{}) (int64, error) {
        m, ok := kv[0].(map[string]interface{})
        if !ok {
                m = make(map[string]interface{})
                for i := 0; i < len(kv); i += 2 {
                        m[kv[i].(string)] = kv[i+1]
                }
        }
        s.clause.Set(clause.UPDATE, s.RefTable().Name, m)
        sql, vars := s.clause.Build(clause.UPDATE, clause.WHERE)
        result, err := s.Raw(sql, vars...).Exec()
        if err != nil {
                return 0, err
        }
        return result.RowsAffected()
}

func (s *Session) Delete() (int64, error) {
        s.clause.Set(clause.DELETE, s.RefTable().Name)
        sql, vars := s.clause.Build(clause.DELETE, clause.WHERE)
        result, err := s.Raw(sql, vars...).Exec()
        if err != nil {
                return 0, err
        }
        return result.RowsAffected()
}

func (s *Session) Count() (int64, error) {
        s.clause.Set(clause.COUNT, s.RefTable().Name)
        sql, vars := s.clause.Build(clause.COUNT, clause.WHERE)
        row := s.Raw(sql, vars...).QueryRow()
        var tmp int64
        if err := row.Scan(&tmp); err != nil {
                return 0, err
        }
        return tmp, nil
}

func (s *Session) Limit(num int) *Session {
        s.clause.Set(clause.LIMIT, num)
        return s
}

func (s *Session) Where(desc string, args ...interface{}) *Session {
        var vars []interface{}
        s.clause.Set(clause.WHERE, append(append(vars, desc), args...)...)
        return s
}

func (s *Session) OrderBy(desc string) *Session {
        s.clause.Set(clause.ORDERBY, desc)
        return s
}

func (s *Session) First(value interface{}) error {
        dest := reflect.Indirect(reflect.ValueOf(value))
        destSlice := reflect.New(reflect.SliceOf(dest.Type())).Elem()
        if err := s.Limit(1).Find(destSlice.Addr().Interface()); err != nil {
                return err
        }
        if destSlice.Len() == 0 {
                return errors.New("NOT FOUND")
        }
        dest.Set(destSlice.Index(0))
        return nil
}
```

测试：
record_test.go

```golang
func TestSession_Limit(t *testing.T) {
        s := testRecordInit(t)
        var users []User
        err := s.Limit(1).Find(&users)
        if err != nil || len(users) != 1 {
                t.Fatal("failed to query with limit condition")
        }
}

func TestSession_Update(t *testing.T) {
        s := testRecordInit(t)
        affected, _ := s.Where("Name = ?", "Tom").Update("Age", 30)
        u := &User{}
        _ = s.OrderBy("Age DESC").First(u)

        if affected != 1 || u.Age != 30 {
                t.Fatal("failed to update")
        }
}

func TestSession_DeleteAndCount(t *testing.T) {
        s := testRecordInit(t)
        affected, _ := s.Where("Name = ?", "Tom").Delete()
        count, _ := s.Count()

        if affected != 1 || count != 1 {
                t.Fatal("failed to delete or count")
        }
}
```

# hooks

当我们在进行 crud 的时候，可能会进行一些例如脱敏相关的操作，这个时候，hooks 的存在会大大简化我们的代码，这一章我们就来实现它：

```golang
package session

import (
        "aktorm/log"
        "reflect"
)

// Hooks constants
const (
        BeforeQuery  = "BeforeQuery"
        AfterQuery   = "AfterQuery"
        BeforeUpdate = "BeforeUpdate"
        AfterUpdate  = "AfterUpdate"
        BeforeDelete = "BeforeDelete"
        AfterDelete  = "AfterDelete"
        BeforeInsert = "BeforeInsert"
        AfterInsert  = "AfterInsert"
)

func (s *Session) CallMethod(method string, value interface{}) {
        fm := reflect.ValueOf(s.RefTable().Model).MethodByName(method)
        if value != nil {
                fm = reflect.ValueOf(value).MethodByName(method)
        }
        param := []reflect.Value{reflect.ValueOf(s)}
        if fm.IsValid() {
                if v := fm.Call(param); len(v) > 0 {
                        if err, ok := v[0].Interface().(error); ok {
                                log.Error(err)
                        }
                }
        }
        return
}
```

然后我们只需要在对应的事件上添加上即可：

```golang
func (s *Session) Insert(values ...interface{}) (int64, error) {
        recordValues := make([]interface{}, 0)
        for _, value := range values {
                s.CallMethod(BeforeInsert, value)
                table := s.Model(value).RefTable()
                s.clause.Set(clause.INSERT, table.Name, table.FieldNames)
                recordValues = append(recordValues, table.RecordValues(value))

        if err != nil {
                return 0, err
        }
        s.CallMethod(AfterInsert, nil)
        return result.RowsAffected()
}
```

测试：

```golang
package session

import (
        "aktorm/log"
        "testing"
)

type Account struct {
        ID       int `aktorm:"PRIMARY KEY"`
        Password string
}

func (account *Account) BeforeInsert(s *Session) error {
        log.Info("before inert", account)
        account.ID += 1000
        return nil
}

func (account *Account) AfterQuery(s *Session) error {
        log.Info("after query", account)
        account.Password = "******"
        return nil
}

func TestSession_CallMethod(t *testing.T) {
        s := NewSession().Model(&Account{})
        _ = s.DropTable()
        _ = s.CreateTable()
        _, _ = s.Insert(&Account{1, "123456"}, &Account{2, "qwerty"})

        u := &Account{}

        err := s.First(u)
        if err != nil || u.ID != 1001 || u.Password != "******" {
                t.Fatal("Failed to call hooks after query, got", u)
        }
}
```

# Transaction

> 数据库事务( transaction)是访问并可能操作各种数据项的一个数据库操作序列，这些操作要么全部执行,要么全部不执行，是一个不可分割的工作单位。  事务由事务开始与事务结束之间执行的全部数据库操作组成。

事务举个例子简单来说，就是 A 转账给 B 1000 元，操作就是 a -1000, b +1000，如果有任何一个操作出错，则视为不成功，回滚至操作前。

事务主要有四个特性：

1. 原子性（Atomicity）
   　　原子性是指事务包含的所有操作要么全部成功，要么全部失败回滚，这和前面两篇博客介绍事务的功能是一样的概念，因此事务的操作如果成功就必须要完全应用到数据库，如果操作失败则不能对数据库有任何影响。
2. 一致性（Consistency）
   　　一致性是指事务必须使数据库从一个一致性状态变换到另一个一致性状态，也就是说一个事务执行之前和执行之后都必须处于一致性状态。
   　　拿转账来说，假设用户 A 和用户 B 两者的钱加起来一共是 5000，那么不管 A 和 B 之间如何转账，转几次账，事务结束后两个用户的钱相加起来应该还得是 5000，这就是事务的一致性。
3. 隔离性（Isolation）
   　　隔离性是当多个用户并发访问数据库时，比如操作同一张表时，数据库为每一个用户开启的事务，不能被其他事务的操作所干扰，多个并发事务之间要相互隔离。
   　　即要达到这么一种效果：对于任意两个并发的事务 T1 和 T2，在事务 T1 看来，T2 要么在 T1 开始之前就已经结束，要么在 T1 结束之后才开始，这样每个事务都感觉不到有其他事务在并发地执行。
   　　关于事务的隔离性数据库提供了多种隔离级别，稍后会介绍到。
4. 持久性（Durability）
   　　持久性是指一个事务一旦被提交了，那么对数据库中的数据的改变就是永久性的，即便是在数据库系统遇到故障的情况下也不会丢失提交事务的操作。

例如我们在使用 JDBC 操作数据库时，在提交事务方法后，提示用户事务操作完成，当我们程序执行完成直到看到提示后，就可以认定事务以及正确提交，即使这时候数据库出现了问题，也必须要将我们的事务完全执行完成，否则就会造成我们看到提示事务处理完毕，但是数据库因为故障而没有执行事务的重大错误。

```sql
BEGIN;
DELETE FROM User WHERE Age > 25;
INSERT INTO User VALUES ("Tom", 25), ("Jack", 18);
COMMIT;
```

我们先在 raw.go 中添加上

```golang
type Session struct {
        db       *sql.DB
        dialect  dialect.Dialect
        // new
        tx       *sql.Tx
        refTable *schema.Schema
        clause   clause.Clause
        sql      strings.Builder
        sqlVars  []interface{}
}

type CommonDB interface {
        Query(query string, args ...interface{}) (*sql.Rows, error)
        QueryRow(query string, args ...interface{}) *sql.Row
        Exec(query string, args ...interface{}) (sql.Result, error)
}

var _ CommonDB = (*sql.DB)(nil)
var _ CommonDB = (*sql.Tx)(nil)

func (s *Session) DB() CommonDB {
        if s.tx != nil {
                return s.tx
        }
        return s.db
}
```

然后创建 transaction：

```golang
package session

import "aktorm/log"

func (s *Session) Begin() error {
        log.Info("transaction begin")
        var err error
        if s.tx, err = s.db.Begin(); err != nil {
                log.Error(err)
                return err
        }
        return nil
}

func (s *Session) Commit() error {
        log.Info("transaction commit")
        var err error
        if err = s.tx.Commit(); err != nil {
                log.Error(err)
        }
        return err
}

func (s *Session) Rollback() error {
        log.Info("transaction rollback")
        var err error
        if err = s.tx.Rollback(); err != nil {
                log.Error(err)
        }
        return err
}
```

最后在 engine 中加上：

```golang
type TxFunc func(*session.Session) (interface{}, error)

func (engine *Engine) Transaction(f TxFunc) (result interface{}, err error) {
        s := engine.NewSession()
        if err := s.Begin(); err != nil {
                return nil, err
        }
        defer func() {
                if p := recover(); p != nil {
                        _ = s.Rollback()
                        panic(p)
                } else if err != nil {
                        _ = s.Rollback()
                } else {
                        err = s.Commit()
                }
        }()

        return f(s)
}
```

测一下：

```golang
// aktorm_test.go
type User struct {
        Name string `aktorm:"PRIMARY KEY"`
        Age  int
}

func transactionRollback(t *testing.T) {
        engine := OpenDB(t)
        defer engine.Close()
        s := engine.NewSession()
        _ = s.Model(&User{}).DropTable()
        _, err := engine.Transaction(func(s *session.Session) (result interface{}, err error) {
                _ = s.Model(&User{}).CreateTable()
                _, err = s.Insert(&User{"Tom", 18})
                return nil, errors.New("Error")
        })
        if err == nil || s.HasTable() {
                t.Fatal("failed to rollback")
        }
}

func transactionCommit(t *testing.T) {
        engine := OpenDB(t)
        defer engine.Close()
        s := engine.NewSession()
        _ = s.Model(&User{}).DropTable()
        _, err := engine.Transaction(func(s *session.Session) (result interface{}, err error) {
                _ = s.Model(&User{}).CreateTable()
                _, err = s.Insert(&User{"Tom", 18})
                return
        })
        u := &User{}
        _ = s.First(u)
        if err != nil || u.Name != "Tom" {
                t.Fatal("failed to commit")
        }
}

func TestEngine_Transaction(t *testing.T) {
        t.Run("rollback", func(t *testing.T) {
                transactionRollback(t)
        })
        t.Run("commit", func(t *testing.T) {
                transactionCommit(t)
        })
}
```

# Migrate

当我们的 struct 发生改变时，我们应当能自动更新表结构。
增加字段的语法比较简单：

```sql
ALTER TABLE table_name ADD COLUMN col_name, col_type;
```

而删除稍微麻烦一点：

```sql
CREATE TABLE new_table AS SELECT col1, col2, ... from old_table
DROP TABLE old_table
ALTER TABLE new_table RENAME TO old_table;
```

我们来实现一下它：

```golang
func difference(a []string, b []string) []string {
        diff := []string{}
        diffMap := map[string]bool{}
        for _, v := range b {
                diffMap[v] = true
        }
        for _, v := range a {
                if _, ok := diffMap[v]; !ok {
                        diff = append(diff, v)
                }
        }
        return diff
}

// ALTER TABLE table_name ADD COLUMN col_name, col_type;

// CREATE TABLE new_table AS SELECT col1, col2, ... from old_table
// DROP TABLE old_table
// ALTER TABLE new_table RENAME TO old_table;

func (engine *Engine) Migrate(value interface{}) error {
        _, err := engine.Transaction(func(s *session.Session) (result interface{}, err error) {
                if !s.Model(value).HasTable() {
                        log.Infof("table %s doesn't exist", s.RefTable().Name)
                        return nil, s.CreateTable()
                }
                table := s.RefTable()
                rows, _ := s.Raw(fmt.Sprintf("SELECT * FROM %s LIMIT 1", table.Name)).QueryRows()
                columns, _ := rows.Columns()
                addCols := difference(table.FieldNames, columns)
                delCols := difference(columns, table.FieldNames)
                log.Infof("added cols %v, deleted cols %v", addCols, delCols)

                for _, col := range addCols {
                        f := table.GetField(col)
                        sqlStr := fmt.Sprintf("ALTER TABLE %s ADD COLUMN %s %s;", table.Name, f.Name, f.Type)
                        if _, err = s.Raw(sqlStr).Exec(); err != nil {
                                return
                        }
                }

                if len(delCols) == 0 {
                        return
                }
                tmp := "tmp_" + table.Name
                fieldStr := strings.Join(table.FieldNames, ", ")
                s.Raw(fmt.Sprintf("CREATE TABLE %s AS SELECT %s from %s;", tmp, fieldStr, table.Name))
                s.Raw(fmt.Sprintf("DROP TABLE %s;", table.Name))
                s.Raw(fmt.Sprintf("ALTER TABLE %s RENAME TO %s;", tmp, table.Name))
                _, err = s.Exec()
                return
        })
        return err
}
```

测试：

```golang
func TestEngine_Migrate(t *testing.T) {
        engine := OpenDB(t)
        defer engine.Close()
        s := engine.NewSession()
        _, _ = s.Raw("DROP TABLE IF EXISTS User;").Exec()
        _, _ = s.Raw("CREATE TABLE User(Name text PRIMARY KEY, XXX integer);").Exec()
        _, _ = s.Raw("INSERT INTO User(`Name`) values (?), (?)", "Tom", "Sam").Exec()
        engine.Migrate(&User{})

        rows, _ := s.Raw("SELECT * FROM User").QueryRows()
        columns, _ := rows.Columns()
        if !reflect.DeepEqual(columns, []string{"Name", "Age"}) {
                t.Fatal("Failed to migrate table User, got columns", columns)
        }
}
```

# 结尾

这个 orm 的框架较为简单，比如 struct 嵌套，外键，复合主键等等都没有考虑，一个好的 orm 框架，会有大量的代码来实现特性及相关功能。这里我就不写了，相信大家也能知道。
这个框架的意义是在于为了能了解到 orm 框架的设计及编写的思路，如果能从中学习到一些知识，那目的就达到了。
