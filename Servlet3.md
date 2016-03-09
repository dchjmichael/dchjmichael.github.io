#Servlet 3无web.xml项目配置

## 1 创建项目
![创建项目](http://i.imgur.com/wa4gaSJ.jpg)

选择servlet版本至少3.0

![servlet版本](http://i.imgur.com/Se5XaOD.jpg)

创建完成后查看WEB-INF目录，发行没有web.xml(我选择的是3.1)

![](http://i.imgur.com/ykfrmwF.png)

编写Servlet

```java
@WebServlet(urlPatterns="/hello")
public class HelloServlet extends HttpServlet {

	@Override
	protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
		resp.getWriter().print("hello");
	}
	
}
```

HelloServlet的上面有个WebServlet注解，表明这是一个servlet。

@WebServlet有很多的属性：

asyncSupported：声明Servlet是否支持异步操作模式。

description：　　  Servlet的描述。

displayName：     Servlet的显示名称。

initParams：        Servlet的init参数。

name：　　　　    Servlet的名称。

urlPatterns：　　  Servlet的访问URL。

value：　　　       Servlet的访问URL。

然后部署到tomcat(tomcat7以上)，访问：http://localhost:8080/servlet3_demo/hello