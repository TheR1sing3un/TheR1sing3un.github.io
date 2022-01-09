# Java实现生产者消费者模型


# Java实现生产者消费者模型

## 前言

> 什么是生产者消费者模型？

简单来讲就是有两种线程，分别称为生产者线程和消费者线程。生产者线程生产出"产品"放置到公共的一个队列中，然后消费者线程从队列中去取该"商品"。这就是该模型的简单描述。

> 异步和解耦

该模型实现了生产者和消费的**异步**和**解耦**。生产者只需要生产出"产品"放到不满的队列中就可以，并不需要关心是谁来消费，可能是小红可能是大黄，也不需要等待消费者消费完了之后再接着生产。消费者也不需要关心"产品"是谁生产的，只要队列里面有"产品"，我就可以拿去用，不用等生产者一个个生产。

> 那该怎么实现？

总结一下该模型的几点需要实现的点：

- 生产者和消费者线程需要通信
- 需要一个线程安全的队列来存放"产品"

**那么我们可以根据所学过的线程间通信的方法来选择**

*线程通信方式*

- `volatile`关键字保证共享变量在线程间的可见性

- 基于`synchronized`锁的`wait`/`notify`的等待通知机制
- 基于AQS并发包的`ReentrantLock`等并发工具
- 管道流

*保证线程安全对立*

- 使用线程不安全的队列，但是对其访问进行加锁
- 使用线程安全的队列

**那么可以总结出以下几种方法**

1. 基于`synchronized`锁的`wait`/`notify`的等待通知机制 + 线程不安全的队列
2. 基于AQS并发包的`Lock`和`Condition`的条件等待机制 + 线程不安全的队列
3. 基于`BlockingQueue`阻塞队列的入列和出列机制

---

## synchronized

先讲基于`synchronized`的等待通知机制。

> synchronized锁

当我们对`queue`加上`synchronized`锁之后，我们调用方法`queue.wait`就会在该锁的一个叫做`waitSet`的等待队列上进行等待，直到有别的线程调用`notify`/`notifyAll`(`notify`则进行随机唤醒)。那么我们就可以得到如下的流程图。

![image-20220109211436965](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20220109211436965.png "使用synchronized锁的生产者消费者模型")

每个线程大致的步骤是：

1. 线程运行到获取的锁的位置
2. 尝试获取锁，若获取失败，则在同步队列中继续获取，回到步骤1。若成功则拿到锁则到步骤3。
3. 获取锁之后，尝试将产品加入到`queue`/从`queue`拿出产品。若不符合条件(生产者发现队列满了，消费者发现队列为空)，则加入到该锁的等待队列waitSet中，并且让出该锁，到步骤4。若符合条件则正常操作，并且唤醒所有等待在该锁的waitSet上的线程，也就是`notifyAll`，并且让出该锁、回到步骤1。
4. 在waitSet等待唤醒，若被唤醒则从到锁的同步队列中继续尝试获取锁，若获取成功则到直接到操作队列步骤。失败则继续尝试获取锁。

### 代码实例

废话不多说，直接上代码。

> 定义

- 生产者：新手程序员
- 消费者：老手程序员
- 产品：Bug
- 对立：系统

*新手程序员作为Bug的生产者来生产Bug到系统中，由老手程序员作为消费者来从系统中找到Bug并修复*

> Bug类

```java
/**
 * @author TheR1sing3un
 * @date 2022/1/9 17:18
 * @description Bug类
 */

public class Bug {

    private Integer bugId;

    public Bug(Integer bugId){
        this.bugId = bugId;
    }

    public Integer getBugId() {
        return bugId;
    }
}
```

> Producer类

```java
import java.util.Queue;
import java.util.Random;

/**
 * @author TheR1sing3un
 * @date 2022/1/9 17:03
 * @description
 */

public class Producer extends Thread {

    private String name;

    private int maxSize;

    private Queue<Bug> queue;

    /**
     * 构造方法
     * @param name
     * @param queue
     * @param maxSize
     */
    public Producer(String name,Queue<Bug> queue,int maxSize){
        this.name = name;
        this.maxSize = maxSize;
        this.queue = queue;
    }

    /**
     * 重写run方法,来不断生产bug到队列中
     */
    @Override
    public void run() {
        Random random = new Random();
        while(true){
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            //生产者不断生产bug
            Bug bug = this.produceBug(random.nextInt());
            //模拟耗时操作

            //加锁,避免并发问题
            synchronized (this.queue){
                while(queue.size() == maxSize){
                    //当前的队列已满,无法将bug放进去,那么就等待,直到被唤醒(消费者会来唤醒的)
                    try {
                        System.out.println("[" + name + "]: 当前系统的Bug达到上限,歇会儿,不满的时候跟俺说一声~");
                        queue.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                //现在可以把bug放进去了
                queue.add(bug);
                System.out.println("[" + name + "]: 俺往系统里面放了一个Bug" + bug.getBugId() + ",嘿嘿~嘿嘿嘿~");
                //唤醒在睡眠的消费者(当时那些消费者消费的时候发现队列为空,就sleep去了,现在我刚放进去一个bug,队列肯定不为空,所以唤醒他们)
                queue.notifyAll();
            }
        }
    }

    /**
     *
     * 生产者生产一个带编号的Bug(可真是和我一模一样呢)
     * @param i
     * @return
     */
    public Bug produceBug(int i){
        Bug bug = new Bug(i);
        return bug;
    }
}
```

> Cunsumer类

```java
import java.util.Queue;

/**
 * @author TheR1sing3un
 * @date 2022/1/9 17:32
 * @description 消费者
 */

public class Consumer extends Thread{

    private String name;

    private Queue<Bug> queue;

    public Consumer(String name, Queue<Bug> queue) {
        this.name = name;
        this.queue = queue;
    }

    /**
     * 重写run方法,消费者从队列中拿Bug,然后去修复(消费者就是修复Bug的可怜程序员)
     */
    @Override
    public void run() {
        while (true){
            //操作队列就要上锁!
            synchronized (queue){
                while (queue.isEmpty()){
                    //当队列中没有Bug时,就等待
                    try {
                        System.out.println("[" + name + "]: 这系统做的可以,咋没Bug,整挺好,我摸鱼去了,有Bug叫我");
                        queue.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                //有bug时取出
                Bug bug = queue.poll();
                //模拟耗时操作
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                fixBug(bug);
                //唤醒生产者线程
                queue.notifyAll();
            }
        }
    }

    /**
     * 消费者修复Bug
     * @param bug
     */
    public void fixBug(Bug bug){
        System.out.println("[" + name + "]: 修复了Bug" + bug.getBugId());
    }
}
```

> 测试

```java
import java.util.LinkedList;
import java.util.Queue;

/**
 * @author TheR1sing3un
 * @date 2022/1/9 17:41
 * @description
 */

public class Test {
    public static void main(String[] args) {
        Queue<Bug> queue = new LinkedList<>();
        int maxSize = 3;
        for (int i = 0; i < 5; i++) {
            new Producer("新手程序员"+i+"号",queue,maxSize).start();
        }
        for (int i = 0; i < 2; i++) {
            new Consumer("老手程序员"+i+"号",queue).start();
        }
    }
}
```

*测试截图如下：*

![image-20220109220149471](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20220109220149471.png)

---

## AQS并发包

方案二采取AQS并发包中的`Lock`和`Condition`的条件等待来实现。

**加锁步骤和方案一差不多，只需要将`queue.wait`改成相应的条件等待，以及唤醒改成条件唤醒即可**

> 生产者

- 生产者尝试往队列里面放的时候若队列是满的，则在条件变量noFull上等队列不为满的时候。
- 若生产者成功将元素放进队列，那么此时队列一定不为空，所以唤醒在noEmpty条件变量上等待的消费者。

> 消费者

- 消费者尝试从队列里面取的时候若队列为空，则在条件变量noEmpty上等到队列不为空的时候。
- 若消费者成功从队列取到元素，那么此时队列一定不是满的，所以唤醒在noFull条件变量上等待的生产者。

### 代码实例

> Bug类

```java
package ProAndCon;

/**
 * @author TheR1sing3un
 * @date 2022/1/9 17:18
 * @description Bug类
 */

public class Bug {

    private Integer bugId;

    public Bug(Integer bugId){
        this.bugId = bugId;
    }

    public Integer getBugId() {
        return bugId;
    }
}
```

> Producer类

```java
package ProAndCon;


import javax.security.auth.login.Configuration;
import java.util.Queue;
import java.util.Random;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;

/**
 * @author TheR1sing3un
 * @date 2022/1/9 22:03
 * @description 生产者类
 */

public class Producer extends Thread{

    private String name;

    private int maxSize;

    private Queue<Bug> queue;

    private Lock lock;

    private Condition noFull;

    private Condition noEmpty;

    /**
     * 构造方法
     * @param name
     * @param queue
     * @param maxSize
     */
    public Producer(String name, Queue<Bug> queue, int maxSize, Lock lock, Condition noFull, Condition noEmpty){
        this.name = name;
        this.maxSize = maxSize;
        this.queue = queue;
        this.lock = lock;
        this.noFull = noFull;
        this.noEmpty = noEmpty;
    }

    /**
     * 重写run方法,来不断生产bug到队列中
     */
    @Override
    public void run() {
        Random random = new Random();
        while(true){
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            //生产者不断生产bug
            Bug bug = this.produceBug(random.nextInt());
            //模拟耗时操作
            //加锁,避免并发问题
            lock.lock();
            try{
                while(queue.size() == maxSize){
                    //当前的队列已满,无法将bug放进去,那么就等待,直到被唤醒(消费者会来唤醒的)
                    try {
                        System.out.println("[" + name + "]: 当前系统的Bug达到上限,歇会儿,不满的时候跟俺说一声~");
                        //在noFull条件上等待,等待不为满的时候将其唤醒
                        noFull.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                //现在可以把bug放进去了
                queue.add(bug);
                System.out.println("[" + name + "]: 俺往系统里面放了一个Bug" + bug.getBugId() + ",嘿嘿~嘿嘿嘿~");
                //唤醒在睡眠的消费者(当时那些消费者消费的时候发现队列为空,就在noEmpty条件变量上等待,现在我刚放进去一个bug,队列肯定不为空,所以唤醒他们)
                noEmpty.signalAll();
            }finally {
                //解锁
                lock.unlock();
            }
        }
    }

    /**
     *
     * 生产者生产一个带编号的Bug(可真是和我一模一样呢)
     * @param i
     * @return
     */
    public Bug produceBug(int i){
        Bug bug = new Bug(i);
        return bug;
    }
}
```

> Consumer类

```java
package ProAndCon;

import java.util.Queue;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;

/**
 * @author TheR1sing3un
 * @date 2022/1/9 17:32
 * @description 消费者
 */

public class Consumer extends Thread{

    private String name;

    private Queue<Bug> queue;

    private Lock lock;

    private Condition noFull;

    private Condition noEmpty;

    public Consumer(String name, Queue<Bug> queue, Lock lock, Condition noFull, Condition noEmpty) {
        this.name = name;
        this.queue = queue;
        this.lock = lock;
        this.noEmpty = noEmpty;
        this.noFull = noFull;
    }

    /**
     * 重写run方法,消费者从队列中拿Bug,然后去修复(消费者就是修复Bug的可怜程序员)
     */
    @Override
    public void run() {
        while (true){
            //操作队列就要上锁!
            lock.lock();
            try{
                while (queue.isEmpty()){
                    //当队列中没有Bug时,就等待
                    try {
                        System.out.println("[" + name + "]: 这系统做的可以,咋没Bug,整挺好,我摸鱼去了,有Bug叫我");
                        //当前队列为空,需要等待不为空的时候继续,于是在noEmpty上等待
                        noEmpty.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                //有bug时取出
                Bug bug = queue.poll();
                //模拟耗时操作
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                fixBug(bug);
                //唤醒在noFull上等待的生产者,因为刚刚该线程才消费了,目前肯定不满了
                noFull.signalAll();
            }finally {
                //解锁
                lock.unlock();
            }
        }
    }

    /**
     * 消费者修复Bug
     * @param bug
     */
    public void fixBug(Bug bug){
        System.out.println("[" + name + "]: 修复了Bug" + bug.getBugId());
    }
}
```

> Test测试

```java
package ProAndCon;

import java.util.LinkedList;
import java.util.Queue;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * @author TheR1sing3un
 * @date 2022/1/9 17:41
 * @description
 */

public class Test {
    public static void main(String[] args) {
        Queue<Bug> queue = new LinkedList<>();
        Lock lock = new ReentrantLock();
        Condition noFull = lock.newCondition();
        Condition noEmpty = lock.newCondition();
        int maxSize = 5;
        for (int i = 0; i < 10; i++) {
            new Producer("新手程序员"+i+"号",queue,maxSize,lock,noFull,noEmpty).start();
        }
        for (int i = 0; i < 10; i++) {
            new Consumer("老手程序员"+i+"号",queue,lock,noFull,noEmpty).start();
        }
    }
}
```

*测试截图：*

![image-20220109225147992](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20220109225147992.png)

---

## BlockingQueue

**BlockingQueue**，人称阻塞队列，是一个线程安全的容器，而且其内部结合了AQS的`Lock`和`Condition`，来保证从中取的时候，如果没有元素会被阻塞直到不为空；往里面放的时候，如果队列满了也会阻塞直到不满。

那这就简单了，我们就只需要从这个容器取/放，其他的交给它来。

### 代码实例

> Bug类(不再贴了)

> Producer类

```java
package ProAndCon2;

import ProAndCon.Bug;

import java.util.Random;
import java.util.concurrent.BlockingDeque;

/**
 * @author TheR1sing3un
 * @date 2022/1/9 22:58
 * @description
 */

public class Producer extends Thread{

    private String name;

    private BlockingDeque blockingDeque;

    public Producer(String name, BlockingDeque blockingDeque){
        this.name = name;
        this.blockingDeque = blockingDeque;
    }
    /**
     * 重写run方法,来不断生产bug到队列中
     */
    @Override
    public void run() {
        Random random = new Random();
        while(true){
            try {
                //模拟耗时操作
                Thread.sleep(new Random().nextInt(1000));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            //生产者不断生产bug
            ProAndCon.Bug bug = this.produceBug(random.nextInt());
            //往阻塞队列里面放
            try {
                blockingDeque.put(bug);
                System.out.println("[" + name + "]: 俺往系统里面放了一个Bug" + bug.getBugId() + ",嘿嘿~嘿嘿嘿~");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    /**
     *
     * 生产者生产一个带编号的Bug(可真是和我一模一样呢)
     * @param i
     * @return
     */
    public ProAndCon.Bug produceBug(int i){
        ProAndCon.Bug bug = new Bug(i);
        return bug;
    }
}
```

> Consumer类

```java
package ProAndCon2;

import ProAndCon.Bug;

import java.util.Random;
import java.util.concurrent.BlockingDeque;


/**
 * @author TheR1sing3un
 * @date 2022/1/9 23:00
 * @description
 */

public class Consumer extends Thread{
    private String name;

    private BlockingDeque<Bug> blockingDeque;

    public Consumer(String name, BlockingDeque blockingDeque) {
        this.name = name;
        this.blockingDeque = blockingDeque;
    }

    /**
     * 重写run方法,消费者从队列中拿Bug,然后去修复(消费者就是修复Bug的可怜程序员)
     */
    @Override
    public void run() {
        while (true){
            //从阻塞队列中取bug
            Bug bug = null;
            try {
                bug = blockingDeque.take();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            try {
                //模拟耗时操作
                Thread.sleep(new Random().nextInt(1000));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            fixBug(bug);
        }
    }

    /**
     * 消费者修复Bug
     * @param bug
     */
    public void fixBug(Bug bug){
        System.out.println("[" + name + "]: 修复了Bug" + bug.getBugId());
    }
}
```

> Test测试

```java
package ProAndCon2;

import java.util.concurrent.BlockingDeque;
import java.util.concurrent.LinkedBlockingDeque;

/**
 * @author TheR1sing3un
 * @date 2022/1/9 17:41
 * @description
 */

public class Test {
    public static void main(String[] args) {
        BlockingDeque<Bug> blockingDeque = new LinkedBlockingDeque<>(10);
        for (int i = 0; i < 10; i++) {
            new Producer("新手程序员"+i+"号",blockingDeque).start();
        }
        for (int i = 0; i < 10; i++) {
            new Consumer("老手程序员"+i+"号",blockingDeque).start();
        }
    }
}
```

*测试截图：*

![image-20220109230819362](https://ther1sing3un-personal-resource.oss-cn-beijing.aliyuncs.com/typora/images/image-20220109230819362.png)

---

## 总结

上述就是三种实现简易的生产者消费者模型的方法，实际中比较少用Java中的生产者消费者模型，更多的使用消息队列来当作一种生产者消费者模型。主要是理解这个多线程间通信和生产者消费者模式的特性。


