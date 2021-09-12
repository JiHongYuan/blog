---
title: 为什么ArrayList ForEach不能修改结构？  
date: 2021-09-12 14:00:45  
categories: 踩坑记录  
---
# 前言
我相信大家都知道ArrayList使用ForEach遍历时，不能修改结构。  
但是我一直知其然不知其所以然，有一次面试官问我这个问题，我发现从来都没考虑过原理。所以有了这篇博客。
# 一、环境（Java）
```java
public class ArrayListForEachRemoveTest {
    public static void main(String[] args) {
        ArrayList<Integer> list = new ArrayList<>();

        for (int i = 0; i < 10; i++) {
            list.add(i);
        }
    
        // 第40行
        for (Integer integer : list) {
            list.remove(1);
        }
    }
}
```
当我准备编译的时候`idea已经建议 不要在遍历中使用【list.remove】`
# 二、执行代码&处理结果
不出意料的抛了一个异常。
```java
Exception in thread "main" java.util.ConcurrentModificationException
	at java.util.ArrayList$Itr.checkForComodification(ArrayList.java:911)
	at java.util.ArrayList$Itr.next(ArrayList.java:861)
	at com.github.test.ArrayListForEachRemoveTest.main(ArrayListForEachRemoveTest.java:40)
```
明明只是遍历了`list`,为什么报错在第40行循环上呢？  
下面让我们看看java文件编译后的结果。
```java
package com.github.test;

import java.util.ArrayList;
import java.util.Iterator;

public class ArrayListForEachRemoveTest {
    public ArrayListForEachRemoveTest() {
    }

    public static void main(String[] args) {
        ArrayList<Integer> list = new ArrayList();

        for(int i = 0; i < 10; ++i) {
            list.add(i);
        }

        Iterator var4 = list.iterator();

        while(var4.hasNext()) {
            Integer integer = (Integer)var4.next();
            list.remove(1);
        }

    }
}
```
原来`ForEach`语句只是一个语法糖，前端编译器只是将这个语法生成一个迭代器进行处理。
`异常`结合`class文件`可知迭代器执行`next()`方法时抛出异常。
```java
private class Itr implements Iterator<E> {
        int expectedModCount = modCount;
        // ...
        public E next() {
            checkForComodification();
            // ...
        }
  
        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }

}
```
`list`有一个成员变量`modCount`，当创建迭代器时，直接通过外部类赋值`expectedModCount`变量。  
每次执行`next()`方法比较外部类和迭代器中的`Count`。
当`list`调用`add()`、`remove()`等方法时，会修改`modCount`的值。
```java
public E remove(int index) {
    rangeCheck(index);

    modCount++;
    E oldValue = elementData(index);

    int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;
}
```

# 总结
以前写代码从来没有注意到的细节，还是缺少了一些源码阅读的意识。