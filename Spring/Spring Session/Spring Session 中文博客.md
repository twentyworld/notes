Spring session
---

Spring Session provides an API and implementations for managing a user’s session information.

---

### Session与Cookie基础
由于http协议是无状态的协议，为了能够记住请求的状态，于是引入了Session和Cookie的机制。

我们应该有一个很明确的概念，那就是Session是存在于服务器端的，在单体式应用中，他是由tomcat管理的，存在于tomcat的内存中，当我们为了解决分布式场景中的session共享问题时，引入了redis，其共享内存，以及支持key自动过期的特性，非常契合session的特性，我们在企业开发中最常用的也就是这种模式.

而Cookie则是存在于客户端，更方便理解的说法，可以说存在于浏览器中。http协议允许从服务器返回Response时携带一些Cookie，并且同一个域下对Cookie的数量有所限制，之前说过Session的持久化依赖于服务端的策略，而Cookie的持久化则是依赖于本地文件。虽然说Cookie并不常用，但是有一类特殊的Cookie却是我们需要额外关注的，那便是与Session相关的sessionId，他是真正维系客户端和服务端的桥梁。
