# Java基础

## Static

~~~java
public class SimpleLock {
    private static class Sync extends AbstractQueuedSynchronizer {
        @Override
        protected boolean tryAcquire(int acquires) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        @Override
        protected boolean tryRelease(int releases) {
            if (getState() == 0) throw new IllegalMonitorStateException();
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        @Override
        protected boolean isHeldExclusively() {
            return getState() == 1;
        }
    }

    private final Sync sync = new Sync();

    public void lock() {
        sync.acquire(1);
    }

    public void unlock() {
        sync.release(1);
    }
}

~~~

> Static不是代表只有一份吗？那静态内部类怎么说呢？首先单纯的一个类是不可以被static修饰的，但静态内部类可以，而且它本身还是个类，那我new这个静态内部类的时候，可以有多个对象吗？那不是和static冲突了吗？

### 静态内部类

**静态内部类的 `static`，和静态变量的 `static`，意思不一样。**

#### 先拆清楚：两个 `static` 各管什么

| 写法                    | static 修饰谁  | static 的含义                                  |
| :---------------------- | :------------- | :--------------------------------------------- |
| `static int count;`     | 修饰**字段**   | 这个字段属于**类**，全局共享一份               |
| `static class Inner {}` | 修饰**内部类** | 这个内部类**不持有外部类引用**，是个"独立的类" |

**关键：`static` 修饰类的时候，不是"这个类只有一份实例"，而是"这个类不依赖外部类实例"。**

#### 静态内部类的 static 到底啥意思

对比两种内部类：

```java
public class Outer {
    // 非静态内部类
    class Inner1 {
        // 隐式持有 Outer.this
    }

    // 静态内部类
    static class Inner2 {
        // 不持有 Outer.this，独立
    }
}
```

- **非静态内部类**：每个 `Inner1` 实例都偷偷拿着一个 `Outer` 实例的引用，所以 `new Inner1()` 必须先有 `Outer` 实例
- **静态内部类**：`Inner2` **不持有** `Outer` 实例引用，就是个普通类，只是"写在 Outer 里面"而已

所以静态内部类的 `static` 表达的是：**"我不需要外部类实例就能存在"**，跟"只能有一个实例"没有任何关系。

#### 那能 new 多个对象吗

**当然可以，而且和 static 完全不冲突。**

```java
Outer.Inner2 a = new Outer.Inner2();
Outer.Inner2 b = new Outer.Inner2();
Outer.Inner2 c = new Outer.Inner2();
// a、b、c 是三个独立对象，完全合法
```

`static` 在这里**没有限制实例数量**。它只是说"这个类不依赖外部类实例"，至于你 new 几个，随便。

#### 那"静态"到底体现在哪

体现在**创建方式**和**依赖关系**上：

java

```
// 非静态内部类：必须先有外部实例
Outer outer = new Outer();
Outer.Inner1 i = outer.new Inner1();   // 语法特殊，依附外部实例

// 静态内部类：直接 new，不需要外部实例
Outer.Inner2 i = new Outer.Inner2();   // 跟普通类一样
```



- 非静态内部类：`outer.new Inner1()`，**依附于一个外部实例**
- 静态内部类：`new Outer.Inner2()`，**独立创建**

**"静态" = "不依附外部实例"，不是"只有一个实例"。**

#### 为什么你会觉得冲突

你的直觉是：

> "static 不是表示只有一份吗？那 new 多个不就冲突了？"

这个直觉来自**静态变量**：

```java
static int count;   // 所有实例共享这一份 count
```

但静态变量是"**字段**"，`static` 修饰字段时表示"字段属于类，共享一份"。

而 `static` 修饰**类**时，含义变了——表示"**这个类不持有外部类引用**"。

**同一个关键字 `static`，修饰不同东西，含义不同。**

#### 顺带：什么时候用静态内部类

因为静态内部类**不持有外部引用**，所以：

- **省内存**：不像非静态内部类那样多一个 `this$0` 指针
- **避免内存泄漏**：不会因为内部类实例活着，导致外部类回收不掉
- **语义清晰**：如果内部类不需要访问外部实例成员，就该用 static

这也是为什么 `AQS` 的 `Sync`、`ReentrantLock` 的 `Sync` 都用 `static` —— 它们不需要外部锁实例的引用。

#### 一句话

> **静态内部类的 `static` 不是"只有一份实例"，而是"不持有外部类引用、可独立创建"。** 所以你当然能 `new` 多个静态内部类对象，和 `static` 完全不冲突。冲突感来自把"修饰字段的 static（共享一份）"和"修饰类的 static（不依附外部实例）"混为一谈了。

## 集合

Java 集合框架主要分为两大接口体系：**Collection** 和 **Map**。

### 一、整体结构图

text

```
Iterable
└── Collection
    ├── List（有序、可重复）
    │   ├── ArrayList
    │   ├── LinkedList
    │   └── Vector
    │       └── Stack
    ├── Set（无序、不可重复）
    │   ├── HashSet
    │   │   └── LinkedHashSet
    │   ├── TreeSet
    │   └── (EnumSet)
    └── Queue（队列）
        ├── PriorityQueue
        ├── Deque
        │   ├── ArrayDeque
        │   └── LinkedList
        └── BlockingQueue（并发包）

Map（键值对）
├── HashMap
│   └── LinkedHashMap
├── TreeMap
├── Hashtable
│   └── Properties
└── ConcurrentHashMap（并发包）
```



### 二、Collection 体系

#### 1. List（有序、可重复、有索引）

| 实现类         | 底层结构    | 特点                                    | 适用场景                |
| :------------- | :---------- | :-------------------------------------- | :---------------------- |
| **ArrayList**  | 动态数组    | 查询快 O(1)，增删慢 O(n)，非线程安全    | 查询多、增删少          |
| **LinkedList** | 双向链表    | 增删快 O(1)，查询慢 O(n)，可作队列/栈   | 频繁增删                |
| **Vector**     | 动态数组    | 线程安全（方法加 synchronized），性能低 | 已过时，不推荐          |
| **Stack**      | 继承 Vector | 后进先出（LIFO）                        | 已过时，推荐 ArrayDeque |

#### 2. Set（无序、不可重复）

| 实现类            | 底层结构          | 特点                                    |
| :---------------- | :---------------- | :-------------------------------------- |
| **HashSet**       | HashMap           | 无序，查询快，依赖 hashCode/equals      |
| **LinkedHashSet** | LinkedHashMap     | 保持插入顺序                            |
| **TreeSet**       | TreeMap（红黑树） | 有序（自然排序或 Comparator），O(log n) |
| **EnumSet**       | 位向量            | 专用于枚举，性能极高                    |

#### 3. Queue / Deque（队列）

| 实现类            | 底层结构 | 特点                                                    |
| :---------------- | :------- | :------------------------------------------------------ |
| **PriorityQueue** | 二叉堆   | 按优先级出队（非 FIFO）                                 |
| **ArrayDeque**    | 循环数组 | 高效双端队列，可作栈/队列，性能优于 Stack 和 LinkedList |
| **LinkedList**    | 双向链表 | 也实现了 Deque 接口                                     |

### 三、Map 体系（键值对）

| 实现类                | 底层结构         | 有序性        | 线程安全       | 特点                   |
| :-------------------- | :--------------- | :------------ | :------------- | :--------------------- |
| **HashMap**           | 数组+链表+红黑树 | 无序          | 否             | 最常用，允许 null 键值 |
| **LinkedHashMap**     | HashMap+双向链表 | 插入/访问顺序 | 否             | 可做 LRU 缓存          |
| **TreeMap**           | 红黑树           | 键有序        | 否             | 支持排序，O(log n)     |
| **Hashtable**         | 数组+链表        | 无序          | 是（全表锁）   | 已过时，不允许 null    |
| **Properties**        | 继承 Hashtable   | 无序          | 是             | 配置文件专用           |
| **ConcurrentHashMap** | 数组+链表+红黑树 | 无序          | 是（分段/CAS） | 高并发首选             |

### 四、并发集合（java.util.concurrent）

| 类别         | 实现类                                                       |
| :----------- | :----------------------------------------------------------- |
| 并发 Map     | **ConcurrentHashMap**、ConcurrentSkipListMap                 |
| 并发 List    | **CopyOnWriteArrayList**                                     |
| 并发 Set     | **CopyOnWriteArraySet**、ConcurrentSkipListSet               |
| 阻塞队列     | ArrayBlockingQueue、LinkedBlockingQueue、PriorityBlockingQueue、SynchronousQueue、DelayQueue |
| 双端阻塞队列 | LinkedBlockingDeque                                          |

### 五、选型建议速查

- **需要索引、查询多** → `ArrayList`
- **频繁头尾增删、作栈/队列** → `ArrayDeque` / `LinkedList`
- **去重** → `HashSet`；**去重且保序** → `LinkedHashSet`；**去重且排序** → `TreeSet`
- **键值存储** → `HashMap`；**保序** → `LinkedHashMap`；**排序** → `TreeMap`
- **多线程环境** → `ConcurrentHashMap`、`CopyOnWriteArrayList`、阻塞队列
- **优先级处理** → `PriorityQueue`

### 六、核心对比要点

1. **ArrayList vs LinkedList**：数组 vs 链表，随机访问 vs 频繁增删。
2. **HashMap vs TreeMap vs LinkedHashMap**：哈希无序 vs 红黑树有序 vs 链表保插入序。
3. **HashSet vs TreeSet**：哈希 O(1) 无序 vs 红黑树 O(log n) 有序。
4. **HashMap vs Hashtable vs ConcurrentHashMap**：非线程安全 vs 全表锁 vs CAS+分段锁。
5. **ArrayDeque vs Stack**：推荐用 ArrayDeque 实现栈（性能更好、无同步开销）。

## Duration

`Duration` 是 **Java 8** 引入的一个时间类，位于 `java.time` 包下。它的核心作用是：**表示一段精确的时间间隔（比如几秒、几分钟、几小时）**。

它和 `LocalDateTime`（表示某个具体时刻，比如“2026年10月25日 12:00”）是互补的：一个表示“时间点”，一个表示“时间段”。

------

### 一、基本用法

java

```
// 1. 表示 5 分钟
Duration duration = Duration.ofMinutes(5);

// 2. 表示 2 小时 30 分钟
Duration duration2 = Duration.ofHours(2).plusMinutes(30);

// 3. 获取具体的秒数
long seconds = duration.getSeconds(); // 输出 300

// 4. 转换为毫秒（常用于设置超时时间）
long millis = duration.toMillis(); // 输出 300000
```



------

### 二、最常见的使用场景

#### 1. 设置 Redis 缓存过期时间

这是你在写代码时最常遇到的地方。Spring Data Redis 的 API 要求传入一个 `Duration`：

java

```
// 设置 key 在 10 分钟后过期
redisTemplate.opsForValue().set(key, value, Duration.ofMinutes(10));

// 或者设置 1 天过期
redisTemplate.expire(key, Duration.ofDays(1));
```



#### 2. 计算两个时间点之间的间隔

java

```
LocalDateTime start = LocalDateTime.now();
// ... 执行一些耗时操作 ...
LocalDateTime end = LocalDateTime.now();

// 计算中间相隔了多少时间
Duration timeElapsed = Duration.between(start, end);
System.out.println("耗时：" + timeElapsed.toMillis() + " 毫秒");
```



#### 3. 配合 `Thread.sleep()` 或超时控制

java

```
// 让线程休眠 2 秒钟（Java 19+ 支持直接传 Duration）
// Thread.sleep(Duration.ofSeconds(2)); 

// 在 Java 8 中，通常这样转：
Thread.sleep(Duration.ofSeconds(2).toMillis());
```



------

### 三、常用方法速查

| 方法                     | 作用                                 |
| :----------------------- | :----------------------------------- |
| `Duration.ofSeconds(30)` | 创建 30 秒的间隔                     |
| `Duration.ofMinutes(10)` | 创建 10 分钟的间隔                   |
| `Duration.ofHours(2)`    | 创建 2 小时的间隔                    |
| `Duration.ofDays(1)`     | 创建 1 天的间隔                      |
| `duration.toMillis()`    | 转成毫秒数（long）                   |
| `duration.toSeconds()`   | 转成秒数（long）                     |
| `duration.getSeconds()`  | 获取内部的秒数部分                   |
| `duration.isNegative()`  | 判断是否为负数（比如两个时间算反了） |

------

### 💡 为什么不用 `int` 或 `long` 直接表示时间？

在旧的代码里，你可能会看到：

java

```
redisTemplate.expire(key, 10, TimeUnit.MINUTES);
```



这种写法需要额外传一个 `TimeUnit` 枚举。而 `Duration` 把**数值**和**单位**打包在了一起（`Duration.ofMinutes(10)`），语义更清晰，不容易传错参数（比如把毫秒当成了秒传进去）。

**一句话总结：`Duration` 就是 Java 8 里专门用来表示“一段时间”的类，在 Redis 过期时间、耗时计算、超时设置等场景中，比传统的 `long + TimeUnit` 更优雅、更安全。**
