---
tags: [go, 代码调优]
date: 2018-04-18
author: Cyeam
author_link: http://blog.cyeam.com/golang/2017/02/08/go-optimize-slice-pool
author_avatar: https://blog.cyeam.com/assets/images/logo.jpg
---


> 带垃圾回收的语言，虽然对于刚刚上手的程序员是友好的，但是后期随着项目变得越来越巨大，维护的内存问题也会逐渐暴露出来。今天讲一种优化内存申请的方法——临时对象池。

- [写在前面](#写在前面)
- [堆还是栈？](#堆还是栈)
- [内存碎片化](#内存碎片化)
- [临时对象池](#临时对象池)
- [结论](#结论)

------

## 写在前面

在高并发的情况下，如果每次请求都需要申请一块用于计算的内存，比如：

```go
make([]int64, 0, len(ids))
```

将会是一件成本很高的事情。为了定位项目中的慢语句，我曾经采用“二分法”的方式打印慢日志，定位程序变慢的代码位置。它并不是每次都慢，而是每过几秒钟就突然变得极其慢，TPS能从2000降到200。引起这个问题就是类似于上面这条语句。

初始化一个`slice`，初学者会用：

```go
make([]int64, 0)
```

高级一些的程序员都会知道，这样第一次分配内存相当于没有分配，如果要后续`append`元素，会引起`slice`以指数形式扩充，可以参考下面的代码，追加了3个元素，`slice`扩容了3次。

```go
a := make([]int64, 0)
fmt.Println(cap(a), len(a))

for i := 0; i < 3; i++ {
	a = append(a, 1)
	fmt.Println(cap(a), len(a))
}
```

```
0 0
1 1
2 2
4 3
```

每一次扩容空间，都是会重新申请一块区域，把就空间里面的元素复制进来，把新的追加进来。那旧空间里面的元素怎么办？等着垃圾回收呗。

简单的优化方式，就是给自己要用的`slice`提前申请好空间，类似于最开头的那行代码。

```go
make([]int64, 0, len(ids))
```

这样做避免了`slice`多次扩容申请内存，但还是有问题的。

## 堆还是栈？

程序会从操作系统申请一块内存，而这块内存也会被分成堆和栈。栈可以简单得理解成一次函数调用内部申请到的内存，它们会随着函数的返回把内存还给系统。

```go
func F() {
	temp := make([]int, 0, 20)
	...
}
```

类似于上面代码里面的`temp`变量，只是内函数内部申请的临时变量，并不会作为返回值返回，它就是被编译器申请到栈里面。申请到栈内存好处：**函数返回直接释放，不会引起垃圾回收，对性能没有影响**。

```go
func F() []int{
	a := make([]int, 0, 20)
	return a
}
```

而上面这段代码，申请的代码一模一样，但是申请后作为返回值返回了，编译器会认为变量之后还会被使用，当函数返回之后并不会将其内存归还，那么它就会被申请到堆上面了。**申请到堆上面的内存才会引起垃圾回收**。

那么考考大家，下面这三种情况怎么解释？

```go
func F() {
	a := make([]int, 0, 20)
	b := make([]int, 0, 20000)

	l := 20
	c := make([]int, 0, l)
}
```

`a`和`b`代码一样，就是申请的空间不一样大，但是它们两个的命运是截然相反的。`a`前面已经介绍过，会申请到栈上面，而`b`，由于申请的内存较大，编译器会把这种申请内存较大的变量转移到堆上面。**即使是临时变量，申请过大也会在堆上面申请**。

而`c`，对我们而言其含义和`a`是一致的，但是**编译器对于这种不定长度的申请方式，也会在堆上面申请，即使申请的长度很短**。

可以通过下面的命令查看变量申请的位置。详细内容可以参考我之前的文章[《【译】优化Go的模式》](http://blog.cyeam.com/golang/2016/08/18/apatternforoptimizinggo)

```go
go build -gcflags='-m' . 2>&1
```

## 内存碎片化

实际项目基本都是通过`c := make([]int, 0, l)`来申请内存，长度都是不确定的。自然而然这些变量都会申请到堆上面了。Golang使用的垃圾回收算法是『标记——清除』。简单得说，就是程序要从操作系统申请一块比较大的内存，内存分成小块，通过链表链接。每次程序申请内存，就从链表上面遍历每一小块，找到符合的就返回其地址，没有合适的就从操作系统再申请。如果申请内存次数较多，而且申请的大小不固定，就会引起内存碎片化的问题。申请的堆内存并没有用完，但是用户申请的内存的时候却没有合适的空间提供。这样会遍历整个链表，还会继续向操作系统申请内存。这就能解释我一开始描述的问题，申请一块内存变成了慢语句。

## 临时对象池

如何解决这个问题，首先想到的就是对象池。Golang在`sync`里面提供了对象池`Pool`。一般大家都叫这个为对象池，而我喜欢叫它临时对象池。因为每次垃圾回收会把池子里面不被引用的对象回收掉。

> `func (p *Pool) Get() interface{}`
>
> Get selects an arbitrary item from the Pool, removes it from the Pool, and returns it to the caller. Get may choose to ignore the pool and treat it as empty. Callers should not assume any relation between values passed to Put and the values returned by Get.

需要注意的是，`Get`方法会把返回的对象从池子里面删除。所以用完了的对象，还是得重新放回池子。

很快，我写出了第一版对象池优化方案：

```go
var idsPool = sync.Pool{
	New: func() interface{} {
		ids := make([]int64, 0, 20000)
		return &ids
	},
}

func NewIds() []int64 {
	ids := idsPool.Get().(*[]int64)
	*ids = (*ids)[:0]
	idsPool.Put(ids)
	return *ids
}
```

这样的实现，是把所有`slice`都放到同一个池子里面了。为了应对变长的问题，都是按照一个较大的值申请的变量。虽然是一种优化，但是使用超大的`slice`计算，性能并没有怎么提升。

紧接着参考了达达大神的代码[sync_pool.go](https://github.com/funny/slab/blob/master/sync_pool.go)，又写了一版：

```go
var DEFAULT_SYNC_POOL *SyncPool

func NewPool() *SyncPool {
	DEFAULT_SYNC_POOL = NewSyncPool(
		5,     
		30000, 
		2,     
	)
	return DEFAULT_SYNC_POOL
}

func Alloc(size int) []int64 {
	return DEFAULT_SYNC_POOL.Alloc(size)
}

func Free(mem []int64) {
	DEFAULT_SYNC_POOL.Free(mem)
}

// SyncPool is a sync.Pool base slab allocation memory pool
type SyncPool struct {
	classes     []sync.Pool
	classesSize []int
	minSize     int
	maxSize     int
}

func NewSyncPool(minSize, maxSize, factor int) *SyncPool {
	n := 0
	for chunkSize := minSize; chunkSize <= maxSize; chunkSize *= factor {
		n++
	}
	pool := &SyncPool{
		make([]sync.Pool, n),
		make([]int, n),
		minSize, maxSize,
	}
	n = 0
	for chunkSize := minSize; chunkSize <= maxSize; chunkSize *= factor {
		pool.classesSize[n] = chunkSize
		pool.classes[n].New = func(size int) func() interface{} {
			return func() interface{} {
				buf := make([]int64, size)
				return &buf
			}
		}(chunkSize)
		n++
	}
	return pool
}

func (pool *SyncPool) Alloc(size int) []int64 {
	if size <= pool.maxSize {
		for i := 0; i < len(pool.classesSize); i++ {
			if pool.classesSize[i] >= size {
				mem := pool.classes[i].Get().(*[]int64)
				// return (*mem)[:size]
				return (*mem)[:0]
			}
		}
	}
	return make([]int64, 0, size)
}

func (pool *SyncPool) Free(mem []int64) {
	if size := cap(mem); size <= pool.maxSize {
		for i := 0; i < len(pool.classesSize); i++ {
			if pool.classesSize[i] >= size {
				pool.classes[i].Put(&mem)
				return
			}
		}
	}
}
```

调用例子：

```go
attrFilters := cache.Alloc(len(ids))
defer cache.Free(attrFilters)
```

重点在`Alloc`方法。为了能支持变长的`slice`，这里申请了多个池子，其大小是从5开始，最大到30000，以2为倍数。也就是5、10、20……

```go
DEFAULT_SYNC_POOL = NewSyncPool(
	5,     
	30000, 
	2,     
)
```

- 分配内存的时候，从池子里面找满足容量切最小的池子。比如申请长度是2的，就分配大小为5的那个池子。如果是11，就分配大小是20的那个池子里面的对象；
- 如果申请的`slice`很大，超过了上限30000，这种情况就不使用池子了，直接从内存申请；
- 当然这些参数可以根据自己实际情况调整；
- 和之前的做法有所区别，把对象重新放回池子是通过`Free`方法实现的。

## 结论

为了优化接口，前前后后搞了一年。结果还是不错的，TPS提升了最少30%，TP99也降低很多。