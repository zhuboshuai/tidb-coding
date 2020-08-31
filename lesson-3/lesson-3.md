# 压测环境  
近期正在对titan进行压测，于是选择进行压测的titan集群进行性能分析  
机器配置：  
服务端：3台物理机  
CPU: 2 x Intel Xeon Gold 5118 @ 2.30GHz, 24 Cores 48 Threads in total  
Memory: 196GB  
SSD: 8 x INTEL SSDSC2KB96 (960GB)  
客户端：3台物理机  
CPU: 2 x Intel Xeon Gold 6148 @ 2.40GHz, 40 Cores 80 Threads in total  
Memory: 512GB  
拓扑结构为：  
每台机器上分别部署一个pd-server和一个tikv-server，tikv-server使用的是titan存储引擎。  
无tidb-server部署，所以是一个KV数据库使用场景。  
workload情况： 
数据集为500M x 1KB x 3副本，已经提前写入；key为1到500M的数字；value全部为1K的随机字符串；分为50K读和50K写/100K读两个场景；  
已经通过重写入的方式将value导入到titan引擎  
48小时的QPS情况如下：  
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/lesson-3/qps.png)
# 性能分析  
由于是纯KV数据库场景，无法访问dashboard，通过perf命令分析性能瓶颈  
## 火焰图 
```  
git clone https://github.com/pingcap/tidb-inspect-tools
cd tracing_tools/perf
sh ./cpu_tikv.sh {PID of tikv-server}
```  
遇到报错：  
./cpu_tikv.sh: line 18: /opt/FlameGraph/stackcollapse-perf.pl: No such file or directory  
./cpu_tikv.sh: line 18: /opt/FlameGraph/flamegraph.pl: No such file or directory  
安装FlameGraph:  
```
wget https://github.com/brendangregg/FlameGraph/archive/master.zip
unzip master.zip
mv FlameGraph-master/ /opt/FlameGraph
```  
分析时间稍长。火焰图如下：  
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/lesson-3/flamegraph.png)  
选择一部分进行详细分析：  
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/lesson-3/collect-0.png)  
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/lesson-3/collect.png)  
可见Collect这个方法是个比较大的平层，调用rallocx和_rjem_je_arena_ralloc时间占比都不多。  
分析下代码：
```
impl BatchCollector<Vec<RaftMessage>, RaftMessage> for RaftMsgCollector {
    fn collect(&mut self, v: &mut Vec<RaftMessage>, e: RaftMessage) -> Option<RaftMessage> {
        let mut msg_size = e.start_key.len() + e.end_key.len();
        for entry in e.get_message().get_entries() {
            msg_size += entry.data.len();
        }
        if self.size > 0 && self.size + msg_size + GRPC_SEND_MSG_BUF >= self.limit as usize {
            self.size = 0;
            return Some(e);
        }
        self.size += msg_size;
        v.push(e);
        None
    }
}
```  
是否可以在创建RaftMessage时将每个RaftMessage的msg_size一并计算好，不再需要遍历了呢？issue如下：      
https://github.com/tikv/titan/issues/184  
## 有向无环图  
sh ./dot_tikv.sh {PID of tikv-server}  
生成图形如下：  
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/lesson-3/cpu-graph-0.png)   
## 内存火焰图  
```
perf probe -x /data/tikv-kvmode/bin/tikv-server} -a malloc  
#替换mem.shd第一行:  
perf record -e probe_tikv:malloc -F 99 -p $1 -g -- sleep 10
sh mem.sh {PID of tikv-server} 
```
火焰图如下： 
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/lesson-3/mem-graph.png)  
可见store-read-low-阶段中malloc占比是比较大的：  
![image](https://github.com/zhuboshuai/tidb-coding/blob/master/lesson-3/store-read-low.png)  
是否可以改成内存池方式由titan自己管理内存来加快内存分配速度呢？  
