### Simplest version of word count
There are numbers of files. Each line in files is a word. Count how many words totally.

    words = ctx.textFile(file_path, splitSize=32<<20)
    return words.count() # repeated words are treated as different words 
    
    return words.uniq().count() # repeated words are treated as one word

### updated version I
There are numbers of files. Each line in files is some words separated by ','. Count how many words totally. Repeated words are treated as different words.

    words = ctx.textFile(file_path, splitSize=32<<20)
    return words.map(
            lambda line: len(line.strip().split(','))
        ).reduce(lambda x, y: x+y)

### updated version II
Given numbers of papers and a word, count how many times this word appears. 

    article = ctx.textFile(file_path)
    word_count = ctx.accumulator(0)
    article.foreach(lambda line: word_count.add(line.count(word)))
    return word_count.value

### updated version III
There are numbers of files. Each line is some words separated by ','. Count how many words longer than 10. Repeated words are treated as different.

    N = 10
    words = ctx.textFile(file_path, splitSize=32<<20)
    return words.flatMap(
            lambda line: line.strip().split(',')
        ).filter(
            lambda word: len(word) > N
        ).count()

### Updated version IV
There are numbers of files. Each line contains one word. Count how many words there are and save in a file.

    words = ctx.textFile(file_path, splitSize=32<<20)
    return words.map(
            lambda word: (word.strip(), 1) # textFile returns with '\n', call strip() to delete '\n'
        ).reduceByKey(
            lambda x, y: x+y
        ).map(
            lambda (word, count): '%s %s' % (word, count) # saveAsTextFile only receive string, so convert is needed
        ).saveAsTextFile("/mfs/tmp/search-keyword-count", overwrite=True)

### Updated version V
There are a number of files. Each line contains a 'word,uid' pair. Given a word list, for every word in this list, calculate all 'uid's mapped to this word.

    user_word = ctx.csvFile(file_path, splitSize=32<<20)
    words = ctx.broadcast(words_list) # word_list may be very large, so we use broadcast to speed up
    return user_word.map(tuple).filter(
            lambda (word, uid): word in words.value
        ).groupByKey().mapValue(set).collectAsMap()

### Updated version VI
There is a number of files. Each line contains a 'word,uid' pair. For every uid, count the 10 most frequent words.

    N = 10
    from dpark.dependency import Aggregator
    from operator import itemgetter

    def createCombiner(value):
        return [value]
    def mergeValue(c, v):
        return mergeCombiner(c, createCombiner(v))
    def mergeCombiner(c1, c2):
        # Every combiner is an array of (word, count) 
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

