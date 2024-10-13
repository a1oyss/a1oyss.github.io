---
title: HttpServletRequest的几种获取方式
tags:
  - 源码分析
  - Java
categories:
  - 后端开发
abbrlink: 5457
date: 2022-11-28 20:07:22
updated: 2022-11-28 20:07:22
---
这几天写项目，有分页需求，每个接口上定义一次参数有点麻烦，就想到RuoYi写了个`getPageDomain`的静态方法，其原理是直接通过`HttpServletRequest`对象获取url中的参数。

```java

public static PageDomain getPageDomain(){

    PageDomain pageDomain = new PageDomain();

    pageDomain.setPageNum(Convert.toInt(ServletUtils.getParameter(PAGE_NUM), 1));

    pageDomain.setPageSize(Convert.toInt(ServletUtils.getParameter(PAGE_SIZE), 10));

    pageDomain.setOrderByColumn(ServletUtils.getParameter(ORDER_BY_COLUMN));

    pageDomain.setIsAsc(ServletUtils.getParameter(IS_ASC));

    pageDomain.setReasonable(ServletUtils.getParameterToBool(REASONABLE));

    return pageDomain;

}

```

其中`ServletUtils`中封装了获取请求参数的方法，那么为了在每次请求都能拿到正确的参数，就必须先获取到`HttpServletRequest`，`ServletUtils`的做法是通过`RequestContextHolder`来获取

```java

public static String getParameter(String name){

    return getRequest().getParameter(name);

}

```

```java

public static HttpServletRequest getRequest(){

    return getRequestAttributes().getRequest();

}

```

```java

public static ServletRequestAttributes getRequestAttributes(){

    RequestAttributes attributes = RequestContextHolder.getRequestAttributes();

    return (ServletRequestAttributes) attributes;

}

```

可以看到，最后是通过一个`RequestContextHolder`，将获取到的`RequestAttributes`强转为`ServletRequestAttributes`，接着调用`getRequest`方法。

从字面意思来看，`ServletRequestAttributes`是`Servlet`请求属性，为什么能拿到`HttpServletRequest`对象呢？源码上对该类的描述是：

> Servlet-based implementation of the RequestAttributes interface.

> Accesses objects from servlet request and HTTP session scope, with no distinction between "session" and "global session".

  

大意就是基于Servlet的RequestAttributes接口的实现。

从servlet请求和HTTP会话范围访问对象，至于为啥给了一个方法返回`HttpServletRequest`，大概也没什么特别的原因，只是接下来的实现类要用，所以就返回了一个封装好的`HttpServletRequest`吧，

接着看`RequestContextHolder`，

```java

public abstract class RequestContextHolder  {

    private static final ThreadLocal<RequestAttributes> requestAttributesHolder =

            new NamedThreadLocal<>("Request attributes");

  

    private static final ThreadLocal<RequestAttributes> inheritableRequestAttributesHolder =

            new NamedInheritableThreadLocal<>("Request context");

    @Nullable

    public static RequestAttributes getRequestAttributes() {

        RequestAttributes attributes = requestAttributesHolder.get();

        if (attributes == null) {

            attributes = inheritableRequestAttributesHolder.get();

        }

        return attributes;

    }

}

```

发现其内部维护了两个ThreadLocal变量，可以看到虽然都是`ThreadLocal`，但是一个多了个`Inheritable`，`NamedInheritableThreadLocal`继承了`InheritableThreadLocal`，该类主要是用于子线程创建时，需要自动继承父线程的`ThreadLocal`变量。

然后通过`getRequestAttributes`返回`RequestAttributes`，那么有get就有set，需要找到它是在什么时候被添加进去的。通过查找引用，发现在`RequestContextListener`中调用过`setRequestAttributes`。

```java

@Override

public class RequestContextListener implements ServletRequestListener {

    public void requestInitialized(ServletRequestEvent requestEvent) {

        if (!(requestEvent.getServletRequest() instanceof HttpServletRequest)) {

            throw new IllegalArgumentException(

                    "Request is not an HttpServletRequest: " + requestEvent.getServletRequest());

        }

        HttpServletRequest request = (HttpServletRequest) requestEvent.getServletRequest();

        ServletRequestAttributes attributes = new ServletRequestAttributes(request);

        request.setAttribute(REQUEST_ATTRIBUTES_ATTRIBUTE, attributes);

        LocaleContextHolder.setLocale(request.getLocale());

        RequestContextHolder.setRequestAttributes(attributes);

    }

}

```

该类实现了`ServletRequestListener`接口，源码中的描述是：

> A ServletRequestListener can be implemented by the developer interested in being notified of requests coming in and out of scope in a web component. A request is defined as coming into scope when it is about to enter the first servlet or filter in each web application, as going out of scope when it exits the last servlet or the first filter in the chain.

  

```java

public interface ServletRequestListener extends EventListener {

  

    /**

     * The request is about to go out of scope of the web application.

     * The default implementation is a NO-OP.

     * @param sre Information about the request

     */

    public default void requestDestroyed (ServletRequestEvent sre) {

    }

  

    /**

     * The request is about to come into scope of the web application.

     * The default implementation is a NO-OP.

     * @param sre Information about the request

     */

    public default void requestInitialized (ServletRequestEvent sre) {

    }

}

```

Web应用中，当一个请求进入第一个Servlet或者Filter时，就会触发`requestInitialized`，当退出链中的最后一个Servlet或者第一个Filter时，会触发`requestDestroyed`。`RequestContextListener`重写了`requestInitialized`方法，通过`ServletRequestEvent`拿到了`HttpServletRequest`，并创建`ServletRequestAttributes`，添加到`ThreadLocal`中。