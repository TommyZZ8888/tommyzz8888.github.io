---
title: juc入门
date: '2025-12-28T09:38:23+08:00'

categories:
- juc
---


# juc

**狂神版**

# 什么是JUC

JUC就是java.util.concurrent下面的类包，专门用于多线程的开发。

# 线程和进程

进程是操作系统中的应用程序、是资源分配的基本单位，线程是用来执行具体的任务和功能，是CPU调度和分派的最小单位

一个进程往往可以包含多个线程，至少包含一个

例如：

进程：一个程序，QQ.exe Music.exe 程序的集合；
一个进程往往可以包含多个线程，至少包含一个！

> Java默认有几个线程？

2个，**main线程、GC线程**

------

线程：开了一个进程 Typora，写字，自动保存（线程负责的）
对于Java而言：Thread、Runnable、Callable

> Java 真的可以开启线程吗？

**开不了**。Java是没有权限去开启线程、操作硬件的，这是一个native的一个本地方法，它调用的底层的C++代码。

start底层代码：



```java
public synchronized void start() {
    /**
     * This method is not invoked for the main method thread or "system"
     * group threads created/set up by the VM. Any new functionality added
     * to this method in the future may have to also be added to the VM.
     *
     * A zero status value corresponds to state "NEW".
     */
    if (threadStatus != 0)
        throw new IllegalThreadStateException();

    /* Notify the group that this thread is about to be started
     * so that it can be added to the group's list of threads
     * and the group's unstarted count can be decremented. */
    group.add(this);

    boolean started = false;
    try {
        start0();
        started = true;
    } finally {
        try {
            if (!started) {
                group.threadStartFailed(this);
            }
        } catch (Throwable ignore) {
            /* do nothing. If start0 threw a Throwable then
              it will be passed up the call stack */
        }
    }
}
//这是一个C++底层，Java是没有权限操作底层硬件的
private native void start0();
```

## 并发和并行

并发：多个任务在同一个CPU 核上，按细分的时间片轮流（交替）执行，从逻编上来看那些任务是同时执行。

- 狂神笔记：
  - 多线程操作同一个资源
  - CPU 一核 ，模拟出来多条线程，天下武功，唯快不破，快速交替

并行： 单位时间内，多个处理器或多核处理器同时处理多个任务， 是真正意义上的“同时进行” 。

- 狂神笔记：
  - 多个人一起行走
  - CPU 多核 ，多个线程可以同时执行； 线程池

> 补充

并发

[![img](/img/juc/juc/并发.png)](https://images2015.cnblogs.com/blog/35158/201611/35158-20161125153718143-2147483584.png)

并行

[![img](/img/juc/juc/并行.png)](https://images2015.cnblogs.com/blog/35158/201611/35158-20161125153717753-1040268003.png)

串行： 有n个任务， 由一个线程按顺序执行。由于任务、方法都在一个线程执行，所以不存在线程不安全情况，也就不存在临界区的问题。

[![img](/img/juc/juc/串行.png)](https://images2015.cnblogs.com/blog/35158/201611/35158-20161125153717362-1273681161.png)

------

> 获取cpu的核数



```java
public class Test1 {
    public static void main(String[] args) {
        // 获取cpu的核数
		// CPU 密集型，IO密集型
        System.out.println(Runtime.getRuntime().availableProcessors());
    }
}
```

## 线程有几个状态



```java
public enum State {

    	//运行
        NEW,

    	//运行
        RUNNABLE,

    	//阻塞
        BLOCKED,

    	//等待
        WAITING,

    	//超时等待
        TIMED_WAITING,

    	//终止
        TERMINATED;
    }
```

## wait/sleep

1、来自不同的类

wait => Object

sleep => Thread

2、关于锁的释放

wait 会释放锁；

sleep睡觉了，不会释放锁；

3、使用的范围是不同的

wait 必须在同步代码块中；

sleep 可以在任何地方睡；

4、是否需要捕获异常

wait是不需要捕获异常；

sleep必须要捕获异常；

# Lock锁（重点）

> 传统的 synchronized



```java
import lombok.Synchronized;

public class Demo01 {
    public static void main(String[] args) {
        final Ticket ticket = new Ticket();

        new Thread(()->{
            for (int i = 0; i < 40; i++) {
                ticket.sale();
            }
        },"A").start();
        new Thread(()->{
            for (int i = 0; i < 40; i++) {
                ticket.sale();
            }
        },"B").start();
        new Thread(()->{
            for (int i = 0; i < 40; i++) {
                ticket.sale();
            }
        },"C").start();
    }
}
// 资源类 OOP 属性、方法
class Ticket {
    private int number = 30;

    //卖票的方式
    public synchronized void sale() {
        if (number > 0) {
            System.out.println(Thread.currentThread().getName() + "卖出了第" + (number--) + "张票剩余" + number + "张票");
        }
    }
}
```

> Lock

[![img](D:\test\items\Tommy\notes\img\juc\juc\lock.png)](https://img-service.csdnimg.cn/img_convert/77ca36442c3ced0659a5af7f06909bd2.png)

[![img](/img/juc/juc/lock02.png)](https://img-service.csdnimg.cn/img_convert/d0070945de646cdc612f11c99dd5fb7d.png)

公平锁： 十分公平，必须先来后到~；

非公平锁： 十分不公平，可以插队；(默认为非公平锁)

**补充：**

公平锁：多个线程按照申请锁的顺序去获得锁，线程会直接进入队列去排队，永远都是队列的第一位才能得到锁。

- 优点：所有的线程都能得到资源，不会饿死在队列中。
- 缺点：吞吐量会下降很多，队列里面除了第一个线程，其他的线程都会阻塞，cpu唤醒阻塞线程的开销会很大。

非公平锁：多个线程去获取锁的时候，会直接去尝试获取，获取不到，再去进入等待队列，如果能获取到，就直接获取到锁。

- 优点：可以减少CPU唤醒线程的开销，整体的吞吐效率会高点，CPU也不必取唤醒所有线程，会减少唤起线程的数量。

- 缺点：你们可能也发现了，这样可能导致队列中间的线程一直获取不到锁或者长时间获取不到锁，导致饿死。

  
  
  ```java
  import java.util.concurrent.locks.Lock;
  import java.util.concurrent.locks.ReentrantLock;
  
  public class LockDemo {
  public static void main(String[] args) {
  final Ticket2 ticket = new Ticket2();
  new Thread(() -> {
            for (int i = 0; i < 40; i++) {
                ticket.sale();
            }
        }, "A").start();
        new Thread(() -> {
            for (int i = 0; i < 40; i++) {
                ticket.sale();
            }
        }, "B").start();
        new Thread(() -> {
            for (int i = 0; i < 40; i++) {
                ticket.sale();
            }
        }, "C").start();
    }
  }
  
  ```
  
  
  
  ```java
  //lock三部曲
  //1、 Lock lock=new ReentrantLock();
  //2、 lock.lock() 加锁
  //3、 finally=> 解锁：lock.unlock();
  class Ticket2 {
  private int number = 30;
  // 创建锁
    Lock lock = new ReentrantLock();
    //卖票的方式
    public synchronized void sale() {
        lock.lock(); // 开启锁
        try {
            if (number > 0) {
                System.out.println(Thread.currentThread().getName() + "卖出了第" + (number--) + "张票剩余" + number + "张票");
            }
        }finally {
            lock.unlock(); // 关闭锁
        }
  
    }}
      
  ```

  

## Synchronized 与Lock 的区别

1、Synchronized 内置的Java关键字，Lock是一个Java类

2、Synchronized 无法判断获取锁的状态，Lock可以判断

3、Synchronized 会自动释放锁，lock必须要手动加锁和手动释放锁！可能会遇到死锁

4、Synchronized 线程1(获得锁->阻塞)、线程2(等待)；lock就不一定会一直等待下去，lock会有一个trylock去尝试获取锁，不会造成长久的等待。

5、Synchronized 是可重入锁，不可以中断的，非公平的；Lock，可重入的，可以判断锁，可以自己设置公平锁和非公平锁；

6、Synchronized 适合锁少量的代码同步问题，Lock适合锁大量的同步代码；

# 生产者和消费者的关系

## 生产者和消费者问题 Synchronized 版



```java
/**
* 线程之间的通信问题：生产者和消费者问题！ 等待唤醒，通知唤醒
* 线程交替执行 A B 操作同一个变量 num = 0
* A num+1
* B num-1
*/

public class ConsumeAndProduct {
    public static void main(String[] args) {
        Data data = new Data();

        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                try {
                    data.increment();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "A").start();
        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                try {
                    data.decrement();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "B").start();
    }
}

// 判断等待，业务，通知
class Data {
    private int num = 0;

    // +1
    public synchronized void increment() throws InterruptedException {
        // 判断等待
        if (num != 0) {
            this.wait();
        }
        num++;
        System.out.println(Thread.currentThread().getName() + "=>" + num);
        // 通知其他线程 +1 执行完毕
        this.notifyAll();
    }

    // -1
    public synchronized void decrement() throws InterruptedException {
        // 判断等待
        if (num == 0) {
            this.wait();
        }
        num--;
        System.out.println(Thread.currentThread().getName() + "=>" + num);
        // 通知其他线程 -1 执行完毕
        this.notifyAll();
    }
}
```

## 问题存在，A B C D 4 个线程！ 虚假唤醒

[![img](/img/juc/juc/虚假唤醒01.png)](https://img-service.csdnimg.cn/img_convert/1af314564aae84ed73a7ee90525f9d91.png)

[![img](/img/juc/juc/虚假唤醒02.png)](https://img-service.csdnimg.cn/img_convert/a917a32c3497007b58353dc2f5168fd6.png)

解决方式 ，if 改为while即可，防止虚假唤醒



```java
public class ConsumeAndProduct {
    public static void main(String[] args) {
        Data data = new Data();

        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                try {
                    data.increment();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "A").start();
        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                try {
                    data.decrement();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "B").start();
        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                try {
                    data.increment();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "C").start();
        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                try {
                    data.decrement();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "D").start();
    }
}

class Data {
    private int num = 0;

    // +1
    public synchronized void increment() throws InterruptedException {
        // 判断等待
        while (num != 0) {
            this.wait();
        }
        num++;
        System.out.println(Thread.currentThread().getName() + "=>" + num);
        // 通知其他线程 +1 执行完毕
        this.notifyAll();
    }

    // -1
    public synchronized void decrement() throws InterruptedException {
        // 判断等待
        while (num == 0) {
            this.wait();
        }
        num--;
        System.out.println(Thread.currentThread().getName() + "=>" + num);
        // 通知其他线程 -1 执行完毕
        this.notifyAll();
    }
}
```

> 结论：就是用if判断的话，唤醒后线程会从wait之后的代码开始运行，但是不会重新判断if条件，直接继续运行if代码块之后的代码，而如果使用while的话，也会从wait之后的代码运行，但是唤醒后会重新判断循环条件，如果不成立再执行while代码块之后的代码块，成立的话继续wait。
> 这也就是为什么用while而不用if的原因了，因为线程被唤醒后，执行开始的地方是wait之后

## JUC版的生产者和消费者问题

[![img](/img/juc/juc/生产者消费者.png)](https://img-service.csdnimg.cn/img_convert/dad7044c4b8b46648084823841cb6781.png)

> 补充：

Condition是在java 1.5中才出现的，它用来替代传统的Object的wait()、notify()实现线程间的协作，相比使用Object的wait()、notify()，使用Condition的await()、signal()这种方式实现线程间协作更加安全和高效。因此通常来说比较推荐使用Condition，阻塞队列实际上是使用了Condition来模拟线程间协作。

Condition是个接口，基本的方法就是await()和signal()方法；

- Conditon中的await()对应Object的wait()；

- Condition中的signal()对应Object的notify()；

- Condition中的signalAll()对应Object的notifyAll()。

  import java.util.concurrent.locks.Condition;
  import java.util.concurrent.locks.Lock;
  import java.util.concurrent.locks.ReentrantLock;

Condition依赖于Lock接口，生成一个Condition的基本代码是`lock.newCondition()`
调用Condition的await()和signal()方法，都必须在lock保护之内，就是说必须在lock.lock()和lock.unlock之间才可以使用



```java
public class LockCAP {
    public static void main(String[] args) {
        Data2 data = new Data2();

        new Thread(() -> {
            for (int i = 0; i < 10; i++) {

                try {
                    data.increment();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

            }
        }, "A").start();
        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                try {
                    data.decrement();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "B").start();
        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                try {
                    data.increment();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "C").start();
        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                try {
                    data.decrement();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "D").start();
    }
}

class Data2 {
    private int num = 0;
    Lock lock = new ReentrantLock();
    Condition condition = lock.newCondition();
	//condition.await(); // 等待
	//condition.signalAll(); // 唤醒全部
    // +1
    public  void increment() throws InterruptedException {
        lock.lock();
        try {
            // 判断等待
            while (num != 0) {
                condition.await();
            }
            num++;
            System.out.println(Thread.currentThread().getName() + "=>" + num);
            // 通知其他线程 +1 执行完毕
            condition.signalAll();
        }finally {
            lock.unlock();
        }

    }

    // -1
    public  void decrement() throws InterruptedException {
        lock.lock();
        try {
            // 判断等待
            while (num == 0) {
                condition.await();
            }
            num--;
            System.out.println(Thread.currentThread().getName() + "=>" + num);
            // 通知其他线程 -1 执行完毕
            condition.signalAll();
        }finally {
            lock.unlock();
        }

    }
}
```

## Condition 精准的通知和唤醒线程

如果我们要指定通知的下一个进行顺序怎么办呢？ 我们可以使用Condition来指定通知进程



```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * Description：
 * A 执行完 调用B
 * B 执行完 调用C
 * C 执行完 调用A
 *
 **/

public class ConditionDemo {
    public static void main(String[] args) {
        Data3 data3 = new Data3();

        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                data3.printA();
            }
        },"A").start();
        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                data3.printB();
            }
        },"B").start();
        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                data3.printC();
            }
        },"C").start();
    }

}
class Data3 {
    private Lock lock = new ReentrantLock();
    private Condition condition1 = lock.newCondition();
    private Condition condition2 = lock.newCondition();
    private Condition condition3 = lock.newCondition();
    private int num = 1; // 1A 2B 3C

    public void printA() {
        lock.lock();
        try {
            // 业务代码 判断 -> 执行 -> 通知
            while (num != 1) {
                condition1.await();
            }
            System.out.println(Thread.currentThread().getName() + "==> AAAA" );
            num = 2;
            condition2.signal();
        }catch (Exception e) {
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
    }
    public void printB() {
        lock.lock();
        try {
            // 业务代码 判断 -> 执行 -> 通知
            while (num != 2) {
                condition2.await();
            }
            System.out.println(Thread.currentThread().getName() + "==> BBBB" );
            num = 3;
            condition3.signal();
        }catch (Exception e) {
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
    }
    public void printC() {
        lock.lock();
        try {
            // 业务代码 判断 -> 执行 -> 通知
            while (num != 3) {
                condition3.await();
            }
            System.out.println(Thread.currentThread().getName() + "==> CCCC" );
            num = 1;
            condition1.signal();
        }catch (Exception e) {
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
    }
}
/*
A==> AAAA
B==> BBBB
C==> CCCC
A==> AAAA
B==> BBBB
C==> CCCC
...
*/
```

# 8锁现象

如何判断锁的是谁！永远的知道什么锁，锁到底锁的是谁！

## 例子1

两个同步方法，先执行发短信还是打电话



```java
import java.util.concurrent.TimeUnit;

public class Test1 {
    public static void main(String[] args) {
        Phone phone = new Phone();

        //锁的存在
        new Thread(()->{
            phone.sendSms();
        }, "A").start();

        //捕获
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        new Thread(()->{
            phone.call();
        },"B").start();
    }
}

class Phone{
    // synchronized 锁的对象是方法的调用者！
    // 两个方法用的是同一个锁，谁先拿到谁执行
    public synchronized void sendSms(){
        try {
            TimeUnit.SECONDS.sleep(4);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("发短信");
    }

    public synchronized void call(){
        System.out.println("打电话");
    }
}
```

两种情况：

情况1：两个线程在标准情况下运行，即没有休眠4s

> 就是注释代码中的这一段



```
//        try {  
//            TimeUnit.SECONDS.sleep(4);  
//        } catch (InterruptedException e) {  
//            e.printStackTrace();  
//        }  
```

情况1输出结果为：



```
发短信
打电话
```

先打印发短信，过了1s再打印打电话

情况2，加上了上述注释中的代码

大约5s，发短信和打电话一起被打印

## 例子2

情况1：只有一个对象



```java
import java.util.concurrent.TimeUnit;

public class Test2 {
    public static void main(String[] args) {

        Phone2 phone1 = new Phone2();

        //锁的存在
        new Thread(()->{
            phone1.sendSms();
        }, "A").start();

        //捕获
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        new Thread(()->{
            phone1.hello();
        },"B").start();
    }
}

class Phone2{
    public synchronized void sendSms(){

        // synchronized 锁的对象是方法的调用者！
        try {
            TimeUnit.SECONDS.sleep(4);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("发短信");
    }

    public synchronized void call(){
        System.out.println("打电话");
    }

    // 这里没有锁！不是同步方法，不受锁的影响
    public void hello(){
        System.out.println("hello");
    }
}
```

输出结果



```
hello
发短信
```

1s后打印hello，4s后打印发短信，hello没有锁，不受锁影响

> 如果去掉延迟4s的话就是发短信先了，因为不用等待4s了，而hello要等待main中 sleep 1s

情况2：

两个对象，两个线程分别调用不同对象的发短信和打电话方法



```java
import java.util.concurrent.TimeUnit;

public class Test2 {
    public static void main(String[] args) {

        Phone2 phone1 = new Phone2();
        Phone2 phone2 = new Phone2();

        //锁的存在
        new Thread(()->{
            phone1.sendSms();
        }, "A").start();

        //捕获
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        new Thread(()->{
            phone2.call();
        },"B").start();
    }
}

class Phone2{
    public synchronized void sendSms(){

        // synchronized 锁的对象是方法的调用者！
        try {
            TimeUnit.SECONDS.sleep(4);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("发短信");
    }

    public synchronized void call(){
        System.out.println("打电话");
    }

    // 这里没有锁！不是同步方法，不受锁的影响
    public void hello(){
        System.out.println("hello");
    }
}
```

输出结果：



```
打电话
发短信
```

两把锁不一样，按时间来，打电话更快

## 例子3

情况1：增加两个静态的同步方法，只有一个对象，先打印 发短信？打电话？



```java
import java.util.concurrent.TimeUnit;

public class Test3 {
    public static void main(String[] args) {
        Phone3 phone1 = new Phone3();

        new Thread(()->{
            phone1.sendSms();
        }, "A").start();

        //捕获
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        new Thread(()->{
            phone1.call();
        },"B").start();
    }
}

class Phone3{
    public static synchronized void sendSms(){
        try {
            TimeUnit.SECONDS.sleep(4);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("发短信");
    }

    public static synchronized void call(){
        System.out.println("打电话");
    }
}
```

输出结构：



```
发短信
打电话
```

说明：
synchronized 锁的对象是方法的调用者！
static 静态方法，类一加载就有了！锁的是Class对象

再看另一个情况加深理解：

情况2：两个对象！增加两个静态的同步方法， 先打印 发短信？打电话？



```java
import java.util.concurrent.TimeUnit;

public class Test3 {
    public static void main(String[] args) {
        Phone3 phone1 = new Phone3();
        Phone3 phone2 = new Phone3();

        new Thread(()->{
            phone1.sendSms();
        }, "A").start();

        //捕获
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        new Thread(()->{
            phone2.call();
        },"B").start();
    }
}

class Phone3{
    public static synchronized void sendSms(){
        try {
            TimeUnit.SECONDS.sleep(4);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("发短信");
    }

    public static synchronized void call(){
        System.out.println("打电话");
    }
}
```

输出结果：



```
发短信
打电话
```

说明：发短信永远在前面，因为static的修饰，锁的是对象Phone3，而对象只有一个，所以按照线程的顺序来获取锁。

## 例子4

情况1：1个静态的同步方法，1个普通的同步方法 ，一个对象，先打印 发短信？打电话？



```java
import java.util.concurrent.TimeUnit;

public class Test4 {
    public static void main(String[] args) {
        Phone4 phone1 = new Phone4();
        
        new Thread(()->{
            phone1.sendSms();
        }, "A").start();

        //捕获
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        new Thread(()->{
            phone1.call();
        },"B").start();
    }
}

class Phone4{
    public static synchronized void sendSms(){
        try {
            TimeUnit.SECONDS.sleep(4);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("发短信");
    }

    public synchronized void call(){
        System.out.println("打电话");
    }
}
```

输出结果：



```
打电话
发短信
```

说明：加了static锁的是class类模板，普通同步方法锁的是调用者

情况2：1个静态的同步方法，1个普通的同步方法 ，两个对象，先打印 发短信？打电话？



```java
import java.util.concurrent.TimeUnit;

public class Test4 {
    public static void main(String[] args) {
        Phone4 phone1 = new Phone4();
        Phone4 phone2 = new Phone4();

        new Thread(()->{
            phone1.sendSms();
        }, "A").start();

        //捕获
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        new Thread(()->{
            phone2.call();
        },"B").start();
    }
}

class Phone4{
    public static synchronized void sendSms(){
        try {
            TimeUnit.SECONDS.sleep(4);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("发短信");
    }

    public synchronized void call(){
        System.out.println("打电话");
    }
}
```

输出结果：



```
打电话
发短信
```

## 小结

new this 具体的一个手机
static Class 唯一的一个模板

# 集合类不安全

## List不安全



```java
//java.util.ConcurrentModificationException 并发修改异常！
public class ListTest {
    public static void main(String[] args) {

        List<Object> arrayList = new ArrayList<>();

        for(int i=1;i<=10;i++){
            new Thread(()->{
                arrayList.add(UUID.randomUUID().toString().substring(0,5));
                System.out.println(arrayList);
            },String.valueOf(i)).start();
        }

    }
}
```

> 会导致 java.util.ConcurrentModificationException 并发修改异常！

解决方案：

- List list = new Vector<>();
- List list = Collections.synchronizedList(new ArrayList<>());
- List list = new CopyOnWriteArrayList<>();

推荐使用第三种的`CopyOnWriteArrayList`!



```java
public class ListTest {
    public static void main(String[] args) {
        List<String> list = new CopyOnWriteArrayList<>();
        
        for (int i = 1; i <=10; i++) {
            new Thread(() -> {
                list.add(UUID.randomUUID().toString().substring(0,5));
                System.out.println(list);
            },String.valueOf(i)).start();
        }
    }
}
```

CopyOnWriteArrayList：写入时复制！ COW 计算机程序设计领域的一种优化策略

核心思想是，如果有多个调用者（Callers）同时要求相同的资源（如内存或者是磁盘上的数据存储），他们会共同获取相同的指针指向相同的资源，直到某个调用者视图修改资源内容时，系统才会真正复制一份专用副本（private copy）给该调用者，而其他调用者所见到的最初的资源仍然保持不变。这过程对其他的调用者都是透明的（transparently）。此做法主要的优点是如果调用者没有修改资源，就不会有副本（private copy）被创建，因此多个调用者只是读取操作时可以共享同一份资源。

读的时候不需要加锁，如果读的时候有多个线程正在向CopyOnWriteArrayList添加数据，读还是会读到旧的数据，因为写的时候不会锁住旧的CopyOnWriteArrayList。

多个线程调用的时候，list，读取的时候，固定的，写入（存在覆盖操作）；在写入的时候避免覆盖，造成数据错乱的问题；

### CopyOnWriteArrayList比Vector厉害在哪里？

Vector底层是使用synchronized关键字来实现的：效率特别低下。

[![img](/img/juc/juc/vector.png)](https://img-service.csdnimg.cn/img_convert/ec0193ec03b4e67874350644b0f7b6ec.png)

CopyOnWriteArrayList使用的是Lock锁，效率会更加高效！

[![img](/img/juc/juc/copyOnWriteArrayList.png)](https://img-service.csdnimg.cn/img_convert/65d81d2d9f14913e79f91dcf23f8eea7.png)

## Set不安全

Set和List同理可得: 多线程情况下，普通的Set集合是线程不安全的；

解决方案还是两种：

- 使用Collections工具类的synchronized包装的Set类
  - `Set<String> set = Collections.synchronizedSet(new HashSet<>());`
- 使用CopyOnWriteArraySet 写入复制的JUC解决方案（推荐！）
  - `Set<String> set = new CopyOnWriteArraySet<>();`

示例代码：



```
public class SetTest {
    public static void main(String[] args) {

        Set<String> set = new CopyOnWriteArraySet<>();

        for (int i = 1; i <= 30; i++) {
            new Thread(() -> {
                set.add(UUID.randomUUID().toString().substring(0,5));
                System.out.println(set);
            },String.valueOf(i)).start();
        }
    }
}
```

### HashSet底层是什么？

hashSet底层就是一个HashMap；

[![img](/img/juc/juc/hashset.png)](https://img-service.csdnimg.cn/img_convert/35c7c667ffc7a2e2bb001a147d30b5f9.png)

## Map不安全

回顾Map基本操作，看下hashmap源码



```java
    /**
     * The default initial capacity - MUST be a power of two.
     */
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

    /**
     * The maximum capacity, used if a higher value is implicitly specified
     * by either of the constructors with arguments.
     * MUST be a power of two <= 1<<30.
     */
    static final int MAXIMUM_CAPACITY = 1 << 30;

    /**
     * The load factor used when none specified in constructor.
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
```

源码中从上到下三个常量：

- DEFAULT_INITIAL_CAPACITY：默认初始容量16
- MAXIMUM_CAPACITY：最大容量1 << 30，即 2^30=1073741824
- DEFAULT_LOAD_FACTOR: 负载因子：默认值为0.75。 当元素的总个数>当前数组的长度 * 负载因子。

同样的HashMap基础类也存在并发修改异常！

解决方案：

- 使用Collections工具类的synchronizedMap包装的Map类
  - `Map<String, String> map = Collections.synchronizedMap(new HashMap<>());`
- 使用ConcurrentHashMap（推荐！）
  - `Map<String, String> map = new ConcurrentHashMap<>();`
  - ConcurrentHashMap是面试中常问的问题

示例代码：



```java
public class MapTest {
    public static void main(String[] args) {
        /**
         * 解决方案
         * 1. Map<String, String> map = Collections.synchronizedMap(new HashMap<>());
         *  Map<String, String> map = new ConcurrentHashMap<>();
         */
        Map<String, String> map = new ConcurrentHashMap<>();
        //加载因子、初始化容量
        for (int i = 1; i < 100; i++) {
            new Thread(()->{
                map.put(Thread.currentThread().getName(), UUID.randomUUID().toString().substring(0,5));
                System.out.println(map);
            },String.valueOf(i)).start();
        }
    }
}
```

# Callable

1、可以有返回值；
2、可以抛出异常；
3、方法不同，run()/call()



```java
public class CallableTest {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        for (int i = 1; i < 10; i++) {
            MyThread1 myThread1 = new MyThread1();

            FutureTask<Integer> futureTask = new FutureTask<>(myThread1);
            // 放入Thread中使用，结果会被缓存
            new Thread(futureTask,String.valueOf(i)).start();
            // 这个get方法可能会被阻塞，如果在call方法中是一个耗时的方法，所以一般情况我们会把这个放在最后，或者使用异步通信
            int a = futureTask.get();
            System.out.println("返回值:" + s);
        }

    }

}
class MyThread1 implements Callable<Integer> {

    @Override
    public Integer call() throws Exception {
        System.out.println("call()");// 会打印几个call
        return 1024;
    }
}
```

细节：
1、有缓存
2、结果可能需要等待，会阻塞！

# 常用的辅助类(必会)

## CountDownLatch



```java
import java.util.concurrent.CountDownLatch;

public class CountDownLatchDemo {
    public static void main(String[] args) throws InterruptedException {
        // 总数是6，必须要执行任务的时候，再使用！
        CountDownLatch countDownLatch = new CountDownLatch(6);

        for (int i = 1; i <= 6; i++) {
            new Thread(()->{
                System.out.println(Thread.currentThread().getName() + "Go out");
                countDownLatch.countDown();// 数量-1
            }, String.valueOf(i)).start();
        }

        countDownLatch.await();

        System.out.println("Close Door");
    }
输出结果（顺序不一定是一样的）：
```



```
1: Go out
6: Go out
4: Go out
3: Go out
2: Go out
5: Go out
Close Door
```

原理：
`countDownLatch.countDown();`// 数量-1

`countDownLatch.await();` // 等待计数器归零，然后再向下执行

每次有线程调用 `countDown()` 数量-1，假设计数器变为0，`countDownLatch.await()` 就会被唤醒，继续
执行！

## CyclicBarrier

[![img](/img/juc/juc/cyclicBarrier.png)](https://img-service.csdnimg.cn/img_convert/dee6ef3d75096d41547b6729fcce3037.png)

加法计数器



```java
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;

public class CyclicBarrierDemo {
    public static void main(String[] args) {
        //集齐7颗龙珠召唤神龙

        CyclicBarrier cyclicBarrier = new CyclicBarrier(7, ()->{
            System.out.println("召唤神龙成功！");
        });

        for (int i = 1; i <= 7 ; i++) {
            // lambda能操作到 i 吗？不行，所以用一个final定义一个临时常量
            final int temp = i;
            new Thread(()->{
                System.out.println(Thread.currentThread().getName() + "收集第" + temp + "个龙珠！");
                try {
                    cyclicBarrier.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
}
```

通过await等待，看线程是否达到7个！

输出结果（顺序不一定是一样的）：



```
Thread-0收集第1个龙珠！
Thread-1收集第2个龙珠！
Thread-2收集第3个龙珠！
Thread-3收集第4个龙珠！
Thread-5收集第6个龙珠！
Thread-4收集第5个龙珠！
Thread-6收集第7个龙珠！
召唤神龙成功！
```

如果cyclicbarrier设置为8，那么达不到8个线程就无法“召唤神龙”成功。

## Semaphore

Semaphore：信号量

[![img](/img/juc/juc/semaphore.jpg)](https://images.cnblogs.com/cnblogs_com/kylinxxx/1675669/o_2012060230581607221821(1).jpg)

例子：抢车位！
6车---3个停车位置



```java
import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

public class SemaphoreDemo {
    public static void main(String[] args) {

        // 线程数量，停车位，限流
        Semaphore semaphore = new Semaphore(3);
        for (int i = 0; i <= 6; i++) {
            new Thread(() -> {
                // acquire() 得到
                try {
                    semaphore.acquire();
                    System.out.println(Thread.currentThread().getName() + "抢到车位");
                    TimeUnit.SECONDS.sleep(2);
                    System.out.println(Thread.currentThread().getName() + "离开车位");
                }catch (Exception e) {
                    e.printStackTrace();
                }finally {
                    semaphore.release(); // release() 释放
                }
            }).start();
        }
    }
}
```

输出结果（顺序不一定是一样的）：



```
Thread-0抢到车位
Thread-1抢到车位
Thread-3抢到车位
Thread-0离开车位
Thread-1离开车位
Thread-3离开车位
Thread-2抢到车位
Thread-4抢到车位
Thread-5抢到车位
Thread-5离开车位
Thread-2离开车位
Thread-4离开车位
Thread-6抢到车位
Thread-6离开车位
```

原理：

semaphore.acquire()获得资源，如果资源已经使用完了，就等待资源释放后再进行使用！

semaphore.release()释放，会将当前的信号量释放+1，然后唤醒等待的线程！

作用： 多个共享资源互斥的使用！ 并发限流，控制最大的线程数！

# 读写锁ReadWriteLock

[![img](/img/juc/juc/readAndWriteLock.jpg)](https://images.cnblogs.com/cnblogs_com/kylinxxx/1675669/o_2012060237131607222207(1).jpg)

> 补充

读写锁包含一对相关的锁，读锁用于只读操作，写锁用于写操作。读锁可能由多个读线程同时运行，写锁是唯一的。

1、读锁和写锁之间是互斥的，同一时间只能有一个在运行。但是可以有多个线程同时读取数据。

2、写入数据之前必须重新确认(ReCheck)状态，因为其他的线程可能会拿到写锁再一次修改我们已经修改过的值。这是因为前一个线程拿到写锁之后，后面的线程会被阻塞。当前一个线程释放写锁之后，被阻塞的线程会继续运行完成被阻塞的部分代码，所以才会出现这样的情况。

3、当某一个线程上了写锁之后，自己仍然可以上读锁，之后在释放写锁，这是一种降级(Downgrade)的处理方法。

读写锁（ReadWriteLock）包含如下两个方法：

1.读锁

```
Lock readLock()
```

2.写锁

```
Lock writeLock()
```

> 例子

先看看数据不可靠的例子



```java
public class ReadWriteLockDemo {
    public static void main(String[] args) {
        MyCache myCache = new MyCache();
        int num = 6;
        for (int i = 1; i <= num; i++) {
            int finalI = i;
            new Thread(() -> {

                myCache.write(String.valueOf(finalI), String.valueOf(finalI));

            },String.valueOf(i)).start();
        }

        for (int i = 1; i <= num; i++) {
            int finalI = i;
            new Thread(() -> {

                myCache.read(String.valueOf(finalI));

            },String.valueOf(i)).start();
        }
    }
}

/**
 *  方法未加锁，导致写的时候被插队
 */
class MyCache {
    private volatile Map<String, String> map = new HashMap<>();

    public void write(String key, String value) {
        System.out.println(Thread.currentThread().getName() + "线程开始写入");
        map.put(key, value);
        System.out.println(Thread.currentThread().getName() + "线程写入ok");
    }

    public void read(String key) {
        System.out.println(Thread.currentThread().getName() + "线程开始读取");
        map.get(key);
        System.out.println(Thread.currentThread().getName() + "线程读取ok");
    }
}
```

输出结果（顺序不一定是一样的）：



```java
1线程开始写入
4线程开始写入
3线程开始写入
3线程写入ok
2线程开始写入
6线程开始写入
1线程写入ok
5线程开始写入
5线程写入ok
4线程写入ok
1线程开始读取
6线程写入ok
2线程写入ok
5线程开始读取
5线程读取ok
1线程读取ok
2线程开始读取
2线程读取ok
6线程开始读取
6线程读取ok
3线程开始读取
3线程读取ok
4线程开始读取
4线程读取ok
```

可以看到上面的结果不是先写完在读取，而是有可能被其他线程插队的。所以如果我们不加锁的情况，多线程的读写会造成数据不可靠的问题。

我们也可以采用synchronized这种重量锁和轻量锁 lock去保证数据的可靠。

但是这次我们采用更细粒度的锁：ReadWriteLock 读写锁来保证



```java
public class ReadWriteLockDemo {
    public static void main(String[] args) {
        MyCache2 myCache = new MyCache2();
        int num = 6;
        for (int i = 1; i <= num; i++) {
            int finalI = i;
            new Thread(() -> {

                myCache.write(String.valueOf(finalI), String.valueOf(finalI));

            },String.valueOf(i)).start();
        }

        for (int i = 1; i <= num; i++) {
            int finalI = i;
            new Thread(() -> {

                myCache.read(String.valueOf(finalI));

            },String.valueOf(i)).start();
        }
    }

}
class MyCache2 {
    private volatile Map<String, String> map = new HashMap<>();
    private ReadWriteLock lock = new ReentrantReadWriteLock();

    public void write(String key, String value) {
        lock.writeLock().lock(); // 写锁
        try {
            System.out.println(Thread.currentThread().getName() + "线程开始写入");
            map.put(key, value);
            System.out.println(Thread.currentThread().getName() + "线程写入ok");

        }finally {
            lock.writeLock().unlock(); // 释放写锁
        }
    }

    public void read(String key) {
        lock.readLock().lock(); // 读锁
        try {
            System.out.println(Thread.currentThread().getName() + "线程开始读取");
            map.get(key);
            System.out.println(Thread.currentThread().getName() + "线程读取ok");
        }finally {
            lock.readLock().unlock(); // 释放读锁
        }
    }
}
```

输出结果（顺序不一定是一样的）：



```
1线程开始写入
1线程写入ok
3线程开始写入
3线程写入ok
5线程开始写入
5线程写入ok
6线程开始写入
6线程写入ok
2线程开始写入
2线程写入ok
4线程开始写入
4线程写入ok
1线程开始读取
1线程读取ok
5线程开始读取
5线程读取ok
2线程开始读取
6线程开始读取
3线程开始读取
3线程读取ok
2线程读取ok
6线程读取ok
4线程开始读取
4线程读取ok
```

> 总结

- 独占锁（写锁） 一次只能被一个线程占有
- 共享锁（读锁） 多个线程可以同时占有
- ReadWriteLock
- 读-读 可以共存！
- 读-写 不能共存！
- 写-写 不能共存！

# 阻塞队列

[![img](/img/juc/juc/fifo.png)](https://img-service.csdnimg.cn/img_convert/3b6b0b33e6e9b0f2261a89b6e42e78ea.png)

[![img](/img/juc/juc/blockingQueue.png)](https://img-service.csdnimg.cn/img_convert/d651ccc40069352ee6c8b86ae2cee8eb.png)

## BlockQueue

是Collection的一个子类

什么情况下我们会使用阻塞队列？多线程并发处理、线程池

[![img](/img/juc/juc/blockingQueue02.png)](https://img-service.csdnimg.cn/img_convert/cae49c50458adc0997d57b2044666ccb.png)

BlockingQueue 有四组api

| 方式         | 抛出异常 | 不会抛出异常，有返回值 | 阻塞，等待 | 超时等待                |
| :----------- | :------- | :--------------------- | :--------- | :---------------------- |
| 添加         | add      | offer                  | put        | offer(timenum.timeUnit) |
| 移出         | remove   | poll                   | take       | poll(timenum,timeUnit)  |
| 判断队首元素 | element  | peek                   | -          | -                       |



```java
/**
 * 抛出异常
 */
public static void test1(){
    //需要初始化队列的大小
    ArrayBlockingQueue blockingQueue = new ArrayBlockingQueue<>(3);

    System.out.println(blockingQueue.add("a"));
    System.out.println(blockingQueue.add("b"));
    System.out.println(blockingQueue.add("c"));
    //抛出异常：java.lang.IllegalStateException: Queue full
    //System.out.println(blockingQueue.add("d"));
    System.out.println(blockingQueue.remove());
    System.out.println(blockingQueue.remove());
    System.out.println(blockingQueue.remove());
    //如果多移除一个
    //这也会造成 java.util.NoSuchElementException 抛出异常
    System.out.println(blockingQueue.remove());
}
=======================================================================================
/**
 * 不抛出异常，有返回值
 */
public static void test2(){
    ArrayBlockingQueue blockingQueue = new ArrayBlockingQueue<>(3);
    System.out.println(blockingQueue.offer("a"));
    System.out.println(blockingQueue.offer("b"));
    System.out.println(blockingQueue.offer("c"));
    //添加 一个不能添加的元素 使用offer只会返回false 不会抛出异常
    System.out.println(blockingQueue.offer("d"));

    System.out.println(blockingQueue.poll());
    System.out.println(blockingQueue.poll());
    System.out.println(blockingQueue.poll());
    //弹出 如果没有元素 只会返回null 不会抛出异常
    System.out.println(blockingQueue.poll());
}
=======================================================================================
/**
 * 等待 一直阻塞
 */
public static void test3() throws InterruptedException {
    ArrayBlockingQueue blockingQueue = new ArrayBlockingQueue<>(3);

    //一直阻塞 不会返回
    blockingQueue.put("a");
    blockingQueue.put("b");
    blockingQueue.put("c");

    //如果队列已经满了， 再进去一个元素  这种情况会一直等待这个队列 什么时候有了位置再进去，程序不会停止
	//blockingQueue.put("d");

    System.out.println(blockingQueue.take());
    System.out.println(blockingQueue.take());
    System.out.println(blockingQueue.take());
    //如果我们再来一个  这种情况也会等待，程序会一直运行 阻塞
    System.out.println(blockingQueue.take());
}
=======================================================================================
/**
 * 等待 超时阻塞
 *  这种情况也会等待队列有位置 或者有产品 但是会超时结束
 */
public static void test4() throws InterruptedException {
    ArrayBlockingQueue blockingQueue = new ArrayBlockingQueue<>(3);
    blockingQueue.offer("a");
    blockingQueue.offer("b");
    blockingQueue.offer("c");
    System.out.println("开始等待");
    blockingQueue.offer("d",2, TimeUnit.SECONDS);  //超时时间2s 等待如果超过2s就结束等待
    System.out.println("结束等待");
    System.out.println("===========取值==================");
    System.out.println(blockingQueue.poll());
    System.out.println(blockingQueue.poll());
    System.out.println(blockingQueue.poll());
    System.out.println("开始等待");
    blockingQueue.poll(2,TimeUnit.SECONDS); //超过两秒 我们就不要等待了
    System.out.println("结束等待");
}
```

## 同步队列

同步队列 没有容量，也可以视为容量为1的队列；

进去一个元素，必须等待取出来之后，才能再往里面放入一个元素；

put方法 和 take方法；

Synchronized 和 其他的BlockingQueue 不一样 它不存储元素；

put了一个元素，就必须从里面先take出来，否则不能再put进去值！

并且SynchronousQueue 的take是使用了lock锁保证线程安全的。



```java
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.SynchronousQueue;
import java.util.concurrent.TimeUnit;

public class SynchronousQueueDemo {
    public static void main(String[] args) {
        BlockingQueue<String> blockingQueue = new SynchronousQueue<>();

        new Thread(()->{
            try {
                System.out.println(Thread.currentThread().getName() + "put 1");
                blockingQueue.put("1");
                System.out.println(Thread.currentThread().getName() + "put 2");
                blockingQueue.put("2");
                System.out.println(Thread.currentThread().getName() + "put 3");
                blockingQueue.put("3");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }, "T1").start();

        new Thread(()->{
            try {
                TimeUnit.SECONDS.sleep(3);
                System.out.println(Thread.currentThread().getName() + "==>" + blockingQueue.take());
                TimeUnit.SECONDS.sleep(3);
                System.out.println(Thread.currentThread().getName() + "==>" + blockingQueue.take());
                TimeUnit.SECONDS.sleep(3);
                System.out.println(Thread.currentThread().getName() + "==>" + blockingQueue.take());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

        },"T2").start();
    }
}
```



输出结果（顺序不一定是一样的）：



```
T1put 1
T2==>1
T1put 2
T2==>2
T1put 3
T2==>3
```

# 线程池

线程池：三大方式、七大参数、四种拒绝策略

池化技术

程序的运行，本质：占用系统的资源！我们需要去优化资源的使用 ===> 池化技术

线程池、JDBC的连接池、内存池、对象池 等等。。。。

资源的创建、销毁十分消耗资源

池化技术：事先准备好一些资源，如果有人要用，就来我这里拿，用完之后还给我，以此来提高效率。

## 线程池的好处：

1、降低资源的消耗；

2、提高响应的速度；

3、方便管理；

线程复用、可以控制最大并发数、管理线程；

## 线程池：三大方法

- ExecutorService threadPool = Executors.newSingleThreadExecutor();//单个线程
- ExecutorService threadPool2 = Executors.newFixedThreadPool(5); //创建一个固定的线程池的大小
- ExecutorService threadPool3 = Executors.newCachedThreadPool(); //可伸缩的

[![img](/img/juc/juc/pool.jpg)](https://images.cnblogs.com/cnblogs_com/kylinxxx/1675669/o_2012060328261607225266.jpg)



```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

//Executors 工具类、3大方法
public class Demo01 {
    public static void main(String[] args) {
        ExecutorService threadPool = Executors.newSingleThreadExecutor();//单个线程
        //ExecutorService threadPool2 = Executors.newFixedThreadPool(5);//创建一个固定的线程池的大小
        //ExecutorService threadPool3 = Executors.newCachedThreadPool()//可伸缩的，遇强则强，遇弱则弱

        try {
            for (int i = 0; i < 100; i++) {
                // 使用了线程池之后，使用线程池来创建线程
                threadPool.execute(()->{
                    System.out.println(Thread.currentThread().getName() + "ok");
                });
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            // 线程池用完，程序结束，关闭线程池
            threadPool.shutdown();
        }
    }
}
```

## 七大参数

源码分析



```java
public ThreadPoolExecutor(int corePoolSize,  //核心线程池大小
                          int maximumPoolSize, //最大的线程池大小
                          long keepAliveTime,  //超时了没有人调用就会释放
                          TimeUnit unit, //超时单位
                          BlockingQueue<Runnable> workQueue, //阻塞队列
                          ThreadFactory threadFactory, //线程工厂 创建线程的 一般不用动
                          RejectedExecutionHandler handler //拒绝策略
                         ) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

狂神的银行排队例子

[![img](/img/juc/juc/bank_wait.jpg)](https://images.cnblogs.com/cnblogs_com/kylinxxx/1675669/o_2012060700541607238022(1).jpg)

## 4种拒绝策略

1. new ThreadPoolExecutor.AbortPolicy()： //该拒绝策略为：银行满了，还有人进来，不处理这个人的，并抛出异常。超出最大承载，就会抛出异常：队列容量大小+maxPoolSize
2. new ThreadPoolExecutor.CallerRunsPolicy()： //该拒绝策略为：哪来的去哪里 main线程进行处理
3. new ThreadPoolExecutor.DiscardPolicy(): //该拒绝策略为：队列满了,丢掉异常，不会抛出异常。
4. new ThreadPoolExecutor.DiscardOldestPolicy()： //该拒绝策略为：队列满了，尝试去和最早的进程竞争，不会抛出异常

## 如何设置线程池的大小

1、CPU密集型：电脑的核数是几核就选择几；选择maximunPoolSize的大小



```java
// 获取cpu 的核数
        int max = Runtime.getRuntime().availableProcessors();
        ExecutorService service =new ThreadPoolExecutor(
                2,
                max,
                3,
                TimeUnit.SECONDS,
                new LinkedBlockingDeque<>(3),
                Executors.defaultThreadFactory(),
                new ThreadPoolExecutor.AbortPolicy()
        );
```

2、I/O密集型：

在程序中有15个大型任务，io十分占用资源；I/O密集型就是判断我们程序中十分耗I/O的线程数量，大约是最大I/O数的一倍到两倍之间。

## 回顾：手动创建一个线程池



```java
import java.util.concurrent.*;

public class Demo02 {
    public static void main(String[] args) {
        // 自定义线程池！工作 ThreadPoolExecutor
        ExecutorService threadPool = new ThreadPoolExecutor(
                2,
                5,
                3,
                TimeUnit.SECONDS,
                new LinkedBlockingDeque<>(3),
                Executors.defaultThreadFactory(),
                new ThreadPoolExecutor.DiscardPolicy()
        );
        // 最大承载：Deque + max
        // 超过 RejectedExecutionException
        try {
            for (int i = 1; i <= 9; i++) {
                threadPool.execute(()->{
                    System.out.println(Thread.currentThread().getName() + "ok");
                });
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            threadPool.shutdown();
        }
    }
}
```





## 1.synchronized

 在jdk1.5之前 synchronized是一个重量级锁，相对于juc包的lock来讲，synchronized显得比较笨重

###  1、synchronized的作用

1. 原子性

   ​	**原子性指的是一个操作或者多个操作要么全部执行完成要么就不执行，执行中不会被其他操作所干扰**。被synchronized修饰的类或实例的对象都是具有原子性，在操作之前都会先获取到类或对象的锁，在执行结束释放

2. 可见性

   ​	可见性是指多个线程访问同一资源时，该资源的的状态，信息值等对其他线程可见。 synchronized和volatile都具有可见性，synchronized对一个类或者对象加锁时，其他线程想要访问该类或者方法时，必须要先获取该线程的锁，此线程的锁的状态对于其他线程是可见的，并且该线程执行结束，释放锁前，会将对变量的修改刷新到内存中，保证资源的可见性。

3. 有序性

      **有序性指程序的执行按照代码顺序先后执行。**synchronized和volatile都具有有序性，java允许编译器和处理器对对指令进行重排，但指令重排不会影响单线程的顺序，它影响的是多线程并发执行的顺序。synchronized保证了每个时刻只有一个线程访问同步代码块，也就确定了同步代码块里的代码顺序执行，从而保证了有序性。

##### 1.1 synchronized的使用

​	1.**synchronized修饰同步代码块**。锁定指定对象。表示要进入同步代码块，必须要先获取指定对象的锁。

```java
synchronized(this) {  *//业务代码* }
```

​	2. **synchronized修饰普通方法**。 锁定的是调用当前方法的对象实例。要进入该代码块，需要获取当前对象实例的锁。

```java
synchronized void method() {  *//业务代码* }
```

3. **sunchronized修饰静态方法**。 锁的是当前的类，也会作用于所有当前类的对象实例。进入静态同步代码块之前，要先获得class的锁。因为静态成员不属于任何一个实例对象，是属于类，所以线程a调用该类的非静态synchronized方法，不会影响线程b调用该类的静态synchronized方法。因为访问静态方法占用的锁属于当前类的锁，访问非静态方法占用的锁属于当前实例对象的锁。

   ```java
   synchronized void staic method() {  *//业务代码* }
   ```

​      简单总结一下：

`synchronized` 关键字加到 `static` 静态方法和 `synchronized(class)` 代码块上都是是给 Class 类上锁。

`synchronized` 关键字加到实例方法上是给对象实例上锁。

**synchronized使用的一个经典案例----线程安全的单例**

```java
public class Singleton{
    private volatile static Singleton sinleton;
    
    private Singleton(){}
    
    public Singleton getInstance(){
        if(singleton==null){
            synchronized(Singleton.class){
                if(singleton==null){
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}

```



### 2. synchronized的同步原理

​    

```java
public class SynchronizedDemo {
	public void method() {
		synchronized (this) {
			System.out.println("synchronized 代码块");
		}
	}
}
```

通过 JDK 自带的 `javap` 命令查看 `SynchronizedDemo` 类的相关字节码信息：首先切换到类的对应目录执行 `javac SynchronizedDemo.java` 命令生成编译后的 .class 文件，然后执行`javap -c -s -v -l SynchronizedDemo.class`。

![image-20210210160804090](/img/juc/juc/synchronized01.png)

------

从图中可以看出：

`synchronized` 同步语句块的实现使用的是 `monitorenter` 和 `monitorexit` 指令，其中 `monitorenter` 指令指向同步代码块的开始位置， `monitorexit` 指令则指明同步代码块的结束位置。**

当执行 `monitorenter` 指令时，线程试图获取锁也就是获取 **对象监视器 `monitor`** 的持有权。

> 在 Java 虚拟机(HotSpot)中，Monitor 是基于 C++实现的，由ObjectMonitor实现的。每个对象中都内置了一个 `ObjectMonitor`对象。
>
> 另外，**`wait/notify`等方法也依赖于`monitor`对象，这就是为什么只有在同步的块或者方法中才能调用`wait/notify`等方法，否则会抛出`java.lang.IllegalMonitorStateException`的异常的原因。**

在执行`monitorenter`时，会尝试获取对象的锁，如果锁的计数器为 0 则表示锁可以被获取，获取后将锁计数器设为 1 也就是加 1。

在执行 `monitorexit` 指令后，将锁计数器设为 0，表明锁被释放。如果获取对象锁失败，那当前线程就要阻塞等待，直到锁被另外一个线程释放为止。

##### 2.1、synchronized 修饰方法原理

复制代码

```java
public class SynchronizedDemo2 {
	public synchronized void method() {
		System.out.println("synchronized 方法");
	}
}
```

反编译一下：

![image-20210210161841561](/img/juc/juc/synchronized02.png)

`synchronized` 修饰的方法并没有 `monitorenter` 指令和 `monitorexit` 指令，取得代之的确实是 `ACC_SYNCHRONIZED` 标识，该标识指明了该方法是一个同步方法。JVM 通过该 `ACC_SYNCHRONIZED` 访问标志来辨别一个方法是否声明为同步方法，从而执行相应的同步调用。

**简单总结一下**：

`synchronized` 同步语句块的实现使用的是 `monitorenter` 和 `monitorexit` 指令，其中 `monitorenter` 指令指向同步代码块的开始位置，`monitorexit` 指令则指明同步代码块的结束位置。

`synchronized` 修饰的方法并没有 `monitorenter` 指令和 `monitorexit` 指令，取得代之的确实是 `ACC_SYNCHRONIZED` 标识，该标识指明了该方法是一个同步方法。

**不过两者的本质都是对对象监视器 monitor 的获取。**

### 3、synchronized同步概念

##### 1、Java对象头

在JVM中，对象在内存中的布局分为三块区域：**对象头、实例数据和对齐填充**。

![img](/img/juc/juc/synchronized03.png)

`synchronized`用的锁是存在Java对象头里的。

Hotspot 有两种对象头：

- 数组类型，如果对象是数组类型，则虚拟机用3个字宽 （Word）存储对象头
- 非数组类型：如果对象是非数组类型，则用2字宽存储对象头。

对象头由两部分组成

- Mark Word：存储自身的运行时数据，例如 HashCode、GC 年龄、锁相关信息等内容。
- Klass Pointer：类型指针指向它的类元数据的指针。

64 位虚拟机 Mark Word 是 64bit，在运行期间，Mark Word里存储的数据会随着锁标志位的变化而变化。

![img](/img/juc/juc/synchronized04.png)

##### 2、监视器（Monitor）

任何一个对象都有一个Monitor与之关联，当且一个Monitor被持有后，它将处于锁定状态。Synchronized在JVM里的实现都是 基于进入和退出Monitor对象来实现方法同步和代码块同步，虽然具体实现细节不一样，但是都可以通过成对的MonitorEnter和MonitorExit指令来实现。

1. **MonitorEnter指令：插入在同步代码块的开始位置，当代码执行到该指令时，将会尝试获取该对象Monitor的所有权，即尝试获得该对象的锁；**
2. **MonitorExit指令：插入在方法结束处和异常处，JVM保证每个MonitorEnter必须有对应的MonitorExit；**

那什么是Monitor？可以把它理解为 一个同步工具，也可以描述为 一种同步机制，它通常被 描述为一个对象。

与一切皆对象一样，所有的Java对象是天生的Monitor，每一个Java对象都有成为Monitor的潜质，因为在Java的设计中 ，**每一个Java对象自打娘胎里出来就带了一把看不见的锁，它叫做内部锁或者Monitor锁**。

也就是通常说Synchronized的对象锁，MarkWord锁标识位为10，其中指针指向的是Monitor对象的起始地址。在Java虚拟机（HotSpot）中，Monitor是由ObjectMonitor实现的。

### 4、synchronized优化

从JDK5引入了现代操作系统新增加的CAS原子操作（ **JDK5中并没有对synchronized关键字做优化，而是体现在J.U.C中，所以在该版本concurrent包有更好的性能** ），从JDK6开始，就对synchronized的实现机制进行了较大调整，**包括使用JDK5引进的CAS自旋之外，还增加了自适应的CAS自旋、锁消除、锁粗化、偏向锁、轻量级锁这些优化策略**。由于此关键字的优化使得性能极大提高，同时语义清晰、操作简单、无需手动关闭，所以推荐在允许的情况下尽量使用此关键字，同时在性能上此关键字还有优化的空间。

锁主要存在四种状态，依次是：**无锁状态、偏向锁状态、轻量级锁状态、重量级锁状态**，锁可以从偏向锁升级到轻量级锁，再升级的重量级锁。**但是锁的升级是单向的，也就是说只能从低到高升级，不会出现锁的降级**。

![img](/img/juc/juc/synchronized05.png)

##### 1、偏向锁

偏向锁是JDK6中的重要引进，因为HotSpot作者经过研究实践发现，**在大多数情况下，锁不仅不存在多线程竞争，而且总是由同一线程多次获得**，为了让线程获得锁的代价更低，引进了偏向锁。

当一个线程访问同步块并获取锁时，会在对象头和栈帧中的锁记录里存储锁偏向的线程ID，以后该线程在进入和退出同步块时不需要进行CAS操作来加锁和解锁，只需简单地测试一下对象头的Mark Word里是否存储着指向当前线程的偏向锁。

如果测试成功，表示线程已经获得了锁。如果测试失败，则需要再测试一下Mark Word中偏向锁的标识是否设置成1（表示当前是偏向锁）：如果没有设置，则使用CAS竞争锁；如果设置了，则尝试使用CAS将对象头的偏向锁指向当前线程。

偏向锁使用了一种等到竞争出现才释放锁的机制，所以当其他线程尝试竞争偏向锁时， 持有偏向锁的线程才会释放锁。

偏向锁的撤销，需要等待全局安全点（在这个时间点上没有正在执行的字节码）。它会首先暂停拥有偏向锁的线程，然后检查持有偏向锁的线程是否活着， 如果线程不处于活动状态，则将对象头设置成无锁状态；如果线程仍然活着，拥有偏向锁的栈会被执行，遍历偏向对象的锁记录，栈中的锁记录和对象头的Mark Word要么重新偏向于其他线程，要么恢复到无锁或者标记对象不适合作为偏向锁，最后唤醒暂停的线程。

下图中的线 程1演示了偏向锁初始化的流程，线程2演示了偏向锁撤销的流程：

![img](/img/juc/juc/synchronized06.png)

##### 2、轻量级锁

引入轻量级锁的主要目的是 在没有多线程竞争的前提下，减少传统的重量级锁使用操作系统互斥量产生的性能消耗。当关闭偏向锁功能或者多个线程竞争偏向锁导致偏向锁升级为轻量级锁，则会尝试获取轻量级锁。

**（1）轻量级锁加锁**

线程在执行同步块之前，JVM会先在当前线程的栈桢中创建用于存储锁记录的空间，并将对象头中的Mark Word复制到锁记录中，官方称为Displaced Mark Word。然后线程尝试使用 CAS将对象头中的Mark Word替换为指向锁记录的指针。如果成功，当前线程获得锁，如果失败，表示其他线程竞争锁，当前线程便尝试使用自旋来获取锁。

**（2）轻量级锁解锁**

轻量级解锁时，会使用原子的CAS操作将Displaced Mark Word替换回到对象头，如果成 功，则表示没有竞争发生。如果失败，表示当前锁存在竞争，锁就会膨胀成重量级锁。

下图是 两个线程同时争夺锁，导致锁膨胀的流程图：

![img](/img/juc/juc/synchronized07.png)

因为自旋会消耗CPU，为了避免无用的自旋（比如获得锁的线程被阻塞住了），一旦锁升级成重量级锁，就不会再恢复到轻量级锁状态。当锁处于这个状态下，其他线程试图获取锁时， 都会被阻塞住，当持有锁的线程释放锁之后会唤醒这些线程，被唤醒的线程就会进行新一轮的夺锁之争。

##### 3、锁的优缺点比较

各种锁并不是相互代替的，**而是在不同场景下的不同选择**，绝对不是说重量级锁就是不合适的。**每种锁是只能升级，不能降级，即由偏向锁->轻量级锁->重量级锁**，而这个过程就是开销逐渐加大的过程。

> **如果是单线程使用，那偏向锁毫无疑问代价最小**，并且它就能解决问题，连CAS都不用做，仅仅在内存中比较下对象头就可以了；
>
> **如果出现了其他线程竞争**，则偏向锁就会升级为轻量级锁；
>
> **如果其他线程通过一定次数的CAS尝试没有成功**，则进入重量级锁；

锁的优缺点的对比如下表：

| 锁       | 优点                                                         | 缺点                                             | 适用场景                           |
| -------- | ------------------------------------------------------------ | ------------------------------------------------ | ---------------------------------- |
| 偏向锁   | 加锁和解锁不需要额外的消耗，和执行非同步方法仅有纳米级的差距 | 如果线程间存在锁的竞争，会带来额外的锁撤销的消耗 | 适用于只有一个线程访问的同步块场景 |
| 轻量级锁 | 竞争的线程不会阻塞，提高了程序的相应速度                     | 如果始终得不到锁竞争的线程，使用自旋会消耗CPU    | 追求响应时间 同步响应非常快        |
| 重量级锁 | 线程竞争不使用自旋，不会消耗CPU                              | 线程阻塞，响应时间缓慢                           | 追求吞吐量 同步块执行速度较长      |

 



### 2. java内存模型（JMM）

##### 1、java内存模型的基础

   ![image-20230303161134619](..\.\notes\img\juc\juc\jmm.png)

线程之间的共享变量存储在主内存，每个线程都有个本地内存，本地内存存储着该线程读写共享变量的副本，本地内存是JMM的一个抽象概念，并不真实存在。

这两个步骤实际上就是线程a向线程b发消息，而且这个通信过程必定要经过主内存。JMM通过控制主内存与每个线程的本地内存之间的交互，来为Java程序员提供内存可见性的保证。

##### 2、**指令重排序**

![image-20230303161549418](..\..\notes\img\juc\juc\指令重排序.png)

1属于编译器重排序，2和3属于处理器重排序，这些重排序可能会导致多线程程序出现内存可见性问题。对于编译器，JMM的编译器重排序规则会禁止特定类型的编译器重排序。对于处理器重排序，JMM的处理器重排序规则会要求在生成指令序列时，插入特定类型的内存屏障。

编译器和处理器重排序时会遵守数据一致性，不会改变存在数据依赖性关系的两个操作执行顺序。

###### 1、happen-before

​    在JMM中，如果一个操作对于另一个操作可见，那么两个操作之间一定存在happen-before关系。这两个操作可以在一个线程中，也可以在多个线程之间。

两个操作之间具有happen-before关系，并不意味着钱一个操作必须要在后一个操作之前执行。happen-before仅仅要求前一个操作（执行的结果）对后一个操作可见，且钱一个操作按顺序排在第二个操作之前。

###### 2、as-if-serial

意思就是：不管怎么重排序，不能改变程序的执行结果。

as-if-serial使得单线程程序员无需担心重排序会干扰他们，也无需担心内存可见性的问题。但对于多线程来说，对存在控制依赖关系的操作重排序，可能会改变程序的执行结果。

##### 3、顺序一致性

顺序一致性内存模型：1.一个线程中所有操作必须按照程序的顺序执行；2.所有线程都只能看到一个单一的操作执行顺序。在顺序一致性内存模型中，每个操作都必须是原子执行且立刻对所有线程可见。

##### 4、volatile的内存语义

理解volatile特性的一个方法就是把对volatile变量的单个读写，看成使用同一个锁对这些单个读写操作做了同步。

**特性**：

​    1.可见性。对一个volatile变量的读，总能看到任意线程对这个volatile变量的最后写入。

​    2.原子性。对任意单个volatile变量的读写具有原子性。

为了实现volatile的内存语义，编译器在生成字节码时，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序。

##### 5、锁的内存语义

线程A释放锁，线程B获取锁，这个过程实质上就是线程A通过主内存向线程B发送消息。

##### 6、final域的内存语义









