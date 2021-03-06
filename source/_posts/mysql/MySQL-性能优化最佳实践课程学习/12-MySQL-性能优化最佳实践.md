---
title: MySQL性能优化最佳实践 - 12 MySQL性能优化的最佳20+条经验
date: 2017-12-5
tags:
- mysql
categories:
- MySQL性能优化最佳实践
---

本章有部分内容与《[MySQL性能优化最佳实践 - 10 MySQL写出高效SQL](/2017/11/09/mysql/MySQL-性能优化最佳实践课程学习/10-MySQL-性能优化最佳实践/)》互为补充。

## 从程序员的角度优化
* 数据库的操作越来越成为整个应的的性能瓶颈，这点对于WEB应用尤其明显。
* 关于数据库性能，这不只是DBA才需要担心的事，更是程序员需要去关注的事情。
* 当设计表结构和操作数据库（DML操作）时，需要注意数据操作的性能。

### 为查询缓存优化你的查询
大多数的MySQL服务器都开启了查询缓存

### EXPLAIN 你的SELECT 查询
* 使用EXPLAIN 关键字可以让你知道MySQL是如何处理你的SQL语句的。这可以帮你分析你的查询语句或是表结构的性能瓶颈。
* EXPLAIN 的查询结果还会告诉你你的索引主键被如何利用的，你的数据表是如何被搜索和排序的等等。
* 挑一个你的SELECT语句（推荐挑选那个最复杂的，有多表联接的），把关键字EXPLAIN加到前面。你可以使用phpmyadmin来做这个事。然后，你会看到一张表格。下面的这个示例中，我们忘记加上了group_id索引，并且有表联接。
* 当我们为group_id字段加上索引后：我们可以看到，前一个结果显示搜索了7883 行，而后一个只是搜索了两个表的9 和16 行。查看rows列可以让我们找到潜在的性能问题。

<!-- more -->

### 当只要一行数据时使用LIMIT 1

### 为搜索字段建索引

### 在Join表的时候使用相当类型的例，并将其索引

### 千万不要ORDER BY RAND()

### 避免SELECT *

### 永远为每张表设置一个ID
* 我们应该为数据库里的每张表都设置一个ID做为其主键，而且最好的是一个INT型的（推荐使用UNSIGNED），并设置上自动增加的AUTO_INCREMENT标志。
* 就算是你users 表有一个主键叫“email”的字段，你也别让它成为主键。使用VARCHAR类型来当主键会使用得性能下降。另外，在你的程序中，你应该使用表的ID来构造你的数据结构。
* 而且，在MySQL数据引擎下，还有一些操作需要使用主键，在这些情况下，主键的性能和设置变得非常重要，比如，集群，分区……
* 在这里，只有一个情况是例外，那就是“关联表”的“外键”，也就是说，这个表的主键，通过若干个别的表的主键构成。我们把这个情况叫做“外键”。比如：有一个“学生表”有学生的ID，有一个“课程表”有课程ID，那么，“成绩表”就是“关联表”了，其关联了学生表和课程表，在成绩表中，学生ID和课程ID叫“外键”其共同组成主键。

### 使用ENUM 而不是VARCHAR
* ENUM 类型是非常快和紧凑的。在实际上，其保存的是TINYINT，但其外表上显示为字符串。这样一来，用这个字段来做一些选项列表变得相当的完美。
* 如果你有一个字段，比如“性别”，“国家”，“民族”，“状态”或“部门”，你知道这些字段的取值是有限而且固定的，那么，你应该使用ENUM 而不是VARCHAR。
* MySQL也有一个“建议”（见第十条）告诉你怎么去重新组织你的表结构。当你有一个VARCHAR 字段时，这个建议会告诉你把其改成ENUM类型。使用PROCEDURE ANALYSE() 你可以得到相关的建议。

### 从PROCEDURE ANALYSE() 取得建议
* PROCEDURE ANALYSE()会让MySQL帮你去分析你的字段和其实际的数据，并会给你一些有用的建议。只有表中有实际的数据，这些建议才会变得有用，因为要做一些大的决定是需要有数据作为基础的。
* 例如，如果你创建了一个INT 字段作为你的主键，然而并没有太多的数据，那么，PROCEDURE ANALYSE()会建议你把这个字段的类型改成MEDIUMINT 。或是你使用了一个VARCHAR 字段，因为数据不多，你可能会得到一个让你把它改成ENUM的建议。这些建议，都是可能因为数据不够多，所以决策做得就不够准。
* 在phpmyadmin里，你可以在查看表时，点击“Propose table structure” 来查看这些建议
* 一定要注意，这些只是建议，只有当你的表里的数据越来越多时，这些建议才会变得准确。一定要记住，你才是最终做决定的人。

### 尽可能的使用NOT NULL

### Prepared Statements

### 无缓冲的查询

### 把IP地址存成UNSIGNED INT

### 固定长度的表会更快
* 如果表中的所有字段都是“固定长度”的，整个表会被认为是“static” 或“fixed-length”。例如，表中没有如下类型的字段：VARCHAR，TEXT，BLOB。只要你包括了其中一个这些字段，那么这个表就不是“固定长度静态表”了，这样，MySQL引擎会用另一种方法来处理。
* 固定长度的表会提高性能，因为MySQL搜寻得会更快一些，因为这些固定的长度是很容易计算下一个数据的偏移量的，所以读取的自然也会很快。
* 而如果字段不是定长的，那么，每一次要找下一条的话，需要程序找到主键。
* 并且，固定长度的表也更容易被缓存和重建。不过，唯一的副作用是，固定长度的字段会浪费一些空间，因为定长的字段无论你用不用，他都是要分配那么多的空间。
* 使用“垂直分割”技术（见下一条），你可以分割你的表成为两个一个是定长的，一个则是不定长的。

### 垂直分割
* “垂直分割”是一种把数据库中的表按列变成几张表的方法，这样可以降低表的复杂度和字段的数目，从而达到优化的目的。（以前，在银行做过项目，见过一张表有100多个字段，很恐怖）
* 示例一：在Users表中有一个字段是家庭地址，这个字段是可选字段，相比起，而且你在数据库操作的时候除了个人信息外，你并不需要经常读取或是改写这个字段。那么，为什么不把他放到另外一张表中呢？
* 这样会让你的表有更好的性能，大家想想是不是，大量的时候，我对于用户表来说，只有用户ID，用户名，口令，用户角色等会被经常使用。小一点的表总是会有好的性能。
* 示例二：你有一个叫“last_login”的字段，它会在每次用户登录时被更新。但是，每次更新时会导致该表的查询缓存被清空。所以，你可以把这个字段放到另一个表中，这样就不会影响你对用户ID，用户名，用户角色的不停地读取了，因为查询缓存会帮你增加很多性能。
* 另外，你需要注意的是，这些被分出去的字段所形成的表，你不会经常性地去Join他们，不然的话，这样的性能会比不分割时还要差，而且，会是极数级的下降。

### 拆分大的DELETE 或INSERT 语句

### 越小的列会越快
* 对于大多数的数据库引擎来说，硬盘操作可能是最重大的瓶颈。所以，把你的数据变得紧凑会对这种情况非常有帮助，因为这减少了对硬盘的访问。
* 参看MySQL 的文档Storage Requirements 查看所有的数据类型。
* 如果一个表只会有几列罢了（比如说字典表，配置表），那么，我们就没有理由使用INT 来做主键，使用MEDIUMINT, SMALLINT 或是更小的TINYINT 会更经济一些。如果你不需要记录时间，使用DATE 要比DATETIME 好得多。
* 当然，你也需要留够足够的扩展空间，不然，你日后来干这个事，你会死的很难看，参看Slashdot的例子（2009年11月06日），一个简单的ALTER TABLE语句花了3个多小时，因为里面有一千六百万条数据。

### 选择正确的存储引擎
* 在MySQL 中有两个存储引擎MyISAM和InnoDB，每个引擎都有利有弊。酷壳以前文章《MySQL: InnoDB还是MyISAM?》讨论和这个事情。
* MyISAM适合于一些需要大量查询的应用，但其对于有大量写操作并不是很好。甚至你只是需要update一个字段，整个表都会被锁起来，而别的进程，就算是读进程都无法操作直到读操作完成。另外，MyISAM对于SELECT COUNT(*) 这类的计算是超快无比的。
* InnoDB的趋势会是一个非常复杂的存储引擎，对于一些小的应用，它会比MyISAM还慢。他是它支持“行锁”，于是在写操作比较多的时候，会更优秀。并且，他还支持更多的高级应用，比如事务。

### 使用一个对象关系映射器（Object Relational Mapper）
* 使用ORM (Object Relational
Mapper)，你能够获得可靠的性能增涨。一个ORM可以做的所有事情，也能被手动的编写出来。但是，这需要一个高级专家。
* ORM 的最重要的是“LazyLoading”，也就是说，只有在需要的去取值的时候才会去真正的去做。但你也需要小心这种机制的副作用，因为这很有可能会因为要去创建很多很多小的查询反而会降低性能。
* ORM 还可以把你的SQL语句打包成一个事务，这会比单独执行他们快得多得多。

### 小心“永久链接”
* “永久链接”的目的是用来减少重新创建MySQL链接的次数。当一个链接被创建了，它会永远处在连接的状态，就算是数据库操作已经结束了。而且，自从我们的Apache开始重用它的子进程后——也就是说，下一次的HTTP请求会重用Apache的子进程，并重用相同的MySQL 链接。
* 在理论上来说，这听起来非常的不错。但是从个人经验（也是大多数人的）上来说，这个功能制造出来的麻烦事更多。因为，你只有有限的链接数，内存问题，文件句柄数，等等。
* 而且，Apache运行在极端并行的环境中，会创建很多很多的了进程。这就是为什么这种“永久链接”的机制工作地不好的原因。在你决定要使用“永久链接”之前，你需要好好地考虑一下你的整个系统的架构。

