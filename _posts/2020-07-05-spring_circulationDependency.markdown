---
layout:     post
title:      "Spring源码--循环依赖"
subtitle:   ""
date:       2020-07-05
author:     "Deer"
header-img: "img/star-universe-space-all-clouds-cumulus-clouds.jpg"
tags:
    - Spring
---

最近在面试，被问到了循环依赖有哪几种。因此写下这篇博客，记录一下。（毕竟在面试之前，自己一直以为只有一种，无知的自己，好想锤死自己！）

## Catagory

1. [什么是循环依赖](#什么是循环依赖)
2. [循环依赖的分类](#循环依赖的分类)
	1. [构造函数注入的循环依赖](#构造函数注入的循环依赖)
	2. [set方式注入的循环依赖](#set方式注入的循环依赖)

---


## 什么是循环依赖
> 两个及两个以上的bean互相持有对方，比如：A中有B，B中有C，C中有A



---

## 循环依赖的分类
spring中的bean可以通过构造函数注入，也可以通过set方式注入，那么循环依赖也会存在这两种情况。



### 构造函数注入的循环依赖
首先，这种方式是不可行的，接下来通过例子来跟源码进行调试，说明为什么不可以
```java
package com.deer.study.circleDependency;

import org.springframework.stereotype.Component;

/**
 * @author lu
 * @description
 * @date 2020-07-04
 */

@Component("a")
public class A {


    private B b;

    public void setB(B b) {
        this.b = b;
    }

    public A(B b) {
        this.b = b;
    }
}
```

```java
package com.deer.study.circleDependency;

import org.springframework.stereotype.Component;

/**
 * @author lu
 * @description
 * @date 2020-07-04
 */

@Component("b")
public class B {
    private C c;

    public void setC(C c) {
        this.c = c;
    }

    public B(C c) {
        this.c = c;
    }
}
```

```java
package com.deer.study.circleDependency;

import org.springframework.stereotype.Component;

/**
 * @author lu
 * @description
 * @date 2020-07-04
 */


@Component("c")
public class C {

    private A a;

    public void setA(A a) {
        this.a = a;
    }

    public C(A a) {
        this.a = a;
    }
}
```

```java
package com.deer.study.circleDependency;

import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

/**
 * @author lu
 * @description
 * @date 2020-07-04
 */
@Configuration
@ComponentScan("com.deer.study.circleDependency")
public class Config {
}
```

```java
package com.deer.study.circleDependency;

import org.springframework.context.annotation.AnnotationConfigApplicationContext;

/**
 * @author lu
 * @description
 * @date 2020-07-04
 */
public class Test {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext context=new AnnotationConfigApplicationContext(Config.class);
        System.out.println(context.getBean(A.class));
    }
}

```
程序的运行结果截取部分：

```java
Caused by: org.springframework.beans.factory.BeanCurrentlyInCreationException: Error creating bean with name 'a': Requested bean is currently in creation: Is there an unresolvable circular reference?
	at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.beforeSingletonCreation(DefaultSingletonBeanRegistry.java:345)
	at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:221)
	at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:315)
	at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:199)
	at org.springframework.beans.factory.config.DependencyDescriptor.resolveCandidate(DependencyDescriptor.java:251)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.doResolveDependency(DefaultListableBeanFactory.java:1138)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.resolveDependency(DefaultListableBeanFactory.java:1065)
	at org.springframework.beans.factory.support.ConstructorResolver.resolveAutowiredArgument(ConstructorResolver.java:815)
	at org.springframework.beans.factory.support.ConstructorResolver.createArgumentArray(ConstructorResolver.java:721)
	... 42 more
```

Spring首先实例化A类，在初始化A类的时候，发现通过构造函数需要B类，那么Spring去初始化B类，发现需要C类，
那么接着去初始化C类，发现需要A类，到这里，又去Spring中加载A类：
那么接下来重头戏来了，跟源代码调试：

![代码调试](/img/recursion/circular/circular2.png "代码调试")

![代码调试](/img/recursion/circular/circular3.png "代码调试")

![代码调试](/img/recursion/circular/circular4.png "代码调试")

![代码调试](/img/recursion/circular/circular5.png "代码调试")

![代码调试](/img/recursion/circular/circular6.png "代码调试")

由上面的调试过程可知，当C类试图在加载A的时候，程序抛出异常。那么C无法成功创建，那么依赖C的B，依赖B的A同样
无法创建。所以构造函数注入的循环依赖是无法成功的。









### set方式注入的循环依赖

```java
package com.deer.study.circleDependency;

import org.springframework.stereotype.Component;

/**
 * @author lu
 * @description
 * @date 2020-07-04
 */
@Component("d")

public class D {


    E e;

    public void setE(E e) {
        this.e = e;
    }


}
```

```java
package com.deer.study.circleDependency;

import org.springframework.context.annotation.Scope;
import org.springframework.stereotype.Component;

/**
 * @author lu
 * @description
 * @date 2020-07-04
 */
@Component("e")
public class E {
    D d;

    public void setD(D d) {
        this.d = d;
    }


}
```

```java
package com.deer.study.circleDependency;

import org.springframework.context.annotation.AnnotationConfigApplicationContext;

/**
 * @author lu
 * @description
 * @date 2020-07-04
 */
public class Test {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext context=new AnnotationConfigApplicationContext(Config.class);
        System.out.println(context.getBean(D.class));
    }
}

```
运行结果：
![运行结果](/img/recursion/circular/circular11.png "运行结果")

通过set方式注入的循环依赖，就可以成功！为什么呢？来看调试！


![代码调试](/img/recursion/circular/circular7.png "代码调试")

由上面的调试可知，D依赖E，E依赖D。D在创建的时候，去找E，那么E开始创建，E创建时去找到了D。那么E创建
成功，随之D也创建成功。因此，set方式注入的循环依赖是可行的。

解答一下为什么在构造函数注入的循环依赖，A->B->C->A时的getSingleton("a")为null，而set方式的D->E->D
时的getSingleton("d")就可以获取到对象呢？看代码！

![代码调试](/img/recursion/circular/circular8.png "代码调试")

![代码调试](/img/recursion/circular/circular9.png "代码调试")

singletonObject = singletonFactory.getObject(); 这个方法代表的意思是：如果当前bean存在于spring容器中，
那么就直接获取；如果不存在，那么就去创建这个bean。重头戏来了！如果是通过构造函数注入的，spring会通过这个有参的
构造函数去创建bean，然而如上调试过程，构造函数的参数都获取不到，那么在创建的过程中抛出异常报错，因此创建失败。那么代码也就无法往下继续执行，也就无法执行到
addSingleton(beanName, singletonObject);这个方法。反之，如果是通过set方法注入的，spring会通过无参数
构造函数去实例化这个bean，程序可以顺利执行，创建了bean，赋值给了singletonObject，继续往下执行，执行到
addSingleton(beanName, singletonObject);为什么会强调这个方法呢？来，看代码！

![代码调试](/img/recursion/circular/circular10.png "代码调试")

在这里我们尤为关注的可能就是这个方法中的this.singletonObjects.put(beanName, singletonObject);
对于set方式注入，因为在第一次初始化D的时候，就已经把D对象放入了singletonObjects中，那么当E再获取D的时候，
通过getSingleton方法就可以取到了！然而对于通过构造方式注入的，在创建对象的时候已经出错了，都没执行到这个方法，
因此在getSingleton获取到的是null。


<small>这篇博客后期一定会逐步完善，把不是那么完善的内容放出来的原因就是逼着自己去完善！不然拖延症.....逼自己一把！（下一篇博客：tx-lcn源码分析，淦！）</small>




