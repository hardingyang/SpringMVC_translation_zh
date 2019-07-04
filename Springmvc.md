##建立在Servlet上的Web ##
版本5.1.8RELEASE

------------

这部分的文档包含了对建立在原生Servlet API和部署在Servlet容器上的Servlet-stack web应用的支持。每个章节都是独立的，包括了SpringMVC、View Technologies、CORS Support、and WebSocket Support。

##1.Spring Web MVC
SpringMVC是原生的建立在Servlet API上的Web框架，并且在很早以前就被纳入了Spring框架中。最正式的名字是“Spring Web MVC”，名字来源于它的资源模块名：“Spring-webmvc”，但是更通俗的名字是"Spring MVC"。

Spring框架5.0版本中引用了名字叫：“Spring WebFlux”的reactive-stack web框架，它名字的由来与Spring Web MVC很很相似，都是因为它的资源模块名字(Spring-webflux)。
####1.1 DispatcherServlet
Spring MVC与其他web框架一样，都是围绕着Servlet前端控制器模式而设计的。DispatcherServlet类对了Request请求过程提供了一个共享式算法，然而通用的工作形式是通过配置文件。这种模式是固定的，并且只吃各种工作环境。
作为一个Servlet，DispatcherServlet类也需要被配置成映射对象，要么使用Java在Servlet上配置，要么配置在Web.xml文件中。然而如果DispatcherServlet要使用Spring配置文件去代理Request请求，则需要Requet请求映射，视图解析等技术才可

下面的例子演示了Java去配置DispatcherServlet，并且它是自动被Servlet容器启用的。

    public class MyWebApplicationInitializer implements WebApplicationInitializer {
    	@Override
    	public void onStartup(ServletContext servletCxt) {
    
    // Load Spring web application configuration
    AnnotationConfigWebApplicationContext ac = new AnnotationConfigWebApplicationContext();
    ac.register(AppConfig.class);
    ac.refresh();
    
    // Create and register the DispatcherServlet
    DispatcherServlet servlet = new DispatcherServlet(ac);
    ServletRegistration.Dynamic registration = servletCxt.addServlet("app", servlet);
    registration.setLoadOnStartup(1);
    registration.addMapping("/app/*");
    }
    }
    

下面的例子演示了用Web.xml文件去配置DispatcherServlet类

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
    

####1.1.1 上下文体系
DispatcherServlet类包含了一个WebApplicationContext类(继承了原生的ApplicationContext类)作为它自己的配置属性。WebApplicationContext类与ServletContext类和Servlet类是有联系的。WebApplicationContext类也有一个ServletContext类，这样应用就可以使用静态方法RequestContextUtils()去查看WebApplicationContext类是否需要去接收它。

对于很多应用，只有唯一一个简单强大WebApplicationContext配置，对于SpringMVC的上下文体系，它有一个根WebApplicationContext配置，并且通过DispatcherServlet前端控制器(或者其他Servlet)对象，每一个对象它可以有自己的子WebApplicationContext配置。

根WebApplicationContext也包含了bean架构，比如可以通过各种Servlet对象分享数据存储、事务等。这些bean是可以在子WebApplicationContext配置下继承和重写的。下面这个图展示了他们之间的相互关系：

![](https://docs.spring.io/spring/docs/current/spring-framework-reference/images/mvc-context-hierarchy.png)

[下面我仅展示了web.xml里面的配置]
   
	 <web-app>
	    <listener>
	    	<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
	    </listener>
	    
	    <context-param>
	    	<param-name>contextConfigLocation</param-name>
	    	<param-value>/WEB-INF/root-context.xml</param-value>
			<!--这里对应的就是上下文根目录-->
	    </context-param>
	    
	    <servlet>
		    <servlet-name>app1</servlet-name>
		    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		    <init-param>
		    <param-name>contextConfigLocation</param-name>
		    <param-value>/WEB-INF/app1-context.xml</param-value>
			<!--这里对应子上下文目录，可以重写和继承根上下文目录-->
		    </init-param>
		    <load-on-startup>1</load-on-startup>
	    </servlet>
	    
	    <servlet-mapping>
	    	<servlet-name>app1</servlet-name>
			<!--过滤所有发生过来的请求url-->
	    	<url-pattern>/app1/*</url-pattern>
	    </servlet-mapping>
    
    </web-app>

####1.1.2 特定的Bean类型

DispatcherServlet前端控制器委托特定的Bean去处理Request请求，并且渲染响应。特定的Bean，意思是实现了Spring框架的受Spring管理的Object对象。这些Bean一般都是建立在Spring上，但是你也可以自己定制这种Bean的属性，通过继承或者替换他们。

| Bean类型        | 描述         
|------------- |------------
| HandlerMapping      | 映射一个Request请求一般都带有一个前置和后置拦截器。这种映射是基于一些标准，详细信息你可以查看HandlerMapping接口  
| HandlerAdapter      | 协助DispatcherServlet前端控制器调用对于Request请求的控制器映射，它不会管理处理器是怎么调用的。例如：调用一个注解控制器请求解析注解。这主要的目的就是保护DispatcherServlet前端控制器。   
| HandlerExceptionResolver | 当出现异常时，可能会映射它给处理器，再渲染给Html一个error页面或者其他自己配置的页面及信息。
|ViewResolver、LocaleContextResolver|解析Locale，给客户端一个他们正在的时区。为了提供一个国际化视图。
|ThemeResolver|解析Web应用可以使用的主题——比如提供私有化布局。
|MultipartResolver|解析其他类型的Request请求(比如从客户端上传一个文件)
|FlashMapManager|储存和释放“input”和“output”，FlashMap通过重定向使用属性从一个Request请求到另外一个Request请求。


####1.1.3 Web MVC配置