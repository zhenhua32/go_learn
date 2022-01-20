# 前言

原始文档是从哪个网址中获取的, 已经遗忘了. 所幸保留着 pdf 文件, 以供对比.

[spark.pdf](../file/spark.pdf)

# 目标

Spark 是并行处理和集群计算的事实标准. 我们会介绍 PySpark 的基础,Spark 的 Python API, 以及对 Spark 机器学习包做一个简短的介绍.

# Apache Spark

在主要的计算框架之上, Spark 提供了机器学习, SQL, 图分析和流式处理库.

Spark 的 Python API 可以通过 PySpark 包访问. 从本地执行环境或者远程连接到一个集群, 开始使用 conda 或 pip 命令安装.

```bash
# PySpark installation with conda
>>> conda install -c conda-forge pyspark
# PySpark installation with pip
>>> pip install pyspark
```

# PySpark

要运行交互式的命令行, 可以在命令行中启动 `pyspark`.

在交互式命令行中, `SparkSession` 对象会作为 `spark` 变量预先存在, 可直接使用.
而在普通的 python 脚本或 IPython 中, 你需要明确地定义这个对象.

当你运行完成后, 应该调用 `spark.stop()` 停止 `SparkSession`.

```python
>>> from pyspark.sql import SparkSession
# instantiate your SparkSession object
>>> spark = SparkSession\
... .builder\
... .appName("app_name")\
... .getOrCreate()
# stop your SparkSession
>>> spark.stop()
```

# RDD: Resilient Distributed Datasets

RDD 是最基础的数据结构. RDD 是**不可变**的分布式的对象的集合.

在 RDD 上执行一个操作, 会产生一个新的 RDD, 而不会改变原始的 RDD.

有两种主要的创建 RDD 的方式:

- 直接将一个文件或目录读入 Spark
- 并行化一个已存在的集合(list, numpy array, pandas dataframe 等)

```python
# SparkSession available as spark
# load the data directly into an RDD
>>> titanic = spark.sparkContext.textFile('titanic.csv')
# the file is of the format
# Survived | Class | Name | Sex | Age | Siblings/Spouses Aboard | Parents/←-
Children Aboard | Fare
>>> titanic.take(2)
['0,3,Mr. Owen Harris Braund,male,22,1,0,7.25',
'1,1,Mrs. John Bradley (Florence Briggs Thayer) Cumings,female,38,1,0,71.283']
# note that each element is a single string - not particularly useful
# one option is to first load the data into a numpy array
>>> np_titanic = np.loadtxt('titanic.csv', delimiter=',', dtype=list)
# use sparkContext to parallelize the data into 4 partitions
>>> titanic_parallelize = spark.sparkContext.parallelize(np_titanic, 4)
>>> titanic_parallelize.take(2)
[array(['0', '3', ..., 'male', '22', '1', '0', '7.25'], dtype=object),
array(['1', '1', ..., 'female', '38', '1', '0', '71.2833'], dtype=object)]

```

# RDD 操作

对 RDD 有两个主要的操作, transformations 和 actions.
transformations 是通过函数从已存在的 RDD 产生一个新的 RDD, 是惰性的.

## transformations

最常用的 transformations 是 `map` 函数, 通过对 RDD 中的每一个元素应用一个函数.

`flatMap` 类似于 `map`, 返回的是抚平后的结果, 就是如果是嵌套数组, 变成一个单一的数组.

```python
# use map() to format the data
>>> titanic = spark.sparkContext.textFile('titanic.csv')
>>> titanic.take(2)
['0,3,Mr. Owen Harris Braund,male,22,1,0,7.25',
'1,1,Mrs. John Bradley (Florence Briggs Thayer) Cumings,female,38,1,0,71.283']
# apply split(',') to each element of the RDD with map()
>>> titanic.map(lambda row: row.split(','))\
... .take(2)
[['0', '3', 'Mr. Owen Harris Braund', 'male', '22', '1', '0', '7.25'],
['1', '1', ..., 'female', '38', '1', '0', '71.283']]
# compare to flatMap(), which flattens the results of each row
>>> titanic.flatMap(lambda row: row.split(','))\
... .take(2)
['0', '3']
```

使用 `filter` 可以过滤数据, 只留下满足条件的.

```python
# create a new RDD containing only the female passengers
>>> titanic_f = titanic.filter(lambda row: row[3] == 'female')
>>> titanic_f.take(3)
[['1', '1', ..., 'female', '38', '1', '0', '71.2833'],
['1', '3', ..., 'female', '26', '0', '0', '7.925'],
['1', '1', ..., 'female', '35', '1', '0', '53.1']]

```

一个非常有用的 transformation 函数是 `distinct`, 可以用来去重.

```python
titanic.map(lambda row: row[1])\
... .distinct()\
... .collect()
```

| spark 命令     | 转换效果                                          |
| -------------- | ------------------------------------------------- |
| map(f)         | 对 RDD 中每个元素应用 f 函数, 返回一个新 RDD      |
| flatMap(f)     | 和 map(f) 相同, 除了返回的结果是扁平的            |
| filter(f)      | 返回满足 f 函数的元素                             |
| distinct()     | 返回去重后的元素                                  |
| reduceByKey(f) | RDD 中元素是 (key, val) 结构的, 用 f 函数合并结果 |
| sortBy(f)      | 使用给定的函数 f 排序 RDD                         |
| sortByKey(f)   | RDD 中元素是 (key, val) 结构的, 用 f 函数排序 RDD |
| groupBy(f)     | 基于函数 f, 返回分组后的 RDD                      |
| groupByKey(f)  | RDD 中元素是 (key, val) 结构的, 用 f 函数分组 RDD |

函数名中带有 `Key` 的是对 RDD 中每个元素的格式有要求, 要求是 (key, val) 组成的.
然后调用后, 返回的元素也是 (key, value) 结构的.

```python
# the following counts the number of passengers in each class
# note that this isn't necessarily the best way to do this

# create a new RDD of (pclass, 1) elements to count occurances
>>> pclass = titanic.map(lambda row: (row[1], 1))
>>> pclass.take(5)
[('3', 1), ('1', 1), ('3', 1), ('1', 1), ('3', 1)]

# count the members of each class
>>> pclass = pclass.reduceByKey(lambda x, y: x + y)
>>> pclass.collect()
[('3', 487), ('1', 216), ('2', 184)]

# sort by number of passengers in each class, ascending order
>>> pclass.sortBy(lambda row: row[1]).collect()
[('2', 184), ('1', 216), ('3', 487)]
```

## actions

actions 是返回非 RDD 对象的操作.

两个主要的 actions 是 `take(n)` 和 `collect()`. `take(n)` 返回前 N 个元素, `collect` 返回整个 RDD 数据.

另一个重要的 action 是 `reduce(func)`. 注意, func 必须是结合和交换的二元操作.

```python
# create an RDD with the first million integers in 4 partitions
>>> ints = spark.sparkContext.parallelize(range(1, 1000001), 4)
# [1, 2, 3, 4, 5, ..., 1000000]
# sum the first one million integers
>>> ints.reduce(lambda x, y: x + y)
500000500000
# create a new RDD containing only survival data
>>> survived = titanic.map(lambda row: int(row[0]))
>>> survived.take(5)
[0, 1, 1, 1, 0]
# find total number of survivors
>>> survived.reduce(lambda x, y: x + y)
342
```

| spark 命令           | 转换效果                                      |
| -------------------- | --------------------------------------------- |
| take(n)              | 返回 RDD 的前 N 个元素                        |
| collect()            | 返回整个 RDD 数据                             |
| reduce(f)            | 使用函数 f 合并数据                           |
| count()              | 返回 RDD 中元素的数量                         |
| min(); max(); mean() | 返回 RDD 的最小, 最大, 平均值                 |
| sum()                | 对 RDD 中的元素求和                           |
| saveAsTextFile(path) | 将 RDD 中的数据保存在目录中, 每个分片一个文件 |
| foreach(f)           | 对 RDD 中的每个元素立即调用函数 f             |

# DataFrames

RDD 用 Python 实现比 Scala 和 Java 慢, 所以需要使用 DataFrames.

DataFrames 也是不可变的分布式数据集合, 但是 DataFrames 是用列组织的.
非常类似于关系数据库, 或者是 pandas 的 DataFrame.

使用 DataFrames 时, Spark’s Catalyst Optimizer 创建逻辑计划, 然后在实际执行时优化成物理计划.

# Spark SQL 和 DataFrames

从已有的 text, csv, json 文件中创建 DataFrame, 要比创建 RDD 容易些.
DataFrame API 有各种参数可以处理表格头, 或者自动推断 schema.

```python
# load the titanic dataset using default settings
>>> titanic = spark.read.csv('titanic.csv')
>>> titanic.show(2)
+---+---+--------------------+------+---+---+---+-------+
|_c0|_c1| _c2| _c3|_c4|_c5|_c6| _c7|
+---+---+--------------------+------+---+---+---+-------+
| 0| 3|Mr. Owen Harris B...| male| 22| 1| 0| 7.25|
| 1| 1|Mrs. John Bradley...|female| 38| 1| 0|71.2833|
+---+---+--------------------+------+---+---+---+-------+
only showing top 2 rows
# spark.read.csv('titanic.csv', inferSchema=True) will try to infer
# data types for each column
# load the titanic dataset specifying the schema
>>> schema = ('survived INT, pclass INT, name STRING, sex STRING, '
... 'age FLOAT, sibsp INT, parch INT, fare FLOAT'
... )
>>> titanic = spark.read.csv('titanic.csv', schema=schema)
>>> titanic.show(2)
+--------+------+--------------------+------+---+-----+-----+-------+
|survived|pclass| name| sex|age|sibsp|parch| fare|
+--------+------+--------------------+------+---+-----+-----+-------+
| 0| 3|Mr. Owen Harris B...| male| 22| 1| 0| 7.25|
| 1| 1|Mrs. John Bradley...|female| 38| 1| 0|71.2833|
+--------+------+--------------------+------+---+-----+-----+-------+
only showing top 2 rows

# for files with headers, the following is convenient
spark.read.csv('my_file.csv', header=True, inferSchema=True
```

要从 DataFrame 转换成 RDD ,而可以直接使用 `my_df.rdd`,
要从 RDD 转换成 DataFrame, 可以使用 `spark.createDataFrame(my_rdd)`.

DataFrame 可以使用 SQL 操作更新, 查询和分析. `pyspark.sql.functions` 模块下有很多用于分析的函数.

```python
# select data from the survived column
>>> titanic.select(titanic.survived).show(3) # or titanic.select("survived")

# find all distinct ages of passengers (great for data exploration)
>>> titanic.select("age")\
... .distinct()\
... .show(3)

# filter the DataFrame for passengers between 20-30 years old (inclusive)
>>> titanic.filter(titanic.age.between(20, 30)).show(3)

# find total fare by pclass (or use .avg('fare') for an average)
>>> titanic.groupBy('pclass')\
... .sum('fare')\
... .show()

# group and count by age and survival; order age/survival descending
>>> titanic.groupBy("age", "survived").count()\
... .sort("age", "survived", ascending=False)\
... .show(2)

# join two DataFrames on a specified column (or list of columns)
>>> titanic_cabins.show(3)
>>> titanic.join(titanic_cabins, on='name').show(3)
```

如果喜欢直接写 SQL 语句, 可以使用 `spark.sql("SQL QUERY")`.
注意, 首先要创建一个 DataFrame 的临时视图.

```python
# create the temporary view so we can access the table through SQL
>>> titanic.createOrReplaceTempView("titanic")

# query using SQL syntax
>>> spark.sql("SELECT age, COUNT(*) AS count\
... FROM titanic\
... GROUP BY age\
... ORDER BY age DESC").show(3)
```

| Spark SQL 命令                  | SQLite 命令                       |
| ------------------------------- | --------------------------------- |
| select(\*cols)                  | SELECT                            |
| groupBy(\*cols)                 | GROUP BY                          |
| sort(\*cols, \*\*kwargs)        | ORDER BY                          |
| filter(condition)               | WHERE                             |
| when(condition, value)          | WHEN                              |
| between(lowerBound, upperBound) | BETWEEN                           |
| drop(\*col)                     | DROP                              |
| join(other, on=None, how=None)  | JOIN(join, type specified by how) |
| count()                         | COUNT()                           |
| sum(\*cols)                     | SUM()                             |
| avg(\*cols) 或 mean(\*cosl)     | AVG()                             |
| collect()                       | fetchall()                        |

# 机器学习

主要的机器学习 API 是 `pyspark.ml`, 这是基于 DataFrame 的.
基于 RDD 的机器学习 API 是 `pyspark.mllib`, 且更慢.

```python
# prepare data
# convert the 'sex' column to binary categorical variable
>>> from pyspark.ml.feature import StringIndexer, OneHotEncoderEstimator
>>> sex_binary = StringIndexer(inputCol='sex', outputCol='sex_binary')

# one-hot-encode pclass (Spark automatically drops a column)
>>> onehot = OneHotEncoderEstimator(inputCols=['pclass'],
... outputCols=['pclass_onehot'])

# create single features column
from pyspark.ml.feature import VectorAssembler
features = ['sex_binary', 'pclass_onehot', 'age', 'sibsp', 'parch', 'fare']
features_col = VectorAssembler(inputCols=features, outputCol='features')

# now we create a transformation pipeline to apply the operations above
# this is very similar to the pipeline ecosystem in sklearn
>>> from pyspark.ml import Pipeline
>>> pipeline = Pipeline(stages=[sex_binary, onehot, features_col])
>>> titanic = pipeline.fit(titanic).transform(titanic)

# drop unnecessary columns for cleaner display (note the new columns)
>>> titanic = titanic.drop('pclass', 'name', 'sex')
>>> titanic.show(2)

# split into train/test sets (75/25)
>>> train, test = feature_vector.randomSplit([0.75, 0.25], seed=11)

# initialize logistic regression
>>> from pyspark.ml.classification import LogisticRegression
>>> lr = LogisticRegression(labelCol='survived', featuresCol='features')

# run a train-validation-split to fit best elastic net param
# ParamGridBuilder constructs a grid of parameters to search over.
>>> from pyspark.ml.tuning import ParamGridBuilder, TrainValidationSplit
>>> from pyspark.ml.evaluation import MulticlassClassificationEvaluator as MCE
>>> paramGrid = ParamGridBuilder()\
... .addGrid(lr.elasticNetParam, [0, 0.5, 1]).build()

# TrainValidationSplit will try all combinations and determine best model using
# the evaluator (see also CrossValidator)
>>> tvs = TrainValidationSplit(estimator=lr,
... estimatorParamMaps=paramGrid,
... evaluator=MCE(labelCol='survived'),
... trainRatio=0.75,
... seed=11)

# we train the classifier by fitting our tvs object to the training data
>>> clf = tvs.fit(train)

# use the best fit model to evaluate the test data
>>> results = clf.bestModel.evaluate(test)
>>> results.predictions.select(['survived', 'prediction']).show(5)

# performance information is stored in various attributes of "results"
>>> results.accuracy
>>> results.weightedRecall
>>> results.weightedPrecision

# many classifiers do not have this object-oriented interface (yet)
# it isn't much more effort to generate the same statistics for a ←-
DecisionTreeClassifier, for example
>>> dt_clf = dt_tvs.fit(train) # same process, except for a different paramGrid
# generate predictions - this returns a new DataFrame
>>> preds = clf.bestModel.transform(test)
>>> preds.select('survived', 'probability', 'prediction').show(5)

# initialize evaluator object
>>> dt_eval = MCE(labelCol='survived')
>>> dt_eval.evaluate(preds, {dt_eval.metricName: 'accuracy'})
```

| PySpark ML Module         | Module Purpose                                                       |
| ------------------------- | -------------------------------------------------------------------- |
| pyspark.ml.feature        | provides functions to transform data into feature vectors            |
| pyspark.ml.tuning         | grid search, cross validation, and train/validation split functions  |
| pyspark.ml.evaluation     | tools to compute prediction metrics (accuracy, f1, etc.)             |
| pyspark.ml.classification | classification models (logistic regression, SVM, etc.)               |
| pyspark.ml.clustering     | clustering models (k-means, Gaussian mixture, etc.)                  |
| pyspark.ml.regression     | regression models (linear regression, decision tree regressor, etc.) |

# 额外操作

```python
# some immediately accessible functions
# covariance between pclass and fare
>>> titanic.cov('pclass', 'fare')

# summary of statistics for selected columns
>>> titanic.select("pclass", "age", "fare")\
... .summary().show()

# additional functions from the functions module
>>> from pyspark.sql import functions as sqlf
# finding the mean of a column without grouping requires sqlf.avg()
# alias(new_name) allows us to rename the column on the fly
>>> titanic.select(sqlf.avg("age").alias("Average Age")).show()

# use .agg([dict]) on GroupedData to specify [multiple] aggregate
# functions, including those from pyspark.sql.functions
>>> titanic.groupBy('pclass')\
... .agg({'fare': 'var_samp', 'age': 'stddev'})\
... .show(3)

# perform multiple aggregate actions on the same column
>>> titanic.groupBy('pclass')\
... .agg(sqlf.sum('fare'), sqlf.stddev('fare'))\
... .show()
```

| pyspark.sql.functions    | Operation                                                                                                  |
| ------------------------ | ---------------------------------------------------------------------------------------------------------- |
| ceil(col)                | computes the ceiling of each element in col                                                                |
| floor(col)               | computes the floor of each element in col                                                                  |
| min(col), max(col)       | returns the minimum/maximum value of col                                                                   |
| mean(col)                | returns the average of the values of col                                                                   |
| stddev(col)              | returns the unbiased sample standard deviation of col                                                      |
| var_samp(col)            | returns the unbiased variance of the values in col                                                         |
| rand(seed=None)          | generates a random column with i.i.d. samples from [0, 1]                                                  |
| randn(seed=None)         | generates a random column with i.i.d. samples from the standard normal distribution                        |
| exp(col)                 | computes the exponential of col                                                                            |
| log(arg1, arg2=None)     | returns arg1-based logarithm of arg2; if there is only one argument, then it returns the natural logarithm |
| cos(col), sin(col), etc. | computes the given trigonometric or inverse trigonometric (asin(col), etc.) function of col                |
