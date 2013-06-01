---
layout: post
title: "MySQL select实现原理"
categories: [technology, summary, storage]
description: 工作中需要借鉴MySQL对于select的具体实现，在网上搜了很久，几乎都是介绍原理的，对于实现细节都没有介绍，无奈之下只得自己对着源码gdb。结合以前对于sql解析的了解，对mysql select的具体实现有了大致的了解，总结一下。如果要gdb单步调试，需要在编译MySQl时加上debug选项，参见
---

工作中需要借鉴MySQL对于select的具体实现，在网上搜了很久，几乎都是介绍原理的，对于实现细节都没有介绍，无奈之下只得自己对着源码gdb。结合以前对于sql解析的了解，对mysql select的具体实现有了大致的了解，总结一下。

如果要gdb单步调试，需要在编译MySQl时加上debug选项，参见[这篇博客][debug].编译好以后就可以用gdb启动了。如果希望mysql运行时有日志输出，可以指定输出文件的路径和日志类型：`--debug=d,info,error,query,enter,general,where:O,/tmp/mysqld.trace`日志对MySQl内部逻辑的了解还是挺有用的。

MySQl在设计时，采用了这样的思路：针对主要应用场景选择一个或几个性能优异的核心算法作为引擎，然后努力将一些非主要应用场景作为该算法的特例或变种植入到引擎当中。具体而言，MySQL的select查询中，核心功能就是JOIN查询，因此在设计时，核心实现JOIN功能，对于其它功能，都通过转换为JOIN来实现。

比如`select id, name from student;`，MySQL在执行时，也会转换为JOIN来操作。

用gdb单步跟踪后，bt信息如下：
![][bt]

可以看出MySQL的执行过程大致如下：

1. 收到请求后分配线程处理；
2. sql解析，MySQL解析完sql以后，会生成很多item类。item类是sql解析和执行中最重要的类之一，对于它的介绍可以参见[这里][item]。
3. 执行sql，可以看到`JOIN::exec`，MySQL是将任何select都转换为JOIN来处理的。

以sql：`select A.id, B.score from student A left join subject B on A.id=B.id where A.age > 10 and B.score > 60;`为例来说明上面的步骤3的具体过程。

首先，MySQL在执行sql之前，会对sql进行优化处理，具体是在`JOIN::optimise`函数中完成。MySQL针对JOIN的优化做的非常好，因此才会将其他操作都转换为性能实现的非常好的JOIN操作。对于上面的sql，MySQL在执行时，会将join的key也转换为一个where条件：`A.id=B.id`来执行，那么经过处理后，上面的sql就有了3个where条件：

1. `A.age > 10`；
2. `A.id = B.id`；
3. `B.score > 60`；

预处理完以后开始执行，即`JOIN::exec`函数，首先会调用`send_fields`函数，将最终结果的信息返回，然后调用`do_select`。MySQL的join是采用nested loop join，可以参见[这篇博客][select]。在`do_select`函数中，通过调用`sub_select`函数来具体实现join功能。

在上面的例子中，需要完成2个join：先join表A，再join表B（这里请注意，不是涉及几个表，就需要join几个表，MySQL的join优化还是挺强大的，具体解释见后）。在MySQL进行sql解析时，会生成一个需要join的表的list，后面会挨个对该list的表进行join操作。

继续gdb，在`sub_select`函数中，可以看到这样一行代码：`(*join_tab->read_first_record)(join_tab)`这个就是读取表A的第一行结果，可以看`join_tab`里面的信息有表A的名字。接下来就是很关键的一个函数：`evaluate_join_record`，这个函数主要做2件事：

1. 将当前已经拿到的信息进行where条件计算，判断是否需要继续往下走；
2. 递归JOIN；

还是以上面的sql为例，首先执行第一个join，此时会遍历表A的每一行结果，每遍历一个结果，会进行where条件的判断。这里需要**注意**：当前的where条件判断只会判断**已经读出来的列**，由于此时只读出来表A的数据，因此现在只能对第一个where条件，即`A.age > 10`进行判断，如果满足，则递归调用join：`sql_select.cc: 11037  rc=(*join_tab->next_select)(join, join_tab+1, 0);`，这里的next_select函数就是`sub_select`，MySQL就是这样来实现**递归**操作的。如果不满足，则不会递归join，而是继续到下一行数据，从而达到剪枝的目的。

继续跟下去，此时通过上面的`next_select`递归的又调用到`sub_select`上，同样会走上面的逻辑，即先`read_first_record`，然后`evaluate_join_record`，这里由于表A和表B的数据都有了，于是可以对上面后面2个where条件：`A.id = B.id`和`B.score > 60`进行判断了。到此，所有的where条件都已经判断完毕，如果当前行对3个where条件都满足，就可以将结果输出。

以上就是select实现的大体过程，主要有2点，一个是join是采用递归实现的，另一个是每读一个表的数据，会将当前的where条件进行计算，剪枝。还有一个细节没有提到：MySQL是如何进行where条件判断的？或者说，MySQL是如何进行表达式计算的？

答案就是前面提到的item类。当MySQL在解析时，会将sql解析为很多item，同时也会建立各个item之间的关系。对于表达式，会生成一棵语法树。比如表达式：`B.score > 60`，此时会生成3个item：`B.score`、`>`和`60`，其中`B.score`和`60`分别是`>`的左右孩子，这样，求表达式的值时，就是求`>`的`val_int()`，然后就会递归的调用左右子树的`val_int()`，再做比较判断即可。

还有一个问题：如何求`B.score`的`val_int()`？对于此问题的答案我没有具体看过，根据之前一个同事的sql实现方式，我是这样推测的：`B.score`是数据表中的真实值，因此它的值肯定是通过去表中获取。在item类中，有一个函数：`fix_field`，它是用于告诉外界，去哪里获取此item的值，往往在sql执行的预处理阶段调用。于是在预处理时，告诉该item去某个固定buffer读取结果，同时，每当从表中读出一行数据时，将该数据保存在该buffer中，这样就可以将两者关联起来。这个部分纯属个人推测，感兴趣的同学可以自己根据源码看看。

再回到之前提到的一点，如果我们将sql稍微改一下：`select A.id, B.score from student A left join subject B on A.id=B.id where B.score > 60;`，即去掉第一个where条件，此时会发生什么？

答案是，MySQL会做一个优化，将sql转换为`select B.id, B.score from subject B where B.score > 60`，这样就不需要A同B join的逻辑了。实际上最开始我在gdb时就用的这条sql，结果死活看不到递归调用`sub_select`的场景，还以为原理不对，后来才发现是MySQL优化捣的乱。










[bt]:/image/gdb_mysql.JPG 
[debug]:http://www.cnblogs.com/end/archive/2011/05/18/2050277.html
[item]:http://dev.mysql.com/doc/internals/en/item-class.html
[select]:http://blog.csdn.net/tonyxf121/article/details/7796657
