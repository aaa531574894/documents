### Unsafe类

首先要明白一点，cas是底层操作系统预留给我们的一个变成接口，它保证了操作的原子性！而java.unsafe类实现了对其的系统调用，分析一下Unsafe这个类，为什么不安全呢？

#### 一、提供了操作堆外内存的能力

```java
public native long allocateMemory(long var1);  //分配内存
public native long reallocateMemory(long var1, long var3);  //重新分配内存 并做旧数据拷贝
public native void freeMemory(long var1);   //释放内存
```

一般来讲，是不需要操作堆外内存的，应该使用堆内内存，将垃圾回收的任务交给GC去做；否则，稍有不慎就会造成内存泄露



#### **二、提供了cas支持**

**cas**即  Unsafe类的 compareAndSwap 方法，它支撑起了juc包，是核心中的核心。

Unsafe 类中的cas操作是原子的，即 compareAndSwapInt，compareAnsSwapLong等操作。以unsafe类的compareAndSwapInt为例，

```java
public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
/**
这个方法有四个参数，第一个参数为对象，第二个参数为 位移，  这两个参数是为了在内存中找到对应的地址的。
第三个参数为期待值，如果内存中的值与期待值相等（即compare），则替换为第四个参数（swap）

Unsafe类的这个方法是个本地方法，具体实现先不管，但是要知道，这个类的操作是原子的。
当然有一种例外情况，比如说 ABA！，即被修改后又被修回原值。
CAS 的缺点，存在这样一个逻辑漏洞：如果一个变量V初次读取的时候是A值
如果在这段期间它的值曾经被改成了B，然后又改回A，那CAS操作就会误认为它从来没有被修改过。这个漏洞称为CAS操作的"ABA"问题。
java.util.concurrent包为了解决这个问题，提供了一个带有标记的原子引用类"AtomicStampedReference"，它可以通过控制变量值的版本来保证CAS的正确性。
不过目前来说这个类比较"鸡肋"，大部分情况下ABA问题并不会影响程序并发的正确性，
如果需要解决ABA问题，使用传统的互斥同步可能回避原子类更加高效。
**/
```



再解读一个方法，Unsafe类的 getAndAddInt，这个方法是基于cas方法的。

```java
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        var5 = this.getIntVolatile(var1, var2);                         //  先根据位移 获取对象在内存中最新的值
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));    //  cas操作，如果和目标值相等就替换为 指定值；否则 继续循环比对，直到满足条件。
    return var5;
}
```





虽然Unsafe设置了安全检查器一般不让用，但是通过强大的反射我们可以对其一探究竟：

```java
public class TestUnsafe {
    public static void main(String[] args) throws Throwable{
        // 通过反射获取unsafe实例
        Field field = Unsafe.class.getDeclaredField("theUnsafe");
        field.setAccessible(true);
        Unsafe unsafe = (Unsafe) field.get(null); 
        
        //通过unsafe分配一个对象
        TestBean testBean = (TestBean) unsafe.allocateInstance(TestBean.class);

        //通过unsafe的方法计算成员变量的位移
        long offset = unsafe.objectFieldOffset(TestBean.class.getDeclaredField("age"));

        testBean.age = 24;
        System.out.println(testBean.age);
        unsafe.compareAndSwapInt(testBean, offset, 24, 25);
        System.out.println(testBean.age);

    }
    //测试bean
    static class TestBean{
        int age = 24;
    }
}

==============================================
！ 输出结果为：
24
25
```

Unsafe类一共提供了三个方法用于计算 偏移量（offset）， 通过 对象实例， offset 这两个参数就可以进行关键的cas原子操作啦！ 但这只是实现原理，一般来讲我们是用不到的。

```java
// 计算静态成员变量的偏移量， 传入通过反射得到的field即可
public native long staticFieldOffset(Field var1);
// 计算对象成员变量的偏移量
public native long objectFieldOffset(Field var1);

```



#### 三、**线程挂起和恢复能力**

```java

public native void unpark(Object var1);

public native void park(boolean var1, long var2);
```

