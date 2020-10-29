# FutureTask

```java
/*
 * ORACLE PROPRIETARY/CONFIDENTIAL. Use is subject to license terms.
 *
 *
 *
 *
 *
 *
 *
 *
 *
 *
 *
 *
 *
 *
 *
 *
 *
 *
 *
 *
 */

/*
 *
 *
 *
 *
 *
 * Written by Doug Lea with assistance from members of JCP JSR-166
 * Expert Group and released to the public domain, as explained at
 * http://creativecommons.org/publicdomain/zero/1.0/
 */

package java.util.concurrent;
import java.util.concurrent.locks.LockSupport;

/**
 * A cancellable asynchronous computation.  This class provides a base
 * implementation of {@link Future}, with methods to start and cancel
 * a computation, query to see if the computation is complete, and
 * retrieve the result of the computation.  The result can only be
 * retrieved when the computation has completed; the {@code get}
 * methods will block if the computation has not yet completed.  Once
 * the computation has completed, the computation cannot be restarted
 * or cancelled (unless the computation is invoked using
 * {@link #runAndReset}).
 *
 * <p>A {@code FutureTask} can be used to wrap a {@link Callable} or
 * {@link Runnable} object.  Because {@code FutureTask} implements
 * {@code Runnable}, a {@code FutureTask} can be submitted to an
 * {@link Executor} for execution.
 *
 * <p>In addition to serving as a standalone class, this class provides
 * {@code protected} functionality that may be useful when creating
 * customized task classes.
 *
 * @since 1.5
 * @author Doug Lea
 * @param <V> The result type returned by this FutureTask's {@code get} methods
 */
public class FutureTask<V> implements RunnableFuture<V> {
    /*
     * Revision notes: This differs from previous versions of this
     * class that relied on AbstractQueuedSynchronizer, mainly to
     * avoid surprising users about retaining interrupt status during
     * cancellation races. Sync control in the current design relies
     * on a "state" field updated via CAS to track completion, along
     * with a simple Treiber stack to hold waiting threads.
     *
     * Style note: As usual, we bypass overhead of using
     * AtomicXFieldUpdaters and instead directly use Unsafe intrinsics.
     */

    /**
     * The run state of this task, initially NEW.  The run state
     * transitions to a terminal state only in methods set,
     * setException, and cancel.  During completion, state may take on
     * transient values of COMPLETING (while outcome is being set) or
     * INTERRUPTING (only while interrupting the runner to satisfy a
     * cancel(true)). Transitions from these intermediate to final
     * states use cheaper ordered/lazy writes because values are unique
     * and cannot be further modified.
     *
     * Possible state transitions:
     * NEW -> COMPLETING -> NORMAL
     * NEW -> COMPLETING -> EXCEPTIONAL
     * NEW -> CANCELLED
     * NEW -> INTERRUPTING -> INTERRUPTED
     */

    //表示当前任务的状态
    private volatile int state;
    //当前任务尚未执行
    private static final int NEW          = 0;
    //当前任务正在结束，尚未完全结束，一种临界状态
    private static final int COMPLETING   = 1;
    //当前任务正常结束
    private static final int NORMAL       = 2;
    //当前任务执行过程中发生了异常，内部封装callable.run()call()向上抛出了异常了
    private static final int EXCEPTIONAL  = 3;
    //当前任务被取消
    private static final int CANCELLED    = 4;
    //当前任务中断中...
    private static final int INTERRUPTING = 5;
    //当前任务已中断
    private static final int INTERRUPTED  = 6;

    /** The underlying callable; nulled out after running */
    //submit(runnable/callable) runnable使用装饰着模式伪装成Callable接口
    private Callable<V> callable;
    /** The result to return or exception to throw from get() */
    //任务正常执行结束 outcome 保存执行结果 callable的返回值
    //     非正常情况 callable向上抛出异常 outcome保存异常
    private Object outcome; // non-volatile, protected by state reads/writes
    /** The thread running the callable; CASed during run() */
    //当前任务被线程执行期间，保存党风前执行任务的线程对象引用
    private volatile Thread runner;
    /** Treiber stack of waiting threads */
    //因为会有很多的线程去get当前任务的结果，所以这里使用了一种数据结构 stack 头插头取的队列
    private volatile WaitNode waiters;

    /**
     * Returns result or throws exception for completed task.
     *
     * @param s completed state value
     */
    @SuppressWarnings("unchecked")
    private V report(int s) throws ExecutionException {
        //正常情况下 outcome 保存的是callable运行结果
        //非正常情况下 保存的是callable抛出的异常
        Object x = outcome;
        //条件成立 当前任务状态正常结束
        if (s == NORMAL)
            //返回callable运算结果
            return (V)x;
        //被cancel 中断了
        if (s >= CANCELLED)
            throw new CancellationException();
        //callable接口有bug
        throw new ExecutionException((Throwable)x);
    }

    /**
     * Creates a {@code FutureTask} that will, upon running, execute the
     * given {@code Callable}.
     *
     * @param  callable the callable task
     * @throws NullPointerException if the callable is null
     */
    public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        //callable就是程序员自己实现的业务类
        this.callable = callable;
        //设置当前任务状态为NEW  初始状态 尚未执行
        this.state = NEW;       // ensure visibility of callable
    }

    /**
     * Creates a {@code FutureTask} that will, upon running, execute the
     * given {@code Runnable}, and arrange that {@code get} will return the
     * given result on successful completion.
     *
     * @param runnable the runnable task
     * @param result the result to return on successful completion. If
     * you don't need a particular result, consider using
     * constructions of the form:
     * {@code Future<?> f = new FutureTask<Void>(runnable, null)}
     * @throws NullPointerException if the runnable is null
     */
    public FutureTask(Runnable runnable, V result) {
        //通过装饰着模式把runnable和V封装成一个callable接口
        //当前任务执行结束时，结果可能传入什么返回什么 不是真正的计算结果
        this.callable = Executors.callable(runnable, result);
        //设置当前任务状态为NEW 初始状态 尚未执行
        this.state = NEW;       // ensure visibility of callable
    }

    public boolean isCancelled() {
        return state >= CANCELLED;
    }

    public boolean isDone() {
        return state != NEW;
    }

    public boolean cancel(boolean mayInterruptIfRunning) {
        // 条件一：state == NEW 成立 表是当前任务处于运行中 或者 处于线程池 任务队列中
        // 条件二： UNSAFE.compareAndSwapInt(this, stateOffset, NEW,mayInterruptIfRunning ? INTERRUPTING : CANCELLED)
        //条件成立 说明修改状态成功，可以执行下面的逻辑 否则 返回false 表示cancel失败
        if (!(state == NEW &&
              UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
                  mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
            return false;
        try {    // in case call to interrupt throws exception
            if (mayInterruptIfRunning) {
                try {
                    //执行当前futuretask的线程 有可能现在是null（当前任务在队列中还没有线程获取到它）
                    Thread t = runner;
                    // 条件成立 说明当前线程 runner 正在执行task
                    if (t != null)
                        //给runner线程一个中断信号 如果你的程序是响应中断的 会走中断逻辑，假如不是响应中断的 啥也不会发生
                        t.interrupt();
                } finally { // final state
                    //设置任务状态为终端完成
                    UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
                }
            }
        } finally {
            //唤醒所有get（）阻塞的程序
            finishCompletion();
        }
        return true;
    }

    /**
     * @throws CancellationException {@inheritDoc}
     */
    //场景：多个线程等该当前任务执行完成后的结果
    public V get() throws InterruptedException, ExecutionException {
        //获取当前任务状态
        int s = state;
        //条件成立：未执行、正在执行、正在完成，就是未完成的情况下 调用get()外部线程会被阻塞在get方法上
        if (s <= COMPLETING)
            //拿到当前线程的状态 如果异常就抛出去 给调用的线程
            s = awaitDone(false, 0L);
        return report(s);
    }

    /**
     * @throws CancellationException {@inheritDoc}
     */
    public V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException {
        if (unit == null)
            throw new NullPointerException();
        int s = state;
        if (s <= COMPLETING &&
            (s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)
            throw new TimeoutException();
        return report(s);
    }

    /**
     * Protected method invoked when this task transitions to state
     * {@code isDone} (whether normally or via cancellation). The
     * default implementation does nothing.  Subclasses may override
     * this method to invoke completion callbacks or perform
     * bookkeeping. Note that you can query status inside the
     * implementation of this method to determine whether this task
     * has been cancelled.
     */
    protected void done() { }

    /**
     * Sets the result of this future to the given value unless
     * this future has already been set or has been cancelled.
     *
     * <p>This method is invoked internally by the {@link #run} method
     * upon successful completion of the computation.
     *
     * @param v the value
     */
    protected void set(V v) {
        // 使用CAS方式设置当前任务状态为 完成中
        // 有没有可能失败呢？外部线程等不及了，直接在set执行CAS之前将task cancel掉，小概率事件
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            //
            outcome = v;
            //将结果赋值给outcome之后，马上会将当前任务状态修改为 NORMAL 正常结束状态
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
            //等会说
            finishCompletion();
        }
    }

    /**
     * Causes this future to report an {@link ExecutionException}
     * with the given throwable as its cause, unless this future has
     * already been set or has been cancelled.
     *
     * <p>This method is invoked internally by the {@link #run} method
     * upon failure of the computation.
     *
     * @param t the cause of failure
     */
    protected void setException(Throwable t) {
        // 使用CAS方式设置当前任务状态为 完成中
        // 有没有可能失败呢？外部线程等不及了，直接在set执行CAS之前将task cancel掉，小概率事件
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            //引用得是callable 向上层抛出来的异常
            outcome = t;
            //将当前任务状态 修改为EXCEPTIONAL
            UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
            //回头再说 get()之后
            finishCompletion();
        }
    }
//任务执行入口
    public void run() {
        //state != NEW 条件成立，说明当前task已经被执行过了或者被cancel 总之非NEW状态的任务 现成就不处理了
        //!UNSAFE.compareAndSwapObject(this(当前对象), runnerOffset(偏移量),null(预期值), Thread.currentThread()(设置的值))
        //条件成立：cas失败，说明当前任务已经被其他线程抢占了
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        //执行到这里，当前task一定是NEW状态，而且 当前线程也抢占task成功
        try {
            // callable 就是程序员自己封装逻辑的callable 或者装饰后的runnable
            Callable<V> c = callable;
            //条件一：c != null 防止空指针异常 构建的时候传null
            //条件二：state == NEW 防止外部线程cancel掉任务
            if (c != null && state == NEW) {
                //call方法执行的结果引用
                V result;
                //true 表示callable.call代码块执行成功 未抛出异常
                //false 表示callable.call代码执行失败 抛出异常
                boolean ran;
                try {
                    //调用程序员自己实现的callable或者被装饰得runnable
                    result = c.call();
                    //c.call() 未抛出异常 ran= true 代码块执行成功
                    ran = true;
                } catch (Throwable ex) {
                    //说明程序员自己实现的callable或者被装饰得runnable有bug
                    result = null;
                    //c.call() 抛出异常 ran= false 代码块执行失败
                    ran = false;
                    //设置异常返回，修改task为异常状态
                    setException(ex);
                }
                if (ran)
                    //说明当前c.call正常执行了 然后设置结果到outcome
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                //回头再说。。。讲了cancel（）就明白了
                handlePossibleCancellationInterrupt(s);
        }
    }

    /**
     * Executes the computation without setting its result, and then
     * resets this future to initial state, failing to do so if the
     * computation encounters an exception or is cancelled.  This is
     * designed for use with tasks that intrinsically execute more
     * than once.
     *
     * @return {@code true} if successfully run and reset
     */
    protected boolean runAndReset() {
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return false;
        boolean ran = false;
        int s = state;
        try {
            Callable<V> c = callable;
            if (c != null && s == NEW) {
                try {
                    c.call(); // don't set result
                    ran = true;
                } catch (Throwable ex) {
                    setException(ex);
                }
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
        return ran && s == NEW;
    }

    /**
     * Ensures that any interrupt from a possible cancel(true) is only
     * delivered to a task while in run or runAndReset.
     */
    private void handlePossibleCancellationInterrupt(int s) {
        // It is possible for our interrupter to stall before getting a
        // chance to interrupt us.  Let's spin-wait patiently.
        if (s == INTERRUPTING)
            while (state == INTERRUPTING)
                Thread.yield(); // wait out pending interrupt

        // assert state == INTERRUPTED;

        // We want to clear any interrupt we may have received from
        // cancel(true).  However, it is permissible to use interrupts
        // as an independent mechanism for a task to communicate with
        // its caller, and there is no way to clear only the
        // cancellation interrupt.
        //
        // Thread.interrupted();
    }

    /**
     * Simple linked list nodes to record waiting threads in a Treiber
     * stack.  See other classes such as Phaser and SynchronousQueue
     * for more detailed explanation.
     */
    static final class WaitNode {
        volatile Thread thread;
        volatile WaitNode next;
        WaitNode() { thread = Thread.currentThread(); }
    }

    /**
     * Removes and signals all waiting threads, invokes done(), and
     * nulls out callable.
     */
    private void finishCompletion() {
        // assert state > COMPLETING;
        //q指向waiters链表头结点
        for (WaitNode q; (q = waiters) != null;) {
            // 使用CAS设置waiters为null 是因为怕外部线程使用cancel取消当前线程 也会触发finishCompletion方法 小概率事件
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
                for (;;) {
                    //获取当前node节点封装的thread
                    Thread t = q.thread;
                    //条件成立 说明当前线程不为null
                    if (t != null) {
                        q.thread = null; //help GC
                        // 唤醒当前节点对应的线程
                        LockSupport.unpark(t);
                    }
                    // next 当前节点的 下一个节点
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }

        done();
        // 将callable置为null help GC
        callable = null;        // to reduce footprint
    }

    /**
     * Awaits completion or aborts on interrupt or timeout.
     *
     * @param timed true if use timed waits
     * @param nanos time to wait, if timed
     * @return state upon completion
     */
    private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
        //0 不带超时
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        //引用当前线程 封装 WaitNode 对象
        WaitNode q = null;
        // 表示当前线程 WaitNode 对象有没有入队/压栈
        boolean queued = false;
        //自旋
        for (;;) {
            //条件成立说明当前线程唤醒是被其他线程使用中断这种方式唤醒的
            //interrupted()返回true后会将thread的中断标记重置为false
            if (Thread.interrupted()) {
                //当前线程node出队
                removeWaiter(q);
                //向get方法抛出异常 终端一场
                throw new InterruptedException();
            }
            //假设当前线程是被其他线程 使用unpack(thread)唤醒的话 会正常自旋 走下面逻辑

            //获取当前任务最新状态
            int s = state;
            //条件成立说明当前任务 已经有结果了 可能是好 可能是坏
            if (s > COMPLETING) {
                //条件成立说明已经为当前线程创建了node，此时需要将node.thread = null helpGC
                if (q != null)
                    q.thread = null;
                //直接返回当前状态
                return s;
            }
            // 条件成立说明当前任务接近完成状态 这里让当前线程再释放cpu 进行下一次的抢占
            else if (s == COMPLETING) // cannot time out yet
                Thread.yield();

            //条件成立：第一次自旋，当前线程还未创建 WaitNode 对象，此时为当前线程创建 WaitNode 对象
            else if (q == null)
                q = new WaitNode();
            //条件成立：第二次自旋，当前线程已经创建 WaitNode对象了，但是node对象还未入队
            else if (!queued)
            // 首先当前线程 node节点 next节点指向队列的头节点   说明：waiters一直指向队列的头
            // 在利用CAS设置waiters指向当前线程 成功的话 queued 等于true 否则 可能其他线程先你一步入队了
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                     q.next = waiters, q);
            else if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {
                    removeWaiter(q);
                    return state;
                }
                LockSupport.parkNanos(this, nanos);
            }
            else
                //当前get操作的线程就会被park了 线程状态会变为 WAITING状态 相当于休眠了
                // 除非有其他线程将你唤醒 或者 将当前线程中断
                LockSupport.park(this);
        }
    }

    /**
     * Tries to unlink a timed-out or interrupted wait node to avoid
     * accumulating garbage.  Internal nodes are simply unspliced
     * without CAS since it is harmless if they are traversed anyway
     * by releasers.  To avoid effects of unsplicing from already
     * removed nodes, the list is retraversed in case of an apparent
     * race.  This is slow when there are a lot of nodes, but we don't
     * expect lists to be long enough to outweigh higher-overhead
     * schemes.
     */
    private void removeWaiter(WaitNode node) {
        if (node != null) {
            node.thread = null;
            retry:
            for (;;) {          // restart on removeWaiter race
                for (WaitNode pred = null, q = waiters, s; q != null; q = s) {
                    s = q.next;
                    if (q.thread != null)
                        pred = q;
                    else if (pred != null) {
                        pred.next = s;
                        if (pred.thread == null) // check for race
                            continue retry;
                    }
                    else if (!UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                          q, s))
                        continue retry;
                }
                break;
            }
        }
    }

    // Unsafe mechanics
    private static final sun.misc.Unsafe UNSAFE;
    private static final long stateOffset;
    private static final long runnerOffset;
    private static final long waitersOffset;
    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> k = FutureTask.class;
            stateOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("state"));
            runnerOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("runner"));
            waitersOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("waiters"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }

}

```

