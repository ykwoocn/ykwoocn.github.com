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



### 锁升级过程



