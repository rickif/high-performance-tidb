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
![机器配置](https://github.com/rickif/high-performance-tidb/blob/master/asset/lesson2/machines.png)
采用4台腾讯云4C8G云主机作为部署介质，部署1个pd节点、1个tidb节点以及3个tikv节点。其中，pd节点和tidb节点混部在1台机器上，tikv节点独占3台机器，prometheus等监控组件跟tidb、pd部署同一台云主机    

注：本作业过程中购买了多次云主机，所以不同的测试用例使用的云主机可能不同

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
2. 调整region的大小，让数据进一步分散。同时，需要注意的是由于本测试用例只有8个table，每个table包括200000条记录。本身的region数量也较少，因此容易出现热点region

# go-ycsb 测试

![集群拓扑结构](https://github.com/rickif/high-performance-tidb/blob/master/asset/lesson2/ycsb-topology.png)
## 测试用例

1. 加载测试数据
```
./bin/go-ycsb load mysql -P workloads/workloadb -p recordcount=200000 -p mysql.host=127.0.0.1 -p mysql.port=4000 -p mysql.db=db_test --threads 16
```
2. 执行测试
```
./bin/go-ycsb run mysql -P workloads/workloadb -p operationcount=10000 -p mysql.host=127.0.0.1 -p mysql.port=4000 -p mysql.db=db_test --threads 16
```

测试用例输入20万条数据，采用workloadb数据集，ycsb的各个数据集的特征如下：
![ycsb-workload](https://github.com/rickif/high-performance-tidb/blob/master/asset/lesson2/ycsb-workload.png)  
可以看到，workloadb是读多写少的场景，读写比例为95:5

## 监控

![tidb query summary duration](https://github.com/rickif/high-performance-tidb/blob/master/asset/lesson2/ycsb-tidb-query-summary.png)
![tidb query summary QPS](https://github.com/rickif/high-performance-tidb/blob/master/asset/lesson2/ycsb-tidb-query-summary-qps.png)
![tikv cluster details cpu](https://github.com/rickif/high-performance-tidb/blob/master/asset/lesson2/ycsb-tikv-details-cpu.png)
![tikv cluster details QPS](https://github.com/rickif/high-performance-tidb/blob/master/asset/lesson2/ycsb-tikv-details-qps.png)
![tikv cluster details region&leader](https://github.com/rickif/high-performance-tidb/blob/master/asset/lesson2/ycsb-tikv-details-region-leader.png)
![tikv details grpc QPS&duration](https://github.com/rickif/high-performance-tidb/blob/master/asset/lesson2/ycsb-tikv-grpc-duration-qps.png)

## 结果分析
从输出结果来看，依然可以看到有较大的负载不均衡问题出现。由于负载测试时间较短，pd的调度还没有生效。因此，解决此类问题，还可调整pd的调度策略，使其对于突发负载更为敏感，实现较快的负载调度

# TPC-C 测试

![集群拓扑结构](https://github.com/rickif/high-performance-tidb/blob/master/asset/lesson2/tpcc-topology.png)

## 测试用例
1. 准备数据
```
./bin/go-tpc tpcc --warehouses 50 prepare -H 127.0.0.1 -P 4000
```
2. 执行测试
```
./bin/go-tpc tpcc --warehouses 50 run -H 127.0.0.1 -P 4000
```

## 监控
![tidb query summary QPS&duration](https://github.com/rickif/high-performance-tidb/blob/master/asset/lesson2/tpcc-tidb-query-summary.png)
![tikv cluster details cpu&QPS](https://github.com/rickif/high-performance-tidb/blob/master/asset/lesson2/tpcc-tikv-details-cpu-qps.png)
![tikv cluster details region&leader](https://github.com/rickif/high-performance-tidb/blob/master/asset/lesson2/tpcc-tikv-details-region-leader.png)
![tikv details grpc QPS&duration](https://github.com/rickif/high-performance-tidb/blob/master/asset/lesson2/tpcc-tikv-grpc-duration-qps.png)