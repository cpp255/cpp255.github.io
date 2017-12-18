---
layout: post
title:  "concurrent atomic包 和 Unsafe 类"
date:   2017-12-17 14:10:20
tagline: "Supporting tagline"
tags : Java,Unsafe,concurrent
published: true
categories: 技术分享
---

#concurrent atomic包 和 Unsafe 类
------
### atomic 包
AtomicBoolean、AtomicInteger、AtomicIntegerArray(数组)、AtomicLong、AtomicLongArray、AtomicReference<V>、AtomicReferenceArray<E>类。   
涉及到更新操作主要使用了 Unsafe 对 value 进行更新，value 是 volatile 修饰的；  

``` java
public class AtomicInteger extends Number implements java.io.Serializable {
// setup to use Unsafe.compareAndSwapInt for updates
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
      try {
        valueOffset = unsafe.objectFieldOffset
            (AtomicInteger.class.getDeclaredField("value"));
      } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value;
}
```

### Unsafe 类
 并发编程下面有一个类是不能避免的，Unsafe，通过使用该类提供的一系列方法，可以执行更低层次不安全的操作；虽然这个类和它的所有方法都是 puplic 的，但是被限制使用再只有可信任的代码才能获得实例；注释大体是这个意思，应用层代码不提供实例化的方法，得通过以下方法调用      
 
 ``` java
 private static Unsafe getUnsafeInstance() throws SecurityException,         NoSuchFieldException, IllegalArgumentException, IllegalAccessException {     Field theUnsafeInstance = Unsafe.class.getDeclaredField("theUnsafe");     theUnsafeInstance.setAccessible(true);     return (Unsafe) theUnsafeInstance.get(Unsafe.class); }

// offset 就是具体类的字段内存偏移量
static {     
    try {     
        unsafe = getUnsafeInstance(); 
        offset = unsafe.objectFieldOffset(UnsafeTest.class.getDeclaredField("flag")); 
    } catch (NoSuchFieldException e) {         
        e.printStackTrace();    
    } catch (IllegalAccessException e) { 
        e.printStackTrace();  
    } 
}

// 类似调用 Unsafe 的方法就可以了
private boolean doSwap(long offset, int expect, int update) {     
  return unsafe.compareAndSwapInt(this, offset, expect, update); 
}

```
 
``` java
/**
 * A collection of methods for performing low-level, unsafe operations.
 * Although the class and all methods are public, use of this class is
 * limited because only trusted code can obtain instances of it.
 *
 * @author John R. Rose
 * @see #getUnsafe
 */

public final class Unsafe {
	// 内部实例的名字，
    private static final Unsafe theUnsafe = new Unsafe();

    // 唯一的get获取实例方法，但是被限制了，根据类加载机制，一般的应用代码肯定会抛出异常，只能通过上面的方法获取
    @CallerSensitive
    public static Unsafe getUnsafe() {
        Class cc = Reflection.getCallerClass();
        if (cc.getClassLoader() != null)
            throw new SecurityException("Unsafe");
        return theUnsafe;
    }
}
```

