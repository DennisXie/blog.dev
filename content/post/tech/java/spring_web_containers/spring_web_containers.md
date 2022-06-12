---
title: "Spring Web容器创建及管理"
date: 2022-06-08T21:49:58+08:00
draft: true
---
# 1 背景
## 1.1 问题
最近接触到两个问题。问题本身并不难，但是我自己因为对Servlet和Spring不够熟没有回答正确。问题如下：

1. Spring Web程序中, Filter对象能不能使用Spring容器中的Bean？
2. Spring Web程序中, 能不能给Service Bean注入Controller Bean?

### 1.2.2 问题2
这个问题需要分情况

1. 通过Spring Boot加载的Spring Web程序
2. 通过Servlet容器加载的Spring Web程序取决于是否有做Context继承

# 2 分析
> **_备注: 本文基于Servlet 3.0及Spring Web MVC 5.3.x分析_**
## 2.1 通过Servlet容器加载Servlet
### 2.1.1 Servlet 3.0加载Servlet的方式
Servlet 3.0开始支持通过SPI加载Servlet的方案
Spring通过 _org.springframework.web.SpringServletContainerInitializer_ 实现了Servlet 3.0中的 _ServletContainerInitializer_ 接口。
_SpringServletContainerInitializer_ 则会在初始化的时候去加载所有的实现了 _org.springframework.web.WebApplicationInitializer_ 的类。
用户通过实现该类即可创建和加载Servlet对象。
### 2.1.2 创建单Spring容器的Servlet
官方[代码示例](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/web.html#mvc-servlet)如下：
~~~Java
public class MyWebApplicationInitializer implements WebApplicationInitializer {

    @Override
    public void onStartup(ServletContext servletContext) {

        // Load Spring web application configuration
        AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
        context.register(AppConfig.class);

        // Create and register the DispatcherServlet
        DispatcherServlet servlet = new DispatcherServlet(context);
        ServletRegistration.Dynamic registration = servletContext.addServlet("app", servlet);
        registration.setLoadOnStartup(1);
        registration.addMapping("/app/*");
    }
}
~~~
从代码中我们可以看到全文只创建了一个Spring容器，然后DispatcherServlet自身也只持有这个容器。所以该DispatcherServlet只会有一个容器。
### 2.1.3 创建带有容器继承的Servlet
官方[代码示例](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/web.html#mvc-servlet-context-hierarchy)如下:
~~~Java
public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class<?>[] { RootConfig.class };
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[] { App1Config.class };
    }

    @Override
    protected String[] getServletMappings() {
        return new String[] { "/app1/*" };
    }
}
~~~
这段代码就是一个经典的双容器继承模式。
_AbstractAnnotationConfigDispatcherServletInitializer_ 及其父类则会实现相关的Root容器和Servlet的创建。
Servlet自身会创建自身的容器并将Root容器设置为父容器。
一般将数据库访问这一类服务放在父容器中，_Controller_ 和 _ViewResolver_ 这些和Servlet强相关的Bean则放在Servlet自身的容器中。
## 2.2 通过Spring Boot启动Spring Web服务
### 2.2.1 启动流程
简单的启动流程如图所示：
![image](images/spring_boot_start.png)
### 2.2.1 Embedded Server
通过Spring Boot启动Spring Web服务和通过Servlet容器加载Servlet有很大的不同。
Spring Boot会创建Embedded Server，默认情况下Embedded Server不会运行Servlet 3.0+的 _ServletContainerInitializer_ 来加载Servlet。
Embedded Server是由 _org.springframework.boot.web.servlet.server.ServletWebServerFactory_ Bean(注意，这里并不是常规意义上的FactoryBean，只是这个Factory是由Spring容器所管理的)所创建的。
### 2.2.2 加载Servlet
Spring Boot中Servlet, Filter和Listener都是Spring容器管理的Bean。
Servlet, Filter这些Bean都是通过 _RegistrationBean_ 添加到Servlet容器中的。
其中Servlet Bean和RegistrationBean在创建 _ServletWebServerFactory_ Bean的时候即会被创建好。
Embedded Server在启动的时候会运行RegistrationBean的初始化代码，此时就会将Servlet, Filter等Bean添加到Servlet容器中。

# 3 结论
## 3.1 通过Servlet容器加载Spring Web程序
### 3.1.1 无容器继承
这种情况下，第二个问题答案都是可以。
1. 
2. Controller和Service都在同一个容器中，Service自然可以注入Controller。
### 3.1.2 有容器继承
这种情况下，第二个问题答案是不可以。
1. TODO
2. Controller在子容器中，Service在父容器中，父容器是不能获取到子容器中的Bean的。
## 3.2 通过Spring Boot加载Spring Web程序
这种情况下，以上两个问题的答案都是可以。
1. Filter自身也被Spring容器管理，当然可以注入和使用Spring容器中的Bean。
2. Spring Boot中只有一个容器，所以可以在Service中注入Controller。

> **_PS: Service中注入Controller可能会被同事打，建议不要这么干_**
# 4 引用
1. [Spring Web MVC Official Doc](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/web.html#mvc-servlet)
2. [Spring Boot Official Doc](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#web.servlet.embedded-container)
3. [DelegatingFilterProxy](https://www.baeldung.com/spring-delegating-filter-proxy)
