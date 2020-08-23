## 理解题意: 
根据题目意思，需要在事务开启时，打印hello transaction.根据架构图，不难得需要确认的代码在tidb (无状态sql层).

事务启动方式有两种，一种是显式启动事务,主动声明语句start transaction  , 第二种是 隐式事务, 在单句写操作执行时 update, insert , 实际上也启动了事务, 只是事务只有一条执行语句.

## 启动服务器:

**1.启动pd:**
nohup /home/www/github/pd/bin/pd-server --data-dir=/home/www/github/pd --log-file=/data/log/pd/pd.log -L info  &


**2.启动tidb-server:**
nohup /home/www/github/tidb/bin/tidb-server --store=tikv --path='127.0.0.1:2379' --log-file=/data/log/tidb/tidb.log -L info > /data/log/tidb/tidb_stdout.log &

**3.启动3个tikv-server:**
nohup /home/www/github/tikv/target/release/tikv-server --pd='127.0.0.1:2379' --data-dir=/home/www/github/tikv --log-file=/data/log/tikv/tikv.log.1 -L info -A 127.0.0.1:9000  &

nohup /home/www/github/tikv-server2/target/release/tikv-server --pd='127.0.0.1:2379' --data-dir=/home/www/github/tikv-server2 --log-file=/data/log/tikv/tikv.log.2 -L info -A 127.0.0.1:9001 &

nohup /home/www/github/tikv-server3/target/release/tikv-server --pd='127.0.0.1:2379' --data-dir=/home/www/github/tikv-server3 --log-file=/data/log/tikv/tikv.log.3 -L info -A 127.0.0.1:9002 &

## 定位语句执行代码链路: 

**1.通过架构文章缩小其对应的所在的package或者类:** 在文章https://pingcap.com/blog-cn/tidb-source-code-reading-2/ 源码架构汇总文章中，搜索transaction 关键词，点击后弹出https://github.com/pingcap/tidb/blob/source-code/kv/kv.go#L121 ，里面有一个transaction 类，可以猜测这是统一掌管事务的接口，但是这个类是一个interface, 我们需要找到其实现类.
然后通过搜索实现了其方法的类, 如: grep ") StartTS(" * -R  -n  定位到store/tikv/txn.go

**2.打印其运行时堆栈:** 根据第一步，不难发现 newTikvTxnWithStartTS 函数是事务的创建, 在里面加上 fmt.Println(string(debug.Stack())) 在标准输出打印其调用堆栈。 

**3.触发用户显式声明事务链路:** 使用mysql client连接，通过对比触发前，触发后的新增堆栈 定位到其新增堆栈.(cat  tidb_stdout.log | grep ".go:" | grep "[^ ]*" -o | grep ":" | sort | uniq > no_trx; diff no_trx with_trx) 
堆栈如下:
```
new trx:  goroutine 1561 [running]:
runtime/debug.Stack(0x5cfc8b3d4f40001, 0x0, 0x0)
        /usr/local/go/src/runtime/debug/stack.go:24 +0x9d
github.com/pingcap/tidb/store/tikv.newTikvTxnWithStartTS(0xc000162500, 0x5cfc8b3d4f40001, 0x5cfc8b330b83816, 0x0, 0x0, 0x0)
        /home/www/github/tidb/store/tikv/txn.go:99 +0x34
github.com/pingcap/tidb/store/tikv.newTiKVTxn(0xc000162500, 0x2, 0x2, 0xc0014fc160)
        /home/www/github/tidb/store/tikv/txn.go:93 +0x182
github.com/pingcap/tidb/store/tikv.(*tikvStore).Begin(0xc000162500, 0x35f56a5, 0x29, 0xc001502380, 0x2)
        /home/www/github/tidb/store/tikv/kv.go:282 +0x2f
github.com/pingcap/tidb/session.(*session).NewTxn(0xc001044b40, 0x3b5ce80, 0xc001510090, 0xc0011d05e0, 0x12bd818)
        /home/www/github/tidb/session/session.go:1507 +0x6d
github.com/pingcap/tidb/executor.(*SimpleExec).executeBegin(0xc001439ae0, 0x3b5ce80, 0xc001510090, 0xc001601ef0, 0x0, 0xc0011d0658)
        /home/www/github/tidb/executor/simple.go:557 +0x28e
github.com/pingcap/tidb/executor.(*SimpleExec).Next(0xc001439ae0, 0x3b5ce80, 0xc001510090, 0xc0014be640, 0x0, 0x0)
        /home/www/github/tidb/executor/simple.go:116 +0x5fd
github.com/pingcap/tidb/executor.Next(0x3b5ce80, 0xc001510090, 0x3b67800, 0xc001439ae0, 0xc0014be640, 0x0, 0x0)
        /home/www/github/tidb/executor/executor.go:269 +0x11d
github.com/pingcap/tidb/executor.(*ExecStmt).handleNoDelayExecutor(0xc0014655f0, 0x3b5ce80, 0xc001510090, 0x3b67800, 0xc001439ae0, 0x0, 0x0, 0x0, 0x0)
        /home/www/github/tidb/executor/adapter.go:502 +0x2d6
github.com/pingcap/tidb/executor.(*ExecStmt).handlePessimisticDML(0xc0014655f0, 0x3b5ce80, 0xc001510090, 0x3b67800, 0xc001439ae0, 0x11, 0x5cfc8aee6740001)
        /home/www/github/tidb/executor/adapter.go:521 +0x142
github.com/pingcap/tidb/executor.(*ExecStmt).handleNoDelay(0xc0014655f0, 0x3b5ce80, 0xc001510090, 0x3b67800, 0xc001439ae0, 0x5802201, 0x3, 0x0, 0x33a0e60, 0x8, ...)
        /home/www/github/tidb/executor/adapter.go:382 +0xc7
github.com/pingcap/tidb/executor.(*ExecStmt).Exec(0xc0014655f0, 0x3b5ce80, 0xc001510090, 0x0, 0x0, 0x0, 0x0)
        /home/www/github/tidb/executor/adapter.go:352 +0x40e
github.com/pingcap/tidb/session.runStmt(0x3b5ce80, 0xc001601f20, 0xc001044b40, 0x3b66d00, 0xc0014655f0, 0x0, 0x0, 0x0, 0x0)
        /home/www/github/tidb/session/session.go:1197 +0x256
github.com/pingcap/tidb/session.(*session).ExecuteStmt(0xc001044b40, 0x3b5ce80, 0xc001601f20, 0x3b65a00, 0xc001601ef0, 0x0, 0x0, 0x0, 0x0)
        /home/www/github/tidb/session/session.go:1162 +0x7bc
github.com/pingcap/tidb/server.(*TiDBContext).ExecuteStmt(0xc001617b90, 0x3b5ce80, 0xc001601f20, 0x3b65a00, 0xc001601ef0, 0xc00153a060, 0x3b5ce80, 0xc001601f20, 0xc0011d1201)
        /home/www/github/tidb/server/driver_tidb.go:198 +0x68
github.com/pingcap/tidb/server.(*clientConn).handleStmt(0xc001644840, 0x3b5ce80, 0xc001601f20, 0x3b65a00, 0xc001601ef0, 0x5822e28, 0x0, 0x0, 0x1, 0x0, ...)
        /home/www/github/tidb/server/conn.go:1443 +0xec
github.com/pingcap/tidb/server.(*clientConn).handleQuery(0xc001644840, 0x3b5ce80, 0xc001617800, 0xc00121e981, 0x11, 0x0, 0x0)
        /home/www/github/tidb/server/conn.go:1336 +0x3b2
github.com/pingcap/tidb/server.(*clientConn).dispatch(0xc001644840, 0x3b5ce80, 0xc001617800, 0xc00121e980, 0x12, 0x11, 0x0, 0x0)
        /home/www/github/tidb/server/conn.go:925 +0x4a0
github.com/pingcap/tidb/server.(*clientConn).Run(0xc001644840, 0x3b5ce80, 0xc001617800)
        /home/www/github/tidb/server/conn.go:727 +0x27f
github.com/pingcap/tidb/server.(*Server).onConn(0xc001070fd0, 0xc001644840)
        /home/www/github/tidb/server/server.go:418 +0xb18
created by github.com/pingcap/tidb/server.(*Server).Run
        /home/www/github/tidb/server/server.go:333 +0x709
```

**4.跟踪链路:** 从tidb/server/server.go:333开始跟，已经比较明显了。大致链路: 
accept 内核连接后 -> 生成clientConn 对象 -> 开启协程(没有协程池):go clientConn::Run(clientConn) -> 循环readPacket -> clientConn::dispatch(packet) 

## 总结: 
最后在clientConn::dispatch里面，在判断其cmd是 mysql.ComQuery 且 dataStr为 start transaction时增加输出信息 hello transaction.
