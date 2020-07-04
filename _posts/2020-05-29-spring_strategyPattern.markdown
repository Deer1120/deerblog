---
layout:     post
title:      "Spring源码-策略模式"
subtitle:   ""
date:       2020-05-29
author:     "Deer"
header-img: "img/star-universe-space-all-clouds-cumulus-clouds.jpg"
tags:
    - Spring
---

为了长久的职业发展，不能仅满足于CRUD，因此，开始了源码的阅读。这篇博客描述的是策略模式在Spring源码中的应用。
当然，Spring源码那么庞大，自己也没有读完，所以，源码系列的博客会一直更新。

## Catagory

1. [什么是策略模式](#什么是策略模式)
2. [策略模式在Spring源码中的应用](#策略模式在Spring源码中的应用)
	1. [判断是否进行代理](#判断是否进行代理)
	2. [推断构造函数，实例化对象](#推断构造函数，实例化对象)
	3. [处理合并的BeanDefinition](#处理合并的BeanDefinition)
	4. [循环依赖](#循环依赖)
	5. [属性填充](#属性填充)
	6. [初始化Bean](#初始化Bean)
3. [参考资料](#参考资料)

---


## 什么是策略模式
> 策略模式作为一种软件设计模式，指对象有某个行为，但是在不同的场景中，该行为有不同的实现算法。(百度百科)

将上述的定义用java来表示，那便可以用接口来表示对象的某个行为，接口的实现类就是对象在不同场景的具体行为了。


---

## 策略模式在Spring源码中的应用
一个接口的方法被很多个实现类重写，这就注定了这个方法的功能不是单一的。比如，MergedBeanDefinitionPostProcessor接口的postProcessMergedBeanDefinition方法
在不同的实现类里有不同的实现：

（1）AutowiredAnnotationBeanPostProcessor中的postProcessMergedBeanDefinition方法是找到@Autowired @Value的属性方法

（2）CommonAnnotationBeanPostProcessor中的postProcessMergedBeanDefinition方法是找到@Resouurce的属性方法

..........

我们从Spring的生命周期说起：

### 判断是否进行代理
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsBeforeInstantiation
Spring会判断该对象要不要执行代理，如果前期已经被代理过或者该对象是切面类等不需要被代理的，那就放入adviseBeans中。

```java
@Nullable
protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
	for (BeanPostProcessor bp : getBeanPostProcessors()) {
		if (bp instanceof InstantiationAwareBeanPostProcessor) {
			InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
			Object result = ibp.postProcessBeforeInstantiation(beanClass, beanName);
			if (result != null) {
				return result;
			}
		}
	}
	return null;
}
```
InstantiationAwareBeanPostProcessor接口的postProcessBeforeInstantiation方法


### 推断构造函数，实例化对象
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#determineConstructorsFromBeanPostProcessors
```java
@Nullable
protected Constructor<?>[] determineConstructorsFromBeanPostProcessors(@Nullable Class<?> beanClass, String beanName)
		throws BeansException {

	if (beanClass != null && hasInstantiationAwareBeanPostProcessors()) {
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
				// 推断构造函数的BeanPostProcessor：AutowiredAnnotationBeanPostProcessor
				SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
				Constructor<?>[] ctors = ibp.determineCandidateConstructors(beanClass, beanName);
				if (ctors != null) {
					return ctors;
				}
			}
		}
	}
	return null;
}
```
SmartInstantiationAwareBeanPostProcessor接口的determineCandidateConstructors方法
	


### 处理合并的BeanDefinitio
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#applyMergedBeanDefinitionPostProcessors
为了接下来的属性注入，要在这步中，将BeanDefinition的属性拿出来
```java
protected void applyMergedBeanDefinitionPostProcessors(RootBeanDefinition mbd, Class<?> beanType, String beanName) {
	for (BeanPostProcessor bp : getBeanPostProcessors()) {
		if (bp instanceof MergedBeanDefinitionPostProcessor) {
			MergedBeanDefinitionPostProcessor bdp = (MergedBeanDefinitionPostProcessor) bp;
			/**
			 * 把@Autowired @Value @Resource这些注解的方法，属性都拿出来了
			 * AutowiredAnnotationBeanPostProcessor--@Autowired @Value
			 * CommonAnnotationBeanPostProcessor--@Resource
			 */
			bdp.postProcessMergedBeanDefinition(mbd, beanType, beanName);
		}
	}
}
```
MergedBeanDefinitionPostProcessor的postProcessMergedBeanDefinition的方法


### 循环依赖
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#getEarlyBeanReference
循环依赖，需要提前注入一个完整的bean，那么本该在后面初始化阶段完成的AOP功能就提前在这里完成了。
```java
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
	Object exposedObject = bean;
	if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
				SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
				exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
			}
		}
	}
	return exposedObject;
}
```
SmartInstantiationAwareBeanPostProcessor接口的getEarlyBeanReference方法

### 属性填充
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#populateBean
在属性填充阶段，会有两次策略模式的应用
第一次，判断是否要进行属性注入
```java
/**
 * 判断是否可以完成属性注入。若重写了postProcessAfterInstantiation方法，让其返回false,那么bean就不会完成属性注入
 * 但是spring默认返回true
 *
 * 这一步是判断是不是所有的bean都要完成属性注入，并不是进行属性注入。在下面会再进行属性注入的具体操作
 */
if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
	for (BeanPostProcessor bp : getBeanPostProcessors()) {
		if (bp instanceof InstantiationAwareBeanPostProcessor) {
			InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
			if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
				return;
			}
		}
	}
}

```
InstantiationAwareBeanPostProcessor接口的postProcessAfterInstantiation方法

第二次，完成属性注入
```java
/**
 * 调用后置处理器完成注解(@Autowired @Resource @Value....)的属性填充
 */
for (BeanPostProcessor bp : getBeanPostProcessors()) {
	if (bp instanceof InstantiationAwareBeanPostProcessor) {
		InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
		/**
		 * 在这里完成属性注入
		 */
		PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
		if (pvsToUse == null) {
			if (filteredPds == null) {
				filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
			}
			pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
			if (pvsToUse == null) {
				return;
			}
		}
		pvs = pvsToUse;
	}
}
```
InstantiationAwareBeanPostProcessor接口的postProcessProperties方法

### 初始化Bean
org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsBeforeInitialization
```java
@Override
public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
		throws BeansException {

	Object result = existingBean;
	for (BeanPostProcessor processor : getBeanPostProcessors()) {
		Object current = processor.postProcessBeforeInitialization(result, beanName);
		if (current == null) {
			return result;
		}
		result = current;
	}
	return result;
}
```
BeanPostProcessor接口的postProcessBeforeInitialization方法。


---

## 参考资料 
- <small>[策略模式的定义](https://baike.baidu.com/item/策略模式/646307?fr=aladdin)</small>

---




