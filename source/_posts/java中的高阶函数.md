---
title: java中的高阶函数
date: 2021-08-08 22:57:40
tags:[CS61B,语法]

---
CS61b中稍微介绍了一下老式java(before java8)中使用*高阶函数*的方法。一开始看到*高阶函数*，我还在想这是什么高阶玩意儿。但是看了他的定义和举例，就让我顿觉：就这？

>A higher order function is a function that treats other functions as data.

```python
def tenX(x):
    return 10*x

def do_twice(f, x):
    return f(f(x))
```

以上分别是课程中对于高阶函数的定义和用python的实例说明。在我看来，这就是c++中函数指针的用法，非常简单（当然，python比c++还要简单）。但是，在老式java中，要实现类似的操作，非常麻烦：首先，我们需要先定义一个接口，里面有我们要使用的“低阶函数”。随后，我们需要创建一个实现这个接口的类。接下来，我们需要在另一个类中创建真正的高阶函数，并且在高阶函数中实例化“低阶函数”所在的类，并且借其实例调用方法。代码如下：

```java
public static int do_twice(IntUnaryFunction f, int x) {
    return f.apply(f.apply(x));// f is a instantiate of IntUnaryFunction。
}

```

非常的繁琐复杂。。一时之间我甚至不知道说什么。但是在cs61B里面，并没有提及现代java(after java8)中是如何实现此类操作的。于是我搜索一番，发现java8使用了functional interface来实现类似的功能，如下：

```java

/**
 * 输入一定数量的参数，然后统一求值
 * @param size 需要求值的个数
 * @param fn 求值函数
 * @return 函数对象
 * 从函数的定义就可以看出，Java函数编程的内在思想还是面向对象
 */
public IntFunction<Integer> addNum(int size, ToIntFunction<List<Integer>> fn) {
    //  声明局部变量，用于存储传入参数
    final List<Integer> args = Lists.newArrayList();
    return new IntFunction<Integer>() {

        @Override
        public Integer apply(int value) {
            //  没有达到定义的数量之前，不求值
            int result = -1;
            if(args.size() == size) {
                result = fn.applyAsInt(args);
            } else {
                args.add(value);
            }
            //  返回结果
            return result;
        }

    };

}
```

看上去也很奇怪.

