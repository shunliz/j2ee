###### 嵌套事务的回滚

2017年05月02日 16:33:32

阅读数：1974

嵌套事务和事务保存点的错误处理

对于嵌套事务。  
1.外部起事务，内部起事务，内外都有Try Catch  
内部出错:如果内部事务出错，内部和外部事物全部回滚，外部回滚之前的操作全部不存在，但是之后的操作继续执行。  
外部出错:如果外部事物出错，内部和外部事物全部回滚，外部回滚之前的操作全部不存在，但是之后的操作继续执行。  
注:如果内部的事务不起事务名称，内部如果出错，将会回滚掉会话中的全部事务，而且报异常。

2.外部起事务，内部起事务，内部没有Try Catch  
内部出错:如果内部事务出错，内部和外部事物全部回滚，外部回滚之前的操作全部不存在，但是之后的操作继续执行。  
外部出错:如果内部事务出错，内部和外部事物全部回滚，外部回滚之前的操作全部不存在，但是之后的操作继续执行。

3.外部起事务，内部不起事务，但有Try Catch。  
内部出错:外部事物正常提交，外部事物不会进入ROLLBACK,内部出错之后的记录也会正常执行。内部操作中，Try部分在错误出现之前的操作正常，Try部分在操作之后的操作不执行，然后进入Catch块中执行操作。  
外部出错:内部和外部事物全部回滚，外部回滚之前的操作全部不存在，但是之后的操作继续执行。

4.外部起事务，内部不起事务，但没有Try Catch.  
内部出错:如果内部事务出错，内部和外部事物全部回滚，外部回滚之前的操作全部不存在，但是之后的操作继续执行。  
外部出错:如果内部事务出错，内部和外部事物全部回滚，外部回滚之前的操作全部不存在，但是之后的操作继续执行。

5.外部不起事务，内部起事务，但有Try Catch.  
内部出错:外部操作被正常执行，内部ROLLBACK操作前全部回滚，之后的操作正常执行。  
外部出错:出错操作之前的操作不会回滚，出错之后的操作不执行，跳入Catch块中，内部事务不会回滚。

6.外部不起事务，内部起事务，但没有Try Catch.  
内部出错:外部操作被正常执行，内部ROLLBACK操作前全部回滚。由于没有catch块，所以外部操作全部执行。  
外部出错:内部事务正常提交，外部只有当条记录失败，其他操作正常执行，但是有严重错误报出来。

对于事务保存点  
事务保存点只有SAVE和ROLLBACK操作，当外部调用内部保存点，内部出现问题不影响外部事务，外部操作正常执行。当外部操作出现问题时，内部所有操作都回滚掉。

如：外部起事务，内部起保存点，内外都有Try Catch  
内部出错:外部操作正常，不进入Catch，内部事务回滚到保存点，之后的继续执行。  
外部出错:如果外部事物在保存点之前出现异常，那么外部和内部所有操作回滚。如果外部事物在保存点之前出现异常，由于保存点已经提交了事务，导致外部rollback找不到对应的事务点。

[http://blog.sina.com.cn/s/blog\_6003801e0100drus.html](http://blog.sina.com.cn/s/blog_6003801e0100drus.html)

事务的嵌套

PRINT 'Trancount before transaction: ' + CAST\(@@TRANCOUNT as char\(1\)\)

BEGIN TRAN

PRINT 'After first BEGIN TRAN: ' + CAST\(@@TRANCOUNT as char\(1\)\)

BEGIN TRAN

PRINT 'After second BEGIN TRAN: ' + CAST\(@@TRANCOUNT as char\(1\)\)

COMMIT TRAN

PRINT 'After first COMMIT TRAN: ' + CAST\(@@TRANCOUNT as char\(1\)\)

COMMIT TRAN

PRINT 'After second COMMIT TRAN: ' + CAST\(@@TRANCOUNT as char\(1\)\)

在结果中，可以看到每一个BEGIN TRAN 语句都会使@@TRANCOUNT增加1并且每一个COMMIT TRAN语句都会使其减少1。如前所述，一个值为0的@@TRANCOUNT意味着没有打开的事务。因此，在@@TRANCOUNT值从1降到0时结束的事务发生在外层事务提交的时候。因此，每一个内部事务都需要提交。由于事务起始于第一个BEGIN TRAN并结束于最后一个COMMIT TRAN，因此**最外层的事务决定了是否完全提交内部的事务。如果最外层的事务没有被提交，其中嵌套的事务也不会被提交。**

键入并执行以下批来检验事务回滚时所发生的情况：

BEGIN TRAN

PRINT 'After 1st BEGIN TRAN: ' + CAST\(@@TRANCOUNT as char\(1\)\)

BEGIN TRAN

PRINT 'After 2nd BEGIN TRAN: ' + CAST\(@@TRANCOUNT as char\(1\)\)

BEGIN TRAN

PRINT 'After 3rd BEGIN TRAN: ' + CAST\(@@TRANCOUNT as char\(1\)\)

UPDATE Data1

SET value1 = 1000000

WHERE Id = 1

COMMIT TRAN

PRINT 'After first COMMIT TRAN: ' + CAST\(@@TRANCOUNT as char\(1\)\)

ROLLBACK TRAN

PRINT 'After ROLLBACK TRAN: ' + CAST\(@@TRANCOUNT as char\(1\)\)

SELECT \* FROM Data1

WHERE Id = 1;

在这个示例中，数据表Data1在一个嵌套事务中被更新，这会被立即提交。然后ROLLBACK TRAN被执行。ROLLBACK TRAN将@@TRANCOUNT减为0并回滚整个事务及其中嵌套的事务，无论它们是否已经被提交。因此，嵌套事务中所做的更新被回滚，数据没有任何改变。

始终牢记，**在嵌套的事务中，只有最外层的事务决定着是否提交内部事务**。每一个COMMIT TRAN语句总是应用于最后一个执行的BEGIN TRAN。因此，对于每一个COMMIT TRAN，必须调用一个COMMIT TRAN来提交事务。ROLLBACK TRAN语句总是属于最外层的事务，并且因此总是回滚整个事务而不论其中打开了多少嵌套事务。正因为此，管理嵌套事务很复杂。如果每一个嵌套存储过程都在自身中开始一个事务，那么嵌套事务大部分会发生在嵌套存储过程中。要避免嵌套事务，可以在过程开始处检查@@TRANCOUNT的值，以此来确定是否需要开始一个事务。如果@@TRANCOUNT大于0，因为过程已经处于一个事务中并且调用实例可以在错误发生时回滚事务。

存储过程和触发器中回滚　　如果 @@TRANCOUNT 的值在存储过程完成时与过程执行时不同，则会生成一个 266 信息类错误。该错误不是由触发器中同一个条件生成的。

当调用存储过程时，如果 @@TRANCOUNT 为 1 或更大，并且该过程执行 ROLLBACK TRANSACTION 或 ROLLBACK WORK 语句，则会产生 266 号错误。这是因为 ROLLBACK 回滚所有未完成的事务，并将 @@TRANCOUNT 减到 0，该值比调用过程时要小。

如果在触发器中发出 ROLLBACK TRANSACTION：

对当前事务中的那一点所做的所有数据修改都将回滚，包括触发器所做的修改。

触发器继续执行 ROLLBACK 语句之后的所有其余语句。如果这些语句中的任意语句修改数据，则不回滚这些修改。执行其余的语句不会激发嵌套触发器。

在批处理中，所有位于激发触发器的语句之后的语句都不被执行。

触发器中的 ROLLBACK 关闭并释放所有在包含激发触发器的语句的批处理中声明和打开的游标。这其中包括了在激发触发器的批处理所调用的存储过程中声明和打开的游标。在激发触发器 的批处理之前的批处理中所声明的游标将只是关闭，但是在以下条件下，STATIC 或 INSENSITIVE 游标不关闭：

CURSOR\_CLOSE\_ON\_COMMIT 设置为OFF。

静态游标要么是同步游标，要么是完全填充的异步游标。

当执行触发器时，触发器的操作总是好像有一个未完成的事务在起作用。如果激发触发器的语句是在隐性或显式事务中，则肯定会这样。在自动提交模式下，也是 如此。当语句开始以自动提交模式执行时，如果遇到错误，则会有隐含的 BEGIN TRANSACTION 语句允许恢复该语句生成的所有修改。该隐含的事务对批处理中的其它语句没有影响，因为当语句完成时，该事务要么提交，要么回滚。但是，当调用触发器时，该 隐含的事务将仍然有效。

这意味着，只要触发器中发出 BEGIN TRANSACTION 语句，则实际上就开始了一个嵌套事务。因为当回滚嵌套事务时，嵌套的 BEGIN TRANSACTION 语句将被忽略，触发器中发出的 ROLLBACK TRANSACTION 总是回滚过去该触发器本身发出的所有 BEGIN TRANSACTION 语句。ROLLBACK 回滚到最外部的 BEGIN TRANSACTION。

若要在触发器中进行部分回滚，则即使总是以自动提交模式进行调用，也必须使用 SAVE TRANSACTION 语句。以下的触发器阐明了这一点：

CREATE TRIGGER TestTrig ON TestTab FOR UPDATE AS

SAVE TRANSACTION MyName

INSERT INTO TestAudit

SELECT \* FROM inserted

IF \(@@error

&lt;

&gt;

0\)

BEGIN

ROLLBACK TRANSACTION MyName

END

这也影响触发器中 BEGIN TRANSACTION 语句后面的COMMIT TRANSACTION 语句。因为 BEGIN TRANSACTION 启动一个嵌套事务，而随后的 COMMIT 语句只应用于该嵌套事务。如果在 COMMIT 之后执行 ROLLBACK TRANSACTION 语句，那么 ROLLBACK 将一直回滚到最外部的 BEGIN TRANSACTION。以下的触发器阐明了这一点：

CREATE TRIGGER TestTrig ON TestTab FOR UPDATE AS

BEGIN TRANSACTION



INSERT INTO TrigTarget

SELECT \* FROM inserted

COMMIT TRANSACTION

ROLLBACK TRANSACTION

此触发器绝对不会在 TrigTarget 表中插入。BEGIN TRANSACTION 总是启动一个嵌套事务。COMMIT TRANSACTION 只提交嵌套事务，而下面的 ROLLBACK TRANSACTION 则一直回滚到最外部的 BEGIN TRANSACTION。

