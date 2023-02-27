# 第十六章-Optimize Trace的神奇功效	

## 	简介

​		之前我们可以通过EXPLAIN语句查看到优化器最终决定使用的执行计划，却无法知道它为什么做出这样的决策。MySQL5.6以及之后的版本中，设计Mysql的大叔为我们提供了`optimizer trace`的功能，这个功能可以让用户方便地查看**优化器生成执行计划**的整个过程，这个功能的开启与关闭由系统变量`optimizer_trace`来决定。

​		先在命令行中使用`SET optimizer_trace="enabled=on"`打开该功能，然后输入想要查看优化过程的查询语句，当该查询语句执行完成后，就可以到`informatin_schema`数据库下的OPTIMIZER_TRACE表中查看完整的执行计划生成过程，OPTIMIZER_TRACE表有4列，分别如下

  - QUERY：表示我们输入的查询语句

  - TRACE：表示优化过程的JSON格式的文本

  - MISSING_BYTES_BEYOND_MAX_MEM_SIZE：在执行计划的生成过程中可能会输出很多内容，若超过某个限制，多余的文本就不会显示，该字段展示被忽略的文本字节数

  - INSURFFICIENT_PRIVILEGES：表示是否有权限查看执行计划的生成过程，默认值是0，表示有权限

    ​	

    优化过程大致分为三个阶段：

    - Prepare阶段
    - Optimize阶段
    - Execute阶段

    基于成本的优化主要集中在Optimize阶段，单表查询主要关注Optimize阶段的`rows_estimation`过程；多表连接查询主要关注`considered_execution_plans`过程

