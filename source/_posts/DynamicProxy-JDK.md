---
title: 动态代理之JDK Proxy
date: 2018-06-19 17:11:34
tags:
    - Java
    - DynamicProxy
---

先来看看什么是静态代理:

## 静态代理
代码如下:
```java
package ren.eggpain;

import java.util.Arrays;

public class StaticProxy {
  // 抽象角色
  interface Barkable {
    void bark(String... args);
  }
  
  // 真实角色
  static class Husky implements Barkable {
    @Override
    public void bark(String... args) {
      System.out.println("barking from husky with args: " + Arrays.toString(args));
    }
  }
  
  // 代理角色
  static class ProxyBarkable implements Barkable {

    private Barkable barkable;

    public ProxyBarkable(Barkable barkable) {
      this.barkable = barkable;
    }

    @Override
    public void bark(String... args) {
      // 前置处理
      System.out.println("Before bark");
      barkable.bark(args);
      // 后置处理
      System.out.println("After bark");
    }

  }

  public static void main(String[] args) {
    Barkable proxyBarkable = new ProxyBarkable(new Husky());
    proxyBarkable.bark("Hello", "Wildog");
  }
  
}
```
运行结果如下:

    Before bark
    barking from husky with args: [Hello, Wildog]
    After bark

对于以上示例, 可以抽象为:
![StaticProxy](/images/Barkable.png  "StaticProxy")

可以看到, 代理模式通过接口、聚合、委托来实现:
  + Barkable: 接口类.
  + Husky: target类，实际业务逻辑执行者.
  + ProxyBarkable: proxy类，聚合target类，并在方法调用时执行代理逻辑，最后调用target方法来委托.

代理模式的优点：
  + 职责清晰，target类关注业务逻辑，proxy类用于处理非业务逻辑
  + 高扩展性，由于接口类固定，proxy并不关心具体执行业务的target类

## 动态代理
以上就是静态代理，但实际使用过程中我们可能会对target类的很多方法都进行代理，那么实现过程中就会写很多相同的代码，有没有可能我们写一个方法来完成所有的事情呢?

下面我们来看看JDK提供的动态代理:
```java 
package ren.eggpain;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.util.Arrays;

public class DynamicProxy {

  interface Barkable {
    void bark(String... args);
  }

  static class Husky implements Barkable {
    @Override
    public void bark(String... args) {
      System.out.println("Bark from husky with args:" + Arrays.toString(args));
    }
  }

  static class BarkableHandler implements InvocationHandler {
    Barkable barkable;

    BarkableHandler(Barkable barkable) {
      this.barkable = barkable;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      System.out.println("Before calling: " + method.getName());
      Object ret = method.invoke(barkable, args);
      System.out.println("After calling: " + method.getName());
      return ret;
    }
  }

  public static void main(String[] args) {
    InvocationHandler invocationHandler = new BarkableHandler(new Husky());
    Barkable barkable = (Barkable) Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(),
        new Class[]{Barkable.class}, invocationHandler);
    barkable.bark("Wow");

  }
  
}

```

执行结果如下:

    Before calling: bark
    Bark from husky with args:[Wow]
    After calling: bark
    Before calling: bite
    Bite from husky
    After calling: bite


总结一下， 动态代理的几个步骤:
  + 创建抽象角色
  + 创建真实角色
  + 实现InvocationHandler, 聚合真实角色
  + 创建代理对象

## 源码分析
### InvocationHandler类

```java 
public interface InvocationHandler {
    
    /**
     * @Param proxy 生成的代理对象
     * @Param method 调用的方法
     * @Param args 调用的参数列表
     * @Return 方法调用之后的返回值
     */
    public Object invoke(Object proxy, Method method, Object[] args)
        throws Throwable;
}
```

### Proxy类
Proxy类用于创建代理对象，有以下公开静态方法:
```java
// 获取指定interface代理类
public static Class<?> getProxyClass(ClassLoader loader, Class<?>...interfaces){}
// 创建指定接口的代理对象
public static Object newProxyInstance(ClassLoader loader, Class<?>...interfaces){}
// 判断指定类是否为代理类
public static boolean isProxyClass(Class<?> cl){}
// 获取代理类的InvocationHandler
public static InvocationHandler getInvocationHandler(Object proxy){}
```
Proxy类的非静态成员有:
```java
protected InvocationHandler h;
```
创建代理对象时，实际生成的Class类继承自Proxy, 并且有一个``Proxy(InvocationHandler h)``作为构造器.


看看如何创建代理对象:
``newProxyInstance``:
```java
public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException
    {
        Objects.requireNonNull(h);

        final Class<?>[] intfs = interfaces.clone();
        final SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
        }

        /*
         * Look up or generate the designated proxy class.
         */
        // 获取代理的class对象
        Class<?> cl = getProxyClass0(loader, intfs);

        /*
         * Invoke its constructor with the designated invocation handler.
         */
        try {
            if (sm != null) {
                checkNewProxyPermission(Reflection.getCallerClass(), cl);
            }
            // 获取对应的构造器
            final Constructor<?> cons = cl.getConstructor(constructorParams);
            final InvocationHandler ih = h;
            if (!Modifier.isPublic(cl.getModifiers())) {
                AccessController.doPrivileged(new PrivilegedAction<Void>() {
                    public Void run() {
                        cons.setAccessible(true);
                        return null;
                    }
                });
            }
            // 创建代理对象
            return cons.newInstance(new Object[]{h});
        } catch (IllegalAccessException|InstantiationException e) {
            throw new InternalError(e.toString(), e);
        } catch (InvocationTargetException e) {
            Throwable t = e.getCause();
            if (t instanceof RuntimeException) {
                throw (RuntimeException) t;
            } else {
                throw new InternalError(t.toString(), t);
            }
        } catch (NoSuchMethodException e) {
            throw new InternalError(e.toString(), e);
        }
    }
```

往里面看，``getProxyClass0``：
```java
private static Class<?> getProxyClass0(ClassLoader loader,
                                           Class<?>... interfaces) {
        if (interfaces.length > 65535) {
            throw new IllegalArgumentException("interface limit exceeded");
        }

        // If the proxy class defined by the given loader implementing
        // the given interfaces exists, this will simply return the cached copy;
        // otherwise, it will create the proxy class via the ProxyClassFactory
        return proxyClassCache.get(loader, interfaces);
}
```

``proxyClassCache``:
```java
private static final WeakCache<ClassLoader, Class<?>[], Class<?>>
        proxyClassCache = new WeakCache<>(new KeyFactory(), new ProxyClassFactory());
```
proxyClassCache是一个WeakCache，拿不到缓存时通过ProxyClassFactory来创建类:
```java
 private static final class ProxyClassFactory
        implements BiFunction<ClassLoader, Class<?>[], Class<?>>
    {
        // prefix for all proxy class names
        private static final String proxyClassNamePrefix = "$Proxy";

        // next number to use for generation of unique proxy class names
        private static final AtomicLong nextUniqueNumber = new AtomicLong();

        @Override
        public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {

            Map<Class<?>, Boolean> interfaceSet = new IdentityHashMap<>(interfaces.length);
            for (Class<?> intf : interfaces) {
                /*
                 * Verify that the class loader resolves the name of this
                 * interface to the same Class object.
                 */
                Class<?> interfaceClass = null;
                try {
                    interfaceClass = Class.forName(intf.getName(), false, loader);
                } catch (ClassNotFoundException e) {
                }
                if (interfaceClass != intf) {
                    throw new IllegalArgumentException(
                        intf + " is not visible from class loader");
                }
                /*
                 * Verify that the Class object actually represents an
                 * interface.
                 */
                if (!interfaceClass.isInterface()) {
                    throw new IllegalArgumentException(
                        interfaceClass.getName() + " is not an interface");
                }
                /*
                 * Verify that this interface is not a duplicate.
                 */
                if (interfaceSet.put(interfaceClass, Boolean.TRUE) != null) {
                    throw new IllegalArgumentException(
                        "repeated interface: " + interfaceClass.getName());
                }
            }

            String proxyPkg = null;     // package to define proxy class in
            int accessFlags = Modifier.PUBLIC | Modifier.FINAL;

            /*
             * Record the package of a non-public proxy interface so that the
             * proxy class will be defined in the same package.  Verify that
             * all non-public proxy interfaces are in the same package.
             */
            for (Class<?> intf : interfaces) {
                int flags = intf.getModifiers();
                if (!Modifier.isPublic(flags)) {
                    accessFlags = Modifier.FINAL;
                    String name = intf.getName();
                    int n = name.lastIndexOf('.');
                    String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
                    if (proxyPkg == null) {
                        proxyPkg = pkg;
                    } else if (!pkg.equals(proxyPkg)) {
                        throw new IllegalArgumentException(
                            "non-public interfaces from different packages");
                    }
                }
            }

            if (proxyPkg == null) {
                // if no non-public proxy interfaces, use com.sun.proxy package
                proxyPkg = ReflectUtil.PROXY_PACKAGE + ".";
            }

            /*
             * Choose a name for the proxy class to generate.
             */
            long num = nextUniqueNumber.getAndIncrement();
            String proxyName = proxyPkg + proxyClassNamePrefix + num;

            /*
             * Generate the specified proxy class.
             */
            byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
                proxyName, interfaces, accessFlags);
            try {
                return defineClass0(loader, proxyName,
                                    proxyClassFile, 0, proxyClassFile.length);
            } catch (ClassFormatError e) {
                /*
                 * A ClassFormatError here means that (barring bugs in the
                 * proxy class generation code) there was some other
                 * invalid aspect of the arguments supplied to the proxy
                 * class creation (such as virtual machine limitations
                 * exceeded).
                 */
                throw new IllegalArgumentException(e.toString());
            }
        }
    }
```
从以上代码我们可以看出:
 + 使用JDK的代理，必须使用接口来创建代理
 + 非public的接口只允许有一个，因为会将产生的代理类的包声明方到对应的包下面，多个会出现访问不到，创建类时会异常，这里遍历interface，fast-fail

最后调用 ProxyGenerator.generateProxyClass以及defineClass0, defineClass0为native方法, 
generateProxyClass调用实例的generateClassFile, 涉及到字节码层面的内容，部分方法如下:
```java
try {
        var14.writeInt(-889275714); // cafe babe
        var14.writeShort(0);
        var14.writeShort(49);
        this.cp.write(var14);
        var14.writeShort(this.accessFlags);
        var14.writeShort(this.cp.getClass(dotToSlash(this.className)));
        var14.writeShort(this.cp.getClass("java/lang/reflect/Proxy"));
        var14.writeShort(this.interfaces.length);
        Class[] var17 = this.interfaces;
        int var18 = var17.length;

        for(int var19 = 0; var19 < var18; ++var19) {
          Class var22 = var17[var19];
          var14.writeShort(this.cp.getClass(dotToSlash(var22.getName())));
        }

        var14.writeShort(this.fields.size());
        var15 = this.fields.iterator();

        while(var15.hasNext()) {
          ProxyGenerator.FieldInfo var20 = (ProxyGenerator.FieldInfo)var15.next();
          var20.write(var14);
        }

        var14.writeShort(this.methods.size());
        var15 = this.methods.iterator();

        while(var15.hasNext()) {
          ProxyGenerator.MethodInfo var21 = (ProxyGenerator.MethodInfo)var15.next();
          var21.write(var14);
        }

        var14.writeShort(0);
        return var13.toByteArray();
      } catch (IOException var9) {
        throw new InternalError("unexpected I/O Exception", var9);
      }

```

创建类结构之后，通过调用defineClass0来获取对应的Class实例, 该Class包含一个InvocationHandler作为参数的构造器, 
在newProxyInstance中调用constructor来创建实例.


输出代理类的class文件：
```java

  public static void main(String[] args) throws IOException {
    byte[] bytes = ProxyGenerator.generateProxyClass("ProxyBarkable", new Class[]{Barkable.class});
    FileOutputStream fileOutputStream = new FileOutputStream("/home/daniel/tmp/ProxyBarkable.class");
    fileOutputStream.write(bytes);
  }
```
通过javap查看类结构:
```bash
javap -v ProxyBarkable.class
```
部分内容如下:

    Classfile /home/drjr/tmp/ProxyBarkable.class
            Last modified Jun 19, 2018; size 2134 bytes
            MD5 checksum 64aaf34e3586f2db8400bcb1e6dd74b2
          public final class ProxyBarkable extends java.lang.reflect.Proxy implements ren.eggpain.DynamicProxy$Barkable
            minor version: 0
            major version: 49
            flags: ACC_PUBLIC, ACC_FINAL, ACC_SUPER
          Constant pool:
              #16 = Fieldref           #6.#15        // java/lang/reflect/Proxy.h:Ljava/lang/reflect/InvocationHandler;
              #20 = Fieldref           #18.#19       // ProxyBarkable.m1:Ljava/lang/reflect/Method;
              #50 = Fieldref           #18.#49       // ProxyBarkable.m4:Ljava/lang/reflect/Method;
              #55 = Fieldref           #18.#54       // ProxyBarkable.m2:Ljava/lang/reflect/Method;
              #62 = Fieldref           #18.#61       // ProxyBarkable.m3:Ljava/lang/reflect/Method;
              #67 = Fieldref           #18.#66       // ProxyBarkable.m0:Ljava/lang/reflect/Method;
          {
            public ProxyBarkable(java.lang.reflect.InvocationHandler) throws ;
              descriptor: (Ljava/lang/reflect/InvocationHandler;)V
              flags: ACC_PUBLIC
              Code:
                stack=10, locals=2, args_size=2
                   0: aload_0
                   1: aload_1
                   2: invokespecial #8                  // Method java/lang/reflect/Proxy."<init>":(Ljava/lang/reflect/InvocationHandler;)V
                   5: return
              Exceptions:
                throws
                
            public final void bite() throws ;
              descriptor: ()V
              flags: ACC_PUBLIC, ACC_FINAL
              Code:
                stack=10, locals=2, args_size=1
                   0: aload_0
                   1: getfield      #16                 // Field java/lang/reflect/Proxy.h:Ljava/lang/reflect/InvocationHandler;
                   4: aload_0
                   5: getstatic     #50                 // Field m4:Ljava/lang/reflect/Method;
                   8: aconst_null
                   // 最终的调用
                   9: invokeinterface #28,  4           // InterfaceMethod java/lang/reflect/InvocationHandler.invoke:(Ljava/lang/Object;Ljava/lang/reflect/Method;[Ljava/lang/Object;)Ljava/lang/Object;
                  14: pop
                  15: return
                  16: athrow
                  17: astore_1
                  18: new           #42                 // class java/lang/reflect/UndeclaredThrowableException
                  21: dup
                  22: aload_1
                  23: invokespecial #45                 // Method java/lang/reflect/UndeclaredThrowableException."<init>":(Ljava/lang/Throwable;)V
                  26: athrow
                Exception table:
                   from    to  target type
                       0    16    16   Class java/lang/Error
                       0    16    16   Class java/lang/RuntimeException
                       0    16    17   Class java/lang/Throwable
              Exceptions:
                throws
          
          }


可以观察到, 生成的类为public final类，　包含5个字段，其中有4个字段为静态的(m0,m1,m2,m3,m4), 1个实例成员(InvocationHandler).

观察bite方法, 实际最终的调用是invocationHandler.invoke方法.

查看m0~m4对应哪些方法:
```java
  public static void main(String[] args) throws IllegalAccessException {
    Class<?> clz = Proxy.getProxyClass(Thread.currentThread().getContextClassLoader(), Barkable.class);
    for (Field field : clz.getDeclaredFields()) {
      if (Modifier.isStatic(field.getModifiers())) {
        field.setAccessible(true);
        Method method = (Method) field.get(null);
        System.out.println(field.getName() + " -> " + method.getName());
      }
    }
  }
```
输出:
    
    m1 -> equals
    m4 -> bite
    m2 -> toString
    m3 -> bark
    m0 -> hashCode

小结：
  + JDK动态代理使用的是接口实现， 聚合InvocationHandler, 在InvocationHandler中操作target类
  + 需要控制创建代理对象过程(当然所有都需要， 可以通过各种方式决定代理策略)
  + JDK动态代理无法代理普通类的方法，必须是接口方法