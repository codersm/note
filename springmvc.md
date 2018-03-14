---
title: SpringMVC异常处理
date: 2016-08-16 22:17:03
tags: SpringMVC
---
在web项目中异常处理是非常重要的，确保服务端错误信息不会直接在客户端展现。SpringMVC提供了@ExceptionHandler, @ControllerAdvice和 HandlerExceptionResolver三种方式处理异常。
<!-- more -->

### 1、SpringMVC框架提供如下3种方式处理异常：  

- **Controller Based**

We can define exception handler methods in our controller classes. All we need is to annotate these methods with @ExceptionHandler annotation. This annotation takes Exception class as argument. So if we have defined one of these for Exception class, then all the exceptions thrown by our request handler method will have handled.
These exception handler methods are just like other request handler methods and we can build error response and respond with different error page. We can also send JSON error response。

If there are multiple exception handler methods defined, then handler method that is closest to the Exception class is used. For example, if we have two handler methods defined for IOException and Exception and our request handler method throws IOException, then handler method for IOException will get executed.

- **Global Exception Handler**

Exception Handling is a cross-cutting concern, it should be done for all the pointcuts in our application. We have already looked into Spring AOP and that’s why Spring provides @ControllerAdvice annotation that we can use with any class to define our global exception handler.
The handler methods in Global Controller Advice is same as Controller based exception handler methods and used when controller class is not able to handle the exception.

- **HandlerExceptionResolver**

For generic exceptions, most of the times we serve static pages. Spring Framework provides HandlerExceptionResolver interface that we can implement to create global exception handler. The reason behind this additional way to define global exception handler is that Spring framework also provides default implementation classes that we can define in our spring bean configuration file to get spring framework exception handling benefits.

SimpleMappingExceptionResolver is the default implementation class, it allows us to configure exceptionMappings where we can specify which resource to use for a particular exception. We can also override it to create our own global handler with our application specific changes, such as logging of exception messages.

### 2、通过HandlerExceptionResolver实现全局异常处理

```java
public class CustomerGlobalException extends SimpleMappingExceptionResolver {
    @Override
    protected ModelAndView doResolveException(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {

        HandlerMethod handlerMethod = (HandlerMethod) handler;
        Class clazz = handlerMethod.getMethod().getReturnType();
        if (clazz.equals(String.class) || clazz.equals(ModelAndView.class)) {
            return super.doResolveException(request, response, handler, ex);
        }else{
            try {
                response.setCharacterEncoding("UTF-8");
                response.setContentType("application/json; charset=utf-8");
                PrintWriter pw = response.getWriter();
                pw.write("{\"code\":-1,\"message\":\"系统出错\"}");
                pw.flush();
                pw.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
            return null;
        }
    }
}