---
title: list
date: '2025-12-28T09:38:23+08:00'

categories:
- java 基础
---

# List

##### 1、list.toArray(T[] t) 和 list.toArray()

​	下面代码是jdk ArrayList中的源码



```java
public Object[] toArray() {
        // Estimate size of array; be prepared to see more or fewer elements
        Object[] r = new Object[size()];
        Iterator<E> it = iterator();
        for (int i = 0; i < r.length; i++) {
            if (! it.hasNext()) // fewer elements than expected
                return Arrays.copyOf(r, i);
            r[i] = it.next();
        }
        return it.hasNext() ? finishToArray(r, it) : r;
    }
```



```java
public <T> T[] toArray(T[] a) {
    // Estimate size of array; be prepared to see more or fewer elements
    int size = size();
    T[] r = a.length >= size ? a :
              (T[])java.lang.reflect.Array
              .newInstance(a.getClass().getComponentType(), size);
    Iterator<E> it = iterator();

    for (int i = 0; i < r.length; i++) {
        if (! it.hasNext()) { // fewer elements than expected
            if (a == r) {
                r[i] = null; // null-terminate
            } else if (a.length < i) {
                return Arrays.copyOf(r, i);
            } else {
                System.arraycopy(r, 0, a, 0, i);
                if (a.length > i) {
                    a[i] = null;
                }
            }
            return a;
        }
        r[i] = (T)it.next();
    }
    // more elements than expected
    return it.hasNext() ? finishToArray(r, it) : r;
}
```

1.该方法用了泛型,并且是用在方法的创建中(<T> 相当于定义泛型，T[]是在使用泛型T)
泛型是Java SE 1.5的新特性，泛型的本质是参数化类型，也就是说所操作的数据类型被指定为一个参数。这种参数类型可以用在类、接口和方法的创建中，分别称为泛型类、泛型接口、泛型方法
2.该方法返回集合中所有元素的数组；返回数组的运行时类型与指定数组的运行时类型相同。


3.与 public Object[] toArray() 的比较

从源码中可以看出它仅能返回 Object[]类型的，相当于toArray(new Object[0]) 注意：数组不能强制转换

不带参数的toArray方法，是构造的一个Object数组，然后进行数据拷贝，此时进行转型就会产生ClassCastException

String[] tt =(String[]) list.toArray(new String[0]);
这段代码是没问题的，但我们看到String[] tt =(String[]) list.toArray(new String[0]) 中的参数很奇怪，然而去掉这个参数new String[0]却在运行时报错。。。

该容器中的元素已经用泛型限制了，那里面的元素就应该被当作泛型类型的来看了，然而在目前的java中却不是的，当直接String[] tt =(String[]) list.toArray()时，运行报错。回想一下，应该是java中的强制类型转换只是针对单个对象的，想要偷懒将整个数组转换成另外一种类型的数组是不行的，，这和数组初始化时需要一个个来也是类似的。



带参数的toArray方法，则是根据参数数组的类型，构造了一个对应类型的，长度跟ArrayList的size一致的空数组，虽然方法本身还是以 Object数组的形式返回结果，不过由于构造数组使用的ComponentType跟需要转型的ComponentType一致，就不会产生转型异常。




解决方案． Solutions

　　因此在使用toArray的时候可以参考以下三种方式

　　1. Long[] l = new Long[<total size>];

　　  list.toArray(l);

　　2. Long[] l = (Long[]) list.toArray(new Long[0]);

　　3. Long[] a = new Long[<total size>];

　　  Long[] l = (Long[]) list.toArray(a);


1).参数指定空数组，节省空间
String[] y = x.toArray(new String[0]);
2).指定大数组参数浪费时间，采用反射机制
String[] y = x.toArray(new String[100]); //假设数组size大于100
3).姑且认为最好的
String[] y = x.toArray(new String[x.size()]);


以下代码会出现ClassCastException
List list = new ArrayList(); 
list.add(new Long(1));
list.add(new Long(2)); 
list.add(new Long(3));
list.add(new Long(4)); 
Long[] l = (Long[])list.toArray();//这个语句会出现ClassCastException


处理方式如下面代码：
Long [] l = (Long []) list.toArray(new Long[list.size()]); 

​    