### Semaphore--原理及应用

##### 1、Semaphore应用--限流

demo：

```java
package SemaphoreDemo;

import java.util.concurrent.Semaphore;

public class SemaphoreDemo {
	//AQS(限流)
	//permits;令牌
	//公平和非公平
	static class Car extends Thread{
		private int num;
		private Semaphore semaphore;
		public Car(int num,Semaphore semaphore) {
			this.num=num;
			this.semaphore=semaphore;
		}
		public void run(){
			try {
				semaphore.acquire();//获得一个令牌，如果拿不到令牌就会阻塞
				System.out.println(num+"号车抢占车位....");
				Thread.sleep(5000);
				System.out.println(num+"号车离开车位...");
				semaphore.release();
			} catch (InterruptedException e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			}
		}
	}
	public static void main(String[] args) {
		Semaphore semaphore=new Semaphore(5);
		for(int i=0;i<10;i++){
			new Car(i, semaphore).start();
		}
	}
}

```

##### 2、Semaphore的acquire方法

```java
public void acquire() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
```

Semaphore的acquire方法就是代用sync的共享锁的方法也就一种共享锁的实现

```java
public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }

```

来看看Semaphore对tryAcquireShared(arg)方法的实现，以非公平锁为例

```java
protected int tryAcquireShared(int acquires) {
            return nonfairTryAcquireShared(acquires);
        }
```

调用非公平到的nonfairTryAcquireShared方法

```java
final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
```

这里getState()方法是获取当前令牌的数量，注意Semaphore中的state代表的令牌的数量；

通过Semaphore的构造器我们可以看出Semaphore默认是创建一个非公平的锁并将令牌数赋值给state；

```java
public Semaphore(int permits) {
        sync = new NonfairSync(permits);
    }
```

nonfairTryAcquireShared方法通过自旋的方式对当前的令牌数进行减一操作并加以判断，如果当前的令牌数为0则放回-1，也就意味着tryAcquireShared方法返回-1，因此在acquireSharedInterruptibly方法中的tryAcquireShared(arg) < 0判断成立，进入doAcquireSharedInterruptibly(arg)方法；

3、doAcquireSharedInterruptibly(arg)方法和CountDownLatch的相同，作用就是初始化aqs的同步队列并利用自旋的方式将未获得令牌的线程封装成类型为SHARED的node添加到初始化好的aqs等待队列中，并将每个节点的前置节点的状态修改为SIGNAL（-1）同时完成对线程的阻塞操作；

##### 3、Semaphore的release方法

```java
 public void release() {
        sync.releaseShared(1);
    }
```

Semaphore的release调用sync的releaseShared方法

```java
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```

releaseShared方法中的tryReleaseShared(arg)是利用自旋的方式对当前的令牌进行加一操作，并将加一后的令牌更新为原有令牌；

```java
        protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current + releases;
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next))
                    return true;
            }
        }
```

如果令牌更新成功则tryReleaseShared返回true，因此会执行releaseShared方法中的doReleaseShared方法

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

doReleaseShared和CountDownLatch中的相同，就是通过自旋的方式更新头结点并唤醒相应的线程，由于线程被唤醒因此会再次来到线程阻塞的位置也就是doAcquireSharedInterruptibly方法，由于当前的令牌数量修改为加一后的数量因此tryAcquireShared（arg）=0，所以会执行setHeadAndPropagate方法在setHeadAndPropagate还会调用doReleaseShared方法这样就将aqs中所有是共享状态的节点对应的线程全部唤醒；

4、公平和非公平区别

非公平的tryAcquireShared方法

```java
        protected int tryAcquireShared(int acquires) {
            return nonfairTryAcquireShared(acquires);
        }
        
                final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
```

公平的tryAcquireShared方法

```java
        protected int tryAcquireShared(int acquires) {
            for (;;) {
                if (hasQueuedPredecessors())
                    return -1;
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
```

公平与非公平的差别就是，在公平锁中会调用hasQueuedPredecessors判断当前aqs的同步队列中是否存在节点

如果存在就直接返回true从而导致tryAcquireShared放回-1，因此会直接调用doAcquireSharedInterruptibly将线程封装成node节点添加到aqs等待队列中；

所谓的公平与非公平就是否允许插队操作；