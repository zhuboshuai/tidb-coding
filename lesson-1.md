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
终于在target/release看到了binary文件  

# 2. 实现打印日志功能

一开始把题目看错了，看成了输出"hello transaction"给客户端，心想需要改返回值，于是想到COM_QUERY Response：   
https://dev.mysql.com/doc/internals/en/com-query-response.html  
是不是要改OK_Packet呢？  
https://dev.mysql.com/doc/internals/en/packet-OK_Packet.html  
但是不对，这样就改了MySQL的客户端-服务端协议；需要客户端也做适配。  
再看题目是输出到日志。  
于是捋了一遍SQL Parser相关过程：  
https://pingcap.com/blog-cn/tidb-source-code-reading-3  
于是想到在执行语句的时候打印日志：  
session.executeStatement  
发现有个方法调用：logStmt(stmt, s.sessionVars)  
于是扩展这个方法：  
如果是ast.BeginStmt，则打印日志  
如果是Query语句，同时是自动提交，也打印日志。  
但是有个问题，显式事务里面的Query语句怎么处理？  



