# 基于Servlet3的无配置SpringMVC

先介绍几个接口

## `org.springframework.web.WebApplicationInitializer`

> Interface to be implemented in Servlet 3.0+ environments in order to configure the ServletContext programmatically -- as opposed to (or possibly in conjunction with) the traditional web.xml-based approach. 
> 
> Implementations of this SPI will be detected automatically by SpringServletContainerInitializer, which itself is bootstrapped automatically by any Servlet 3.0 container. See its Javadoc for details on this bootstrapping mechanism. 

举个栗子，传统的基于XML的配置方式：
```xml
 <servlet>
   <servlet-name>dispatcher</servlet-name>
   <servlet-class>
     org.springframework.web.servlet.DispatcherServlet
   </servlet-class>
   <init-param>
     <param-name>contextConfigLocation</param-name>
     <param-value>/WEB-INF/spring/dispatcher-config.xml</param-value>
   </init-param>
   <load-on-startup>1</load-on-startup>
 </servlet>

 <servlet-mapping>
   <servlet-name>dispatcher</servlet-name>
   <url-pattern>/</url-pattern>
 </servlet-mapping>
```
用WebApplicationInitializer代码方式：

```java
 public class MyWebAppInitializer implements WebApplicationInitializer {

    @Override
    public void onStartup(ServletContext container) {
      XmlWebApplicationContext appContext = new XmlWebApplicationContext();
      appContext.setConfigLocation("/WEB-INF/spring/dispatcher-config.xml");

      ServletRegistration.Dynamic dispatcher =
        container.addServlet("dispatcher", new DispatcherServlet(appContext));
      dispatcher.setLoadOnStartup(1);
      dispatcher.addMapping("/");
    }

 }
```
## javax.servlet.ServletContainerInitializer

这个接口只有一个方法：

```java
  public void onStartup(Set<Class<?>> c, ServletContext ctx) throws ServletException; 
```


这个接口会在web应用启动阶段被通知，以进行任何必要的servlet，filter，listener注册。

这个接口的实现类可以添加`@HandlesTypes`，用来在onStartup方法中接收`@HandlesTypes`注解中标注接口的实现类集合。

如果`ServletContainerInitializer`接口的实现类没有使用`@HandlesTypes`注解，或者没有`@HandlesTypes`所指定接口的实现类，那么容器会传一个null集合给onStartup方法。

这个接口的实现类必须在jar包的META-INF/services目录中进行声明，方法是在该目录下创建一个名为javax.servlet.ServletContainerInitializer的文件，内容是这个接口的实现类全路径名(Qualified Name),这其实就是典型的SPI模式，容器启动的时候会自动查询。

##org.springframework.web.SpringServletContainerInitializer

```java
@HandlesTypes(WebApplicationInitializer.class)
public class SpringServletContainerInitializer implements ServletContainerInitializer{
	@Override
	public void onStartup(Set<Class<?>> webAppInitializerClasses, ServletContext servletContext)
			throws ServletException {
		//......
	}

}
```

按照之前`ServletContainerInitializer`的文档说明，`SpringServletContainerInitializer`会在容器启动的时候调用onStartup方法，由于`SpringServletContainerInitializer`标注了`@HandlesTypes(WebApplicationInitializer.class)`，所以`Set<Class<?>> webAppInitializerClasses`里面传入的都是实现了`WebApplicationInitializer`接口的类。

再看看META-INF下面的 SPI 声明

![](http://i.imgur.com/wWs0O9k.png)

打开这个文件，里面就一行：

    org.springframework.web.SpringServletContainerInitializer

跟规范约定的一致。

onStartup方法内容：

```java
@Override
	public void onStartup(Set<Class<?>> webAppInitializerClasses, ServletContext servletContext)
			throws ServletException {

		List<WebApplicationInitializer> initializers = new LinkedList<WebApplicationInitializer>();

		if (webAppInitializerClasses != null) {
			for (Class<?> waiClass : webAppInitializerClasses) {
				// Be defensive: Some servlet containers provide us with invalid classes,
				// no matter what @HandlesTypes says...
				if (!waiClass.isInterface() && !Modifier.isAbstract(waiClass.getModifiers()) &&
						WebApplicationInitializer.class.isAssignableFrom(waiClass)) {
					try {
						initializers.add((WebApplicationInitializer) waiClass.newInstance());
					}
					catch (Throwable ex) {
						throw new ServletException("Failed to instantiate WebApplicationInitializer class", ex);
					}
				}
			}
		}

		if (initializers.isEmpty()) {
			servletContext.log("No Spring WebApplicationInitializer types detected on classpath");
			return;
		}

		AnnotationAwareOrderComparator.sort(initializers);
		servletContext.log("Spring WebApplicationInitializers detected on classpath: " + initializers);

		for (WebApplicationInitializer initializer : initializers) {
			initializer.onStartup(servletContext);
		}
	}
```

注释中解释了这段防御性代码的原因，有些容器会提供一些无效的类，不管`@HandlesTypes`里写些啥。

这段代码就是把Set中的所有class遍历下，把`WebApplicationInitializer`类的有效实现类放到list中，最后依此调用每个`WebApplicationInitializer`实例的onStartup方法。


##实战下

###1 定义配置类
```java
@Configuration
@ComponentScan(basePackages = "spring3.web.demo")
public class DefaultAppConfig {
}
```
###2 定义WebApplicationInitializer

```java
public class DefaultWebApplicationInitializer implements WebApplicationInitializer {

	@Override
	public void onStartup(ServletContext appContext) throws ServletException {
		AnnotationConfigWebApplicationContext rootContext = new AnnotationConfigWebApplicationContext();
		rootContext.register(DefaultAppConfig.class);

		appContext.addListener(new ContextLoaderListener(rootContext));

		ServletRegistration.Dynamic dispatcher = appContext.addServlet("dispatcher", new DispatcherServlet(rootContext));
		dispatcher.setLoadOnStartup(1);
		dispatcher.addMapping("/");

	}
}
```

###3 控制器
```java
@Controller
public class HelloController {

	@RequestMapping("/hello.do")
	public void test(HttpServletResponse response) throws IOException {
		response.getWriter().write("Hello,world");
	}
}
```

###4 部署到tomcat

Ok，就这点东西，没有xml配置文件，没有web.xml，部署后，访问：http://localhost:8180/web-demo/hello.do



