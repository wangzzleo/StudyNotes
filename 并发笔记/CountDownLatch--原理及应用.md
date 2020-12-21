### CountDownLatch--原理及应用

##### 1、CountDownLatch的作用？

允许一个或者多个线程一直等待直到其他线程执行完毕之后在进行执行操作；

##### 2、CountDownLatch的使用？

demo1：一个线程等待多个线程执行之后再执行

```java
package countDownLatchDemo;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

public class CountDownLatchDemo1 {
	public static void main(String[] args) {
        System.out.println("main---start");
		final CountDownLatch countDownLatch=new CountDownLatch(3);
		new Thread(new Runnable() {
			
			public void run() {
				// TODO Auto-generated method stub
				System.out.println("Thread1--start");
				try {
					Thread.sleep(2000);
				} catch (InterruptedException e) {
					// TODO Auto-generated catch block
					e.printStackTrace();
				}
				System.out.println("Thread1--end");
				countDownLatch.countDown();
			}
		}).start();
		new Thread(new Runnable() {
			
			public void run() {
				// TODO Auto-generated method stub
				System.out.println("Thread2--start");
				try {
					Thread.sleep(2000);
				} catch (InterruptedException e) {
					// TODO Auto-generated catch block
					e.printStackTrace();
				}
				System.out.println("Thread2--end");
				countDownLatch.countDown();
			}
		}).start();
		new Thread(new Runnable() {
			
			public void run() {
				// TODO Auto-generated method stub
				System.out.println("Thread3--start");
				try {
					Thread.sleep(2000);
				} catch (InterruptedException e) {
					// TODO Auto-generated catch block
					e.printStackTrace();
				}
				System.out.println("Thread3--end");
				countDownLatch.countDown();
			}
		}).start();
		try {
			countDownLatch.await();
		} catch (InterruptedException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		System.out.println("main---end");
	}
}

```

demo2：多个线程等待一个线程执行完毕

```java
package countDownLatchDemo;

import java.util.concurrent.CountDownLatch;

public class CountDownLatchDemo2 extends Thread{

	static CountDownLatch countDownLatch=new CountDownLatch(1);
	
	public void run() {
		// TODO Auto-generated method stub
		try {
			countDownLatch.await();
		} catch (InterruptedException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		System.out.println("ThreadName:"+Thread.currentThread().getName());
		System.out.println("对文件进行操作");
	}
	
	public static void main(String[] args) {
		for(int i=0;i<3;i++){
			new CountDownLatchDemo2().start();
		}
		try {
			System.out.println("读取文件....");
			Thread.sleep(2000);
		} catch (InterruptedException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		System.out.println("读取文件完毕");
		countDownLatch.countDown();
	}
	
}

```

##### 3、CountDownLatch的await方法？

CountDownLatch使用的是aqs中的共享锁

1、CountDownLatch的await方法

```java
 public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
```

调用aqs（同步器）中的acquireSharedInterruptibly方法

2、acquireSharedInterruptibly表示共享的可中断

```java
 public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }
```

3、tryAcquireShared(arg) 方法判断当前的计数器是否等于0

```java
 protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }
```

在CountDownLatch中state代表的是当前计数器的个数，这样也印证了共享锁不是互斥的因此我们不需要对当前的锁的状态进行记录；

CountDownLatch的构造方法

```java
 public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }
```

CountDownLatch内部的aqs（同步器的构造方法）

```java
Sync(int count) {
            setState(count);
        }
```

通过CountDownLatch我们可以看出CountDownLatch中的state表示的就是计数器的个数；

如果tryAcquireShared(arg)判断不成功成功则不会执行doAcquireSharedInterruptibly(arg);

4、当tryAcquireShared(arg)判断成功执行doAcquireSharedInterruptibly(arg)

```java
    private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

doAcquireSharedInterruptibly(arg)方法和共享锁的方法大致是一样的；

首先会初始化aqs的同步队列同时将执行await方法的线程封装成node添加到aqs的同步队列中去

![](\images\menu.saveimg.savepath20200411163954.jpg)

接下来对aqs同步队列中的节点进行自旋操作去竞争锁调用tryAcquireShared(arg)方法竞争锁，由于当前的计数器的个数不等于0，一次会失败，失败的话会执行if (shouldParkAfterFailedAcquire(p, node) &&
parkAndCheckInterrupt())操作，因此处于于aqs同步队列中的线程应该出于阻塞的状态；

![](\images\menu.saveimg.savepath20200411164853.jpg)

在添加队列完毕时每个节点的前置节点的状态是会改变的

![](\images\menu.saveimg.savepath20200413100210.jpg)

##### 4、CountDownLatch的countDown方法？

CountDownLatch的countDown方法会代用aqs的releaseShared方法来释放

```java
 public void countDown() {
        sync.releaseShared(1);
    }
```

1、releaseShared方法来释放

```java
public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```

2、首先通过tryReleaseShared方法进行判断，而tryReleaseShared方法是通过自旋的方式对当前的计数器执行cas的操作将计数器的个数进行减一操作，如果减一操作执行后计数器的个数仍然不是0则返回false；

```java
protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
```

如果tryReleaseShared方法返回false则releaseShared方法也执行完毕；

如果tryReleaseShared方法返回true则执行doReleaseShared()方法唤醒在aqs同步队列中的线程；

3、doReleaseShared方法来唤醒在aqs同步队列中的线程，释放共享锁

```java
    private void doReleaseShared() {
        /*
         * Ensure that a release propagates, even if there are other
         * in-progress acquires/releases.  This proceeds in the usual
         * way of trying to unparkSuccessor of head if it needs
         * signal. But if it does not, status is set to PROPAGATE to
         * ensure that upon release, propagation continues.
         * Additionally, we must loop in case a new node is added
         * while we are doing this. Also, unlike other uses of
         * unparkSuccessor, we need to know if CAS to reset status
         * fails, if so rechecking.
         */
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
```

当前的aqs同步队列中的节点是这样的

![](\images\menu.saveimg.savepath20200413100210.jpg)

doReleaseShared就是通过自旋的方式将aqs同步队列中节点的状态由SIGNAL（-1）改成0然后调用unparkSuccessor方法唤醒当前节点对应的线程；

当前ThreadA线程被唤醒了，由于ThreadA被唤醒了所以ThreadA会从await阻塞的方法中继续执行操作也就再次来到了doAcquireSharedInterruptibly方法

```java
 private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

由于是parkAndCheckInterrupt方法是ThreadA挂起的，当ThreadA被唤醒之后重返doAcquireSharedInterruptibly的自旋，由于当前计数器state已经等于0，因此tryAcquireShared(arg)=1

所以会执行setHeadAndPropagate(node, r)方法

```java
    private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        setHead(node);
        /*
         * Try to signal next queued node if:
         *   Propagation was indicated by caller,
         *     or was recorded (as h.waitStatus either before
         *     or after setHead) by a previous operation
         *     (note: this uses sign-check of waitStatus because
         *      PROPAGATE status may transition to SIGNAL.)
         * and
         *   The next node is waiting in shared mode,
         *     or we don't know, because it appears null
         *
         * The conservatism in both of these checks may cause
         * unnecessary wake-ups, but only when there are multiple
         * racing acquires/releases, so most need signals now or soon
         * anyway.
         */
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }
```

setHeadAndPropagate(node, r)主要是将当前ThreadA对应的节点设置为头结点，

![](\images\menu.saveimg.savepath20200413102904.jpg)

if (propagate > 0 || h == null || h.waitStatus < 0 || (h = head) == null || h.waitStatus < 0)判断条件是成立的因此会执行后续代码，再判断ThreadA的下一个节点是不是SHARED状态的如果是就执行 doReleaseShared()，这样就完成当计数器为0的时候将所有出于aqs同步队列中的出于共享状态中的节点对应的线程全部唤醒；

![](\images\menu.saveimg.savepath20200413103329.jpg)

