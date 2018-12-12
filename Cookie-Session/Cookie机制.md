# `Cookie`机制

## 1. `Cookie`解决的问题
`Cookie`技术是客户端的解决方案，`Cookie`就是由服务器发给客户端的特殊信息，而这些信息以文本文件的方式存放在客户端，然后客户端每次向服务器发送请求的时候都会带上这些特殊的信息。

当用户使用浏览器访问一个支持Cookie的网站的时候，用户会提供包括用户名在内的个人信息并且提交至服务器；接着，服务器在向客户端回传相应的超文本的同时也会发回这些个人信息，当然这些信息并不是存放在`HTTP`响应体`Response Body`中的，而是存放于`HTTP`响应头`Response Header`中,当客户端浏览器接收到来自服务器的响应之后，浏览器会将这些信息存放在一个统一的位置。

自此，客户端再向服务器发送请求的时候，都会把相应的`Cookie`再次发回至服务器。而这次，`Cookie`信息则存放在`HTTP`请求头`Request Header`了。

有了`Cookie`这样的技术实现，服务器在接收到来自客户端浏览器的请求之后，就能够通过分析存放于请求头的`Cookie`得到客户端特有的信息，从而动态生成与该客户端相对应的内容。

在程序中，会话跟踪是很重要的事情。理论上，一个用户的所有请求操作都应该属于同一个会话，而另一个用户的所有请求操作则应该属于另一个会话，二者不能混淆。而`Web`应用程序是使用`HTTP`协议传输数据的。`HTTP`协议是无状态的协议。一旦数据交换完毕，客户端与服务器端的连接就会关闭，再次交换数据需要建立新的连接。这就意味着服务器无法从连接上跟踪会话。
`Cookie`就是这样的一种机制。它可以弥补`HTTP`协议无状态的不足。在`Session`出现之前，基本上所有的网站都采用`Cookie`来跟踪会话。




## 2. `Cookie`定义
`Cookie`实际上是一小段的文本信息。客户端请求服务器，如果服务器需要记录该用户状态，就使用`response`向客户端浏览器颁发一个`Cookie`。客户端浏览器会把`Cookie`保存起来。当浏览器再请求该网站时，浏览器把请求的网址连同该`Cookie`一同提交给服务器。服务器检查该`Cookie`，以此来辨认用户状态。服务器还可以根据需要修改`Cookie`的内容。

有两个`http`头部是专门负责设置以及发送`cookie`的,它们分别是`Set-Cookie`以及`Cookie`。当服务器返回给客户端一个`http`响应信息时，其中如果包含`Set-Cookie`这个头部时，意思就是指示客户端建立一个`cookie`，并且在后续的`http`请求中自动发送这个`cookie`到服务器端，直到这个`cookie`过期。如果`cookie`的生存时间是整个会话期间的话，那么浏览器会将`cookie`保存在内存中，浏览器关闭时就会自动清除这个`cookie`。另外一种情况就是保存在客户端的硬盘中，浏览器关闭的话，该`cookie`也不会被清除，下次打开浏览器访问对应网站时，这个`cookie`就会自动再次发送到服务器端。


一个`cookie`的设置以及发送过程分为以下四步:
- 客户端发送一个`http`请求到服务器端
- 服务器端发送一个`http`响应到客户端，其中包含`Set-Cookie`头部
- 客户端发送一个`http`请求到服务器端，其中包含`Cookie`头部
- 服务器端发送一个`http`响应到客户端

![cookie interactive  step][1]

`Java`中把`Cookie`封装成了`javax.servlet.http.Cookie`类。每个`Cookie`都是该`Cookie类`的对象。服务器通过操作`Cookie类`对象对客户端`Cookie`进行操作。通过`request.getCookie()`获取客户端提交的所有`Cookie`（以`Cookie[]`数组形式返回），通过`response.addCookie(Cookie cookie)`向客户端设置`Cookie`。

`Cookie对象`使用`key-value`属性对的形式保存用户状态，一个`Cookie对象`保存一个属性对，一个`request`或者`response`同时使用多个`Cookie`。因为`Cookie类`位于包`javax.servlet.http.*` 下面，所以`JSP`中不需要`import`该类。

## 3. `Cookie`的特性

很多网站都会使用`Cookie`。例如，`Google`会向客户端颁发`Cookie`，`Baidu`也会向客户端颁发`Cookie`。那浏览器访问`Google`会不会也携带上`Baidu`颁发的`Cookie`呢？或者`Google`能不能修改`Baidu`颁发的`Cookie`呢？

答案是否定的。`Cookie`具有不可跨域名性。根据`Cookie`规范，浏览器访问`Google`只会携带`Google`的`Cookie`，而不会携带`Baidu`的`Cookie`。`Google`也只能操作`Google`的`Cookie`，而不能操作`Baidu`的`Cookie`。

`Cookie`在客户端是由浏览器来管理的。浏览器能够保证`Google`只会操作`Google`的`Cookie`而不会操作`Baidu`的`Cookie`，从而保证用户的隐私安全。浏览器判断一个网站是否能操作另一个网站`Cookie`的依据是域名。`Google`与`Baidu`的域名不一样，因此`Google`不能操作`Baidu`的`Cookie`。

需要注意的是，虽然网站`images.google.com`与网站`www.google.com`同属于`Google`，但是域名不一样，二者同样不能互相操作彼此的`Cookie`。

注意：用户登录网站`www.google.com`之后会发现访问`images.google.com`时登录信息仍然有效，而普通的`Cookie`是做不到的。这是因为`Google`做了特殊处理。本章后面也会对`Cookie`做类似的处理。


## 4. `Cookie`的属性设置

除了`name`与`value`之外，`Cookie`还具有其他几个常用的属性。每个属性对应一个`getter`方法与一个`setter`方法。

- `String name`：该`Cookie`的名称。`Cookie`一旦创建，名称便不可更改。
- `Object value`：该`Cookie`的值。如果值为`Unicode`字符，需要为字符编码。如果值为二进制数据，则需要使用`BASE64`编码。
- `int maxAge`：该`Cookie`失效的时间，单位秒。如果为正数，则该`Cookie`在`maxAge`秒之后失效。如果为负数，该`Cookie`为临时`Cookie`，关闭浏览器即失效，浏览器也不会以任何形式保存该`Cookie`。如果为`0`，表示删除该`Cookie`。默认为`–1`。
- `boolean secure`：该`Cookie`是否仅被使用安全协议传输。安全协议。安全协议有`HTTPS`，`SS`L等，在网络上传输数据之前先将数据加密。默认为`false`。
- `String path`：该`Cookie`的使用路径。如果设置为`/sessionWeb/`，则只有`contextPath`为`/sessionWeb`的程序可以访问该`Cookie`。如果设置为`/`，则本域名下`contextPath`都可以访问该`Cookie`。注意最后一个字符必须为`/`。
- `String domain`：可以访问该`Cookie`的域名。如果设置为`.google.com`，则所有以`google.com`结尾的域名都可以访问该`Cookie`。注意第一个字符必须为`.`。
- `String comment`：该`Cookie`的用处说明。浏览器显示`Cookie`信息的时候显示该说明。
- `int version`：该`Cookie`使用的版本号。`0`表示遵循`Netscape`的`Cookie`规范，`1`表示遵循`W3C`的`RFC 2109`规范。


### 4.1 `Cookie`的有效期

`Cookie`的`maxAge`决定着`Cookie`的有效期，单位为秒`Second`。`Cookie`中通过`getMaxAge()`方法与`setMaxAge(int maxAge)`方法来读写`maxAge`属性。 如果`maxAge`属性为正数，则表示该`Cookie`会在`maxAge`秒之后自动失效。浏览器会将`maxAge`为正数的`Cookie`持久化，即写到对应的`Cookie`文件中。无论客户关闭了浏览器还是电脑，只要还在`maxAge`秒之前，登录网站时该`Cookie`仍然有效。下面代码中的`Cookie`信息将永远有效。

```Java
Cookie cookie = new Cookie("username","helloweenvsfei"); // 新建Cookie
cookie.setMaxAge(Integer.MAX_VALUE); // 设置生命周期为MAX_VALUE
response.addCookie(cookie); // 输出到客户端
```
如果`maxAge`为负数，则表示该`Cookie`仅在本浏览器窗口以及本窗口打开的子窗口内有效，关闭窗口后该`Cookie`即失效。`maxAge`为负数的`Cookie`，为临时性`Cookie`，不会被持久化，不会被写到`Cookie`文件中。`Cookie`信息保存在浏览器内存中，因此关闭浏览器该`Cookie`就消失了。`Cookie`默认的`maxAge`值为`–1`。

如果`maxAge`为`0`，则表示删除该`Cookie`。`Cookie`机制没有提供删除`Cookie`的方法，因此通过设置该`Cookie`即时失效实现删除`Cookie`的效果。失效的`Cookie`会被浏览器从`Cookie`文件或者内存中删除：

```Java
Cookie cookie = new Cookie("username","helloweenvsfei"); // 新建Cookie
cookie.setMaxAge(0); // 设置生命周期为0，不能为负数
response.addCookie(cookie); // 必须执行这一句
```

`response`对象提供的`Cookie`操作方法只有一个添加操作`add(Cookie cookie)`。要想修改`Cookie`只能使用一个同名的`Cookie`来覆盖原来的`Cookie`，达到修改的目的。删除时只需要把`maxAge`修改为0即可。

注意：从客户端读取`Cookie`时，包括`maxAge`在内的其他属性都是不可读的，也不会被提交。浏览器提交`Cookie`时只会提交`name`与`value`属性。`maxAge`属性只被浏览器用来判断`Cookie`是否过期。


### 4.2 `Cookie`的修改、删除
`Cookie`并不提供修改、删除操作。如果要修改某个`Cookie`，只需要新建一个同名的`Cookie`，添加到`response`中覆盖原来的`Cookie`。如果要删除某个`Cookie`，只需要新建一个同名的`Cookie`，并将`maxAge`设置为`0`，并添加到`response`中覆盖原来的`Cookie`。注意是`0`而不是负数。负数代表其他的意义。读者可以通过上例的程序进行验证，设置不同的属性。

注意：修改、删除`Cookie`时，新建的`Cookie`除`value`、`maxAge`之外的所有属性，例如`name`、`path`、`domain`等，都要与原`Cookie`完全一样。否则，浏览器将视为两个不同的`Cookie`不予覆盖，导致修改、删除失败。

### 4.3` Cookie`的域名
`Cookie`是不可跨域名的。域名`www.google.com`颁发的`Cookie`不会被提交到域名`www.baidu.com`去。这是由`Cookie`的隐私安全机制决定的。隐私安全机制能够禁止网站非法获取其他网站的`Cookie`。

正常情况下，同一个一级域名下的两个二级域名如`www.helloweenvsfei.com`和`images.helloweenvsfei.com`也不能交互使用`Cookie`，因为二者的域名并不严格相同。如果想所有`helloweenvsfei.com`名下的二级域名都可以使用该`Cookie`，需要设置`Cookie`的`domain`参数。

```Java
Cookie cookie = new Cookie("time","20080808"); // 新建Cookie
cookie.setDomain(".helloweenvsfei.com"); // 设置域名
cookie.setPath("/"); // 设置路径
cookie.setMaxAge(Integer.MAX_VALUE); // 设置有效期
response.addCookie(cookie); // 输出到客户端
```

### 4.4 `Cookie`的路径
`domain`属性决定运行访问`Cookie`的域名，而`path`属性决定允许访问`Cookie`的路径`ContextPath`。

如果只允许`/sessionWeb/`下的程序使用`Cookie`:

```Java
Cookie cookie = new Cookie("time","20080808"); // 新建Cookie
cookie.setPath("/session/"); // 设置路径
response.addCookie(cookie); // 输出到客户端
```

设置为`/`时允许所有路径使用`Cookie`。`path`属性需要使用符号`/`结尾。`name`相同但`domain`不同的两个`Cookie`也是两个不同的`Cookie`。

>页面只能获取它属于的Path的Cookie。例如/session/test/a.jsp不能获取到路径为/session/abc/的Cookie。使用时一定要注意。

- `domain`表示的是`cookie`所在的域，默认为请求的地址，如网址为`www.test.com/test/test.aspx`，那么`domain`默认为`www.test.com`。而跨域访问，如域`A`为`t1.test.com`，域`B`为`t2.test.com`，那么在域`A`生产一个令域`A`和域`B`都能访问的`cookie`就要将该`cookie`的`domain`设置为`.test.com`；如果要在域`A`生产一个令域`A`不能访问而域`B`能访问的`cookie`就要将该`cookie`的`domain`设置为`t2.test.com`。

- `path`表示`cookie`所在的目录，默认为`/`，就是根目录。在同一个服务器上有目录如下：`/test/`,`/test/cd/`,`/test/dd/`，现设一个`cookie1`的`path`为`/test/`，`cookie2`的`path`为`/test/cd/`，那么`test`下的所有页面都可以访问到`cookie1`，而`/test/`和`/test/dd/`的子页面不能访问`cookie2`。这是因为`cookie`能让其`path`路径下的页面访问。

- 浏览器会将`domain`和`path`都相同的`cookie`保存在一个文件里，`cookie`间用`*`隔开。

### 4.5 `Cookie`的安全属性

`HTTP`协议不仅是无状态的，而且是不安全的。使用`HTTP`协议的数据不经过任何加密就直接在网络上传播，有被截获的可能。使用`HTTP`协议传输很机密的内容是一种隐患。如果不希望`Cookie`在`HTTP`等非安全协议中传输，可以设置`Cookie`的`secure`属性为`true`。浏览器只会在`HTTPS`和`SSL`等安全协议中传输此类`Cookie`。

```Java
Cookie cookie = new Cookie("time", "20080808"); // 新建Cookie
cookie.setSecure(true); // 设置安全属性
response.addCookie(cookie); // 输出到客户端
```
`secure`属性并不能对`Cookie`内容加密，因而不能保证绝对的安全性。如果需要高安全性，需要在程序中对`Cookie`内容加密、解密，以防泄密。

`Cookie`是保存在浏览器端的，因此浏览器具有操作`Cookie`的先决条件。浏览器可以使用脚本程序如`JavaScript`或者`VBScript`等操作`Cookie`。
