DPark is a cluster computing framework based on [Mesos] (http://www.mesosproject.org/). DPark is a python clone of [Spark](http://www.spark-project.org/). It is a MapReduce alike computing framework, but is more Flexible. It is convenient to use Python to do distributed computing. DPark provides more functions for iterative computation.

The computation model of DPark has two main ideas: a. parallel computing for distributed data set; b. limited numbers of shared variables that could be visited from different machines. The goal is to provide a tool similar to global address space programming model, for example OpenMP. Types of shared variables must be easy to implement in distributed system. At this time, we only support two types: a. read only data and b. accumulators that supports one modification method. DPark has an important character: a distributed data set is able to be recycling used in multiple parallel. This character makes DPark different from other streaming framework like Hadoop and Dryad.

## User Guide
##### [Download and Installation](https://github.com/charnugagoo/dpark/wiki/Download-and-Installation)   

1. How to download source code
2. How to install on top of moses and necessary configuration

##### [How to use DPark](https://github.com/charnugagoo/dpark/wiki/How-to-use-DPark)   

1. DPark, the first view
2. How to run DPark on top of Mesos on local machine with multiprocessing
3. Resilient Distributed Dataset (RDD)
4. Variables sharing
5. [Examples](https://github.com/charnugagoo/dpark/wiki/Examples)   

## Developer Guide
##### 1. [Inside RDD](https://github.com/charnugagoo/dpark/wiki/Inside-RDD)  
##### 2. [Task scheduling mechanism of DPark](https://github.com/charnugagoo/dpark/wiki/Task-scheduling-mechanism-of-DPark)
##### 3. [Implementation of variable sharing](https://github.com/charnugagoo/dpark/wiki/Implementation-of-variable-sharing)
##### 4. [Differences between DPark and Spark](https://github.com/charnugagoo/dpark/wiki/Differences-between-DPark-and-Spark)

