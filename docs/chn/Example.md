### word count最简单版本
有一堆文件，每一行是一个word，统计一共有多少个word

    words = ctx.textFile(file_path, splitSize=32<<20)
    return words.count() # 重复算作多次
    
    return words.uniq().count() # 重复算作一次

### 升级版一
有一堆文件，每一行是一些word，word之间以','隔开，统计一共有多少个word，重复算作多次

    words = ctx.textFile(file_path, splitSize=32<<20)
    return words.map(
            lambda line: len(line.strip().split(','))
        ).reduce(lambda x, y: x+y)

### 升级版二
有一些文章，给定一个word，统计这个word出现的次数

    article = ctx.textFile(file_path)
    word_count = ctx.accumulator(0)
    article.foreach(lambda line: word_count.add(line.count(word)))
    return word_count.value

### 升级版三
有一堆文件，每一行是一些word，word之间以','隔开，统计有多少长度大于10的word，重复算作多次

    N = 10
    words = ctx.textFile(file_path, splitSize=32<<20)
    return words.flatMap(
            lambda line: line.strip().split(',')
        ).filter(
            lambda word: len(word) > N
        ).count()

### 升级版四
有一堆文件，每一行是一个word，统计每个word有多少个，并保存到文件

    words = ctx.textFile(file_path, splitSize=32<<20)
    return words.map(
            lambda word: (word.strip(), 1) # 由于textFile会将回车符包含在内，所以需要调用strip()去掉回车符
        ).reduceByKey(
            lambda x, y: x+y
        ).map(
            lambda (word, count): '%s %s' % (word, count) # saveAsTextFile只接受字符串类型，所以需要进行这一步转换
        ).saveAsTextFile("/mfs/tmp/search-keyword-count", overwrite=True)

### 升级版五
有一堆文件，每一行的格式为`word,uid`，给定一个word的列表，要求统计此列表中每个word对应的所有uid

    user_word = ctx.csvFile(file_path, splitSize=32<<20)
    words = ctx.broadcast(words_list) # 由于word_list可能很大，所以我们用广播变量来提高效率
    return user_word.map(tuple).filter(
            lambda (word, uid): word in words.value
        ).groupByKey().mapValue(set).collectAsMap()

### 升级版六
有一堆文件，每一行的格式为`word,uid`，统计每个uid对应出现次数最多的10个word

    N = 10
    from dpark.dependency import Aggregator
    from operator import itemgetter

    def createCombiner(value):
        return [value]
    def mergeValue(c, v):
        return mergeCombiner(c, createCombiner(v))
    def mergeCombiner(c1, c2):
        # 每个combiner都是(word, count)的数组
        return sorted(c1+c2, key=itemgetter(1), reverse=True)[:N]
    agg = Aggregator(createCombiner, mergeValue, mergeCombiner)

    WORD, UID = range(2)
    words = ctx.csvFile(file_path, splitSize=32<<20)
    user_words = words.map(
            lambda line: ((line[UID], line[WORD]), 1)
        ).reduceByKey(lambda x, y: x+y)
    return user_words.map(
            lambda ((uid, word), count): (uid, (word, count))
        ).combineByKey(agg).collectAsMap()

