
## Spring Security blog.

---

### 核心组件
---

#### 1.1 SecurityContextHolder

`SecurityContextHolder`用于存储安全上下文`security context`的信息。当前操作的用户是谁，该用户是否已经被认证，他拥有哪些角色权限…这些都被保存在`SecurityContextHolde`r中。`SecurityContextHolder`默认使用`ThreadLocal`策略来存储认证信息。看到`ThreadLocal`也就意味着，这是一种与线程绑定的策略。`Spring Security`在用户登录时自动绑定认证信息到当前线程，在用户退出时，自动清除当前线程的认证信息。但这一切的前提，是你在web场景下使用`Spring Security`，而如果是`Swing`界面，`Spring`也提供了支持，`SecurityContextHolder`的策略则需要被替换，鉴于我的初衷是基于web来介绍`Spring Security`，所以这里以及后续，非`web`的相关的内容都一笔带过。


**获取当前用户的信息**

因为身份信息是与线程绑定的，所以可以在程序的任何地方使用静态方法获取用户信息。一个典型的获取当前登录用户的姓名的例子如下所示：
```Java
Object principal = SecurityContextHolder.getContext().getAuthentication().getPrincipal();

if (principal instanceof UserDetails) {
  String username = ((UserDetails)principal).getUsername();
} else {
  String username = principal.toString();
}
```
`getAuthentication()`返回了认证信息，再次`getPrincipal()`返回了身份信息，`UserDetails`便是`Spring`对身份信息封装的一个接口。`Authentication`和`UserDetails`的介绍在下面的小节具体讲解，本节重要的内容是介绍`SecurityContextHolder`这个容器。

---

#### 1.2 Authentication

先看看这个接口的源码长什么样：
```Java
package org.springframework.security.core;// <1>

public interface Authentication extends Principal, Serializable { // <1>

    Collection<? extends GrantedAuthority> getAuthorities(); // <2>

    Object getCredentials();// <2>

    Object getDetails();// <2>

    Object getPrincipal();// <2>

    boolean isAuthenticated();// <2>

    void setAuthenticated(boolean var1) throws IllegalArgumentException;
}
```
1. <1> `Authentication`是`spring security`包中的接口，直接继承自`Principal`类，而`Principal`是位于`java.security`包中的。可以见得，`Authentication`在`spring security`中是最高级别的身份/认证的抽象。

2. <2> 由这个顶级接口，我们可以得到用户拥有的权限信息列表，密码，用户细节信息，用户身份信息，认证信息。

还记得1.1节中，`authentication.getPrincipal()`返回了一个`Object`，我们将`Principal`强转成了`Spring Security`中最常用的`UserDetails`，这在`Spring Security`中非常常见，接口返回`Object`，使用`instanceof`判断类型，强转成对应的具体实现类。接口详细解读如下：

* `getAuthorities()`，权限信息列表，默认是`GrantedAuthority`接口的一些实现类，通常是代表权限信息的一系列字符串。
* `getCredentials()`，密码信息，用户输入的密码字符串，在认证过后通常会被移除，用于保障安全。
* `getDetails()`，细节信息，`web`应用中的实现接口通常为 `WebAuthenticationDetails`，它记录了访问者的`ip`地址和`sessionId`的值。
* `getPrincipal()`，敲黑板！！！最重要的身份信息，大部分情况下返回的是`UserDetails`接口的实现类，也是框架中的常用接口之一。`UserDetails`接口将会在下面的小节重点介绍。

**Spring Security是如何完成身份认证的**

1. 用户名和密码被过滤器获取到，封装成`Authentication`,通常情况下是`UsernamePasswordAuthenticationToken`这个实现类。
2. `AuthenticationManager` 身份管理器负责验证这个`Authentication`
3. 认证成功后，`AuthenticationManager`身份管理器返回一个被填充满了信息的（包括上面提到的权限信息，身份信息，细节信息，但密码通常会被移除）`Authentication`实例。
4. `SecurityContextHolder`安全上下文容器将第3步填充了信息的`Authentication`，通过`SecurityContextHolder.getContext().setAuthentication(…)`方法，设置到其中。

`AuthenticationManager`将上述的流程转换成代码，便是如下的流程：

```Java
public class AuthenticationExample {
    private static AuthenticationManager am = new SampleAuthenticationManager();

    public static void main(String[] args) throws Exception {
        BufferedReader in = new BufferedReader(new InputStreamReader(System.in));

        while (true) {
            System.out.println("Please enter your username:");
            String name = in.readLine();
            System.out.println("Please enter your password:");
            String password = in.readLine();
            try {
                Authentication request = new UsernamePasswordAuthenticationToken(name, password);
                Authentication result = am.authenticate(request);
                SecurityContextHolder.getContext().setAuthentication(result);
                break;
            } catch (AuthenticationException e) {
                System.out.println("Authentication failed: " + e.getMessage());
            }
        }
        System.out.println("Successfully authenticated. Security context contains: " +
          SecurityContextHolder.getContext().getAuthentication());
    }
}

class SampleAuthenticationManager implements AuthenticationManager {
    static final List<GrantedAuthority> AUTHORITIES = new ArrayList<GrantedAuthority>();

    static {
        AUTHORITIES.add(new SimpleGrantedAuthority("ROLE_USER"));
    }

    public Authentication authenticate(Authentication auth) throws AuthenticationException {
        if (auth.getName().equals(auth.getCredentials())) {
            return new UsernamePasswordAuthenticationToken(auth.getName(), auth.getCredentials(), AUTHORITIES);
        }
        throw new BadCredentialsException("Bad Credentials");
    }
}

```
---

#### 1.3 AuthenticationManager

初次接触`Spring Security`的朋友相信会被`AuthenticationManager`，`ProviderManager` ，`AuthenticationProvider` …这么多相似的`Spring`认证类搞得晕头转向，但只要稍微梳理一下就可以理解清楚它们的联系和设计者的用意。`AuthenticationManager`（接口）是认证相关的核心接口，也是发起认证的出发点，因为在实际需求中，我们可能会允许用户使用用户名+密码登录，同时允许用户使用邮箱+密码，手机号码+密码登录，甚至，可能允许用户使用指纹登录（还有这样的操作？没想到吧），所以说`AuthenticationManager`一般不直接认证，`AuthenticationManager`接口的常用实现类`ProviderManager` 内部会维护一个`List<AuthenticationProvider>`列表，存放多种认证方式，实际上这是委托者模式的应用（Delegate）。也就是说，核心的认证入口始终只有一个：`AuthenticationManage`，不同的认证方式：用户名+密码（`UsernamePasswordAuthenticationToken`），邮箱+密码，手机号码+密码登录则对应了三个`AuthenticationProvider`。这样一来四不四就好理解多了？熟悉`shiro`的朋友可以把`AuthenticationProvider`理解成`Realm`。在默认策略下，只需要通过一个`AuthenticationProvider`的认证，即可被认为是登录成功。

```Java
public class ProviderManager implements AuthenticationManager, MessageSourceAware, InitializingBean {

    // 维护一个AuthenticationProvider列表
    private List<AuthenticationProvider> providers = Collections.emptyList();

    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        Class<? extends Authentication> toTest = authentication.getClass();
        AuthenticationException lastException = null;
        Authentication result = null;

        // 依次认证
        for (AuthenticationProvider provider : getProviders()) {
            if (!provider.supports(toTest)) {
                continue;
            }
            try {
                result = provider.authenticate(authentication);

                if (result != null) {
                    copyDetails(authentication, result);
                    break;
                }
            }
          ...
          catch(AuthenticationException e){
                lastException = e;
            }
        }
        // 如果有Authentication信息，则直接返回
        if (result != null) {
            if (eraseCredentialsAfterAuthentication && (result instanceof CredentialsContainer)) {
                //移除密码
                ((CredentialsContainer) result).eraseCredentials();
            }
            //发布登录成功事件
            eventPublisher.publishAuthenticationSuccess(result);
            return result;
        }
       ...
        //执行到此，说明没有认证成功，包装异常信息
        if (lastException == null) {
            lastException = new ProviderNotFoundException(messages.getMessage("ProviderManager.providerNotFound", new Object[]{toTest.getName()}, "No AuthenticationProvider found for {0}"));
        }
        prepareException(lastException, authentication);
        throw lastException;
    }
}

```

`ProviderManager`中的`List`，会依照次序去认证，认证成功则立即返回，若认证失败则返回null，下一个`AuthenticationProvider`会继续尝试认证，如果所有认证器都无法认证成功，则`ProviderManager`会抛出一个`ProviderNotFoundException`异常。

到这里，如果不纠结于`AuthenticationProvider`的实现细节以及安全相关的过滤器，认证相关的核心类其实都已经介绍完毕了：身份信息的存放容器`SecurityContextHolder`，身份信息的抽象`Authentication`，身份认证器`AuthenticationManager`及其认证流程。

---

#### 1.4 DaoAuthenticationProvider

**AuthenticationProvider最最最常用的一个实现便是DaoAuthenticationProvider**.

顾名思义，`Dao`正是数据访问层的缩写，也暗示了这个身份认证器的实现思路。由于本文是一个Overview，姑且只给出其UML类图.

按照我们最直观的思路，怎么去认证一个用户呢？用户前台提交了用户名和密码，而数据库中保存了用户名和密码，认证便是负责比对同一个用户名，提交的密码和保存的密码是否相同便是了。在`Spring Security`中。提交的用户名和密码，被封装成了`UsernamePasswordAuthenticationToken`，而根据用户名加载用户的任务则是交给了`UserDetailsService`，在`DaoAuthenticationProvider`中，对应的方法便是`retrieveUser`，虽然有两个参数，但是`retrieveUser`只有第一个参数起主要作用，返回一个`UserDetails`。还需要完成`UsernamePasswordAuthenticationToken`和`UserDetails`密码的比对，这便是交给`additionalAuthenticationChecks`方法完成的，如果这个`void`方法没有抛异常，则认为比对成功。比对密码的过程，用到了`PasswordEncoder`和`SaltSource`，密码加密和盐的概念相信不用我赘述了，它们为保障安全而设计，都是比较基础的概念。

---

#### 1.5 UserDetails与UserDetailsService

```Java
public interface UserDetails extends Serializable {

   Collection<? extends GrantedAuthority> getAuthorities();

   String getPassword();

   String getUsername();

   boolean isAccountNonExpired();

   boolean isAccountNonLocked();

   boolean isCredentialsNonExpired();

   boolean isEnabled();
}

```
它和`Authentication`接口很类似，比如它们都拥有`username，authorities`，区分他们也是本文的重点内容之一。`Authentication`的`getCredentials()`与`UserDetails`中的`getPassword()`需要被区分对待，前者是用户提交的密码凭证，后者是用户正确的密码，认证器其实就是对这两者的比对。`Authentication`中的`getAuthorities()`实际是由`UserDetails`的`getAuthorities()`传递而形成的。还记得`Authentication`接口中的`getUserDetails()`方法吗？其中的`UserDetails`用户详细信息便是经过了`AuthenticationProvider`之后被填充的。

```Java
public interface UserDetailsService {
   UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```
`UserDetailsService`和`AuthenticationProvider`两者的职责常常被人们搞混，关于他们的问题在文档的FAQ和issues中屡见不鲜。记住一点即可，敲黑板！！！
`UserDetailsService`只负责从特定的地方（通常是数据库）加载用户信息，仅此而已，记住这一点，可以避免走很多弯路。`UserDetailsService`常见的实现类有`JdbcDaoImpl`，`InMemoryUserDetailsManager`，前者从数据库加载用户，后者从内存中加载用户，也可以自己实现`UserDetailsService`，通常这更加灵活。

---

#### 1.6 架构概览图
![image][1]



----


#### 2 核心配置详解

---

##### 一般配置表
```Java
@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

  @Override
  protected void configure(HttpSecurity http) throws Exception {
      http
          .authorizeRequests()
              .antMatchers("/", "/home").permitAll()
              .anyRequest().authenticated()
              .and()
          .formLogin()
              .loginPage("/login")
              .permitAll()
              .and()
          .logout()
              .permitAll();
  }

  @Autowired
  public void configureGlobal(AuthenticationManagerBuilder auth) throws Exception {
      auth
          .inMemoryAuthentication()
              .withUser("admin").password("admin").roles("USER");
  }
}
```

---

#### 2.1  @EnableWebSecurity
我们自己定义的配置类`WebSecurityConfig`加上了`@EnableWebSecurity`注解，同时继承了`WebSecurityConfigurerAdapter`。你可能会在想谁的作用大一点，毫无疑问`@EnableWebSecurity`起到决定性的配置作用，它其实是个组合注解。
```Java
@Import({ WebSecurityConfiguration.class, // <2>
      SpringWebMvcImportSelector.class }) // <1>
@EnableGlobalAuthentication // <3>
@Configuration
public @interface EnableWebSecurity {
   boolean debug() default false;
}
```
* `SpringWebMvcImportSelector`的作用是判断当前的环境是否包含`springmvc`，因为`spring security`可以在非`spring`环境下使用，为了避免`DispatcherServlet`的重复配置，所以使用了这个注解来区分。
* `WebSecurityConfiguration`顾名思义，是用来配置`web`安全的，下面的小节会详细介绍。
* `@EnableGlobalAuthentication`注解的源码如下：

#####  2.1.1 @EnableGlobalAuthentication

```Java
@Import(AuthenticationConfiguration.class)
@Configuration
public @interface EnableGlobalAuthentication {
}
```

##### 2.1.2 WebSecurityConfiguration
在这个配置类中，有一个非常重要的Bean被注册了。
```Java
@Configuration
public class WebSecurityConfiguration {

	//DEFAULT_FILTER_NAME = "springSecurityFilterChain"
	@Bean(name = AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME)
    public Filter springSecurityFilterChain() throws Exception {
    	...
    }
 }
```

在未使用`springboot`之前，大多数人都应该对`springSecurityFilterChain`这个名词不会陌生，他是`spring security`的核心过滤器，是整个认证的入口。

`WebSecurityConfiguration`中完成了声明`springSecurityFilterChain`的作用，并且最终交给`DelegatingFilterProxy`这个代理类，负责拦截请求（注意`DelegatingFilterProxy`这个类不是`spring security`包中的，而是存在于`web`包中，`spring`使用了代理模式来实现安全过滤的解耦）。

##### 2.1.3 AuthenticationConfiguration

```Java
@Configuration
@Import(ObjectPostProcessorConfiguration.class)
public class AuthenticationConfiguration {

    @Bean
    public AuthenticationManagerBuilder authenticationManagerBuilder(
            ObjectPostProcessor<Object> objectPostProcessor) {
        return new AuthenticationManagerBuilder(objectPostProcessor);
    }

    public AuthenticationManager getAuthenticationManager() throws Exception {
        ...
    }
}

```

---

#### 2.2 WebSecurityConfigurerAdapter
适配器模式在`spring`中被广泛的使用，在配置中使用`Adapter`的好处便是，我们可以选择性的配置想要修改的那一部分配置，而不用覆盖其他不相关的配置。`WebSecurityConfigurerAdapter`中我们可以选择自己想要修改的内容，来进行重写，而其提供了三个`configure`重载方法，是我们主要关心的：
```Java
protected void configure(AuthenticationManagerBuilder auth) throws Exception {
  this.disableLocalConfigureAuthenticationBldr = true;
}

public void configure(WebSecurity web) throws Exception {
}

protected void configure(HttpSecurity http) throws Exception {
  logger.debug("Using default configure(HttpSecurity). If subclassed this will potentially override subclass configure(HttpSecurity).");

  http
    .authorizeRequests()
      .anyRequest().authenticated()
      .and()
    .formLogin().and()
    .httpBasic();
}
```
分别是对`AuthenticationManagerBuilder`，`WebSecurity`，`HttpSecurity`进行个性化的配置。.

##### 2.2.1HttpSecurity常用配置

```Java
@Configuration
@EnableWebSecurity
public class CustomWebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeRequests()
                .antMatchers("/resources/**", "/signup", "/about").permitAll()
                .antMatchers("/admin/**").hasRole("ADMIN")
                .antMatchers("/db/**").access("hasRole('ADMIN') and hasRole('DBA')")
                .anyRequest().authenticated()
                .and()
            .formLogin()
                .usernameParameter("username")
                .passwordParameter("password")
                .failureForwardUrl("/login?error")
                .loginPage("/login")
                .permitAll()
                .and()
            .logout()
                .logoutUrl("/logout")
                .logoutSuccessUrl("/index")
                .permitAll()
                .and()
            .httpBasic()
                .disable();
    }
}
```

上述是一个使用`Java Configuration`配置`HttpSecurity`的典型配置，其中`http`作为根开始配置，每一个`and()`对应了一个模块的配置（等同于`xml`配置中的结束标签），并且`and()`返回了`HttpSecurity`本身，于是可以连续进行配置。他们配置的含义也非常容易通过变量本身来推测，

* `authorizeRequests()`配置路径拦截，表明路径访问所对应的权限，角色，认证信息。
* `formLogin()`对应表单认证相关的配置
* `logout()`对应了注销相关的配置
* `httpBasic()`可以配置`basic`登录

他们分别代表了`http`请求相关的安全配置，这些配置项无一例外的返回了`Configurer`类，而所有的`http`相关配置可以通过查看`HttpSecurity`的主要方法得知：
需要对`http`协议有一定的了解才能完全掌握所有的配置，不过，`springboot`和`spring security`的自动配置已经足够使用了。其中每一项`Configurer（e.g.FormLoginConfigurer,CsrfConfigurer）`都是`HttpConfigurer`的细化配置项。


##### 2.2.2WebSecurityBuilder
```Java
@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    public void configure(WebSecurity web) throws Exception {
        web
            .ignoring()
            .antMatchers("/resources/**");
    }
}
```

##### 2.2.3AuthenticationManagerBuilder
```Java
@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth
            .inMemoryAuthentication()
            .withUser("admin").password("admin").roles("USER");
    }
}
```
----

### 3 过滤器详解

-----
Spring Security使用了springSecurityFillterChian作为了安全过滤的入口，这一节主要分析一下这个过滤器链都包含了哪些关键的过滤器，并且各自的使命是什么。

---

#### 3.1 Demo
你可以在 [github][2]中看到一个简单的demo.

由于过滤器链路中的过滤较多，即使是Spring Security的官方文档中也并未对所有的过滤器进行介绍，以此为例，来看看这过程中Spring Security都帮我们自动配置了哪些过滤器。

```Java
Creating filter chain: org.springframework.security.web.util.matcher.AnyRequestMatcher@1,
[
	org.springframework.security.web.context.request.async.WebAsyncManagerIntegrationFilter@85ec632,
	org.springframework.security.web.context.SecurityContextPersistenceFilter@1deb2c43,
	org.springframework.security.web.header.HeaderWriterFilter@5a2f016d,
	org.springframework.security.web.csrf.CsrfFilter@e72dba7,
	org.springframework.security.web.authentication.logout.LogoutFilter@4acf72b6,
	org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter@7a34b7b8,
	org.springframework.security.web.savedrequest.RequestCacheAwareFilter@1cefc4b3,
	org.springframework.security.web.servletapi.SecurityContextHolderAwareRequestFilter@3a7704c,
	org.springframework.security.web.authentication.AnonymousAuthenticationFilter@1c05a54d,
	org.springframework.security.web.session.SessionManagementFilter@3ad394e6,
	org.springframework.security.web.access.ExceptionTranslationFilter@3aaf4f07,
	org.springframework.security.web.access.intercept.FilterSecurityInterceptor@7b02e036
]
```
`spring security`的过滤器日志有一个特点：`log`打印顺序与实际配置顺序符合，也就意味着`SecurityContextPersistenceFilter`是整个过滤器链的第一个过滤器，而`FilterSecurityInterceptor`则是末置的过滤器。

- **SecurityContextPersistenceFilter** 两个主要职责：请求来临时，创建`SecurityContext`安全上下文信息，请求结束时清空`SecurityContextHolder`。
- `HeaderWriterFilter `(文档中并未介绍，非核心过滤器) 用来给`http`响应添加一些`Header`,比如`X-Frame-Options`, `X-XSS-Protection`，`X-Content-Type-Options`.
- `CsrfFilter` 在`spring4`这个版本中被默认开启的一个过滤器，用于防止`csrf`攻击，了解前后端分离的人一定不会对这个攻击方式感到陌生，前后端使用json交互需要注意的一个问题。
- `LogoutFilter` 顾名思义，处理注销的过滤器
- **UsernamePasswordAuthenticationFilter** 这个会重点分析，表单提交了`username`和`password`，被封装成`token`进行一系列的认证，便是主要通过这个过滤器完成的，在表单认证的方法中，这是最最关键的过滤器。
- `RequestCacheAwareFilter` (文档中并未介绍，非核心过滤器) 内部维护了一个`RequestCache`，用于缓存`request`请求
- `SecurityContextHolderAwareRequestFilter` 此过滤器对`ServletRequest`进行了一次包装，使得`request`具有更加丰富的`API`
- **AnonymousAuthenticationFilter** 匿名身份过滤器，这个过滤器个人认为很重要，需要将它与`UsernamePasswordAuthenticationFilter` 放在一起比较理解，`spring security`为了兼容未登录的访问，也走了一套认证流程，只不过是一个匿名的身份。
- `SessionManagementFilter` 和`session`相关的过滤器，内部维护了一个`SessionAuthenticationStrategy`，两者组合使用，常用来防止`session-fixation protection attack`，以及限制同一用户开启多个会话的数量
- **ExceptionTranslationFilter** 直译成异常翻译过滤器，还是比较形象的，这个过滤器本身不处理异常，而是将认证过程中出现的异常交给内部维护的一些类去处理，具体是那些类下面详细介绍
- **FilterSecurityInterceptor** 这个过滤器决定了访问特定路径应该具备的权限，访问的用户的角色，权限是什么？访问的路径需要什么样的角色和权限？这些判断和处理都是由该类进行的。

---

#### 3.2 code analyse

##### 3.2.1 SecurityContextPersistenceFilter

用户在登录过一次之后，后续的访问便是通过`sessionId`来识别，从而认为用户已经被认证。具体在何处存放用户信息，便是第一篇文章中提到的`SecurityContextHolder`；认证相关的信息是如何被存放到其中的，便是通过`SecurityContextPersistenceFilter`。

`SecurityContextPersistenceFilter`的两个主要作用便是请求来临时，创建`SecurityContext`安全上下文信息和请求结束时清空`SecurityContextHolder`。

顺带提一下：微服务的一个设计理念需要实现服务通信的无状态，而`http`协议中的无状态意味着不允许存在`session`，这可以通过`setAllowSessionCreation(false)` 实现，这并不意味着`SecurityContextPersistenceFilter`变得无用，因为它还需要负责清除用户信息。在`Spring Security`中，虽然安全上下文信息被存储于`Session`中，但我们在实际使用中不应该直接操作`Session`，而应当使用`SecurityContextHolder`。


>org.springframework.security.web.context.SecurityContextPersistenceFilter

```Java
public class SecurityContextPersistenceFilter extends GenericFilterBean {

    static final String FILTER_APPLIED = "__spring_security_scpf_applied";
    //安全上下文存储的仓库
    private SecurityContextRepository repo;

    public SecurityContextPersistenceFilter() {
        //HttpSessionSecurityContextRepository是SecurityContextRepository接口的一个实现类
        //使用HttpSession来存储SecurityContext
        this(new HttpSessionSecurityContextRepository());
    }

    public SecurityContextPersistenceFilter(SecurityContextRepository repo) {
        this.repo = repo;
    }

    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest) req;
        HttpServletResponse response = (HttpServletResponse) res;

        if (request.getAttribute(FILTER_APPLIED) != null) {
            // ensure that filter is only applied once per request
            chain.doFilter(request, response);
            return;
        }
        request.setAttribute(FILTER_APPLIED, Boolean.TRUE);
        //包装request，response
        HttpRequestResponseHolder holder = new HttpRequestResponseHolder(request, response);
        //从Session中获取安全上下文信息
        SecurityContext contextBeforeChainExecution = repo.loadContext(holder);
        try {
            //请求开始时，设置安全上下文信息，这样就避免了用户直接从Session中获取安全上下文信息
            SecurityContextHolder.setContext(contextBeforeChainExecution);
            chain.doFilter(holder.getRequest(), holder.getResponse());
        } finally {
            //请求结束后，清空安全上下文信息
            SecurityContext contextAfterChainExecution = SecurityContextHolder.getContext();
            SecurityContextHolder.clearContext();
            repo.saveContext(contextAfterChainExecution, holder.getRequest(), holder.getResponse());
            request.removeAttribute(FILTER_APPLIED);
            if (debug) {
                logger.debug("SecurityContextHolder now cleared, as request processing completed");
            }
        }
    }

}
```

##### 3.2.2 HttpSessionSecurityContextRepository

```Java
public class HttpSessionSecurityContextRepository implements SecurityContextRepository {
   // 'SPRING_SECURITY_CONTEXT'是安全上下文默认存储在Session中的键值
   public static final String SPRING_SECURITY_CONTEXT_KEY = "SPRING_SECURITY_CONTEXT";
   ...
   private final Object contextObject = SecurityContextHolder.createEmptyContext();
   private boolean allowSessionCreation = true;
   private boolean disableUrlRewriting = false;
   private String springSecurityContextKey = SPRING_SECURITY_CONTEXT_KEY;

   private AuthenticationTrustResolver trustResolver = new AuthenticationTrustResolverImpl();

   //从当前request中取出安全上下文，如果session为空，则会返回一个新的安全上下文
   public SecurityContext loadContext(HttpRequestResponseHolder requestResponseHolder) {
      HttpServletRequest request = requestResponseHolder.getRequest();
      HttpServletResponse response = requestResponseHolder.getResponse();
      HttpSession httpSession = request.getSession(false);
      SecurityContext context = readSecurityContextFromSession(httpSession);
      if (context == null) {
         context = generateNewContext();
      }
      ...
      return context;
   }

   ...

   public boolean containsContext(HttpServletRequest request) {
      HttpSession session = request.getSession(false);
      if (session == null) {
         return false;
      }
      return session.getAttribute(springSecurityContextKey) != null;
   }

   private SecurityContext readSecurityContextFromSession(HttpSession httpSession) {
      if (httpSession == null) {
         return null;
      }
      ...
      // Session存在的情况下，尝试获取其中的SecurityContext
      Object contextFromSession = httpSession.getAttribute(springSecurityContextKey);
      if (contextFromSession == null) {
         return null;
      }
      ...
      return (SecurityContext) contextFromSession;
   }

   //初次请求时创建一个新的SecurityContext实例
   protected SecurityContext generateNewContext() {
      return SecurityContextHolder.createEmptyContext();
   }

}

```
`SecurityContextPersistenceFilter`和`HttpSessionSecurityContextRepository`配合使用，构成了`Spring Security`整个调用链路的入口，为什么将它放在最开始的地方也是显而易见的，后续的过滤器中大概率会依赖`Session`信息和安全上下文信息。

##### 3.2.3 UsernamePasswordAuthenticationFilter

表单认证是最常用的一个认证方式，一个最直观的业务场景便是允许用户在表单中输入用户名和密码进行登录，而这背后的`UsernamePasswordAuthenticationFilter`，在整个`Spring Security`的认证体系中则扮演着至关重要的角色。
![uml][3]

上述的时序图，可以看出`UsernamePasswordAuthenticationFilter`主要肩负起了调用身份认证器，校验身份的作用，至于认证的细节，在前面几章花了很大篇幅进行了介绍，到这里，其实Spring Security的基本流程就已经走通了。

>org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter#attemptAuthentication
```Java
public Authentication attemptAuthentication(HttpServletRequest request,
      HttpServletResponse response) throws AuthenticationException {
   //获取表单中的用户名和密码
   String username = obtainUsername(request);
   String password = obtainPassword(request);
   ...
   username = username.trim();
   //组装成username+password形式的token
   UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(
         username, password);
   // Allow subclasses to set the "details" property
   setDetails(request, authRequest);
   //交给内部的AuthenticationManager去认证，并返回认证信息
   return this.getAuthenticationManager().authenticate(authRequest);
}
```

`UsernamePasswordAuthenticationFilter`本身的代码只包含了上述这么一个方法，非常简略，而在其父类`AbstractAuthenticationProcessingFilter`中包含了大量的细节.

```Java
public abstract class AbstractAuthenticationProcessingFilter extends GenericFilterBean
      implements ApplicationEventPublisherAware, MessageSourceAware {
	//包含了一个身份认证器
	private AuthenticationManager authenticationManager;
	//用于实现remeberMe
	private RememberMeServices rememberMeServices = new NullRememberMeServices();
	private RequestMatcher requiresAuthenticationRequestMatcher;
	//这两个Handler很关键，分别代表了认证成功和失败相应的处理器
	private AuthenticationSuccessHandler successHandler = new SavedRequestAwareAuthenticationSuccessHandler();
	private AuthenticationFailureHandler failureHandler = new SimpleUrlAuthenticationFailureHandler();

	public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
			throws IOException, ServletException {

		HttpServletRequest request = (HttpServletRequest) req;
		HttpServletResponse response = (HttpServletResponse) res;
		...
		Authentication authResult;
		try {
			//此处实际上就是调用UsernamePasswordAuthenticationFilter的attemptAuthentication方法
			authResult = attemptAuthentication(request, response);
			if (authResult == null) {
				//子类未完成认证，立刻返回
				return;
			}
			sessionStrategy.onAuthentication(authResult, request, response);
		}
		//在认证过程中可以直接抛出异常，在过滤器中，就像此处一样，进行捕获
		catch (InternalAuthenticationServiceException failed) {
			//内部服务异常
			unsuccessfulAuthentication(request, response, failed);
			return;
		}
		catch (AuthenticationException failed) {
			//认证失败
			unsuccessfulAuthentication(request, response, failed);
			return;
		}
		//认证成功
		if (continueChainBeforeSuccessfulAuthentication) {
			chain.doFilter(request, response);
		}
		//注意，认证成功后过滤器把authResult结果也传递给了成功处理器
		successfulAuthentication(request, response, chain, authResult);
	}

}
```

##### 3.2.4 AnonymousAuthenticationFilter
匿名认证过滤器，可能有人会想：匿名了还有身份？我自己对于`Anonymous`匿名身份的理解是`Spirng Security`为了整体逻辑的统一性，即使是未通过认证的用户，也给予了一个匿名身份。而`AnonymousAuthenticationFilter`该过滤器的位置也是非常的科学的，它位于常用的身份认证过滤器（如`UsernamePasswordAuthenticationFilter`、`BasicAuthenticationFilter`、`RememberMeAuthenticationFilter`）之后，意味着只有在上述身份过滤器执行完毕后，`SecurityContext`依旧没有用户信息，`AnonymousAuthenticationFilter`该过滤器才会有意义—-基于用户一个匿名身份。
```Java
public class AnonymousAuthenticationFilter extends GenericFilterBean implements
      InitializingBean {

   private AuthenticationDetailsSource<HttpServletRequest, ?> authenticationDetailsSource = new WebAuthenticationDetailsSource();
   private String key;
   private Object principal;
   private List<GrantedAuthority> authorities;


   //自动创建一个"anonymousUser"的匿名用户,其具有ANONYMOUS角色
   public AnonymousAuthenticationFilter(String key) {
      this(key, "anonymousUser", AuthorityUtils.createAuthorityList("ROLE_ANONYMOUS"));
   }

   /**
    *
    * @param key key用来识别该过滤器创建的身份
    * @param principal principal代表匿名用户的身份
    * @param authorities authorities代表匿名用户的权限集合
    */
   public AnonymousAuthenticationFilter(String key, Object principal,
         List<GrantedAuthority> authorities) {
      Assert.hasLength(key, "key cannot be null or empty");
      Assert.notNull(principal, "Anonymous authentication principal must be set");
      Assert.notNull(authorities, "Anonymous authorities must be set");
      this.key = key;
      this.principal = principal;
      this.authorities = authorities;
   }

   ...

   public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
         throws IOException, ServletException {
      //过滤器链都执行到匿名认证过滤器这儿了还没有身份信息，塞一个匿名身份进去
      if (SecurityContextHolder.getContext().getAuthentication() == null) {
         SecurityContextHolder.getContext().setAuthentication(
               createAuthentication((HttpServletRequest) req));
      }
      chain.doFilter(req, res);
   }

   protected Authentication createAuthentication(HttpServletRequest request) {
     //创建一个AnonymousAuthenticationToken
      AnonymousAuthenticationToken auth = new AnonymousAuthenticationToken(key,
            principal, authorities);
      auth.setDetails(authenticationDetailsSource.buildDetails(request));

      return auth;
   }
   ...
}
```

其实对比`AnonymousAuthenticationFilter`和`UsernamePasswordAuthenticationFilter`就可以发现一些门道了，`UsernamePasswordAuthenticationToken`对应`AnonymousAuthenticationToken`，他们都是`Authentication`的实现类，而`Authentication`则是被`SecurityContextHolder(SecurityContext)`持有的，一切都被串联在了一起。

##### 3.2.5 ExceptionTranslationFilter
`ExceptionTranslationFilter`异常转换过滤器位于整个`springSecurityFilterChain`的后方，用来转换整个链路中出现的异常，将其转化，顾名思义，转化以意味本身并不处理。一般其只处理两大类异常：`AccessDeniedException`访问异常和`AuthenticationException`认证异常。

这个过滤器非常重要，因为它将`Java`中的异常和`HTTP`的响应连接在了一起，这样在处理异常时，我们不用考虑密码错误该跳到什么页面，账号锁定该如何，只需要关注自己的业务逻辑，抛出相应的异常便可。如果该过滤器检测到`AuthenticationException`，则将会交给内部的`AuthenticationEntryPoint`去处理，如果检测到`AccessDeniedException`，需要先判断当前用户是不是匿名用户，如果是匿名访问，则和前面一样运行`AuthenticationEntryPoint`，否则会委托给`AccessDeniedHandler`去处理，而`AccessDeniedHandler`的默认实现，是`AccessDeniedHandlerImpl`。所以`ExceptionTranslationFilter`内部的`AuthenticationEntryPoint`是至关重要的，顾名思义：认证的入口点。


##### 3.2.6 FilterSecurityInterceptor 拦截器
由什么控制哪些资源是受限的，这些受限的资源需要什么权限，需要什么角色…这一切和访问控制相关的操作，都是由`FilterSecurityInterceptor`完成的。
`FilterSecurityInterceptor`的工作流程用笔者的理解可以理解如下：`FilterSecurityInterceptor`从`SecurityContextHolder`中获取`Authentication`对象，然后比对用户拥有的权限和资源所需的权限。前者可以通过`Authentication`对象直接获得，而后者则需要引入我们之前一直未提到过的两个类：`SecurityMetadataSource`，`AccessDecisionManager`。理解清楚决策管理器的整个创建流程和`SecurityMetadataSource`的作用需要花很大一笔功夫，这里，暂时只介绍其大概的作用。

```Java
@Override
protected void configure(HttpSecurity http) throws Exception {
  http
    .authorizeRequests()
    .antMatchers("/resources/**", "/signup", "/about").permitAll()
    .antMatchers("/admin/**").hasRole("ADMIN")
    .antMatchers("/db/**").access("hasRole('ADMIN') and hasRole('DBA')")
    .anyRequest().authenticated()
    .withObjectPostProcessor(new ObjectPostProcessor<FilterSecurityInterceptor>() {
    		public <O extends FilterSecurityInterceptor> O postProcess(
    						O fsi) {
    				fsi.setPublishAuthorizationSuccess(true);
    				return fsi;
    		}
    });
}

```

在`ObjectPostProcessor`的泛型中看到了`FilterSecurityInterceptor`，以笔者的经验，目前并没有太多机会需要修改`FilterSecurityInterceptor`的配置。


---

[1]:http://ov0zuistv.bkt.clouddn.com/spring%20security%20architecture.png
[2]:https://github.com/twentyworld/learn/tree/master/SpringSecurityLearn
[3]:http://ov0zuistv.bkt.clouddn.com/2011121410543010.jpg
