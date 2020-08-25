# 部署
上节课是单机部署TiDB,为了压测更加真实，本节课准备多机部署。但是机器资源有限，决定虚机部署。  
机器配置如下：  

- CPU:Intel(R) Xeon(R) CPU E5-2650 v4 @ 2.20GHz 共4核  
- 7.6G内存  
- 60G HDD磁盘  
- 操作系统为centos7-update4  

仍然用v4.0.2版本。tiup很快，但是稳定性还需要加强一下(此处省略几百字)，我暂时仍用ansible部署。  
在对机器环境做了一系列的配置之后，部署成功了。包括以下修改：
- 去掉benchmark步骤(虚机是跑步通过ansible脚本自带的fio benchmark的)
- 重新配置ntpd
- swap off
- nodelalloc
- 去掉部署spark(没有在ini中配置部署spark,也要下载spark-2.4.3-bin-hadoop2.7.tgz，这点比较坑；不过我没有使用tiup，所以自己扛着吧)
- 限制ansible并发数(虚机环境下并发数过高经常跑脚本中途失败)等

拓扑结构上是这样的：  
3台虚机，每台虚机上分别有一个TiDB,一个pd，一个TiKV。  
贴个部署成功的图吧：  
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/lesson-2/%E9%83%A8%E7%BD%B2%E6%88%90%E5%8A%9F.png)    
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
## 观察性能情况   
QPS:  
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/lesson-2/QPS.png)  
最大没有超过8K  
duration:  
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/lesson-2/duration.png)  
p95超过了1分钟；p80都到了48秒,已经很高了，分析下原因：    
先看kv errors:  
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/lesson-2/KV%20errors.png)   
主要错误是not_leader; backoff OPS最大的region miss,高峰达到了接近3K。因为选择的场景是readproportion=0.5，updateproportion=0.5，所以有可能大量的更新导致了region leader transfer,或者region split,从而导致region Miss。   
继续去看看TiKV的情况：  
grpc:  
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/lesson-2/grpc.png)  
coprocessor最高达到1s才能执行完，kv_pessimistic_lock最大执行时间也超过1s（同时可知我开启了悲观锁）  
那么是不是thread CPU有瓶颈呢？  
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/lesson-2/thread%20cpu%201.png)    
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/lesson-2/thread%20cpu%202.png)    
看上去各thread占用的CPU并不高，最高60%左右，没有占满一个核  
那么rocksdb的存取速度如何呢？  
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/lesson-2/rocksdb%20-%20kv.png)     
get操作最大1.4s,seek操作最大700多ms，也比较高了。  
去看看机器层面的监控吧：  
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/lesson-2/cpu%20idle.png)  
压测期间CPU idle都已经成为零了，CPU已经很繁忙了。  
再看看内存：  
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/lesson-2/memory.png)  
内存使用量的高峰期几乎全部用完了所有可用内存。  
IO情况：  
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/lesson-2/io%20util.png)
最大的时候IO Util已经到了100%。  
可见无论从CPU/内存，还是磁盘IO都已经打满了。所以，想生产环境用TiDB的话，机器太差是不行的。  
解决了机器性能问题，才是考虑其它瓶颈的时候。比如：  
- 各tikv线程是否打满  
- 缓存命中率如何  
- 锁冲突是否过高  
- PD集群grpc时间是否过长  
- scheduler模块对事务排序时是否冲突太多  
- rocksdb是否触发了流控  
等等。3台虚机也没有什么可以调整部署的空间了，机器多的话可以考虑把pd和tikv的存储分开以进行IO隔离，把tidb和tikv的部署分开以避免内存不够用发生OOM。  
下面部署下另外两个压测工具  
# go-tpc  
## go-tpc编译
```
git clone https://github.com/pingcap/go-tpc.git $GOPATH/src/github.com/pingcap/go-tpc
make build
```   
编译过程和结果：  
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/lesson-2/git%20clone.png)  
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/lesson-2/make%20build.png)  
## go-tpc压测  
准备数据：  
```./bin/go-tpc tpcc -H 192.168.0.1 -P 4000 -D tpcc --warehouses 10 prepare```
其中IP已经脱敏。因为虚机磁盘有限，所以只设置了10个warehouse。      
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/lesson-2/tpcc-tables.png)  
进行压测:  
```./bin/go-tpc tpcc -H 192.168.0.1 -P 4000 -D tpcc --warehouses 10 run```  
压测结果：  
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/lesson-2/tpmC.png)  
可见得到的tpmC数值是557.1，并不高 
压测期间的duration/QPS监控：  
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/lesson-2/TPCC-%E7%9B%91%E6%8E%A7.png)  
具体压测数据低的原因ycsb压测已经详述此处不再赘述。  
# Sysbench
## Sysbench编译  

## Sysbench压测  



