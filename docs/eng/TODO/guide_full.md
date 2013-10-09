Dpark Guid
====================

Basic Concepts
--------------------

RDD
~~~~~~~~~~~~~~~~~~~~

RDD(Resilient Distributed Datasets) as the key concept of DPark, is data sets that siupports high fault-tolerance parallel computing.

- Two methods to create RDD: Created by specific function from storage hardwares (memory or hard disk); Created by other RDD
- RDD has information how itself depends on other RDD. If some failure happens, itself could be re-created.
- RDD is read-only, so that it is easy to recover a RDD form a topology.
- RDD is able to be used multiple times.
- One RDD is made up by several Split. Splits are basic units for parallel computing.
- RDD supports two operations, "Narrow" deps map and "Wide" deps reduce

Narrow Dependency
~~~~~~~~~~~~~~~~~~~~

- Narrow dependency operation compute every line of data, similar to map function in Python
- Narrow dependency operation is streaming computation, so its memory usage is small
- Frequentrly used functions: map / flatMap / filter / mapValue

Wide Dependency
~~~~~~~~~~~~~~~~~~~~

- Wide Dependency operation should do computation after needed data is generated completely. So memory usage is huge
- Frequently used functions reduce / reduceByKey / uniq / groupBy / groupByKey / combineByKey
- reduceByKey splits keys of data into M classes. When computing on each class, all keys are stored in memory, so memory usage is huge
- reduce could be treated as reduceByKey with only one key and only one class. It merges results in present thread.

Stage / Job / Task
~~~~~~~~~~~~~~~~~~~~

- 由多个 RDD 组成的链式计算过程会根据依赖关系被划分为多个Stage
- 每次 Shuffle 过程（由 combineByKey 产生）都会把计算过程拆分成前后两个 Stage
- 每个 Stage 会有多个 Job，对应一个或者多个类似的 RDD
- 一个 Job 有多个 Task，数量由 RDD 的 Split 决定
- 同一个Stage里面的 Job 和 Task 是可以并行执行的
- 每个 Task 对应一个进程，如果服务器资源充足，就可以完全并行，否则就只能部分并行

共享变量
~~~~~~~~~~~~~~~~~~~~

- Dpark 在运行时，会将 RDD 以及针对它的函数序列化后发送到执行节点去，反序列化并执行。函数所依赖的全局变量，模块和闭包对象等，也都会随着函数一块发送过去。
- 每个 Task 都会发送一次，所以当依赖很大或者依赖中等但是 Task 很多，就会影响性能。这时候需要用到广播

广播
~~~~~~~~~~~~~~~~~~~~

- 广播适用于比较大的数据集，但它不能超过单机的内存限制
- 广播之后，使用时会在集群中各台机器之间交换数据，不会像序列化一样依赖于执行脚本的机器
- 序列化后超过 100k 的对象都需要广播，我们称手工写 dpark.broadcast(xxx) 为显式广播
- Dpark 可以自动的发现大对象并广播出去，代码中不需要手工写广播代码，我们称这种为隐式广播
- Dpark 可能不能正确发现大对象，也有可能一个大对象同时被多个函数使用的情况，所以需要有选择的使用显式广播

命令行参数
--------------------
\-M xxxx
    每个 task 申请的内存

--err 0.001
    如果有脏数据，用这个可以忽略掉

常用函数
--------------------

union
~~~~~~~~~~~~~~~~~~~~

::

    rdd3 = rdd1.union(rdd2)
    rdd3 = dpark.union([rdd1, rdd2])

map / flatMap
~~~~~~~~~~~~~~~~~~~~

::

    b = dpark.parallelize([1, 2])
    r1 = b.map(lambda x: (x, x)).collect()  # [(1, 1), (2, 2)]
    r2 = b.flatMap(lambda x: (x, x)).collect()  # [1, 1, 2, 2]

mapValue / flatMapValue
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

    b = dpark.parallelize([(1, 11), (2, 22)])
    r1 = b.mapValue(lambda x: (x, x)).collect()  # [(1, (11, 11)), (2, (22, 22))]
    r2 = b.flatMapValue(lambda x: (x, x)).collect()  # [(1, 11), (1, 11), (2, 22), (2, 22)]

filter
~~~~~~~~~~~~~~~~~~~~

::

    b = dpark.parallelize([[1, 11], 0, [2, 22]])
    r = b.filter(lambda x: x).collect()  # [[1, 11], [2, 22]]

uniq
~~~~~~~~~~~~~~~~~~~~

::

    b = dpark.parallelize([(1, 11), 0, (1, 11)])
    r = b.uniq().collect()  # [0, (1, 11)]

groupBy / groupByKey
~~~~~~~~~~~~~~~~~~~~

::

    b = dpark.parallelize([(1, 11), (1, 12), (2, 22)])
    r1 = b.map(lambda x: (x[0], x)).groupByKey().collect()  # [(1, [(1, 12), (1, 11)]), (2, [(2, 22)])]
    r2 = b.groupBy(lambda x: x[0]).collect()  # [(1, [(1, 12), (1, 11)]), (2, [(2, 22)])]

reduce / reduceByKey
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

    b = dpark.parallelize([(1, 11), (1, 12), (2, 22)])
    r1 = b.reduceByKey(lambda x, y: x + y).collect()  # [(1, 23), (2, 22)]
    r2 = b.reduce(lambda x, y: (x[0] + y[0], x[1] + y[1]))  # (4, 45)

join / leftOuterJoin / rightOuterJoin / outerJoin / groupWith
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

    rdd1 = dpark.parallelize([(1, 11), (2, 12), (3, 22)])
    rdd2 = dpark.parallelize([(1, 33), (2, 44), (4, 55)])
    r1 = rdd1.join(rdd2).collect()  # [(1, (11, 33)), (2, (12, 44))]
    r2 = rdd1.leftOuterJoin(rdd2).collect()  # [(1, (11, 33)), (2, (12, 44)), (3, (22, None))]
    r3 = rdd1.rightOuterJoin(rdd2).collect()  # [(1, (11, 33)), (2, (12, 44)), (4, (None, 55))]
    r4 = rdd1.outerJoin(rdd2).collect()  # [(1, (11, 33)), (2, (12, 44)), (3, (22, None)), (4, (None, 55))]

    rdd3 = dpark.parallelize([(1, 100), (2, 101), (4, 201)])
    r5 = rdd1.groupWith(rdd2).collect()  # [(1, ([11], [33])), (2, ([12], [44])), (3, ([22], [])), (4, ([], [55]))]
    r6 = rdd1.groupWith([rdd2, rdd3]).collect()  # [(1, ([11], [33], [100])), (2, ([12], [44], [101])), (3, ([22], [], [])), (4, ([], [55], [201]))]

读相关
~~~~~~~~~~~~~~~~~~~~

::

    textFile(self, path, ext='', followLink=True, maxdepth=0, cls=TextFileRDD, *ka, **kws)

    # 读单个文件，每个 Split 最大 16 M
    rdd = dpark.textFile('xxxx.csv', splitSize=16 << 20)

    # 读多个压缩文件（目前textFile支持 .bz2 和 .gz），每个文件分成 10 个 Split
    rdd = dpark.textFile(['xxxx.bz2', 'xxxxx.gz'], numSplits=10)

    # 递归读目录，扩展名为.csv，PS：隐藏文件会被忽略
    rdd = dpark.textFile('/xxxx/xxxx', ext='.csv')

    # 其他文件类型请参见 rdd.py，或使用 pydoc dpark.rdd

写相关
~~~~~~~~~~~~~~~~~~~~

::

    # 写文件，扩展名.csv，gz 格式压缩
    rdd.saveAsTextFile(path, ext='.csv', compress=True)

    # 按 key 写入多个目录，扩展名.csv，path 下如已有文件则删除
    rdd = dpark.parallelize([('1', '1'), ('2', '2')])
    rdd.saveAsTextFileByKey(path, ext='.csv', overwrite=True) # path 下会生成 1 和 2 两个目录

    # 其他文件类型请参见 rdd.py，或使用 pydoc dpark.rdd


代码风格
--------------------

我们先来看个例子

::

    data.map(
        lambda line: line.strip().split(' ')
    ).filter(
        lambda line: len(line)>=3
    ).map(
        lambda line: (line[1],line[2])
    ).map(
        lambda line: (line[0].split(':'),line[1])
    ).filter(
        lambda line: len(line[0])>=2
    ).map(
        lambda line: (line[0][1],line[1]))

这种代码写起来方便，但是欠缺可读性。换个写法

::

    def split_row(r):
        return r.strip().split(' ')
    
    def cal(r):
        if len(r) < 3:
            return
    
        _, bus, date = r[:3]
        t = bus.split(':')
        if len(t) < 2:
            return
    
        return t[1], date
    
    data.map(split_row).map(cal).filter(lambda x: x)

上面的代码就会好很多

开发注意事项
--------------------

- 先用小数据将代码调通，再执行大数据
- 执行未调优的脚本要关注 log 中的警告和错误，随时准备停掉脚本
- 务必以低并行度访问数据库，否则员外会找你喝茶
- 执行 collect / collectAsMap 会将数据读入当前内存，建议先 saveAsTextFile 看看大小，确保不会过大
- 了解自己的数据，才能有针对性的做优化

性能调优
--------------------

优化非 Dpark 部分
~~~~~~~~~~~~~~~~~~~~

- 先优化 map 依赖的函数，避免效率过低的操作，比如反复对大 list 执行 in 操作，反复的 re.compile 同一个表达式
- 组织数据时，适当压缩大小，比如纯数字的字符串先转 int

使用广播的时机
~~~~~~~~~~~~~~~~~~~~

一个简单的例子

::

    dpark = DparkContext()
    bid_data = dpark.parallelize(map(lambda x: (str(x), str(x)), range(10)))
    rdd = dpark.parallelize(map(lambda x: (str(x * 2), str(x)), range(100)))
    
    bids = bid_data.map(lambda r: r[1]).collect()
    r = rdd.filter(lambda r: r[1] in bids).collect()

bids 中的元素都是 string，如果条件允许而 bids 确实非常大，可以转成 int

::    

    bids = dpark.parallelize(data).map(lambda r: int(r[1])).collect()
    
bids 是一个 list，反复对 list 执行 in 操作，效率很低，转成 set 或者 dict

::

    bids = set(bids)
    bids = dict(((u, 1) for u in bids))
    bids = bid_data.map(lambda r: int(r[1])).map(lambda x: (x, 1)).collectAsMap()
    
如果 bids 很大，就需要使用广播（Dpark 可能会在这里使用隐式广播）

::

    bids_b = dpark.broadcast(bids)
    r = rdd.filter(lambda r: int(r[1]) in bids_b.value).collect()
    
如果 bids 特别大，到了会影响网络 IO 的程度……

::

    bids = bid_data.map(lambda r: r[1]).map(lambda x: (x, 1))
    r = rdd.map(lambda r: (r[1], r)).join(bids).filter(lambda r: r[1][1]).map(lambda r: r[1][0]).collect()

视情况使用 leftOuterJoin 等，实战代码 /mfs/datasupport/xiliang_moria/agg_index_product_total_uv.py

尽快减小数据集
~~~~~~~~~~~~~~~~~~~~

- 比如有两个独立操作 map 和 filter，先 filter 后 map 就可以减少一些不必要的计算
- 同理，uniq 和 map 也可以如此处理

使用 groupBy / groupByKey 的注意事项
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- 通常 key 小 value 大，所以不会大幅减少数据
- 在 key 不均衡的情况下，会导致某个 task 过大而出错，极端情况脚本挂掉
- 如果可能，优先使用 reduce 方式

::

    dpark = DparkContext()
    big_data = dpark.parallelize(range(10) + range(20) + range(30))
    
    r1 = big_data.groupBy(lambda x: x).mapValue(len).collect()
    r2 = big_data.map(lambda x: (x, 1)).groupByKey().mapValue(len).collect()

这两种做法都可能有上述隐患，更好的做法是

::

    r3 = big_data.map(lambda x: (x, 1)).reduceByKey(lambda x, y: x + y).collect()

用 reduceByKey 来加快缩小数据。对合并后的 value 没整体需求的，都可以考虑用这种方式。

合理设置 Task 和 Memory
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- 大部分 reduce 函数都支持设置 Task 数量和 每个 Task 占用的内存，现在默认分别为 12 和 1000M
- 通常，一个脚本中的各个 Job 所需要的资源是不一样的，而 -M 参数会统一设置内存，所以建议复杂脚本不要使用 -M
- Task 最大使用申请内存的 1.5 倍(将来会改成 1 倍)，超过会失败，会在当前申请内存上乘 2 重试，最多重试 4 次，这个过程可以从 log 中看到
- 因为现在允许内存适当超标，所以也可能发生 Task 所在机器的内存不够而杀掉进程的情况
- 如果 log 中发现大量的内存报错，可以适当的增加 Task 和 Memroy
- reduce 类的可以只增加 Task
- groupBy 可能导致数据不平衡，需要兼顾 Task 和 Memory
- 调整要逐步进行，重复进行“看警告，调参数”这个过程


一些实际的例子
--------------------

延时计算陷阱
~~~~~~~~~~~~~~~~~~~~

Dpark 是延时计算的，因此在使用结果的时候，要考虑是否已经计算过了

::

    dpark = DparkContext()
    rdd = dpark.parallelize(range(10))
    acc = dpark.accumulator(0)
    def sum(x):
        acc.add(x)
        return x
    
    rdd = rdd.map(sum)  # 如去掉赋值则属于无用代码
    print acc.value  # 0
    rdd.count()
    print acc.value  # 45


闭包陷阱
~~~~~~~~~~~~~~~~~~~~

Python 本身的闭包可能会导致一些问题，开发的时候要注意一下

::

    from copy import copy
    dpark = DparkContext()
    # expect: [(0,0), (0,1), (1,1), (0,2), (1,2), (2,2)]
    
    rdd = dpark.union([dpark.makeRDD(range(i+1)).map(lambda x: (x,i)) for i in range(3)])
    print rdd.collect()  # but failed
    
    rdd = dpark.union([dpark.makeRDD(range(i+1)).map(lambda x: (x,copy(i))) for i in range(3)])
    print rdd.collect()  # still failed

这个问题是因为 Python 的变量绑定是语义范围，即闭包中的对象是由某个环境 + 变量名来决定的，而不是对象本身。一个解决办法是使用两层函数，另一个更简单的办法是使用函数的默认值，比如

::

    for i in range(10):
       dpark.map((lambda i: lambda x: x + i)(i))  # 第一种方法，嵌套函数
       dpark.map(lambda x,i=i: x + i)  # 第二种方法，默认值


合理使用 groupBy
~~~~~~~~~~~~~~~~~~~~

- 也有必须使用 groupBy 的场合，比如使用 bid 计算 session
- 还有需要利用 groupBy 来减少耗时操作的场合，比如现有 UA 库过慢，先对 UA 做 groupBy 以减少解析次数，或者对出现过多的 UA 预先进行解析，然后广播出去
- 这种情况需要考虑数据不均衡的情况，大体思路都是拆分过大的 splits，但是仍然需要设置合适的 Memory
- 具体例子可以看 /mfs/datasupport/xiliang_moria/fact_web_log2.py
