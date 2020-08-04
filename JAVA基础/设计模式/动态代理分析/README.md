### 动态代理

所谓动态代理即java在运行时在内存中创建代理类，增强类的某些能力。java中存在两种动态代理的方式，一种是jdk自带的基于接口进行代理的方式；其二是第三方包cglib提供的基于子类代理的方式，分别介绍两种代理方式。

#### 1、JDK Proxy 接口代理

java提供了  Proxy类、InvocationHandler接口。

##### 使用步骤

```java
1、定义接口，proxy类是基于接口代理的，没有接口无法代理
public interface Student {
    void lean();
    void play();
}

2、将要被代理的类
public class HighSchoolStudnet implements Student {
    public void lean() {
        System.out.println("学习使我快乐");
    }
    public void play() {
        System.out.println("玩耍");
    }
}

3、实现InvocationHandler接口,封装需增强的方法
public class EnchancerHishSchoolStudnet  implements InvocationHandler {
    private HishSchoolStudnet target;  //持有被代理的类
    public EnchancerHishSchoolStudnet(HishSchoolStudnet target){
        this.target = target;
    }
    //针对需要增强的方法做封装
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Object result = null;
        if (method.getName().equalsIgnoreCase("lean")) {
            System.out.println("先平心静气准备学习");
            result = method.invoke(target);
            System.out.println("学习完,做笔记");
        }else {
            System.out.println("方法被拦截,不做任何事,只准学习");
        }
        return result;
    }
}

4、生成代理类，运行，查看运行结果。
public static void main(String[] args) {
    //输出jdk代理类到硬盘中,反编译可看到代理类的内容,一般输出在项目根目录
    System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
    Student me = new HishSchoolStudnet();
    Student proxyStudent = (Student) Proxy.newProxyInstance(HishSchoolStudnet.class.getClassLoader(), new Class[]{Student.class}, new EnchancerHishSchoolStudnet(new HishSchoolStudnet()));
    proxyStudent.play();
    proxyStudent.lean();

}
运行结果为：
    方法被拦截,不做任何事,只准学习
    先平心静气准备学习
    学习使我快乐
    学习完,做笔记
```



##### 原理分析：

将上面例子中生成的代理类拿出来看:

```java
public final class $Proxy0 extends Proxy implements Student {
    /**
    1、该动态代理类继承自Proxy类,实现了Studnet接口;
    2、所有方法都通过调用父类的 成员h的invode方法: super.h.invoke(...)
    3、再看h是什么,h是我们实现了InvocationHandler接口的代理类。
    4、故：jdk动态代理是通过重写代理类的方法，将所有方法调用委派至InvocationHandler
    的invoke方法来完成代理的，而我们的所有代理逻辑，都写在invoke方法中。
            public class Proxy   {
                protected InvocationHandler h;
            }
    **/
    
    private static Method m1;
    private static Method m2;
    private static Method m3;
    private static Method m0;
    private static Method m4;

    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }
    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }
    public final void play() throws  {
        try {
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }
    public final void lean() throws  {
        try {
            super.h.invoke(this, m4, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }
    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m3 = Class.forName("com.asiainfo.usuage.proxy.interfaces.Student").getMethod("play");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
            m4 = Class.forName("com.asiainfo.usuage.proxy.interfaces.Student").getMethod("lean");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```





#### 2、cglib 子类代理

##### 使用方法

```java
1、定义一个接口
public interface Student {
    void lean();
    void play();
}

2、接口的基本实现
public class BaseStudnet implements Student {
     public void lean() {
        System.out.println("学习使我快乐");
    }
    public void play() {
        System.out.println("玩耍");
    }
}

3、实现MethodInterceptor接口，封装需被增强的方法。
public class CglibProxyHighSchoolStudent implements MethodInterceptor {
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        Object result = null;
        if (method.getName().equalsIgnoreCase("lean")) {
            System.out.println("先平心静气准备学习");
            result = methodProxy.invokeSuper(o,objects); //此处注意, 是用methodProxy访问代理类的super方法。
            System.out.println("学习完,做笔记");
        }else {
            System.out.println("方法被拦截,不做任何事,只准学习");
        }
        return result;
    }
}

4、写一个代理类工厂，帮我们快速生成代理类
public class CglibProxyClassFactory {
    private  static CglibProxyClassFactory instance = null;
    private CglibProxyClassFactory() {     }
    public static CglibProxyClassFactory getInstance() {
        if (null == instance) {
            synchronized (CglibProxyClassFactory.class) {
                if (null == instance) {
                    instance = new CglibProxyClassFactory();
                }
            }
        }
        return instance;
    }
    public <T>  T getProxyInstance(T instance, MethodInterceptor interceptor) {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(instance.getClass());
        enhancer.setCallback(interceptor);
        return (T) enhancer.create();
    }
}

5、写一个入口，测试我们cglib动态代理
public static void main(String[] args) throws IllegalAccessException, InstantiationException {
    //设置cglib输出代理类的字节码至指定目录
    System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "C:\\class");

    Student me = new BaseStudnet();
 
    Student student = CglibProxyClassFactory.getInstance().getProxyInstance(me, new CglibProxyHighSchoolStudent());
    student.lean();
    student.play();
}
//运行结果为:
先平心静气准备学习
学习使我快乐
学习完,做笔记
方法被拦截,不做任何事,只准学习
```



##### 原理分析

```java
1、分析cglib提供的方法拦截接口
public interface MethodInterceptor extends Callback {
    /**
    var1:  动态生成的代理子类对象
    var2:  被代理的方法签名
    var3:  被代理的方法入参
    var4:  代理方法签名
    **/
    Object intercept(Object var1, Method var2, Object[] var3, MethodProxy var4) throws Throwable;
}

2、分析输出的代理类，继承了被代理类，实现了cglib的factory接口。
代理类将全部方法都委派至了其MethodInterceptor对象的intercept方法。

public class BaseStudnet$$EnhancerByCGLIB$$9c4e6891 extends BaseStudnet implements Factory {
    public final void lean() {
        MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
        if (var10000 == null) {
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_0;
        }
        if (var10000 != null) {
            var10000.intercept(this, CGLIB$lean$0$Method, CGLIB$emptyArgs, CGLIB$lean$0$Proxy);
        } else {
            super.lean();
        }
    }
}
```





#### 3.两种代理对比

* 基于接口代理(JDK代理)
  基于接口代理，凡是类的方法非public修饰，或者用了static关键字修饰，那这些方法都不能被Spring AOP增强
* 基于CGLib代理(子类代理)
  基于子类代理，凡是类的方法使用了private、static、final修饰，那这些方法都不能被Spring AOP增强