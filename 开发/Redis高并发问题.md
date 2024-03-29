---
author: 初一七月
tags: [redis]
date: 2013-01-28
---
# Redis并发问题

Redis为单进程单线程模式，采用队列模式将并发访问变为串行访问。Redis本身没有锁的概念，Redis对于多个客户端连接并不存在竞争，但是在Jedis客户端对Redis进行并发访问时会发生连接超时、数据转换错误、阻塞、客户端关闭连接等问题，这些问题均是由于客户端连接混乱造成。对此有2种解决方法：

1.客户端角度，为保证每个客户端间正常有序与Redis进行通信，对连接进行池化，同时对客户端读写Redis操作采用内部锁synchronized。

2.服务器角度，利用setnx实现锁。

对于第一种，需要应用程序自己处理资源的同步，可以使用的方法比较通俗，可以使用synchronized也可以使用lock；第二种需要用到Redis的setnx命令，但是需要注意一些问题。

## SETNX命令（SET if Not eXists）

语法：
SETNX key value

功能：
将 key 的值设为 value ，当且仅当 key 不存在；若给定的 key 已经存在，则 SETNX 不做任何动作。

时间复杂度：
O(1)
返回值：
设置成功，返回 1 。
设置失败，返回 0 。

模式：将 SETNX 用于加锁(locking)

SETNX 可以用作加锁原语(locking primitive)。比如说，要对关键字(key) foo 加锁，客户端可以尝试以下方式：

SETNX lock.foo <current Unix time + lock timeout + 1>

如果 SETNX 返回 1 ，说明客户端已经获得了锁， key 设置的unix时间则指定了锁失效的时间。之后客户端可以通过 DEL lock.foo 来释放锁。

如果 SETNX 返回 0 ，说明 key 已经被其他客户端上锁了。如果锁是非阻塞(non blocking lock)的，我们可以选择返回调用，或者进入一个重试循环，直到成功获得锁或重试超时(timeout)。

但是已经证实仅仅使用SETNX加锁带有竞争条件，在特定的情况下会造成错误。

处理死锁(deadlock)

上面的锁算法有一个问题：如果因为客户端失败、崩溃或其他原因导致没有办法释放锁的话，怎么办？

这种状况可以通过检测发现——因为上锁的 key 保存的是 unix 时间戳，假如 key 值的时间戳小于当前的时间戳，表示锁已经不再有效。

但是，当有多个客户端同时检测一个锁是否过期并尝试释放它的时候，我们不能简单粗暴地删除死锁的 key ，再用 SETNX 上锁，因为这时竞争条件(race condition)已经形成了：

C1 和 C2 读取 lock.foo 并检查时间戳， SETNX 都返回 0 ，因为它已经被 C3 锁上了，但 C3 在上锁之后就崩溃(crashed)了。
C1 向 lock.foo 发送 DEL 命令。
C1 向 lock.foo 发送 SETNX 并成功。
C2 向 lock.foo 发送 DEL 命令。
C2 向 lock.foo 发送 SETNX 并成功。
出错：因为竞争条件的关系，C1 和 C2 两个都获得了锁。


幸好，以下算法可以避免以上问题。来看看我们聪明的 C4 客户端怎么办：

C4 向 lock.foo 发送 SETNX 命令。
因为崩溃掉的 C3 还锁着 lock.foo ，所以 Redis 向 C4 返回 0 。
C4 向 lock.foo 发送 GET 命令，查看 lock.foo 的锁是否过期。如果不，则休眠(sleep)一段时间，并在之后重试。
另一方面，如果 lock.foo 内的 unix 时间戳比当前时间戳老，C4 执行以下命令：
GETSET lock.foo <current Unix timestamp + lock timeout + 1>

因为 GETSET 的作用，C4 可以检查看 GETSET 的返回值，确定 lock.foo 之前储存的旧值仍是那个过期时间戳，如果是的话，那么 C4 获得锁。
如果其他客户端，比如 C5，比 C4 更快地执行了 GETSET 操作并获得锁，那么 C4 的 GETSET 操作返回的就是一个未过期的时间戳(C5 设置的时间戳)。C4 只好从第一步开始重试。
注意，即便 C4 的 GETSET 操作对 key 进行了修改，这对未来也没什么影响。

这里假设锁key对应的value没有实际业务意义，否则会有问题，而且其实其value也确实不应该用在业务中。

为了让这个加锁算法更健壮，获得锁的客户端应该常常检查过期时间以免锁因诸如 DEL 等命令的执行而被意外解开，因为客户端失败的情况非常复杂，不仅仅是崩溃这么简单，还可能是客户端因为某些操作被阻塞了相当长时间，紧接着 DEL 命令被尝试执行(但这时锁却在另外的客户端手上)。


## GETSET命令

语法：
GETSET key value

功能：
将给定 key 的值设为 value ，并返回 key 的旧值(old value)。当 key 存在但不是字符串类型时，返回一个错误。

时间复杂度：
O(1)

返回值：
返回给定 key 的旧值；当 key 没有旧值时，也即是， key 不存在时，返回 nil 。
