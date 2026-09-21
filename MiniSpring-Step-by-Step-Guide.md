# Build Your Own Dependency Injection Framework — MiniSpring

A step-by-step guide to building **MiniSpring**, a small dependency-injection and web framework inspired by the core ideas behind Spring Framework.

The goal is **not** to reproduce Spring internally. The goal is to understand the mechanisms that make a framework such as Spring Boot possible:

- Component scanning
- Bean definitions
- Dependency injection
- Reflection
- Constructor and field injection
- Bean lifecycle
- Application context
- Configuration
- Circular-dependency detection
- MVC routing
- Embedded HTTP server
- A single `MiniApplication.run(...)` entry point

By the end, you will have a small but extensible framework with an API similar to:

```java
@MiniApplication
public class AppConfig {
    public static void main(String[] args) {
        MiniApplication.run(AppConfig.class, args);
    }
}
```

And application classes such as:

```java
@Service
public class UserService {

    private final UserRepository userRepository;

    @Autowired
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

```java
@Repository
public class UserRepository {

    public String findUser() {
        return "Arpan";
    }
}
```

```java
@Controller
@RequestMapping("/users")
public class UserController {

    private final UserService userService;

    @Autowired
    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping("/{id}")
    public String getUser(@PathVariable("id") String id) {
        return "User: " + id;
    }
}
```

---

# 1. What We Are Building

MiniSpring will contain four major layers:

```text
                         +--------------------------+
                         |       Application        |
                         | @Service @Controller ... |
                         +------------+-------------+
                                      |
                                      v
                         +--------------------------+
                         |    MiniApplication       |
                         |      run(...)            |
                         +------------+-------------+
                                      |
                                      v
                         +--------------------------+
                         |    ApplicationContext    |
                         |                          |
                         |  scan -> register ->    |
                         |  instantiate -> inject  |
                         +------------+-------------+
                                      |
                                      v
                         +--------------------------+
                         |       BeanFactory        |
                         |                          |
                         | create / cache / resolve |
                         +------------+-------------+
                                      |
                                      v
                         +--------------------------+
                         |      Reflection          |
                         |                          |
                         | constructors / fields    |
                         | methods / annotations    |
                         +--------------------------+
```

For the web layer:

```text
HTTP Request
     |
     v
+-------------------+
| Embedded HTTP     |
| Server            |
+---------+---------+
          |
          v
+-------------------+
| DispatcherServlet |
+---------+---------+
          |
          v
+-------------------+
| HandlerMapping    |
+---------+---------+
          |
          v
@Controller method
          |
          v
HTTP Response
```

---

# 2. Learning Objectives

After completing this project, you should understand:

1. How Java annotations work.
2. How runtime annotations can be discovered using reflection.
3. How classpath scanning works.
4. How dependency injection containers work.
5. Why a BeanFactory is different from an ApplicationContext.
6. How Spring-style stereotypes can be implemented.
7. How constructor injection works.
8. How field injection works.
9. How bean scopes can be implemented.
10. How singleton caching works.
11. How circular dependencies can be detected.
12. How bean lifecycle callbacks work.
13. How an HTTP request can be mapped to a controller method.
14. How `@PathVariable` and `@RequestParam` can be resolved.
15. How an embedded HTTP server can expose controllers.
16. How a framework bootstrapping API can be designed.
17. How to evolve a simple implementation toward a scalable architecture.

---

# 3. Recommended Technology Stack

Use:

```text
Java             21+
Build Tool       Maven
Testing          JUnit 5
HTTP Server      JDK HttpServer initially
Logging          SLF4J + Logback (optional)
```

Do not use Spring for the framework implementation.

The entire point is to understand how the framework works underneath.

---

# 4. Project Structure

Start with a multi-module Maven project.

```text
mini-spring/
│
├── pom.xml
│
├── minispring-core/
│   ├── pom.xml
│   └── src/main/java/
│       └── com/minispring/
│           ├── annotation/
│           ├── bean/
│           ├── context/
│           ├── exception/
│           ├── scanner/
│           ├── factory/
│           └── lifecycle/
│
├── minispring-web/
│   ├── pom.xml
│   └── src/main/java/
│       └── com/minispring/web/
│           ├── annotation/
│           ├── dispatcher/
│           ├── mapping/
│           ├── handler/
│           ├── argument/
│           └── server/
│
├── minispring-boot/
│   ├── pom.xml
│   └── src/main/java/
│       └── com/minispring/boot/
│
└── mini-spring-demo/
    ├── pom.xml
    └── src/main/java/
        └── com/example/app/
            ├── AppConfig.java
            ├── controller/
            ├── service/
            └── repository/
```

Conceptually:

```text
minispring-core
       |
       +---- DI container
       |
       +---- Component scanning
       |
       +---- Bean lifecycle

minispring-web
       |
       +---- MVC
       |
       +---- HTTP server

minispring-boot
       |
       +---- Application bootstrap

mini-spring-demo
       |
       +---- User application
```

---

# 5. Phase 1 — Create the Base Maven Project

Root `pom.xml`:

```xml
<project>
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.minispring</groupId>
    <artifactId>mini-spring</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>

    <modules>
        <module>minispring-core</module>
        <module>minispring-web</module>
        <module>minispring-boot</module>
        <module>mini-spring-demo</module>
    </modules>

    <properties>
        <java.version>21</java.version>
        <maven.compiler.source>21</maven.compiler.source>
        <maven.compiler.target>21</maven.compiler.target>
    </properties>
</project>
```

Build:

```bash
mvn clean install
```

---

# 6. Phase 2 — Design the Annotation System

We want:

```java
@Component
@Service
@Repository
@Controller
```

The first version can make all stereotypes directly discoverable.

## 6.1 `@Component`

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Component {
}
```

Important:

```text
@Target(TYPE)
```

means it can be placed on classes.

```text
@Retention(RUNTIME)
```

means reflection can discover it while the application is running.

---

# 7. Stereotype Annotations

Create:

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Component
public @interface Service {
}
```

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Component
public @interface Repository {
}
```

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Component
public @interface Controller {
}
```

This demonstrates a very important framework concept:

```text
@Controller
@Service
@Repository
       |
       v
   @Component
       |
       v
Component Scanner
```

A more advanced implementation should not rely on annotation-to-annotation behavior directly because Java reflection does not automatically treat a custom annotation as if it were annotated with another annotation in every lookup scenario.

Instead, MiniSpring can explicitly recognize stereotypes:

```java
public boolean isComponent(Class<?> type) {

    return type.isAnnotationPresent(Component.class)
        || type.isAnnotationPresent(Service.class)
        || type.isAnnotationPresent(Repository.class)
        || type.isAnnotationPresent(Controller.class);
}
```

Later, create a generic stereotype/meta-annotation mechanism.

---

# 8. Phase 3 — `@Autowired`

Create:

```java
@Target({
        ElementType.CONSTRUCTOR,
        ElementType.FIELD
})
@Retention(RetentionPolicy.RUNTIME)
public @interface Autowired {
}
```

This gives us two injection styles:

```java
@Autowired
public UserService(UserRepository repository) {
    this.repository = repository;
}
```

and:

```java
@Autowired
private UserRepository repository;
```

Constructor injection should be the preferred approach.

---

# 9. Phase 4 — Component Scanning

This is one of the most important parts of the project.

Spring-like frameworks need to answer:

> "Given package `com.example.app`, which classes are components?"

We need a:

```java
ClassPathScanner
```

API:

```java
public interface ClassPathScanner {

    Set<Class<?>> scan(String basePackage);
}
```

Implementation:

```java
public class DefaultClassPathScanner
        implements ClassPathScanner {

    @Override
    public Set<Class<?>> scan(String basePackage) {
        // discover .class files
        // convert to class names
        // load classes
        // inspect annotations
        return ...;
    }
}
```

---

# 10. Understanding Java Classpath Scanning

Suppose:

```text
com.example.app
```

contains:

```text
com/example/app/AppConfig.class
com/example/app/UserService.class
com/example/app/UserRepository.class
com/example/app/UserController.class
```

We need to discover:

```text
com.example.app.UserService
com.example.app.UserRepository
com.example.app.UserController
```

The process:

```text
Base Package
     |
     v
ClassLoader
     |
     v
Classpath resources
     |
     v
Directory / JAR
     |
     v
.class files
     |
     v
Class.forName(...)
     |
     v
Class<?>
     |
     v
Annotation inspection
```

---

# 11. Getting the Package Location

Use:

```java
String path = basePackage.replace('.', '/');

ClassLoader classLoader =
        Thread.currentThread().getContextClassLoader();

Enumeration<URL> resources =
        classLoader.getResources(path);
```

For every URL:

```java
while (resources.hasMoreElements()) {

    URL resource = resources.nextElement();

    System.out.println(resource);
}
```

Initially support:

```text
file:
```

resources.

Later add:

```text
jar:
```

resources.

---

# 12. Scanning Directory Classes

For a file-system resource:

```java
File directory =
        new File(resource.toURI());
```

Recursively scan:

```java
private void scanDirectory(
        File directory,
        String packageName,
        Set<Class<?>> classes) {

    for (File file : directory.listFiles()) {

        if (file.isDirectory()) {

            scanDirectory(
                file,
                packageName + "." + file.getName(),
                classes
            );

        } else if (file.getName().endsWith(".class")) {

            String className =
                    packageName + "." +
                    file.getName()
                        .replace(".class", "");

            loadClass(className, classes);
        }
    }
}
```

Load:

```java
private void loadClass(
        String className,
        Set<Class<?>> classes) {

    try {

        Class<?> clazz =
                Class.forName(className);

        if (isComponent(clazz)) {
            classes.add(clazz);
        }

    } catch (ClassNotFoundException e) {
        throw new MiniSpringException(e);
    }
}
```

---

# 13. Exclude Non-Instantiable Types

Do not register:

```text
interfaces
abstract classes
annotations
enums
```

Use:

```java
if (clazz.isInterface()) {
    return;
}

if (Modifier.isAbstract(clazz.getModifiers())) {
    return;
}

if (clazz.isAnnotation()) {
    return;
}
```

Eventually you should also decide how to handle:

```text
inner classes
records
synthetic classes
anonymous classes
```

---

# 14. Jar Scanning

Production frameworks cannot assume classes exist only as directories.

When packaged:

```text
app.jar
 ├── com/example/App.class
 ├── com/example/UserService.class
 └── ...
```

The scanner must support:

```text
file:
jar:
```

Use:

```java
JarURLConnection
```

or:

```java
JarFile
```

to enumerate entries.

The scanner architecture should therefore separate:

```java
ResourceResolver
```

from:

```java
ClassPathScanner
```

For example:

```text
ClassPathScanner
       |
       v
ResourceResolver
       |
       +---- FileResourceResolver
       |
       +---- JarResourceResolver
```

This follows the Open/Closed Principle.

---

# 15. Phase 5 — BeanDefinition

Do not immediately create objects when scanning.

First create metadata.

```java
public class BeanDefinition {

    private Class<?> beanClass;

    private String beanName;

    private Scope scope;

    private boolean lazy;

    // getters/setters
}
```

Enum:

```java
public enum Scope {
    SINGLETON,
    PROTOTYPE
}
```

This gives us:

```text
Class
 |
 v
BeanDefinition
 |
 v
BeanFactory
 |
 v
Object
```

This separation becomes extremely important later.

---

# 16. Bean Naming

Default bean name:

```java
UserService
```

becomes:

```text
userService
```

Implement:

```java
public String generateBeanName(Class<?> type) {

    String simpleName = type.getSimpleName();

    return Character.toLowerCase(simpleName.charAt(0))
            + simpleName.substring(1);
}
```

For:

```java
UserService
```

result:

```text
userService
```

For:

```java
OrderRepository
```

result:

```text
orderRepository
```

---

# 17. Custom Bean Name

Extend annotations:

```java
public @interface Component {

    String value() default "";
}
```

Then:

```java
@Component("primaryUserService")
public class UserService {
}
```

If `value()` is empty:

```text
userService
```

Otherwise:

```text
primaryUserService
```

---

# 18. Phase 6 — BeanFactory

Create the main abstraction:

```java
public interface BeanFactory {

    Object getBean(String name);

    <T> T getBean(Class<T> type);

    boolean containsBean(String name);
}
```

Implementation:

```java
public class DefaultBeanFactory
        implements BeanFactory {
}
```

Internal state:

```java
private final Map<String, BeanDefinition> definitions;

private final Map<String, Object> singletonObjects;
```

Conceptually:

```text
BeanDefinition Map

"userService"
      |
      v
UserService.class


Singleton Map

"userService"
      |
      v
UserService@12345
```

---

# 19. Singleton Behavior

Calling:

```java
context.getBean(UserService.class);
```

twice should return the same object:

```java
UserService a =
    context.getBean(UserService.class);

UserService b =
    context.getBean(UserService.class);

System.out.println(a == b);
```

Expected:

```text
true
```

Implementation:

```java
if (singletonObjects.containsKey(beanName)) {
    return singletonObjects.get(beanName);
}
```

Otherwise create:

```java
Object bean = createBean(beanName);
singletonObjects.put(beanName, bean);

return bean;
```

---

# 20. Phase 7 — Constructor Injection

Suppose:

```java
@Service
public class UserService {

    private final UserRepository repository;

    @Autowired
    public UserService(UserRepository repository) {
        this.repository = repository;
    }
}
```

Reflection:

```java
Constructor<?>[] constructors =
        beanClass.getDeclaredConstructors();
```

Find the constructor:

```java
Constructor<?> autowiredConstructor = null;

for (Constructor<?> constructor : constructors) {

    if (constructor.isAnnotationPresent(
            Autowired.class)) {

        autowiredConstructor = constructor;
        break;
    }
}
```

Resolve parameters:

```java
Class<?>[] parameterTypes =
        constructor.getParameterTypes();
```

Then:

```java
Object dependency =
        getBean(parameterType);
```

Finally:

```java
constructor.setAccessible(true);

Object bean =
        constructor.newInstance(dependencies);
```

---

# 21. Constructor Selection Rules

Define deterministic rules.

Recommended MiniSpring rules:

```text
1. Exactly one @Autowired constructor
      -> use it

2. Multiple @Autowired constructors
      -> throw error

3. No @Autowired constructor
      + exactly one constructor
      -> use it

4. No @Autowired constructor
      + multiple constructors
      -> use no-arg constructor

5. No usable constructor
      -> throw error
```

Example:

```java
@Service
public class PaymentService {

    public PaymentService() {
    }

    public PaymentService(PaymentRepository repository) {
    }
}
```

If there are multiple constructors and none is annotated, MiniSpring should either use the no-arg constructor or report an ambiguity error. Pick one behavior and document it.

---

# 22. Phase 8 — Field Injection

Example:

```java
@Service
public class UserService {

    @Autowired
    private UserRepository repository;
}
```

After constructing the object:

```java
for (Field field : beanClass.getDeclaredFields()) {

    if (!field.isAnnotationPresent(Autowired.class)) {
        continue;
    }

    Object dependency =
            getBean(field.getType());

    field.setAccessible(true);
    field.set(bean, dependency);
}
```

This works because Java reflection allows access to private fields when explicitly enabled.

---

# 23. Why Constructor Injection Should Be Preferred

Constructor injection provides:

```text
Required dependencies
       |
       v
Constructor
       |
       v
Fully initialized object
```

Benefits:

- Immutable dependencies
- Easier unit testing
- Dependencies are explicit
- Object cannot be constructed incorrectly as easily
- Circular dependencies become easier to detect

Field injection is useful for learning because it demonstrates reflection-based property injection.

---

# 24. Phase 9 — Dependency Resolution

Suppose:

```text
UserController
       |
       v
UserService
       |
       v
UserRepository
```

Calling:

```java
getBean(UserController.class)
```

should cause:

```text
UserController
     |
     +--> UserService
              |
              +--> UserRepository
```

Algorithm:

```text
getBean(UserController)
        |
        v
create UserController
        |
        v
resolve UserService
        |
        v
create UserService
        |
        v
resolve UserRepository
        |
        v
create UserRepository
        |
        v
return UserRepository
        |
        v
return UserService
        |
        v
return UserController
```

This is the heart of dependency injection.

---

# 25. Phase 10 — Circular Dependency Detection

Consider:

```java
@Service
public class A {

    private final B b;

    @Autowired
    public A(B b) {
        this.b = b;
    }
}
```

```java
@Service
public class B {

    private final A a;

    @Autowired
    public B(A a) {
        this.a = a;
    }
}
```

Request:

```java
getBean(A.class);
```

Flow:

```text
A
 |
 v
B
 |
 v
A
 |
 v
B
 |
 v
...
```

We need a creation stack.

```java
private final ThreadLocal<Deque<String>>
        creationStack =
        ThreadLocal.withInitial(ArrayDeque::new);
```

Before creation:

```java
if (creationStack.get().contains(beanName)) {
    throw new CircularDependencyException(
        "Circular dependency detected: "
        + creationStack.get()
    );
}
```

Then:

```java
creationStack.get().push(beanName);

try {

    return createBean(beanName);

} finally {

    creationStack.get().pop();
}
```

---

# 26. Improve Circular Dependency Reporting

Instead of:

```text
Circular dependency detected
```

report:

```text
Circular dependency detected:

userController
 -> userService
 -> paymentService
 -> userService
```

Maintain:

```java
Deque<String>
```

and format it into an understandable dependency path.

This is an important framework-quality improvement.

---

# 27. Constructor Circular Dependency vs Field Circular Dependency

Constructor cycle:

```text
A -> B -> A
```

cannot be resolved by simply creating partially initialized objects.

Field injection theoretically allows:

```text
create A
create B
inject A into B
inject B into A
```

This is one reason real DI containers have sophisticated singleton creation and early-reference mechanisms.

For MiniSpring V1:

> Detect circular dependencies and fail fast.

Later:

> Implement three-level singleton caches to explore how Spring handles certain circular-reference scenarios.

---

# 28. Phase 11 — ApplicationContext

Now introduce:

```java
public interface ApplicationContext
        extends BeanFactory {

}
```

Implementation:

```java
public class AnnotationConfigApplicationContext
        implements ApplicationContext {

    private final DefaultBeanFactory beanFactory;

    public AnnotationConfigApplicationContext(
            Class<?> configClass) {

        // scan
        // register definitions
        // initialize beans
    }
}
```

Responsibilities:

```text
ApplicationContext
    |
    +---- configuration
    |
    +---- component scanning
    |
    +---- BeanDefinition registration
    |
    +---- BeanFactory
    |
    +---- lifecycle
    |
    +---- application events
```

---

# 29. Context Startup Flow

When:

```java
new AnnotationConfigApplicationContext(AppConfig.class);
```

execute:

```text
1. Determine base package
2. Scan classes
3. Detect components
4. Create BeanDefinitions
5. Register definitions
6. Instantiate eager singleton beans
7. Perform dependency injection
8. Execute lifecycle callbacks
9. Start application
```

---

# 30. Determining the Base Package

Suppose:

```java
package com.example.app;

@MiniApplication
public class AppConfig {
}
```

Use:

```java
String packageName =
        AppConfig.class.getPackageName();
```

Result:

```text
com.example.app
```

Scan from there.

This makes:

```java
MiniApplication.run(AppConfig.class, args);
```

the equivalent of:

```text
"Start scanning from the package containing AppConfig."
```

---

# 31. Phase 12 — `@MiniApplication`

Create:

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface MiniApplication {
}
```

Application:

```java
@MiniApplication
public class AppConfig {

    public static void main(String[] args) {
        MiniApplication.run(
            AppConfig.class,
            args
        );
    }
}
```

---

# 32. Naming Conflict

Java cannot have:

```java
@MiniApplication
public class MiniApplication
```

and also:

```java
MiniApplication.run(...)
```

unless the annotation and bootstrap class are separated.

Recommended:

```text
com.minispring.boot.annotation.MiniApplication
```

and:

```text
com.minispring.boot.MiniApplication
```

Then:

```java
import com.minispring.boot.MiniApplication;
import com.minispring.boot.annotation.MiniApplication;
```

creates a naming conflict.

Better use:

```java
@MiniBootApplication
```

for the annotation.

Then:

```java
MiniApplication.run(...)
```

for the bootstrap API.

This is cleaner:

```java
@MiniBootApplication
public class AppConfig {

    public static void main(String[] args) {
        MiniApplication.run(AppConfig.class, args);
    }
}
```

---

# 33. Phase 13 — Bean Lifecycle

Introduce:

```java
public interface InitializingBean {

    void afterPropertiesSet();
}
```

After dependency injection:

```java
if (bean instanceof InitializingBean initializingBean) {
    initializingBean.afterPropertiesSet();
}
```

Example:

```java
@Service
public class UserService
        implements InitializingBean {

    @Override
    public void afterPropertiesSet() {
        System.out.println(
            "UserService initialized"
        );
    }
}
```

Lifecycle:

```text
Constructor
    |
    v
Dependency Injection
    |
    v
afterPropertiesSet()
    |
    v
Ready
```

---

# 34. Add `@PostConstruct`

For a more Spring-like API:

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface PostConstruct {
}
```

Then:

```java
for (Method method : beanClass.getDeclaredMethods()) {

    if (method.isAnnotationPresent(PostConstruct.class)) {

        method.setAccessible(true);
        method.invoke(bean);
    }
}
```

Example:

```java
@Service
public class CacheService {

    @PostConstruct
    public void initialize() {
        System.out.println("Cache initialized");
    }
}
```

---

# 35. Lifecycle Ordering

Define and document:

```text
1. Constructor
2. Dependency injection
3. @PostConstruct
4. InitializingBean.afterPropertiesSet()
5. Bean becomes ready
```

Later add:

```text
BeanPostProcessor
```

which gives:

```text
Before initialization
        |
        v
@PostConstruct
        |
        v
afterPropertiesSet
        |
        v
After initialization
```

---

# 36. Phase 14 — BeanPostProcessor

This is an important extensibility mechanism.

```java
public interface BeanPostProcessor {

    Object postProcessBeforeInitialization(
            Object bean,
            String beanName);

    Object postProcessAfterInitialization(
            Object bean,
            String beanName);
}
```

Bean creation becomes:

```text
Instantiate
    |
    v
Inject dependencies
    |
    v
BeforeInitialization
    |
    v
@PostConstruct
    |
    v
InitializingBean
    |
    v
AfterInitialization
    |
    v
Store singleton
```

This opens the door to:

- Proxies
- Transactions
- Validation
- Logging
- Security
- AOP

---

# 37. Phase 15 — Interface-Based Injection

Consider:

```java
public interface PaymentService {
}
```

```java
@Service
public class StripePaymentService
        implements PaymentService {
}
```

Controller:

```java
@Autowired
private PaymentService paymentService;
```

Calling:

```java
getBean(PaymentService.class)
```

cannot simply search:

```java
Map<Class<?>, Object>
```

because the registered class is:

```text
StripePaymentService
```

while requested type is:

```text
PaymentService
```

---

# 38. Type Index

Maintain:

```java
Map<Class<?>, List<String>> typeIndex;
```

Example:

```text
PaymentService
    |
    +--> stripePaymentService

UserRepository
    |
    +--> userRepository
```

During registration:

```java
typeIndex
    .computeIfAbsent(interfaceType,
        key -> new ArrayList<>())
    .add(beanName);
```

When resolving:

```java
List<String> candidates =
        typeIndex.get(requiredType);
```

---

# 39. Ambiguous Dependencies

Suppose:

```java
@Service
public class StripePaymentService
        implements PaymentService {
}
```

and:

```java
@Service
public class PaypalPaymentService
        implements PaymentService {
}
```

Then:

```java
@Autowired
PaymentService paymentService;
```

has two candidates.

MiniSpring should throw:

```text
NoUniqueBeanDefinitionException

Expected one bean of type PaymentService
but found:

- stripePaymentService
- paypalPaymentService
```

Do not silently pick one.

---

# 40. Add `@Primary`

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Primary {
}
```

Example:

```java
@Service
@Primary
public class StripePaymentService
        implements PaymentService {
}
```

Resolution:

```text
Candidates
   |
   +--> StripePaymentService @Primary
   |
   +--> PaypalPaymentService
```

Choose the primary candidate.

---

# 41. Add `@Qualifier`

```java
@Target({
    ElementType.FIELD,
    ElementType.PARAMETER
})
@Retention(RetentionPolicy.RUNTIME)
public @interface Qualifier {

    String value();
}
```

Usage:

```java
@Autowired
@Qualifier("paypalPaymentService")
private PaymentService paymentService;
```

For constructor:

```java
public OrderService(
    @Qualifier("paypalPaymentService")
    PaymentService paymentService) {
}
```

Now dependency resolution becomes:

```text
Type
 +
Qualifier
 +
Primary
```

---

# 42. Phase 16 — Bean Scopes

Support:

```java
@Scope("singleton")
@Scope("prototype")
```

Annotation:

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Scope {

    String value() default "singleton";
}
```

Behavior:

```text
Singleton
---------
one instance per ApplicationContext


Prototype
---------
new instance for every getBean()
```

Implementation:

```java
if (definition.getScope()
        .equals("singleton")) {

    return singletonObjects.computeIfAbsent(
        beanName,
        this::createBean
    );
}

return createBean(beanName);
```

---

# 43. Phase 17 — Application Context API

Expose:

```java
public interface ApplicationContext
        extends BeanFactory {

    <T> List<T> getBeansOfType(Class<T> type);

    boolean containsBean(Class<?> type);

    void close();
}
```

Example:

```java
ApplicationContext context =
    MiniApplication.run(
        AppConfig.class,
        args
    );

UserService service =
    context.getBean(UserService.class);
```

---

# 44. Phase 18 — Configuration Properties

A framework becomes more useful when behavior is configurable.

Create:

```java
public class MiniSpringProperties {

    private String serverHost = "localhost";

    private int serverPort = 8080;

    private boolean webEnabled = true;

    private boolean lazyInitialization = false;
}
```

Configuration:

```text
minispring.properties
```

Example:

```properties
server.host=localhost
server.port=8080
web.enabled=true
lazy.initialization=false
```

Later support:

```text
application.properties
application.yml
environment variables
system properties
```

---

# 45. Phase 19 — Tiny MVC Layer

Now build a very small Spring MVC-like layer.

Annotations:

```java
@Controller
@RequestMapping
@GetMapping
@PostMapping
@PathVariable
@RequestParam
```

The flow:

```text
Browser
   |
   v
HTTP Server
   |
   v
DispatcherServlet
   |
   v
HandlerMapping
   |
   v
Controller Method
   |
   v
Response
```

---

# 46. `@RequestMapping`

```java
@Target({
    ElementType.TYPE,
    ElementType.METHOD
})
@Retention(RetentionPolicy.RUNTIME)
public @interface RequestMapping {

    String value() default "";

    HttpMethod method() default HttpMethod.ANY;
}
```

Enum:

```java
public enum HttpMethod {
    GET,
    POST,
    PUT,
    DELETE,
    PATCH,
    ANY
}
```

---

# 47. `@GetMapping`

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@RequestMapping
public @interface GetMapping {

    String value() default "";
}
```

Similarly:

```java
@PostMapping
@PutMapping
@DeleteMapping
```

You can implement these as specialized mapping annotations.

---

# 48. Controller Example

```java
@Controller
@RequestMapping("/users")
public class UserController {

    @GetMapping
    public String users() {
        return "all users";
    }

    @GetMapping("/{id}")
    public String user(
            @PathVariable("id") String id) {

        return "user = " + id;
    }
}
```

Routes:

```text
GET /users
GET /users/10
GET /users/25
```

---

# 49. Phase 20 — HandlerMethod

Represent a controller endpoint as:

```java
public class HandlerMethod {

    private final Object bean;

    private final Method method;

    public HandlerMethod(
            Object bean,
            Method method) {

        this.bean = bean;
        this.method = method;
    }
}
```

It represents:

```text
Controller Object
        +
Java Method
```

Example:

```text
UserController@1234
        +
getUser(String)
```

---

# 50. Phase 21 — HandlerMapping

Create:

```java
public interface HandlerMapping {

    HandlerMethod getHandler(
            HttpRequest request);
}
```

Implementation:

```java
public class AnnotationHandlerMapping
        implements HandlerMapping {

    private final Map<String, HandlerMethod>
            handlers;
}
```

Simple key:

```text
GET:/users
GET:/users/{id}
POST:/users
```

---

# 51. Discover Controller Methods

After all beans are initialized:

```java
List<Object> controllers =
        context.getBeansWithAnnotation(
            Controller.class
        );
```

For each controller:

```java
Class<?> controllerClass =
        controller.getClass();

for (Method method :
        controllerClass.getDeclaredMethods()) {

    if (method.isAnnotationPresent(
            GetMapping.class)) {

        registerHandler(...);
    }
}
```

---

# 52. Controller-Level Mapping

If:

```java
@Controller
@RequestMapping("/users")
public class UserController {
```

and:

```java
@GetMapping("/{id}")
public String getUser(...) {
```

combine:

```text
/users
+
/{id}
=
/users/{id}
```

Normalize:

```text
//
/
```

and trailing slashes consistently.

---

# 53. Phase 22 — Path Matching

Request:

```text
GET /users/123
```

Mapping:

```text
/users/{id}
```

Need to extract:

```text
id = 123
```

Convert:

```text
/users/{id}
```

to a regex:

```regex
^/users/([^/]+)$
```

Capture:

```text
123
```

Then:

```java
@PathVariable("id")
```

receives:

```text
123
```

---

# 54. Better Route Representation

Do not store routes as raw strings forever.

Create:

```java
public class Route {

    private HttpMethod method;

    private String pattern;

    private Pattern compiledPattern;

    private List<String> pathVariables;

    private HandlerMethod handler;
}
```

Example:

```text
Pattern:
^/users/([^/]+)$

Variables:
[id]

Handler:
UserController.getUser(...)
```

This makes dispatching cleaner.

---

# 55. Phase 23 — `@PathVariable`

```java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface PathVariable {

    String value();
}
```

During invocation:

```java
Parameter[] parameters =
        method.getParameters();
```

For each parameter:

```java
PathVariable annotation =
    parameter.getAnnotation(
        PathVariable.class
    );
```

Then:

```java
String value =
    pathVariables.get(
        annotation.value()
    );
```

---

# 56. Type Conversion

A path variable might be:

```java
@GetMapping("/{id}")
public String getUser(
        @PathVariable("id") Long id) {
}
```

HTTP gives:

```text
"123"
```

But method expects:

```text
Long
```

Create:

```java
ConversionService
```

Interface:

```java
public interface ConversionService {

    <T> T convert(
        String value,
        Class<T> targetType
    );
}
```

Support initially:

```text
String
Integer
Long
Boolean
Double
Float
```

---

# 57. Phase 24 — `@RequestParam`

Annotation:

```java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface RequestParam {

    String value();

    boolean required() default true;

    String defaultValue() default "";
}
```

Controller:

```java
@GetMapping("/search")
public String search(
        @RequestParam("name") String name) {

    return "search = " + name;
}
```

Request:

```text
GET /users/search?name=Arpan
```

Result:

```text
name = Arpan
```

---

# 58. Optional Request Parameters

Example:

```java
@GetMapping("/search")
public String search(
        @RequestParam(
            value = "page",
            required = false,
            defaultValue = "0"
        )
        int page) {

    return "page = " + page;
}
```

Request:

```text
/users/search
```

Result:

```text
page = 0
```

---

# 59. Phase 25 — HTTP Request Abstraction

Do not make controllers depend directly on the JDK HTTP server.

Create:

```java
public class HttpRequest {

    private HttpMethod method;

    private String path;

    private Map<String, String> queryParameters;

    private Map<String, String> headers;

    private String body;
}
```

Response:

```java
public class HttpResponse {

    private int status;

    private Map<String, String> headers;

    private byte[] body;
}
```

This abstraction lets you replace the HTTP server later.

---

# 60. Phase 26 — Embedded HTTP Server

Initially use:

```java
com.sun.net.httpserver.HttpServer
```

Example:

```java
HttpServer server =
        HttpServer.create(
            new InetSocketAddress(
                "localhost",
                8080
            ),
            0
        );
```

Register:

```java
server.createContext(
    "/",
    exchange -> {
        // dispatch
    }
);
```

Start:

```java
server.start();
```

---

# 61. DispatcherServlet

Create:

```java
public class DispatcherServlet {

    private final HandlerMapping handlerMapping;

    private final ArgumentResolverComposite
            argumentResolvers;

    public void service(
            HttpRequest request,
            HttpResponse response) {

        HandlerMethod handler =
                handlerMapping.getHandler(request);

        // resolve arguments
        // invoke method
        // create response
    }
}
```

This becomes the center of the MVC framework.

---

# 62. Complete Request Flow

Example request:

```text
GET /users/123?verbose=true
```

Flow:

```text
HTTP Server
    |
    v
HttpRequest
    |
    v
DispatcherServlet
    |
    v
HandlerMapping
    |
    v
/users/{id}
    |
    v
UserController.getUser(...)
    |
    +---- PathVariable id = 123
    |
    +---- RequestParam verbose = true
    |
    v
Method Invocation
    |
    v
"User 123"
    |
    v
HttpResponse
```

---

# 63. Phase 27 — ArgumentResolver

Avoid putting all argument resolution logic inside `DispatcherServlet`.

Create:

```java
public interface HandlerMethodArgumentResolver {

    boolean supportsParameter(
            Parameter parameter);

    Object resolveArgument(
            Parameter parameter,
            HttpRequest request,
            Map<String, String> pathVariables);
}
```

Implement:

```text
PathVariableArgumentResolver
RequestParamArgumentResolver
RequestBodyArgumentResolver
```

Architecture:

```text
DispatcherServlet
      |
      v
ArgumentResolverComposite
      |
      +--> PathVariableResolver
      |
      +--> RequestParamResolver
      |
      +--> RequestBodyResolver
```

This is much more extensible.

---

# 64. Phase 28 — Return Value Handling

A controller may return:

```java
String
```

or:

```java
User
```

or:

```java
ResponseEntity<User>
```

Start with:

```text
String -> text/plain
```

Then add:

```text
Object -> JSON
```

Create:

```java
public interface HandlerMethodReturnValueHandler {

    boolean supportsReturnType(
            Class<?> returnType);

    void handle(
            Object value,
            HttpResponse response);
}
```

---

# 65. Simple JSON Support

Do not initially build a full JSON serializer.

You can either:

1. Add Jackson as a dependency.
2. Build a deliberately tiny serializer for learning.

Recommended:

```text
MiniSpring core
    -> no Jackson dependency

MiniSpring web
    -> optional Jackson module
```

Then:

```java
User user = ...;
```

becomes:

```json
{
  "id": 10,
  "name": "Arpan"
}
```

---

# 66. Phase 29 — Exception Handling

Controller can throw:

```java
throw new UserNotFoundException();
```

Dispatcher should not leak stack traces to clients.

Create:

```java
public class HandlerExceptionResolver {

    public HttpResponse resolve(
            Exception exception) {

        ...
    }
}
```

Later support:

```java
@ExceptionHandler
```

Example:

```java
@ExceptionHandler(UserNotFoundException.class)
public ResponseEntity<?> handle(...) {
}
```

---

# 67. Phase 30 — HTTP Status Codes

Support:

```text
200 OK
201 CREATED
204 NO_CONTENT
400 BAD_REQUEST
404 NOT_FOUND
405 METHOD_NOT_ALLOWED
500 INTERNAL_SERVER_ERROR
```

For example:

```java
return ResponseEntity.status(201)
        .body(user);
```

---

# 68. Phase 31 — `ResponseEntity`

Create:

```java
public class ResponseEntity<T> {

    private final int status;

    private final Map<String, String> headers;

    private final T body;

    public static <T> ResponseEntity<T> ok(T body) {
        return new ResponseEntity<>(
            200,
            Map.of(),
            body
        );
    }
}
```

Later:

```java
ResponseEntity.created(...)
ResponseEntity.badRequest(...)
ResponseEntity.notFound(...)
```

---

# 69. Phase 32 — Dependency Injection + MVC Integration

At startup:

```text
MiniApplication.run()
        |
        v
ApplicationContext
        |
        +---- scan components
        |
        +---- create beans
        |
        +---- inject dependencies
        |
        +---- initialize lifecycle
        |
        v
Find @Controller beans
        |
        v
Build HandlerMapping
        |
        v
Start Embedded Server
```

This is the point where MiniSpring starts feeling like a real framework.

---

# 70. Phase 33 — `MiniApplication.run`

Create:

```java
public final class MiniApplication {

    public static ApplicationContext run(
            Class<?> primarySource,
            String[] args) {

        ApplicationContext context =
            new AnnotationConfigApplicationContext(
                primarySource
            );

        WebApplicationContext webContext =
            new WebApplicationContext(context);

        webContext.start();

        return context;
    }
}
```

Application:

```java
@MiniBootApplication
public class AppConfig {

    public static void main(String[] args) {

        MiniApplication.run(
            AppConfig.class,
            args
        );
    }
}
```

---

# 71. Startup Timeline

Expected console output:

```text
=====================================
 MiniSpring
=====================================

Starting application...

Base package:
com.example.app

Scanning components...

Found:
 - UserController
 - UserService
 - UserRepository

Registering bean definitions...

Creating singleton beans...

Created:
 - userRepository
 - userService
 - userController

Building MVC mappings...

GET  /users
GET  /users/{id}
POST /users

Starting HTTP server...

MiniSpring started on:
http://localhost:8080
```

---

# 72. Full Demo Application

## Repository

```java
@Repository
public class UserRepository {

    public String findById(Long id) {
        return "User-" + id;
    }
}
```

## Service

```java
@Service
public class UserService {

    private final UserRepository repository;

    @Autowired
    public UserService(
            UserRepository repository) {

        this.repository = repository;
    }

    public String getUser(Long id) {
        return repository.findById(id);
    }
}
```

## Controller

```java
@Controller
@RequestMapping("/users")
public class UserController {

    private final UserService service;

    @Autowired
    public UserController(
            UserService service) {

        this.service = service;
    }

    @GetMapping("/{id}")
    public String getUser(
            @PathVariable("id") Long id) {

        return service.getUser(id);
    }
}
```

---

# 73. Start the Application

```java
@MiniBootApplication
public class AppConfig {

    public static void main(String[] args) {

        MiniApplication.run(
            AppConfig.class,
            args
        );
    }
}
```

Then:

```bash
curl http://localhost:8080/users/10
```

Expected:

```text
User-10
```

---

# 74. Complete MiniSpring Architecture

At this point the project should look like:

```text
                         MiniApplication.run()
                                  |
                                  v
                       +----------------------+
                       | ApplicationContext   |
                       +----------+-----------+
                                  |
             +--------------------+--------------------+
             |                    |                    |
             v                    v                    v
      ComponentScanner      BeanDefinitionRegistry  Lifecycle
             |                    |                    |
             +--------------------+--------------------+
                                  |
                                  v
                           BeanFactory
                                  |
                 +----------------+----------------+
                 |                |                |
                 v                v                v
           Constructor       Field Injection   Bean Lifecycle
           Injection
                 |
                 v
              Beans
                 |
                 v
            @Controller
                 |
                 v
          HandlerMapping
                 |
                 v
         DispatcherServlet
                 |
                 v
         ArgumentResolvers
                 |
                 v
          Return Handlers
                 |
                 v
          Embedded Server
```

---

# 75. Recommended Package Structure

```text
com.minispring
│
├── core
│   │
│   ├── annotation
│   │   ├── Component.java
│   │   ├── Service.java
│   │   ├── Repository.java
│   │   ├── Autowired.java
│   │   ├── Scope.java
│   │   ├── Primary.java
│   │   └── Qualifier.java
│   │
│   ├── bean
│   │   ├── BeanDefinition.java
│   │   ├── BeanDefinitionRegistry.java
│   │   └── Scope.java
│   │
│   ├── factory
│   │   ├── BeanFactory.java
│   │   └── DefaultBeanFactory.java
│   │
│   ├── context
│   │   ├── ApplicationContext.java
│   │   └── AnnotationConfigApplicationContext.java
│   │
│   ├── scanner
│   │   ├── ClassPathScanner.java
│   │   ├── DefaultClassPathScanner.java
│   │   ├── ResourceResolver.java
│   │   ├── FileResourceResolver.java
│   │   └── JarResourceResolver.java
│   │
│   ├── lifecycle
│   │   ├── InitializingBean.java
│   │   ├── BeanPostProcessor.java
│   │   └── LifecycleProcessor.java
│   │
│   └── exception
│       ├── MiniSpringException.java
│       ├── NoSuchBeanDefinitionException.java
│       ├── NoUniqueBeanDefinitionException.java
│       └── CircularDependencyException.java
│
├── web
│   │
│   ├── annotation
│   │   ├── Controller.java
│   │   ├── RequestMapping.java
│   │   ├── GetMapping.java
│   │   ├── PostMapping.java
│   │   ├── PathVariable.java
│   │   └── RequestParam.java
│   │
│   ├── handler
│   │   ├── HandlerMethod.java
│   │   ├── HandlerMapping.java
│   │   └── AnnotationHandlerMapping.java
│   │
│   ├── argument
│   │   ├── HandlerMethodArgumentResolver.java
│   │   ├── ArgumentResolverComposite.java
│   │   ├── PathVariableArgumentResolver.java
│   │   └── RequestParamArgumentResolver.java
│   │
│   ├── converter
│   │   └── ConversionService.java
│   │
│   ├── dispatcher
│   │   └── DispatcherServlet.java
│   │
│   └── server
│       └── MiniHttpServer.java
│
└── boot
    ├── MiniApplication.java
    └── MiniBootApplication.java
```

---

# 76. Important Design Principle — Separate Responsibilities

Avoid creating one giant class such as:

```java
MiniSpringApplication.java
```

containing:

```text
scan classes
create objects
inject dependencies
resolve routes
start server
parse HTTP
serialize JSON
```

That becomes impossible to extend.

Instead:

```text
ClassPathScanner
       |
BeanDefinitionRegistry
       |
BeanFactory
       |
ApplicationContext
       |
HandlerMapping
       |
DispatcherServlet
       |
HttpServer
```

Each component should have one primary responsibility.

---

# 77. SOLID Mapping

## Single Responsibility

```text
Scanner
 -> scanning

BeanFactory
 -> object creation

ApplicationContext
 -> application lifecycle

HandlerMapping
 -> route mapping

DispatcherServlet
 -> request dispatch
```

## Open/Closed

Adding:

```text
@PutMapping
```

should not require rewriting the entire dispatcher.

Adding:

```text
@RequestBody
```

should mean adding:

```text
RequestBodyArgumentResolver
```

## Liskov Substitution

```java
ClassPathScanner scanner;
```

can use:

```text
DefaultClassPathScanner
JarClassPathScanner
TestClassPathScanner
```

## Interface Segregation

Prefer:

```text
BeanFactory
ApplicationContext
HandlerMapping
ArgumentResolver
```

rather than one enormous framework interface.

## Dependency Inversion

The dispatcher should depend on:

```java
HandlerMapping
```

not:

```java
AnnotationHandlerMapping
```

---

# 78. Dependency Graph

Represent dependencies internally as a graph.

Example:

```text
UserController
       |
       v
UserService
       |
       v
UserRepository
```

Graph:

```text
UserController -> UserService
UserService    -> UserRepository
```

Circular:

```text
A -> B
B -> C
C -> A
```

This can be detected using DFS.

---

# 79. Explicit Graph-Based Circular Dependency Detection

DFS state:

```text
WHITE = not visited
GRAY  = currently visiting
BLACK = completed
```

Algorithm:

```text
visit(node):

    if node == GRAY:
        cycle detected

    if node == BLACK:
        return

    mark node GRAY

    for dependency:
        visit(dependency)

    mark node BLACK
```

This can be useful before actual bean creation.

---

# 80. Improve Startup Performance

Initial implementation:

```text
Every getBean()
    -> reflection
    -> annotation lookup
    -> constructor lookup
    -> field lookup
```

Better:

At startup calculate metadata:

```java
BeanMetadata
```

containing:

```text
selected constructor
injection fields
injection parameters
@PostConstruct methods
scope
qualifiers
```

Then runtime creation becomes much faster.

---

# 81. Reflection Metadata Cache

Example:

```java
Map<Class<?>, BeanMetadata>
        metadataCache;
```

Store:

```java
public class BeanMetadata {

    private Constructor<?> constructor;

    private List<Field> autowiredFields;

    private List<Method> postConstructMethods;
}
```

Now:

```text
Reflection
    |
    v
Metadata
    |
    v
Cache
    |
    v
Fast bean creation
```

---

# 82. Thread Safety

A web framework is concurrent.

Multiple HTTP requests may call:

```java
getBean(...)
```

simultaneously.

Singleton creation must be safe.

Avoid:

```java
if (!map.containsKey(name)) {
    map.put(name, createBean());
}
```

because two threads can create two instances.

Use:

```java
ConcurrentHashMap
```

and controlled creation.

However, be careful with:

```java
computeIfAbsent()
```

when bean creation recursively requests another bean. A dedicated creation mechanism is often easier to reason about.

---

# 83. Application Startup Thread Safety

Application startup should normally happen once:

```text
ApplicationContext
       |
       v
refresh()
       |
       v
initialize
       |
       v
RUNNING
```

Use lifecycle state:

```java
enum ContextState {
    NEW,
    STARTING,
    RUNNING,
    STOPPING,
    STOPPED
}
```

This gives better lifecycle control.

---

# 84. Phase 34 — Events

Add:

```java
ApplicationEvent
```

and:

```java
ApplicationEventPublisher
```

Example:

```java
context.publishEvent(
    new UserCreatedEvent(user)
);
```

Listener:

```java
@EventListener
public void handleUserCreated(
        UserCreatedEvent event) {
}
```

Architecture:

```text
ApplicationContext
        |
        v
EventPublisher
        |
        v
ListenerRegistry
        |
        v
@EventListener methods
```

This introduces another important framework concept.

---

# 85. Phase 35 — Profiles

Eventually support:

```java
@Profile("dev")
```

```java
@Profile("prod")
```

Configuration:

```properties
minispring.profile=dev
```

Scanner:

```text
@Service
@Profile("prod")
```

should only register the component when:

```text
activeProfile == prod
```

---

# 86. Phase 36 — Conditional Beans

Introduce:

```java
@Conditional
```

Example:

```java
@Conditional(OnDatabaseAvailable.class)
@Repository
public class DatabaseUserRepository {
}
```

The condition can inspect:

```text
environment
class path
properties
existing beans
```

This is a major step toward auto-configuration.

---

# 87. Phase 37 — Auto Configuration

Create:

```text
META-INF/minispring.factories
```

or a custom metadata file.

Example:

```properties
com.example.config.DatabaseAutoConfiguration
com.example.config.WebAutoConfiguration
```

Startup:

```text
MiniApplication
      |
      v
Load AutoConfigurations
      |
      v
Evaluate Conditions
      |
      v
Register Beans
```

This is conceptually similar to how Spring Boot turns a collection of framework libraries into an opinionated application platform.

---

# 88. Phase 38 — `@Bean`

Support configuration classes:

```java
@Configuration
public class AppConfig {

    @Bean
    public UserService userService(
            UserRepository repository) {

        return new UserService(repository);
    }
}
```

Now MiniSpring has two ways to create beans:

```text
@Component
@Service
@Repository
@Controller
```

and:

```text
@Bean
```

---

# 89. Configuration Method Processing

Scan:

```java
@Configuration
```

Then find:

```java
@Bean
```

methods.

Create:

```java
BeanDefinition
```

for each method.

Example:

```text
@Bean
DataSource dataSource()
```

becomes:

```text
BeanDefinition
name = dataSource
factoryMethod = dataSource()
returnType = DataSource
```

This is a significant extension because bean creation is no longer always:

```text
Class -> Constructor
```

It can also be:

```text
Configuration object -> Factory method
```

---

# 90. Phase 39 — FactoryBean Concept

A more advanced concept:

```java
FactoryBean<T>
```

allows a bean to produce another object.

Example:

```text
RepositoryFactoryBean
       |
       v
UserRepository proxy
```

This is a useful stepping stone toward Spring Data-style functionality.

---

# 91. Phase 40 — Proxy Support

Eventually implement:

```text
java.lang.reflect.Proxy
```

for interface-based proxies.

Example:

```java
public interface UserService {
    String getUser(Long id);
}
```

Create:

```java
Proxy.newProxyInstance(...)
```

to intercept:

```text
before method
    |
actual method
    |
after method
```

---

# 92. AOP-Like Interceptor

Create:

```java
public interface MethodInterceptor {

    Object invoke(
        Object target,
        Method method,
        Object[] args
    ) throws Throwable;
}
```

Then:

```text
Controller
   |
   v
Proxy
   |
   +--> LoggingInterceptor
   |
   +--> SecurityInterceptor
   |
   +--> Target Method
```

This demonstrates the core idea behind many framework proxy mechanisms.

---

# 93. `@Transactional` as a Future Feature

Eventually:

```java
@Transactional
public void transferMoney(...) {
}
```

could be intercepted:

```text
BEGIN TRANSACTION
        |
        v
method()
        |
        +--> success -> COMMIT
        |
        +--> exception -> ROLLBACK
```

This is a good advanced MiniSpring exercise.

---

# 94. Testing Strategy

Do not only test the final web application.

Test each layer independently.

## Scanner Tests

```text
Find @Component
Find @Service
Find @Repository
Find @Controller
Ignore interfaces
Ignore abstract classes
```

## BeanFactory Tests

```text
Create bean
Singleton behavior
Prototype behavior
Constructor injection
Field injection
Missing dependency
Circular dependency
Multiple candidates
Qualifier
Primary
```

## Context Tests

```text
Startup
Bean registration
Lifecycle
PostConstruct
Context close
```

## MVC Tests

```text
GET mapping
POST mapping
Path variable
Request parameter
Type conversion
404
405
Controller invocation
```

---

# 95. Example Unit Test — Singleton

```java
@Test
void shouldReturnSameSingleton() {

    UserService first =
        context.getBean(UserService.class);

    UserService second =
        context.getBean(UserService.class);

    assertSame(first, second);
}
```

---

# 96. Example Unit Test — Constructor Injection

```java
@Test
void shouldInjectRepository() {

    UserService service =
        context.getBean(UserService.class);

    assertNotNull(service);
}
```

Prefer testing behavior rather than private implementation details.

---

# 97. Example Circular Dependency Test

```java
@Test
void shouldDetectCircularDependency() {

    assertThrows(
        CircularDependencyException.class,
        () -> context.getBean(A.class)
    );
}
```

---

# 98. MVC Integration Test

Start server on a random port:

```java
int port = findFreePort();
```

Call:

```bash
curl http://localhost:<port>/users/10
```

Assert:

```text
HTTP 200

Body:
User-10
```

Also test:

```text
/users/abc
```

if the controller expects `Long`.

Expected:

```text
400 BAD_REQUEST
```

---

# 99. Test Matrix

Create a table like:

| Feature | Unit Test | Integration Test |
|---|---:|---:|
| Component scan | Yes | Yes |
| Constructor injection | Yes | Yes |
| Field injection | Yes | Yes |
| Singleton | Yes | Yes |
| Prototype | Yes | Yes |
| Circular dependency | Yes | Yes |
| Qualifier | Yes | Yes |
| Primary | Yes | Yes |
| GET mapping | Yes | Yes |
| POST mapping | Yes | Yes |
| PathVariable | Yes | Yes |
| RequestParam | Yes | Yes |
| Type conversion | Yes | Yes |
| 404 | Yes | Yes |
| JSON | Yes | Yes |

---

# 100. Common Failure Scenarios

MiniSpring should provide clear errors.

## Bean Not Found

```text
NoSuchBeanDefinitionException:

No bean of type:
com.example.UserService

Available beans:
userRepository
orderService
```

## Multiple Beans

```text
NoUniqueBeanDefinitionException:

Required:
PaymentService

Candidates:
stripePaymentService
paypalPaymentService

Use @Primary or @Qualifier.
```

## Circular Dependency

```text
CircularDependencyException:

userService
 -> orderService
 -> paymentService
 -> userService
```

## Invalid Controller

```text
InvalidHandlerMethodException:

GET /users/{id}

Method:
getUser()

requires @PathVariable("id")
```

Good errors dramatically improve framework usability.

---

# 101. Recommended Implementation Order

Do not attempt everything at once.

Implement in this exact progression:

```text
STEP 1
Create Maven project
        |
STEP 2
Create annotations
        |
STEP 3
Build component scanner
        |
STEP 4
Build BeanDefinition
        |
STEP 5
Build BeanDefinitionRegistry
        |
STEP 6
Build BeanFactory
        |
STEP 7
Implement constructor injection
        |
STEP 8
Implement field injection
        |
STEP 9
Implement singleton cache
        |
STEP 10
Implement circular dependency detection
        |
STEP 11
Build ApplicationContext
        |
STEP 12
Add lifecycle callbacks
        |
STEP 13
Add qualifiers / primary
        |
STEP 14
Add scopes
        |
STEP 15
Build @Controller
        |
STEP 16
Build @RequestMapping
        |
STEP 17
Build HandlerMapping
        |
STEP 18
Build DispatcherServlet
        |
STEP 19
Build argument resolvers
        |
STEP 20
Add embedded HTTP server
        |
STEP 21
Implement MiniApplication.run()
        |
STEP 22
Add JSON
        |
STEP 23
Add exception handling
        |
STEP 24
Add events
        |
STEP 25
Add @Bean
        |
STEP 26
Add configuration properties
        |
STEP 27
Add profiles / conditions
        |
STEP 28
Add proxies / AOP
```

---

# 102. Suggested Git Commit Strategy

Build the project incrementally.

```text
commit 01:
Initialize MiniSpring Maven project

commit 02:
Add component annotations

commit 03:
Implement classpath scanner

commit 04:
Add BeanDefinition

commit 05:
Implement BeanDefinitionRegistry

commit 06:
Implement BeanFactory

commit 07:
Implement constructor injection

commit 08:
Implement field injection

commit 09:
Add singleton scope

commit 10:
Add circular dependency detection

commit 11:
Implement ApplicationContext

commit 12:
Add bean lifecycle

commit 13:
Add qualifiers and primary

commit 14:
Add prototype scope

commit 15:
Add controller annotations

commit 16:
Implement HandlerMapping

commit 17:
Implement DispatcherServlet

commit 18:
Implement PathVariable

commit 19:
Implement RequestParam

commit 20:
Add embedded HTTP server

commit 21:
Implement MiniApplication.run

commit 22:
Add JSON response support

commit 23:
Add exception handling

commit 24:
Add integration tests
```

This makes the project useful as an interview-learning repository because every commit represents one framework concept.

---

# 103. Interview Questions You Should Be Able to Answer

After completing MiniSpring, you should be able to explain:

### Dependency Injection

- What is dependency injection?
- Why use constructor injection?
- How does a DI container instantiate a class?
- How does it discover dependencies?
- How does it resolve interface dependencies?
- What happens when two beans implement the same interface?

### Reflection

- What is `Class<?>`?
- How do you inspect annotations?
- How do you invoke a constructor dynamically?
- How do you set a private field?
- What are the costs and limitations of reflection?

### Classpath Scanning

- How does Spring find components?
- What is the classpath?
- What is the difference between directory and JAR scanning?
- How would you scan nested packages?

### Bean Lifecycle

- What is a BeanDefinition?
- What is a singleton bean?
- When should an object be instantiated?
- What is a BeanPostProcessor?
- Why is lifecycle processing separate from construction?

### Circular Dependency

- How can a DI container detect cycles?
- Why are constructor cycles difficult?
- How can a graph algorithm detect dependency cycles?
- Why can some containers support certain field-injection cycles?

### Spring MVC

- What is DispatcherServlet?
- What is HandlerMapping?
- How does `/users/{id}` match `/users/10`?
- How are controller method arguments resolved?
- How does the framework convert `"10"` into `Long 10`?

### Spring Boot

- What does `SpringApplication.run()` conceptually do?
- How does component scanning begin?
- How does auto-configuration work conceptually?
- Why are configuration properties useful?

---

# 104. Final Architecture

The finished project should conceptually look like this:

```text
                        ┌─────────────────────────┐
                        │     Application         │
                        │                         │
                        │ @Service                │
                        │ @Repository             │
                        │ @Controller             │
                        └────────────┬────────────┘
                                     │
                                     v
                        ┌─────────────────────────┐
                        │ MiniApplication.run()   │
                        └────────────┬────────────┘
                                     │
                                     v
                        ┌─────────────────────────┐
                        │ ApplicationContext      │
                        └────────────┬────────────┘
                                     │
                 ┌───────────────────┼───────────────────┐
                 │                   │                   │
                 v                   v                   v
        ┌────────────────┐  ┌─────────────────┐  ┌───────────────┐
        │ Classpath      │  │ BeanDefinition  │  │ Lifecycle     │
        │ Scanner        │  │ Registry        │  │ Processor     │
        └───────┬────────┘  └────────┬────────┘  └───────────────┘
                │                    │
                └──────────┬─────────┘
                           v
                 ┌─────────────────────┐
                 │     BeanFactory     │
                 └──────────┬──────────┘
                            │
          ┌─────────────────┼──────────────────┐
          │                 │                  │
          v                 v                  v
   Constructor DI      Field DI          Bean Scopes
          │                 │                  │
          └─────────────────┼──────────────────┘
                            v
                       Application
                          Beans
                            │
                            v
                     ┌──────────────┐
                     │ Controllers  │
                     └──────┬───────┘
                            │
                            v
                     HandlerMapping
                            │
                            v
                    DispatcherServlet
                            │
                ┌───────────┼────────────┐
                │           │            │
                v           v            v
          PathVariable  RequestParam  RequestBody
                │           │            │
                └───────────┼────────────┘
                            v
                    Controller Method
                            │
                            v
                    Return Value Handler
                            │
                            v
                     HTTP Response
```

---

# 105. Definition of Done

MiniSpring V1 is complete when the following application works without Spring:

```java
@MiniBootApplication
public class AppConfig {

    public static void main(String[] args) {

        MiniApplication.run(
            AppConfig.class,
            args
        );
    }
}
```

```java
@Repository
public class UserRepository {

    public User findById(Long id) {
        return new User(id, "Arpan");
    }
}
```

```java
@Service
public class UserService {

    private final UserRepository repository;

    public UserService(
            UserRepository repository) {

        this.repository = repository;
    }

    public User find(Long id) {
        return repository.findById(id);
    }
}
```

```java
@Controller
@RequestMapping("/users")
public class UserController {

    private final UserService service;

    public UserController(
            UserService service) {

        this.service = service;
    }

    @GetMapping("/{id}")
    public User getUser(
            @PathVariable("id") Long id) {

        return service.find(id);
    }
}
```

And:

```bash
curl http://localhost:8080/users/10
```

returns:

```json
{
  "id": 10,
  "name": "Arpan"
}
```

At that point you have implemented the essential concepts behind:

```text
Spring Core
       +
Spring Context
       +
a small part of Spring MVC
       +
a small part of Spring Boot
```

---

# 106. Suggested V2 Enhancements

After V1, evolve MiniSpring toward a more realistic framework.

## Container

- `@Bean`
- `@Configuration`
- Factory methods
- Prototype scope
- Request scope
- Lazy beans
- Bean aliases
- `ObjectProvider`
- Provider-style lazy lookup
- Conditional beans
- Profiles
- Environment abstraction
- Property sources
- Configuration binding

## Dependency Injection

- `@Qualifier`
- `@Primary`
- Generic type resolution
- Collection injection
- `List<T>`
- `Set<T>`
- `Map<String, T>`
- Optional dependencies
- `Optional<T>`

## Lifecycle

- `BeanPostProcessor`
- `BeanFactoryPostProcessor`
- `SmartInitializingSingleton`
- Destroy callbacks
- `@PreDestroy`

## Web

- `@RequestBody`
- `@ResponseBody`
- `@ResponseStatus`
- `@ExceptionHandler`
- `@ControllerAdvice`
- JSON serialization
- Content negotiation
- Headers
- Cookies
- Multipart upload
- Interceptors
- Filters

## Infrastructure

- Logging
- Metrics
- Health endpoint
- Graceful shutdown
- Thread pools
- Async execution
- Configuration refresh

## AOP

- JDK proxies
- CGLIB-like subclass proxies
- Method interceptors
- Pointcuts
- Advice
- `@Transactional`
- `@Cacheable`
- `@Async`

---

# 107. Suggested V3 — MiniSpring Boot

Once the core framework is stable, create a separate module:

```text
minispring-boot
```

Its responsibility should be:

```text
configuration
+
auto configuration
+
embedded server
+
startup lifecycle
+
application defaults
```

Then the user application should need only:

```java
@MiniBootApplication
public class Application {

    public static void main(String[] args) {
        MiniApplication.run(
            Application.class,
            args
        );
    }
}
```

This gives you the architectural distinction:

```text
MiniSpring Core
    =
Framework infrastructure


MiniSpring Boot
    =
Opinionated application bootstrap
```

---

# 108. Final Learning Path

The most important thing is to build the framework in layers rather than trying to copy Spring all at once.

```text
                 ┌─────────────────────┐
                 │  MiniSpring Boot    │
                 │  run()              │
                 └──────────┬──────────┘
                            │
                 ┌──────────▼──────────┐
                 │ MiniSpring Web      │
                 │ MVC / HTTP          │
                 └──────────┬──────────┘
                            │
                 ┌──────────▼──────────┐
                 │ ApplicationContext  │
                 └──────────┬──────────┘
                            │
                 ┌──────────▼──────────┐
                 │ BeanFactory         │
                 └──────────┬──────────┘
                            │
                 ┌──────────▼──────────┐
                 │ Dependency Injection│
                 └──────────┬──────────┘
                            │
                 ┌──────────▼──────────┐
                 │ BeanDefinition      │
                 └──────────┬──────────┘
                            │
                 ┌──────────▼──────────┐
                 │ Component Scanner   │
                 └──────────┬──────────┘
                            │
                 ┌──────────▼──────────┐
                 │ Java Reflection     │
                 └─────────────────────┘
```

The core lesson is:

> A framework is primarily a collection of abstractions, metadata, lifecycle rules, registries, and extension points built around reflection and runtime orchestration.

Once MiniSpring V1 works, the next step is not to add random features. Instead, identify where the current implementation violates separation of concerns and introduce the abstractions that make the next feature possible without rewriting existing code.

That iterative process is the most valuable part of the exercise.
