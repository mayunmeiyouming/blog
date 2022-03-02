---
layout: post
title:  "SpringMVC"
date:   2019-09-20 11:00:00 +0800
categories: [Tech]
tag: 
  - Springmvc
  - Java
---

### 入门

web.xml

```xml
<!DOCTYPE web-app PUBLIC
 "-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"
 "http://java.sun.com/dtd/web-app_2_3.dtd" >

<web-app>
  <display-name>Archetype Created Web Application</display-name>
  
  <servlet>
    <servlet-name>DispatcherServlet</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <load-on-startup>1</load-on-startup>
  </servlet>
  <servlet-mapping>
    <servlet-name>DispatcherServlet</servlet-name>
    <url-pattern>*.do</url-pattern>
  </servlet-mapping>
</web-app>
```

DispatcherServlet-servlet.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:mvc="http://www.springframework.org/schema/mvc"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:aop="http://www.springframework.org/schema/aop" xmlns:tx="http://www.springframework.org/schema/tx"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans-3.2.xsd
        http://www.springframework.org/schema/mvc
        http://www.springframework.org/schema/mvc/spring-mvc-3.2.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context-3.2.xsd
        http://www.springframework.org/schema/aop
        http://www.springframework.org/schema/aop/spring-aop-3.2.xsd
        http://www.springframework.org/schema/tx
        http://www.springframework.org/schema/tx/spring-tx-3.2.xsd">


    <!-- 配置url映射 -->
    <bean class="org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping"></bean>

    <!-- 配置控制器处理适配器 -->
    <bean class="org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter"></bean>

    <!-- 配置控制器,相当于配置了访问路径 -->
    <bean name="/user.do" class="Controller.UserController" ></bean>

    <!-- 资源解析器 -->
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="prefix" value="/WEB-INF/views/"></property>
        <property name="suffix" value=".jsp"></property>
    </bean>

</beans>
```

UserController

```java
package Controller;

import org.springframework.web.servlet.ModelAndView;
import org.springframework.web.servlet.mvc.Controller;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

public class UserController implements Controller {
    @Override
    public ModelAndView handleRequest(HttpServletRequest httpServletRequest,
                                      HttpServletResponse httpServletResponse) throws Exception {
        ModelAndView modelAndView = new ModelAndView("userlist");
        modelAndView.addObject("name", "黄伟");
        return modelAndView;
    }
}
```

### url映射

#### BeanNameUrlHandlerMapping

```xml
<bean class="org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping"></bean>

<bean name="/user.do" class="Controller.UserController" ></bean>
```

#### SimpleUrlHandlerMapping

```xml
<bean class="org.springframework.web.servlet.handler.SimpleUrlHandlerMapping">
    <property name="mappings">
        <props>
            <prop key="/user1.do">userController</prop>
            </props>
    </property>
</bean>

<bean id="userController" class="Controller.UserController" ></bean>
```

### HandlerAdapter

#### SimpleControllerHandlerAdapter

控制器调用handleRequest方法,返回ModelAndView

```xml
<bean class="org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter"></bean>
```

控制器实现:

```java
package Controller;

import org.springframework.web.servlet.ModelAndView;
import org.springframework.web.servlet.mvc.Controller;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

public class UserController implements Controller {
    @Override
    public ModelAndView handleRequest(HttpServletRequest httpServletRequest,
                                      HttpServletResponse httpServletResponse) throws Exception {
        ModelAndView modelAndView = new ModelAndView("userlist");
        modelAndView.addObject("name", "黄伟");
        return modelAndView;
    }
}
```

#### HttpRequestHandlerAdapter

```xml
<bean class="org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter"></bean>
```

类:

```java
public class HttpController implements HttpRequestHandler {
    @Override
    public void handleRequest(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        request.setAttribute("name", "黄伟");
        request.getRequestDispatcher("/WEB-INF/views/userlist.jsp").forward(request,response);
    }
}
```

### 注解

xml配置:

```xml
<!-- 配置控制器 -->
<!-- 扫描包 -->
<context:component-scan base-package="Controller"></context:component-scan>

<!-- 处理映射 -->
<bean class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping"></bean>

<!-- 配置适配器 -->
<bean class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter"></bean>

<!-- 资源解析器 -->
<bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
    <property name="prefix" value="/WEB-INF/views/"></property>
    <property name="suffix" value=".jsp"></property>
</bean>
```

Controller

```java
@org.springframework.stereotype.Controller
@RequestMapping("/user")
public class UserController {

    @RequestMapping("/list")
    public String list() {
        return "userlist";
    }
}
```

### spring接收表单数据

1.

```java
@RequestMapping("/register")
public String register(String username, String password) {
    System.out.println(username);
    System.out.println(password);
    return "userlist";
}
```

2. 模型

```java
@RequestMapping("/register2")
public String register(User user) {
    System.out.println(user.getUsername());
    System.out.println(user.getPassword());
    return "userlist";
}
```

3.

```java
public class UserExt {

    private User user;

    @Override
    public String toString() {
        return "UserExt{" +
                "user=" + user +
                '}';
    }

    public User getUser() {
        return user;
    }

    public void setUser(User user) {
        this.user = user;
    }
}
```

```html
<form action="/springmvc/user/register2.do">
    用户名: <input type="text" name="user.username"><br>
    密码: <input type="password" name="user.password"><br>
    <button type="submit" name="提交"></button><br>
</form>
```

```java
@RequestMapping("/register2")
public String register(UserExt userExt) {
    System.out.println(userExt);
    return "userlist";
}
```

### 数据回显

使用Model

### url

```java
@RequestMapping("/list/{id}")
public String list(@PathVariable int id, Model model) {
    model.addAttribute("username", "黄伟");
    return "userlist";
}
```

### 转发和重定向

```java
return "forward:userlist.do";
```

```java
return "redirect:userlist.do";
```

### json数据

spring5.0.8

jackson包

```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-annotations</artifactId>
    <version>2.9.5</version>
</dependency>
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-core</artifactId>
    <version>2.9.5</version>
</dependency>
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
      <version>2.9.5</version>
</dependency>
```

```java
@RequestMapping("/save")
@ResponseBody
public User save(@RequestBody User user) {
    System.out.println(user);
    user.setPassword("黄伟");
    return user;
}
```

```xml
<mvc:annotation-driven />
<!-- 配置适配器 -->
<bean class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter">
    <property name="messageConverters">
        <list>
            <bean class="org.springframework.http.converter.json.MappingJackson2HttpMessageConverter"/>
            <bean class="org.springframework.http.converter.ByteArrayHttpMessageConverter"/>
            <bean class="org.springframework.http.converter.xml.SourceHttpMessageConverter"/>
            <bean class="org.springframework.http.converter.FormHttpMessageConverter"/>
            <bean class="org.springframework.http.converter.StringHttpMessageConverter"/>
        </list>
    </property>
</bean>
```

### spring多视图

url后面加.json或.xml

```xml
<!-- 多视图 -->
    <bean class="org.springframework.web.servlet.view.ContentNegotiatingViewResolver">
        <!-- 配置支持类型 -->
        <property name="contentNegotiationManager">
            <bean class="org.springframework.web.accept.ContentNegotiationManagerFactoryBean">
                <property name="mediaTypes">
                    <map>
                        <entry key="json" value="application/json"></entry>
                        <entry key="xml" value="application/xml"></entry>
                    </map>
                </property>
            </bean>
        </property>

        <!-- 配置默认视图 -->
        <property name="defaultViews">
            <list>
                <!-- json -->
                <bean class="org.springframework.web.servlet.view.json.MappingJackson2JsonView"></bean>

                <!-- xml -->
                <bean class="org.springframework.web.servlet.view.xml.MarshallingView">
                    <constructor-arg>
                        <bean class="org.springframework.oxm.jaxb.Jaxb2Marshaller">
                            <property name="classesToBeBound">
                                <list>
                                    <value>model.User</value>
                                </list>
                            </property>
                        </bean>
                    </constructor-arg>
                </bean>
            </list>
        </property>
    </bean>
```
