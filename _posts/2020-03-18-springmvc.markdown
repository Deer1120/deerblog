---
layout:     post
title:      "SpringMVC零配置"
subtitle:   ""
date:       2020-05-14
author:     "Deer"
header-img: "img/star-universe-space-all-clouds-cumulus-clouds.jpg"
tags:
    - Spring
---

大学里接受到的知识，一个SpringMVC项目，要有web.xml,applicationContext.xml,springmvc.xml这三个最基本的配置文件。
其实从Spring4版本以来，官网就推荐零配置了。所以，我们还是要关注官网才能跟得上潮流啊！

## Catagory

1. [SpringMVC配置文件](#SpringMVC配置文件)
2. [零配置的实现](#零配置的实现)
	1. [为什么可以实现零配置](#为什么可以实现零配置)
	2. [代码实现](#代码实现)
3. [参考资料](#参考资料)

---


## SpringMVC配置文件
众所周知，一个SpringMVC起码要有web.xml,applicationContext.xml,springmvc.xml这三个最基本的配置文件。那么这三个配置文件有什么用呢？
我们只有知道它们的具体作用才可以用别的方式来取代。



```xml
<web-app>

    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/app-context.xml</param-value>
    </context-param>

    <servlet>
        <servlet-name>app</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value></param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>app</servlet-name>
        <url-pattern>/app/*</url-pattern>
    </servlet-mapping>

</web-app>
```


以上是官网提供的web.xml配置文件代码。Tomcat启动时，会去解析web.xml文件，listener标签和context-param负责初始化Spring环境。

servlet和servlet-mapping标签是负责将servlet注册给tomcat容器。

applicationContext.xml和springmvc.xml这两个配置文件是负责扫描项目中的配置。

---

## 零配置的实现
### 为什么可以实现零配置
如果我们可以用其他方式来实现这三个配置文件的功能，那么就可以实现零配置了。

在org.springframework.spring-web包下的META-INF下有一个service配置，内容是org.springframework.web.SpringServletContainerInitializer。
Servlet3+要求Web容器(在这里就是tomcat)在启动时必须实现该类的onStartup方法.
![service配置内容](/img/springmvc/mavenclass.png "service配置内容")

HandlesTypes注解的作用：该类的onStartup方法会运行注解中WebApplicationInitializer接口实现类的onStartup方法
![SpringServletContainerInitializer类](/img/springmvc/initializer.png "SpringServletContainerInitializer类")

因此我们可以自己写一个类，实现WebApplicationInitializer接口，重写onStartup方法，在方法中进行Spring环境的初始化，servlet的注册以及项目配置的扫描功能。

### 代码实现

首先建立一个spring web项目。项目采用的内嵌tomcat，零配置的方式。项目的结构图如下：
<img src="/img/springmvc/project-structure.png" alt="项目结构" width="300" height="200">

AppConfig类

```java
package com.deer.study;

import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.EnableWebMvc;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

/**
 * @author deer
 * @description 配置类
 * @date 2020-05-14
 */

@Configuration
@ComponentScan("com.deer.study")
@EnableWebMvc
public class AppConfig implements WebMvcConfigurer {
}

```

IndexController类

```java
package com.deer.study;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseBody;

/**
 * @author deer
 * @description 请求处理类
 * @date 2020-05-14
 */
@Controller
public class IndexController {

    @RequestMapping("/index.do")
    @ResponseBody
    public String index(){
        System.out.println("text index");
        return "index";
    }

}
```

InnerTomcat类

```java
package com.deer.study;

import org.apache.catalina.Context;
import org.apache.catalina.LifecycleListener;
import org.apache.catalina.startup.Tomcat;

/**
 * @author deer
 * @description 内嵌tomcat类
 * @date 2020-05-14
 */
public class InnerTomcat {
    public static void main(String[] args) throws Exception {
        Tomcat tomcat = new Tomcat();
        tomcat.setPort(9098);
        Context context =  tomcat.addContext("/",System.getProperty("java.io.tmpdir"));
        context.addLifecycleListener((LifecycleListener) Class.forName(tomcat.getHost().getConfigClass()).newInstance());
        tomcat.start();
        tomcat.getServer().await();
    }
}

```

WebInitializer类

```java
package com.deer.study;

import org.springframework.web.WebApplicationInitializer;
import org.springframework.web.context.support.AnnotationConfigWebApplicationContext;
import org.springframework.web.servlet.DispatcherServlet;

import javax.servlet.ServletContext;
import javax.servlet.ServletRegistration;

/**
 * @author deer
 * @description springMVC配置类
 * @date 2020-05-14
 */
public class WebInitializer implements WebApplicationInitializer {
    @Override
    public void onStartup(ServletContext servletContext) {
        System.out.println("11111");
        //初始化Spring环境
        AnnotationConfigWebApplicationContext ac = new AnnotationConfigWebApplicationContext();
        //AppConfig类的@Configure注解和@ComponentScan注解会扫描项目中的配置
        ac.register(AppConfig.class);

        //Servlet的注册
        DispatcherServlet servlet=new DispatcherServlet(ac);
        ServletRegistration.Dynamic ds=servletContext.addServlet("ds",servlet);
        ds.addMapping("*.do");
        ds.setLoadOnStartup(1);
    }
}

```

由此，配置结束，不用写基本的xml配置文件，通过WebInitializer类即可完成配置。将项目通过内置tomcat部署并启动，访问localhost:9098/index.do，就能看到在控制台打印出test index，并且在页面上会显示出index。

总的来说，可以实现零配置依靠的是JavaConfig(注解)和Servlet3+新标准。

(给自己要求是每周一更，结果隔了两个月才有新货出来。为了让自己不犯懒，强制自己更新，于是决定了下一篇博客主题：策略模式在Spring源码中的应用。主题都有了，内容还远吗？)

---

## 参考资料 
- <small>[spring官网](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html)</small>

---




