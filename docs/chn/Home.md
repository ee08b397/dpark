DPark是一个基于[Mesos](http://www.mesosproject.org/)的集群计算框架(cluster computing framework)，是[Spark](http://www.spark-project.org/)的Python实现版本，类似于MapReduce，但是比其更灵活，可以用Python非常方便地进行分布式计算，并且提供了更多的功能以便更好的进行迭代式计算。

DPark的计算模型是基于两个中心思想的：对分布式数据集的并行计算以及一些有限的可以在计算过程中、从不同机器访问的共享变量类型。这个的目标是为了提供一种类似于global address space programming model的工具，例如OpenMP，但是我们要求共享变量的类型必须是那些很容易在分布式系统当中实现的，当前支持的共享变量类型有只读的数据和支持一种数据修改方式的累加器(accumulators)。DPark具有的一个很重要的特性：分布式的数据集可以在多个不同的并行循环当中被重复利用。这个特性将其与其他数据流形式的框架例如Hadoop和Dryad区分开来。

## User Guide
##### [下载源代码和安装指导](https://github.com/jackfengji/test_pro/wiki/%E4%B8%8B%E8%BD%BD%E5%92%8C%E5%AE%89%E8%A3%85)   

1. 如何下载源代码 
2. 如何安装在mesos上并进行必要的配置

##### [使用DPark](https://github.com/jackfengji/test_pro/wiki/%E4%BD%BF%E7%94%A8DPark)   

1. 初识DPark    
1. 如何在本机、多线程、mesos上运行DPark程序   
1. 弹性分布式数据集(RDD)   
1. 共享变量    
1. [Examples](https://github.com/jackfengji/test_pro/wiki/Examples)   

## Developer Guide
##### 1. [RDD的原理](https://github.com/jackfengji/test_pro/wiki/RDD%E7%9A%84%E5%8E%9F%E7%90%86)  
##### 2. [DPark的任务调度机制](https://github.com/jackfengji/test_pro/wiki/%E4%BB%BB%E5%8A%A1%E8%B0%83%E5%BA%A6%E6%9C%BA%E5%88%B6)
##### 3. [共享变量的实现](https://github.com/jackfengji/test_pro/wiki/%E5%85%B1%E4%BA%AB%E5%8F%98%E9%87%8F%E7%9A%84%E5%AE%9E%E7%8E%B0)
##### 4. [DPark和Spark的区别](https://github.com/jackfengji/test_pro/wiki/DPark%E5%92%8CSpark%E7%9A%84%E5%8C%BA%E5%88%AB)