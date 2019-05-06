1. mutex是非重入锁，因为它不与goroutine绑定，所以无法检测当前请求锁的goroutine是否是已经获取锁的那个goroutine。
2. unlock锁时，如果锁不是lock状态，会panic。
3. 写锁之间互斥， 读锁之间不互斥，读写锁之间互斥。
4. 请求解锁的goroutine将进入一个FIFO的队列，锁可用时被唤醒时并一起竞争锁。
5. 对于写锁，正常状态下，请求锁是公平的，不会区分先来后到，但是一个goroutine等待超过1ms时，锁将进入饥饿状态，此状态下，获取锁将不再公平，直接由队列中的第一个goroutine获取锁，直到某个获取到锁的goroutine等待时间小于1ms或者等待队列为空。
6. 等待队列中同时存在写锁和读锁时，写锁会优先获取锁，所以禁止在代码中使用递归锁，以防死锁。递归锁的示例代码：
```
l
package main

import "sync"

var rwm sync.RWMutex

func main() {
	ch := make(chan int)
	go func() {
		for {
			rloop() //递归锁
		}
	}()
	go func() {
		for {
			//第一个读锁获取到后如果产生写锁请求，将导致第二个读锁请求阻塞
			//进而无法执行rloop中的defer语句解锁，产生死锁
			loop()
		}
	}()
	<-ch
}

func rloop() {
	rwm.RLock()
	defer rwm.RUnlock()
	rwm.RLock()
	rwm.RUnlock()
}

func loop() {
	rwm.Lock()
	rwm.Unlock()
}

```
