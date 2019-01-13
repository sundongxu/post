---
title: "database/sql: Stmt的使用以及坑"
date: 2019-01-14T00:37:49+08:00
categories:
    - Golang
tags: 
    - Golang
    - database/sql
---

## 基本知识

众所周知，Golang 操作数据库，是通过 `database/sql` 包，以及第三方的实现了 `database/sql/driver` 接口的数据库驱动包来共同完成的。

其中 `database/sql/driver` 中的接口 `Conn` 和 `Stmt`，官方交给第三方实现驱动，并且是协程不安全的。官方实现的 `database/sql` 包中的 `DB` 和 `Stmt` 是协程安全的，因为内部实现是连接池。

## 如何使用

`database/sql` 的 MySQL 驱动的[使用范例](https://github.com/go-sql-driver/mysql/wiki/Examples) 基本类似于

```
// Insert
stmtIns, err := db.Prepare("INSERT INTO squareNum VALUES( ?, ? )")
if err != nil {
    panic(err.Error()) // proper error handling instead of panic in your app
}
defer stmtIns.Close()

stmtIns.Exec(i, j)

// Query
stmtOut, err := db.Prepare("SELECT squareNumber FROM squarenum WHERE number = ?")
if err != nil {
    panic(err.Error()) // proper error handling instead of panic in your app
}
defer stmtOut.Close()

stmtOut.QueryRow(13).Scan(&squareNum)
```

还有另外一种直接使用

```
// Insert
db.Exec("INSERT INTO squareNum VALUES( ?, ? )", i, j)

// Query
db.Query("SELECT squareNumber FROM squarenum WHERE number = ?", 13)
```

并且官方的 wiki 里面也推荐第一种 Prepare 过程，这样做的方式可以一定程度上防止SQL注入。

## 内部处理流程介绍

### database/sql 处理逻辑

都是程序员了，直接上代码，这里 `Query` 和 `Exec` 其实代码逻辑比较类似，可以类比，下面简单介绍一下 `Query` 是如何实现的。

`Go 1.10` 标准包的逻辑，默认的执行过程中，`Query` 语句会创建预处理语句，然后在 `Stmt` 上执行语句。

不过如果使用的驱动实现特定的接口，会执行驱动自己的 `Query` 逻辑，而不是使用标准包的逻辑

```
func (db *DB) query(ctx context.Context, query string, args []interface{}, strategy connReuseStrategy) (*Rows, error) {
	dc, err := db.conn(ctx, strategy)
	if err != nil {
		return nil, err
	}

	return db.queryDC(ctx, nil, dc, dc.releaseConn, query, args)
}
```

```
func (db *DB) queryDC(ctx, txctx context.Context, dc *driverConn, releaseConn func(error), query string, args []interface{}) (*Rows, error) {
	queryerCtx, ok := dc.ci.(driver.QueryerContext)
	var queryer driver.Queryer
	if !ok {
		queryer, ok = dc.ci.(driver.Queryer)
	}
	if ok {
		var nvdargs []driver.NamedValue
		var rowsi driver.Rows
		var err error
		withLock(dc, func() {
			nvdargs, err = driverArgsConnLocked(dc.ci, nil, args)
			if err != nil {
				return
			}
			rowsi, err = ctxDriverQuery(ctx, queryerCtx, queryer, query, nvdargs)
		})
		if err != driver.ErrSkip {
			if err != nil {
				releaseConn(err)
				return nil, err
			}
			// Note: ownership of dc passes to the *Rows, to be freed
			// with releaseConn.
			rows := &Rows{
				dc:          dc,
				releaseConn: releaseConn,
				rowsi:       rowsi,
			}
			rows.initContextClose(ctx, txctx)
			return rows, nil
		}
	}

	var si driver.Stmt
	var err error
	withLock(dc, func() {
		si, err = ctxDriverPrepare(ctx, dc.ci, query)
	})
	if err != nil {
		releaseConn(err)
		return nil, err
	}

	ds := &driverStmt{Locker: dc, si: si}
	rowsi, err := rowsiFromStatement(ctx, dc.ci, ds, args...)
	if err != nil {
		ds.Close()
		releaseConn(err)
		return nil, err
	}

	// Note: ownership of ci passes to the *Rows, to be freed
	// with releaseConn.
	rows := &Rows{
		dc:          dc,
		releaseConn: releaseConn,
		rowsi:       rowsi,
		closeStmt:   ds,
	}
	rows.initContextClose(ctx, txctx)
	return rows, nil
}
```

### driver 处理逻辑

我们使用的 `MySQL driver` 实现了 `Queryer` 接口。进入 `Mysql` 实现逻辑里面一探究竟，其实除了对一些字段特性的处理，也没啥逻辑，还是走到了标准接口 `Query`。逻辑还是比较简单，针对标准包的实现有一些优化，集中在查询语句的处理。

官方提供的驱动，仍然在贯彻标准包的思想，创建预处理语句，然后在 `Stmt` 上执行语句。

不过需要注意的是，`Go` 官方给出的 `Mysql Driver` 在 v1.3 和之前的版本，在对于 `Prepare` 过程的处理上有着一定差别。

v1.0 ~ v1.2 版本上的处理逻辑基本相同，在不提供 args 时，会省去 `Prepare` 过程，否则必定去通过 Server 端来获得预处理语句。

``` v1.0
func (mc *mysqlConn) Query(query string, args []driver.Value) (driver.Rows, error) {
	if len(args) == 0 { // no args, fastpath
		// Send command
		err := mc.writeCommandPacketStr(comQuery, query)
		if err == nil {
			// Read Result
			var resLen int
			resLen, err = mc.readResultSetHeaderPacket()
			if err == nil {
				rows := &mysqlRows{mc, false, nil, false}

				if resLen > 0 {
					// Columns
					rows.columns, err = mc.readColumns(resLen)
				}
				return rows, err
			}
		}

		return nil, err
	}

	// with args, must use prepared stmt
	return nil, driver.ErrSkip
}
```

v1.3 版本上修改了必须通过 DB Server 来获得预处理语句的逻辑，倾向于使用 Client 端来提前生成，来减少处理耗时

```v1.3
func (mc *mysqlConn) Query(query string, args []driver.Value) (driver.Rows, error) {
	if mc.netConn == nil {
		errLog.Print(ErrInvalidConn)
		return nil, driver.ErrBadConn
	}
	if len(args) != 0 {
		if !mc.cfg.InterpolateParams {
			return nil, driver.ErrSkip
		}
		// try client-side prepare to reduce roundtrip
		prepared, err := mc.interpolateParams(query, args)
		if err != nil {
			return nil, err
		}
		query = prepared
		args = nil
	}
	// Send command
	err := mc.writeCommandPacketStr(comQuery, query)
	if err == nil {
		// Read Result
		var resLen int
		resLen, err = mc.readResultSetHeaderPacket()
		if err == nil {
			rows := new(textRows)
			rows.mc = mc

			if resLen == 0 {
				// no columns, no more data
				return emptyRows{}, nil
			}
			// Columns
			rows.columns, err = mc.readColumns(resLen)
			return rows, err
		}
	}
	return nil, err
}
```

### Prepare 与 Stmt
创建 `stmt` 的 `Prepare` 方式是 `Golang` 的一个设计，其目的是 Prepare once, execute many times。为了批量执行 sql 语句。但是通常会造成所谓的三次网络请求（three network round-trips）。即 `preparing`、`executing` 和 `closing` 三次请求。

对于大多数数据库，`Prepare` 的过程都是，先发送一个带占位符的 sql 语句到服务器，服务器返回一个statement id，然后再把这个id和参数发送给服务器执行，最后再发送关闭statement命令。

`Golang` 实现了连接池，处理 `Prepare` 方式也需要特别注意。调用 `Prepare` 方法返回 `stmt` 的时候，会在某个空闲的连接上进行 `Prepare` 语句，然后就把连接释放回到连接池，可是会记住这个连接，当需要执行参数的时候，就再次找到之前记住的连接进行执行，等到 `stmt.Close` 调用的时候，再释放该连接。

在执行参数的时候，如果记住的连接正处于忙碌阶段，将会从新选一个新的空闲连接进行 `re-prepare`。当然，即使是重新 prepare，同样也会遇到刚才的问题。那么将会一而再再而三的进行 reprepare。直到找到空闲连接进行查询的时候。

这种情况将会导致 leak 连接的情况，尤其是再高并发的情景。将会导致大量的 `Prepare` 过程。因此使用`Stmt` 的情况需要仔细考虑应用场景，通常在应用程序中。多次执行同一个sql语句的情况并不多，因此减少prepare 语句的使用。

### 如何选择 Exec／Query 还是使用 Stmt

那么大家也看出来了，如果不是批量的操作，是没必要使用 `db.Papare` 方法的，否则即多了 `Stmt` 创建和关闭的性能开销，又多写了两行代码，有点得不偿失。如果是批量的操作，那么毋庸置疑，肯定是 `db.Papare` 拿到 `Stmt`，再由 `Stmt` 去执行 sql，这样保证批量操作只进行一次预处理。

而且对于不同版本的 `Driver` ，使用上也是有使用的选择余地，下面给个推荐

批量操作：
- v1.0 ~ v1.2 使用 `Stmt`
- v1.3 使用 `Stmt`

单次操作:
- v1.0 ~ v1.2 使用 `Exec(query)` & `Query(query)`
- v1.3 `Exec(query, args...)` & `Exec(query)` & `Query(query)` & `Query(query, args...)`

## 发现的问题

1.连接消耗问题

默认 DB 默认的最大 open 连接数是0。在数据库操作很频繁的实际使用场景中，尤其是一波又一波访问高峰不间断来临的时候，数据库性能会不断的消耗在连接的创建和销毁上，这是很拖累数据和和机器的，所以我们根据提供 的 `SetMaxIdleConns` 和 `SetMaxOpenConns`，设置合理的值之后，可以保持机器的稳定

2.避免 Stmt 的滥用

如果直接使用 Stmt 来进行简单查询，会增加数据库 Stmt 开启和关闭，增加延时消耗，而且使用不当，高并发情况下会导致数据库 Stmt 数量暴增，直到达到限制报错。

- 如果 sql driver 版本过老的情况（如第三方依赖限制），建议查询还是直接使用 Sql 语句来避免 `Prepare` 过程。

- 较新的 sql driver 版本倾向于 Client 端进行预处理，没有限制。

- 需要大量执行相同 Sql 的情况下，可以提前生成 Stmt ，来避免重复的 Stmt 过程，记得必须关闭 Stmt。
