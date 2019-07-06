# Spring Boot参考指南

**2.1.6.RELEASE**

#### 29.1.11 错误处理

默认情况下，Spring Boot提供==/error==映射，以合理的方式处理所有错误，并在servlet容器中注册为“全局(global)”错误页面。对于移动端(非浏览器客户端)，它会生成一个JSON响应，其中包含错误，HTTP状态和异常消息的详细信息。对于浏览器客户端，有一个“whitelabel”错误视图，以HTML格式呈现相同的数据（要自定义它，添加一个解析为==error==的==View==）。要完全替换默认行为，您可以实现==ErrorController==接口并注册该类型的bean定义，或者添加==ErrorAttributes==类型的bean来使用现有机制但替换内容。

**==BasicErrorController==可以用作自定义==ErrorController==的基类。如果你想要为新内容类型添加处理程序，则此功能特别有用（默认是专门处理==text / html==并为其他所有内容提供后备）。需要做，首先，继承==BasicErrorController==，添加一个具有==produce==属性的==@RequestMapping==注解的公共方法，并创建你的新类型的bean。**

您还可以定义一个使用==@ControllerAdvice==注解的类，以自定义JSON文档以返回特定控制器 和/或 异常类型，如以下示例所示：

```java
@ControllerAdvice(basePackageClasses = AcmeController.class)
public class AcmeControllerAdvice extends ResponseEntityExceptionHandler {

	@ExceptionHandler(YourException.class)
	@ResponseBody
	ResponseEntity<?> handleControllerException(HttpServletRequest request, Throwable ex) {
		HttpStatus status = getStatus(request);
		return new ResponseEntity<>(new CustomErrorType(status.value(), ex.getMessage()), status);
	}

	private HttpStatus getStatus(HttpServletRequest request) {
		Integer statusCode = (Integer) request.getAttribute("javax.servlet.error.status_code");
		if (statusCode == null) {
			return HttpStatus.INTERNAL_SERVER_ERROR;
		}
		return HttpStatus.valueOf(statusCode);
	}

}
```

在前面的示例中，如果在与==AcmeController==相同的包中定义的控制器抛出==YourException==，则使用==CustomErrorType== POJO的JSON表示来替代==ErrorAttributes==表示。



##### 自定义错误页面

如果要显示给定状态代码的自定义HTML错误页面，可以将文件添加到==/ error==文件夹。错误页面可以是静态HTML（即，添加到任何静态资源文件夹下），也可以使用模板构建。文件名应该是确切的状态代码或系列掩码。

例如，要将404映射到静态HTML文件，您的文件夹结构将如下所示：

~~~ 
src/
 +- main/
     +- java/
     |   + <source code>
     +- resources/
         +- public/
             +- error/
             |   +- 404.html
             +- <other public assets>
~~~

要使用FreeMarker模板映射所有5xx错误，您的文件夹结构如下：

~~~
src/
 +- main/
     +- java/
     |   + <source code>
     +- resources/
         +- templates/
             +- error/
             |   +- 5xx.ftl
             +- <other templates>
~~~

对于更复杂的映射，您还可以添加实现ErrorViewResolver接口的bean，如以下示例所示：

~~~java
public class MyErrorViewResolver implements ErrorViewResolver {

	@Override
	public ModelAndView resolveErrorView(HttpServletRequest request,
			HttpStatus status, Map<String, Object> model) {
		// 使用请求或状态可选择返回ModelAndView
		return ...
	}

}
~~~

您还可以使用常规的Spring MVC功能，例如==@ExceptionHandler==方法和==@ControllerAdvice==。然后，==ErrorController==将获取任何未处理的异常。



##### 映射Spring MVC之外的错误页面

对于不使用Spring MVC的应用程序，您可以使用==ErrorPageRegistrar==接口直接注册==ErrorPages==。这种抽象方法直接与底层嵌入式servlet容器一起工作，即使你没有Spring MVC DispatcherServlet也可以工作。

~~~java
@Bean
public ErrorPageRegistrar errorPageRegistrar(){
	return new MyErrorPageRegistrar();
}

// ...

private static class MyErrorPageRegistrar implements ErrorPageRegistrar {

	@Override
	public void registerErrorPages(ErrorPageRegistry registry) {
		registry.addErrorPages(new ErrorPage(HttpStatus.BAD_REQUEST, "/400"));
	}

}
~~~

**如果您注册一个==ErrorPage==，其路径最终由==Filter==处理（这与一些非Spring Web框架（如Jersey和Wicket）一样），那么==Filter==必须显式注册为==ERROR==调度程序，如以下示例：**

~~~java
@Bean
public FilterRegistrationBean myFilter() {
	FilterRegistrationBean registration = new FilterRegistrationBean();
	registration.setFilter(new MyFilter());
	...
	registration.setDispatcherTypes(EnumSet.allOf(DispatcherType.class));
	return registration;
}
~~~

请注意，默认的==FilterRegistrationBean==不包含==ERROR==调度程序类型。

注意：当部署到servlet容器时，Spring Boot使用其错误页面过滤器将具有错误状态的请求转发到相应的错误页面。如果尚未提交响应，则只能将请求转发到正确的错误页面。默认情况下，WebSphere Application Server 8.0及更高版本在成功完成servlet的服务方法后提交响应。您应该通过将==com.ibm.ws.webcontainer.invokeFlushAfterService==设置为==false==来禁用此行为。

