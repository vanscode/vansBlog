# LockSupport
> LockSupport是什么？

```
模块  java.base
软件包  java.util.concurrent.locks
Class LockSupport
java.lang.Object
java.util.concurrent.locks.LockSupport
public class LockSupport
extends Object
用于创建锁和其他同步类的基本线程阻塞原语。
该类与使用它的每个线程关联一个许可证（在Semaphore类的意义上）。 如果许可证可用，将立即返回park ，并在此过程中消费; 否则可能会阻止。 如果尚未提供许可，则致电unpark获得许可。 （与Semaphores不同，许可证不会累积。最多只有一个。）可靠的使用需要使用volatile（或原子）变量来控制何时停放或取消停放。 对于易失性变量访问保持对这些方法的调用的顺序，但不一定是非易失性变量访问。

方法park和unpark提供了阻止和解除阻塞线程的有效方法，这些线程没有遇到导致不推荐使用的方法Thread.suspend和Thread.resume无法用于此类目的的问题：一个线程调用park和另一个线程尝试unpark将保留活跃性，由于许可证。 此外，如果调用者的线程被中断，则会返回park ，并且支持超时版本。 park方法也可以在任何其他时间返回，“无理由”，因此通常必须在返回时重新检查条件的循环内调用。 在这个意义上， park可以作为“忙碌等待”的优化，不会浪费太多时间旋转，但必须与unpark配对才能生效。

三种形式的park每个也支持blocker对象参数。 在线程被阻塞时记录此对象，以允许监视和诊断工具识别线程被阻止的原因。 （此类工具可以使用方法getBlocker(Thread)访问阻止程序 。）强烈建议使用这些表单而不是没有此参数的原始表单。 在锁实现中作为blocker提供的正常参数是this 。

这些方法旨在用作创建更高级别同步实用程序的工具，并且对于大多数并发控制应用程序本身并不有用。 park方法仅用于以下形式的构造：

   while (!canProceed()) { // ensure request to unpark is visible to other threads ... LockSupport.park(this); } 
在调用park之前，线程没有发布请求park需要锁定或阻塞。 因为每个线程只有一个许可证，所以任何中间使用park ，包括隐式地通过类加载，都可能导致无响应的线程（“丢失unpark”）。
样品使用。 以下是先进先出非重入锁定类的草图：

   class FIFOMutex { private final AtomicBoolean locked = new AtomicBoolean(false); private final Queue<Thread> waiters = new ConcurrentLinkedQueue<>(); public void lock() { boolean wasInterrupted = false; // publish current thread for unparkers waiters.add(Thread.currentThread()); // Block while not first in queue or cannot acquire lock while (waiters.peek() != Thread.currentThread() || !locked.compareAndSet(false, true)) { LockSupport.park(this); // ignore interrupts while waiting if (Thread.interrupted()) wasInterrupted = true; } waiters.remove(); // ensure correct interrupt status on return if (wasInterrupted) Thread.currentThread().interrupt(); } public void unlock() { locked.set(false); LockSupport.unpark(waiters.peek()); } static { // Reduce the risk of "lost unpark" due to classloading Class<?> ensureLoaded = LockSupport.class; } } 
```
**LockSupport提供了更强线程的等待唤醒机制park()和unpark()的作用分别是阻塞线程和解除阻塞线程**
## 线程等待唤醒机制(wait/notify)
### 三种让线程等待和唤醒的方法
1. 使用object中的wait()方法让线程等待，使用Object中的notify()方法唤醒线程

   ```java
   /**
    * wait和notify方法必须要在同步块或者方法里面且成对出现使用
    * 先wait后 notify才可以
    */
   public class DeadLockDemo {
       static Object lock = new Object();
       public static void main(String[] args) {
           new Thread(() ->{
               synchronized (lock){
                   try {
                       System.out.println("线程被阻塞");
                       lock.wait();
   
                   } catch (InterruptedException e) {
                       e.printStackTrace();
                   }
               }
               System.out.println("被唤醒");
           },"A").start();
   
           new Thread(() ->{
               synchronized (lock){
                   System.out.println("通知");
                   lock.notify();
               }
           },"B").start();
       }
   }
   ```

   

2. 使用JUC包中Condition的await()方法让线程等待，使用signal()方法唤醒线程

   ```java
   /**
    * await和signal方法必须要在lock块中面且成对出现使用
    * 先await后 signal才OK
    */
   public class DeadLockDemo {
       static Lock lock = new ReentrantLock();
       static Condition condition = lock.newCondition();
   
       public static void main(String[] args) {
            new Thread(() -> {
                lock.lock();
                try {
                 System.out.println(Thread.currentThread().getName() + "\t" + "进入");
                 System.out.println(Thread.currentThread().getName() + "\t" + "等待");
                    condition.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    lock.unlock();
                }
              System.out.println(Thread.currentThread().getName() + "\t" + "被唤醒");
            },"A").start();
   
           new Thread(() -> {
               lock.lock();
               try {
                 System.out.println(Thread.currentThread().getName() + "\t" + "进入");
                   condition.signal();
                 System.out.println(Thread.currentThread().getName() + "\t" + "通知");
               } finally {
                   lock.unlock();
               }
           },"B").start();
       }
   
   ```
   >总结：线程先要获取并持有锁，必须在锁块（synchronized或lock）中
   >            必须要先等待后唤醒，线程才能够被唤醒

3. LockSupport类可以阻塞当前线程以及唤醒指定被阻塞的线 程

   LockSupport通过pack()和unpack()方法来实现阻塞和唤醒线程的操作

   原理：LockSupport类使用了一种名为Permit（许可）的概念来做到阻塞和唤醒现成的功能，每个线程都有一个许可（Permit），premin只有两个值1和0，默认是0，可以把许可看成是一种（0，1）信号量（Semaphore）,但与Semaphore不同的是，Permit的上线是1

   

   ```java
   /**
    * LockSupport是用来创建锁和其他同步类的基本线程阻塞原语
    * LockSupport是一个线程阻塞工具类，所有的方法都是静态方法，可以让线程在任意位置阻塞，阻塞之后也有对应的唤醒方法。归根结底，LockSupport调用的Unsafe中的native代码
    * LockSupport提供了pack()和unpack()方法实现阻塞线程和接触线程阻塞的过程
    * LockSupport和每个使用它的线程都有一个许可permit关联 permit相当于1,0的开关，默认是0
    * 调用一次unpack就加1变成1
    * 调用一次pack就回消费permit，也就是将1变成0，同时pack立即返回
    * 如再次调用pack会变成阻塞（因为permit为0了会阻塞在这里，一直permit变为1），这时调用unpack会把permit置为1
    * 每个线程都有一个相关的permit，permit最多只有一个，重复调用unpack也不会累计凭证
    * 形象的理解
    * 线程阻塞需要消耗凭证（permit）这个凭证最多只有1
    * 当调用pack方法时
    * 如果有凭证，则会直接消耗掉这个凭证然后正常退出
    * 如果没有凭证，就必须阻塞等待凭证可用
    * 而unpack则相反，它会增加一个凭证，但凭证最多只能有一个，累加无效
    *
    */
   public class DeadLockDemo {
       static Lock lock = new ReentrantLock();
       static Condition condition = lock.newCondition();
   
       public static void main(String[] args) {
           Thread a = new Thread(() -> {
               try {
                   TimeUnit.SECONDS.sleep(2l);
               } catch (InterruptedException e) {
                   e.printStackTrace();
               }
               System.out.println(Thread.currentThread().getName() + "\t" + "进入");
               System.out.println(Thread.currentThread().getName() + "\t" + "等待");
               //被阻塞。。。等待通知等待放行，它要通过需求许可证
               LockSupport.park();
               System.out.println(Thread.currentThread().getName() + "\t" + "被唤醒");
           }, "A");
           a.start();
   
   
           Thread b = new Thread(() -> {
   
               System.out.println(Thread.currentThread().getName() + "\t" + "进入");
               LockSupport.unpark(a);
               System.out.println(Thread.currentThread().getName() + "\t" + "通知");
   
           }, "B");
           b.start();
       }
   ```

   

**面试题：**

> 为什么可以先唤醒线程后阻塞线程？

因为unpack获得了一个凭证，之后调用pack方法，就可以名正言顺的凭证消费，故不会阻塞

> 为什么唤醒两次后阻塞两次，但最终结果还会阻塞线程？

因为凭证的数量最多为1，连续调用两次unpack和调用一次unpack效果一样，只会增加一个凭证，而调用两次pack却需要消费两个凭证，凭证(permit)不够，不能放行
