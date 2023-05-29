---
title: Spring MVC 之 DispatchServlet
categories:
  - Blog
  - SpringMVC
tags:
  - Spring
  - SpringMVC
---

## DispatchServlet 源码分析

首先我们看看 DispatcherServlet 的类结构，可以清楚地发现实际 DispatcherServlet 就是 Servlet 接口的一个子类。

![](https://raw.githubusercontent.com/Traserve/traserve.github.io/master/_posts/blog/Spring%20MVC/images/DispatcherServlet.png){: .align-center}

#### DispatchServlet 初始化

既然最顶层是 Servlet，那么初始化入口肯定是`Servlet#init(ServletConfig)`方法，沿着这个思路去看源码，可以得大体框架图：

<img src="https://raw.githubusercontent.com/Traserve/traserve.github.io/master/_posts/blog/Spring%20MVC/images/SpringMVC-DispatchServlet-Init.drawio.png" alt="image-20230529215830090" style="zoom:80%;" />{: .align-center}

##### HttpServletBean#init 

```java
@Override
public final void init() throws ServletException {
 
    // Set bean properties from init parameters.
    // 解析web.xml中init-param标签，存储到PropertyValues
    PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
    if (!pvs.isEmpty()) {//通过xml方式配置springmvc，会进入这个if。如果是注解方式则不进入
        try {
            //这里的this指向的DispatcherServlet，下面的代码是给DispatcherServlet中contextLocation属性赋值
            BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
            ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
            bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
            initBeanWrapper(bw);
            bw.setPropertyValues(pvs, true); //赋值操作
        }
        catch (BeansException ex) {
            if (logger.isErrorEnabled()) {
                logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
            }
            throw ex;
        }
    }
 
    // Let subclasses do whatever initialization they like.
    initServletBean(); //子类实现
}
```

 这段代码就是保存springmvc配置的，基于 xml 方式配置 springmvc，会进入 if 分支。`PropertyValues` 是一个键值对，内容就是 xml 配置 `init-param` 标签，`key=contextConfigLocation`，`value=classpath:springmvc04.xml`。

##### FrameworkServlet#initServletBean

 这个方法大部分都是日志检查，最核心方法就是initWebApplicationContext，初始化IOC容器。initFrameworkServlet目前是空方法。

```java
protected final void initServletBean() throws ServletException {
    getServletContext().log("Initializing Spring " + getClass().getSimpleName() + " '" + getServletName() + "'");
    if (logger.isInfoEnabled()) {
        logger.info("Initializing Servlet '" + getServletName() + "'");
    }
    long startTime = System.currentTimeMillis();

    try {
        //初始化IOC容器 ApplicationContext
        this.webApplicationContext = initWebApplicationContext();
        initFrameworkServlet(); // 这方法是空
    }
    catch (ServletException | RuntimeException ex) {
        logger.error("Context initialization failed", ex);
        throw ex;
    }

    if (logger.isDebugEnabled()) {
        String value = this.enableLoggingRequestDetails ?
                "shown which may lead to unsafe logging of potentially sensitive data" :
                "masked to prevent unsafe logging of potentially sensitive data";
        logger.debug("enableLoggingRequestDetails='" + this.enableLoggingRequestDetails +
                "': request parameters and headers will be " + value);
    }

    if (logger.isInfoEnabled()) {
        logger.info("Completed initialization in " + (System.currentTimeMillis() - startTime) + " ms");
    }
}
```

```java
protected WebApplicationContext initWebApplicationContext() {
    WebApplicationContext rootContext =
            WebApplicationContextUtils.getWebApplicationContext(getServletContext());
    WebApplicationContext wac = null;
    //如果是注解方式的springmvc进入if分支，xml方式不进入
    if (this.webApplicationContext != null) {
        // A context instance was injected at construction time -> use it
        wac = this.webApplicationContext;
        if (wac instanceof ConfigurableWebApplicationContext) {
            ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
            if (!cwac.isActive()) {
                // The context has not yet been refreshed -> provide services such as
                // setting the parent context, setting the application context id, etc
                if (cwac.getParent() == null) {
                    // The context instance was injected without an explicit parent -> set
                    // the root application context (if any; may be null) as the parent
                    cwac.setParent(rootContext);
                }
                configureAndRefreshWebApplicationContext(cwac);
            }
        }
    }
    if (wac == null) {
        // No context instance was injected at construction time -> see if one
        // has been registered in the servlet context. If one exists, it is assumed
        // that the parent context (if any) has already been set and that the
        // user has performed any initialization such as setting the context id
        wac = findWebApplicationContext();
    }
    if (wac == null) {
        // No context instance is defined for this servlet -> create a local one
        wac = createWebApplicationContext(rootContext); //这里创建ioc容器
    }

    //spring异步事件，在创建ioc容器时会产生一个异步事件，去刷新ioc容器
    //这里简单起见，假设没有接收到事件，刷新操作会在当前线程完成
    if (!this.refreshEventReceived) {
        // Either the context is not a ConfigurableApplicationContext with refresh
        // support or the context injected at construction time had already been
        // refreshed -> trigger initial onRefresh manually here.
        synchronized (this.onRefreshMonitor) {
            onRefresh(wac);
        }
    }

    //到这里只需要设置属性即可
    if (this.publishContext) {
        // Publish the context as a servlet context attribute.
        String attrName = getServletContextAttributeName();
        getServletContext().setAttribute(attrName, wac);
    }

    return wac;
}
```

##### DispatcherServlet#onRefresh

本质是初始化各种解析器

```java
 
	/**
	 * This implementation calls {@link #initStrategies}.
	 */
	@Override
	protected void onRefresh(ApplicationContext context) {
		initStrategies(context);
	}
 
	/**
	 * Initialize the strategy objects that this servlet uses.
	 * <p>May be overridden in subclasses in order to initialize further strategy objects.
	 */
	protected void initStrategies(ApplicationContext context) {
		initMultipartResolver(context); //初始化文件上传解析器
		initLocaleResolver(context); //初始化本地域解析器
		initThemeResolver(context); 
		initHandlerMappings(context); //初始化处理器映射
		initHandlerAdapters(context);
		initHandlerExceptionResolvers(context); //异常解析器
		initRequestToViewNameTranslator(context); 
		initViewResolvers(context); //视图解析器
		initFlashMapManager(context);
	}
```

#### DispatchServlet 处理流程

DispatcherServlet 类上可以看到很多熟悉的成员变量（**组件**），所以看下来，我们要的东西，DispatcherServlet 可**全都有**。

```java
// 文件处理器
private MultipartResolver multipartResolver;

// 映射器
private List<HandlerMapping> handlerMappings;

// 适配器
private List<HandlerAdapter> handlerAdapters;

// 异常处理器
private List<HandlerExceptionResolver> handlerExceptionResolvers;

// 视图解析器
private List<ViewResolver> viewResolvers;
```

它们在`initStrategies()`上初始化。

请求进到 DispatcherServlet，其实全部都会打到 `doService()` 方法上。我们看看这个`doService()`方法做了啥：

```java
protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {

    // 设置一些上下文...(省略一大部分)
    request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
    request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);

    try {
        // 调用doDispatch
        doDispatch(request, response);
    }
    finally {
        if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
            if (attributesSnapshot != null) {
                restoreAttributesAfterInclude(request, attributesSnapshot);
            }
        }
    }
}
```

所以请求会走到 `doDispatch(request, response);` 里边，我们再进去看看：

```java
@SuppressWarnings("deprecation")
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    boolean multipartRequestParsed = false;

    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

    try {
        ModelAndView mv = null;
        Exception dispatchException = null;

        try {
            // 检查是不是文件上传 multipart 请求
            processedRequest = checkMultipart(request);
            multipartRequestParsed = (processedRequest != request);

            // Determine handler for the current request. 获取处理器映射器，找到HandlerExecutionChain
            mappedHandler = getHandler(processedRequest);
            if (mappedHandler == null) {
                //没有找到映射器处理器 则报404 not found
                noHandlerFound(processedRequest, response);
                return;
            }

            // Determine handler adapter for the current request. 获取处理器适配器，通常是RequestMappingHandlerAdapter
            HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

            // Process last-modified header, if supported by the handler.
            String method = request.getMethod();
            boolean isGet = HttpMethod.GET.matches(method);
            if (isGet || HttpMethod.HEAD.matches(method)) { //判断是否为GET、HEAD请求
                long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
                if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
                    return;
                }
            }
            //拦截前置处理，preHandle方法
            if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                return;
            }

            // Actually invoke the handler.
            // 处理器适配器，执行真正处理方法，在内部创建 ModelAndView 对象
            // 可能调用到 AbstractHandlerMethodAdapter 中的 handle 方法
            mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

            if (asyncManager.isConcurrentHandlingStarted()) {
                return;
            }
            
            // 设置ModelAndView 中视图名称
            applyDefaultViewName(processedRequest, mv);
            
            // 拦截器后置处理，postHandle方法
            mappedHandler.applyPostHandle(processedRequest, response, mv);
        }
        catch (Exception ex) {
            dispatchException = ex;
        }
        catch (Throwable err) {
            // As of 4.3, we're processing Errors thrown from handler methods as well,
            // making them available for @ExceptionHandler methods and other scenarios.
            dispatchException = new NestedServletException("Handler dispatch failed", err);
        }
        //渲染视图
        processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
    }
    catch (Exception ex) {
        triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
    }
    catch (Throwable err) {
        triggerAfterCompletion(processedRequest, response, mappedHandler,
                               new NestedServletException("Handler processing failed", err));
    }
    finally {
        if (asyncManager.isConcurrentHandlingStarted()) {
            // Instead of postHandle and afterCompletion
            if (mappedHandler != null) {
                mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
            }
        }
        else {
            // Clean up any resources used by a multipart request.
            if (multipartRequestParsed) {
                cleanupMultipart(processedRequest);
            }
        }
    }
}
```

从源码可以知道的是，原来 SpringMVC 的拦截器是在 MappingHandler 的时候一齐返回的，返回的是一个`HandlerExecutionChain` 对象：

```java
public class HandlerExecutionChain {

    private static final Log logger = LogFactory.getLog(HandlerExecutionChain.class);

    // 真实的handler
    private final Object handler;

    // 拦截器List
    private HandlerInterceptor[] interceptors;
    private List<HandlerInterceptor> interceptorList;

    private int interceptorIndex = -1;
}
```

OK，整体的流程我们是已经看完了。

> 更详细流程参考：https://mp.weixin.qq.com/s/BO_CPQ0x-kBMIYBOviG3Xg

流程图如下：

![图片](https://raw.githubusercontent.com/Traserve/traserve.github.io/master/_posts/blog/Spring%20MVC/images/DispatcherServlet-process.png){: .align-center}