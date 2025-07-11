# 从论文学隔离级别
>https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr-95-51.pdf
> 
>https://tidb.net/blog/765da817

## 核心总结

这里有很多混乱的概念，统一总结如下：

隔离级别，是随着解决常见的异常现象(读写，写写问题)的程度依次上升的。

而隔离级别的实现，通常有`单版本数据模型`和`多版本数据模型`两种。

- 单版本数据模型，全程通过锁机制来实现，也就是本文上面的Table所介绍的，通过规定锁的行为并组合起来，解决读写 & 写写问题，实现不同的隔离级别。
- 多版本数据模型，也就是引入多数据版本机制，也就是MVCC。此时MVCC机制配合其他优化机制，可以一起解决读写 & 写写问题，从而实现不同的隔离级别。针对读写问题，也就是Dirty Read，Fuzzy Read，以及Phantom，都可以通过只通过MVCC机制来解决，它的目的就是保证读不阻塞写，写不阻塞读(针对幻读比较特殊，实际上也是使用了`predicate Lock`，或者说是`gap lock`才能解决)。而至于写写问题，最主要就是Lost update问题，可以通过First commiter Wins，以及First updater wins来解决。前者是乐观锁，同时应用多个更改，在提交的时候检测冲突，后提交的直接abort。之所以乐观就是因为它假设大部分写写并不冲突。后者是悲观锁，单次只能应用一个更改，也就是在写的时候就直接检测冲突，之所以悲观就是因为它假设认为大部分写写是冲突的。

注意这里的Lost update，并不是Dirty Write。后者指的是两个事务未提交前互相更新冲突，而前者指的是后提交的事务覆盖前事务的修改。

## ANSI SQL-92 隔离级别定义

SQL-92，通过解决三类现象，定义了四种隔离级别。隔离级别的实现并不限定实现方式(无论是锁还是MVCC)
> P1 (Dirty Read): Transaction T1 modifies a data item.
Another transaction T2 then reads that data item before T1
performs a COMMIT or ROLLBACK. If T1 then performs a
ROLLBACK, T2 has read a data item that was never
committed and so never really existed.

现象1：Dirty Read，又称为脏读。

- 宽泛解释P : w1[x]...r2[x]...（(c1 or a1) ... (c2 or a2) in any order）
- 严格解释A : w1[x]...r2[x]...a1 and c2（T1 回滚(rollback or abort)后 T2 已提交）

> P2 (Non-repeatable or Fuzzy Read): Transaction T1
reads a data item. Another transaction T2 then modifies or
deletes that data item and commits. If T1 then attempts to
reread the data item, it receives a modified value or
discovers that the data item has been deleted.

现象2：Fuzzy Read，又称为不可重复读。

- 宽泛解释P : r1[x]...w2[x]...（(c1 or a1) ... (c2 or a2) in any order）
- 严格解释A : r1[x]...w2[x]...c2...r1[x]...c1（T1 两次读之间 T2 提交

>P3 (Phantom): Transaction T1 reads a set of data items
satisfying some "search condition". Transaction T2
then creates data items that satisfy T1’s "search condition" and commits. If T1 then repeats its read with the
same "search condition", it gets a set of data items
different from the first read.

现象3：Phantom, 又称为幻读。

- 宽泛解释P : r1[P]...w2[y in P]...（(c1 or a1) ... (c2 or a2) in any order）
- 严格解释A : r1[P]...w2[y in P]...c2...r1[P]...c1（T1 第二次查询前 T2 提交）

> 其中A表示一定会发生问题的严格情况，而P表示可能发生问题

根据解决的现象，定义四种隔离级别：
![sqlLevel](sqlLevel.png)

ANSI-SQL-92的定义是由问题的。实际上，只解决了这三种问题，并到不了串行化的级别。
还需要额外解决一些其他异常(A)。并且，这里的P & A定义是基于`单版本数据模型`(基于锁的悲观并发控制)，而实际上还有基于MVCC，以及乐观的并发控制(也就是`多版本数据模型`)的实现。


## 基于锁的悲观并发控制

### 锁的分类
数据库中的锁，分为共享锁(读锁)和排他锁(写锁)。只有`read lock & read lock`是不冲突的。

对于这两种锁，又都可以分为Predicate lock & item lock，也就是谓词锁和记录锁。前者锁住满足条件的所有记录，以及间隙(针对幻影数据)。而后者只是锁定特殊记录。

### 锁的行为规范

> A transaction has well-formed writes (reads) if it requests
a Write (Read) lock on each data item or predicate before
writing (reading) that data item, or set of data items
defined by a predicate. The transaction is well-formed if it
has well-formed writes and reads. A transaction has twp phase writes (reads) if it does not set a new Write (Read)
lock on a data item after releasing a Write (Read) lock. A
transaction exhibits two-phase locking if it does not
request any new locks after releasing some lock.

一个事务是well-formed的，需要满足以下条件：

- 写入之前必须先获取写锁(Predicate lock or item lock)
- 读取之前必须先获取读锁(Predicate lock or item lock)

一个事务是two-phase-read/write的，需要满足以下条件：

- 不能在对一个data item 释放一个写锁之后再获取一个写锁
- 不能在对一个data item 释放一个读锁之后再获取一个读锁

而锁的持续时间，有如下定义:

- Long Duration Lock: 锁被该事务持有，直到该事务提交或者回滚。
- Short Duration Lock: 锁被该事务持有，该操作完成直接释放锁。

通过上述锁的行为描述，可以将锁实现的隔离级别进行如下描述:

![lock_base_iso](lock_base_iso.png)

这里和ANSI-SQL-92有一些核心区别：

- 等级划分还是4种，但是均比ANSI-SQL-92严格
- 针对Repeatable Read，ASNI允许Phantom出现，但是Locking不允许。

实际上，ANSI的Repeatable Read，并不能阻止幻读出现。前者聚焦于item,而后者聚焦于predicate。
如果站在是否`Repeatable`，则ANSI的Repeatable Read实际上并不能防止predicate情况下的幻读，后者层次更高，需要更严格的控制。

## 级别的修正与重新定义

这里从`Dirty Write`部分开始考虑。

定义`Dirty Write`：w1[x]...w2[x]...（T1 和 T2 的提交/回滚顺序任意），所有的隔离级别都应该禁止脏写。一般是通过锁机制来防止的。

正确的隔离级别应该采用P描述而不是A描述，也就是宽泛而不是严格。修正之后的隔离级别如下：

![modify_iso](modify_iso.png)

这个新版本就是锁机制的又一种表达方式，它们一一对应。

## Snapshot Isolation
Snapshot Isolation，也就是快照隔离。每个事务都从ReadView中获取数据，这个View是某个时间戳(Start_timestamp)之前的快照。
运行在快照隔离下的事务，读取的时候不会受到锁的阻塞，写入的时候也是修改的快照数据，不会立刻影响其他事务。

一个例子：r1[x0=50] w1[x1=10] r2[x0=50] r2[y0=50] c2 r1[y0=50] w1[y1=90] c1

采用First-committer-wins ：当T1准备commit的时候，会获得Commit-TimeStamp,该值大于所有已经存在的start_timestamp，以及commit_stamp。
当没有T2在[T1 start, T1 Commit]区间内写过T1也写过的数据时候，T1才能commit成功，否则T1被abort。T1 commit之后，其更改对所有start-ts > t1.commit_ts的事务可见。
采用该规则的核心是为了避免更新丢失。

快照隔离并不是等价于完全可串行化的效果，其阻止了幻读，但是没有阻止Write Skew。而repeatable read则阻止了幻读，但是没有阻止Write Skew。
由此可见，两者是同一水平的隔离。事实上，MySQL如今宣称的实现的RR级别，其实实现了SI级别的隔离。



![iso_summary](iso_summary.png)


