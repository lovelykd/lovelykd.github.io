---
layout: post
title: "java.util.Objects 方法一览"
date: 2018-05-16 16:18:00 +0800
tags: [java]
---

### 概述

`java.util.Objects`是从JDK 1.7开始加入的一个工具类，它提供了支持null对象的`equals`, `toString`和`hashCode`等方法。在这些场景下，开发者可以调用对应的方法，从而节省编写null check代码的时间，降低了NullPointerException的风险。主要的实现方式就是将null check包装在方法体内，在不为null的前提下调用原始的方法。

### equals

{% highlight java %}
    public static boolean equals(Object a, Object b) {
        return (a == b) || (a != null && a.equals(b));
    }
{% endhighlight %}

首先用`==`运算符比较两个对象是否相等，然后在a不为null的前提下，调用`a.equals(b)`方法来作比较。两种判断有一种返回true则判定a和b相等。

### toString

{% highlight java %}
    public static String toString(Object o) {
        return String.valueOf(o);
    }
{% endhighlight %}

这里直接调用了`java.util.String`类的`valueOf`方法，查看这个方法的代码可以得知：当对象为null时，返回字符串null;当对象不为null时，调用它的`toString`方法

### hashCode

{% highlight java %}
    public static int hashCode(Object o) {
        return o != null ? o.hashCode() : 0;
    }
{% endhighlight %}

对象为null时，返回0；不为null时，调用它的`hashCode`方法。除此之外，Objects类还提供了一个方法，用于计算多个对象的哈希值

{% highlight java %}
    public static int hash(Object... values) {
        return Arrays.hashCode(values);
    }
{% endhighlight %}

此处调用了`java.util.Arrays`类的`hashCode`工具方法。其内部实现逻辑是：

{% highlight java %}
    public static int hashCode(Object a[]) {
        if (a == null)
            return 0;

        int result = 1;

        for (Object element : a)
            result = 31 * result + (element == null ? 0 : element.hashCode());

        return result;
    }
{% endhighlight %}

简而言之就是以1为初始值，循环对象列表，乘31加上元素的哈希值。
当我们定义了一个包含多个域的类，想要override`hashCode`方法，而不知该如何选取hash算法的时候，`Objects.hash(x, y, z)`可能是一个不错的选择。
此处顺便提一下hashCode和equals方法的规约(Contract)：

> If two objects are equal according to the equals(Object) method, 
> then calling the hashCode method on each of the two objects must produce the same integer result. 

意思是如果两个对象经过`equals`方法判定相等，那么他们的`hashCode`方法应该返回同样的结果。所以在override`equals`方法的时候，不要忘记override`hashCode`方法。这句话我最初是在 _Effective Java_ 一书上看到的，后来当我仔细看`equals`方法的Java doc的时候，发现原来它早就在那里：

> Note that it is generally necessary to override the hashCode method whenever this method is overridden, 
> so as to maintain the general contract for the hashCode method, which states that equal objects must have equal hash codes.

所以其实学习框架的使用或者研究其源码的时候，阅读Java doc和官方提供的文档是最直接的方式，书本上说的东西只不过把这些内容重新组织了而已。在阅读别人写的注释和文档的同时，我们也可以从中学习到怎么去写好它们。

### requireNonNull

这个方法主要用于参数的null校验，当对象为null时抛出空指针异常，否则返回该对象。

{% highlight java %}
    public static <T> T requireNonNull(T obj) {
        if (obj == null)
            throw new NullPointerException();
        return obj;
    }
{% endhighlight %}

如果我们在做函数参数校验的时候，采取这样的策略，那么我们可以在方法体最开始加上

{% highlight java%}
  Objects.requireNonNull(obj);
{% endhighlight %}

效果和使用 [lombok](https://projectlombok.org/) 的 `@NonNull` 注解一样。说不定两者之间是有某种关联的，有时间可以研究一下源码。

### isNull & nonNull 

这两个方法是1.8加入的，其内容很简单，就是把 `obj == null` 和 `obj != null` 这两个条件判断语句封装成方法了。不知道从什么时候开始，在项目的代码里面经常能看到如下的代码：

{% highlight java %}
    if(Objects.isNull(obj)){
        // do something
    }
{% endhighlight %}

不禁要问了，这样写相比于直接写条件语句有什么好处呢？可读性更强？如果你说 `list.isEmpty()` 相比于 `list.size() == 0` 可读性更强，似乎有几分道理，但是如果要说 `Objects.isNull(obj)` 比 `obj == null` 可读性更强，那就有点牵强了，因为`obj == null` 已经够直观了。
仔细一想，方法调用还会带来额外的开销（尽管比较小），那为何不直接写条件语句呢？不知道是谁起了这个头，然后大家就开始盲目效仿（不明白为什么要这样写，好像很厉害的样子，我也这样写好了）。

其实只要稍微看一下Java doc就能明白了：

> This method exists to be used as a java.util.function.Predicate, filter(Objects::isNull)

这两个方法其实是在Java8引入lamba表达式之后加入并用作function predicate的。举例而言，相比于

{% highlight java %}
    list.stream().filter(p -> p != null)
{% endhighlight %}

下面的写法会更简洁

{% highlight java %}
    list.stream().filter(Objects::nonNull)
{% endhighlight %}

所以要了解清楚用途再去使用，不能盲目地去效仿别人写的代码。