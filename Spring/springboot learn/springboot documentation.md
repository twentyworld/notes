# Spring boot
---


### basic code.
When a class doesn’t include a package declaration it is considered to be in the “default package”.
The use of the “default package” is generally discouraged, and should be avoided.
It can cause particular problems for Spring Boot applications that use @ComponentScan,
 @EntityScan or @SpringBootApplication annotations, since every class from every jar, will be read.

We generally recommend that you locate your main application class in a root package above other classes.
The @EnableAutoConfiguration annotation is often placed on your main class, and it implicitly defines a base “search package” for certain items.

Using a root package also allows the @ComponentScan annotation to be used without needing to specify a basePackage attribute.
 You can also use the @SpringBootApplication annotation if your main class is in the root package.

@Import: import additional configuration class.

@Importource: If you absolutely must use XML based configuration, we recommend that you still start with a @Configuration class.
 You can then use an additional @ImportResource annotation to load XML configuration files.


@EnableAutoConfiguration：Spring Boot auto-configuration attempts to automatically configure your Spring application
 based on the jar dependencies that you have added.
```
  @Configuration
  @EnableAutoConfiguration(exclude={DataSourceAutoConfiguration.class})
  public class MyConfiguration {
  }
```

@ComponentScan:a All application components (@Component, @Service, @Repository, @Controller etc.)
 will be automatically registered as Spring Beans.

```Java
@Service
public class DatabaseAccountService implements AccountService {
    private final RiskAssessor riskAssessor;
    @Autowire
    public DatabaseAccountService(RiskAssessor riskAssessor) {
        this.riskAssessor = riskAssessor;
    }
}
```
And if a bean has one constructor, you can omit the @Autowired.

```Java
@Service
public class DatabaseAccountService implements AccountService {
    private final RiskAssessor riskAssessor;
    @Autowired
    public DatabaseAccountService(RiskAssessor riskAssessor) {
        this.riskAssessor = riskAssessor;
    }
}
```

@SpringBootApplication: equivalent to using @Configuration, @EnableAutoConfiguration and @ComponentScan with their default attributes.
@SpringBootApplication also provides aliases to customize the attributes of @EnableAutoConfiguration and @ComponentScan.



---
### Spring Boot features

##### Application events and listeners

Some events are actually triggered before the ApplicationContext is created so you cannot register a listener on those as a @Bean.
You can register them via the SpringApplication.addListeners(…​) or SpringApplicationBuilder.listeners(…​) methods.
If you want those listeners to be registered automatically regardless of the way the application is created
 you can add a META-INF/spring.factories file to your project and reference your listener(s) using the
org.springframework.context.ApplicationListener key.
```
org.springframework.context.ApplicationListener=com.example.project.MyListener
```
> Application events are sent in the following order, as your application runs:
> * An ApplicationStartingEvent is sent at the start of a run, but before any processing except the registration of listeners and initializers.
> * An ApplicationEnvironmentPreparedEvent is sent when the Environment to be used in the context is known, but before the context is created.
> * An ApplicationPreparedEvent is sent just before the refresh is started, but after bean definitions have been loaded.
> * An ApplicationReadyEvent is sent after the refresh and any related callbacks have been processed to indicate the application is ready to service requests.
> * An ApplicationFailedEvent is sent if there is an exception on startup.

##### Web environment

A SpringApplication will attempt to create the right type of ApplicationContext on your behalf.
By default, an AnnotationConfigApplicationContext or AnnotationConfigEmbeddedWebApplicationContext will be used,
depending on whether you are developing a web application or not.

The algorithm used to determine a ‘web environment’ is fairly simplistic
(based on the presence of a few classes). You can use setWebEnvironment(boolean webEnvironment) if you need to override the default.


##### Accessing application arguments
If you need to access the application arguments that were passed to SpringApplication.run(…​) you can inject a org.springframework.boot.ApplicationArguments bean.
The ApplicationArguments interface provides access to both the raw String[] arguments as well as parsed option and non-option arguments:
```Java
@Component
public class MyBean {
    @Autowired
    public MyBean(ApplicationArguments args) {
        boolean debug = args.containsOption("debug");
        List<String> files = args.getNonOptionArgs();
        // if run with "--debug logfile.txt" debug=true, files=["logfile.txt"]
    }
}
```

##### Using the ApplicationRunner or CommandLineRunner
You can additionally implement the org.springframework.core.Ordered interface or use the org.springframework.core.annotation.Order annotation if several CommandLineRunner or ApplicationRunner beans are defined that must be called in a specific order.

----
### Externalized Configuration
Spring Boot uses a very particular PropertySource order that is designed to allow sensible overriding of values. Properties are considered in the following order:
>* Devtools global settings properties on your home directory (~/.spring-boot-devtools.properties when devtools is active).
* @TestPropertySource annotations on your tests.
* @SpringBootTest#properties annotation attribute on your tests.
* Command line arguments.
* Properties from SPRING_APPLICATION_JSON (inline JSON embedded in an environment variable or system property)
* ServletConfig init parameters.
* ServletContext init parameters.
* JNDI attributes from java:comp/env.
* Java System properties (System.getProperties()).
* OS environment variables.
* A RandomValuePropertySource that only has properties in random.\*.
* Profile-specific application properties outside of your packaged jar (application-{profile}.properties and YAML variants)
* Profile-specific application properties packaged inside your jar (application-{profile}.properties and YAML variants)
* Application properties outside of your packaged jar (application.properties and YAML variants).
* Application properties packaged inside your jar (application.properties and YAML variants).
* @PropertySource annotations on your @Configuration classes.
* Default properties (specified using SpringApplication.setDefaultProperties).

YAML files can’t be loaded via the @PropertySource annotation. So in the case that you need to load values that way, you need to use a properties file.

---
### Developing web applications
