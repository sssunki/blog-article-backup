---
title: 《Thinking in Java》第二十一章————并发  笔记与思考（一）
date: 2019-05-20 19:05:43
tags: Java
categories: Java
---
学期初学习Android的时候，出于时间原因，只通过博客简单的了解了一下Java中与并发编程相关的基本知识，就直接开始了Android相关知识的学习，现在打算结合一些博客，源码深入的读一读《Java编程思想》中关于并发编程的一个章节，并且写几篇文章总结学习中的一些体会，这篇文章是这个系列中的第一章，主要讲并发的特性，基本的线程机制和资源共享中的问题。由于笔者接触编程时间还不长，文章中如有错误，希望读者斧正。
<!--more--> 
# 并发的多面性
不带着目的学习一个东西必然是枯燥的，学习Java中并发编程的使用之前，我们先了解一下它的优势以及一些我们需要面临的问题。
## 速度
我们现在使用的处理器大多数都是多核心处理器，相当于多个处理器在同时运行，对于多处理器中并发带来的速度提升很好理解：并发编程赋予了程序员使用额外的处理器的能力，将程序分为多个片段，分到多个处理器上运行，可以带来速度上的提升，这也是多处理器Web服务器的常见使用方法。当我们使用并发编程时，操作系统或虚拟机承担了把任务分发给不同处理器的任务，这个就不需要我们关心了。
另外，对于单处理器上的任务，并发也可能带来性能上的提升，并且并发通常情况下也都是提高在单处理器上运行的程序的性能。读者可能会感到奇怪。我们首先了解一下阻塞的概念：
> 当程序中的某个任务因为该程序控制范围之外的条件（通常是IO）停滞在了某个环节，不能继续运行，那么我们就说这个程序阻塞了。

然后我们再来探讨速度问题。对于单处理器，并发的开展形式是在任务间不断的切换。在不发生阻塞时，这种方式确实会因为上下文切换带来额外开销，但当阻塞发生时，如果没有并发，整个程序都会停下来，直到外界条件发生改变。对于这种情况。事件驱动的编程是一个常见的例子。程序在需要进行连续的操作时，同时也需要让用户对程序能进行一定程度上的控制，例如一个“退出”按钮。如果我们在每段代码里都对这个按钮的状态进行检查的话肯定是很尴尬的。并发为这个问题提供了一种听起来就像有两个处理器在同时运行的方法：“执行连续操作的同时，返回对剩余部分的控制。”，但实际上还是一个处理器通过不断的切换上下文在运行。

## 改进代码设计
单CPU机器上使用多线程的程序任意时刻仍然只是在执行一个线程中的任务，因此理论上讲，不使用多线程也可以写出同样的程序，但并发提供了一个组织架构上的好处，代码可以得到极大的优化。
## 线程数量和协作式多线程系统
Java默认的线程机制是抢占式的：调动机制会周期性的中断线程，切换上下文，从而为每个线程都提供时间片，使得每个线程都有合理的时间去驱动他的任务。
编程中有一些问题，例如仿真，可能会涉及大量的交互元素，每一个元素“都有其自己的想法”，也就是要视为一个单独的任务，但通常情况下，多线程系统都会将可用的线程限制为一个较小的数字。
解决这个问题的典型方法是协作式多线程系统：程序员有意识的插入某种类型的让步语句，让每个任务自动的放弃控制。这样做有两面性：可以降低切换上下文时带来的开销，同时执行的线程数量理论上也可以不再受限制。但有些协作式系统没有设计为可在多个处理器之间分配任务，这可能会带来困难。
## 其他困难
Java使用的并发系统会共享内存和IO这样的资源，我们需要协调不同线程之间对这些任务的使用，以使得这些资源不会被多个任务同时访问。

并发的特性就先介绍到这里，跟着书本走，我们接下来来分析Java的基本线程机制。
# 基本线程机制
## 定义任务
线程可以驱动任务，Runnable接口为我们提供了描述任务的方式。要定义任务，只需要实现Runnable接口并重写run方法。下面的LiftOff任务将显示发射之前的倒计时：
```java
public class LiftOff implements Runnable {
    protected int countDown = 10;
    private static int taskCount = 0;
    private final int id = taskCount++;
    
    public LiftOff() {
    }

    public LiftOff(int countDown) {
        this.countDown = countDown;
    }

    public String status() {
        return "0" + id + "(" + (countDown > 0 ? countDown : "Liftoff!") + "),";
    }

    public void run(){
        while(countDown-- > 0){
            System.out.print(status());
            Thread.yield();;
        }
    }
}
```
这里解释一下Thread.yield方法，这个方法的调用是对线程调度器的一种建议（线程调度器是Java线程机制的一部分，可以将CPU从一个线程转移到另一个线程），它在声明：这个任务已经执行完最重要的部分了，可以切换给其他任务执行了。在目前它还是完全没必要的，但之后会在实例中产生更有趣的输出：可以从中看到任务换进换出的证据。
只要重写了run方法，这个任务就定义好了，但目前它还只是一个普通的类，不具备任何的线程能力，要实现线程行为，我们必须显式的把一个任务附到线程上。
## Thread类
赋予Runnable对象线程能力的传统方法把它提交给一个Thread构造器来驱动。下面实例展示了这种方法：
```Java
public class Main {

    public static void main(String[] args) {
        Thread t = new Thread(new LiftOff());
        t.start();
        System.out.println("Waiting for LiftOff!");
    }
}
```
Thread构造器需要一个Runnable对象。调用start方法做了两件事：为该线程执行必需的初始化操作，然后在这个新线程中调用Runnable的run方法。运行这段代码时可以看到Waiting for LiteOff在run方法之前执行了，程序并不是顺序执行，这就可以说明程序是在以多线程的方式执行了。
这里是在main方法中开启的新线程，实际上，我们可以在任意一个线程中开启新线程。
我们可以很容易的添加线程去驱动新的任务，下面这段代码中可以看到任务之间是相互呼应的。
```java
public class Main {

    public static void main(String[] args) {
        for(int i = 0; i < 5 ; i ++)
            new Thread(new LiftOff()).start();
        System.out.println("Waiting for LiftOff!");
    }
}
```
不同任务换进换出时混在了一起，这是线程调度器自动控制的，如果你的机器上有多个处理器，线程调度器还会在这些处理器之间默默分发线程。
这里的输出结果可能不同，因为线程调度机制是非确定性的。不同jdk版本之间也会产生巨大的差异，Sun公司也并未提及这些种类JDK之间的差异，因此我们不能依赖于几次实验中产生的一致性，最好还是在编写代码时尽可能的保守。
Thread对象被回收的机制也不同于一般对象，一般对象在不被引用时就可能会被回收，而Thread对象在run完成之前，垃圾回收器无法清除他。
## Executor
Executor（执行器）是Java提供的方便我们管理线程的一个接口，它在客户端和任务执行之间提供了一个间接层，作为中介对象，代替客户端执行任务。并且，Executor允许我们管理多个异步任务的执行，而不需要显式的管理各个线程的生命周期。
下面例子展示了Executor的用法，ExecutorService是一种具有生命周期的Executor，它可以构建恰当的上下文来执行Runnable对象，ExecutorService对象是由Executors的静态方法确定Executor类型并创建的。
```Java
public class CachedThreadPool {
    public static void main(String[] args){
        ExecutorService exec = Executors.newCachedThreadPool();
        for(int i = 0; i < 5 ; i ++){
            exec.execute(new LiftOff());
        }
        exec.shutdown();
    }
}
```
这段代码中，Executors通过静态方法newCachedThreadPool确定好executor的类型创建了ExecutorService对象exec，exec的方法execute接受了一个Runnable对象，开启线程并执行了un方法中描述的任务。shutdown方法用来平滑的关闭exec对象，当这个方法被调用时，exec不再接受新的任务（如果再调用execute方法就会抛出异常），并且等待所有已提交任务完成后关闭自己。
CachedThreadPool方法也可以替换为不同类型的executor，下面例子中的FIxedThreadPool使用了有限的线程集来执行所提交的任务
```Java
public class FixedThreadPool {
    public static void main(String[] args){
        ExecutorService exec = Executors.newFixedThreadPool(5);
        for(int i = 0; i < 5; i++)
            exec.execute(new LiftOff());
        exec.shutdown();
    }
}

```
FixedThreadPool可以一次性预先执行代价高昂的线程分配，在执行任务时，减少了创建线程的消耗，节省了时间。我们之前提到的事件驱动编程中，如果采用这种方式，需要线程的事件处理器可以直接从线程池中获取线程，尽快开始任务执行。我们之前提到的“退出”按钮的例子，如果我们采用这种方式驱动，在点击这个按钮时，就可以尽快开始“退出”任务的执行。
在可能的情况下，Fixed线程池中线程也会被自动复用。
还用一种executor叫SingleThreadExecutor，就像是只有1个线程的FixedThreadPool，对于我们希望在另一个线程中长期运行的任务（比如下载）是很有用的，同时对于一些我们希望在线程中运行的短任务也很方便，例如更新本地或远程日志的小任务，或者是事件分发的线程。
如果提交给SingleThreadExecutor多个任务，那么这些任务将按照提交的顺序排队执行。这里书的作者谈到了和序列化有关的一些东西，但我还不是很懂序列化是个什么，过后再补充。
## 任务的返回值
如果需要从任务中得到返回值，应该使用Callable接口而不是RUnnable接口，Callable是一种具有类型参数的泛型，它的类型参数表示的是从call方法中返回的值，并且必须从ExecutorService.submit方法调用他，下面是一个简单的例子。
```Java
public class TaskWithResult implements Callable<String> {
    private int id;
    public TaskWithResult(int id) {
        this.id = id;
    }
    public String call(){
        return "result of task with result " + id + "\n";
    }
}


public class CallableDemo {
    public static void main(String[] args){
        ExecutorService exec = Executors.newSingleThreadExecutor();
        ArrayList<Future<String>> strings = new ArrayList<>();
        for (int i = 0; i < 3; i++) {
            strings.add(exec.submit(new TaskWithResult(i)));
        }
        try {
            for (Future<String> fs : strings) {
                System.out.print(fs.get());
            }
        }catch (Exception e){
            e.printStackTrace();
        }
        exec.shutdown();
    }
}
```
这里的submit方法会产生Future对象，它用Callable的放回结果的特定类型进行了参数化。这里可以用isDone来查询Future是否完成。完成后可以调用Future对象的get方法获取callable的结果。get方法会产生阻塞，知道结果被获得。
## 休眠
sleep方法可以使任务中止执行给定的时间，本质上sleep也是在造成阻塞，进而影响任务的行为，下面这个例子输出时的行为可以说明这个问题
```Java
public class SleepTask extends LiftOff {
    public void run(){
        try{
            while (countDown-- > 0){
                System.out.print(status());
                TimeUnit.MILLISECONDS.sleep(100)0;
            }
        }catch (Exception e){
            e.printStackTrace();
        }
    }

    public static void main(String[] args){
        ExecutorService exec = Executors.newCachedThreadPool();
        for(int i = 0; i < 5; i++){
            exec.execute(new SleepTask());
        }
        exec.shutdown();
    }
}
```
可以看到，不同于之前的LiftOff直接输出的所有结果，这段代码是在一段时间之后才完成所有的输出的。
这里涉及到了异常不能跨线程传播的问题，这段代码中sleep可能抛出的异常不会传到驱动main的线程中，所以需要在run方法中捕获。
同时，你可能会观察到这段代码的输出行为看上去非常完美，但这依赖于底层的线程机制，在不同的操作系统之间会有差异，所以不要依赖于它，后面会讲到正确的控制方法。（同步控制）
## 优先级
线程的优先级仅仅是给调度器的一个建议，根据环境的不同会有其他很多因素影响线程调度的优先权，按照书上的代码我并没有得到正确的结果，这个问题之后再研究，代码例子如下。
```Java
public class SimplePriorities implements Runnable {
    private int countDown = 5;
    private volatile double d;          // No optimization
    private int prtority;
    public SimplePriorities(int prtority){
        this.prtority = prtority;
    }
    public String toString(){
        return Thread.currentThread() + ": " + countDown;
    }
    public void run(){
        Thread.currentThread().setPriority(prtority);
        while (true){
            for(int i = 1; i < 100000; i++){
                d += (Math.PI + Math.E) / (double) i;
                if(i % 1000 == 0) 
                    Thread.yield();
            }
            System.out.println(this);
            if(--countDown == 0)
                return;
        }
    }
    public static void main(String[] args){
        ExecutorService exec = Executors.newCachedThreadPool();
        for (int i = 0; i < 5; i++){
            exec.execute(new SimplePriorities(Thread.MIN_PRIORITY));
        }
        exec.execute(new SimplePriorities(Thread.MAX_PRIORITY));
        exec.shutdown();
    }
}
```
可以看到，线程的优先级是在run方法开头设置的。在线程内部，可以调用Thread.currentThread获得当前线程的引用。
这里也涉及到了一个移植性问题，不同环境下的优先级数量也不同，通常只有MIN，MAX，NORM三个常量是可以保证具有移植性的。
## 让步
前面我们已经提到，线程的让步工作通常由yield方法来作出，但这只是在建议相同优先级的线程继续运行，没有任何机制保证这个建议会被采纳。实际上，对于大多数重要的控制或在调整应用时，都不能依赖于yield()。
## 守护线程
Java中的线程可以分为用户线程和守护线程，我们之前使用的都是用户线程，守护线程也叫后台线程（Deamon），通常用于在程序运行时提供通用的后台服务（例如垃圾回收机制）。
守护线程和用户线程在正常运行时行为上没有任何区别，但因为守护线程是作为服务用户线程的线程存在的，所以当所有用户线程结束时，守护线程也就没有存在的必要了，会伴随着程序的结束被杀死（即使是被承诺了一定会执行的finally语句也不会再执行）。从下面示例的输出中可以观察到这一现象
```Java
public class Daemons {

    public static void main(String[] args){
        Thread d = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    int i = 0;
                    while(true){
                    System.out.println("run times" + i);
                    i++;
                    TimeUnit.MILLISECONDS.sleep(1000);
                    }
                }catch (Exception e){
                        e.printStackTrace();
                }finally {
                    System.out.println("this won't run");
                }
            }
        });
        d.setDaemon(true);
        d.start();
        try {
            TimeUnit.MILLISECONDS.sleep(5000);
        }catch (Exception e){
            e.printStackTrace();
        }
    }
}
```
这里我们通过setDaemon方法把线程d设置成了守护线程，注意，setDaemon方法必须在线程start之前调用。另外，由守护线程创建的线程会被自动设置为后台线程（即使没有显式的设置）。
## ThreadFactory
Executors类返回一个线程池的静态方法，例如newCachedThreadPool，除了不需要参数的默认版本之外，还有一个可以接受一个ThreadFactory参数的重载，实现ThreadFactory接口可以用于自定义通过Executors产生的线程类型，下面这个例子中利用Executors返回的线程池中的线程就都是我们刚刚提到的守护线程。
```Java
class DeamonThreadFactory implements ThreadFactory{
    public Thread newThread(Runnable r){
        Thread t = new Thread(r);
        t.setDaemon(true);
        return t;
    }
}

public class ThreadFactoryDemo {
    public static void main(String[] args){
        ExecutorService exec = Executors.newCachedThreadPool(new DeamonThreadFactory());
        exec.execute(new Runnable() {
            @Override
            public void run() {
                int i = 0;
                while (true){
                    i++;
                    System.out.println("times of run " + i);
                    try {
                        TimeUnit.MILLISECONDS.sleep(1000);
                    }catch (Exception e){
                        e.printStackTrace();
                    }
                }
            }
        });
        try{
        TimeUnit.MILLISECONDS.sleep(5000);
        }catch (Exception e){
            e.printStackTrace();
        }
    }
}

```
首先我们可以从这段代码中猜想一下ThreadFactory的运作方式，可以看到，ThreadFactory中有一个newThread方法返回一个Thread对象，ThreadFactory对象本身作为参数传递给了newCachedThreadPool，我们可以联想到，newCachedThreadPool内部应该是调用了ThreadFactory的newThread方法创建了线程池，但猜想毕竟只是猜想，下面我们结合源码分析一下默认ThreadFactory的运作方式：
```java
public interface ThreadFactory {
    Thread newThread(Runnable var1);
}
```
默认ThreadFactory接口只有一个newThread方法，用来返回Thread对象。
```Java
public static ExecutorService newCachedThreadPool(ThreadFactory var0) {
    return new ThreadPoolExecutor(0, 2147483647, 60L, TimeUnit.SECONDS, new SynchronousQueue(), var0);
}
```
重载的newCachedThreadPool方法中的ThreadFactory参数被传递给了ThreadPoolExecutor类的构造器，解释一下ThreadPoolExecutor，它是抽象类AbstractExecutorService的子类，AbstractExecutorService是一个实现了ExecutorService的接口。
```Java
public ThreadPoolExecutor(int var1, int var2, long var3, TimeUnit var5, BlockingQueue<Runnable> var6, ThreadFactory var7) {
    this(var1, var2, var3, var5, var6, var7, defaultHandler);
}

public ThreadPoolExecutor(int var1, int var2, long var3, TimeUnit var5, BlockingQueue<Runnable> var6, ThreadFactory var7, RejectedExecutionHandler var8) {
    this.ctl = new AtomicInteger(ctlOf(-536870912, 0));
    this.mainLock = new ReentrantLock();
    this.workers = new HashSet();
    this.termination = this.mainLock.newCondition();
    if (var1 >= 0 && var2 > 0 && var2 >= var1 && var3 >= 0L) {
        if (var6 != null && var7 != null && var8 != null) {
            this.acc = System.getSecurityManager() == null ? null : AccessController.getContext();
            this.corePoolSize = var1;
            this.maximumPoolSize = var2;
            this.workQueue = var6;
            this.keepAliveTime = var5.toNanos(var3);
            this.threadFactory = var7;
            this.handler = var8;
        } else {
              throw new NullPointerException();
            }
        } else {
         throw new IllegalArgumentException();
    }
 }
 ```
 在ThreadPoolExecutor中，传递过来的ThreadFactory被设置成了ThreadPoolExcutor的默认成员，结合上面的分析，这个传入的ThreadFactory最终被设置成了我们创建的ExecutorService类对象的一个默认成员。
```Java
 Worker(Runnable var2) {        // worker是ThreadPoolExecutor的一个内部类
    this.setState(-1);
    this.firstTask = var2;
    this.thread = ThreadPoolExecutor.this.getThreadFactory().newThread(this);
}
```
这里就是ThreadFactory起作用的地方了，结合上面的分析来看，符合我们之前的猜想。
## Join
一个线程可以在其他线程上调用join方法，其效果是把原本“同时”执行的两个线程变为顺序执行，比如线程a和线程b，如果在a线程中调用b.join方法，那么a将会被挂起，直到b执行完毕或被interrupt之后再开始执行。下面了例子展示了其具体行为：
```java
class Sleeper extends Thread{
    private int sleepTime;

    public Sleeper(String name, int sleepTime){
        super(name);
        this.sleepTime = sleepTime;
        start();
    }

    public void run(){
        try{
            TimeUnit.MILLISECONDS.sleep(sleepTime);
        }catch (Exception e){
            System.out.println(getName() + "was interrupted");
            return;
        }
        System.out.println(getName() + " awake");
    }
}

class Joiner extends Thread{
    private Sleeper sleeper;
    public Joiner(String name, Sleeper sleeper){
        super(name);
        this.sleeper = sleeper;
        start();
    }
    public void run(){
        try{
            sleeper.join();
        }catch (Exception e){
            System.out.println("join fail");
        }
        System.out.println(getName() + " join");
    }
}
public class JoinDemo {
    public static void main(String[] args) {
        Sleeper
                sleepy = new Sleeper("Sleepy", 1500),
                grumy = new Sleeper("Grumy", 1500);
        Joiner
                dopey = new Joiner("Dopey", sleepy),
                doc = new Joiner("Doc", grumy);
        grumy.interrupt();
    }
}
```
我们以dopey和sleepy为例子分析一下整个过程：首先声明了sleepy，并且sleepy开始执行，然后声明了dopey，dopey开始执行时，在dopey的线程内调用了sleepy的join方法，这意味着正在执行着的sleepy加入到了dopey的执行过程中，dopey被挂起，sleepy执行完毕后，dopey才继续开始执行。
join方法也可以接受参数，读者可以自己观察那种情况下的现象。
## 异常
之前我们提到过，在一个线程里抛出的异常无法在其他线程捕捉到，为了解决这个问题，javaSE5提供了新的接口Thread.UncaughtExceptionHandler，下面例子展示了这个接口的使用方法
```java
class ExceptionThread implements Runnable{
    public void run(){
        throw new RuntimeException();
    }
}

class mUncaughtExceptionHandler implements Thread.UncaughtExceptionHandler{
    public void uncaughtException(Thread t, Throwable e){
        System.out.println("get it");
    }
}

public class UncaughtExceptionHandlerDemo {
    public static void main(String[] args){
        Thread t = new Thread(new ExceptionThread());
        t.setUncaughtExceptionHandler(new mUncaughtExceptionHandler());
        t.start();
    }
}
```
这段代码中，我们只需要给线程t设置一个异常处理器，就可以捕获到线程中会抛出的异常。
除了这种方式外，也可以通过Thread类的静态方法setDefaultUncaughtExceptionHandler给整个程序设置默认的uncaughtExceptionHandler，如果一个线程没有专门的设置异常处理器，这个处理器就会被调用。

Java的基本线程机制基本上就是这些内容，有些地方还需要结合源码理解，之后可能会写，读者也可以去翻阅文档或者源码，下面我们开始了学习资源共享相关的知识。
# 共享受限资源
## 一个错误实例
这一节的例子是这样的：一个任务产生偶数，其他任务消费这些数字，而在这个例子中我们简化一下消费者的任务，它只需要检查偶数的有效性。
首先我们需要定义消费者任务，在这里就是EvenChecker。我们这里需要定义两个类，一个就是EvenChecker，另一个是用于生成数字，并且提供EvenChecker需要知道的方法next的抽象类。代码如下
```java
public abstract class IntGenerator {
    private volatile boolean canceled = false;
    public abstract int next();

    public void cancal(){
        canceled = true;
    }

    public boolean isCanceled(){
        return canceled;
    }
}

public class EvenChecker implements Runnable {
    private IntGenerator generator;
    private final int id;
    
    public EvenChecker(IntGenerator generator, int id){
        this.generator = generator;
        this.id = id;
    }
    
    public void run(){
        while(!generator.isCanceled()){
            int val = generator.next();
            if(val % 2 == 0){
                System.out.println(val + "not even");
                generator.cancal();
            }
        }
    }
    
    public static void test(IntGenerator gp, int count){
        System.out.println("Press Control-C to exit");
        ExecutorService exec = Executors.newCachedThreadPool();
        for(int i = 0; i < count; i++){
            exec.execute(new EvenChecker(gp, i));
        }
        exec.shutdown();
    }
    
    public static void test(IntGenerator gp){
        test(gp, 10);
    }
}
```
然后我们要定义一个IntGenerator的子类EvenGenerator，再用EvenChecker的test方法去检测它
```java
public class EvenGenerator extends IntGenerator {
    private int currentEvenValue = 0;
    public int next(){
        ++currentEvenValue;
        ++currentEvenValue;
        return currentEvenValue;
    }
    public static void main(String[] args){
        EvenChecker.test(new EvenGenerator());
    }
}
```
运行时可以发现，这段代码产生的输出很异常，而且多次运行的输出结果之间没有任何规律。我们分析一下这段代码的逻辑：抽象类IntGenerator的存在是为了给EvenChecker提供必须要了解的方法next，EvenGenerator是实际上用于产生被消费者消费的数据的类，在main方法中，静态方法test中开启了线程，执行run中定义的任务，任务终止的条件为：检测到奇数就将Generator设置为cancel，然后跳出循环，终止运行。EvenGennerator中的next方法看似每次生成的都是偶数，我们的程序应该无限循环下去，但我们的程序最终还是终止了。因为实际上当有多个线程同时访问它时会产生危险，比如线程a访问next方法时，由于多线程的不确定性，currentEvenValue只自増了一次，这个线程就被挂起，切换到了线程b，线程b对next访问如果顺利完成，在线程b中currentEvenValue就自増了两次，那么在这次next返回之前这个值就自増了三次，next返回的数就会是一个奇数，程序就会退出。
这就是访问共享资源时会带来的问题，下面我们来寻找解决这个问题的办法。
## 解决共享资源竞争
防止这种问题发生的方法是当资源被一个任务使用时，在其上加锁，第一个访问这个资源的任务会锁定这项资源，在解锁之前，其他任务无法访问这项资源。
几乎所有并发模式在解决冲突问题的时候，采用的都是序列化访问共享资源的方式，通常这通过在代码前面加上一条锁语句实现，这会使得一段时间内只有一个任务可以运行这段代码。
因为锁语句产生了一种互相排斥的效果所以这种机制常成为互斥量。
### synchronized
java提供了synchronized关键字，为防止资源冲突提供了内置的支持。当任务要执行被synchronized保护的代码片段的时候，它会检测锁是否可用，然后获取锁，执行代码，然后释放锁。
共享资源一般是以对象的形式存在的内存片段，也可以时文件，IO端口，甚至打印机。要控制对共享资源的访问，需要先把它包装在一个对象里，然后把访问这个资源的方法标记为synchronized，如果方法A被做了这样的处理，那么当线程A访问方法A时，在方法返回之前，其他任何要访问这个方法的线程都会阻塞。
并且锁是归属于对象的，所有对象都有一个单一的锁，并且一个对象中所有synchroized方法共享同一个锁，我们假设有一个共享资源对象A，如果对象A有两个synchroized方法f()和g()，当一个线程访问方法f时，整个对象都会被加锁，也就是说，其他线程不仅不能访问f方法，也不能访问g方法。这与前面描述的，
一个任务可以多次获得对象的锁，换句话说，一个对象可被多次加锁，比如这个任务访问了对象A的f方法后，在f方法中又访问了g方法，那么这个任务就会获得两次这个对象的锁，也就说这个对象会被加锁两次。JVM会跟踪对象被加锁的次数，当一个对象被加锁时，计数+1,当任务从synchroized方法中返回时，也就是说释放锁时，计数-1，当这个对象计数为0时，才可以被其他任务访问。
把变量设置为private是必要的，不然synchroized也不能保证不会产生冲突。
下面我们来看例子：为上一节中的EvenGenerator加上synchroized语句：
```java
public class EvenGenerator extends IntGenerator {
    private int currentEvenValue = 0;
    public synchronized int next(){
        ++currentEvenValue;
        ++currentEvenValue;
        return currentEvenValue;
    }
    public static void main(String[] args){
        EvenChecker.test(new EvenGenerator());
    }
}
```
我们可以看到，这种情况下程序就不会被终止了，说明没有遇到我们上一节遇到的问题。
### Lock对象
Java中提供了另一种方式实现锁：显式的创建Lock对象，并显式的进行锁定，释放工作。下面是一个例子：
```java
public class EvenGenerator extends IntGenerator {
    private int currentEvenValue = 0;
    private Lock lock = new ReentrantLock();
    public int next(){
        try{
        lock.lock();
        ++currentEvenValue;
        ++currentEvenValue;
        return currentEvenValue;
        }finally {
            lock.unlock();
        }
    }
    public static void main(String[] args){
        EvenChecker.test(new EvenGenerator());
    }
}
```
可以看到，这种方式写出的代码不如synchroized优雅，而且需要使用try-finally语句（为什么要使用是很显然的）。

<- to be continue

