# Android 崩溃监控实践

在 Android 开发过程中经常碰到 app 崩溃，对于开发阶段出现的崩溃，开发者可以从后台日志中获取崩溃堆栈进行分析，而线上出现的崩溃，开发者看不到后台日志，无法获取崩溃堆栈，这就需要一款可以监控线上应用崩溃情况的工具，当应用出现崩溃时及时收集堆栈信息进行分析，然后上报给服务端，开发者就可以在控制台实时了解应用的崩溃情况。XXX 是 XXXXXX 推出的一款线上应用崩溃监控工具，实现了 Java、Native、ANR 崩溃的监控功能，目前已赋能近 XXX 个应用，本文就来介绍一下 XXX 在 Java 和 Native 层崩溃监控实践。

## 捕获 Java Crash

捕获 Java 层的崩溃相对比较简单，系统为我们提供了专门的类来处理：

```java
 /**
     * Interface for handlers invoked when a <tt>Thread</tt> abruptly
     * terminates due to an uncaught exception.
     *
     * <p>When a thread is about to terminate due to an uncaught exception
     * the Java Virtual Machine will query the thread for its
     * <tt>UncaughtExceptionHandler</tt> using
     * {@link #getUncaughtExceptionHandler} and will invoke the handler's
     * <tt>uncaughtException</tt> method, passing the thread and the
     * exception as arguments.
     *
     * If a thread has not had its <tt>UncaughtExceptionHandler</tt>
     * explicitly set, then its <tt>ThreadGroup</tt> object acts as its
     * <tt>UncaughtExceptionHandler</tt>. If the <tt>ThreadGroup</tt> object
     * has no
     * special requirements for dealing with the exception, it can forward
     * the invocation to the {@linkplain #getDefaultUncaughtExceptionHandler
     * default uncaught exception handler}.
     *
     * @since 1.5
     */
    @FunctionalInterface
    public interface UncaughtExceptionHandler {
        /**
         * Method invoked when the given thread terminates due to the
         * given uncaught exception.
         * <p>Any exception thrown by this method will be ignored by the
         * Java Virtual Machine.
         *
         * @param t the thread
         * @param e the exception
         */
        void uncaughtException(Thread t, Throwable e);
    }
```
UncaughtExceptionHandler 未捕获异常处理接口，当一个线程由于一个未捕获异常即将崩溃时，JVM 将会通过 getUncaughtExceptionHandler() 方法获取该线程的 UncaughtExceptionHandler，并将该线程和异常作为参数传给 uncaughtException()方法。

如果没有显式设置线程的 UncaughtExceptionHandler，那么会将其 ThreadGroup 对象会作为 UncaughtExceptionHandler。如果其 ThreadGroup 对象没有特殊的处理异常的需求，那么就会调 getDefaultUncaughtExceptionHandler() 方法获取默认的 UncaughtExceptionHandler 来处理异常。

我们都知道应用程序通常都会创建很多线程，如果为每一个线程都设置一次 UncaughtExceptionHandler 未免太过麻烦，既然出现未处理异常后 JVM 最终都会调 getDefaultUncaughtExceptionHandler()，那么我们可以在应用启动时设置一个默认的未捕获异常处理器：

```java
public class MyApp extends Application {

    @Override
    public void onCreate() {
        super.onCreate();
        MyCrashHandler handler = new MyCrashHandler();
        Thread.setDefaultUncaughtExceptionHandler(handler);
    }
}
```

Thread.setDefaultUncaughtExceptionHandler(handler)方法，如果多次调用的话，会以最后一次传递的 handler 为准，所以如果用了第三方的统计模块，可能会出现失灵的情况。对于这种情况，在设置默认 hander 之前，可以先通过 getDefaultUncaughtExceptionHandler() 方法获取并保留旧的 hander，然后在默认 handler 的uncaughtException 方法中调用其他 handler 的 uncaughtException 方法，保证都会收到异常信息。

## 捕获 Native Crash

在 Android 平台上，Native 崩溃监控相对 Java 崩溃比较麻烦一些，先来了解一下 Linux 系统的信号机制。
