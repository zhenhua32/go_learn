<!-- TOC -->

- [全文索引](#全文索引)
- [准备工作](#准备工作)
- [MYSQL 对全文索引的支持](#mysql-对全文索引的支持)
- [搜索模式](#搜索模式)
- [简单示例](#简单示例)
- [实现细节与中文搜索](#实现细节与中文搜索)
- [布尔搜索](#布尔搜索)
- [ngram 解析器](#ngram-解析器)
  - [配置 token size](#配置-token-size)
  - [使用 ngram 解析器创建 FULLTEXT 索引](#使用-ngram-解析器创建-fulltext-索引)
  - [其他细节](#其他细节)
- [参考](#参考)

<!-- /TOC -->

## 全文索引

FULLTEXT 索引是创建在文本类型(CHAR, VARCHAR, TEXT)的列上的, 用来加速字符匹配上的查询.

## 准备工作

启动 mysql:

```bash
docker run --name ttsql -e MYSQL_ROOT_PASSWORD=1234 LANG=C.UTF-8 -d mysql
```

连接前面启动的 mysql:

```bash
docker exec -it ttsql mysql -uroot -p1234
```

创建测试用的数据库, 并切换:

```bash
CREATE DATABASE test;
use test
```

## MYSQL 对全文索引的支持

- 在 MYSQL 中全文索引是类型为 `FULLTEXT` 的索引.
- 全文索引只能用在 `InnoDB` 或 `MyISAM` 表上, 且只能在 `CHAR`, `VARCHAR`, `TEXT` 类型的列上创建
- MYSQL 提供了内置的基于 ngram 的解析器, 用于支持中韩日文;
  还有一个可安装的 MeCAb 解析器插件来支持日文.
- `FULLTEXT` 索引可以在创建表时通过 `CREATE TABLE` 中定义,
  或者在之后 `ALTER TABLE` 和 `CREATE INDEX`.
- 对于大数据集, 应该将数据插入没有 `FULLTEXT` 索引的表中, 再创建全文索引,
  这比将数据插入已创建全文索引的表中更快.

## 搜索模式

使用 `MATCH (col1,col2,...) AGAINST (expr [search_modifier])` 执行全文搜索.

```mysql
search_modifier:
  {
       IN NATURAL LANGUAGE MODE
     | IN NATURAL LANGUAGE MODE WITH QUERY EXPANSION
     | IN BOOLEAN MODE
     | WITH QUERY EXPANSION
  }
```

有以下几种搜索模式:

- 自然语言搜索(`IN NATURAL LANGUAGE MODE`): 是默认的搜索模式.
  将搜索字符串(search string)理解为短语, 同时对双引号 `"` 没有特殊的操作.
- 布尔搜索(`IN BOOLEAN MODE`): 使用特殊查询预言的规则 解释 搜索字符串.
  字符串中包含要搜索的单词. 也会包含特殊指定的规则, 比如某个单词必须在匹配的行中存在或者不存在.
  特定的公共单词(stopwords)会在搜索索引中会忽略, 如果在搜索字符串中出现也不会进行匹配.
- 查询扩展搜索是自然预言搜索的变种.
  搜索字符串用于执行自然预言搜索. 搜索返回最相关的行会被添加到搜索字符串中, 并再次执行搜索.
  会将第二次搜索返回的行作为最终的结果.
  `IN NATURAL LANGUAGE MODE WITH QUERY EXPANSION` 或 `WITH QUERY EXPANSION` 修饰符
  指定一个查询扩展搜索.

## 简单示例

```sql
CREATE TABLE articles (
  id INT UNSIGNED AUTO_INCREMENT NOT NULL PRIMARY KEY,
  title VARCHAR(200),
  body TEXT,
  FULLTEXT (title,body)
) ENGINE=InnoDB;
```

这会在 (title,body) 上创建全文索引, 算是一个复合的全文索引.

```sql
INSERT INTO articles (title,body) VALUES
('MySQL Tutorial','DBMS stands for DataBase ...'),
('How To Use MySQL Well','After you went through a ...'),
('Optimizing MySQL','In this tutorial we will show ...'),
('1001 MySQL Tricks','1. Never run mysqld as root. 2. ...'),
('MySQL vs. YourSQL','In the following database comparison ...'),
('MySQL Security','When configured properly, MySQL ...');
```

插入测试数据.

```sql
SELECT * FROM articles
WHERE MATCH (title,body)
AGAINST ('database' IN NATURAL LANGUAGE MODE);
```

执行搜索, 查找 title 或 body 出现 `database` 的行.

## 实现细节与中文搜索

默认情况下, 搜索是以 **不区分大小写** 的方式执行的.
要区分大小写, 必须对索引列使用区分大小写或二进制的排序规则.
比如, 使用 `utf8mb4` 字符集的列可以使用 `utf8mb4_0900_as_cs` 或 `utf8mb4_bin` 排序规则.

包含在双引号中的短语, 将按字面值进行匹配.
全文索引会将短语分解为单词, 并在 FULLTEXT 索引中搜索单词. 非单词字符不要求完全匹配.
搜索仅要求匹配项包含与短语完全相同的单词, 并且顺序相同.
例如, 短语 "test phrase" 与文本 "test, phrase" 就匹配.

MYSQL FULLTEXT 实现会将任何 **字符序列(字母, 数字, 下划线)** 视作一个单词.
这个字符序列中可以包含单引号(\`), 但在一行中不能多于一个.
所以 `aaa'bbb` 是一个单词, 但 `aaa''bbb` 就不是一个单词.
同时 FULLTEXT 会将单词开头或结尾的单引号删除.

内置的解析器通过查找某些定界符(delimiter characters)来确定单词的开头或结果.
常见的定界符有空格, 逗号, 句号. 对于单词之间没有定界符分隔的语言, 例如中文,
内置的解析器就无法确定单词的开始或结束位置.

对于这种情况, 为了将这些语言中的单词添加到全文索引中, 有两种方式处理.

1. 对文本进行预处理, 使得单词之间存在任意的定界符. 通常是分词.
2. 创建 FULLTEXT 时使用 ngram 解析器(中日韩文适用)或 MeCab 解析器(日文适用).

在全文索引中, 某些单词会被忽略掉:

- 任何太短的单词. 全文索引的默认最小单词长度在 InnoDB 是三个字符, 在 MyISAM 中是四个字符.
  这个特性对 ngram 解析器不适用, ngram 解析器中单词长度由 `ngram_token_size` 选项确定.
- 在 stopwords 中的单词将被忽略.

集合和查询中的每个正确的单词都被根据其在集合或查询中的重要性进行加权. 类似于 TF-IDF.
在多数文档中出现的单词权重比较低. 在所有文档中较少出现的单词权重较高.

## 布尔搜索

使用 `` 修饰符可以进行布尔型的全文搜索. 某些在搜索字符串中单词的开头或结尾的字符具有特殊的意义,

```sql
SELECT * FROM articles WHERE MATCH (title,body)
AGAINST ('+MySQL -YourSQL' IN BOOLEAN MODE);
```

上面这个例子中, `+MySQL` 表示 `MySQL` 必须存在, `-YourSQL` 表示 `YourSQL` 不能存在.
综合起来就是, 在 title 或 body 中存在 `MySQL`, 但没有 `YourSQL` 的行.

布尔型的全文搜索支持以下运算符:

- `+` 前导或尾随的加号表示单词必须出现在返回的每一行中. InnoDB 仅支持前导的加号.
- `-` 前导或尾随的减号表示单词不能再返回的任何行中出现. InnoDB 仅支持前导的减号.
  `-` 仅能排除其他条件搜索到的行. 仅有 `-` 会返回空的结果.
- (没有操作符) 默认情况, 该单词为可选, 但包含该单词的行评分较高.
- `@distance` 仅用于 InnoDB 表. 测试两个或多个单词出现的距离是否在 distance 的值之内.
  distance 的单位是单词的个数. 使用双引号指定要比较的单词.
  `MATCH(col1) AGAINST('"word1 word2 word3" @8' IN BOOLEAN MODE)`
  word1, word2, word3 之间的距离(单词数)在 8 之内.
- `>` `<` 用于修改单词对所在行的相关性的贡献度.
- `()` 括号用于将单词分组为子表达式. 括号可以嵌套.
- `~` 前导的 ~ 表示否定运算符, 使得单词对行的相关性的贡献度为负数.
  这对于标记噪音字符很有用. 包含这类单词的行的相关性低于其他行, 但不完全排除.
- `*` 星号用作截断(通配符)运算符.
  当一个单词使用截断符指定时, 即使它太短或者在 stopwords 列表中, 也不会被忽略.
- `"` 包含在引号中的短语, 将按字面值进行匹配.

下面是几个例子:

- `'apple banana'` 返回至少包含其中一个单词的行.
- `'+apple ~macintosh'` 返回包含单词 apple 的行, 如果行中有 macintosh, 它的评级会更低.
- `'+apple +(>turnover <strudel)'`
  返回行中同时有 apple 和 turnover, 或者同时有 apple 和 strudel 的行.
  apple 和 turnover 的行, 比 apple 和 strudel 的行优先级高.
- `apple*` 返回单词以 apple 开头的行, 例如 “apple”, “apples”, “applesauce”, “applet”.
- `'"some words"'`
  返回行中明确包含短语 "some words" 的行, 可以是 “some words of wisdom” 但不能是 “some noise words”.

InnoDB 上的全文搜索是建立在 Sphinx 全文搜索引擎上的, 算法是基于 BM25 和 TF-IDF 排名算法.

相关性排名的计算方式

```
# 计算 IDF 逆文档频率
${IDF} = log10( ${total_records} / ${matching_records} )
# 文档多次包含一个单词时
${TF} * ${IDF}

# 单个单词的相关性, 不知道为什么要多乘次 ${IDF}
${rank} = ${TF} * ${IDF} * ${IDF}

# 多个单词的相关性
${rank} = ${TF} * ${IDF} * ${IDF} + ${TF} * ${IDF} * ${IDF}
```

## ngram 解析器

一个 ngram 是 n 个字符的连续序列. ngram 解析器将文本序列标记为连续的 n 字符序列.
对于文本序列 "abcd" 可以使用 ngram 解析器标记为不同的 n 版本:

```
n=1: 'a', 'b', 'c', 'd'
n=2: 'ab', 'bc', 'cd'
n=3: 'abc', 'bcd'
n=4: 'abcd'
```

ngram 全文解析器是内置的服务器插件, 启动服务器时会自动加载该插件.

### 配置 token size

ngram 解析器默认的 ngram token size 是 2(bigram).

使用
[`ngram_token_size`][ngram_token_size]
配置选项设置 token size, 可配置的最小值是 1, 最大值是 10.
通常将 `ngram_token_size` 配置为你想要搜索的最大 token 的长度.

可以在启动的时候设置:

```bash
mysqld --ngram_token_size=2
```

或者在配置文件中设置:

```ini
[mysqld]
ngram_token_size=2
```

查看当前配置:

```sql
show variables like '%ngram%';
```

### 使用 ngram 解析器创建 FULLTEXT 索引

```sql
CREATE TABLE articles2 (
  id INT UNSIGNED AUTO_INCREMENT NOT NULL PRIMARY KEY,
  title VARCHAR(200),
  body TEXT,
  FULLTEXT (title,body) WITH PARSER ngram
) ENGINE=InnoDB CHARACTER SET utf8mb4;
```

插入些测试数据:

```sql
SET NAMES utf8mb4;
INSERT INTO articles2 (title,body) VALUES
    ('数据库管理','在本教程中我将向你展示如何管理数据库'),
    ('数据库应用开发','学习开发数据库应用程序');
```

在表 [INFORMATION_SCHEMA.INNODB_FT_INDEX_CACHE][information_schema] 中查看 tokenized data:

```sql
SET GLOBAL innodb_ft_aux_table="test/articles2";
SELECT * FROM INFORMATION_SCHEMA.INNODB_FT_INDEX_CACHE ORDER BY doc_id, position;
```

搜索查询:

```sql
select * from articles2 where match(title,body) against ("开发");
```

修改已存在的表:

```sql
CREATE TABLE articles (
  id INT UNSIGNED AUTO_INCREMENT NOT NULL PRIMARY KEY,
  title VARCHAR(200),
  body TEXT
) ENGINE=InnoDB CHARACTER SET utf8;

ALTER TABLE articles ADD FULLTEXT INDEX ft_index (title,body) WITH PARSER ngram;

# Or:

CREATE FULLTEXT INDEX ft_index ON articles (title,body) WITH PARSER ngram;
```

### 其他细节

ngram 解析器会在解析时消除空格.

对于 stopwords, ngram 解析器会排除包含 stopwords 的 token.
比如 stopwords 包含逗号, "a,b" 会在 n=2 时解析为 "a," 和 ",b", 这两个 token 都会被排除.
stopwords 中长度大于 `ngram_token_size` 的单词, 将会被忽略.

对于自然语言搜索, 搜索字符串将会被转换为 ngram token 的并集,
比如 "abc" 会被转换为 "ab" 和 "bc".

对于布尔搜索, 搜索字符串将会被转换为 ngram phrase search.
比如 "abc" 会被转换为 ""ab bc"".
给定两个文档, 一个包含 'ab', 另一个包含 'abc', 搜索短语 ""ab bc"" 只会匹配包含 'abc' 的文档.

通配符搜索可能返回意外的结果. 以下行为适用:

- 如果一个通配符搜索的前缀项小于 ngram_token_size, 查询会返回所有 ngram token 以前缀开始的行.
  比如 n=2 时, 搜索 "a\*" 会返回所有 ngram token 中以 "a" 开头的行.
- 如果一个通配符搜索的前缀项大于 ngram_token_size, 前缀项会被转换为 ngram phrase, 并且忽略通配符.
  比如 n=2 时, "abc\*" 通配符搜索会被转换为 "ab bc".

phrase search 会被转换为 ngram phrase search.
比如搜索短语 "abc" 会被转换为 "ab bc", 返回包含 "abc" 或 "ab bc" 的文档.
搜索短语 "abc def" 会被转换为 "ab bc de ef", 返回包含 "abc def" 或 "ab bc de ef" 的文档.
文档包含 "abcdef" 的却不会返回.

最后这个真是迷惑行为大赏, 还是看个例子好了.

```sql
INSERT INTO articles2 (title,body) VALUES ('abcdef','在本教程中我将向你展示如何管理数据库');
```

普通搜索是有数据的:

```sql
select * from articles2 where match(title,body) against ('abc def' IN BOOLEAN MODE);
```

短语搜索没有数据, 注意用 双引号 引起来的表示短语:

```sql
select * from articles2 where match(title,body) against ('"abc def"' IN BOOLEAN MODE);
```

然后切换到自然语言搜索, 两种方式都是有结果的:

```sql
select * from articles2 where match(title,body) against ('abc def');
select * from articles2 where match(title,body) against ('"abc def"');
```

## 参考

- [官方文档 fulltext-search](https://dev.mysql.com/doc/refman/8.0/en/fulltext-search.html)

<!-- 定义链接 -->

[ngram_token_size]: https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_ngram_token_size
[information_schema]: https://dev.mysql.com/doc/refman/8.0/en/innodb-ft-index-cache-table.html
