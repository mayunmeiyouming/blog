---
layout: post
title:  "JSP,Servlet,EL,JSTl"
date:   2019-07-26 16:01:01 +0800
categories: [Tech]
tag: 
  - Jsp
  - Web
---

## JSP

注意: 使用jsp时,需要导入`servlet-api.jar`包.使用EL表达式时,需要导入`jsp-api.jar`包

### JSP指令

page: `<%@ page contentType="text/html;charset=utf8" %>`

include: `<%@include file="sourcePath"%>`

taglib: `<%@ taglib prefix="c" uri=http://java.sun.com/jsp/jstl/core %>`

### JSP注释

```html
<%--

--%>
```

### JSP内置对象

JSP内置对象和servlet对像的关系:


```text
 jsp内置对象                servlet

pageContext             PageContext
request                 HttpServletRequest/ServletRequest
session                 HttpSession
application             ServletContext
response                HttpServletRespons/ServletResponse
out                     JspWriter
page                    this
config                  ServletConfig
exception               Throwable
```

## Servlet

Servlet创建时间

在`<servlet>`标签下

`<load-on-startup><\load-on-startup>`

当中间的值为正整数或0时,是服务器启动是创建.当中间的值为负数时,是第一次访问时创建

## EL表达式

语法: `${expression}`

忽略EL表达式

`isELIgnored="true"` 可以忽略所有EL表达式

在语法前面加`\`可以忽略单个EL表达式

注意: EL表达式只能从域对象中获取值

### empty运算符

`${empty 名}`

### 域对象

```text
pageScope              pageContext
requestScope           request
sessionScope           session
applicationScope       application(ServletContext)
```

如果采用`${name}`语法,其中忽略键名,EL会从最小的域开始寻找值.分别是pageScope,requestScope,sessionScope,applicationScope

### 获取对象的属性(调用对象中的getter方法)

`${域名.对象.方法名去掉get并且首字母小写}`

### 获取list的值

`${域名.键名[索引]}`

### 获取map的值

`${域名.键名.key名}`

`${域名.键名[key名]}`

### 隐式对象

pageContext

## JSTL标签

`<%@ taglib prefix="c" uri=http://java.sun.com/jsp/jstl/core %>`

### if

必需的属性test,接收boolean表达式,值为true才会显示,没有else语句

```java
<c:if test="true">
我是真
</c:if>
```

### choose

```java
<c:choose>
    <c:when test=""></c:when>
    <c:when test=""></c:when>
    <c:otherwise></c:otherwise>
</c:choose>
```

### forEach

普通for循环:

```java
<c:forEach begin="1" end="10" var="i" step="1">
${i}<br>
</c:forEach>
```

`varStatus`的`index`,`count`

`index`是容器元素的索引,从0开始

`count`代表循环次数,从1开始

遍历容器:

items: 容器对象

var: 容器元素的临时变量

```java
<c:forEach items="${list}" var="str" varStatus="s">
${s.index} ${s.count} ${str}
</c:forEach>
```
