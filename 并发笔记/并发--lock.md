### 并发--lock

##### 1、什么是从重入锁？

重入锁是为了解决死锁的问题和释放锁获得锁时导致的上下文切换的问题的，lock和sync都是支持锁重入的，因此这两种锁都可以叫做重入排他锁就是允许当前的线程重入而排斥其他的线程；

##### 2、是否存在共享锁呢，为什么存在？

在lock中是支持共享锁的也就是读写锁，在实际场景中往往是读操作多余写操作，通过读写锁的话就降低锁的一个力度；

允许多个线程同时对共享变量进行读操作，这样也减少了锁在释放和获取时的上下文切换，读写锁在数据产生变化的时候是会发生阻塞的；

##### 3、lock关系的示意图？

![img](images\menu.saveimg.savepath20200402105608.jpg)

##### **4、ReentrentLock调用关系时序图？**

![img](\images\menu.saveimg.savepath20200402111847.jpg)

##### 5、锁的基本要素

锁的状态--用一个共享变量来记录锁的状态，这个状态是被volatile修饰的；

cas就是对这个锁状态进行更改

![img](\images\menu.saveimg.savepath20200402114131.jpg)

就是检查当前内存中的值和期望值是否一致如果相同就将内存中的值更新；

这样就是通过乐观锁来竞争锁；

##### 6、ReentrantLock的核心原理--加锁？

ReentrantLock加锁的过程（以非公平锁为例）

tf：第一个线程（首个做操作的线程）

ts：第二个线程（第二个竞争锁的线程）

1、有一个线程来执行操作了

* 情况一：如果线程是tf首次获得锁将会执行compareAndSetState(0, 1)并执行成功随之就是将内存中的所状态置为同时执行setExclusiveOwnerThread(current)将当前线程置为自己的线程；

* 情况二如果当前线程是ts来了看到已经有线程在占有锁的话，则会执行 acquire(1)

```java
 final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }
```

而acquire(1)调用的就是aqs中的方法

```java
public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

而tryAcquire(arg)则是调用ReentrantLock的nonfairTryAcquire方法

2、nonfairTryAcquire方法中我们在再次对当前锁的状态进行判断

* 情况一：线程ts再次检查锁的状态，当前锁的状态是0，也就是刚刚占有锁的线程tf执行完毕了，因此当前的线程ts可以获取到锁；
* 情况二：如果当前的线程和刚刚正在执行的是同一个线程tf==ts，因为ReentrantLock是支持锁重入的因此不会阻塞，但是需要记录重入的次数，因为存在重入的操作所以内存中的锁状态可能是大于1的；
* 情况三：如果当前来竞争线程ts不满足情况一和二就只能返回fasle

```java
 final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

3、由于2中返回的false随之而来的就是执行acquireQueued(addWaiter(Node.EXCLUSIVE), arg)

调用addWaiter(Node.EXCLUSIVE)方法将当前来竞争的线程ts封装成一个node对象并存放入同步队列中；

```java
private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
```

由于当前来的是第二个来争夺锁的线程，这是同步队列还没有进行初始化呢因此没有头结点和尾节点因此会执行enq(node)方法来对同步队列进行初始化；

```java
 private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                //使用compareAndSetHead(new Node())的方式创建头结点也是为了防止并发的
                //在创建头结点的时候保证了只有一个线程是可以创建成功的
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

因为只有在else中的代码执行后才会退出循环，因此这个循环在首次的话会执行两次;

* 第一次是初始化同步队列将head和tail都执行一个空的node；

  ![](\images\menu.saveimg.savepath20200402155514.jpg)

* 第二次循环是将当前线程对应的节点插入队列的尾部；

  ![](\images\menu.saveimg.savepath20200402155936.jpg)

如果我们完成了队列初始后就会走下面的逻辑去添加同步队列了，当然如果compareAndSetTail(pred, node)失败的话还是会执行enq(node)直至添加队列成功；

```java
  if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
```

4、在完成同步队列的添加之后会运行acquireQueued(addWaiter(Node.EXCLUSIVE), arg)方法，这个方法也是aqs的；

```java
 final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

acquireQueued方法主要是再次尝试获取锁成功的话则出队列失败的话就阻塞；

在执行 if (p == head && tryAcquire(arg))判断时分为成功和失败，成功的话当前的线程就获得锁了，失败的话将继续后续的处理；

* 情况一：失败，失败的条件有两种前驱节点不是头结点和当前的线程在争夺锁的时候失败；

  ```java
   if (shouldParkAfterFailedAcquire(p, node) &&
                      parkAndCheckInterrupt())
                      interrupted = true;
  ```

  失败的情况会调用shouldParkAfterFailedAcquire(p, node)方法

  ```java
      private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
          int ws = pred.waitStatus;
          if (ws == Node.SIGNAL)
              /*
               * This node has already set status asking a release
               * to signal it, so it can safely park.
               */
              return true;
          if (ws > 0) {
              /*
               * Predecessor was cancelled. Skip over predecessors and
               * indicate retry.
               */
              do {
                  node.prev = pred = pred.prev;
              } while (pred.waitStatus > 0);
              pred.next = node;
          } else {
              /*
               * waitStatus must be 0 or PROPAGATE.  Indicate that we
               * need a signal, but don't park yet.  Caller will need to
               * retry to make sure it cannot acquire before parking.
               */
              compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
          }
          return false;
      }
  ```

  通过代码可以看出shouldParkAfterFailedAcquire主要是修改前置节点的状态，将前置节点的状态设置为SIGNAL（在锁释放后会唤醒SIGNAL的节点），同时将CANCELLED状态的节点在队列中移除；

  在shouldParkAfterFailedAcquire方法执行成功之后会调用parkAndCheckInterrupt方法

  ```java
   private final boolean parkAndCheckInterrupt() {
          LockSupport.park(this);
          return Thread.interrupted();
      }
  ```

  parkAndCheckInterrupt的主要作用就是将当前的线程挂起（阻塞），同时返回一个中断状态并将中断进行复位，由于当前线程在执行acquireQueued的方法时是没有办法处理中断响应的，因此在这个阻塞的线程被唤醒之后再对这个线程做中断响应；

经过acquireQueued方法处理后的同步队列（红色代表阻塞的线程）

![](\images\menu.saveimg.savepath20200402171530.jpg)

##### 7、ReentrantLock的核心原理--解锁？

ReentrantLock的unLock会调用aqs中的sync.release(1)方法

```java
public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

aqs的release方法会调用ReentrantLock的tryRelease（）方法

```java
protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```

由于ReentrantLock是可重入的，因此我们在释放锁的过程中会将重入的线程依次释放直至锁状态为0时将当前执行的线程置为空同时返回当前锁释放完毕的标志；

再次来到release方法if (h != null && h.waitStatus != 0)这个代码是验证成功的，因为我们在acquireQueued方法中已经将所有的前置节点的状态都置为了SIGNAL就是-1，判断成功后调用unparkSuccessor方法唤醒线程；

```java
    private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```

在唤醒线程的过程中会将头结点（head）的状态改为0，由于头结点的下一个节点的状态是SIGNALif (s == null || s.waitStatus > 0)判断失败，执行LockSupport.unpark(s.thread)唤醒头结点的下一个节点；

注意我们唤醒之后的线程不是去做其他操作而是在他挂起的位置开始继续执行操作也就说被唤醒的是在acquireQueued方法中继续运行的；

再看看acquireQueued这个方法

```java
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

线程被唤醒之后再次陷入循环当执行到if (p == head && tryAcquire(arg))这行代码时，可以获得锁了（前提是没有其他的线程在中间插队由于我们讨论的非公平锁），也就是说if判断是成功的所以执行if中的代码，将head节点的下个节点设置为head节点，同时将原有的head节点从同步队列中移除；

![](\images\menu.saveimg.savepath20200402184728.jpg)

##### 8、ReentrantLock的核心原理--对中断的处理？

```java
Thread.interrupted();//获取中断状态，以及复位
Thread.interrupt();//中断线程
```

因为在线程竞争锁发生自旋的过程中线程并没有对中断操作做出响应，因此在线程被唤醒的时候如果parkAndCheckInterrupt方法返回的是true则将interrupted置为true，因为现在这个线程获得了锁所以可以去执行操作，因此acquireQueued方法返回的是interrupted也就是true；

```java
public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

因此 if (!tryAcquire(arg) &&acquireQueued(addWaiter(Node.EXCLUSIVE), arg))判断为true执行selfInterrupt方法，selfInterrupt方法就是对中断做出的响应；

```java
 static void selfInterrupt() {
        Thread.currentThread().interrupt();
    }
```

##### 9、ReentrantLock的核心原理--一个设计技巧？

在唤醒的过程中也存在清除CANCELLED状态的节点，就在unparkSuccessor方法中

```java
  Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
```

我们这里清除CANCELLED状态的节点的节点为什么要从尾部开始循环呢，而不是从头循环呢？

原因是因为我们在添加节点到同步队列的时候也有出现同时唤醒的过程，比如enq方法添加节点到同步队列中是分为三步的首先node.prev = t（当前线程的节点的前驱节点指向原来的尾节点）然后compareAndSetTail(t, node)（将当前节点置为尾节点）最后t.next = node（将原有尾节点的后继节点指向当前线程节点）

![](\images\menu.saveimg.savepath20200402191929.jpg)

在第三步操作的时候可能会慢于线程唤醒的过程，因此这个队列从前向后寻找的时候会对开，因此采用从后向向去遍历；

##### 10、ReentrantLock的公平锁与非公平锁

NonfairSync的lock方法

```java
 final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }
```

FairSync的lock方法

```java
 final void lock() {
            acquire(1);
        }
```

从lock方法上看NonfairSync（非公平）允许刚刚到来的新线程做cas操作获取锁，而FairSync（公平锁）是不允许刚刚到来的线程去插队的；

NonfairSync的tryAcquire方法

```java
protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
        
final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

FairSync的tryAcquire方法

```java
 protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

在公平锁中比非公平锁多了!hasQueuedPredecessors()这个判断条件

```java
public final boolean hasQueuedPredecessors() {
        // The correctness of this depends on head being initialized
        // before tail and on head.next being accurate if the current
        // thread is first in queue.
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }
```

如果当前有其他的线程在阻塞的话是不能进行 compareAndSetState(0, acquires)操作的，必须去排队；

##### 11、CANCELLED状态在什么时候触发？

cancelAcquire是用来标记CANCELLED状态的，基本上是在获取锁时候发生异常在finally里面调用cancelAcquire方法来标记当前线程的node的状态的；

