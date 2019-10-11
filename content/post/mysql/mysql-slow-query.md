---
title: "mysql slow query"
date: 2018-12-11 17:05:12
tag: 
  - mysql
  - SlowQuery
categories: [mysql]
---

[TOC]

## 优化 SQL

在得知哪些 SQL 是慢查询之后，我们就可以定位到具体的业务接口并针对性的进行优化了。

首先，你要看是否能在不改变现有业务逻辑的前提下改进查询的速度。一个典型的场景是，你需要查询数据库中是否存在符合某个条件的记录，返回一个布尔值来表示有或者没有，一般用于通知提醒。如果程序员在撰写接口时没把性能放在心上，那么他就有可能写出 `SELECT count(*) FROM tbl_xxx WHERE XXXX` 这样的查询，当数据量一大时（而且索引不恰当或没有索引）这个查询会相当之慢，但如果改成 `SELECT id FROM tbl_xxx WHERE XXXX LIMIT 1` 这样来查询，对速度的提升则是巨大的。这个例子并不是我凭空捏造的，最近在实际项目中我就看到了跟这个例子一模一样的场景。

能够找到上述的通过改变查询方式而又不改变业务逻辑的慢查询是幸运的，因为这些场景往往意味着只需重写 SQL 语句就能带来显著的性能提升，而且稍有经验的程序员在一开始就不会写出能够明显改良的查询语句。在绝大多数情况下，SQL 足够复杂而且难以做任何有价值的改动，这时就需要通过优化索引来提升效率了。

如何更好的创建数据库索引绝对是一门技术活，我也并不觉得简简单单就能厘得很清楚，很多时候还是得具体 SQL 具体分析，甚至多条 SQL 一起来分析。可以先读一读这篇美团点评技术团队的文章：[MySQL索引原理及慢查询优化](http://link.zhihu.com/?target=http%3A//tech.meituan.com/mysql-index.html)，更深入的了解则可以阅读[《高性能MySQL》](http://link.zhihu.com/?target=https%3A//book.douban.com/subject/23008813/)一书。引用一下美团点评技术团队文章中提到的几个原则：

> 1. 最左前缀匹配原则，非常重要的原则，mysql会一直向右匹配直到遇到范围查询(>、<、between、like)就停止匹配，比如a = 1 and b = 2 and c > 3 and d = 4 如果建立(a,b,c,d)顺序的索引，d是用不到索引的，如果建立(a,b,d,c)的索引则都可以用到，a,b,d的顺序可以任意调整；
> 2. =和in可以乱序，比如a = 1 and b = 2 and c = 3 建立(a,b,c)索引可以任意顺序，mysql的查询优化器会帮你优化成索引可以识别的形式；
> 3. 尽量选择区分度高的列作为索引,区分度的公式是count(distinct col)/count(*)，表示字段不重复的比例，比例越大我们扫描的记录数越少，唯一键的区分度是1，而一些状态、性别字段可能在大数据面前区分度就是0，那可能有人会问，这个比例有什么经验值吗？使用场景不同，这个值也很难确定，一般需要join的字段我们都要求是0.1以上，即平均1条扫描10条记录；
> 4. 索引列不能参与计算，保持列“干净”，比如from_unixtime(create_time) = ’2014-05-29’就不能使用到索引，原因很简单，b+树中存的都是数据表中的字段值，但进行检索时，需要把所有元素都应用函数才能比较，显然成本太大。所以语句应该写成create_time = unix_timestamp(’2014-05-29’);
> 5. 尽量的扩展索引，不要新建索引。比如表中已经有a的索引，现在要加(a,b)的索引，那么只需要修改原来的索引即可。

## 同步备份

> 如果是想利用现有的一台SLAVE来做添加一台SLAVE的话，不妨试下下面的方法：
> 停掉现有的SLAVE
> 记录下停止时的位置
> 备份slave
> 启动slave
> 
> 开始新SLAVE
> change master 的时候用上面记录的位置 （file and pos）
> 
> 上述也是我目前的常用做法，不知有何其他高见!

mysqladmin

简单一点的

```
$ mysqladmin -uroot -p -h127.0.0.1 -P3306 -r -i 1 ext |\
awk -F"|" '{\
  if($2 ~ /Variable_name/){\
    print " <-------------    "  strftime("%H:%M:%S") "    ------------->";\
  }\
  if($2 ~ /Questions|Queries|Innodb_rows|Com_select |Com_insert |Com_update |Com_delete |Innodb_buffer_pool_read_requests/)\
    print $2 $3;\
}'
```

使用awk，复杂一点。

```
$ mysqladmin -P3306 -uroot -p -h127.0.0.1 -r -i 1 ext |\
awk -F"|" \
"BEGIN{ count=0; }"\
'{ if($2 ~ /Variable_name/ && ((++count)%20 == 1)){\
    print "----------|---------|--- MySQL Command Status --|----- Innodb row operation ----|-- Buffer Pool Read --";\
    print "---Time---|---QPS---|select insert update delete|  read inserted updated deleted|   logical    physical";\
}\
else if ($2 ~ /Queries/){queries=$3;}\
else if ($2 ~ /Com_select /){com_select=$3;}\
else if ($2 ~ /Com_insert /){com_insert=$3;}\
else if ($2 ~ /Com_update /){com_update=$3;}\
else if ($2 ~ /Com_delete /){com_delete=$3;}\
else if ($2 ~ /Innodb_rows_read/){innodb_rows_read=$3;}\
else if ($2 ~ /Innodb_rows_deleted/){innodb_rows_deleted=$3;}\
else if ($2 ~ /Innodb_rows_inserted/){innodb_rows_inserted=$3;}\
else if ($2 ~ /Innodb_rows_updated/){innodb_rows_updated=$3;}\
else if ($2 ~ /Innodb_buffer_pool_read_requests/){innodb_lor=$3;}\
else if ($2 ~ /Innodb_buffer_pool_reads/){innodb_phr=$3;}\
else if ($2 ~ /Uptime / && count >= 2){\
  printf(" %s |%9d",strftime("%H:%M:%S"),queries);\
  printf("|%6d %6d %6d %6d",com_select,com_insert,com_update,com_delete);\
  printf("|%6d %8d %7d %7d",innodb_rows_read,innodb_rows_inserted,innodb_rows_updated,innodb_rows_deleted);\
  printf("|%10d %11d\n",innodb_lor,innodb_phr);\
}}'
```

### mysqldumpslow常用命令

由上面的常用参数就可以组合出如下的常用命令：


```
mysqldumpslow -s t slow.log.old > slow.1.dat	
#按照query time排序查看日志
mysqldumpslow -s at slow.log.old > slow.2.dat	
#按照平均query time排序查看日志
mysqldumpslow -a -s at slow.log.old > slow.3.dat	
#按照平均query time排序并且不抽象数字的方式排序
mysqldumpslow -a -s c slow.log.old > slow.4.dat 
#安装执行次数排序
```

参考：[mysqldumpslow Manual](http://dev.mysql.com/doc/refman/5.1/en/mysqldumpslow.html)