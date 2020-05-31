# Go Relational Persistence

[![build status](https://img.shields.io/travis/mmrath/gorp/master.svg)](http://travis-ci.org/mmrath/gorp)
[![code coverage](https://img.shields.io/coveralls/mmrath/gorp.svg)](https://coveralls.io/r/mmrath/gorp)
[![issues](https://img.shields.io/github/issues/mmrath/gorp.svg)](https://github.com/mmrath/gorp/issues)
[![godoc](https://img.shields.io/badge/godoc-master-green)](https://pkg.go.dev/github.com/mmrath/gorp?tab=doc)

### Fork

This is a fork of https://github.com/go-gorp/gorp

I have done the following changes after fork
1. Changed `Get` to throw error(`sql.ErrNoRows`) instead of null in case record is not found
2. Changed `Select` and `SelectOne` to return rows in the provided input parameter instead of `[]interface{}`. I found that returning `[]interface{}` means that I need to do type cast later.
3. Removed dynamic table functionality. I thought this would be much cleaner by mapping a new struct for table. I have not really seen the much usage of this functionality.
4. Run tests with pgx driver instead of pq, because I am using pgx

I plan to add some more functionality without breaking the current code. 
I would be happy to accept any PR as long as 
1. It does not break current code
2. It does not introduce full ORM functionality. Gorp will only ever work on one struct(or slice), i.e. relationships will not be supported.

## Introduction

I hesitate to call gorp an ORM.  Go doesn't really have objects, at
least not in the classic Smalltalk/Java sense.  There goes the "O".
gorp doesn't know anything about the relationships between your
structs (at least not yet).  So the "R" is questionable too (but I use
it in the name because, well, it seemed more clever).

The "M" is alive and well.  Given some Go structs and a database, gorp
should remove a fair amount of boilerplate busy-work from your code.

I hope that gorp saves you time, minimizes the drudgery of getting
data in and out of your database, and helps your code focus on
algorithms, not infrastructure.

* Bind struct fields to table columns via API or tag
* Support for embedded structs
* Support for transactions
* Forward engineer db schema from structs (great for unit tests)
* Pre/post insert/update/delete hooks
* Automatically generate insert/update/delete statements for a struct
* Automatic binding of auto increment PKs back to struct after insert
* Delete by primary key(s)
* Select by primary key(s)
* Optional trace sql logging
* Bind arbitrary SQL queries to a struct
* Bind slice to SELECT query results without type assertions
* Use positional or named bind parameters in custom SELECT queries
* Optional optimistic locking using a version column (for
  update/deletes)

## Installation

gorp uses go modules. 

## Versioning

Gorp tries to adhere to semantic versioning and we take backward compatibility seriously. I am hoping that I will not have to make backward incompatible change.  

All the DDL generation functionality is not covered by the above. You should only use DDL generation offered by Gorp in for quick POCs. You should not use this for schema migrations.

## Supported Go versions

This package is guaranteed to be compatible with the latest major
versions of Go.

Any earlier versions are only supported on a best effort basis and can
be dropped any time.  Go has a great compatibility promise. Upgrading
your program to a newer version of Go should never really be a
problem.


## Help/Support

Please raise an issue on this github repo.

## Quickstart

```go
package main

import (
    "database/sql"
    "github.com/mmrath/gorp"
    _ "github.com/mattn/go-sqlite3"
    "log"
    "time"
)

func main() {
    // initialize the DbMap
    dbmap := initDb()
    defer dbmap.Db.Close()

    // delete any existing rows
    err := dbmap.TruncateTables()
    checkErr(err, "TruncateTables failed")

    // create two posts
    p1 := newPost("Go 1.1 released!", "Lorem ipsum lorem ipsum")
    p2 := newPost("Go 1.2 released!", "Lorem ipsum lorem ipsum")

    // insert rows - auto increment PKs will be set properly after the insert
    err = dbmap.Insert(&p1, &p2)
    checkErr(err, "Insert failed")

    // use convenience SelectInt
    count, err := dbmap.SelectInt("select count(*) from posts")
    checkErr(err, "select count(*) failed")
    log.Println("Rows after inserting:", count)

    // update a row
    p2.Title = "Go 1.2 is better than ever"
    count, err = dbmap.Update(&p2)
    checkErr(err, "Update failed")
    log.Println("Rows updated:", count)

    // fetch one row - note use of "post_id" instead of "Id" since column is aliased
    //
    // Postgres users should use $1 instead of ? placeholders
    // See 'Known Issues' below
    //
    err = dbmap.SelectOne(&p2, "select * from posts where post_id=?", p2.Id)
    checkErr(err, "SelectOne failed")
    log.Println("p2 row:", p2)

    // fetch all rows
    var posts []Post
    err = dbmap.Select(&posts, "select * from posts order by post_id")
    checkErr(err, "Select failed")
    log.Println("All rows:")
    for x, p := range posts {
        log.Printf("    %d: %v\n", x, p)
    }

    // delete row by PK
    count, err = dbmap.Delete(&p1)
    checkErr(err, "Delete failed")
    log.Println("Rows deleted:", count)

    // delete row manually via Exec
    _, err = dbmap.Exec("delete from posts where post_id=?", p2.Id)
    checkErr(err, "Exec failed")

    // confirm count is zero
    count, err = dbmap.SelectInt("select count(*) from posts")
    checkErr(err, "select count(*) failed")
    log.Println("Row count - should be zero:", count)

    log.Println("Done!")
}

type Post struct {
    // db tag lets you specify the column name if it differs from the struct field
    Id      int64  `db:"post_id"`
    Created int64
    Title   string `db:",size:50"`               // Column size set to 50
    Body    string `db:"article_body,size:1024"` // Set both column name and size
}

func newPost(title, body string) Post {
    return Post{
        Created: time.Now().UnixNano(),
        Title:   title,
        Body:    body,
    }
}

func initDb() *gorp.DbMap {
    // connect to db using standard Go database/sql API
    // use whatever database/sql driver you wish
    db, err := sql.Open("sqlite3", "/tmp/post_db.bin")
    checkErr(err, "sql.Open failed")

    // construct a gorp DbMap
    dbmap := &gorp.DbMap{Db: db, Dialect: gorp.SqliteDialect{}}

    // add a table, setting the table name to 'posts' and
    // specifying that the Id property is an auto incrementing PK
    dbmap.AddTableWithName(Post{}, "posts").SetKeys(true, "Id")

    // create the table. in a production system you'd generally
    // use a migration tool, or create the tables via scripts
    err = dbmap.CreateTablesIfNotExists()
    checkErr(err, "Create tables failed")

    return dbmap
}

func checkErr(err error, msg string) {
    if err != nil {
        log.Fatalln(msg, err)
    }
}
```

## Examples

### Mapping structs to tables

First define some types:

```go
type Invoice struct {
    Id       int64
    Created  int64
    Updated  int64
    Memo     string
    PersonId int64
}

type Person struct {
    Id      int64
    Created int64
    Updated int64
    FName   string
    LName   string
}

// Example of using tags to alias fields to column names
// The 'db' value is the column name
//
// A hyphen will cause gorp to skip this field, similar to the
// Go json package.
//
// This is equivalent to using the ColMap methods:
//
//   table := dbmap.AddTableWithName(Product{}, "product")
//   table.ColMap("Id").Rename("product_id")
//   table.ColMap("Price").Rename("unit_price")
//   table.ColMap("IgnoreMe").SetTransient(true)
//
// You can optionally declare the field to be a primary key and/or autoincrement
//
type Product struct {
    Id         int64     `db:"product_id, primarykey, autoincrement"`
    Price      int64     `db:"unit_price"`
    IgnoreMe   string    `db:"-"`
}
```

Then create a mapper, typically you'd do this one time at app startup:

```go
// connect to db using standard Go database/sql API
// use whatever database/sql driver you wish
db, err := sql.Open("mymysql", "tcp:localhost:3306*mydb/myuser/mypassword")

// construct a gorp DbMap
dbmap := &gorp.DbMap{Db: db, Dialect: gorp.MySQLDialect{"InnoDB", "UTF8"}}

// register the structs you wish to use with gorp
// you can also use the shorter dbmap.AddTable() if you
// don't want to override the table name
//
// SetKeys(true) means we have a auto increment primary key, which
// will get automatically bound to your struct post-insert
//
t1 := dbmap.AddTableWithName(Invoice{}, "invoice_test").SetKeys(true, "Id")
t2 := dbmap.AddTableWithName(Person{}, "person_test").SetKeys(true, "Id")
t3 := dbmap.AddTableWithName(Product{}, "product_test").SetKeys(true, "Id")
```

### Struct Embedding

gorp supports embedding structs.  For example:

```go
type Names struct {
    FirstName string
    LastName  string
}

type WithEmbeddedStruct struct {
    Id int64
    Names
}

es := &WithEmbeddedStruct{-1, Names{FirstName: "Alice", LastName: "Smith"}}
err := dbmap.Insert(es)
```

See the `TestWithEmbeddedStruct` function in `gorp_test.go` for a full example.

### Create/Drop Tables ###

Automatically create / drop registered tables.  This is useful for unit tests
but is entirely optional.  You can of course use gorp with tables created manually,
or with a separate migration tool (like [sql-migrate](https://github.com/rubenv/sql-migrate), [goose](https://bitbucket.org/liamstask/goose) or [migrate](https://github.com/mattes/migrate)).

```go
// create all registered tables
dbmap.CreateTables()

// same as above, but uses "if not exists" clause to skip tables that are
// already defined
dbmap.CreateTablesIfNotExists()

// drop
dbmap.DropTables()
```

### SQL Logging

Optionally you can pass in a logger to trace all SQL statements.
I recommend enabling this initially while you're getting the feel for what
gorp is doing on your behalf.

Gorp defines a `GorpLogger` interface that Go's built in `log.Logger` satisfies.
However, you can write your own `GorpLogger` implementation, or use a package such
as `glog` if you want more control over how statements are logged.

```go
// Will log all SQL statements + args as they are run
// The first arg is a string prefix to prepend to all log messages
dbmap.TraceOn("[gorp]", log.New(os.Stdout, "myapp:", log.Lmicroseconds))

// Turn off tracing
dbmap.TraceOff()
```

### Insert

```go
// Must declare as pointers so optional callback hooks
// can operate on your data, not copies
inv1 := &Invoice{0, 100, 200, "first order", 0}
inv2 := &Invoice{0, 100, 200, "second order", 0}

// Insert your rows
err := dbmap.Insert(inv1, inv2)

// Because we called SetKeys(true) on Invoice, the Id field
// will be populated after the Insert() automatically
fmt.Printf("inv1.Id=%d  inv2.Id=%d\n", inv1.Id, inv2.Id)
```

### Update

Continuing the above example, use the `Update` method to modify an Invoice:

```go
// count is the # of rows updated, which should be 1 in this example
count, err := dbmap.Update(inv1)
```

### Delete

If you have primary key(s) defined for a struct, you can use the `Delete`
method to remove rows:

```go
count, err := dbmap.Delete(inv1)
```

### Select by Key

Use the `Get` method to fetch a single row by primary key.  It returns
`error` (`sql.ErrNoRows`) if no row is found.

```go
// fetch Invoice with Id=99
obj := &Invoice{}
err := dbmap.Get(obj, 99)
```

### Ad Hoc SQL

#### SELECT

`Select()` and `SelectOne()` provide a simple way to bind arbitrary queries to a slice
or a single struct.

```go
// Select a slice - first return value is not needed when a slice pointer is passed to Select()
var posts []Post
err := dbmap.Select(&posts, "select * from post order by id")

// You can also use primitive types
var ids []string
err := dbmap.Select(&ids, "select id from post")

// Select a single row.
// Returns an error if no row found, or if more than one row is found
var post Post
err := dbmap.SelectOne(&post, "select * from post where id=?", id)
```

Want to do joins?  Just write the SQL and the struct. gorp will bind them:

```go
// Define a type for your join
// It *must* contain all the columns in your SELECT statement
//
// The names here should match the aliased column names you specify
// in your SQL - no additional binding work required.  simple.
//
type InvoicePersonView struct {
    InvoiceId   int64
    PersonId    int64
    Memo        string
    FName       string
}

// Create some rows
p1 := &Person{0, 0, 0, "bob", "smith"}
err = dbmap.Insert(p1)
checkErr(err, "Insert failed")

// notice how we can wire up p1.Id to the invoice easily
inv1 := &Invoice{0, 0, 0, "xmas order", p1.Id}
err = dbmap.Insert(inv1)
checkErr(err, "Insert failed")

// Run your query
query := "select i.Id InvoiceId, p.Id PersonId, i.Memo, p.FName " +
	"from invoice_test i, person_test p " +
	"where i.PersonId = p.Id"

// pass a slice to Select()
var list []InvoicePersonView
err := dbmap.Select(&list, query)

// this should test true
expected := InvoicePersonView{inv1.Id, p1.Id, inv1.Memo, p1.FName}
if reflect.DeepEqual(list[0], expected) {
    fmt.Println("Woot! My join worked!")
}
```

#### SELECT string or int64

gorp provides a few convenience methods for selecting a single string or int64.

```go
// select single int64 from db (use $1 instead of ? for postgresql)
i64, err := dbmap.SelectInt("select count(*) from foo where blah=?", blahVal)

// select single string from db:
s, err := dbmap.SelectStr("select name from foo where blah=?", blahVal)

```

#### Named bind parameters

You may use a map or struct to bind parameters by name.  This is currently
only supported in SELECT queries.

```go
err := dbm.Select(&dest, "select * from Foo where name = :name and age = :age", map[string]interface{}{
  "name": "Rob",
  "age": 31,
})
```

#### UPDATE / DELETE

You can execute raw SQL if you wish.  Particularly good for batch operations.

```go
res, err := dbmap.Exec("delete from invoice_test where PersonId=?", 10)
```

### Transactions

You can batch operations into a transaction:

```go
func InsertInv(dbmap *DbMap, inv *Invoice, per *Person) error {
    // Start a new transaction
    trans, err := dbmap.Begin()
    if err != nil {
        return err
    }

    err = trans.Insert(per)
    checkErr(err, "Insert failed")

    inv.PersonId = per.Id
    err = trans.Insert(inv)
    checkErr(err, "Insert failed")

    // if the commit is successful, a nil error is returned
    return trans.Commit()
}
```

### Hooks

Use hooks to update data before/after saving to the db. Good for timestamps:

```go
// implement the PreInsert and PreUpdate hooks
func (i *Invoice) PreInsert(s gorp.SqlExecutor) error {
    i.Created = time.Now().UnixNano()
    i.Updated = i.Created
    return nil
}

func (i *Invoice) PreUpdate(s gorp.SqlExecutor) error {
    i.Updated = time.Now().UnixNano()
    return nil
}

// You can use the SqlExecutor to cascade additional SQL
// Take care to avoid cycles. gorp won't prevent them.
//
// Here's an example of a cascading delete
//
func (p *Person) PreDelete(s gorp.SqlExecutor) error {
    query := "delete from invoice_test where PersonId=?"
    
    _, err := s.Exec(query, p.Id)
    
    if err != nil {
        return err
    }
    return nil
}
```

Full list of hooks that you can implement:

    PostGet
    PreInsert
    PostInsert
    PreUpdate
    PostUpdate
    PreDelete
    PostDelete

    All have the same signature.  for example:

    func (p *MyStruct) PostUpdate(s gorp.SqlExecutor) error

### Optimistic Locking

#### Note that this behaviour has changed in v2. See [Migration Guide](#migration-guide).

gorp provides a simple optimistic locking feature, similar to Java's
JPA, that will raise an error if you try to update/delete a row whose
`version` column has a value different than the one in memory.  This
provides a safe way to do "select then update" style operations
without explicit read and write locks.

```go
// Version is an auto-incremented number, managed by gorp
// If this property is present on your struct, update
// operations will be constrained
//
// For example, say we defined Person as:

type Person struct {
    Id       int64
    Created  int64
    Updated  int64
    FName    string
    LName    string

    // automatically used as the Version col
    // use table.SetVersionCol("columnName") to map a different
    // struct field as the version field
    Version  int64
}

p1 := &Person{0, 0, 0, "Bob", "Smith", 0}
err = dbmap.Insert(p1)  // Version is now 1
checkErr(err, "Insert failed")
p2 := &Person{}
err := dbmap.Get(p2, p1.Id)
p2.LName = "Edwards"
_,err = dbmap.Update(p2)  // Version is now 2
checkErr(err, "Update failed")

p1.LName = "Howard"

// Raises error because p1.Version == 1, which is out of date
count, err := dbmap.Update(p1)
_, ok := err.(gorp.OptimisticLockError)
if ok {
    // should reach this statement

    // in a real app you might reload the row and retry, or
    // you might propegate this to the user, depending on the desired
    // semantics
    fmt.Printf("Tried to update row with stale data: %v\n", err)
} else {
    // some other db error occurred - log or return up the stack
    fmt.Printf("Unknown db err: %v\n", err)
}
```
### Adding INDEX(es) on column(s) beyond the primary key ###

Indexes are frequently critical for performance. Here is how to add
them to your tables.

NB: SqlServer and Oracle need testing and possible adjustment to the
CreateIndexSuffix() and DropIndexSuffix() methods to make AddIndex()
work for them.

In the example below we put an index both on the Id field, and on the
AcctId field.

```
type Account struct {
	Id      int64
	AcctId  string // e.g. this might be a long uuid for portability
}

// indexType (the 2nd param to AddIndex call) is "Btree" or "Hash" for MySQL.
// demonstrate adding a second index on AcctId, and constrain that field to have unique values.
dbm.AddTable(iptab.Account{}).SetKeys(true, "Id").AddIndex("AcctIdIndex", "Btree", []string{"AcctId"}).SetUnique(true)

err = dbm.CreateTablesIfNotExists()
checkErr(err, "CreateTablesIfNotExists failed")

err = dbm.CreateIndex()
checkErr(err, "CreateIndex failed")

```
Check the effect of the CreateIndex() call in mysql:
```
$ mysql

MariaDB [test]> show create table Account;
+---------+--------------------------+
| Account | CREATE TABLE `Account` (
  `Id` bigint(20) NOT NULL AUTO_INCREMENT,
  `AcctId` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`Id`),
  UNIQUE KEY `AcctIdIndex` (`AcctId`) USING BTREE   <<<--- yes! index added.
) ENGINE=InnoDB DEFAULT CHARSET=utf8 
+---------+--------------------------+

```


## Database Drivers

gorp uses the Go 1 `database/sql` package.  A full list of compliant
drivers is available here:

http://code.google.com/p/go-wiki/wiki/SQLDrivers

Sadly, SQL databases differ on various issues. gorp provides a Dialect
interface that should be implemented per database vendor.  Dialects
are provided for:

* MySQL
* PostgreSQL
* sqlite3

Each of these three databases pass the test suite.  See `gorp_test.go`
for example DSNs for these three databases.

Support is also provided for:

* Oracle (contributed by @klaidliadon)
* SQL Server (contributed by @qrawl) - use driver:
  github.com/denisenkom/go-mssqldb

Note that these databases are not covered by CI and I (@coopernurse)
have no good way to test them locally.  So please try them and send
patches as needed, but expect a bit more unpredicability.

## Sqlite3 Extensions

In order to use sqlite3 extensions you need to first register a custom driver:

```go
import (
	"database/sql"

	// use whatever database/sql driver you wish
	sqlite "github.com/mattn/go-sqlite3"
)

func customDriver() (*sql.DB, error) {

	// create custom driver with extensions defined
	sql.Register("sqlite3-custom", &sqlite.SQLiteDriver{
		Extensions: []string{
			"mod_spatialite",
		},
	})

	// now you can then connect using the 'sqlite3-custom' driver instead of 'sqlite3'
	return sql.Open("sqlite3-custom", "/tmp/post_db.bin")
}
```

## Known Issues

### SQL placeholder portability

Different databases use different strings to indicate variable
placeholders in prepared SQL statements.  Unlike some database
abstraction layers (such as JDBC), Go's `database/sql` does not
standardize this.

SQL generated by gorp in the `Insert`, `Update`, `Delete`, and `Get`
methods delegates to a Dialect implementation for each database, and
will generate portable SQL.

Raw SQL strings passed to `Exec`, `Select`, `SelectOne`, `SelectInt`,
etc will not be parsed.  Consequently you may have portability issues
if you write a query like this:

```go 
// works on MySQL and Sqlite3, but not with Postgresql err :=
dbmap.SelectOne(&val, "select * from foo where id = ?", 30)
```

In `Select` and `SelectOne` you can use named parameters to work
around this.  The following is portable:

```go 
err := dbmap.SelectOne(&val, "select * from foo where id = :id",
map[string]interface{} { "id": 30})
```

Additionally, when using Postgres as your database, you should utilize
`$1` instead of `?` placeholders as utilizing `?` placeholders when
querying Postgres will result in `pq: operator does not exist`
errors. Alternatively, use `dbMap.Dialect.BindVar(varIdx)` to get the
proper variable binding for your dialect.

### time.Time and time zones

gorp will pass `time.Time` fields through to the `database/sql`
driver, but note that the behavior of this type varies across database
drivers.

MySQL users should be especially cautious.  See:
https://github.com/ziutek/mymysql/pull/77

To avoid any potential issues with timezone/DST, consider:

- Using an integer field for time data and storing UNIX time.
- Using a custom time type that implements some SQL types:
  - [`"database/sql".Scanner`](https://golang.org/pkg/database/sql/#Scanner)
  - [`"database/sql/driver".Valuer`](https://golang.org/pkg/database/sql/driver/#Valuer)

## Running the tests

The included tests may be run against MySQL, Postgresql, or sqlite3.
You must set two environment variables so the test code knows which
driver to use, and how to connect to your database.

```sh
# MySQL example:
export GORP_TEST_DSN=gomysql_test/gomysql_test/abc123
export GORP_TEST_DIALECT=mysql

# run the tests
go test

# run the tests and benchmarks
go test -bench="Bench" -benchtime 10
```

Valid `GORP_TEST_DIALECT` values are: "mysql"(for mymysql),
"gomysql"(for go-sql-driver), "postgres", "sqlite" See the
`test_all.sh` script for examples of all 3 databases.  This is the
script I run locally to test the library.

## Performance

I have not benchmarked my fork yet. But the below is from original repo.

gorp uses reflection to construct SQL queries and bind parameters.
See the BenchmarkNativeCrud vs BenchmarkGorpCrud in gorp_test.go for a
simple perf test.  On my MacBook Pro gorp is about 2-3% slower than
hand written SQL.


## Authors and Contributors
https://github.com/mmrath/gorp/graphs/contributors

I have copied some files from PRs manually as it was difficult to apply the PR on my modified code.
I try to list the PRs here
https://github.com/go-gorp/gorp/pull/407


