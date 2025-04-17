# Java

## 一、Java多线程

### 1. Java内存模型

在学习Java内存模型之前需要先了解JVM的内存结构 [ [跳转到 JVM 内存结构](#JVM内存结构) ]。

Java内存模型（JMM）是Java语言规范的一部分，定义了多线程环境下共享变量的访问规则。

#### 1.1 并发编程特性

**原子性**

​	一次操作或者多次操作，要么所有的操作全部都得到执行并且不会受到任何因素的干扰而中断，要么都不执行。

**可见性**

​	当一个线程对共享变量进行了修改，那么另外的线程都是立即可以看到修改后的最新值。

**有序性**

​	由于指令重排序问题，代码的执行顺序未必就是编写代码时候的顺序。

> **指令重排序可以保证串行语义一致，但是没有义务保证多线程间的语义也一致** ，所以在多线程下，指令重排序可能会导致一些问题。



#### 1.2 Java并发问题产生原因

##### 1.1.1 多线程并发内存结构

> **CPU缓存模型**
>
> ​	为了缓解CPU和内存间读写速度差异导致的性能问题，现代计算机通常在CPU和内存间添加三级缓存，越接近CPU的缓存读写越快，但存储容量越小，如下图所示。
>
> ![cpu_cache_njkcnad](C:\Users\陈浩杰\Desktop\Java开发笔记\我的笔记\images\cpu_cache_njkcnad.png)
>
> ​	**CPU缓存的工作方式：** 先将内存数据加载到CPU缓存中，当 CPU 需要用到的时候就可以直接从CPU缓存中读取数据，当运算完成后，再将运算得到的数据写回内存中。但是，这样存在**内存缓存不一致性的问题** ！
>
> ​	**CPU 为了解决内存缓存不一致性问题可以通过制定缓存一致协议（比如 [MESI 协议](https://zh.wikipedia.org/wiki/MESI协议)）或者其他手段来解决。** 

​	Java多线程并发执行时，每个线程会有一个本地内存，同时所有线程会共用主内存，结构如下图：

<img src="C:\Users\陈浩杰\Desktop\Java开发笔记\我的笔记\images\jvm_cmskaf.jpg" alt="jvm_cmskaf" style="zoom: 50%;" />

​	**主内存**：所有线程创建的实例对象都存放在主内存中，不管该实例对象是成员变量，还是局部变量，类信息、常量、静态变量都是放在主内存中。为了获取更好的运行速度，虚拟机及硬件系统可能会让工作内存优先存储于寄存器和高速缓存中。

​	**本地内存**：每个线程都有一个私有的本地内存，本地内存存储了该线程以读 / 写共享变量的副本。每个线程只能操作自己本地内存中的变量，无法直接访问其他线程的本地内存。如果线程间需要通信，必须通过主内存来进行。本地内存是 JMM 抽象出来的一个概念，并不真实存在，它涵盖了缓存、写缓冲区、寄存器以及其他的硬件和编译器优化。

​	若没有JMM，则各个线程的本地内存与主内存间的交互会出现可见性问题，最终导致程序运行错误。



##### 1.1.2 重排序

​	Java 源代码会经历 **编译器优化重排 —> 指令并行重排 —> 内存系统重排** 的过程，最终才变成操作系统可执行的指令序列。

**（1）编译器重排序**

​	针对程序代码语而言，编译器可以在不改变单线程程序语义的情况下，可以对代码语句顺序进行调整重新排序。

**（2）指令集并行的重排序**

​	这个是针对于CPU指令级别来说的，处理器采用了指令集并行技术来将多条指令重叠执行，如果不存在数据依赖性，处理器可以改变语句对应的机器指令执行顺序。

**（3）内存重排序**

​	因为CPU缓存使用缓冲区的方式(Store Buffere )进行延迟写入，这个过程会造成多个CPU缓存可见性的问题，这种可见性的问题导致结果的对于指令的先后执行显示不一致，从表面结果上来看好像指令的顺序被改变了，内存重排序其实是造成可见性问题的主要原因所在。



#### 1.3 JMM规则

##### 1.2.1 八种同步操作

![jmm_cnsknv](C:\Users\陈浩杰\Desktop\Java开发笔记\我的笔记\images\jmm_cnsknv.png)

- **锁定（lock)**: 作用于主内存中的变量，将他标记为一个线程独享变量。

- **解锁（unlock）**: 作用于主内存中的变量，解除变量的锁定状态，被解除锁定状态的变量才能被其他线程锁定。
- **read（读取）**：作用于主内存的变量，它把一个变量的值从主内存传输到线程的工作内存中，以便随后的 load 动作使用
- **load(载入)**：把 read 操作从主内存中得到的变量值放入工作内存的变量的副本中。
- **use(使用)**：把工作内存中的一个变量的值传给执行引擎，每当虚拟机遇到一个使用到变量的指令时都会使用该指令。
- **assign（赋值）**：作用于工作内存的变量，它把一个从执行引擎接收到的值赋给工作内存的变量，每当虚拟机遇到一个给变量赋值的字节码指令时执行这个操作。
- **store（存储）**：作用于工作内存的变量，它把工作内存中一个变量的值传送到主内存中，以便随后的 write 操作使用。
- **write（写入）**：作用于主内存的变量，它把 store 操作从工作内存中得到的变量的值放入主内存的变量中。

**规则**

- read和load，write和store必须成对出现，顺序执行（但不用连续执行）
- assign操作不允许丢弃，即，工作内存中变量改变必须同步给主内存
- use前必须有load，store前必须有assign
- 同一时间一个变量只能被一个线程lock，但该线程可对其lock多次，lock多少次，必须对应unlock对应次数才能解锁
- 如果一个线程lock了某个变量，改变量在工作内存中的值会被清空，使用前必须
- unlock前必须要把该变量写回主存



##### 1.2.2 happens-before规则

​	JDK1.5版本中的Java内存模型中引入了Happens-Before原则。如果两个操作不满足任意一个 happens-before 规则，那么这两个操作就没有顺序的保障，JVM 可以对这两个操作进行重排序。

![happens_before_sanjkvds](.\images\happens_before_sanjkvds.png)

- **程序次序规则**（Program Order Rule）：在一个线程内，按照程序代码顺序，书写在前面的操作先行发生于书写在后面的操作。准确地说，应该是控制流顺序而不是程序代码顺序，因为要考虑分支、循环等结构。
- **管程锁定规则**（Monitor Lock Rule）：一个unlock操作先行发生于后面对同一个锁的lock操作。这里必须强调的是同一个锁，而“后面”是指时间上的先后顺序。
- **volatile变量规则**（Volatile Variable Rule）：对一个volatile变量的写操作先行发生于后面对这个变量的读操作，这里的“后面”同样是指时间上的先后顺序。
- **线程启动规则**（Thread Start Rule）：Thread对象的start()方法先行发生于此线程的每一个动作。
- **线程终止规则**（Thread Termination Rule）：线程中的所有操作都先行发生于对此线程的终止检测，我们可以通过Thread.join()方法结束、Thread.isAlive()的返回值等手段检测到线程已经终止执行。
- **线程中断规则**（Thread Interruption Rule）：对线程interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生，可以通过Thread.interrupted()方法检测到是否有中断发生。
- **对象终结规则**（Finalizer Rule）：一个对象的初始化完成（构造函数执行结束）先行发生于它的finalize()方法的开始。
- **传递性**（Transitivity）：如果操作A先行发生于操作B，操作B先行发生于操作C，那就可以得出操作A先行发生于操作C的结论。



##### 1.2.3 内存屏障

​	在复杂的多线程环境下，编译器和处理器是根本无法分析代码指令的依赖关系的，因此程序员需要显示的告诉编译器和处理器哪些地方是存在逻辑依赖的，这些地方不能进行重排序。

​	**编译器层面**和**CPU层面**都提供了一套内存屏障来禁止重排序的指令，编码人员需要识别存在数据依赖的地方加上一个内存屏障指令，那么此时计算机将不会对其进行指令优化。



### 2. 线程运行状态

#### 1.1 线程运行状态介绍

##### 1.1.1 NEW

处于 NEW 状态的线程此时尚未启动。这里的尚未启动指的是还没调用 Thread 实例的`start()`方法。



##### 1.1.2 RUNNABLE

表示当前线程正在运行中。处于 RUNNABLE 状态的线程在 Java 虚拟机中运行，也有可能在等待 CPU 分配资源。

也就是说，Java 线程的**RUNNABLE**状态其实包括了操作系统线程的**ready**和**running**两个状态。



##### 1.1.3 BLOCKED

阻塞状态。处于 BLOCKED 状态的线程正等待锁的释放以进入同步区。



##### 1.1.4 WAITING

等待状态。处于等待状态的线程变成 RUNNABLE 状态需要其他线程唤醒。



##### 1.1.5 TIMED_WAITING

超时等待状态。线程等待一个具体的时间，时间到后会被自动唤醒。

调用如下方法会使线程进入超时等待状态：

- `Thread.sleep(long millis)`：使当前线程睡眠指定时间；
- `Object.wait(long timeout)`：线程休眠指定时间，等待期间可以通过`notify()`/`notifyAll()`唤醒；
- `Thread.join(long millis)`：等待当前线程最多执行 millis 毫秒，如果 millis 为 0，则会一直执行；
- `LockSupport.parkNanos(long nanos)`： 除非获得调用许可，否则禁用当前线程进行线程调度指定时间；[LockSupport](https://javabetter.cn/thread/LockSupport.html) 我们在后面会细讲；
- `LockSupport.parkUntil(long deadline)`：同上，也是禁止线程进行调度指定时间；

我们继续延续上面的例子来解释一下 TIMED_WAITING 状态：

到了第二天中午，又到了饭点，你还是到了窗口前。

突然间想起你的同事叫你等他一起，他说让你等他十分钟他改个 bug。

好吧，那就等等吧，你就离开了窗口。很快十分钟过去了，你见他还没来，你想都等了这么久了还不来，那你还是先去吃饭好了。

这时你还是线程 t1，你改 bug 的同事是线程 t2。t2 让 t1 等待了指定时间，此时 t1 等待期间就属于 TIMED_WATING 状态。

t1 等待 10 分钟后，就自动唤醒，拥有了去争夺锁的资格。



##### 1.1.6 TERMINATED

终止状态。此时线程已执行完毕。



#### 1.2 线程运行状态变化

![thread_nvdknj](.\images\thread_nvdknj.png)

##### 1.2.1 NEW->RUNNABLE

- 调用Thread的start()方法即可。

> **注意：**Java不允许一个线程重复使用start()来启动，当且仅当线程状态为NEW时才允许启动线程，而线程启动后线程状态就不再是NEW，且不会再变成NEW状态，start()方法源码如下：
>
> ```java
> // 使用synchronized关键字保证这个方法是线程安全的
> public synchronized void start() {
>     // threadStatus != 0 表示这个线程已经被启动过或已经结束了
>     // 如果试图再次启动这个线程，就会抛出IllegalThreadStateException异常
>     if (threadStatus != 0)
>         throw new IllegalThreadStateException();
> 
>     // 以下代码暂不关注
>     group.add(this);
> 
>     boolean started = false;
>     try {
>         start0();
>         started = true;
>     } finally {
>         try {
>             if (!started) {
>                 group.threadStartFailed(this);
>             }
>         } catch (Throwable ignore) {
>         }
>     }
> }
> ```
>
> 可以看到，在`start()`内部，有一个 threadStatus 变量。如果它不等于 0，调用`start()`会直接抛出异常。
>
> 测试代码如下，运行后会因为两次调用一个Thread的start()方法而报错：
>
> ```java
> @Test
> public void testStartMethod() {
>     Thread thread = new Thread(() -> {});
>     thread.start(); // 第一次调用
>     thread.start(); // 第二次调用
> }
> ```



##### 1.2.2 RUNNABLE->BLOCK

- `synchronized` 块或方法：当多个线程尝试获取同一把锁时，未获取到锁的线程将被阻塞。

>三种形式：
>
>1. synchronized修饰的代码块
>
>   ```java
>   Object o = new Object();
>   synchronized(o) {
>       //...
>   }
>   ```
>
>   
>
>2. synchronized修饰的非静态方法
>
>   ```java
>   public synchronized void set() {}
>   ```
>
>   
>
>3. synchronized修饰的静态方法
>
>
>
>
>
>
>
>

- `ReentrantLock.lock()`：尝试获取锁，如果锁被其他线程持有，则当前线程被阻塞。



##### 1.2.3 BLOCK -> RUNNABLE

- 锁被释放：当持有锁的线程执行完同步块或方法，释放锁后，等待的线程将有机会获取锁并变为可运行状态。



##### 1.2.4 RUNNABLE -> WATING

- `Object.wait()`：使当前线程处于等待状态直到另一个线程唤醒它；
- `Thread.join()`：等待线程执行完毕，底层调用的是 Object 的 wait 方法；
- `LockSupport.park()`：除非获得调用许可，否则禁用当前线程进行线程调度。



##### 1.2.5 WAITING -> RUNNABLE

- `Object.notify()` 或 `Object.notifyAll()`：唤醒在此对象监视器上等待的单个或所有线程。



##### 1.2.6 RUNNABLE -> TIMED_WAITING

- `Thread.sleep(long)`：使当前线程休眠指定的时间。
- `Object.wait(long)`：使当前线程等待，直到其他线程调用此对象的 `notify()` 或 `notifyAll()` 方法，或者超过指定的时间。
- `Thread.join(long)`：等待线程执行完毕或超过指定的时间。
- `LockSupport.parkNanos()`：禁用当前线程进行线程调度，除非获得许可，或者超过指定的时间。
- `LockSupport.parkUntil()`：禁用当前线程进行线程调度，直到指定的时间，或者获得许可。



##### 1.2.7 TIMED_WAITING -> RUNNABLE

- 等待时间结束：当指定的等待时间结束，线程将变为可运行状态。
- `Object.notify()` 或 `Object.notifyAll()`：唤醒在此对象监视器上等待的单个或所有线程。
- `LockSupport.unpark(Thread)`：唤醒指定的线程。



##### 1.2.8 RUNNABLE -> RUNNABLE

- 线程调度：操作系统可能会在任何时候调度线程，即使线程已经在运行状态。这通常是为了公平地分配CPU时间给所有线程。
- `yield()`方法：事实上是调用了sleep(0)，但仅仅只是给CPU调度器一个提示(hint)，CPU调度器可能会忽略该提示，若没忽略提示，则让线程下CPU，切换其他线程占用CPU。



##### 1.2.9 RUNNABLE -> TERMINATED

- `run()` 方法结束：当线程的 `run()` 方法执行完毕，或者抛出未捕获的异常，线程将结束其生命周期，变为终止状态。



### 3. 创建线程的方式

#### 1.1 继承Thread类

> Thread类继承自**Runnable接口**。
>
> Thread通过start()方法启动线程，源码如下：
>
> ```java
> // 使用synchronized关键字保证这个方法是线程安全的
> public synchronized void start() {
>     if (threadStatus != 0)
>         throw new IllegalThreadStateException();
> 
>     group.add(this);
> 
>     boolean started = false;
>     try {
>         // 使用native方法启动这个线程
>         start0();
>         // 如果没有抛出异常，那么started被设为true，表示线程启动成功
>         started = true;
>     } finally {
>         // 在finally语句块中，无论try语句块中的代码是否抛出异常，都会执行
>         try {
>             // 如果线程没有启动成功，就从线程组中移除这个线程
>             if (!started) {
>                 group.threadStartFailed(this);
>             }
>         } catch (Throwable ignore) {
>             // 如果在移除线程的过程中发生了异常，我们选择忽略这个异常
>         }
>     }
> }
> ```
>
> 由Thread.start()源码可知，创建一个继承自Thread类的自定义类，通过覆写run()方法即可自定义线程运行执行的流程。

自定义一个类继承自Thread类，覆写run()方法，即可创建一个线程。通过调用自定义类对象的start()方法，即可启动线程。

举个例子：

```java
public class MyThread extends Thread{
    public void run() {
        for(int i = 0; i < 100; i++) {
            System.out.println(currentThread() + ": " + i);  // currentThread()获取当前线程名称
        }
    }
}
```

测试验证如下：

```java
public static void main(String[] args){
    for(int i = 0; i < 5; i++) {
        new MyThread().start();
    }
}
```

执行结果如下图，可以看到多个线程交替执行：

![thread_dsavdfg](.\images\thread_dsavdfg.png)

#### 1.2 实现Runnable接口

> 由1.2.可以知道，Thread类的start()方法底层调用run()方法，而默认run()方法实现源码如下：
>
> ```java
> @Override
> public void run() {
>     if (target != null) {
>         target.run();
>     }
> }
> ```
>
> 其中，tartget是Thread类的一个private属性，类型为Runnable，可以通过如下构造函数来赋值：
>
> ```java
> // 带Runnable类型参数的构造函数
> public Thread(Runnable target) {
>     init(null, target, "Thread-" + nextThreadNum(), 0);
> }
> ```
>
> 由如上源码可以知道，Thread类的run()方法的默认实现方式会检查是否有传入Runnable对象，有的话则执行Runnable对象的run()方法；没有的话则run()方法可以视为空方法，由此可以得到第二个构造线程的方法。

自定义一个类实现Runnable接口，覆写Runnable接口的run()方法，接着将自定义类对象作为参数创建一个Thread对象，调用Thread对象的start()方法即可启动线程。

举个例子：

```java
public class MyThread implements Runnable{
    public void run() {
        for(int i = 0; i < 100; i++) {
            System.out.println(Thread.currentThread() + ": " + i);  // currentThread()获取当前线程名称
        }
    }
}
```

测试验证如下：

```java
public static void main(String[] args){
    MyThread t = new MyThread();
    for(int i = 0; i < 5; i++) {
        new Thread(t).start();
    }
}
```

执行结果如下图，可以看到多个线程交替执行：

![thread_dsavdfg](.\images\thread_dsavdfg.png)



#### 1.3 实现Callable接口

通过前两种创建线程方式的源码解析可以发现，由于run()方法返回值为void，因此线程无法返回结果，由此引出第三种线程创建方式。

> Callable接口：内部仅仅声明了一个call()方法。
>
> 
>
> Future接口：定义了存储和操作运行结果的一套架构。
>
> 
>
> RunnableFuture接口：直接继承了Runnable接口和Future接口，没有额外添加新代码。
>
> 
>
> FutureTask类：实现了RunnableFuture接口，并将Runnable接口和Future接口两个接口架构进行实现和整合，使其既可以作为runnable对象来运行，也可以对线程运行结果进行操作。内部定义了一个Callable类型属性callable，可以借助如下构造函数赋值：
>
> ```java
> public FutureTask(Callable<V> callable) {
>     if (callable == null)
>         throw new NullPointerException();
>     this.callable = callable;
>     this.state = NEW; 
> }
> ```
>
> FutureTask覆写了run()方法，若属性callable不为空并且状态为new，则运行callable的call()方法（这里call()方法的作用相当于run()方法，只是run()方法用来封装获取结果的逻辑了，只能用call()方法来运行线程逻辑了），并将运行结果进行存储，源码如下：
>
> ```java
> public void run() {
> 	// 依旧是对线程状态判断，要求为新建态new
>     if (state != NEW ||
>         !UNSAFE.compareAndSwapObject(this, runnerOffset,
>                                      null, Thread.currentThread()))
>         return;
>     try {
>         Callable<V> c = callable;
>         if (c != null && state == NEW) {
>             V result;
>             boolean ran;
>             try {
>             	// 将线程运行结果存储在result属性中，并设置状态为运行成功
>                 result = c.call();
>                 ran = true;
>             } catch (Throwable ex) {
>             	// 运行异常则设置异常状态
>                 result = null;
>                 ran = false;
>                 setException(ex);
>             }
>             if (ran)
>                 set(result);
>         }
>     } finally {
>         runner = null;
>         int s = state;
>         if (s >= INTERRUPTING)
>             handlePossibleCancellationInterrupt(s);
>     }
> }
> ```
>
> 

自定义一个类实现Callable接口，覆写Callable接口的call()方法，接着将自定义类对象作为参数创建一个FutureTask对象，然后将FutureTask对象作为参数创建一个Thread对象，调用Thread对象的start()方法即可启动线程，线程运行结果可以通过FutureTask对象来获取（原因已在上面分析，具体可以去参考源码）。

举个例子：

```java
public class MyCallable implements Callable<String> {
    @Override
    public String call() throws Exception {
        for(int i = 0; i < 100; i++) {
            System.out.println(Thread.currentThread() + ": " + i);  // currentThread()获取当前线程名称
        }
        return Thread.currentThread().getName();
    }
}
```

测试验证如下：

```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
    MyCallable c = new MyCallable();
    List<FutureTask<String>> ftList = new ArrayList<>();
    for(int i = 0; i < 5; i++) {
        FutureTask<String> ft = new FutureTask<>(c);
        ftList.add(ft);
        Thread t = new Thread(ft);
        t.start();
    }
    for(FutureTask<String> ft: ftList) {
        System.out.println(ft.get());
    }
}
```

执行结果如下两图，可以看到多个线程交替运行，并且都有返回值：

![thread_ankvsn](.\images\thread_ankvsn.png)

![thread_ahrejni](.\images\thread_ahrejni.png)

### 4. 线程组







## 二、JVM

#### 1. JVM内存结构<a id="JVM内存结构"></a>

了解Java内存模型之前，需要先了解JVM内存结构（注意二者区别），JVM内存结构如下图所示，其中方法区和堆是所有线程共享，虚拟机栈、本地方法栈、程序计数器为线程私有。

![jmm_dsadvnk](.\images\jmm_dsadvnk.png)

##### 1.1 程序计数器

​	程序计数器用来存放当前线程接下来将要执行的字节码指令、分支、循环、跳转、异常处理等信息。在任何时候CPU只执行其中一个线程中的指令，为了能够在多线程并发执行发生进程切换时能够回到线程正确的执行位置，每个线程都需要有一个程序计数器，并且各程序计数器之间互不影响，因此程序计数器为线程私有的。

##### 1.2 虚拟机栈

​	虚拟机栈是线程私有，它的生命周期与线程相同，是在JVM运行时所创建，在线程中，方法在执行的时候都会创建一个名为栈帧的数据结构，主要用于存放局部变量表、操作栈、动态链接、方法出口等信息，如下图所示，方法的调用对应着栈帧在虚拟机栈中的压栈和弹栈过程。

![jvm_csanio](.\images\jvm_csanio.jpg)

##### 1.3 本地方法栈

​	本地方法栈是线程私有的，Java中提供了调用本地方法的接口（Java Native Interface），也就是C/C++程序，在线程的执行过程中，经常会碰到调用JNI方法的情况，JVM为本地方法所划分的内存区域便是本地方法栈。

##### 1.4 堆内存

​	堆内存是JVM中最大的一块内存区域，被所有的线程所共享，Java在运行期间创建的所有对象几乎都存放在堆内存中，并且堆内存区是垃圾回收器重点照顾的区域，因此有些时候堆内存也被称为“GC堆”。

​	堆内存一般会被细分为新生代和老年代，更细致的划分为Eden区、From Survivor区和To Survivor区，如下图所示。

![jvm_snckas](C:\Users\陈浩杰\Desktop\Java开发笔记\我的笔记\images\jvm_snckas.webp)

##### 1.5 方法区

​	方法区又叫持久代，被所有线程共享，主要用于存储已经被虚拟机加载的类信息、常量、静态变量、即时编译器（JIT）编译后的代码等数据。方法区在JVM启动时被创建，大小固定，因此容易出现方法区内存分配不足而内存溢出。

​	Java虚拟机规范将方法区划分为堆内存的一个逻辑分区。

​	在HotSpot JVM中，方法区被细分为持久代和代码缓存区，代码缓存区主要用于存储编译后的本地代码以及JIT编译器生成的代码。

##### 1.6 元空间

​	元空间（Meta Space）自JDK1.8版本之后取代了方法区，元空间同样是堆内存的一部分，相比较于方法区，元空间存在于本地内存而不是虚拟机内部，并且元空间的大小是动态的，因此不容易出现内存溢出的情况。



