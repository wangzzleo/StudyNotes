### CyclicBarrier--原理及应用

##### 1、CyclicBarrier应用--循环屏障

demo：

```java
package cycliBarrierDemo;

import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;

public class DataImportThread extends Thread{
	private CyclicBarrier cyclicBarrier;
	private String path;
	
	public DataImportThread(CyclicBarrier cyclicBarrier,String path) {
		this.cyclicBarrier=cyclicBarrier;
		this.path=path;
	}
	
	@Override
	public void run() {
		System.out.println("开始导入数据："+path+"数据");
		try {
			Thread.sleep(2000);
		} catch (InterruptedException e1) {
			// TODO Auto-generated catch block
			e1.printStackTrace();
		}
		System.out.println(path+"数据导入完毕");
		try {
			cyclicBarrier.await();
		} catch (InterruptedException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		} catch (BrokenBarrierException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
	}

}


package cycliBarrierDemo;

import java.util.concurrent.CyclicBarrier;

public class CycliBarrierDemo extends Thread {
	//循环屏障
	//可以使得一组线程达到一个同步点之前阻塞
	@Override
	public void run() {
		System.out.println("开始数据分析");
	}
	public static void main(String[] args) {
		CyclicBarrier cyclicBarrier=new CyclicBarrier(3,new CycliBarrierDemo());
		for(int i=1;i<=3;i++){
			new DataImportThread(cyclicBarrier, "file"+i).start();
		}
	}
}

```

##### 2、CyclicBarrier的构造方法

两个参数的构造方法

```java
 public CyclicBarrier(int parties, Runnable barrierAction) {
        if (parties <= 0) throw new IllegalArgumentException();
        this.parties = parties;
        this.count = parties;
        this.barrierCommand = barrierAction;
    }

```

在CyclicBarrier中有ReentrantLock和Condition两个属性

```java
/** The lock for guarding barrier entry */
    private final ReentrantLock lock = new ReentrantLock();
    /** Condition to wait on until tripped */
    private final Condition trip = lock.newCondition();
```

##### 3、CyclicBarrier的await方法

CyclicBarrier的await会调用自身的dowait方法

```java
public int await() throws InterruptedException, BrokenBarrierException {
        try {
            return dowait(false, 0L);
        } catch (TimeoutException toe) {
            throw new Error(toe); // cannot happen
        }
    }
```

dowait方法有如下几个作用

* 如果是当前的线程被中断的话就会调用breakBarrier()去将condition等待队列中的节点迁移到aqs同步队列中去并对节点进行唤醒操作；
* 如果没有被中断话则继续执行代码通过对count进行减一的操作如果当前屏障的个数不是0则启用自旋的方式调用condition的await方法将线程封装成node并将node加入condition的等待队列中；
* 如果当前的屏障的个数为执行减一之后如果为0则会执行command.run()就是我们构造器中传入的任务，还会执行nextGeneration会唤醒所有的线程；

```java
 private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        final ReentrantLock lock = this.lock;
        lock.lock();//获取ReentrantLock互斥锁
        try {
            final Generation g = generation;//获取generation对象

            if (g.broken)//如果generation损坏，抛出异常
                throw new BrokenBarrierException();

            if (Thread.interrupted()) {
                //如果当前线程被中断，则调用breakBarrier方法，停止CyclicBarrier，并唤醒所有线程
                breakBarrier();
                throw new InterruptedException();
            }

            int index = --count;// 看到这里了吧，count减1 
            //index=0，也就是说，有0个线程未满足CyclicBarrier条件，也就是条件满足，
            //可以唤醒所有的线程了
            if (index == 0) {  // tripped
                boolean ranAction = false;
                try {
                   //这就是构造器的第二个参数，如果不为空的话，就执行这个Runnable的run方法，
                   //你看，这里是执行的是run方法，也就是说，并没有新起一个另外的线程，
                   //而是最后一个执行await操作的线程执行的这个run方法。
                    final Runnable command = barrierCommand;
                    if (command != null)
                        command.run(); //同步执行barrierCommand
                    ranAction = true;
                    nextGeneration(); //执行成功设置下一个nextGeneration
                    return 0;
                } finally {
                    if (!ranAction) . //如果barrierCommand执行失败，进行屏障破坏处理
                        breakBarrier();
                }
            }
            //如果当前线程不是最后一个到达的线程
            // loop until tripped, broken, interrupted, or timed out
            for (;;) {
                try {
                    if (!timed)
                        trip.await(); //调用Condition的await()方法阻塞
                    else if (nanos > 0L)
                        nanos = trip.awaitNanos(nanos); //调用Condition的awaitNanos()方法阻塞
                } catch (InterruptedException ie) {
                //如果当前线程被中断，则判断是否有其他线程已经使屏障破坏。若没有则进行屏障破坏处理，并抛出异常；否则再次中断当前线程
                    if (g == generation && ! g.broken) {
                        breakBarrier();//执行breakBarrier，唤醒所有线程
                        throw ie;
                    } else {
                        // We're about to finish waiting even if we had not
                        // been interrupted, so this interrupt is deemed to
                        // "belong" to subsequent execution.
                        Thread.currentThread().interrupt();
                    }
                }

                if (g.broken)//如果当前generation已经损坏，抛出异常
                    throw new BrokenBarrierException();

                if (g != generation)//如果generation已经更新换代，则返回index
                    return index;
                //如果是参数是超时等待，并且已经超时，则执行breakBarrier()方法
                //唤醒所有等待线程。
                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock();
        }
    }
```



##### 5、总结

CyclicBarrier主要依靠的就是重入锁和condition