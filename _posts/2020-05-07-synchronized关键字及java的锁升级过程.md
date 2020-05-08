[toc]
# synchronized关键字及java的锁升级过程

## synchronized 作用以及区别

为了保证多线程应用程序的并发安全，java提供了synchronized关键字保障多线程竞争执行同一个代码块的线程安全性；

synchronized关键字可以加在方法上、也可以作用于对象实例、甚至可以作用在类上面；

### 修饰方法等价修饰对象
对象锁，针对某个类的具体实例对象加锁，不同实例间不存在竞争；
### 修饰静态方法等价修饰类上
所有类的实例对象共用一把锁，类对应new出来的所有实例执行到同步代码块的时候都需要等待这把锁的释放（如果有实例已经获取到锁）

### 测试案例

```java
public class SyncDemo1 {

  //example 1 sync(obj)
  public void test1(){
    synchronized (this) {//this替换成一个实例成员变量 obj 也是一样的效果
      int i = 0;
      for (; i < 5; i++) {
        System.out.println(
            String.format("threadid:[%d],count_val:[%d]", Thread.currentThread().getId(), i));
      }
    }
  }
  //example 2 sync(class)
  public void test2(){
    synchronized (SyncDemo1.class) {
      int i = 0;
      for (; i < 5; i++) {
        System.out.println(
            String.format("threadid:[%d],count_val:[%d]", Thread.currentThread().getId(), i));
      }
    }
  }
  public static void main(String[] args) throws InterruptedException {
    //test 1
    multiObjSyncObj();
    //test 2
    singleObjSyncObj();
    //test 3
    multiObjSyncClass();
  }
  /**
   * //    threadid:[12],count_val:[0]
   * //    threadid:[13],count_val:[0]
   * //    threadid:[12],count_val:[1]
   * //    threadid:[13],count_val:[1]
   * //    threadid:[12],count_val:[2]
   * //    threadid:[13],count_val:[2]
   * //    threadid:[12],count_val:[3]
   * //    threadid:[12],count_val:[4]
   * //    threadid:[13],count_val:[3]
   * //    threadid:[13],count_val:[4]
   * @throws InterruptedException
   */
  private static void multiObjSyncObj() throws InterruptedException {
    System.out.println("#############################multiObjSyncObj##################################");
    SyncDemo1 syncDemo1 = new SyncDemo1();
    SyncDemo1 syncDemo11 = new SyncDemo1();

    ExecutorService executorService = Executors.newFixedThreadPool(2);
    executorService.execute(()->syncDemo1.test1());
    executorService.execute(()->syncDemo11.test1());
    Thread.sleep(5000);
    executorService.shutdownNow();
  }

  /**
   * threadid:[14],count_val:[0]
   * threadid:[14],count_val:[1]
   * threadid:[14],count_val:[2]
   * threadid:[14],count_val:[3]
   * threadid:[14],count_val:[4]
   * threadid:[15],count_val:[0]
   * threadid:[15],count_val:[1]
   * threadid:[15],count_val:[2]
   * threadid:[15],count_val:[3]
   * threadid:[15],count_val:[4]
   * @throws InterruptedException
   */
  private static void singleObjSyncObj() throws InterruptedException {
    System.out.println("#############################singleObjSyncObj##################################");

    SyncDemo1 syncDemo1 = new SyncDemo1();

    ExecutorService executorService = Executors.newFixedThreadPool(2);
    executorService.execute(()->syncDemo1.test1());
    executorService.execute(()->syncDemo1.test1());
    Thread.sleep(5000);
    executorService.shutdownNow();
  }

  /**
   * threadid:[16],count_val:[0]
   * threadid:[16],count_val:[1]
   * threadid:[16],count_val:[2]
   * threadid:[16],count_val:[3]
   * threadid:[16],count_val:[4]
   * threadid:[17],count_val:[0]
   * threadid:[17],count_val:[1]
   * threadid:[17],count_val:[2]
   * threadid:[17],count_val:[3]
   * threadid:[17],count_val:[4]
   * @throws InterruptedException
   */
  private static void multiObjSyncClass() throws InterruptedException {
    System.out.println("#############################multiObjSyncClass##################################");
    SyncDemo1 syncDemo1 = new SyncDemo1();
    SyncDemo1 syncDemo11 = new SyncDemo1();

    ExecutorService executorService = Executors.newFixedThreadPool(2);
    executorService.execute(()->syncDemo1.test2());
    executorService.execute(()->syncDemo11.test2());
    Thread.sleep(5000);
    executorService.shutdownNow();
  }
}
```

## synchronized 加锁的原理以及锁升级过程

### 加锁原理及对象头信息

#### synchronized 与 monitorenter、monitorexit

通过javap 查看SyncDemo1的字节码指令可以看到，在test1和test2方法中都有对应 monitorenter 和 monitorexit 的指令，其实，我们在代码中添加 synchronized 之后，javac编译之后就会将同步关键字包裹的代码 块在进入和退出（或者异常发生时）加上 monitorenter 和 monitorexit 指令；

#### 对象头信息

![64位对象头信息]({{ site.url }}/img/obj_head.png)

简单来说就是对象的mark word前几位记录了获取锁的线程id信息，后面几位记录的是锁的状态（无锁、偏向锁、轻量级锁、重量级锁）

这里以64位虚拟机来解释mark work（64位）：

- unused（25）：未使用的bit
- identity_hashcode（31）：对象的哈希code
- unused（1）：未使用
- epoch （2）：对象每次获取偏向锁时，会从类的头信息中获取该值并加+1，当该类的值超过4次（2bit）时，那么该类将不再适合偏向锁；
- age（4）：标识对象的回收次数，当超过参数-XX:MaxTenuringThreshold 指定值时将对象移到老年代（永久代），G1收集器默认是15
- biased_lock（1）：偏向锁标识，jdk1.6之后可以通过-XX: -UseBiasedLocking参数开启偏向锁，需要知道的是，jvm启动过程中偏向锁加赞是延迟加载的，可以通过-XX:BiasedLockingStartupDelay=0设置偏向锁不延迟加载；
- lock（2）：锁状态：
  - 00-表示轻量级锁；
  - 01-这里01既表示无锁也表示偏向锁，是否偏向锁根据偏向标识判断；
  - 10-表示重量级锁；
  - 11-标记需要被垃圾回收；
- ptr_to_lock_record（62）：当获取锁时没有竞争，那么jvm使用原子操作去获取锁，而不是操作系统的mutex，此时JVM通过CAS操作设置指向对象头字中的锁记录的指针；
- ptr_to_heavyweight_monitor （62） ：当两个以上的线程操作同步块发生竞争时，轻量级锁就必须升级为重量级锁监视器来管理等待线程，在重量级锁定的情况下，JVM在对象的头字中设置一个指向监视器的指针。

### 锁升级过程

![锁升级过程]({{ site.url }}/img/Synchronization.gif)

#### 无锁（001）

对象初始化完成，未进入同步代码块的过程中，对象的锁状态；

#### 偏向锁（101）

线程执行到同步代码块中，但是此时没有发生竞争，此时mark word中会存储执行线程的线程ID。

获取偏向锁的过程大致就是JVM通过CAS操作更新对象头中所指向的线程id记录的过程。

偏向锁在hotspot源码中的实现：

hotspot/src/share/vm/runtime/synchronizer.cpp/ObjectSynchronizer::fast_enter (偏向锁到轻量级锁升级的过程)

实际设置对象锁偏向信息的逻辑：

hotspot/src/share/vm/runtime/biasedLocking.cpp/BiasedLocking::revoke_and_rebias(这里通过CAS更新对象头信息)

#### 轻量级锁（00）

这里指的是当一个线程获取到对象锁的时候，此时其他线程尝试获取该锁资源，JVM为了优化性能直接通过自适应自旋的形式（老版本有次数限制默认15次），获取锁资源；而不是直接采用操作系统重量级锁实现同步；

这里操作的本质就是JVM会通过CAS操作去更新对象头中的 ptr_to_lock_record 和最后 两位锁状态位；

代码在：

hotspot/src/share/vm/runtime/synchronizer.cpp/ObjectSynchronizer::slow_enter

#### 重量级锁（10）

升级到重量级锁后，对象头信息会变更，同时会将等待的线程加入到一个ObjectMonitor的链表中，等待资源释放后被唤醒，然后尝试获取锁；











