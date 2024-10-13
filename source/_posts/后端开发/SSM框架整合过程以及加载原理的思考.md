---
title: SSM框架整合过程以及加载原理的思考
tags:
  - SpringMVC
  - Spring
  - MyBatis
  - 源码分析
categories:
  - 后端开发
abbrlink: 13493
date: 2022-11-28 20:02:18
updated: 2022-11-28 20:02:18
---
#Spring #SpringMVC #Mybatis #源码
# 添加Sring+SpringMVC相关依赖

因为使用的maven，web容器使用的tomcat，所以先在pom.xml中加上：

```xml

<packaging>war</packaging>

```

这样项目就会自动将依赖放进WEB-INF下面的lib中，否则会出现一系列NoClassFound等错误。

添加spring-webmvc依赖：

```xml

<dependency>

  <groupId>org.springframework</groupId>

  <artifactId>spring-webmvc</artifactId>

  <version>${spring.version}</version>

</dependency>

<!-- 方便查看源码 -->

<dependency>

  <groupId>javax.servlet</groupId>

  <artifactId>javax.servlet-api</artifactId>

  <version>4.0.1</version>

</dependency>

```

只添加一个spring-webmvc就够了

![image.png](https://cdn.nlark.com/yuque/0/2022/png/2658344/1649995273228-2e36f9bc-538f-4064-8d88-ed7576b1c6ef.png#averageHue=%233a4147&clientId=ue7018ef1-4828-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=186&id=u22de6707&margin=%5Bobject%20Object%5D&name=image.png&originHeight=233&originWidth=662&originalType=binary&ratio=1&rotation=0&showTitle=false&size=49187&status=done&style=none&taskId=u0eef87b8-2910-4822-8f66-8be5a43415f&title=&width=529.6)

可以看到已经将相关依赖全部加载进去了。

这里有一个问题，@Contoller和@GetMapping等注解是在spring-web中，然而只有这两个注解，不去配置DispatcherServlet是不会处理请求的，也就是说如果只导入spring-web依赖，而没有spring-webmvc，即使有@Controller，也只是个普通的注解。

# 配置spring

在web.xml中配置

```xml

<listener>

  <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>

</listener>

<!-- 采用注解方式配置 -->

<context-param>

  <param-name>contextClass</param-name>

  <param-value>

    <!-- 使用注解实现类 -->

    org.springframework.web.context.support.AnnotationConfigWebApplicationContext

  </param-value>

</context-param>

<context-param>

  <param-name>contextConfigLocation</param-name>

  <!-- 配置类 -->

  <param-value>com.sk.day31.config.ApplicationConfig</param-value>

</context-param>

```

ContextLoaderListener实现了ServletContextListener接口，ServletContextListener在文档中的解释是

> 该接口的实现可以接收关于它们所处的Web应用程序的Servlet上下文变化的通知。为了接收通知事件，必须在Web应用程序的部署描述符中配置该实现类。

  

ServletContextListener 是 Java EE 标准接口之一，类似 Jetty，Tomcat，JBoss 等 java 容器启动时便会触发该接口的 contextInitialized。

## spring加载流程

  

1. 在Web容器启动后，触发ContextLoaderListener，contextInitialized方法被调用，接着调用initWebApplicationContext方法，并将ServletContext当作参数传入

```java

@Override

public void contextInitialized(ServletContextEvent event) {

    initWebApplicationContext(event.getServletContext());

}

```

  

2. 进入initWebApplicationContext方法后，调用createWebApplicationContext

```java

public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {

        ...

    try {

      // Store context in local instance variable, to guarantee that

      // it is available on ServletContext shutdown.

      if (this.context == null) {

        this.context = createWebApplicationContext(servletContext);

      }

        ...

  }

```

  

3. 进入createWebApplicationContext方法， 源码如下：

```java

protected WebApplicationContext createWebApplicationContext(ServletContext sc) {

    Class<?> contextClass = determineContextClass(sc);

    if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {

      throw new ApplicationContextException("Custom context class [" + contextClass.getName() +

          "] is not of type [" + ConfigurableWebApplicationContext.class.getName() + "]");

    }

    return (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);

  }

```

determineContextClass用来返回一个实现了WebApplicationContext接口的类：

```java

protected Class<?> determineContextClass(ServletContext servletContext) {

    String contextClassName = servletContext.getInitParameter(CONTEXT_CLASS_PARAM);

    if (contextClassName != null) {

      try {

        return ClassUtils.forName(contextClassName, ClassUtils.getDefaultClassLoader());

      }

      catch (ClassNotFoundException ex) {

        throw new ApplicationContextException(

            "Failed to load custom context class [" + contextClassName + "]", ex);

      }

    }

    else {

      contextClassName = defaultStrategies.getProperty(WebApplicationContext.class.getName());

      try {

        return ClassUtils.forName(contextClassName, ContextLoader.class.getClassLoader());

      }

      catch (ClassNotFoundException ex) {

        throw new ApplicationContextException(

            "Failed to load default context class [" + contextClassName + "]", ex);

      }

    }

  }

```

determineContextClass根据web.xml中定义的contextClass，来决定返回的实现类，如果没有配置，则返回默认的XmlWebApplicationContext，最后通过BeanUtils.instantiateClass(contextClass)创建对象。

**那么就出现一个问题，XmlWebApplicationContext是什么？**

首先要先讲一下BeanFactory，根据[官方文档](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#spring-core)，BeanFactory接口提供了一个高级配置机制，能够管理任何类型的对象。简单来说就是Spring的容器，用来管理各种Bean。

```java

import org.springframework.beans.factory.InitializingBean;

import org.springframework.beans.factory.xml.XmlBeanFactory;

import org.springframework.core.io.ClassPathResource;

public class MainApp {

   public static void main(String[] args) {

      XmlBeanFactory factory = new XmlBeanFactory(new ClassPathResource("Beans.xml"));

      HelloWorld obj = (HelloWorld) factory.getBean("helloWorld");

      obj.getMessage();

   }

}

```

上述例子就是用了BeanFactory的其中一个实现类XmlBeanFactory来创建Bean。

而ApplicationContext是BeanFactory的扩展，提供了更多的功能，例如：

  

- 用于访问应用程序组件的Bean工厂方法。从ListableBeanFactory继承而来。

- 以通用方式加载文件资源的能力。从ResourceLoader接口继承而来。

- 向注册的听众发布事件的能力。从ApplicationEventPublisher接口继承而来。

- 解决消息的能力，支持国际化。继承自MessageSource接口。

  

ApplicationContext常用的实现类有ClassPathXmlApplicationContext，以及上面提到的WebApplicationContext，这些实现类相当于特化，比如ClassPathXmlApplicationContext就是从类路径中获取配置文件：

```java

public class TestSpring {

    @Test

    public void test(){

        ApplicationContext context=new ClassPathXmlApplicationContext("applicationContext.xml");        

        Hello hello=(Hello)context.getBean("helloWorld");

        hello.say();

    }

}

```

WebApplicationContext是专门为web应用准备的，它允许从相对于Web根目录的路径中装载资源配置文件完成初始化工作。而WebApplicationContext最常用的实现类也就是上面的XmlWebApplicationContext了，此实现类会默认从WEB-INF的目录下读取applicationContext.xml来创建容器，当然这个路径也可以自定义。

  

4. 再回到前面，determineContextClass返回了XmlWebApplicationContext类，并实例化，将其强转为ConfigurableWebApplicationContext，并执行configureAndRefreshWebApplicationContext

```java

if (this.context instanceof ConfigurableWebApplicationContext) {

        ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;

        if (!cwac.isActive()) {

                    ...

          configureAndRefreshWebApplicationContext(cwac, servletContext);

        }

      }

```

  

5. 进入configureAndRefreshWebApplicationContext方法，通过ServletContext获取web.xml中的contextConfigLocation值，设置到XmlWebApplicationContext中。

```java

String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);

    if (configLocationParam != null) {

      wac.setConfigLocation(configLocationParam);

    }

    ...

    wac.refresh();

```

  

6. 执行refresh方法

```java

public void refresh() throws BeansException, IllegalStateException {

    Object var1 = this.startupShutdownMonitor;

    synchronized(this.startupShutdownMonitor) {

       //容器预先准备，记录容器启动时间和标记

      prepareRefresh();

      //创建bean工厂，里面实现了BeanDefinition的装载

      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

      //配置bean工厂的上下文信息，如类装载器等

      prepareBeanFactory(beanFactory);

      try {

        //在BeanDefinition被装载后，提供一个修改BeanFactory的入口

        postProcessBeanFactory(beanFactory);

        //在bean初始化之前，提供对BeanDefinition修改入口，PropertyPlaceholderConfigurer在这里被调用

        invokeBeanFactoryPostProcessors(beanFactory);

        //注册各种BeanPostProcessors，用于在bean被初始化时进行拦截，进行额外初始化操作

        registerBeanPostProcessors(beanFactory);

        //初始化MessageSource

        initMessageSource();

        //初始化上下文事件广播

        initApplicationEventMulticaster();

        //模板方法

        onRefresh();

        //注册监听器

        registerListeners();

        //初始化所有未初始化的非懒加载的单例Bean

        finishBeanFactoryInitialization(beanFactory);

        //发布事件通知

        finishRefresh();

        } catch (BeansException var5) {

            if(this.logger.isWarnEnabled()) {

                this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + var5);

            }

            this.destroyBeans();

            this.cancelRefresh(var5);

            throw var5;

        }

    }

}

```

完成bean的加载。

## JavaConfig方式配置ApplicationConfig

```java

@Configuration

@ComponentScan(basePackages = "com.sk.day31", excludeFilters = {@ComponentScan.Filter(type = FilterType.ANNOTATION, classes = Controller.class)})

public class ApplicationConfig {

}

```

效果类似

```xml

<context:component-scan base-package="com.sk.day31">

  <context:exclude-filter type="annotation" expression="org.springframework.stereotype.Controller"/>-->

</context:component-scan>

```

# 配置spring-webmvc

在web.xml中配置：

```xml

<servlet>

  <servlet-name>dispatcherServlet</servlet-name>

  <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>

  <init-param>

    <param-name>contextClass</param-name>

    <param-value>

      <!-- 使用注解实现类 -->

      org.springframework.web.context.support.AnnotationConfigWebApplicationContext

    </param-value>

  </init-param>

  <init-param>

    <param-name>contextConfigLocation</param-name>

    <param-value>com.sk.day31.config.SpringMvcConfig</param-value>

  </init-param>

  <load-on-startup>1</load-on-startup>

</servlet>

<servlet-mapping>

  <servlet-name>dispatcherServlet</servlet-name>

  <url-pattern>/</url-pattern>

</servlet-mapping>

```

```java

@Configuration

@ComponentScan(basePackages = "com.sk.day31.controller")

@EnableWebMvc

public class SpringMvcConfig implements WebMvcConfigurer {

    @Override

    public void addResourceHandlers(ResourceHandlerRegistry registry) {

        registry.addResourceHandler("/resources/**")

                .addResourceLocations("classpath:/static/")

                .setCacheControl(CacheControl.maxAge(Duration.ofDays(365)));

    }

  

    @Override

    public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {

        configurer.enable();

    }

}

```

@EnableWebMvc效果等于<mvc:annotation-driven/>。

configureDefaultServletHandling方法作用等于<mvc:default-servlet-handler/>

# 配置mybatis

## 添加依赖

```xml

<dependency>

  <groupId>org.springframework</groupId>

  <artifactId>spring-jdbc</artifactId>

  <version>${spring.version}</version>

</dependency>

<dependency>

  <groupId>mysql</groupId>

  <artifactId>mysql-connector-java</artifactId>

  <version>8.0.28</version>

</dependency>

<dependency>

  <groupId>com.alibaba</groupId>

  <artifactId>druid</artifactId>

  <version>1.1.22</version>

</dependency>

<dependency>

  <groupId>org.mybatis</groupId>

  <artifactId>mybatis-spring</artifactId>

  <version>2.0.7</version>

</dependency>

<dependency>

  <groupId>org.mybatis</groupId>

  <artifactId>mybatis</artifactId>

  <version>3.5.9</version>

</dependency>

```

spring-jdbc必须添加，不然会报错java.lang.NoClassDefFoundError: org/springframework/dao/support/DaoSupport。

## 配置类

```java

@Configuration

@PropertySource("classpath:druid.properties")

@MapperScan(basePackages = "com.sk.day31.mapper")

public class MybatisConfig {

    @Value("${jdbc.driver}")

    private String driver;

    @Value("${jdbc.url}")

    private String url;

    @Value("${jdbc.username}")

    private String username;

    @Value("${jdbc.password}")

    private String password;

  

    @Bean

    public DruidDataSource dataSource(){

        DruidDataSource dataSource=new DruidDataSource();

        dataSource.setDriverClassName(driver);

        dataSource.setUrl(url);

        dataSource.setUsername(username);

        dataSource.setPassword(password);

        return dataSource;

    }

  

    @Bean

    public SqlSessionFactory sqlSessionFactory() throws Exception {

        SqlSessionFactoryBean factoryBean = new SqlSessionFactoryBean();

        factoryBean.setTypeAliasesPackage("com.sk.day31.entity");

        factoryBean.setMapperLocations(new PathMatchingResourcePatternResolver().getResources("classpath:mapper/*.xml"));

        factoryBean.setDataSource(dataSource());

        return factoryBean.getObject();

    }

}

  

```

到此就算是整合完毕了