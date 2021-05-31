# SQLite FTS3和FTS4扩展 
（非官方，LeoKing）

目录
1. 介绍FTS3和FTS4
    1. FTS3和FTS4的区别
    2. 创建和销毁FTS表
    3. 填充FTS表
    4. 简单的FTS查询
    5. 小结
1. 编译并启用FTS3和FTS4
2. 全文索引查询
    1. 使用增强查询语法设置选项
    2. 使用基本查询语法设置选项
3. 辅助功能-摘要，偏移量，匹配
    1. 偏移量
    2. 摘要
    3. 匹配
4. Fts4aux-直接访问全文索引
5. FTS4选项
    1. 压缩选项(comperss==)和解压缩(uncompress==)选项
    2. content==选项
        1. Contentless FTS4表
        2. External FTS4表
    3. languageid= 选项
    4. matchinfo= 选项
    5. notindexed= 选项
    6. prefix= 选项
6. FTS3和FTS4专有命令
    1. "optimize" 命令
    2. "rebuild" 命令
    3. "integrity-check" 命令
    4. "merge=X,Y" 命令
    5. "automerge=N" 命令
7. 分词器
    1. 自定义(应用程序定义)分词器
    2. 查询分词器
8. 数据结构
    1. 影子表
    2. 可变长整型 varint
    3. B树格式
    4. B树叶子结点
    5. B树内部节点
    6. 文档列表格式（Doclist）
9. 局限性
    1. UTF-16字节顺序标记问题
10. 附录A：搜索应用提示


**概要**
FTS3和FTS4是SQLite的虚拟表模块，它可以让用户能够对一组文档进行全文搜索。
实际上，Google，必应，百度等搜索引擎对网络上的文档进行的搜索就是全文搜索。
用户输入一个词语或者一组词语，可能由运算符进行连接，或者组成一个短语。 全文查询系统根据用户指定的操作符和分组找到与这些术语最匹配的文档。
本文将介绍如何部署和使用FTS3和FTS4。

FTS1和FTS2已经过时。
这些较老的模块存在一些已知的问题，应该避免使用它们。
谷歌的Scott Hess将最初的FTS3代码的一部分贡献给SQLite项目。
它现在作为SQLite的一部分进行开发和维护。


1. 介绍FTS3和FTS4
	FTS3和FTS4扩展模块允许用户创建带有内置全文索引的特殊表(以下简称“FTS表”)。
	全文索引允许用户有效地查询数据库中包含一个或多个单词(以下简称“分词”)的所有行，即使该表包含许多大型文档。
	例如，如果“Enron E-Mail 数据集”中的每个517430文档都被插入到一个FTS表和一个使用以下SQL脚本创建的普通SQLite表中:


		/* FTS3 表 */
		CREATE VIRTUAL TABLE enrondata1 USING fts3(content TEXT);  
		/* 普通 表 */
		CREATE TABLE enrondata2(content TEXT);                        


	然后可以执行下面两个查询中的任意一个来查找数据库中包含单词“linux”(351)的文档数量。
	使用一个桌面PC硬件配置，对FTS3表的查询大约需要0.03秒，而对普通表的查询需要22.5秒。

        /* 0.03 秒 */
		SELECT count(*) FROM enrondata1 WHERE content MATCH 'linux';  
		/* 22.5 秒 */
		SELECT count(*) FROM enrondata2 WHERE content LIKE '%linux%'; 

	当然，上面的两个查询并不完全相同.

(未完待续…..)


        
