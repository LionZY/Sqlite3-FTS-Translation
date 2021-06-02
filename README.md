**目录**
1. 介绍FTS3和FTS4
	* FTS3和FTS4的区别
	* 创建和销毁FTS表
    * 填充FTS表
    * 简单的FTS查询
    * 小结
2. 编译并启用FTS3和FTS4
3. 全文索引查询
    * 使用增强查询语法设置选项
    * 使用基本查询语法设置选项
4. 辅助功能-摘要，偏移量，匹配
    * 偏移量
    *  摘要
    *  匹配
5. Fts4aux-直接访问全文索引
6. FTS4选项
    *  压缩选项(comperss==)和解压缩(uncompress==)选项
    * content==选项
        * Contentless FTS4表
        * External FTS4表
    * languageid= 选项
    *  matchinfo= 选项
    *  notindexed= 选项
    *  prefix= 选项
7. FTS3和FTS4专有命令
    *  "optimize" 命令
    *  "rebuild" 命令
    *  "integrity-check" 命令
    *  "merge=X,Y" 命令
    *  "automerge=N" 命令
8. 分词器
    * 自定义(应用程序定义)分词器
    * 查询分词器
9. 数据结构
    * 影子表
    * 可变长整型 varint
    * B树格式
    * B树叶子结点
    * B树内部节点
    * 文档列表格式（Doclist）
10. 局限性
    * UTF-16字节顺序标记问题
11. 附录A：搜索应用提示

---

**概要**

&emsp;FTS3和FTS4是SQLite的虚拟表模块，它可以让用户能够对一组文档进行全文搜索。

&emsp;实际上，Google，必应，百度等搜索引擎对网络上的文档进行的搜索就是全文搜索。

&emsp;用户输入一个词语或者一组词语，可能由运算符进行连接，或者组成一个短语。 全文查询系统根据用户指定的操作符和分组找到与这些术语最匹配的文档。

&emsp;本文将介绍如何部署和使用FTS3和FTS4。

&emsp;FTS1和FTS2已经过时。

&emsp;这些较老的模块存在一些已知的问题，应该避免使用它们。


&emsp;谷歌的Scott Hess将最初的FTS3代码的一部分贡献给SQLite项目。

&emsp;它现在作为SQLite的一部分进行开发和维护。

---

1. 介绍FTS3和FTS4

&emsp;FTS3和FTS4扩展模块允许用户创建带有内置全文索引的特殊表(以下简称“FTS表”)。

&emsp;全文索引允许用户有效地查询数据库中包含一个或多个单词(以下简称“分词”)的所有行，即使该表包含许多大型文档。

&emsp;例如，如果源数据集中的每个517430文档都被插入到一个FTS表和一个使用以下SQL脚本创建的普通SQLite表中:

	CREATE VIRTUAL TABLE enrondata1 USING fts3(content TEXT);     /* FTS3表 */
	CREATE TABLE enrondata2(content TEXT);                        /* 普通表 */


&emsp;然后可以执行下面两个查询中的任意一个来查找数据库中包含单词“linux”(351)的文档数量。
&emsp;使用一个桌面PC硬件配置，对FTS3表的查询大约需要0.03秒，而对普通表的查询需要22.5秒。


	SELECT count(*) FROM enrondata1 WHERE content MATCH 'linux';  /* 0.03秒 */
	SELECT count(*) FROM enrondata2 WHERE content LIKE '%linux%'; /* 22.5秒 */


&emsp;当然，上面的两个查询并不完全相同.

&emsp;比如，按上述查询执行， LIKE查询匹配包含了诸如“linuxophobe”或者“EnterpriseLinux”短语的数据行（然而实际上，源数据集并不包含任何这样的短语），而FTS3表上的MATCH查询只选择那些包含“linux”作为离散分词的数据行。

&emsp;这两种搜索都不区分大小写。

&emsp;FTS3表消耗了大约2006MB的磁盘空间，而普通表只消耗1453MB。

&emsp;在同等的硬件配置下，执行上述语句，FTS3表需要31分钟，普通表需要25分钟。

---

1.1 FTS3和FTS4的区别

FTS3和FTS4几乎完全相同。它们共享的大部分代码是相同的，它们的接口也是相同的。区别是:

	1. FTS4包含查询性能优化，可以显著提高全文查询的性能，这些全文查询包含非常常见的短语(在表行中占比很大)
	2. FTS4支持一些可以与matchinfo()函数一起使用的附加选项。
	3. 由于FTS表将额外的信息存储在两个新的Shadow Table中，用以支持性能优化和额外的matchinfo()选项，
	   所以FTS4表可能比使用FTS3创建的等价表占用更多的磁盘空间。
       一般会多开销1-2%或更少，但如果存储在FTS表中的文档非常小，则可能高达10%。
       通过指定指令" matchfo =fts3"作为FTS4表声明的一部分，
       开销可能会减少，但这是以牺牲一些额外支持的matchfo()选项为代价的。
	4. FTS4提供了Hook(压缩和解压缩选项)，允许以压缩格式存储数据，减少了磁盘使用和IO

FTS4是FTS3的改进。FTS3从SQLite版本3.5.0(2007-09-04)开始可以使用。

我们应该在应用程序中使用FTS3还是FTS4呢?

	1. FTS3和FTS4性能在通常情况下是相似的，FTS4有时候比FTS3快很多，甚至快几个数量级，这取决于查询。
	2. FTS4还提供了增强的matchfo()输出，这在对MATCH操作的结果排序时很有用。
	3. 另一方面，在没有matchfo =fts3指令的情况下，FTS4需要比fts3多一点的磁盘空间，尽管在大多数情况下只有2%。

对于较新的应用程序，推荐使用FTS4; 如果与旧版本的SQLite的兼容性很重要，那么FTS3通常也可以。

---

1.2 创建和销毁FTS表

(未完待续…..)
