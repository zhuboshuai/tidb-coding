# 1. 编译

选了一台内网通外网机器开始下载代码编译。我选了4.0.4代码来编译而不是master

## 1.1 首先是tidb

检出自己的分支:  
```
git checkout release-4.0.4  
git branch zhubs release-4.0.4   
git checkout zhubs
```  
go版本是go version go1.14.6 linux/amd64  
然后cd到根目录开始编译：  
```make```  
问题马上来了：  
Get "https://proxy.golang.org/github.com connection timed out  
很明显网络不好，改成国内镜像：  
```go env -w GOPROXY=https://goproxy.cn```  
查看下设置是否生效：
```go env | grep GOPROXY```  
继续编译，这次成功完成。可见binary文件：  
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/tidb%E7%BC%96%E8%AF%91.png)   

## 1.2 pd

仍然检出自己的分支，这次用的是release-4.0分支  
开始编译，很快报错：  
make: *** [install-go-tools] Error 56  
查看了下MakeFile，有几个go install命令可能因为网络问题执行超时，于是手动执行一遍  
```go install ***```  
然后继续，golangci/golangci-lint输出之后就不继续进行了  
看脚本怀疑是embedded-assets-golang.zip包下载卡住了，直接把scripts/embed-dashboard-ui.sh里的函数调用download_embed_asset注释掉了  
然后在办公机上直接下载包。下载后扔到.dashboard_asset_cache目录下  
结果报错：  
	pkg/dashboard/uiserver/embedded_assets_rewriter.go:31:26: undefined: assets  
	pkg/dashboard/uiserver/embedded_assets_rewriter.go:32:13: undefined: vfsgen۰FS  
	pkg/dashboard/uiserver/embedded_assets_rewriter.go:34:15: undefined: vfsgen۰CompressedFileInfo  
	pkg/dashboard/uiserver/embedded_assets_rewriter.go:42:9: undefined: assets  
看来修改方式太粗暴了，于是重新看了一遍脚本，只注释了部分download_embed_asset的代码，同时把embedded-assets-golang.zip包名修改为embedded-assets-golang-2020.08.07.1.zip  
继续编译，通过  
在bin目录下可见binary:  
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/pd%E7%BC%96%E8%AF%91.png)   

## 1.3 编译tikv

checkout的是release-4.0分支  
立即开始make,然后发现rust也没装。开始安装rust环境  
```curl https://sh.rustup.rs -sSf | sh
select 1
source $HOME/.cargo/env
```  
版本为rustc 1.45.2 (d3fb005a3 2020-07-31)  
继续编译， 报错：  
  error: failed to extract package (perhaps you ran out of disk space?)  
  error: caused by: No space left on device (os error 28)  
  make: *** [release] Error 1  
竟然没有空间了，看来需要下载的库比较多。检查了下发下tiup占用空间较多，于是把tiup牺牲了  
继续编译，继续报错：  
warning: spurious network error (2 tries remaining)  
网络问题忍无可忍啊，果断找镜像：  
$HOME/.cargo/config配置ustc的镜像，然后下载速度飞快  
继续编译和报错：  
  CMake 3.1 or higher is required  
解决：  
```yum install cmake3; mv /usr/bin/cmake /usr/bin/cmake.bak;ln -s /usr/bin/cmake3 /usr/bin/cmake```  
继续编译和报错：  
  CMake Error at CMakeLists.txt:2 (project):  
    The CMAKE_CXX_COMPILER:  
      c++  
    is not a full path and was not found in the PATH.  
解决：  
```yum list gcc-c++;yum install gcc-c++;  ```  
终于在target/release看到了binary文件:  
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/tikv%E7%BC%96%E8%AF%91.png)   

# 2. 启动
创建启动根目录，创建pd所需各目录

## 2.1 启动PD  
隐去IP:   
```
nohup ./pd-server --name="pd" \
          --data-dir="/data/kaifa/deploydir/pd/data" \
          --client-urls="http://192.168.0.1:2379" \
          --peer-urls="http://192.168.0.1:2380" \
          --log-file="/data/kaifa/deploydir/pd/log/pd.log" &
```  
查看pd状态:   
```curl http://192.168.0.1:2379/pd/api/v1/members```   
可见`members` `leader` `etcd_leader`都正常:   
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/pd-member-desen.png)   
查看pd日志:     
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/pd%E5%90%AF%E5%8A%A8%E6%97%A5%E5%BF%97-desen.png)  

## 2.2 启动TiKV 
隐去IP:   
```
nohup ./tikv-server --addr 0.0.0.0:20171 --advertise-addr 192.168.0.1:20171 --status-addr 192.168.0.1:20181 --pd 192.168.0.1:2379 --data-dir /data/kaifa/deploydir/tikv/data --log-file /data/kaifa/deploydir/tikv/log/tikv.log &
```
查看pd状态:   
```url http://192.168.0.1:2379/pd/api/v1/stores```  
结果:  
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/pd-stores-desen.png)  
查看启动日志:      
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/tikv%E5%90%AF%E5%8A%A8%E6%97%A5%E5%BF%97-desen.png)  

## 2.3 启动TiDB
隐去IP:  
```nohup ./tidb-server -P 4000 --status=10080 --advertise-address=192.168.0.1 --path=192.168.0.1:2379  --log-slow-query=/data/kaifa/deploydir/tidb/log/tidb_slow_query.log --log-file=/data/kaifa/deploydir/tidb/log/tidb.log &```
查看启动日志: 
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/tidb%E5%90%AF%E5%8A%A8%E6%97%A5%E5%BF%97-desen.png)    
尝试访问集群:   
```mysql -uroot -h192.168.0.1 -P4000```
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/tidb%E8%AE%BF%E9%97%AE-desen.png)    

# 3. 实现打印日志功能

一开始把题目看错了，看成了输出"hello transaction"给客户端，心想需要改返回值，于是想到COM_QUERY Response：     
https://dev.mysql.com/doc/internals/en/com-query-response.html  
是不是要改OK_Packet呢？  
https://dev.mysql.com/doc/internals/en/packet-OK_Packet.html  
但是不对，这样就改了MySQL的客户端-服务端协议；需要客户端也做适配    
再看题目是输出到日志  
于是捋了一遍SQL Parser相关过程：  
https://pingcap.com/blog-cn/tidb-source-code-reading-2  
https://pingcap.com/blog-cn/tidb-source-code-reading-3  
于是想到在执行语句的时候打印日志：  
session.go中:  
```go
session.executeStatement
```  
发现有个方法调用：`logStmt(stmt, s.sessionVars)`  
于是扩展这个方法：  
- 如果是ast.BeginStmt，则打印日志  
- 如果是Query语句，同时是自动提交状态，也打印日志

但是有个问题，显式事务里面的Query语句怎么处理？显示事务里应该只有在启动事务时(begin)打印日志，事务中的语句不应该再输出了  
起初想到```context.Context```里去看看，发现没有想要的内容   
于是发现```vars```里有相关变量  
整个判断条件变为:  
```go
func logStmt(execStmt *executor.ExecStmt, vars *variable.SessionVars) {
	user := vars.User
	schemaVersion := vars.TxnCtx.SchemaVersion
	switch stmt := execStmt.StmtNode.(type) {
	case *ast.CreateUserStmt, *ast.DropUserStmt, *ast.AlterUserStmt, *ast.SetPwdStmt, *ast.GrantStmt,
		*ast.RevokeStmt, *ast.AlterTableStmt, *ast.CreateDatabaseStmt, *ast.CreateIndexStmt, *ast.CreateTableStmt,
		*ast.DropDatabaseStmt, *ast.DropIndexStmt, *ast.DropTableStmt, *ast.RenameTableStmt, *ast.TruncateTableStmt:
		if ss, ok := execStmt.StmtNode.(ast.SensitiveStmtNode); ok {
			logutil.BgLogger().Info("CRUCIAL OPERATION",
				zap.Uint64("conn", vars.ConnectionID),
				zap.Int64("schemaVersion", schemaVersion),
				zap.String("secure text", ss.SecureText()),
				zap.Stringer("user", user))
		} else {
			logutil.BgLogger().Info("CRUCIAL OPERATION",
				zap.Uint64("conn", vars.ConnectionID),
				zap.Int64("schemaVersion", schemaVersion),
				zap.String("cur_db", vars.CurrentDB),
				zap.String("sql", stmt.Text()),
				zap.Stringer("user", user))
		}
	case *ast.BeginStmt:
		logutil.BgLogger().Info("hello transaction",
			zap.Uint64("conn", vars.ConnectionID),
			zap.Int64("schemaVersion", schemaVersion),
			zap.String("cur_db", vars.CurrentDB),
			zap.String("sql", stmt.Text()),
			zap.Stringer("user", user))
	default:
		if (vars.IsAutocommit() && !vars.InTxn()){
			logutil.BgLogger().Info("hello transaction",
				zap.Uint64("conn", vars.ConnectionID),
				zap.Int64("schemaVersion", schemaVersion),
				zap.String("cur_db", vars.CurrentDB),
				zap.String("sql", stmt.Text()),
				zap.Stringer("user", user))
		}
		logQuery(execStmt.GetTextToLog(), vars)
	}
}
```
重新编译后验证日志输出:  
set @@autocommit = 0;
begin;
insert into test.t values(1);
commit;
发现begin被正常记录，但是没有记录insert：
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/%E9%AA%8C%E8%AF%81%E6%97%A5%E5%BF%971.png) 

再验证场景2：  
set @@autocommit = 1;
insert into test.t values(2);
发现insert被正常记录：
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/%E9%AA%8C%E8%AF%81%E6%97%A5%E5%BF%972.png) 

同时发现输出hello transaction的内部SQL很多，其实也可以去掉，根据user的类别可以实现只输出用户的hello transaction日志，不再赘述。




