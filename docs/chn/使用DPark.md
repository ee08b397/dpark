## 初步认识DPark
DPark的核心概念是提供了弹性分布式数据集(RDD)，它由一些块组成(Split)，每一块中的元素可以被并行的在集群节点上进行操作。RDD目前支持两种数据源：一个已经存在的Python数据集合（例如list等）、以及存储在分布式文件系统上的文件。

DPark提供的另一个概念是共享变量，所谓共享是指其可在并行计算的过程中被使用。默认的，当DPark并行的运行一个函数的时候，它会将函数所需使用的变量拷贝到各个节点上，但是，有时候，需要在不同的分任务之间或者主程序和分任务之间共享一个变量，在这种情况下，DPark目前提供了两种类型的共享变量：广播变量和累加器。

我们首先将通过两个很简单的例子来初步认识一下DPark。

#### 例子一：估算π
一个估算PI很简单的办法是生成一系列在单位正方形((-1, -1) 到 (1, 1))的随机点，计算它们当中有多少是落在单位圆当中的。由于单位元的面积是PI，而单位正方形的面积是4，所以这个的概率应该是PI/4。如果我们取的点数越多，则计算出来的PI越精确。

这个算法的普通Python实现如下：  
 
    from random import random
    N = 100000
    count = 0
    for i in range(0, N):
        x = random() * 2 - 1
        y = random() * 2 - 1
        if x * x + y * y < 1:
            count += 1

    print 'PI is roughly', 4.0 * count / N

将其做一点点修改即可使用DPark来实现这个算法：

    from dpark import DparkContext
    dpark = DparkContext()
    count = dpark.accumulator(0)

    def random_once(*args, **kwrgs):
        x = random() * 2 - 1
        y = random() * 2 - 1
        if x * x + y * y < 1:
            count.add(1)

    dpark.parallelize(range(0, N), 10).foreach(random_once)
    print 'PI is roughly', 4.0 * count.value / N

代码和之前的版本的区别主要如下：

1. 我们新建了一个DparkContext，我们可以在此处规定希望与哪个Mesos主机进行沟通，具体请见第二部分。
1. 由于普通的变量并不能在并行计算过程当中被共享，所以我们新建了一个累加器来进行统计。
1. 为了区分累加器和普通的变量，使用add方法代替了操作符+，并且在print的时候，需要使用count.value。
1. dpark.parallelize(range(0, N), 10)将一个列表变为一个弹性分布式数据集(RDD)。它接受两个参数，第一个是列表类型，第二个表示是分割的块数。对于这句话来说，DPark会将这个列表分为10块，并分布式的执行10个任务。
1. RDD的foreach函数接受一个函数作为参数，在进行计算的时候，这个函数及其所需要的闭包(closure)会和RDD的一块数据被序列化后传递到目标机器上进行计算。


#### 例子二：简单的word count
我们假设现在有一个文件，文件当中每一行都是一些句子，需要统计某一个单词在其中出现的次数。

普通的Python实现为：

    def word_count(file_path, word):
        count = 0
        for line in open(file_path):
            if word in line.strip():
                count += 1
        print word, 'count:', count

变为DPark的并行实现就是：

    from dpark import DparkContext
    
    def word_count(file_path, word):   
        dpark = DparkContext()
        f = dpark.textFile(file_path, splitSize=16<<20)
        
        print word, 'count:', f.map(lambda line: line.strip()).filter(lambda line: word in line).count()

1. 这里使用textFile将一个文件变为一个RDD，其中splitSize定义了多大为一块，DPark会自动将每一行作为一个元素以供之后的各种操作使用，当然这儿的读取文件是惰性的，也就是说只有当你进行一系列变换的时候才会真正的去读取文件。
1. map的作用是对RDD中的元素进行变换。
1. filter的作用是对RDD中的元素进行过滤。
1. count将统计出RDD中一共有多少个元素。

## 运行DPark程序
使用DPark API编写的程序目前支持三种方式的运行：本机单进程，本机多进程，mesos分布式计算。你需要通过命令行参数的方式来告诉DPark采用哪一种方式运行你的程序。

1. 当你不加任何参数运行的时候，默认是本机单进程运行，以方便进行调试    
`python demo.py`
2. 如果加上了`-m process`参数，那么则会使用本机多进程运行，你可以使用`-p 4`来规定希望使用的进程数量    
`python demo.py -m process -p 4`
3. 如果加上了`-m mesos://HOST:PORT`参数，则会使用mesos分布式运行程序     
`python demo.py -m mesos://HOST:PORT`

如果应用脚本有自己的命令行参数时，需要在`dpark.optParser`的基础上进行添加，并且需要在初始化`DparkContext`之前进行添加。比如：

    from dpark import DparkContext, optParser
    optParser.add_option("-o", "--opt")
    dpark = DparkContext()
    options, args = optParser.parse_args()

## 弹性分布式数据集 (RDD)
DPark的核心概念就是使用RDD，RDD是一种支持容错的、可进行并行计算的元素集合。一个RDD由多个块(Split)组成，而块是并行计算过程中的基本单位。
目前DPark支持两种形式的数据源：       
1. 并行化的数据集：即将一个普通的Python数据集合（例如list）拆分成若干块后组成一个RDD。
1. 存储在分布式文件系统上的单个或多个文件：即将一个普通的文件按行拆分成若干块之后组成一个RDD，目前支持文本文件格式和CSV文件格式。

以上通过两种数据源生成的RDD以及其通过并行计算得到的新的RDD都支持相同的操作和变换。      
#### 并行化的数据集
并行化的数据集可以通过调用`DparkContext`的`parallelize`函数得到     
`rdd = ctx.parallelize(seq, numSlices)`    
其中第二个参数numSlices表示seq将被拆分成的块数，如上所述，DPark会把每一块作为一个任务分配给集群进行运算。通常情况下，你应该为mesos集群上的每个CPU分配2-4个任务。numSlices有默认值，是根据你运行模式的不同而决定的。
#### 分布式文件系统
DPark可以从存储在分布式文件系统中的一个或多个文件来得到RDD，目前支持的文件格式有文本文件和CSV文件，未来将会加入更多格式的文件支持，如MySQL，当然你可以自己仿照dpark/rdd.py中`TextFileRDD`来实现对应的RDD。     
###### 文本文件RDD
`text_rdd = ctx.textFile('data.txt', splitSize=64<<20)`   
  
创建出来的RDD中每个元素为源文件中的一行，包含回车符在内。`splitSize`这个参数规定了RDD中每一块(Split)的大小，默认为64M。
###### CSV文件RDD
`csv_rdd = ctx.csvFile('data.csv', splitSize=32<<20, dialect='excel')`    
 
创建出来的RDD中每个元素为源文件中每一行分割后生成的数组。csvFile同样接受splitSize这个参数，另外，它还接受`dialect`参数，用以规定源csv文件使用哪种符号进行分割，具体请参见[csv.reader](http://docs.python.org/library/csv.html#csv.reader)，默认使用逗号('excel')分割。

## RDD支持的并行计算
RDD支持两种类型的并行计算：

+ 变换：将现有的RDD变为另一个RDD，比如`map`, `filter`
+ 操作：对现有的RDD进行聚合运算并立即将结果返回给主程序，比如`reduce`, `count`

所有RDD支持的变换都是惰性的，当调用变换函数如`map`时，并没有立即执行对应的计算，而是生成了一个携带足够计算信息的新的RDD（例如包含了源RDD和变换函数等），只有当RDD需要将某个操作的结果返回主程序时，它才会真正开始计算。这样做有两个好处：1. 提高效率，DPark可以自动的合并多个变换同时运行，以最大程度的减少数据传输过程；2. 容错，当某块的计算失败时，RDD拥有足够的信息来重新开始计算。

RDD支持的其中一个特别且很重要的变换是缓存(cache)，当你调用了某个RDD的`cache()`函数时，每个计算节点会将其被分配到的计算块的结果缓存在该节点的内存中，下一次需要对这个RDD或者其变换而来的RDD进行操作时，计算节点可以直接从内存中获得该RDD这一块的计算结果，或者重新计算并缓存之。很显然，当某一个RDD会被重复利用时，缓存变换将大大提高计算速度。这个方法对于迭代计算非常有利，在命令行交互执行时也会非常有用。

#### 支持的变换
###### 所有RDD都支持的变换
<table>
<thead>
<td>变换</td>
<td>说明</td>
</thead>
<tr>
<td>`map(func)`</td>
<td>对每一个元素调用func函数，并返回由计算结果组成的RDD</td>
</tr>
<tr>
<td>`filter(func)`</td>
<td>对self中每一个元素调用func函数，并返回那些结果为True的元素组成的RDD</td>
</tr>
<tr>
<td>`flatMap(func)`</td>
<td>同map相似，对self中每一个元素调用func函数，将其返回的集合中的每一个元素作为元素组成新的RDD</td>
</tr>
<tr>
<td>`groupBy(func, numSplits)`</td>
<td>将self中的每个元素value按照func(value)，并将结果相同的value合并，返回由`(func(value), [value])`组成的RDD，`numSplits`规定了结果RDD有多少个块</td>
</tr>
<tr>
<td>`union(rdds)`</td>
<td>将self和rdds中的其他RDD组成一个新的RDD，新的RDD中的块为self和rdds中的所有块的简单拼接</td>
</tr>
<tr>
<td>`__slice__(i, j)`</td>
<td>将self的块数组中的一部分组成新的RDD，相当于`rdd._splits[i:j]`</td>
</tr>
<tr>
<td>`glom()`</td>
<td>相当于 mapPartition(lambda x: [x]), 即把原来 RDD 中的每个 Split 变成新的 RDD 中的一个元素。</td>
</tr>
<tr>
<td>`mapPartition(func)`</td>
<td>对self中每一块调用func函数，并将其结果组成一个新的RDD</td>
</tr>
<tr>
<td>`pipe(command)`</td>
<td>将self中的每一块作为一个输入流传递给command命令，并将结果变成新的块，从而组成新的RDD</td>
</tr>
<tr>
<td>`uniq()`</td>
<td>将self中的重复元素去除后的新RDD，类似于`set()`</td>
</tr>
<tr>
<td>`cartesian(rdd)`</td>
<td>笛卡尔乘积，若self中的元素为T，rdd中的元素为U，则返回一个由(T, U)元素组成的RDD，大小为size(self) * size(rdd)</td>
</tr>
<tr>
<td>`cache()`</td>
<td>要求self的计算结果被缓存在计算节点中，使得之后的计算更加快速</td>
</tr>
</table>
###### 元素为(key, value)对的RDD支持的变换
<table>
<thead>
<td>变换</td>
<td>说明</td>
</thead>
<tr>
<td>`mapValue(func)`</td>
<td>对self中每个元素的value执行func并返回由`(key, func(value))`组成的RDD，相当于`map(lambda (key, value): (key, func(value)))`</td>
</tr>
<tr>
<td>`flatMapValue(func)`</td>
<td>对self中的每个元素的value执行func并将其返回的列表中的每个元素变为新RDD中每个元素的value，相当于`flatMap(lambda (key, value): [(key, v) for v in func(value)])`</td>
</tr>
<tr>
<td>`groupByKey(numSplits=20)`</td>
<td>将self中的元素按key进行合并，返回由`(key, [value])`组成的RDD，其接受一个numSplits参数，规定了返回的RDD有多少个块</td>
</tr>
<tr>
<td>`reduceByKey(func, numSplits=20)`</td>
<td>将self中的元素按key进行合并，并对合并后的[value]用func进行reduce</td>
</tr>
<tr>
<td>`combineByKey(agg, numSplits=20)`</td>
<td>将self中的元素按key进行合并，并对value列表用agg进行聚合，agg为一个dpark.dependency.Aggregator对象，具体请查看[例子](https://github.com/jackfengji/test_pro/wiki/Examples)</td>
</tr>
<tr>
<td>`join(rdd)`</td>
<td>若self中K对应的值为V，有n个，rdd中K对应的值为W，有m个，则返回的新的RDD中K对应的值为(V, W))，共有n*m个</td>
</tr>
<tr>
<td>`groupWith(rdds)`</td>
<td>若self中的元素为(K, V)，rdds[i]中的元素为(K, W[i])，则返回的新的RDD中的元素为(K, [V], [W[1]], ..., [W[i]])</td>
</tr>
<tr>
<td>`lookup(key)`</td>
<td>查找某个key对应的所有元素，只能查询合并过的RDD</td>
</tr>
</table>
#### 支持的操作
<table>
<thead>
<td>操作</td>
<td>说明</td>
</thead>
<tr>
<td>`first()`</td>
<td>返回第一个元素</td>
</tr>
<tr>
<td>`take(n)`</td>
<td>返回前n个元素组成的列表</td>
</tr>
<tr>
<td>`count()`</td>
<td>返回一共有多少个元素</td>
</tr>
<tr>
<td>`collect()`</td>
<td>返回所有元素组成的列表list</td>
</tr>
<tr>
<td>`collectAsMap()`</td>
<td>等于dict(self.collect())</td>
</tr>
<tr>
<td>`reduce(func)`</td>
<td>用func函数对self中的元素进行聚合，func应该接受两个参数并返回一个，而且其应该是可以迭代的，例如add，the function should be associative so that it can be computed correctly in parallel</td>
</tr>
<tr>
<td>`reduceByKeyToDriver(func)`</td>
<td>相比reduceByKey而言，这个函数让你能够更多的控制reduce的过程，func函数应该有两个参数，第一个参数是某个key之前所有value的reduce结果或者第一个value，第二个参数是一个新的value</td>
</tr>
<tr>
<td>`saveAsTextFile(dir, overwrite=False)`</td>
<td>将self中的每一块保存为一个文本文件，其中每个元素为文件的一行，所以必须保证每个元素都是一个字符串</td>
</tr>
<tr>
<td>`foreach(func)`</td>
<td>对每个元素执行func函数，无返回值，通常和累加器搭配使用</td>
</tr>
</table>

## 共享变量
通常情况下，DPark的具体计算过程会发生在集群的某个结点上，所以DPark会将RDD的某一块数据以及计算函数（例如map和reduce的函数参数）序列化之后传递到该结点上，并在结点上反序列化后执行，同时该计算函数其所依赖的全局变量、模块和闭包对象等也会随着函数本身一起被复制到计算节点中，所以在节点中对普通变量等的修改并不会影响到主程序中的变量。

不过，如最开始所述，DPark另一个核心概念为共享变量，即可以在不同的任务之间共享的可读写的变量，DPark目前提供了两种类型的共享变量：只读的广播变量以及只可写的累加器。

注意，由于DPark需要将变量、模块、闭包等序列化后传递到计算结点去，对于那些无法被序列化的对象（比如c拓展中的函数），需要进行一些适当的封装才可以，比如：

    def c_func(v):
        from c_module import func
        return func(v)
    rdd.map(c_func)

#### 广播变量
广播变量允许你将某一个变量一次性发布到所有的节点上，避免在每次调用的时候都执行序列化和反序列化的过程。它通常被使用于当计算函数依赖于一个比较大的数据集的情况下。当然，被广播的对象不得超过单个计算结点的内存限制。DPark尽量使用更好的广播算法来提高广播的效率，目前支持分布式文件系统和Tree型结构的广播算法。

广播变量通过对某个python变量`v`执行`ctx.broadcast(v)`来得到，这句话会将`v`用一个广播变量给封装起来，真实的数据需要通过广播变量的`value`属性来获得。

    >>> v = range(100000)
    >>> bv = ctx.broadcast(v)
    >>> bv
    <dpark.broadcast.TreeBroadcast instance at 0xab7e60>
    >>> len(bv.value)
    100000

#### 累加器
累加器可以在小任务的并行计算过程中收集其产生的少量数据，如名字所述，累加器只支持`add`操作，不支持删除、更新等其他操作。默认实现的累加器支持数值类型、list类型以及dict类型，你可以自己拓展。

通过执行`ctx.accumulator(v)`来创建一个初始值为`v`的累加器，之后，在并行计算的过程中，可以调用累加器的`add(value)`方法来增加累加器的值。

    accum = ctx.accumulator(0)
    ctx.parallelize(range(0, 100000), 100).foreach(lambda i: accum.add(i))
    print accum.value

注意：

1. 由于累加器的add操作并不会被马上同步到所有的结点上，所以在并行计算过程中，你无法获得累加器正确的值
2. 由于RDD变换的惰性特点，如果你在变换的过程中对累加器进行一些add操作，由于变换没有被马上执行，此时累加器的值并没有被改变


