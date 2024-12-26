---
title: 线程本地存储：ThreadLocal
description: 介绍了线程本地存储ThreadLoca使用、原理和使用示例等
author: whs
date: 2024-12-26 13:14:52 +0800
categories: [编程, 并发]
tags: [并发, ThreadLocal]
pin: true
math: true
mermaid: true
---
# 线程本地存储模式：没有共享，就没有伤害

我们知道一个变量产生线程安全问题有两个 **必要** 条件：

* 变量在多个线程间共享
* 多个线程同时操作该共享变量

那我们破坏这两个条件中的一个就可以杜绝线程安全问题

例如我们使用 synchronized 关键字或 ReentrantLock 加锁就能使多个线程对共享变量的操作串行化，这样就能保证同一时间只有一个线程操作该变量所以保证线程安全

而我们如果想要变量在线程中独享，使用之前介绍过的局部变量可以实现，局部变量因为存储在线程调用栈的栈帧中所以避免在线程之间共享

本章我们介绍另一种避免变量共享的方式：**线程本地存储** `ThreadLocal`

## ThreadLocal 使用示例

SimpleDateFormat 是 Java 中的一个日期格式化类，但它在多线程环境下是非线程安全的。
主要原因在于其内部状态（如 calendar 和 numberFormat 对象）会被多个线程共享和修改，而这些状态并没有使用同步机制来保证线程安全性。
示例代码见本包下 `SimpleDateFormatTest.java`
在示例代码中我们使用线程池来实现并发操作
ParseDate 任务类无同步措施，线程池执行任务使用的是同一个对象，所以执行会报错
SafeParseDate 类使用了 ThreadLocal 中的 SimpleDateFormat ，能够保证每个线程都能有独立的对象，所以执行不会报错

**注意**：为每个线程分配不同的对象这一功能是在应用层面保证的，ThreadLocal 只是起到简单的容器作用

## ThreadLocal 工作原理

ThreadLocal 的作用是让每个线程都有一个单独的对象，那么我们很容易就想键值对形式的 Map，key 为对应的 Thread 对象，value 为要存储的对象

示意图如下：

但是我们点进去 ThreadLocal 类的 `get()` 方法看却发现事情和我们想的有点不太一样，代码如下：

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            T result = (T) e.value;
            return result;
        }
    }
    return setInitialValue();
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

从上面的代码我们看到最后变量是从 Thread 类中的 ThreadLocalMap 类型的 threadLocals 变量中取出的

下面我们来看看 ThreadLocalMap 的定义：

```java
import java.lang.ref.WeakReference;

static class ThreadLocalMap {

    static class Entry extends WeakReference<ThreadLocal<?>> {

        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }

    private Entry[] table;
}
```

根据上面代码我们发现 ThreadLocalMap 不是 Map 类型，其实是 Entry 类型的数组

而 Entry 是被弱引用封装的 ThreadLocal 对象，不过内部定义了 value 变量存储实际对象

> WeakReference 弱引用：如果一个对象只有弱引用存在，那么在下一次垃圾回收时，无论内存是否充足，该弱引用引用的对象都会被回收
>
> 为什么使用 WeakReference ?
>
> 为了避免内存泄露。如果 ThreadLocal 对象是一个强引用，即使用户不再需要 ThreadLocal 对象，但只要该线程还存活，那么 ThreadLocalMap 的引用仍然会导致 ThreadLocal 和其关联的 Key 无法被垃圾回收，从而引发内存泄露

ThreadLocalMap 示意图如下：
现在我们已经知道了具体存放对象的是 Entry 类型的数组了，让我们往下看取出 Entry 的具体操作过程吧

从上面的 `get()` 代码来看，实际获取 Entry 是 `map.getEntry(this)` 方法，下面是对应的代码：

```java
private Entry getEntry(ThreadLocal<?> key) {
    // 计算索引位置
    // threadLocalHashCode 的分配逻辑是每个 ThreadLocal 对象的 threadLocalHashCode 以固定的步长（0x61c88647）递增
    // & 是位运算，比取模运算%更加高效
    int i = key.threadLocalHashCode & (table.length - 1);
    // 根据索引位置取出Entry对象
    Entry e = table[i];
    // 如果当前索引存储的Key和参数的ThreadLocal相同，直接返回，如果不同说明产生了哈希冲突，执行getEntryAfterMiss()方法
    if (e != null && e.get() == key) {
        return e;
    } else {
        return getEntryAfterMiss(key, i, e);
    }
}

private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key) {
            return e;
        }
        // 当Key即ThreadLocal对象为空时，清除对应的value值
        if (k == null) {
            expungeStaleEntry(i);
        } else {
            // 如果不为空且不等于传入的ThreadLocal对象，则往下一个索引位置循环查找
            i = nextIndex(i, len);
        }
        e = tab[i];
    }
    return null;
}

private static int nextIndex(int i, int len) {
    return ((i + 1 < len) ? i + 1 : 0);
}

```

根据上面的代码可以看出，取对象的逻辑是先根据 ThreadLocal 对象的 `hashcode & len-1` 求出索引，然后根据索引去比对对应索引中 Entry 对象中的 key 是否和传入的 ThreadLocal 对象一致，如果一致就直接返回对应的 value 值，如果不一致就循环便利该数组

那我们可以合理的猜测，存入的逻辑应该是根据 `hashcode & len-1` 运算结果计算索引，如果产生了哈希冲突就往后遍历数组存放在第一个空位置上，实际逻辑和我们猜想的一样。`set(ThreadLocal<?> key, Object value)` 的代码如下：

```java
private void set(ThreadLocal<?> key, Object value) {

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
```

所以总结来说，我们第一直觉猜测数据是存储在 ThreadLocal 中的 Map 结构中，但实际上是存储在各个 Thread 中，且是以 Entry 数组的方式进行存储的。

这样做的优点有下面几个：

1. 从数据的亲缘性来讲，线程专属的数据存储于 Thread 对象是比较合理的，这样独享对象自然就和 Thread 对象同时消亡
2. 不容易产生内存泄露，因为一般情况下 ThreadLocal 的生命周期是长于 Thread，如果数据存放于 ThreadLocal 中的 Map 中，那么 Map 强引用 Thread 就会导致 Thread 只有在 ThreadLocal 回收之后才会被回收。

## ThreadLocal 与内存泄露

上面的章节中，我们两次提到内存泄露，而如果我们使用线程池的话线程永远不会被回收，所以也可能会产生内存泄露问题。所以本章节我们就总结一下这三种内存泄露的原因以及如何避免的

### Thread 内存泄露

Thread 内存泄露在真正的 ThreadLoca 的实现中是不存在的。只是我们一开始猜测的方案中，独享对象是存储在 ThreadLocal 中以 Thread 为 key、独享对象为 value 的 Map 集合，这种方式 Map 对 Thread 对象产生了强引用，会导致即使 Thread 对象生命周期已经结束因为存在这个强引用也不会被回收而产生内存泄露。实际中独享对象是存储在 Thread 中的 ThreadLocalMap 中，所以不会产生内存泄露

### ThreadLocal 内存泄露

ThreadLocal 可能会产生内存泄露的原因是当 ThreadLocal 早于 Thread 消亡时，Thread 存储的 ThreadLocalMap 中的强引用会导致 ThreadLocal 无法被回收导致内存泄露，实际中 Thread 中的 ThreadLocalMap 的底层实现是 Entry 类型的数组，Entry 是封装了 ThreadLocal 的弱引用，所以当 ThreadLocal 对象的强引用消失，那么该对象就会在下一次垃圾回收中被回收掉，所以不会产生内存泄露

### 独享对象内存泄露

使用 ThreadLocal 存储独享对象，Entry 对 ThreadLocal 对象是弱引用，所以 ThreadLocal 没有强引用后就会被回收。而独享对象在 Entry 中是强引用，所以如果不进行手动干预，独享对象是会和 Thread 对象一起消亡，正常情况下这样是能接受的。但是如果在线程池中使用 ThreadLocal，线程池中的线程不会消亡，独享对象就会一直存在，也会导致内存泄露。这种内存泄露就需要我们在应用中解决了，一般使用 `try{}final{}` 方案，示例代码如下：

```java
ExecutorService es;
ThreadLocal tl;
es.execute(()->{
  //ThreadLocal增加变量
  tl.set(obj);
  try {
    // 省略业务逻辑代码
  }finally {
    //手动清理ThreadLocal 
    tl.remove();
  }
});
```

## ThreadLocal 实际中的应用

在我们的应用中大部分的接口都需要登录用户的信息，用户信息又是独属于这个线程的，所以我们可以新增一个 filter 来解析登录用户的信息且存储在 ThreadLocal 中，这样不仅能够省去每个接口解析用户信息的操作，也能保证用户信息对象的线程安全

我们先定义一个用户信息的类：

```java 
public class UserContext {

    private static ThreadLocal<UserContext> local = new ThreadLocal<>();
    public String userId;
    public String userName;
    public List<Integer> userRoleTypes;

    public UserContext(String userId, String userName,
        List<Integer> userRoleTypes) {
        this.userId = userId;
        this.userName = userName;
        this.userRoleTypes = userRoleTypes;
    }

    public static void set(UserContext userContext) {
        local.set(userContext);
    }

    public static void clear() {
        local.remove();
    }

    public static UserContext get() {
        return local.get();
    }

    public static String getUserId() {
        UserContext userContext = local.get();
        if (userContext == null) {
            return null;
        }
        return userContext.userId;
    }

    public static String getUserName() {
        UserContext userContext = local.get();
        if (userContext == null) {
            return null;
        }
        return userContext.userName;
    }
}
```

该类中我们定义了 ThreadLocal 类型的静态对象 local 来存储 UserContext，并且封装了获取对应属性的方法，是用户的感知和操作一般的对象一样

然后定义解析用户信息的 filter 类：

```java
@WebFilter(filterName = "UserFilter")
public class UserFilter implements Filter {

    @Override
    public void init(FilterConfig filterConfig) {
    }

    @SuppressWarnings("all")
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse,
        FilterChain filterChain) throws IOException, ServletException {
        boolean isHttp = false;
        if (servletRequest instanceof HttpServletRequest) {
            isHttp = true;
            // 解析用户上下文并存入UserContext中的ThreadLocal中
            Map<String, String> queryParameterMap = getUrlParams(
                httpServletRequest.getQueryString());
            //当前接口token
            String roleValue = queryParameterMap.get("token");
            UserContext userContext = this.parseToken(token);
            OperUser.set(operUser);
        }
        try {
            filterChain.doFilter(servletRequest, servletResponse);
        } finally {
            if (isHttp) {
                // 手动清除UserContext对象
                OperUser.clear();
            }
        }
    }

    @Override
    public void destroy() {

    }
}
```

该类在请求进来是解析 token 获取用户信息并存入 UserContext 中的 ThreadLocal 中，以备后面使用





