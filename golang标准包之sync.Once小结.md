1. 首先列出讨论的一些结论
q: 如果在任意为止对某个变量进行了原子操作，那么其他所有的位置在使用该变量的时候都应该用原子操作
a:  对

q:并发操作全局的bool 到底要不要加锁 
a:有并发的读写或者写 就要加锁, [uber的实现](https://github.com/uber-go/atomic/blob/master/atomic.go#L237)

q: bool 类型占多少字节
a: 1 byte

2. 贴出源码
``` Golang
package sync

import (
	"sync/atomic"
)

// Once is an object that will perform exactly one action.
type Once struct {
	m    Mutex
	done uint32
}

// Do calls the function f if and only if Do is being called for the
// first time for this instance of Once. In other words, given
// 	var once Once
// if once.Do(f) is called multiple times, only the first call will invoke f,
// even if f has a different value in each invocation. A new instance of
// Once is required for each function to execute.
//
// Do is intended for initialization that must be run exactly once. Since f
// is niladic, it may be necessary to use a function literal to capture the
// arguments to a function to be invoked by Do:
// 	config.once.Do(func() { config.init(filename) })
//
// Because no call to Do returns until the one call to f returns, if f causes
// Do to be called, it will deadlock.
//
// If f panics, Do considers it to have returned; future calls of Do return
// without calling f.
//
func (o *Once) Do(f func()) {
	if atomic.LoadUint32(&o.done) == 1 {
		return
	}
	// Slow-path.
	o.m.Lock()
	defer o.m.Unlock()
	if o.done == 0 {
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}
```

a: 为什么 `o.m.Lock()` 以后，还要进行原子操作 `atomic.StoreUint32(&o.done, 1)` ?

q: 

```diff
+import (
+	"sync/atomic"
+)

// Once is an object that will perform exactly one action.
type Once struct {
	m    Mutex
-	done bool
+	done int32
}

// Do calls the function f if and only if the method is being called for the
@@ -26,10 +30,14 @@ type Once struct {
// Do to be called, it will deadlock.
//
func (o *Once) Do(f func()) {
+	if atomic.AddInt32(&o.done, 0) == 1 {
+		return
+	}
+	// Slow-path.
	o.m.Lock()
	defer o.m.Unlock()
-	if !o.done {
-		o.done = true
+	if o.done == 0 {
+		f()
+		atomic.CompareAndSwapInt32(&o.done, 0, 1)
	}
}
```
刚开始用的是锁+ bool 来搞的，这个效率太低了，换成了 现在这种叫fast-path的方式 详情看 [commit](https://github.com/golang/go/commit/93dde6b0e6c0de50a47f9dc5f3ac7205c36742aa)
如果Once对象初始化以后，接下的操作都不用获取锁，因`atomic.LoadUint32(&o.done) == 1` 读的地方用了原子操作，因此写的地方也应该用原子操作。附上一个非原子写导致错误的例子
```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

func main() {
	var sum uint32 = 100

	var wg sync.WaitGroup
	for i := 0; i < 500000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			sum++ //1
			//atomic.AddUint32(&sum, 1) //2
		}()
	}
	wg.Wait()
	fmt.Println(sum)
	fmt.Println(atomic.LoadUint32(&sum))
}

```