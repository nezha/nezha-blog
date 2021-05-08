# 并发编程-volatile和synchronized


## 并发三大特性
- **原子性**：原子性就是说一个操作不能被打断，要么执行完要么不执行。
- **可见性**：可见性是指一个变量的修改对所有线程可见。即当一条线程修改了这个变量的值，新值对于其他线程来说是可以立即得知的。
- **有序性**：为了提高程序的执行性能，编辑器和处理器都有可能会对程序中的指令进行重排序。

## 关于 `Synchronized`

> `Synchronized` 拥有所有三大属性，但是其性能开销也是做大的
> 如果是对原子性要求高的可以使用`Synchronized`

**先看看 `Volatile` 能否在并发场景下直接使用**

### 程序
```java
public class VolatileTest {
    public volatile int race = 0;
    public void increase() {
        race++;
    }
    public  int getRace(){
        return race;
    }

    public static void main(String[] args) {
        //创建5个线程，同时对同一个volatileTest实例对象执行累加操作
        VolatileTest volatileTest=new VolatileTest();
        int threadCount = 5;
        Thread[] threads = new Thread[threadCount];//5个线程
        for (int i = 0; i < threadCount; i++) {
            //每个线程都执行10000次++操作
            threads[i]  = new Thread(()->{
                for (int j = 0; j < 10000; j++) {
                    volatileTest.increase();
                }
                System.out.println(Thread.currentThread().getName()+"执行10000次++后,race值为："+volatileTest.getRace());
            },"线程"+(i+1));
            threads[i].start();
        }

        //等待所有累加线程都结束
        for (int i = 0; i < threadCount; i++) {
            try {
                threads[i].join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        //所有子线程结束后，race理论值应该是：5*10000=50000,但是执行结果会小于它,这是为啥呢？
        System.out.println("累加结果："+volatileTest.getRace());
    }
}
```

### 结果1

```
线程2执行10000次++后,race值为：19852
线程1执行10000次++后,race值为：19846
线程3执行10000次++后,race值为：23269
线程4执行10000次++后,race值为：26247
线程5执行10000次++后,race值为：36247
累加结果：36247
```

其实`race++`的过程并不是原子操作，其可以分为以下几个步骤：

1. 从内存中获取`race`的值，放在线程缓存中
2. 对`race`进行加`1`计算
3. 让计算后的`race`值写回到内存中

可以看到结果并不是50000，这是因为的5个线程并发执行累加过程中，上面的`1/2/3`步骤可能发生交叉执行导致`A`线程写回的`race`值覆盖了其他线程写入的值.

其中的 `Volatile` 修饰的 `race` 并不能起到原子性的功能。


### 程序2
```java
public class SynchronizedTest {
    public int race = 0;
    //使用synchronized保证++操作原子性
    public synchronized void increase() {
        race++;
    }
    public int getRace(){
        return race;
    }

    public static void main(String[] args) {
        //创建5个线程，同时对同一个volatileTest实例对象执行累加操作
        SynchronizedTest synchronizedTest=new SynchronizedTest();
        int threadCount = 5;
        Thread[] threads = new Thread[threadCount];//5个线程
        for (int i = 0; i < threadCount; i++) {
            //每个线程都执行1000次++操作
            threads[i]  = new Thread(()->{
                for (int j = 0; j < 10000; j++) {
                    synchronizedTest.increase();
                }
                System.out.println(synchronizedTest.getRace());
            });
            threads[i].start();
        }

        //等待所有累加线程都结束
        for (int i = 0; i < threadCount; i++) {
            try {
                threads[i].join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        //所有子线程结束后，race是：5*10000=50000。
        System.out.println("累加结果："+synchronizedTest.getRace());
    }
}
```

### 结果2

```
36930
45366
47132
49210
50000
累加结果：50000
```

在`increase()`方法上增加 `synchronized` 保证累加的原子性，这样结果就没问题了。

## 关于 `Volatile`

> `Volatile` 有可见性和有序性

`volatile` 最为主要的功能是能每次获取变量的内存值，而不是线程的缓存值，这点对于在多线程模式下，同步各个线程中变量的属性就非常有用了。

我们平时定义的变量，在线程中数据往往都是读取的线程缓存，并不能进行线程间变量值同步。

### 程序1
```java
//未使用了volatile
public class NonVolatileDemo {
    public static boolean stop = false;//任务是否停止,普通变量

    public static void main(String[] args) throws Exception {
         Thread thread1 = new Thread(() -> {
            while (!stop) { //stop=false，不满足停止条件，继续执行
                //do someting
            }
            System.out.println("stop=true，满足停止条件。" +
                    "停止时间：" + System.currentTimeMillis());
        });
        thread1.start();

        Thread.sleep(100);//保证主线程修改stop=true，在子线程启动后执行。
        stop = true; //true
        System.out.println("主线程设置停止标识 stop=true。" +
                "设置时间：" + System.currentTimeMillis());
    }
}
```

### 结果1

```
主线程设置停止标识 stop=true。设置时间：1620443309699

```

可以看到主线程修改的值，并没有在线程中生效。
当然这也可能是特殊场景，之后我尝试在 `while` 内部执行一些逻辑，那样的话 `stop` 的值又是可以获取的。
此处只是借用特殊场景介绍 `volatile` 的使用。

### 程序2

```java
//使用了volatile
public class VolatileDemo {
    public static volatile boolean stop = false;//任务是否停止，volatile变量
    public static void main(String[] args) throws Exception {
        Thread thread1 = new Thread(() -> {
            while (!stop) { //stop=false，不满足停止条件，继续执行
                //do someting
            }
            System.out.println("stop=true，满足停止条件。" +
                    "停止时间：" + System.currentTimeMillis());
        });
        thread1.start();

        Thread.sleep(100);//保证主线程修改stop=true，在子线程启动后执行。
        stop = true; //true
        System.out.println("主线程设置停止标识 stop=true。" +
                "设置时间：" + System.currentTimeMillis());
    }
}
```

### 结果2

```
主线程设置停止标识 stop=true。设置时间：1620447063306
stop=true，满足停止条件。停止时间：1620447063306
```

## 参考文献

- <https://zhuanlan.zhihu.com/p/112742540>
- <https://zhuanlan.zhihu.com/p/111559032>
- <https://zhuanlan.zhihu.com/p/111557310>