Explain 详解（下）
=============

标签： MySQL 是怎样运行的

* * *

执行计划输出中各列详解
-----------

本章紧接着上一节的内容，继续唠叨`EXPLAIN`语句输出的各个列的意思。

### Extra

顾名思义，`Extra`列是用来说明一些额外信息的，我们可以通过这些额外信息来更准确的理解`MySQL`到底将如何执行给定的查询语句。`MySQL`提供的额外信息有好几十个，我们就不一个一个介绍了（都介绍了感觉我们的文章就跟文档差不多了～），所以我们只挑一些平时常见的或者比较重要的额外信息介绍给大家哈。

*   `No tables used`
    
    当查询语句的没有`FROM`子句时将会提示该额外信息，比如：
    
        mysql> EXPLAIN SELECT 1;
        +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
        | id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra          |
        +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
        |  1 | SIMPLE      | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | No tables used |
        +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
        1 row in set, 1 warning (0.00 sec)
        
    
*   `Impossible WHERE`
    
    查询语句的`WHERE`子句永远为`FALSE`时将会提示该额外信息，比方说：
    
        mysql> EXPLAIN SELECT * FROM s1 WHERE 1 != 1;
        +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------------+
        | id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra            |
        +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------------+
        |  1 | SIMPLE      | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | Impossible WHERE |
        +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------------+
        1 row in set, 1 warning (0.01 sec)
        
    
*   `No matching min/max row`
    
    当查询列表处有`MIN`或者`MAX`聚集函数，但是并没有符合`WHERE`子句中的搜索条件的记录时，将会提示该额外信息，比方说：
    
        mysql> EXPLAIN SELECT MIN(key1) FROM s1 WHERE key1 = 'abcdefg';
        +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------------------+
        | id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                   |
        +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------------------+
        |  1 | SIMPLE      | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | No matching min/max row |
        +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------------------+
        1 row in set, 1 warning (0.00 sec)
        
    
*   `Using index`
    
    当我们的查询列表以及搜索条件中只包含属于某个索引的列，也就是在可以使用索引覆盖的情况下，在`Extra`列将会提示该额外信息。比方说下边这个查询中只需要用到`idx_key1`而不需要回表操作：
    
        mysql> EXPLAIN SELECT key1 FROM s1 WHERE key1 = 'a';
        +----+-------------+-------+------------+------+---------------+----------+---------+-------+------+----------+-------------+
        | id | select_type | table | partitions | type | possible_keys | key      | key_len | ref   | rows | filtered | Extra       |
        +----+-------------+-------+------------+------+---------------+----------+---------+-------+------+----------+-------------+
        |  1 | SIMPLE      | s1    | NULL       | ref  | idx_key1      | idx_key1 | 303     | const |    8 |   100.00 | Using index |
        +----+-------------+-------+------------+------+---------------+----------+---------+-------+------+----------+-------------+
        1 row in set, 1 warning (0.00 sec)
        
    
*   `Using index condition`
    
    有些搜索条件中虽然出现了索引列，但却不能使用到索引，比如下边这个查询：
    
        SELECT * FROM s1 WHERE key1 > 'z' AND key1 LIKE '%a';
        
    
    其中的`key1 > 'z'`可以使用到索引，但是`key1 LIKE '%a'`却无法使用到索引，在以前版本的`MySQL`中，是按照下边步骤来执行这个查询的：
    
    *   先根据`key1 > 'z'`这个条件，从二级索引`idx_key1`中获取到对应的二级索引记录。
        
    *   根据上一步骤得到的二级索引记录中的主键值进行回表，找到完整的用户记录再检测该记录是否符合`key1 LIKE '%a'`这个条件，将符合条件的记录加入到最后的结果集。
        
    
    但是虽然`key1 LIKE '%a'`不能组成范围区间参与`range`访问方法的执行，但这个条件毕竟只涉及到了`key1`列，所以设计`MySQL`的大叔把上边的步骤改进了一下：
    
    *   先根据`key1 > 'z'`这个条件，定位到二级索引`idx_key1`中对应的二级索引记录。
        
    *   对于指定的二级索引记录，先不着急回表，而是先检测一下该记录是否满足`key1 LIKE '%a'`这个条件，如果这个条件不满足，则该二级索引记录压根儿就没必要回表。
        
    *   对于满足`key1 LIKE '%a'`这个条件的二级索引记录执行回表操作。
        
    
    我们说回表操作其实是一个随机`IO`，比较耗时，所以上述修改虽然只改进了一点点，但是可以省去好多回表操作的成本。设计`MySQL`的大叔们把他们的这个改进称之为`索引条件下推`（英文名：`Index Condition Pushdown`）。
    
    如果在查询语句的执行过程中将要使用`索引条件下推`这个特性，在`Extra`列中将会显示`Using index condition`，比如这样：
    
        mysql> EXPLAIN SELECT * FROM s1 WHERE key1 > 'z' AND key1 LIKE '%b';
          +----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-----------------------+
          | id | select_type | table | partitions | type  | possible_keys | key      | key_len | ref  | rows | filtered | Extra                 |
          +----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-----------------------+
          |  1 | SIMPLE      | s1    | NULL       | range | idx_key1      | idx_key1 | 303     | NULL |  266 |   100.00 | Using index condition |
          +----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-----------------------+
          1 row in set, 1 warning (0.01 sec)
        
    
*   `Using where`
    
    当我们使用全表扫描来执行对某个表的查询，并且该语句的`WHERE`子句中有针对该表的搜索条件时，在`Extra`列中会提示上述额外信息。比如下边这个查询：
    
        mysql> EXPLAIN SELECT * FROM s1 WHERE common_field = 'a';
        +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
        | id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
        +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
        |  1 | SIMPLE      | s1    | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 9688 |    10.00 | Using where |
        +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
        1 row in set, 1 warning (0.01 sec)
        
    
    当使用索引访问来执行对某个表的查询，并且该语句的`WHERE`子句中有除了该索引包含的列之外的其他搜索条件时，在`Extra`列中也会提示上述额外信息。比如下边这个查询虽然使用`idx_key1`索引执行查询，但是搜索条件中除了包含`key1`的搜索条件`key1 = 'a'`，还有包含`common_field`的搜索条件，所以`Extra`列会显示`Using where`的提示：
    
        mysql> EXPLAIN SELECT * FROM s1 WHERE key1 = 'a' AND common_field = 'a';
        +----+-------------+-------+------------+------+---------------+----------+---------+-------+------+----------+-------------+
        | id | select_type | table | partitions | type | possible_keys | key      | key_len | ref   | rows | filtered | Extra       |
        +----+-------------+-------+------------+------+---------------+----------+---------+-------+------+----------+-------------+
        |  1 | SIMPLE      | s1    | NULL       | ref  | idx_key1      | idx_key1 | 303     | const |    8 |    10.00 | Using where |
        +----+-------------+-------+------------+------+---------------+----------+---------+-------+------+----------+-------------+
        1 row in set, 1 warning (0.00 sec)
        
    
*   `Using join buffer (Block Nested Loop)`
    
    在连接查询执行过程中，当被驱动表不能有效的利用索引加快访问速度，`MySQL`一般会为其分配一块名叫`join buffer`的内存块来加快查询速度，也就是我们所讲的`基于块的嵌套循环算法`，比如下边这个查询语句：
    
        mysql> EXPLAIN SELECT * FROM s1 INNER JOIN s2 ON s1.common_field = s2.common_field;
        +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
        | id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                                              |
        +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
        |  1 | SIMPLE      | s1    | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 9688 |   100.00 | NULL                                               |
        |  1 | SIMPLE      | s2    | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 9954 |    10.00 | Using where; Using join buffer (Block Nested Loop) |
        +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
        2 rows in set, 1 warning (0.03 sec)
        
    
    可以在对`s2`表的执行计划的`Extra`列显示了两个提示：
    
    *   `Using join buffer (Block Nested Loop)`：这是因为对表`s2`的访问不能有效利用索引，只好退而求其次，使用`join buffer`来减少对`s2`表的访问次数，从而提高性能。
        
    *   `Using where`：可以看到查询语句中有一个`s1.common_field = s2.common_field`条件，因为`s1`是驱动表，`s2`是被驱动表，所以在访问`s2`表时，`s1.common_field`的值已经确定下来了，所以实际上查询`s2`表的条件就是`s2.common_field = 一个常数`，所以提示了`Using where`额外信息。
        
*   `Not exists`
    
    当我们使用左（外）连接时，如果`WHERE`子句中包含要求被驱动表的某个列等于`NULL`值的搜索条件，而且那个列又是不允许存储`NULL`值的，那么在该表的执行计划的`Extra`列就会提示`Not exists`额外信息，比如这样：
    
        mysql> EXPLAIN SELECT * FROM s1 LEFT JOIN s2 ON s1.key1 = s2.key1 WHERE s2.id IS NULL;
        +----+-------------+-------+------------+------+---------------+----------+---------+-------------------+------+----------+-------------------------+
        | id | select_type | table | partitions | type | possible_keys | key      | key_len | ref               | rows | filtered | Extra                   |
        +----+-------------+-------+------------+------+---------------+----------+---------+-------------------+------+----------+-------------------------+
        |  1 | SIMPLE      | s1    | NULL       | ALL  | NULL          | NULL     | NULL    | NULL              | 9688 |   100.00 | NULL                    |
        |  1 | SIMPLE      | s2    | NULL       | ref  | idx_key1      | idx_key1 | 303     | xiaohaizi.s1.key1 |    1 |    10.00 | Using where; Not exists |
        +----+-------------+-------+------------+------+---------------+----------+---------+-------------------+------+----------+-------------------------+
        2 rows in set, 1 warning (0.00 sec)
        
    
    上述查询中`s1`表是驱动表，`s2`表是被驱动表，`s2.id`列是不允许存储`NULL`值的，而`WHERE`子句中又包含`s2.id IS NULL`的搜索条件，这意味着必定是驱动表的记录在被驱动表中找不到匹配`ON`子句条件的记录才会把该驱动表的记录加入到最终的结果集，所以对于某条驱动表中的记录来说，如果能在被驱动表中找到1条符合`ON`子句条件的记录，那么该驱动表的记录就不会被加入到最终的结果集，也就是说我们没有必要到被驱动表中找到全部符合ON子句条件的记录，这样可以稍微节省一点性能。
    
    > 小贴士： 右（外）连接可以被转换为左（外）连接，所以就不提右（外）连接的情况了。
    
*   `Using intersect(...)`、`Using union(...)`和`Using sort_union(...)`
    
    如果执行计划的`Extra`列出现了`Using intersect(...)`提示，说明准备使用`Intersect`索引合并的方式执行查询，括号中的`...`表示需要进行索引合并的索引名称；如果出现了`Using union(...)`提示，说明准备使用`Union`索引合并的方式执行查询；出现了`Using sort_union(...)`提示，说明准备使用`Sort-Union`索引合并的方式执行查询。比如这个查询的执行计划：
    
        mysql> EXPLAIN SELECT * FROM s1 WHERE key1 = 'a' AND key3 = 'a';
        +----+-------------+-------+------------+-------------+-------------------+-------------------+---------+------+------+----------+-------------------------------------------------+
        | id | select_type | table | partitions | type        | possible_keys     | key               | key_len | ref  | rows | filtered | Extra                                           |
        +----+-------------+-------+------------+-------------+-------------------+-------------------+---------+------+------+----------+-------------------------------------------------+
        |  1 | SIMPLE      | s1    | NULL       | index_merge | idx_key1,idx_key3 | idx_key3,idx_key1 | 303,303 | NULL |    1 |   100.00 | Using intersect(idx_key3,idx_key1); Using where |
        +----+-------------+-------+------------+-------------+-------------------+-------------------+---------+------+------+----------+-------------------------------------------------+
        1 row in set, 1 warning (0.01 sec)
        
    
    其中`Extra`列就显示了`Using intersect(idx_key3,idx_key1)`，表明`MySQL`即将使用`idx_key3`和`idx_key1`这两个索引进行`Intersect`索引合并的方式执行查询。
    
    > 小贴士： 剩下两种类型的索引合并的Extra列信息就不一一举例子了，自己写个查询瞅瞅呗～
    
*   `Zero limit`
    
    当我们的`LIMIT`子句的参数为`0`时，表示压根儿不打算从表中读出任何记录，将会提示该额外信息，比如这样：
    
        mysql> EXPLAIN SELECT * FROM s1 LIMIT 0;
        +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------+
        | id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra      |
        +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------+
        |  1 | SIMPLE      | NULL  | NULL       | NULL | NULL          | NULL | NULL    | NULL | NULL |     NULL | Zero limit |
        +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+------------+
        1 row in set, 1 warning (0.00 sec)
        
    
*   `Using filesort`
    
    有一些情况下对结果集中的记录进行排序是可以使用到索引的，比如下边这个查询：
    
        mysql> EXPLAIN SELECT * FROM s1 ORDER BY key1 LIMIT 10;
        +----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-------+
        | id | select_type | table | partitions | type  | possible_keys | key      | key_len | ref  | rows | filtered | Extra |
        +----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-------+
        |  1 | SIMPLE      | s1    | NULL       | index | NULL          | idx_key1 | 303     | NULL |   10 |   100.00 | NULL  |
        +----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-------+
        1 row in set, 1 warning (0.03 sec)
        
    
    这个查询语句可以利用`idx_key1`索引直接取出`key1`列的10条记录，然后再进行回表操作就好了。但是很多情况下排序操作无法使用到索引，只能在内存中（记录较少的时候）或者磁盘中（记录较多的时候）进行排序，设计`MySQL`的大叔把这种在内存中或者磁盘上进行排序的方式统称为文件排序（英文名：`filesort`）。如果某个查询需要使用文件排序的方式执行查询，就会在执行计划的`Extra`列中显示`Using filesort`提示，比如这样：
    
        mysql> EXPLAIN SELECT * FROM s1 ORDER BY common_field LIMIT 10;
        +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
        | id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra          |
        +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
        |  1 | SIMPLE      | s1    | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 9688 |   100.00 | Using filesort |
        +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------+
        1 row in set, 1 warning (0.00 sec)
        
    
    需要注意的是，如果查询中需要使用`filesort`的方式进行排序的记录非常多，那么这个过程是很耗费性能的，我们最好想办法将使用`文件排序`的执行方式改为使用索引进行排序。
    
*   `Using temporary`
    
    在许多查询的执行过程中，`MySQL`可能会借助临时表来完成一些功能，比如去重、排序之类的，比如我们在执行许多包含`DISTINCT`、`GROUP BY`、`UNION`等子句的查询过程中，如果不能有效利用索引来完成查询，`MySQL`很有可能寻求通过建立内部的临时表来执行查询。如果查询中使用到了内部的临时表，在执行计划的`Extra`列将会显示`Using temporary`提示，比方说这样：
    
        mysql> EXPLAIN SELECT DISTINCT common_field FROM s1;
        +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-----------------+
        | id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra           |
        +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-----------------+
        |  1 | SIMPLE      | s1    | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 9688 |   100.00 | Using temporary |
        +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-----------------+
        1 row in set, 1 warning (0.00 sec)
        
    
    再比如：
    
        mysql> EXPLAIN SELECT common_field, COUNT(*) AS amount FROM s1 GROUP BY common_field;
        +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+---------------------------------+
        | id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                           |
        +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+---------------------------------+
        |  1 | SIMPLE      | s1    | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 9688 |   100.00 | Using temporary; Using filesort |
        +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+---------------------------------+
        1 row in set, 1 warning (0.00 sec)
        
    
    不知道大家注意到没有，上述执行计划的`Extra`列不仅仅包含`Using temporary`提示，还包含`Using filesort`提示，可是我们的查询语句中明明没有写`ORDER BY`子句呀？这是因为`MySQL`会在包含`GROUP BY`子句的查询中默认添加上`ORDER BY`子句，也就是说上述查询其实和下边这个查询等价：
    
        EXPLAIN SELECT common_field, COUNT(*) AS amount FROM s1 GROUP BY common_field ORDER BY common_field;
        
    
    如果我们并不想为包含`GROUP BY`子句的查询进行排序，需要我们显式的写上`ORDER BY NULL`，就像这样：
    
        mysql> EXPLAIN SELECT common_field, COUNT(*) AS amount FROM s1 GROUP BY common_field ORDER BY NULL;
        +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-----------------+
        | id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra           |
        +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-----------------+
        |  1 | SIMPLE      | s1    | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 9688 |   100.00 | Using temporary |
        +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-----------------+
        1 row in set, 1 warning (0.00 sec)
        
    
    这回执行计划中就没有`Using filesort`的提示了，也就意味着执行查询时可以省去对记录进行文件排序的成本了。
    
    另外，执行计划中出现`Using temporary`并不是一个好的征兆，因为建立与维护临时表要付出很大成本的，所以我们最好能使用索引来替代掉使用临时表，比方说下边这个包含`GROUP BY`子句的查询就不需要使用临时表：
    
        mysql> EXPLAIN SELECT key1, COUNT(*) AS amount FROM s1 GROUP BY key1;
        +----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-------------+
        | id | select_type | table | partitions | type  | possible_keys | key      | key_len | ref  | rows | filtered | Extra       |
        +----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-------------+
        |  1 | SIMPLE      | s1    | NULL       | index | idx_key1      | idx_key1 | 303     | NULL | 9688 |   100.00 | Using index |
        +----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-------------+
        1 row in set, 1 warning (0.00 sec)
        
    
    从`Extra`的`Using index`的提示里我们可以看出，上述查询只需要扫描`idx_key1`索引就可以搞定了，不再需要临时表了。
    
*   `Start temporary, End temporary`
    
    我们前边唠叨子查询的时候说过，查询优化器会优先尝试将`IN`子查询转换成`semi-join`，而`semi-join`又有好多种执行策略，当执行策略为`DuplicateWeedout`时，也就是通过建立临时表来实现为外层查询中的记录进行去重操作时，驱动表查询执行计划的`Extra`列将显示`Start temporary`提示，被驱动表查询执行计划的`Extra`列将显示`End temporary`提示，就是这样：
    
        mysql> EXPLAIN SELECT * FROM s1 WHERE key1 IN (SELECT key3 FROM s2 WHERE common_field = 'a');
        +----+-------------+-------+------------+------+---------------+----------+---------+-------------------+------+----------+------------------------------+
        | id | select_type | table | partitions | type | possible_keys | key      | key_len | ref               | rows | filtered | Extra                        |
        +----+-------------+-------+------------+------+---------------+----------+---------+-------------------+------+----------+------------------------------+
        |  1 | SIMPLE      | s2    | NULL       | ALL  | idx_key3      | NULL     | NULL    | NULL              | 9954 |    10.00 | Using where; Start temporary |
        |  1 | SIMPLE      | s1    | NULL       | ref  | idx_key1      | idx_key1 | 303     | xiaohaizi.s2.key3 |    1 |   100.00 | End temporary                |
        +----+-------------+-------+------------+------+---------------+----------+---------+-------------------+------+----------+------------------------------+
        2 rows in set, 1 warning (0.00 sec)
        
    
*   `LooseScan`
    
    在将`In`子查询转为`semi-join`时，如果采用的是`LooseScan`执行策略，则在驱动表执行计划的`Extra`列就是显示`LooseScan`提示，比如这样：
    
        mysql> EXPLAIN SELECT * FROM s1 WHERE key3 IN (SELECT key1 FROM s2 WHERE key1 > 'z');
        +----+-------------+-------+------------+-------+---------------+----------+---------+-------------------+------+----------+-------------------------------------+
        | id | select_type | table | partitions | type  | possible_keys | key      | key_len | ref               | rows | filtered | Extra                               |
        +----+-------------+-------+------------+-------+---------------+----------+---------+-------------------+------+----------+-------------------------------------+
        |  1 | SIMPLE      | s2    | NULL       | range | idx_key1      | idx_key1 | 303     | NULL              |  270 |   100.00 | Using where; Using index; LooseScan |
        |  1 | SIMPLE      | s1    | NULL       | ref   | idx_key3      | idx_key3 | 303     | xiaohaizi.s2.key1 |    1 |   100.00 | NULL                                |
        +----+-------------+-------+------------+-------+---------------+----------+---------+-------------------+------+----------+-------------------------------------+
        2 rows in set, 1 warning (0.01 sec)
        
    
*   `FirstMatch(tbl_name)`
    
    在将`In`子查询转为`semi-join`时，如果采用的是`FirstMatch`执行策略，则在被驱动表执行计划的`Extra`列就是显示`FirstMatch(tbl_name)`提示，比如这样：
    
        mysql> EXPLAIN SELECT * FROM s1 WHERE common_field IN (SELECT key1 FROM s2 where s1.key3 = s2.key3);
        +----+-------------+-------+------------+------+-------------------+----------+---------+-------------------+------+----------+-----------------------------+
        | id | select_type | table | partitions | type | possible_keys     | key      | key_len | ref               | rows | filtered | Extra                       |
        +----+-------------+-------+------------+------+-------------------+----------+---------+-------------------+------+----------+-----------------------------+
        |  1 | SIMPLE      | s1    | NULL       | ALL  | idx_key3          | NULL     | NULL    | NULL              | 9688 |   100.00 | Using where                 |
        |  1 | SIMPLE      | s2    | NULL       | ref  | idx_key1,idx_key3 | idx_key3 | 303     | xiaohaizi.s1.key3 |    1 |     4.87 | Using where; FirstMatch(s1) |
        +----+-------------+-------+------------+------+-------------------+----------+---------+-------------------+------+----------+-----------------------------+
        2 rows in set, 2 warnings (0.00 sec)
        
    

Json格式的执行计划
-----------

我们上边介绍的`EXPLAIN`语句输出中缺少了一个衡量执行计划好坏的重要属性 —— 成本。不过设计`MySQL`的大叔贴心的为我们提供了一种查看某个执行计划花费的成本的方式：

*   在`EXPLAIN`单词和真正的查询语句中间加上`FORMAT=JSON`。

这样我们就可以得到一个`json`格式的执行计划，里边儿包含该计划花费的成本，比如这样：

    mysql> EXPLAIN FORMAT=JSON SELECT * FROM s1 INNER JOIN s2 ON s1.key1 = s2.key2 WHERE s1.common_field = 'a'\G
    *************************** 1. row ***************************
    
    EXPLAIN: {
      "query_block": {
        "select_id": 1,     # 整个查询语句只有1个SELECT关键字，该关键字对应的id号为1
        "cost_info": {
          "query_cost": "3197.16"   # 整个查询的执行成本预计为3197.16
        },
        "nested_loop": [    # 几个表之间采用嵌套循环连接算法执行
        
        # 以下是参与嵌套循环连接算法的各个表的信息
          {
            "table": {
              "table_name": "s1",   # s1表是驱动表
              "access_type": "ALL",     # 访问方法为ALL，意味着使用全表扫描访问
              "possible_keys": [    # 可能使用的索引
                "idx_key1"
              ],
              "rows_examined_per_scan": 9688,   # 查询一次s1表大致需要扫描9688条记录
              "rows_produced_per_join": 968,    # 驱动表s1的扇出是968
              "filtered": "10.00",  # condition filtering代表的百分比
              "cost_info": {
                "read_cost": "1840.84",     # 稍后解释
                "eval_cost": "193.76",      # 稍后解释
                "prefix_cost": "2034.60",   # 单次查询s1表总共的成本
                "data_read_per_join": "1M"  # 读取的数据量
              },
              "used_columns": [     # 执行查询中涉及到的列
                "id",
                "key1",
                "key2",
                "key3",
                "key_part1",
                "key_part2",
                "key_part3",
                "common_field"
              ],
              
              # 对s1表访问时针对单表查询的条件
              "attached_condition": "((`xiaohaizi`.`s1`.`common_field` = 'a') and (`xiaohaizi`.`s1`.`key1` is not null))"
            }
          },
          {
            "table": {
              "table_name": "s2",   # s2表是被驱动表
              "access_type": "ref",     # 访问方法为ref，意味着使用索引等值匹配的方式访问
              "possible_keys": [    # 可能使用的索引
                "idx_key2"
              ],
              "key": "idx_key2",    # 实际使用的索引
              "used_key_parts": [   # 使用到的索引列
                "key2"
              ],
              "key_length": "5",    # key_len
              "ref": [      # 与key2列进行等值匹配的对象
                "xiaohaizi.s1.key1"
              ],
              "rows_examined_per_scan": 1,  # 查询一次s2表大致需要扫描1条记录
              "rows_produced_per_join": 968,    # 被驱动表s2的扇出是968（由于后边没有多余的表进行连接，所以这个值也没啥用）
              "filtered": "100.00",     # condition filtering代表的百分比
              
              # s2表使用索引进行查询的搜索条件
              "index_condition": "(`xiaohaizi`.`s1`.`key1` = `xiaohaizi`.`s2`.`key2`)",
              "cost_info": {
                "read_cost": "968.80",      # 稍后解释
                "eval_cost": "193.76",      # 稍后解释
                "prefix_cost": "3197.16",   # 单次查询s1、多次查询s2表总共的成本
                "data_read_per_join": "1M"  # 读取的数据量
              },
              "used_columns": [     # 执行查询中涉及到的列
                "id",
                "key1",
                "key2",
                "key3",
                "key_part1",
                "key_part2",
                "key_part3",
                "common_field"
              ]
            }
          }
        ]
      }
    }
    1 row in set, 2 warnings (0.00 sec)
    

我们使用`#`后边跟随注释的形式为大家解释了`EXPLAIN FORMAT=JSON`语句的输出内容，但是大家可能有疑问`"cost_info"`里边的成本看着怪怪的，它们是怎么计算出来的？先看`s1`表的`"cost_info"`部分：

    "cost_info": {
        "read_cost": "1840.84",
        "eval_cost": "193.76",
        "prefix_cost": "2034.60",
        "data_read_per_join": "1M"
    }
    

*   `read_cost`是由下边这两部分组成的：
    
    *   `IO`成本
    *   检测`rows × (1 - filter)`条记录的`CPU`成本
    
    > 小贴士： rows和filter都是我们前边介绍执行计划的输出列，在JSON格式的执行计划中，rows相当于rows\_examined\_per_scan，filtered名称不变。
    
*   `eval_cost`是这样计算的：
    
    检测 `rows × filter`条记录的成本。
    
*   `prefix_cost`就是单独查询`s1`表的成本，也就是：
    
    `read_cost + eval_cost`
    
*   `data_read_per_join`表示在此次查询中需要读取的数据量，我们就不多唠叨这个了。
    

> 小贴士： 大家其实没必要关注MySQL为啥使用这么古怪的方式计算出read\_cost和eval\_cost，关注prefix_cost是查询s1表的成本就好了。

对于`s2`表的`"cost_info"`部分是这样的：

    "cost_info": {
        "read_cost": "968.80",
        "eval_cost": "193.76",
        "prefix_cost": "3197.16",
        "data_read_per_join": "1M"
    }
    optimizer trace 表的神奇功效
======================

标签： MySQL 是怎样运行的

* * *

对于`MySQL 5.6`以及之前的版本来说，查询优化器就像是一个黑盒子一样，你只能通过`EXPLAIN`语句查看到最后优化器决定使用的执行计划，却无法知道它为什么做这个决策。这对于一部分喜欢刨根问底的小伙伴来说简直是灾难：“我就觉得使用其他的执行方案比`EXPLAIN`输出的这种方案强，凭什么优化器做的决定和我想的不一样呢？”

在`MySQL 5.6`以及之后的版本中，设计`MySQL`的大叔贴心的为这部分小伙伴提出了一个`optimizer trace`的功能，这个功能可以让我们方便的查看优化器生成执行计划的整个过程，这个功能的开启与关闭由系统变量`optimizer_trace`决定，我们看一下：

    mysql> SHOW VARIABLES LIKE 'optimizer_trace';
    +-----------------+--------------------------+
    | Variable_name   | Value                    |
    +-----------------+--------------------------+
    | optimizer_trace | enabled=off,one_line=off |
    +-----------------+--------------------------+
    1 row in set (0.02 sec)
    

可以看到`enabled`值为`off`，表明这个功能默认是关闭的。

> 小贴士： one_line的值是控制输出格式的，如果为on那么所有输出都将在一行中展示，不适合人阅读，所以我们就保持其默认值为off吧。

如果想打开这个功能，必须首先把`enabled`的值改为`on`，就像这样：

    mysql> SET optimizer_trace="enabled=on";
    Query OK, 0 rows affected (0.00 sec)
    

然后我们就可以输入我们想要查看优化过程的查询语句，当该查询语句执行完成后，就可以到`information_schema`数据库下的`OPTIMIZER_TRACE`表中查看完整的优化过程。这个`OPTIMIZER_TRACE`表有4个列，分别是：

*   `QUERY`：表示我们的查询语句。
    
*   `TRACE`：表示优化过程的JSON格式文本。
    
*   `MISSING_BYTES_BEYOND_MAX_MEM_SIZE`：由于优化过程可能会输出很多，如果超过某个限制时，多余的文本将不会被显示，这个字段展示了被忽略的文本字节数。
    
*   `INSUFFICIENT_PRIVILEGES`：表示是否没有权限查看优化过程，默认值是0，只有某些特殊情况下才会是`1`，我们暂时不关心这个字段的值。
    

完整的使用`optimizer trace`功能的步骤总结如下：

    # 1. 打开optimizer trace功能 (默认情况下它是关闭的):
    SET optimizer_trace="enabled=on";
    
    # 2. 这里输入你自己的查询语句
    SELECT ...; 
    
    # 3. 从OPTIMIZER_TRACE表中查看上一个查询的优化过程
    SELECT * FROM information_schema.OPTIMIZER_TRACE;
    
    # 4. 可能你还要观察其他语句执行的优化过程，重复上边的第2、3步
    ...
    
    # 5. 当你停止查看语句的优化过程时，把optimizer trace功能关闭
    SET optimizer_trace="enabled=off";
    

现在我们有一个搜索条件比较多的查询语句，它的执行计划如下：

    mysql> EXPLAIN SELECT * FROM s1 WHERE
        ->     key1 > 'z' AND
        ->     key2 < 1000000 AND
        ->     key3 IN ('a', 'b', 'c') AND
        ->     common_field = 'abc';
    +----+-------------+-------+------------+-------+----------------------------+----------+---------+------+------+----------+------------------------------------+
    | id | select_type | table | partitions | type  | possible_keys              | key      | key_len | ref  | rows | filtered | Extra                              |
    +----+-------------+-------+------------+-------+----------------------------+----------+---------+------+------+----------+------------------------------------+
    |  1 | SIMPLE      | s1    | NULL       | range | idx_key2,idx_key1,idx_key3 | idx_key2 | 5       | NULL |   12 |     0.42 | Using index condition; Using where |
    +----+-------------+-------+------------+-------+----------------------------+----------+---------+------+------+----------+------------------------------------+
    1 row in set, 1 warning (0.00 sec)
    

可以看到该查询可能使用到的索引有3个，那么为什么优化器最终选择了`idx_key2`而不选择其他的索引或者直接全表扫描呢？这时候就可以通过`otpimzer trace`功能来查看优化器的具体工作过程：

    SET optimizer_trace="enabled=on";
    
    SELECT * FROM s1 WHERE 
        key1 > 'z' AND 
        key2 < 1000000 AND 
        key3 IN ('a', 'b', 'c') AND 
        common_field = 'abc';
        
    SELECT * FROM information_schema.OPTIMIZER_TRACE\G    
    

我们直接看一下通过查询`OPTIMIZER_TRACE`表得到的输出（我使用`#`后跟随注释的形式为大家解释了优化过程中的一些比较重要的点，大家重点关注一下）：

    *************************** 1. row ***************************
    # 分析的查询语句是什么
    QUERY: SELECT * FROM s1 WHERE
        key1 > 'z' AND
        key2 < 1000000 AND
        key3 IN ('a', 'b', 'c') AND
        common_field = 'abc'
    
    # 优化的具体过程
    TRACE: {
      "steps": [
        {
          "join_preparation": {     # prepare阶段
            "select#": 1,
            "steps": [
              {
                "IN_uses_bisection": true
              },
              {
                "expanded_query": "/* select#1 */ select `s1`.`id` AS `id`,`s1`.`key1` AS `key1`,`s1`.`key2` AS `key2`,`s1`.`key3` AS `key3`,`s1`.`key_part1` AS `key_part1`,`s1`.`key_part2` AS `key_part2`,`s1`.`key_part3` AS `key_part3`,`s1`.`common_field` AS `common_field` from `s1` where ((`s1`.`key1` > 'z') and (`s1`.`key2` < 1000000) and (`s1`.`key3` in ('a','b','c')) and (`s1`.`common_field` = 'abc'))"
              }
            ] /* steps */
          } /* join_preparation */
        },
        {
          "join_optimization": {    # optimize阶段
            "select#": 1,
            "steps": [
              {
                "condition_processing": {   # 处理搜索条件
                  "condition": "WHERE",
                  # 原始搜索条件
                  "original_condition": "((`s1`.`key1` > 'z') and (`s1`.`key2` < 1000000) and (`s1`.`key3` in ('a','b','c')) and (`s1`.`common_field` = 'abc'))",
                  "steps": [
                    {
                      # 等值传递转换
                      "transformation": "equality_propagation",
                      "resulting_condition": "((`s1`.`key1` > 'z') and (`s1`.`key2` < 1000000) and (`s1`.`key3` in ('a','b','c')) and (`s1`.`common_field` = 'abc'))"
                    },
                    {
                      # 常量传递转换    
                      "transformation": "constant_propagation",
                      "resulting_condition": "((`s1`.`key1` > 'z') and (`s1`.`key2` < 1000000) and (`s1`.`key3` in ('a','b','c')) and (`s1`.`common_field` = 'abc'))"
                    },
                    {
                      # 去除没用的条件
                      "transformation": "trivial_condition_removal",
                      "resulting_condition": "((`s1`.`key1` > 'z') and (`s1`.`key2` < 1000000) and (`s1`.`key3` in ('a','b','c')) and (`s1`.`common_field` = 'abc'))"
                    }
                  ] /* steps */
                } /* condition_processing */
              },
              {
                # 替换虚拟生成列
                "substitute_generated_columns": {
                } /* substitute_generated_columns */
              },
              {
                # 表的依赖信息
                "table_dependencies": [
                  {
                    "table": "`s1`",
                    "row_may_be_null": false,
                    "map_bit": 0,
                    "depends_on_map_bits": [
                    ] /* depends_on_map_bits */
                  }
                ] /* table_dependencies */
              },
              {
                "ref_optimizer_key_uses": [
                ] /* ref_optimizer_key_uses */
              },
              {
              
                # 预估不同单表访问方法的访问成本
                "rows_estimation": [
                  {
                    "table": "`s1`",
                    "range_analysis": {
                      "table_scan": {   # 全表扫描的行数以及成本
                        "rows": 9688,
                        "cost": 2036.7
                      } /* table_scan */,
                      
                      # 分析可能使用的索引
                      "potential_range_indexes": [
                        {
                          "index": "PRIMARY",   # 主键不可用
                          "usable": false,
                          "cause": "not_applicable"
                        },
                        {
                          "index": "idx_key2",  # idx_key2可能被使用
                          "usable": true,
                          "key_parts": [
                            "key2"
                          ] /* key_parts */
                        },
                        {
                          "index": "idx_key1",  # idx_key1可能被使用
                          "usable": true,
                          "key_parts": [
                            "key1",
                            "id"
                          ] /* key_parts */
                        },
                        {
                          "index": "idx_key3",  # idx_key3可能被使用
                          "usable": true,
                          "key_parts": [
                            "key3",
                            "id"
                          ] /* key_parts */
                        },
                        {
                          "index": "idx_key_part",  # idx_keypart不可用
                          "usable": false,
                          "cause": "not_applicable"
                        }
                      ] /* potential_range_indexes */,
                      "setup_range_conditions": [
                      ] /* setup_range_conditions */,
                      "group_index_range": {
                        "chosen": false,
                        "cause": "not_group_by_or_distinct"
                      } /* group_index_range */,
                      
                      # 分析各种可能使用的索引的成本
                      "analyzing_range_alternatives": {
                        "range_scan_alternatives": [
                          {
                            # 使用idx_key2的成本分析
                            "index": "idx_key2",
                            # 使用idx_key2的范围区间
                            "ranges": [
                              "NULL < key2 < 1000000"
                            ] /* ranges */,
                            "index_dives_for_eq_ranges": true,   # 是否使用index dive
                            "rowid_ordered": false,     # 使用该索引获取的记录是否按照主键排序
                            "using_mrr": false,     # 是否使用mrr
                            "index_only": false,    # 是否是索引覆盖访问
                            "rows": 12,     # 使用该索引获取的记录条数
                            "cost": 15.41,  # 使用该索引的成本
                            "chosen": true  # 是否选择该索引
                          },
                          {
                            # 使用idx_key1的成本分析
                            "index": "idx_key1",
                            # 使用idx_key1的范围区间
                            "ranges": [
                              "z < key1"
                            ] /* ranges */,
                            "index_dives_for_eq_ranges": true,   # 同上
                            "rowid_ordered": false,   # 同上
                            "using_mrr": false,   # 同上
                            "index_only": false,   # 同上
                            "rows": 266,   # 同上
                            "cost": 320.21,   # 同上
                            "chosen": false,   # 同上
                            "cause": "cost"   # 因为成本太大所以不选择该索引
                          },
                          {
                            # 使用idx_key3的成本分析
                            "index": "idx_key3",
                            # 使用idx_key3的范围区间
                            "ranges": [
                              "a <= key3 <= a",
                              "b <= key3 <= b",
                              "c <= key3 <= c"
                            ] /* ranges */,
                            "index_dives_for_eq_ranges": true,   # 同上
                            "rowid_ordered": false,   # 同上
                            "using_mrr": false,   # 同上
                            "index_only": false,   # 同上
                            "rows": 21,   # 同上
                            "cost": 28.21,   # 同上
                            "chosen": false,   # 同上
                            "cause": "cost"   # 同上
                          }
                        ] /* range_scan_alternatives */,
                        
                        # 分析使用索引合并的成本
                        "analyzing_roworder_intersect": {
                          "usable": false,
                          "cause": "too_few_roworder_scans"
                        } /* analyzing_roworder_intersect */
                      } /* analyzing_range_alternatives */,
                      
                      # 对于上述单表查询s1最优的访问方法
                      "chosen_range_access_summary": {
                        "range_access_plan": {
                          "type": "range_scan",
                          "index": "idx_key2",
                          "rows": 12,
                          "ranges": [
                            "NULL < key2 < 1000000"
                          ] /* ranges */
                        } /* range_access_plan */,
                        "rows_for_plan": 12,
                        "cost_for_plan": 15.41,
                        "chosen": true
                      } /* chosen_range_access_summary */
                    } /* range_analysis */
                  }
                ] /* rows_estimation */
              },
              {
                
                # 分析各种可能的执行计划
                #（对多表查询这可能有很多种不同的方案，单表查询的方案上边已经分析过了，直接选取idx_key2就好）
                "considered_execution_plans": [
                  {
                    "plan_prefix": [
                    ] /* plan_prefix */,
                    "table": "`s1`",
                    "best_access_path": {
                      "considered_access_paths": [
                        {
                          "rows_to_scan": 12,
                          "access_type": "range",
                          "range_details": {
                            "used_index": "idx_key2"
                          } /* range_details */,
                          "resulting_rows": 12,
                          "cost": 17.81,
                          "chosen": true
                        }
                      ] /* considered_access_paths */
                    } /* best_access_path */,
                    "condition_filtering_pct": 100,
                    "rows_for_plan": 12,
                    "cost_for_plan": 17.81,
                    "chosen": true
                  }
                ] /* considered_execution_plans */
              },
              {
                # 尝试给查询添加一些其他的查询条件
                "attaching_conditions_to_tables": {
                  "original_condition": "((`s1`.`key1` > 'z') and (`s1`.`key2` < 1000000) and (`s1`.`key3` in ('a','b','c')) and (`s1`.`common_field` = 'abc'))",
                  "attached_conditions_computation": [
                  ] /* attached_conditions_computation */,
                  "attached_conditions_summary": [
                    {
                      "table": "`s1`",
                      "attached": "((`s1`.`key1` > 'z') and (`s1`.`key2` < 1000000) and (`s1`.`key3` in ('a','b','c')) and (`s1`.`common_field` = 'abc'))"
                    }
                  ] /* attached_conditions_summary */
                } /* attaching_conditions_to_tables */
              },
              {
                # 再稍稍的改进一下执行计划
                "refine_plan": [
                  {
                    "table": "`s1`",
                    "pushed_index_condition": "(`s1`.`key2` < 1000000)",
                    "table_condition_attached": "((`s1`.`key1` > 'z') and (`s1`.`key3` in ('a','b','c')) and (`s1`.`common_field` = 'abc'))"
                  }
                ] /* refine_plan */
              }
            ] /* steps */
          } /* join_optimization */
        },
        {
          "join_execution": {    # execute阶段
            "select#": 1,
            "steps": [
            ] /* steps */
          } /* join_execution */
        }
      ] /* steps */
    }
    
    # 因优化过程文本太多而丢弃的文本字节大小，值为0时表示并没有丢弃
    MISSING_BYTES_BEYOND_MAX_MEM_SIZE: 0
    
    # 权限字段
    INSUFFICIENT_PRIVILEGES: 0
    
    1 row in set (0.00 sec)
    

大家看到这个输出的第一感觉就是这文本也太多了点儿吧，其实这只是优化器执行过程中的一小部分，设计`MySQL`的大叔可能会在之后的版本中添加更多的优化过程信息。不过杂乱之中其实还是蛮有规律的，优化过程大致分为了三个阶段：

*   `prepare`阶段
    
*   `optimize`阶段
    
*   `execute`阶段
    

我们所说的基于成本的优化主要集中在`optimize`阶段，对于单表查询来说，我们主要关注`optimize`阶段的`"rows_estimation"`这个过程，这个过程深入分析了对单表查询的各种执行方案的成本；对于多表连接查询来说，我们更多需要关注`"considered_execution_plans"`这个过程，这个过程里会写明各种不同的连接方式所对应的成本。反正优化器最终会选择成本最低的那种方案来作为最终的执行计划，也就是我们使用`EXPLAIN`语句所展现出的那种方案。

如果有小伙伴对使用`EXPLAIN`语句展示出的对某个查询的执行计划很不理解，大家可以尝试使用`optimizer trace`功能来详细了解每一种执行方案对应的成本，相信这个功能能让大家更深入的了解`MySQL`查询优化器。

由于`s2`表是被驱动表，所以可能被读取多次，这里的`read_cost`和`eval_cost`是访问多次`s2`表后累加起来的值，大家主要关注里边儿的`prefix_cost`的值代表的是整个连接查询预计的成本，也就是单次查询`s1`表和多次查询`s2`表后的成本的和，也就是：

    968.80 + 193.76 + 2034.60 = 3197.16
    

Extented EXPLAIN
----------------

最后，设计`MySQL`的大叔还为我们留了个彩蛋，在我们使用`EXPLAIN`语句查看了某个查询的执行计划后，紧接着还可以使用`SHOW WARNINGS`语句查看与这个查询的执行计划有关的一些扩展信息，比如这样：

    mysql> EXPLAIN SELECT s1.key1, s2.key1 FROM s1 LEFT JOIN s2 ON s1.key1 = s2.key1 WHERE s2.common_field IS NOT NULL;
    +----+-------------+-------+------------+------+---------------+----------+---------+-------------------+------+----------+-------------+
    | id | select_type | table | partitions | type | possible_keys | key      | key_len | ref               | rows | filtered | Extra       |
    +----+-------------+-------+------------+------+---------------+----------+---------+-------------------+------+----------+-------------+
    |  1 | SIMPLE      | s2    | NULL       | ALL  | idx_key1      | NULL     | NULL    | NULL              | 9954 |    90.00 | Using where |
    |  1 | SIMPLE      | s1    | NULL       | ref  | idx_key1      | idx_key1 | 303     | xiaohaizi.s2.key1 |    1 |   100.00 | Using index |
    +----+-------------+-------+------------+------+---------------+----------+---------+-------------------+------+----------+-------------+
    2 rows in set, 1 warning (0.00 sec)
    
    mysql> SHOW WARNINGS\G
    *************************** 1. row ***************************
      Level: Note
       Code: 1003
    Message: /* select#1 */ select `xiaohaizi`.`s1`.`key1` AS `key1`,`xiaohaizi`.`s2`.`key1` AS `key1` from `xiaohaizi`.`s1` join `xiaohaizi`.`s2` where ((`xiaohaizi`.`s1`.`key1` = `xiaohaizi`.`s2`.`key1`) and (`xiaohaizi`.`s2`.`common_field` is not null))
    1 row in set (0.00 sec)
    

大家可以看到`SHOW WARNINGS`展示出来的信息有三个字段，分别是`Level`、`Code`、`Message`。我们最常见的就是`Code`为`1003`的信息，当`Code`值为`1003`时，`Message`字段展示的信息类似于查询优化器将我们的查询语句重写后的语句。比如我们上边的查询本来是一个左（外）连接查询，但是有一个`s2.common_field IS NOT NULL`的条件，着就会导致查询优化器把左（外）连接查询优化为内连接查询，从`SHOW WARNINGS`的`Message`字段也可以看出来，原本的`LEFT JOIN`已经变成了`JOIN`。

但是大家一定要注意，我们说`Message`字段展示的信息类似于查询优化器将我们的查询语句重写后的语句，并不是等价于，也就是说`Message`字段展示的信息并不是标准的查询语句，在很多情况下并不能直接拿到黑框框中运行，它只能作为帮助我们理解查`MySQL`将如何执行查询语句的一个参考依据而已。