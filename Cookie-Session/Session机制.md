# `Session`机制

## 1. `Session`解决的问题

除了使用`Cookie`，`Web`应用程序中还经常使用`Session`来记录客户端状态。`Session`是服务器端使用的一种记录客户端状态的机制，使用上比`Cookie`简单一些，相应的也增加了服务器的存储压力。

`Session`技术则是服务端的解决方案，它是通过服务器来保持状态的。

由于`Session`这个词汇包含的语义很多，因此需要在这里明确一下`Session`的含义。
- 我们通常都会把`Session`翻译成会话，因此我们可以把客户端浏览器与服务器之间一系列交互的动作称为一个`Session`。从这个语义出发，我们会提到`Session`持续的时间，会提到在`Session`过程中进行了什么操作等等
- `Session`指的是服务器端为客户端所开辟的存储空间，在其中保存的信息就是用于保持状态。从这个语义出发，我们则会提到往`Session`中存放什么内容，如何根据键值从`Session`中获取匹配的内容等。

要使用`Session`，第一步当然是创建`Session`了。
那么`Session`在何时创建呢？当然还是在服务器端程序运行的过程中创建的，不同语言实现的应用程序有不同创建`Session`的方法，而在`Java`中是通过调用`HttpServletRequest`的`getSession`方法（使用`true`作为参数）创建的。在创建了`Session`的同时，服务器会为该`Session`生成唯一的`Session id`，而这个`Session id`在随后的请求中会被用来重新获得已经创建的`Session`；在`Session`被创建之后，就可以调用`Session`相关的方法往`Session`中增加内容了，而这些内容只会保存在服务器中，发到客户端的只有`Session id`；当客户端再次发送请求的时候，会将这个`Session id`带上，服务器接受到请求之后就会依据`Session id`找到相应的`Session`，从而再次使用之。正式这样一个过程，用户的状态也就得以保持了。

## 2. `Session`的定义

`Session`是另一种记录客户状态的机制，不同的是`Cookie`保存在客户端浏览器中，而`Session`保存在服务器上。客户端浏览器访问服务器的时候，服务器把客户端信息以某种形式记录在服务器上。这就是`Session`。客户端浏览器再次访问时只需要从该`Session`中查找该客户的状态就可以了。

如果说`Cookie`机制是通过检查客户身上的“通行证”来确定客户身份的话，那么`Session`机制就是通过检查服务器上的“客户明细表”来确认客户身份。`Session`相当于程序在服务器上建立的一份客户档案，客户来访的时候只需要查询客户档案表就可以了。

## 3. 实现用户登录

`Session`对应的类为`javax.servlet.http.HttpSession`类。每个来访者对应一个`Session`对象，所有该客户的状态信息都保存在这个`Session`对象里。`Session`对象是在客户端第一次请求服务器的时候创建的。`Session`也是一种`key-value`的属性对，通过`getAttribute(Stringkey)`和`setAttribute(String key，Objectvalue)`方法读写客户状态信息。`Servlet`里通过`request.getSession()`方法获取该客户的`Session`，例如：

```Java
HttpSession session = request.getSession();       // 获取Session对象
session.setAttribute("loginTime", new Date());     // 设置Session中的属性

out.println("登录时间为：" +(Date)session.getAttribute("loginTime"));      // 获取Session属性
```

`request`还可以使用`getSession(boolean create)`来获取`Session`。区别是如果该客户的`Session`不存在，`request.getSession()`方法会返回`null`，而`getSession(true)`会先创建`Session`再将`Session`返回。

> 程序中`Session`中直接保存了`Person`类对象与`Date`类对象，使用起来要比`Cookie`方便。当多个客户端执行程序时，服务器会保存多个客户端的`Session`。获取`Session`的时候也不需要声明获取谁的`Session`。`Session`机制决定了当前客户只会获取到自己的`Session`，而不会获取到别人的`Session`。各客户的`Session`也彼此独立，互不可见。

> `Session`的使用比`Cookie`方便，但是过多的`Session`存储在服务器内存中，会对服务器造成压力。

## 4. Session的生命周期
`Session`保存在服务器端。为了获得更高的存取速度，服务器一般把`Session`放在内存里。每个用户都会有一个独立的`Session`。如果`Session`内容过于复杂，当大量客户访问服务器时可能会导致内存溢出。因此，`Session`里的信息应该尽量精简。

`Session`在用户第一次访问服务器的时候自动创建。需要注意只有访问`JSP`、`Servlet`等程序时才会创建`Session`，只访问`HTML`、`IMAGE`等静态资源并不会创建`Session`。如果尚未生成`Session`，也可以使用`request.getSession(true)`强制生成`Session`。

`Session`生成后，只要用户继续访问，服务器就会更新`Session`的最后访问时间，并维护该`Session`。用户每访问服务器一次，无论是否读写`Session`，服务器都认为该用户的`Session`活跃/`active`了一次。

## 5. Session的有效期
由于会有越来越多的用户访问服务器，因此`Session`也会越来越多。为防止内存溢出，服务器会把长时间内没有活跃的`Session`从内存删除。这个时间就是`Session`的超时时间。如果超过了超时时间没访问过服务器，`Session`就自动失效了。

`Session`的超时时间为`maxInactiveInterval`属性，可以通过对应的`getMaxInactiveInterval()`获取，通过`setMaxInactiveInterval(longinterval)`修改。

`Session`的超时时间也可以在`web.xml`中修改。另外，通过调用`Session`的`invalidate()`方法可以使`Session`失效。


## 5. Session的常用方法
`Session`中包括各种方法，使用起来要比`Cookie`方便得多。`Session`的常用方法如下所示。

- `void setAttribute(String attribute, Object value)`：设置`Session`属性。`value`参数可以为任何`Java Object`。通常为`Java Bean`。`value`信息不宜过大
- `String getAttribute(String attribute)`：返回`Session`属性 `Enumeration getAttributeNames()`：返回`Session`中存在的属性名
- `void removeAttribute(String attribute)`：移除`Session`属性 `String getId()`：返回`Session`的`ID`。该`ID`由服务器自动创建，不会重复
- `long getCreationTime()`：返回`Session`的创建日期。返回类型为`long`，常被转化为`Date`类型，例如：`Date createTime = new Date(session.get CreationTime())`
- `long getLastAccessedTime()`：返回`Session`的最后活跃时间。返回类型为`long`
- `int getMaxInactiveInterval()`：返回`Session`的超时时间。单位为秒。超过该时间没有访问，服务器认为该`Session`失效
- `void setMaxInactiveInterval(int second)`：设置`Session`的超时时间。单位为秒
- `void putValue(String attribute, Object value)`：不推荐的方法。已经被`setAttribute(String attribute, Object Value)`替代
- `Object getValue(String attribute)`：不被推荐的方法。已经被`getAttribute(String attr)`替代`boolean isNew()`：返回该`Session`是否是新创建的
- `void invalidate()`：使该`Session`失效



`Tomca`t中`Session`的默认超时时间为20分钟。通过`setMaxInactiveInterval(int seconds)`修改超时时间。可以修改`web.xml`改变`Session`的默认超时时间。例如修改为60分钟：
```xml
<session-config>
   <session-timeout>60</session-timeout>      <!-- 单位：分钟 -->
</session-config>
```
> 参数的单位为分钟，而setMaxInactiveInterval(int s)单位为秒。

在`server.xml`中定义`context`时采用如下定义（单位为秒）：

```xml
<Context path="/livsorder" docBase="/home/httpd/html/livsorder" defaultSessionTimeOut="3600" isWARExpanded="true"
    isWARValidated="false" isInvokerEnabled="true"
    isWorkDirPersistent="false"/>
```

## 6. `Session`对浏览器的要求

虽然`Session`保存在服务器，对客户端是透明的，它的正常运行仍然需要客户端浏览器的支持。这是因为`Session`需要使用`Cookie`作为识别标志。`HTTP`协议是无状态的，`Session`不能依据`HTTP`连接来判断是否为同一客户，因此服务器向客户端浏览器发送一个名为`JSESSIONID`的`Cookie`，它的值为该`Session`的`id`也就是`HttpSession.getId()`的返回值。`Session`依据该`Cookie`来识别是否为同一用户。

该`Cookie`为服务器自动生成的，它的`maxAge`属性一般为`–1`，表示仅当前浏览器内有效，并且各浏览器窗口间不共享，关闭浏览器就会失效。

因此同一机器的两个浏览器窗口访问服务器时，会生成两个不同的`Session`。但是由浏览器窗口内的链接、脚本等打开的新窗口（也就是说不是双击桌面浏览器图标等打开的窗口）除外。这类子窗口会共享父窗口的`Cookie`，因此会共享一个`Session`。

> 新开的浏览器窗口会生成新的`Session`，但子窗口除外。子窗口会共用父窗口的`Session`。例如，在链接上右击，在弹出的快捷菜单中选择“在新窗口中打开”时，子窗口便可以访问父窗口的`Session`。

如果客户端浏览器将`Cookie`功能禁用，或者不支持`Cookie`怎么办？例如，绝大多数的手机浏览器都不支持`Cookie`。`Java Web`提供了另一种解决方案：`URL`地址重写。

## 7. `URL`地址重写
`URL`地址重写是对客户端不支持`Cookie`的解决方案。`URL`地址重写的原理是将该用户`Session`的`id`信息重写到`URL`地址中。服务器能够解析重写后的`URL`获取`Session`的`id`。这样即使客户端不支持`Cookie`，也可以使用`Session`来记录用户状态。`HttpServletResponse`类提供了`encodeURL(Stringurl)`实现`URL`地址重写.

```html
<td>
    <a href="<%=response.encodeURL("index.jsp?c=1&wd=Java") %>">
    Homepage</a>
</td>
```

该方法会自动判断客户端是否支持`Cookie`。如果客户端支持`Cookie`，会将`URL`原封不动地输出来。如果客户端不支持`Cookie`，则会将用户`Session`的`id`重写到`URL`中。重写后的输出可能是这样的：
```Java
<td>
    <a href="index.jsp;jsessionid=0CCD096E7F8D97B0BE608AFDC3E1931E?c=1&wd=Java">Homepage</a>
</td>
```
即在文件名的后面，在`URL`参数的前面添加了字符串`;jsessionid=XXX`。其中`XXX`为`Session`的`id`。分析一下可以知道，增添的`jsessionid`字符串既不会影响请求的文件名，也不会影响提交的地址栏参数。用户单击这个链接的时候会把`Session`的`id`通过`URL`提交到服务器上，服务器通过解析`URL`地址获得`Session`的`id`。

如下：
```Java
<%
    if(“administrator”.equals(userName)) {
        response.sendRedirect(response.encodeRedirectURL(“administrator.jsp”));
        return;
    }
%>
```

效果跟`response.encodeURL(String url)`是一样的：如果客户端支持`Cookie`，生成原`URL`地址，如果不支持`Cookie`，传回重写后的带有`jsessionid`字符串的地址。

对于`WAP`程序，由于大部分的手机浏览器都不支持`Cookie`，`WAP`程序都会采用`URL`地址重写来跟踪用户会话。

> `TOMCAT`判断客户端浏览器是否支持`Cookie`的依据是请求中是否含有`Cookie`。尽管客户端可能会支持`Cookie`，但是由于第一次请求时不会携带任何`Cookie`（因为并无任何`Cookie`可以携带），`URL`地址重写后的地址中仍然会带有`jsessionid`。当第二次访问时服务器已经在浏览器中写入`Cookie`了，因此`URL`地址重写后的地址中就不会带有`jsessionid`了。

这两种方式对于用户来说是没有区别的，只是服务器在解析的时候处理的方式不同，采用第一种方式也有利于把`session id`的信息和正常程序参数区分开来。为了在整个交互过程中始终保持状态，就必须在每个客户端可能请求的路径后面都包含这个`session id`。

另一种叫做 **表单隐藏字段** 。就是服务器会自动修改表单，添加一个隐藏字段，以便在表单提交时能够把`session id`传递回服务器。比如下面的表单：

```html
<form name="testform" action="/xxx">
    <input type="text">
</form>
```
在被传递给客户端之前将被改写成：

```html
<form name="testform" action="/xxx">
    <input type="hidden" name="jsessionid" value="ByOK3vjFD75aPnrF7C2HmdnV6QZcEbzWoWiBYEnLerjQ99zWpBng!-145788764">
    <input type="text">
</form>
```
**这种技术现在已较少应用**。

在谈论`session`机制的时候，常常听到这样一种误解“只要关闭浏览器，`session`就消失了”。对`sessio`n来说，除非程序通知服务器删除一个`session`，否则服务器会一直保留，程序一般都是在用户做`log off`的时候发个指令去删除`session`。然而浏览器从来不会主动在关闭之前通知服务器它将要关闭，因此服务器根本不会有机会知道浏览器已经关闭，之所以会有这种错觉，是大部分`session`机制都使用会话`cookie`来保存`session id`，而关闭浏览器后这个`session id`就消失了，再次连接服务器时也就无法找到原来的`session`。如果服务器设置的`cookie`被保存到硬盘上，或者使用某种手段改写浏览器发出的`HTTP`请求头，把原来的`session id`发送给服务器，则再次打开浏览器仍然能够找到原来的`session`。

恰恰是由于关闭浏览器不会导致`session`被删除，迫使服务器为`seesion`设置了一个失效时间，当距离客户端上一次使用`session`的时间超过这个失效时间时，服务器就可以认为客户端已经停止了活动，才会把`session`删除以节省存储空间。

## 8. `Session`中禁止使用`Cookie`
既然`WAP`上大部分的客户浏览器都不支持`Cookie`，索性禁止`Session`使用`Cookie`，统一使用`URL`地址重写会更好一些。`Java Web`规范支持通过配置的方式禁用`Cookie`。下面举例说一下怎样通过配置禁止使用`Cookie`。

打开项目`sessionWeb`的`WebRoot`目录下的`META-INF`文件夹（跟`WEB-INF`文件夹同级，如果没有则创建），打开`context.xml`（如果没有则创建），编辑内容如下： `/META-INF/context.xml`：

```xml
<?xml version='1.0' encoding='UTF-8'?>
    <Context path="/sessionWeb"cookies="false">
</Context>
```
或者修改`Tomcat`全局的`conf/context.xml`，修改内容如下：` context.xml`：

```xml
<!-- The contents of this file will be loaded for eachweb application -->
<Context cookies="false">
    <!-- ... 中间代码略 -->
</Context>
```
部署后`TOMCAT`便不会自动生成名`JSESSIONID`的`Cookie`，`Session`也不会以`Cookie`为识别标志，而仅仅以重写后的`URL`地址为识别标志了

注意：该配置只是禁止`Session`使用`Cookie`作为识别标志，并不能阻止其他的`Cookie`读写。也就是说服务器不会自动维护名为`JSESSIONID`的`Cookie`了，但是程序中仍然可以读写其他的`Cookie`。
