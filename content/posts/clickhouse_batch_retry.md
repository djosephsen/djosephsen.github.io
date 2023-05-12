---
title: "Clickhouse_batch_retry"
date: 2023-03-23T18:15:52-05:00
draft: true
---

# Implementing Batch Retry in clickhouse-go

If you want native protocol support without the http overhead, there are only a few ways you can write to clickhouse from within your Go program's runtime. 

 * [ch-go](https://github.com/ClickHouse/ch-go) the low-level protocol implementation, supported by clickhouse.
 * [clickhouse-go](https://github.com/ClickHouse/clickhouse-go) the high-level sql style client built on ch-go by the clickhouse folks, and recommended for production use.
 * [uptrace/go-clickhouse](https://github.com/uptrace/go-clickhouse) even higher level, third-party, uses reflect and supported by this dude named Vlad. 

If you choose the clickhouse-go client, you'll have a few more choices around how to actually implement the writes: 

## [Batch Inserts](https://clickhouse.com/docs/en/cloud/bestpractices/bulk-inserts)
The recommended way is to collect your rows client-side into a batch, and insert them all at once at a recommended rate of no more than 1 insert per second. Even if that single insert operation contains hundreds of millions of rows.

## [Async Inserts](https://clickhouse.com/docs/en/optimize/asynchronous-inserts)
If you can't manage client-side batches because your workers are stateless (or whatever), you can tell clickhouse to manage the batches for you server-side in memory. This is called "Async" mode. You just yeet your rows in there at the speed of light and once a second Clickhouse treats them like a batch and performs the insert for you.

## [Buffered Tables](https://clickhouse.com/docs/en/engines/table-engines/special/buffer)
You can also use the buffered table engine, usually in the event that you want your cluster to remain in _not async_ mode, but you need one or two yolo tables that act like async is enabled.

## The clickhouse-go `Batch` type
If you choose to go with clickhouse-go and batch inserts (the recommended client in the recommended mode) you will encounter [type Batch](https://github.com/ClickHouse/clickhouse-go/blob/main/lib/driver/driver.go#L79), which is an interface defined thusly:

```
Batch interface {
		Abort() error
		Append(v ...interface{}) error
		AppendStruct(v interface{}) error
		Column(int) BatchColumn
		Flush() error
		Send() error
		IsSent() bool
	}
```

The general idea here is that you call Conn.PrepareBatch() to instantiate this Batch type. Then you stuff all of your rows into the batch using the `Append()` or `AppendStruct()` methods you see above, and then (once per second or so) call the `Send()` method, which, as you can see, returns an error. Once the batch is sent, you GC it and prepare a new one (they can't be appended to after `Send()` is called.

I'll spare you the code snippet of this type in action, but if you want to see it working in the wild check out the [batch example](https://github.com/ClickHouse/clickhouse-go/blob/main/examples/clickhouse_api/batch.go) in the clickhouse-go repo.

One of the things that isn't immediatly obvious about the Batch type is that it carries a `net.conn` around inside it. The connection this batch will use to perform your insert was baked into it when you called `PrepareBatch()` to create it. You can see this in the [PrepareBatch()](https://github.com/ClickHouse/clickhouse-go/blob/main/clickhouse.go#LL150) method, which immediatly calls `ch.acquire()` which, in turn, calls `net.dial`.

The upshot of this is the situation you'll find yourself in when you get a non-nill return from `Batch.Send()`. This could happen for myriad reasons. I personally sometimes get them when ZK nodes hold a leader-election and the clickhouse tables go into read-only mode for a moment or two. 

At this point in the runtime, you'll be holding two things, a failed `Batch` and a 
