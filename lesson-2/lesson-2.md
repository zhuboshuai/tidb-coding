# 部署
上节课是单机部署TiDB,为了压测更加真实，本节课准备多机部署。但是机器资源有限，决定虚机部署。  
机器配置如下：  
CPU:Intel(R) Xeon(R) CPU E5-2650 v4 @ 2.20GHz 共4核  
7.6G内存  
60G HDD磁盘  
操作系统为centos7-update4  
仍然用v4.0.2版本。tiup很快，但是稳定性还需要加强一下(此处省略几百字)，我暂时仍用ansible部署。  
在对机器环境做了一系列的配置之后，部署成功了。包括去掉benchmark步骤(虚机是跑步通过ansible脚本自带的fio benchmark的)，重新配置ntpd,swap off,nodelalloc,去掉部署spark(没有在ini中配置部署spark,也要下载spark-2.4.3-bin-hadoop2.7.tgz，这点比较坑；不过我没有使用tiup，所以自己扛着吧)，限制ansible并发数(虚机环境下并发数过高经常跑脚本中途失败)等。贴个部署成功的图吧：  

# goycsb  
## goycsb编译  
按照官网说明部署:https://github.com/pingcap/go-ycsb  
```
git clone https://github.com/pingcap/go-ycsb.git $GOPATH/src/github.com/pingcap/go-ycsb
cd $GOPATH/src/github.com/pingcap/go-ycsb
make
```  
这块没有遇到什么问题。  
## goycsb压测  
首先写入数据：  
``` 
./bin/go-ycsb load mysql -p mysql.host=192.168.0.1 -p mysql.port=4000 -p mysql.user=root -p recordcount=10000000 -P workloads/workloada --threads 16
``` 
recordcount太小对压测没有意义，所以设置成了一千万条  
然后压测：  
把workloads/workloada里的operationcount行改为operationcount=10000000  
``` 
./bin/go-ycsb run mysql  -p mysql.host=192.168.0.1 -p mysql.port=4000 -p mysql.user=root -P workloads/workloada --threads 256
``` 
观察性能情况：   
QPS:  
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/lesson-2/QPS.png)  
最大没有超过8K  
duration:  
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/lesson-2/duration.png)  
p95超过了1分钟；p80都到了48秒,分析下原因：    
先看kv errors:  
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/lesson-2/KV%20errors.png) 
主要错误是not_leader; backoff OPS最大的region miss,高峰达到了接近3K  
去看看TiKV的情况：  
grpc:  
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/lesson-2/grpc.png)  
coprocessor最高达到1s才能执行完，kv_pessimistic_lock最大执行时间也超过1s（同时可知我开启了悲观锁）  
是不是thread CPU有瓶颈呢？  
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/lesson-2/thread%20cpu%201.png)  
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/lesson-2/thread%20cpu%202.png)  
看上去各thread占用的CPU并不高，最高60%左右，没有占满一个核  
那么rocksdb的存取速度如何呢？  
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/lesson-2/rocksdb%20-%20kv.png)   
get操作最大1.4s,seek操作最大700多ms  





