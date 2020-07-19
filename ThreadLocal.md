常见的面试题：

> 1、ThreadLocal是什么？
>
> 2、ThreadLocal工作原理？
>
> 3、ThreadLocal如何解决hash冲突？
>
> 4、ThreadLocal内存泄漏是怎么回事？
>
> 5、为什么ThreadLocal的key的设计是弱引用？如果设计成强引用可以吗？
>
> 6、ThreadLocal的应用场景？

---

### ThreadLocal

#### 1、初识ThreadLocal：a

ThreadLocal是一个线程本地变量。官方的描述是：ThreadLocal类用来提供线程内部的局部变量。这种变量在多线程环境下访问时能保证各个线程的变量相对独立于其他线程内的变量。

ThreadLocal的作用是：**提供线程内的局部变量，不同的线程之间互不干扰，这种变量在线程生命周期内起作用，减少同一个线程内多个函数或组件之间传递公共参数的复杂度**。

总结：

> - 线程并发：多线程并发
> - 传递数据：在同一线程，不同组件内传递公共参数
> - 线程隔离：数据相互独立，互不干扰

---

#### 2、基本使用

##### 2.1 常用方法

| 方法申明        | 描述                          |
| --------------- | ----------------------------- |
| ThreadLocal()   | 空参构造，创建ThreadLocal实例 |
| void set(T val) | 设置跟当前线程绑定的变量      |
| T get()         | 获取当前线程绑定的变量        |
| void remove()   | 移除当前线程绑定的变量        |

##### 2.2 使用案例：

1、设置值&取值

```java
public class ThreadLocalDemo {
    private String content;
    private ThreadLocal<String> tl = new ThreadLocal<>();
    public String getContent() {
        // return content;
        return tl.get();
    }
    public void setContent(String content) {
        // this.content = content;
        tl.set(content);
    }
    public static void main(String[] args) {
        ThreadLocalDemo demo = new ThreadLocalDemo();
        for(int i=0; i<5; i++) {
            new Thread(() -> {
                demo.setContent(Thread.currentThread().getName() + "的数据");
                System.out.println("==================");
                System.out.println(Thread.currentThread().getName() + "---->" + demo.getContent());
            }, "线程" + i).start();
        }
    }
}
```

> 启动五个线程，分别往content中设置值并取值，在多线程环境下会造成取值混乱，而使用ThreadLocal经过改造之后就不会出现这样的问题。
>
> 当然上面的问题也可以使用synchronized关键字来处理，将设置值和取值的步骤放在代码块中也能起到这样的作用。

ThreadLocal和synchronized的对比：

ThreadLocal可以让程序拥有更高的并发性。

2、转账案例：全程无代码，靠想象😁

去TM的转账案例

> 1、转账无非强调转入转出是一个原子操作，因此需要用到数据库的事务，事务在代码里的操作有两步：
>
> - 关闭事务的自动提交：connection.setAutoCommit(false);
>
> - 转账完成：提交事务或回滚事务：
>
>   ```java
>   connection.commit(); // 转账成功
>   connection.rollback(); // 转账失败
>   ```
>
> 2、实际业务中：dao层执行数据库请求，service层执行业务逻辑。而这里的关键是，执行数据库操作的连接和事务处理的连接必须是同一个。有两种方式实现：
>
> - 使用传参的方式：将service层获取到的连接传递到dao层；
> - 使用ThreadLocal：将在service层获取的连接与当前线程绑定，在dao层再获取连接的时候直接从ThreadLocal中取。（可在获取连接的工具类中使用）。
>
> 优缺点：
>
> 传参的方式比较简单易懂，但是会造成代码入侵，将不属于业务数据的连接参数写在方法签名里，不是一个明智之举。ThreadLocal实现方式简洁，没有任何的代码入侵，但是需要注意的是内存泄漏的问题，至于为什么会造成内存泄漏，待我们了解了ThreadLocal的原理之后再来回答这个问题。

#### 3、相关原理

3.1 ThreadLocal内部结构 

<img src="/Users/renbo/Desktop/document/learn_document/imgs/ThreadLocal内部结构.png" style="zoom:50%;" />

3.2 ThreadLocal核心方法源码分析：

1、设置值：set()方法

```java
// 设置值
public void set(T value) {
  Thread t = Thread.currentThread();
  ThreadLocalMap map = getMap(t);
  if (map != null)
    map.set(this, value);
  else
    createMap(t, value);
}

ThreadLocalMap getMap(Thread t) {
  // 从这里能看出在set方法中获取的ThreadLocalMap其实是Thread类中的一个成员变量
  // Thread：ThreadLocal.ThreadLocalMap threadLocals = null;
  return t.threadLocals;
}

void createMap(Thread t, T firstValue) {
  // 如果根据当前线程获取到的ThreadLocalMap为空，则创建一个新的ThreadLocalMap
  // key = this，this指当前ThreadLocal对象，由此证实了ThreadLocalMap的key是ThreadLocal类型
  t.threadLocals = new ThreadLocalMap(this, firstValue);
}

// 设置值实际调用的是该方法
private void set(ThreadLocal<?> key, Object value) {
  // 拿到当前对象的entry数组
  Entry[] tab = table;
  int len = tab.length;
  int i = key.threadLocalHashCode & (len-1);

  for (Entry e = tab[i];
       e != null;
       e = tab[i = nextIndex(i, len)]) {
    ThreadLocal<?> k = e.get();

    if (k == key) {
      e.value = value;
      return;
    }

    if (k == null) {
      replaceStaleEntry(key, value, i);
      return;
    }
  }

  tab[i] = new Entry(key, value);
  int sz = ++size;
  if (!cleanSomeSlots(i, sz) && sz >= threshold)
    rehash();
}

private static int nextIndex(int i, int len) {
  return ((i + 1 < len) ? i + 1 : 0);
}
```











