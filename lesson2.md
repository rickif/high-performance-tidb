# 题目
使用 sysbench、go-ycsb 和 go-tpc 分别对 TiDB 进行测试并且产出测试报告。

测试报告需要包括以下内容：

* 部署环境的机器配置(CPU、内存、磁盘规格型号)，拓扑结构(TiDB、TiKV 各部署于哪些节点)
* 调整过后的 TiDB 和 TiKV 配置
* 测试输出结果
* 关键指标的监控截图
	    * TiDB Query Summary 中的 qps 与 duration
	    * TiKV Details 面板中 Cluster 中各 server 的 CPU 以及 QPS 指标
	    * TiKV Details 面板中 grpc 的 qps 以及 duration

输出：写出你对该配置与拓扑环境和 workload 下 TiDB 集群负载的分析，提出你认为的TiDB 的性能的瓶颈所在(能提出大致在哪个模块即可)

# 环境准备
采用4台腾讯云4C8G云主机作为部署介质，部署1个pd节点、1个tidb节点以及3个tikv节点。其中，pd节点和tidb节点混部在1台机器上，tikv节点独占3台机器，prometheus等监控组件跟tidb、pd部署同一台云主机。

# sysbench 测试
![集群配置](https://github.com/rickif/high-performance-tidb/blob/master/asset/lesson2/conf.png)
![集群拓扑结构](https://github.com/rickif/high-performance-tidb/blob/master/asset/lesson2/topology.png)

## 测试用例
```
sysbench --config-file=sysbench.conf oltp_update_index --tables=8 --table-size=200000 run
```
## sysbench 执行结果
![sysbench执行结果](https://github.com/rickif/high-performance-tidb/blob/master/asset/lesson2/sysbench_output.png)  

查询性能: 20856 QPS/TPS
p95: 0.52ms
## 监控
![tidb query summary QPS&duration](https://github.com/rickif/high-performance-tidb/blob/master/asset/lesson2/sysbench_tidb_query_summary.png)
![tikv cluster details cpu](https://github.com/rickif/high-performance-tidb/blob/master/asset/lesson2/sysbench_tikv_details_cpu.png)
![tikv cluster details QPS](https://github.com/rickif/high-performance-tidb/blob/master/asset/lesson2/sysbench_tikv_details_qps.png)
![tikv details grpc QPS&duration](https://github.com/rickif/high-performance-tidb/blob/master/asset/lesson2/sysbench_tikv_details_grpc_qps.png)
![tikv details grpc QPS&duration](https://github.com/rickif/high-performance-tidb/blob/master/asset/lesson2/sysbench_tikv_details_grpc_qps2.png)
![tikv cluster details leader](https://github.com/rickif/high-performance-tidb/blob/master/asset/lesson2/sysbench_tikv_details_leader.png)

## 结果分析
从监控结果可以看到，tikv集群不同节点存在负载不均的情况。172.30.0.13节点cpu负载达到100%，是其余两个tikv节点的两倍。可以看到leader region在不同主机上的分布也不均匀，但节点的负载并不与leader的分布数量呈正比。
对于该测试场景，可能存在热点region，导致部分tikv节点负载较高，可采取如下措施：
1. 开启follower-read，通过从节点分摊读取负载。需要注意，tidb的follower-read是由一个session级别的系统变量控制
2. 调整region的大小，让数据进一步分散。同时，需要注意的是由于本测试用例只有8个table，每个table包括200000条记录。本身的region数量也较少，因此容易出现热点region。
