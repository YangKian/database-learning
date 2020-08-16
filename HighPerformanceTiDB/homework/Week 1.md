# Week 1

## 任务

>分值：200
>
>题目描述：
>
>本地下载 TiDB，TiKV，PD 源代码，改写源码并编译部署以下环境：
>
>* 1 TiDB
>* 1 PD
>* 3 TiKV
>
>改写后：使得 TiDB 启动事务时，能打印出一个 “hello transaction” 的 日志
>
>输出：一篇文章介绍以上过程
>
>截止时间：本周日 24:00:00（逾期不给分）

## 实验环境

- 系统：Ubuntu 20.04
- go：1.14.6
- rust：1.47.0-nightly

## 操作步骤

### 测试集群搭建

- 编译部署，使用 `tiup` 来辅助完成：

  - `tiup playground` 可以启动一个默认的测试环境
  - `tiup playground --db=1 --pd=1 --kv=3` 配置个组件的数量，启动测试集群
  - `--db.binpath /XXX/tidb-server` 可以使用自己编译的实例替换默认实例，同理 pd 和 kv 也有类似的配置项

- 下载 3 个组件的源码，分别编译后，使用 `tiup` 启动编译好的三个组件实例

  ```bash
  tiup playground \
  	--db.binpath /dbpath/bin/tidb-server --db=1 \
  	--kv.binpath /kvpath/target/release/tikv-server --kv=3 \
  	--pd.binpath /pdpath/bin/pd-serever --pd=1
  ```

  <img src="pic\w1-start.png" alt="avatar" style="zoom:80%;" />

- 连接 MySQL：`mysql --host 127.0.0.1 --port 4000 -u root`

  ```mysql
  mysql> SELECT `TYPE`, `INSTANCE`, `STATUS_ADDRESS`, `VERSION` FROM `INFORMATION_SCHEMA`.`CLUSTER_INFO`;
  
  
  ```

### 源码修改

- 要在事务启动的地方打印日志，则需要先定位到事务启动部分的代码，想到两个点：

  - sql 语句解析后，执行 `BEGIN`、`START TRANSACTION` 语句的代码段；
  - 事务开始时有没有什么特别的标志？
    - 粗略地翻了下 Percolator 的论文，发现所有事务的开始都需要获取一个全局时间戳 `timestamp orcale`，可以尝试在代码中找获取时间戳的地方；
      - 不能直接写在获取时间戳的函数里，因为除了事务开始时会拿时间戳，提交时还会拿一次。
    - TiDB 可以看做是 TiKV 的客户端，那么执行事务的操作最终是要与 TiKV 进行交互的，TiKV 应该有提供相关的 API 给 TiDB 使用，尝试找一找类似的地方；

- 有了大概思路，开始尝试验证：

  - 先开始找执行 `BEGIN`、`START TRANSACTION` 语句的代码段：

  - 因为是执行语句，因此先去看了 `executor` 文件夹，在里面发现很多 `*ast.XXX` 样的字段，应该是抽象语法树解析后的标记，发现有 `*ast.BeginStmt`，表示的就是 `Begin`；

  - 接着搜 `*ast.BeginStmt` 后在 `executor/simple.go` 中找到方法：

     `func (e *SimpleExec) executeBegin(ctx context.Context, s *ast.BeginStmt) error`

    根据注释可以确定，这里就是执行 `BEGIN` 或者 `START TRANSACTION` 语句对应的函数

    ```go
    func (e *SimpleExec) executeBegin(ctx context.Context, s *ast.BeginStmt) error {
    	// If BEGIN is the first statement in TxnCtx, we can reuse the existing transaction, without the
    	// need to call NewTxn, which commits the existing transaction and begins a new one.
        
    	//..................................................................
        
    	// With START TRANSACTION, autocommit remains disabled until you end
    	// the transaction with COMMIT or ROLLBACK. The autocommit mode then
    	// reverts to its previous state.
        
    	//................................................................
    	txn, err := e.ctx.Txn(true)
        //................................................................
    }
    ```

  - 可以在这里打日志，但是考虑到是否有可能有隐式触发的事务，所以进一步跟到 `ctx.Txn()` 方法中，跳转到了 `session/session.go` 文件内：

    ```go
    func (s *session) Txn(active bool) (kv.Transaction, error) {
    	// ...............................................
    	if s.txn.pending() {
            //..............................................
    		// Transaction is lazy initialized.
    		// PrepareTxnCtx is called to get a tso future, makes s.txn a pending txn,
    		// If Txn() is called later, wait for the future to get a valid txn.
            // changePendingToValid 函数中向 PD 请求了一个全局时间戳，同时创建了 TiKV 实例
    		if err := s.txn.changePendingToValid(); err != nil {
    			// ----------------- 错误处理 ---------------------
    		}
            // 将获取到的全局时间戳在 sessionVars 中保存一份 
    		s.sessionVars.TxnCtx.StartTS = s.txn.StartTS()
            
            // *************** 在这里打印日志 *****************
            logutil.BgLogger().Info("Hello transaction !!!", zap.String("txn", s.txn.String()))
    		//......................................................
    }
    
    // txn.changePendingToValid 方法会跳转到这里    
    func (tf *txnFuture) wait() (kv.Transaction, error) {
        // 向 PD 申请一个全局时间戳
    	startTS, err := tf.future.Wait()
        // -------------- 错误处理 -----------------------
    	// 启动 TiKV 实例
    	return tf.store.Begin()
    }
    ```

    - 在 `changePendingToValid()` 方法中向 PD 请求了一个全局时间戳，同时创建了 TiKV 实例，这与之前的猜想完全对上了。选择将日志打在这里
    
  - 然后编译并使用 `tiup` 启动测试集群，借助 `dashboard`观察执行结果，发现什么都不干都会一直打印日志，猜测可能是系统在后台执行语句收集相关状态信息。

  - 为了验证结果，从 `main()` 函数开始跟调用链，尝试能否将执行的语句打印出来

     ```markdown
     main() -> svr.Run() -> s.onConn() -> conn.Run() -> cc.dispatch() -> cc.handleQuery() -> cc.handleStmt() -> cc.ctx.ExecuteStmt() -> tc.Session.ExecuteStmt()
     ```

     最后在 `session/session.go` 文件中：

     ```go
     func (s *session) ExecuteStmt(ctx context.Context, stmtNode ast.StmtNode) (sqlexec.RecordSet, error) {
     	// ......................................................
     	// 准备事务的执行上下文
     	s.PrepareTxnCtx(ctx)
     	// ......................................................
     
     	// Transform abstract syntax tree to a physical plan(stored in executor.ExecStmt).
         // 生成物理执行计划
     	compiler := executor.Compiler{Ctx: s}
     	stmt, err := compiler.Compile(ctx, stmtNode)
     	// ......................................................
     
     	// Execute the physical plan.
     	// 执行物理计划
     	logStmt(stmt, s.sessionVars)
     	recordSet, err := runStmt(ctx, s, stmt)
         
         // ************** 在这里打日志 ******************
         logutil.Logger(ctx).Info("hello, execute sql——1", zap.String("sql", stmtNode.Text()))
         if s.txn.String() != "invalid transaction" {
             logutil.Logger(ctx).Info("hello, execute sql——2", zap.String("txn", s.txn.String()))
         }
     	// ......................................................
     }
     ```

  - 重新编译验证：

     ```mysql
     mysql> use test;
     mysql> START TRANSACTION; # 因为有很多打印 Begin 的语句，为了区分所以选择 START TRANSACTION
     ```

     <img src="pic\w1-check1.png" alt="avatar" style="zoom:80%;" />

     ```mysql
     # 在不开启事务的情况下验证不会打印 Hello transaction
     mysql> create table t(id varchar(20));
     ```

     ![image-20200816155237211](E:\MyWork\MyLearningNote\database-learning\HighPerformanceTiDB\homework\pic\w1-check2.png)

     验证成功。


























