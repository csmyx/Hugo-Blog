+++
title = 'Condition Variable'
date = 2023-12-11T14:34:35+08:00
draft = false
+++
#### 条件变量

```Go
type Cond 
func NeWCond(l Locker) *Cond 
func (c *Cond) Broadcast() 
func (c *Cond) Signal() 
func (c *Cond) Wait()

ready := false
c := sync.NewCond(&sync.Mutex{})

func condition() bool {
  return ready
}
func consume() {
  for {
      c.L.Lock()
      for !condtion() { // A:
        c.Wait() // B
      }
      // consumming...
      c.L.Unlock()  
  }
}

func produce1() {
  // producing...
  c.L.Lock()
  ready = true // C
  c.signal()   // D：在临界区内signal
  c.L.Unlock()  
}

func produce2() {
  // producing...
  c.L.Lock()
  ready = true // C:使得condition成立的逻辑语句
  c.L.Unlock()  
  c.signal() // D：在临界区外signal
}

func produce3() {
  // producing...
  ready = true // C
  c.L.Lock()
  c.signal()   // D：在临界区内signal
  c.L.Unlock()  
}


func produce4() {
  // producing...
  c.L.Lock()
  if (ready == false) {
    c.signal()   // D：在临界区内signal        
  }
  ready = true // C
  c.L.Unlock()  
}

go consume() // consumer
go produce() // producer
```

条件判断 A和 wait调用B 需要加锁保护的原因：保证条件判断语句和 wait语句同时执行的原子性。否则：

- 不加锁的话，若 consumer 执行完A，还未执行 B 时，切换到 producer 执行 D(先signal) 后，再切换到consum执行 B(后wait)，则会导致 **lost wake up** 问题（即signal在wait之前发生，从而导致无法在条件满足时及时唤醒wait的线程）。

避免lost wake up问题，本质是为了 **避免出现如下执行顺序：刚好在consumer执行完条件判断(A)后，producer执行signal(D)，然后consumer再执行wait(B)，即D在A和B之间执行**。需要的条件有：

1. 光加锁保证 consumer 中条件判断和wait执行的原子性还不够，

2. 同时还要对producer 修改条件成立逻辑(逻辑C) 前后也加锁。这里加锁还有一个原因是为了保护共享变量ready，但避免lost wakeup问题才是本质原因，虽然修改条件成立的逻辑通常要使用到共享变量，但哪怕没有共享变量也要加锁。

3. 若采用produce2的方案（在临界区外signal）则还要有一个隐形条件，即条件成立逻辑(C)之后必须要调用signal(D)。有一个常见错误，即颠倒C和D的顺序导致死锁，就是违背了该条件。

   因为此时只保证了C不会在A和B之间，也就是说C：1.要么在A之前，2. 要么在B之后执行。前者导致consumer执行A时条件满足，不会睡眠，就不需要signal来唤醒了（可能有虚假唤醒，定义见下文）；后者的话consumer执行A时条件不满足，就必须要signal来唤醒，若我们保证D在C之后，就能保证D在AB之后，从而满足条件。即**加锁避免C在AB之间执行+D在C之后执行 => 避免D在AB之间执行**。

   这里有引出一个问题，既然本质目的是为了：避免D在AB之间执行，C在AB之间执行不影响正确性。那可不可以在producer中采用对D加锁+保证CD顺序，而要对C加锁+保证CD顺序 来间接实现呢 (如produce3所示)。当然采用produce1对CD同时加锁当然可以，问题是如果只对D加锁，不锁C可不可保证正确性？问题是如前文所述，C通常要访问共享变量，因此也加锁。但我觉得理论是可以只对signal加锁的，**我觉得Linux futex的机制从某种角度来说，就是对signal加锁。**

signal(D)不一定需要被加锁保护，只要满足上诉条件，加不加锁两者的正确性都能保证（即都不会导致死锁），produce1和produce2都可以。

从性能上考虑，各有优劣：

1. 在临界区外signal的方案的劣势是：可能会导致 spurious wakeup（假唤醒），试想如下执行顺序：producer先执行逻辑C，consumer再执行AB，此时consumer条件满足不进入睡眠，然后producer执行signal，可能唤醒之前睡眠的consumer，由于有consumer已经满足条件进行消费了，后被唤醒的consumer可能不再满足条件，这就是所谓假唤醒。但如前文所述，假唤醒不会破坏正确性，即使采用临界区内signal的方案也可能会因为调度器调度，假唤醒未满足条件的线程。不唤醒会导致死锁，而假唤醒不会。
2. 在临界区内signal的方案的劣势是：consumer被唤醒时需重新持有锁，而此时producer可能还未释放锁，需等待。貌似有优化，见参考资料2评论

不考虑性能的话，在临界区内signal的方案有一个额外好处就是即使颠倒了C和D的顺序也不会破坏正确性(如produce4对produce2的优化就利用到该性质，避免了虚假唤醒)，看上去违背了条件3，但实际上由于临界区执行的原子性，此时仍能避免lost wakeup。

- 参考资料
  1. [Go 条件变量用法](https://github.com/CodeFish-xiao/go_concurrent_notes/blob/master/1.%E5%9F%BA%E6%9C%AC%E5%B9%B6%E5%8F%91%E5%8E%9F%E8%AF%AD/1.07%EF%BC%9ACond%EF%BC%9A%E6%9D%A1%E4%BB%B6%E5%8F%98%E9%87%8F%E7%9A%84%E5%AE%9E%E7%8E%B0%E6%9C%BA%E5%88%B6%E5%8F%8A%E9%81%BF%E5%9D%91%E6%8C%87%E5%8D%97/07.00-Cond%EF%BC%9A%E6%9D%A1%E4%BB%B6%E5%8F%98%E9%87%8F%E7%9A%84%E5%AE%9E%E7%8E%B0%E6%9C%BA%E5%88%B6%E5%8F%8A%E9%81%BF%E5%9D%91%E6%8C%87%E5%8D%97.md)
  2. [为什么条件变量和互斥锁总是搭配着使用](https://zhuanlan.zhihu.com/p/55123862)
  3. [C++ 条件变量的使用](https://www.youtube.com/watch?v=13dFggo4t_I)
  4. [条件变量范式](https://blog.wangzhl.com/posts/why-condition-variable-requires-mutex/)
  5. [why the lock argument to sleep()?](https://pdos.csail.mit.edu/6.S081/2020/lec/l-coordination.txt)
  6. 《xv6-riscv》7.6 Code: Sleep and wakeup  
  7. [Go cond源码](https://github.com/golang/go/blob/master/src/sync/cond.go)

