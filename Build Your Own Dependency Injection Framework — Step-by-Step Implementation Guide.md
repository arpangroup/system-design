# Build Your Own Dependency Injection Framework
## Step-by-Step System Design & Implementation Guide (A Mini-Spring in Java)

> **Goal:** Design and implement a small but real Dependency Injection / IoC container from scratch in plain Java — no Spring, no external libraries — that supports `ApplicationContext`, component scanning, bean creation, constructor/field injection, and a minimal MVC layer with `@Controller`, `@GetMapping`, `@PostMapping`, `@PathVariable`, and `@RequestParam`.
>
> The purpose is not to replace Spring, but to **understand how Spring works internally** by rebuilding its core mechanics one layer at a time: annotations → reflection → bean definitions → bean factory → application context → routing → embedded HTTP server.

---

# 1. What Are We Building?

We are building **MiniSpring** — a stripped-down clone of the Spring Framework core (`spring-core` + `spring-context` + a sliver of `spring-webmvc`). By the end of this guide you will have:

- A `@Component` / `@Service` / `@Repository` / `@Controller` stereotype system.
- Classpath **component scanning** that discovers annotated classes.
- A **BeanFactory** that instantiates objects via reflection and wires their dependencies.
- An **ApplicationContext** that ties scanning + bean factory + lifecycle together, exposing `getBean(...)`.
- `@Autowired` constructor and field injection, with circular-dependency detection.
- A tiny **MVC layer**: `@RequestMapping`, `@GetMapping`, `@PostMapping`, `@PathVariable`, `@RequestParam`, dispatched over a real embedded HTTP server.
- A single entry point, `MiniApplication.run(AppConfig.class, args)`, that mirrors `SpringApplication.run(...)`.

```text
@Component classes  ---->  Component Scanner  ---->  BeanDefinitions
                                                          |
                                                          v
                                                     Bean Factory
                                                (reflection + DI wiring)
                                                          |
                                                          v
                                                  ApplicationContext
                                                          |
                                     +--------------------+--------------------+
                                     |                                         |
                              getBean(Class)                          Handler Mapping (@Controller)
                                                                                |
                                                                                v
                                                                    Embedded HTTP Server
                                                                    (routes requests to methods)
```

---

# 2. The Interview / Design Question

> **"Design and implement a lightweight Dependency Injection framework, similar in spirit to Spring, that can scan a package for components, create and wire beans, and expose a simple web controller layer with routing annotations. Explain the internal mechanics: how does `ApplicationContext` know which classes to instantiate? How does `@Autowired` actually inject a value? How does `@GetMapping` get invoked when an HTTP request arrives?"**

This is a common **"build X from scratch"** systems/backend interview question because it tests:

- Reflection API fluency (`Class`, `Constructor`, `Field`, `Method`, annotations).
- Object graph construction and cycle detection (a graph/topological-sort problem in disguise).
- Understanding of IoC, DI, and the Inversion of Control principle — not just "how to use `@Autowired`" but *why* it exists.
- Basic web server / front-controller pattern knowledge (how any MVC framework routes a request to a method).
- API design: how do you expose a clean, minimal public surface (`ApplicationContext`, `MiniApplication.run`) while hiding internal wiring complexity?

---

# 3. Why Build This From Scratch?

Most engineers use `@Autowired` and `@GetMapping` for years without knowing what happens underneath. Rebuilding a miniature version forces you to answer:

| Question | Where the answer lives in this guide |
|---|---|
| How does Spring find my `@Component` classes without me registering them manually? | [§15 — Classpath Component Scanning](#15-step-3-classpath-component-scanning) |
| Why does Spring sometimes throw `NoSuchBeanDefinitionException` at *startup*, not at first use? | [§24 — ApplicationContext](#24-step-5-the-applicationcontext) |
| How does `@Autowired` decide *which* dependency instance to hand you? | [§19 / §20 — Constructor & Field Injection](#19-constructor-injection) |
| Why do circular `@Autowired` constructors fail but circular field injection sometimes works? | [§22 — Circular Dependency Detection](#22-circular-dependency-detection) |
| How does a URL path like `/orders/42` get routed to `OrderController.getOrder(42)`? | [§27–§31 — Web Layer](#27-step-6-the-web-layer--handler-mapping) |

---

# 4. Core Concepts: IoC and Dependency Injection

**Inversion of Control (IoC):** instead of a class creating its own dependencies (`new UserRepository()`), control is inverted — a *container* creates the dependency and hands it to the class. The class no longer controls *how* its dependency is built, only *that* it needs one.

**Dependency Injection (DI):** the specific technique used to achieve IoC — dependencies are "injected" via constructor, field, or setter, rather than looked up or constructed manually.

```java
// Without DI — the class controls its own dependency (tight coupling, hard to test)
public class OrderService {
    private final OrderRepository repository = new JdbcOrderRepository();
}

// With DI — the container controls creation; OrderService only declares a need
public class OrderService {
    private final OrderRepository repository;

    public OrderService(OrderRepository repository) { // injected by the container
        this.repository = repository;
    }
}
```

The container we are building is exactly the thing that performs that injection.

---

# 5. Spring Concepts We Are Re-Creating

| Spring Concept | Our MiniSpring Equivalent |
|---|---|
| `ApplicationContext` | `MiniApplicationContext` |
| `BeanFactory` | `DefaultBeanFactory` |
| `BeanDefinition` | `BeanDefinition` |
| `ClassPathBeanDefinitionScanner` | `ComponentScanner` |
| `@Component` / `@Service` / `@Repository` | identical annotations, hand-rolled |
| `@Controller` | identical, hand-rolled |
| `@Autowired` | identical, hand-rolled |
| `@RequestMapping` / `@GetMapping` / `@PostMapping` | identical, hand-rolled |
| `DispatcherServlet` | `FrontController` |
| `SpringApplication.run()` | `MiniApplication.run()` |
| Embedded Tomcat | JDK's built-in `com.sun.net.httpserver.HttpServer` |

---

# 6. High-Level Architecture

```text
                         +---------------------------+
                         |      MiniApplication       |
                         |  .run(AppConfig.class)     |
                         +--------------+--------------+
                                        |
                                        v
                         +---------------------------+
                         |   MiniApplicationContext    |
                         |  (implements ApplicationCtx)|
                         +--------------+--------------+
             scan()  --> |  ComponentScanner            |
                         +--------------+--------------+
                                        |  List<BeanDefinition>
                                        v
                         +---------------------------+
                         |     DefaultBeanFactory      |
                         |  create + wire beans        |
                         +--------------+--------------+
                                        |  singleton bean cache
                                        v
              +-------------------------+-------------------------+
              |                                                    |
      getBean(Class<T>)                                  HandlerMapping (scans @Controller
      (used by app code)                                  beans, builds route table)
                                                                    |
                                                                    v
                                                       +---------------------------+
                                                       |      FrontController        |
                                                       |  (HttpHandler)               |
                                                       +--------------+--------------+
                                                                      |
                                                                      v
                                                       +---------------------------+
                                                       |  com.sun.net.httpserver     |
                                                       |        HttpServer           |
                                                       +---------------------------+
```

---

# 7. Project Structure

```text
minispring/
├── annotation/
│   ├── Component.java
│   ├── Service.java
│   ├── Repository.java
│   ├── Controller.java
│   ├── Autowired.java
│   ├── RequestMapping.java
│   ├── GetMapping.java
│   ├── PostMapping.java
│   ├── PathVariable.java
│   └── RequestParam.java
├── context/
│   ├── ApplicationContext.java
│   ├── MiniApplicationContext.java
│   ├── BeanDefinition.java
│   ├── ComponentScanner.java
│   ├── DefaultBeanFactory.java
│   └── BeanCurrentlyInCreationException.java
├── web/
│   ├── HandlerMapping.java
│   ├── HandlerMethod.java
│   ├── FrontController.java
│   └── HttpMethod.java
├── MiniApplication.java
└── demo/
    ├── AppConfig.java
    ├── UserRepository.java
    ├── UserService.java
    └── UserController.java
```

---

# 8. Step 1: Custom Annotations

Everything starts with metadata. Annotations are just markers read via reflection at startup — they carry no behavior themselves.

```java
// annotation/Component.java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Component {
    String value() default ""; // optional explicit bean name
}
```

`RetentionPolicy.RUNTIME` is the detail that makes everything else possible — without it, the annotation is erased at compile time and reflection could never see it.

---

# 9. @Component, @Service, @Repository

Spring uses **meta-annotations**: `@Service` and `@Repository` are themselves annotated with `@Component`, so a scanner that only understands `@Component` can still find them if it also checks for annotations *on* the annotations.

```java
// annotation/Component.java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Component {
    String value() default "";
}

// annotation/Service.java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Component // meta-annotation: a @Service IS-A @Component
public @interface Service {
    String value() default "";
}

// annotation/Repository.java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Component
public @interface Repository {
    String value() default "";
}
```

To make the scanner treat `@Service`/`@Repository` as stereotypes too, it must check `annotation.getAnnotation(Component.class) != null`, not just `class.isAnnotationPresent(Component.class)` directly (see [§16](#16-scanning-algorithm-walkthrough)).

---

# 10. @Controller

```java
// annotation/Controller.java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Component // a Controller is also a bean, just one the web layer treats specially
public @interface Controller {
    String value() default "";
}
```

---

# 11. @Autowired

```java
// annotation/Autowired.java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.CONSTRUCTOR, ElementType.FIELD})
public @interface Autowired {
    boolean required() default true;
}
```

Allowing it on both constructors and fields is what lets our factory support both injection styles (see [§19](#19-constructor-injection) / [§20](#20-field-injection)).

---

# 12. @RequestMapping, @GetMapping, @PostMapping

```java
// annotation/RequestMapping.java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface RequestMapping {
    String value() default "";
    HttpMethod method() default HttpMethod.GET;
}

// annotation/GetMapping.java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface GetMapping {
    String value();
}

// annotation/PostMapping.java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface PostMapping {
    String value();
}
```

`@RequestMapping` on a **type** supplies a class-level path prefix (e.g. `/users`); `@GetMapping`/`@PostMapping` on a **method** supply the sub-path and HTTP verb, exactly like real Spring MVC.

---

# 13. @PathVariable and @RequestParam

```java
// annotation/PathVariable.java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
public @interface PathVariable {
    String value();
}

// annotation/RequestParam.java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
public @interface RequestParam {
    String value();
    String defaultValue() default "";
}
```

`ElementType.PARAMETER` retention lets the front controller inspect a handler method's parameters and decide, per-argument, whether to pull the value from the URL path, the query string, or the bean container.

---

# 14. Step 2: BeanDefinition

A `BeanDefinition` is **metadata about a bean**, not the bean itself — Spring separates "what to build" from "the built object" so the container can reason about the whole graph before instantiating anything.

```java
// context/BeanDefinition.java
public class BeanDefinition {
    private final String beanName;
    private final Class<?> beanClass;
    private final boolean isController;

    public BeanDefinition(String beanName, Class<?> beanClass, boolean isController) {
        this.beanName = beanName;
        this.beanClass = beanClass;
        this.isController = isController;
    }

    public String getBeanName() { return beanName; }
    public Class<?> getBeanClass() { return beanClass; }
    public boolean isController() { return isController; }
}
```

Keeping this deliberately small mirrors Spring's real design: `BeanDefinition` in Spring also holds scope, lazy-init flag, property values, and init/destroy method names — we only need what our feature set uses.

---

# 15. Step 3: Classpath Component Scanning

The scanner's job: given a base package like `com.example.demo`, walk every `.class` file on the classpath under that package and return the ones annotated (directly or via meta-annotation) with `@Component`.

```java
// context/ComponentScanner.java
public class ComponentScanner {

    public List<BeanDefinition> scan(String basePackage) {
        List<BeanDefinition> definitions = new ArrayList<>();
        String path = basePackage.replace('.', '/');
        ClassLoader classLoader = Thread.currentThread().getContextClassLoader();

        try {
            Enumeration<URL> resources = classLoader.getResources(path);
            while (resources.hasMoreElements()) {
                URL resource = resources.nextElement();
                File directory = new File(resource.getFile());
                findClasses(directory, basePackage, definitions);
            }
        } catch (IOException e) {
            throw new RuntimeException("Component scan failed for " + basePackage, e);
        }
        return definitions;
    }

    private void findClasses(File directory, String packageName, List<BeanDefinition> out) {
        if (!directory.exists()) return;
        File[] files = directory.listFiles();
        if (files == null) return;

        for (File file : files) {
            if (file.isDirectory()) {
                findClasses(file, packageName + "." + file.getName(), out);
            } else if (file.getName().endsWith(".class")) {
                String className = packageName + '.' + file.getName().replace(".class", "");
                registerIfComponent(className, out);
            }
        }
    }

    private void registerIfComponent(String className, List<BeanDefinition> out) {
        try {
            Class<?> clazz = Class.forName(className);
            if (clazz.isInterface() || clazz.isAnnotation()) return;

            if (isStereotype(clazz)) {
                String beanName = resolveBeanName(clazz);
                boolean controller = clazz.isAnnotationPresent(Controller.class);
                out.add(new BeanDefinition(beanName, clazz, controller));
            }
        } catch (ClassNotFoundException e) {
            throw new RuntimeException(e);
        }
    }

    /** True if the class is @Component, or annotated with something that is itself @Component. */
    private boolean isStereotype(Class<?> clazz) {
        for (Annotation annotation : clazz.getAnnotations()) {
            if (annotation.annotationType().isAnnotationPresent(Component.class)
                    || annotation.annotationType() == Component.class) {
                return true;
            }
        }
        return false;
    }

    private String resolveBeanName(Class<?> clazz) {
        String simple = clazz.getSimpleName();
        return Character.toLowerCase(simple.charAt(0)) + simple.substring(1);
    }
}
```

> **Note:** real Spring Boot supports scanning inside JARs (`jar:file:...!/com/...`) as well as exploded directories. Our version above only walks exploded `.class` directories, which is exactly what you get running from an IDE or `java -cp target/classes`. Extending `findClasses` to open `JarFile` entries is a natural next exercise (see [§38](#38-extending-the-framework-next-steps)).

---

# 16. Scanning Algorithm Walkthrough

1. Convert the base package (`com.example.demo`) into a file-system path (`com/example/demo`).
2. Ask the classloader for every classpath root that contains that path (`getResources`).
3. Recursively walk directories, turning each `.class` file back into a fully-qualified class name.
4. `Class.forName(...)` loads the class (without running static initializers unless you pass `initialize=true`, which we deliberately don't).
5. Check whether the class carries `@Component` — directly, or through a meta-annotation like `@Service`.
6. If yes, derive a bean name (decapitalized simple class name, Spring's own default) and record a `BeanDefinition`.

This produces a **flat list of "things that need to become beans"** — no objects have been created yet. That separation is what lets the next step build a dependency graph before touching the constructors.

---

# 17. Step 4: The Bean Factory

The `DefaultBeanFactory` turns `BeanDefinition`s into live objects, resolving each bean's dependencies recursively and caching singletons.

```java
// context/DefaultBeanFactory.java
public class DefaultBeanFactory {
    private final Map<String, BeanDefinition> beanDefinitions = new LinkedHashMap<>();
    private final Map<String, Object> singletonCache = new ConcurrentHashMap<>();
    private final Set<String> beansCurrentlyInCreation = new HashSet<>();

    public void registerBeanDefinition(BeanDefinition definition) {
        beanDefinitions.put(definition.getBeanName(), definition);
    }

    public Collection<BeanDefinition> getBeanDefinitions() {
        return beanDefinitions.values();
    }

    public Object getBean(String beanName) {
        Object cached = singletonCache.get(beanName);
        if (cached != null) return cached;

        BeanDefinition definition = beanDefinitions.get(beanName);
        if (definition == null) {
            throw new NoSuchBeanDefinitionException(beanName);
        }
        return createBean(definition);
    }

    @SuppressWarnings("unchecked")
    public <T> T getBean(Class<T> type) {
        for (BeanDefinition definition : beanDefinitions.values()) {
            if (type.isAssignableFrom(definition.getBeanClass())) {
                return (T) getBean(definition.getBeanName());
            }
        }
        throw new NoSuchBeanDefinitionException(type.getName());
    }

    private Object createBean(BeanDefinition definition) {
        String beanName = definition.getBeanName();

        if (beansCurrentlyInCreation.contains(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }
        beansCurrentlyInCreation.add(beanName);
        try {
            Object bean = instantiate(definition.getBeanClass());
            injectFields(bean);
            singletonCache.put(beanName, bean); // default scope: singleton
            return bean;
        } finally {
            beansCurrentlyInCreation.remove(beanName);
        }
    }
    // instantiate() and injectFields() below, in §18–§20
}
```

Notice `beansCurrentlyInCreation` — a simple `Set` used as a "currently on the call stack" marker. This is the entire mechanism behind circular-dependency detection (expanded in [§22](#22-circular-dependency-detection)).

---

# 18. Instantiating Beans via Reflection

```java
// DefaultBeanFactory — instantiate()
private Object instantiate(Class<?> beanClass) {
    Constructor<?> constructor = selectConstructor(beanClass);
    Class<?>[] paramTypes = constructor.getParameterTypes();
    Object[] args = new Object[paramTypes.length];

    for (int i = 0; i < paramTypes.length; i++) {
        args[i] = getBean(paramTypes[i]); // recursive resolution
    }

    try {
        constructor.setAccessible(true);
        return constructor.newInstance(args);
    } catch (ReflectiveOperationException e) {
        throw new RuntimeException("Failed to instantiate " + beanClass.getName(), e);
    }
}

private Constructor<?> selectConstructor(Class<?> beanClass) {
    for (Constructor<?> constructor : beanClass.getDeclaredConstructors()) {
        if (constructor.isAnnotationPresent(Autowired.class)) {
            return constructor;
        }
    }
    // no @Autowired constructor -> fall back to the no-arg constructor,
    // exactly like Spring does when there is only one constructor.
    try {
        return beanClass.getDeclaredConstructor();
    } catch (NoSuchMethodException e) {
        throw new RuntimeException(
            beanClass.getName() + " needs a no-arg constructor or an @Autowired one", e);
    }
}
```

Each constructor argument is resolved by **recursively calling `getBean(paramTypes[i])`** — this is the recursive descent that walks the whole dependency graph, one bean at a time, depth-first.

---

# 19. Constructor Injection

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

When `getBean("userService")` runs, `selectConstructor` finds the `@Autowired` constructor, `instantiate` resolves `UserRepository` first (recursively creating it if it isn't cached yet), then calls `new UserService(theRepository)`.

**Why Spring — and this framework — prefer constructor injection:** the object is never in a "half-built, dependency missing" state, and it plays well with `final` fields and immutability.

---

# 20. Field Injection

```java
// DefaultBeanFactory — injectFields()
private void injectFields(Object bean) {
    for (Field field : bean.getClass().getDeclaredFields()) {
        if (!field.isAnnotationPresent(Autowired.class)) continue;

        Object dependency = getBean(field.getType());
        try {
            field.setAccessible(true);
            field.set(bean, dependency);
        } catch (IllegalAccessException e) {
            throw new RuntimeException("Failed to inject field " + field.getName(), e);
        }
    }
}
```

```java
@Controller
public class UserController {
    @Autowired
    private UserService userService; // injected AFTER construction
}
```

Field injection happens **after** the object already exists (`instantiate()` returns, then `injectFields()` runs) — which is exactly why it can quietly paper over a circular dependency that constructor injection cannot (see next section).

---

# 21. Dependency Resolution Order

Because `getBean()` resolves dependencies **lazily and recursively** rather than pre-computing a topological sort, the effective creation order falls out naturally from the call stack:

```text
getBean("userController")
  -> instantiate UserController (no-arg constructor)
  -> injectFields(userController)
       -> field `userService` needs UserService
          -> getBean("userService")
               -> instantiate UserService(UserRepository) [@Autowired constructor]
                    -> constructor arg needs UserRepository
                       -> getBean("userRepository")
                            -> instantiate UserRepository() [no-arg]
                            -> injectFields (none)
                            -> cache "userRepository"
                    -> new UserService(userRepository)
               -> injectFields(userService) (none)
               -> cache "userService"
       -> field set: userController.userService = userService
  -> cache "userController"
```

This depth-first recursive resolution is functionally equivalent to a topological sort of the dependency graph — it just computes the order on demand instead of up front. The trade-off, and the reason real containers *also* do an eager pre-instantiation pass at startup, is in the next section.

---

# 22. Circular Dependency Detection

Two beans depending on each other through **constructors** cannot ever be resolved — there's no valid order:

```java
@Service
public class A {
    @Autowired
    public A(B b) {}   // needs a finished B
}

@Service
public class B {
    @Autowired
    public B(A a) {}   // needs a finished A
}
```

Tracing through `createBean`:

```text
getBean("a")
  beansCurrentlyInCreation.add("a")
  instantiate(A) needs B -> getBean("b")
    beansCurrentlyInCreation.add("b")
    instantiate(B) needs A -> getBean("a")
      "a" is already in beansCurrentlyInCreation -> throw BeanCurrentlyInCreationException
```

```java
// context/BeanCurrentlyInCreationException.java
public class BeanCurrentlyInCreationException extends RuntimeException {
    public BeanCurrentlyInCreationException(String beanName) {
        super("Circular dependency detected while creating bean '" + beanName +
              "' — constructor injection cannot resolve a cycle. " +
              "Break the cycle or switch one side to field/setter injection.");
    }
}
```

**Field injection can survive a cycle** because the object is fully allocated (via its no-arg constructor) *before* its fields are populated — so `A`'s half-built instance can already be cached and handed to `B`, even though `A.userService`-equivalent field isn't set yet. This is exactly why real Spring can resolve some circular dependencies with field/setter injection but always fails on circular constructor injection — our implementation reproduces that same asymmetry for the same underlying reason.

---

# 23. Singleton Cache and Bean Scopes

Our `singletonCache` (a `Map<String, Object>`) is populated the *first* time a bean is requested and reused for every subsequent `getBean()` call — this is Spring's default `singleton` scope.

| Scope | Behavior | Supported here? |
|---|---|---|
| `singleton` | One shared instance per container | ✅ default, always on |
| `prototype` | A new instance every `getBean()` call | ❌ left as an extension (§38) — would mean skipping the cache `put`/`get` for that bean name |
| `request` / `session` | Web-scoped instances | ❌ out of scope — needs the web layer to manage a per-request registry |

Adding prototype scope only requires storing a `scope` flag on `BeanDefinition` and branching in `getBean(String)` before checking `singletonCache`.

---

# 24. Step 5: The ApplicationContext

`ApplicationContext` is the **public facade** — application code never talks to `ComponentScanner` or `DefaultBeanFactory` directly.

```java
// context/ApplicationContext.java
public interface ApplicationContext {
    <T> T getBean(Class<T> type);
    Object getBean(String name);
    void refresh();
}
```

```java
// context/MiniApplicationContext.java
public class MiniApplicationContext implements ApplicationContext {
    private final DefaultBeanFactory beanFactory = new DefaultBeanFactory();
    private final String basePackage;

    public MiniApplicationContext(String basePackage) {
        this.basePackage = basePackage;
        refresh();
    }

    @Override
    public void refresh() {
        // 1. scan: find every class annotated (directly or via meta-annotation) with @Component
        ComponentScanner scanner = new ComponentScanner();
        List<BeanDefinition> definitions = scanner.scan(basePackage);
        definitions.forEach(beanFactory::registerBeanDefinition);

        // 2. eagerly instantiate every singleton bean NOW, not on first getBean() call —
        //    this is what lets a wiring/circular-dependency mistake fail fast at startup
        //    instead of surfacing later in production traffic.
        for (BeanDefinition definition : beanFactory.getBeanDefinitions()) {
            beanFactory.getBean(definition.getBeanName());
        }
    }

    @Override
    public <T> T getBean(Class<T> type) { return beanFactory.getBean(type); }

    @Override
    public Object getBean(String name) { return beanFactory.getBean(name); }

    public DefaultBeanFactory getBeanFactory() { return beanFactory; }
}
```

Step 2 of `refresh()` — the **eager instantiation loop** — is the single most important line for matching real Spring behavior. Without it, a missing dependency or a circular reference would only blow up the first time some unlucky request happens to touch that bean.

---

# 25. AnnotationConfigApplicationContext

Real Spring names this class `AnnotationConfigApplicationContext` and takes a *configuration class* (annotated `@ComponentScan`) rather than a raw package string. Adding that thin layer is mechanical:

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface ComponentScan {
    String value(); // base package to scan
}

@ComponentScan("com.example.demo")
public class AppConfig { }
```

```java
public MiniApplicationContext(Class<?> configClass) {
    ComponentScan scan = configClass.getAnnotation(ComponentScan.class);
    this.basePackage = scan.value();
    refresh();
}
```

This is purely ergonomic — it lets `MiniApplication.run(AppConfig.class, args)` read naturally (see [§33](#33-step-8-miniapplicationrun--wiring-it-all-together)).

---

# 26. getBean() API

```java
ApplicationContext context = new MiniApplicationContext(AppConfig.class);

UserService userService = context.getBean(UserService.class);
Object raw = context.getBean("userService"); // by name, like Spring's context.getBean("userService")
```

Two lookup strategies, matching Spring exactly: by type (fails if zero or more-than-one match — our simplified version above just returns the first match; a stricter implementation would throw `NoUniqueBeanDefinitionException`) and by explicit bean name.

---

# 27. Step 6: The Web Layer — Handler Mapping

A **handler mapping** is a table: `(HTTP method, URL pattern) -> (controller bean, method)`. It is built once, at startup, by scanning every bean whose `BeanDefinition.isController()` is true.

```java
// web/HttpMethod.java
public enum HttpMethod { GET, POST }
```

```java
// web/HandlerMethod.java
public class HandlerMethod {
    private final Object controllerBean;
    private final Method method;
    private final String pathPattern; // e.g. "/users/{id}"
    private final HttpMethod httpMethod;

    public HandlerMethod(Object controllerBean, Method method, String pathPattern, HttpMethod httpMethod) {
        this.controllerBean = controllerBean;
        this.method = method;
        this.pathPattern = pathPattern;
        this.httpMethod = httpMethod;
    }
    // getters omitted for brevity
}
```

```java
// web/HandlerMapping.java
public class HandlerMapping {
    private final List<HandlerMethod> handlers = new ArrayList<>();

    public void registerControllers(ApplicationContext context, DefaultBeanFactory factory) {
        for (BeanDefinition definition : factory.getBeanDefinitions()) {
            if (!definition.isController()) continue;

            Object controllerBean = context.getBean(definition.getBeanName());
            Class<?> controllerClass = definition.getBeanClass();
            String basePath = resolveBasePath(controllerClass);

            for (Method method : controllerClass.getDeclaredMethods()) {
                registerIfMapped(controllerBean, method, basePath);
            }
        }
    }

    private String resolveBasePath(Class<?> controllerClass) {
        RequestMapping typeMapping = controllerClass.getAnnotation(RequestMapping.class);
        return typeMapping != null ? typeMapping.value() : "";
    }

    private void registerIfMapped(Object bean, Method method, String basePath) {
        if (method.isAnnotationPresent(GetMapping.class)) {
            String path = basePath + method.getAnnotation(GetMapping.class).value();
            handlers.add(new HandlerMethod(bean, method, path, HttpMethod.GET));
        } else if (method.isAnnotationPresent(PostMapping.class)) {
            String path = basePath + method.getAnnotation(PostMapping.class).value();
            handlers.add(new HandlerMethod(bean, method, path, HttpMethod.POST));
        }
    }

    public Optional<MatchedHandler> resolve(HttpMethod httpMethod, String requestPath) {
        for (HandlerMethod handler : handlers) {
            if (handler.getHttpMethod() != httpMethod) continue;
            Map<String, String> pathVars = matchAndExtract(handler.getPathPattern(), requestPath);
            if (pathVars != null) return Optional.of(new MatchedHandler(handler, pathVars));
        }
        return Optional.empty();
    }

    /** Turns "/users/{id}" + "/users/42" into {"id": "42"}, or null if no match. */
    private Map<String, String> matchAndExtract(String pattern, String actualPath) {
        String[] patternParts = pattern.split("/");
        String[] actualParts = actualPath.split("/");
        if (patternParts.length != actualParts.length) return null;

        Map<String, String> vars = new HashMap<>();
        for (int i = 0; i < patternParts.length; i++) {
            String part = patternParts[i];
            if (part.startsWith("{") && part.endsWith("}")) {
                vars.put(part.substring(1, part.length() - 1), actualParts[i]);
            } else if (!part.equals(actualParts[i])) {
                return null; // literal segment mismatch
            }
        }
        return vars;
    }
}
```

---

# 28. Building the Route Table

Given:

```java
@Controller
@RequestMapping("/users")
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping("/{id}")
    public User getUser(@PathVariable("id") Long id) {
        return userService.findById(id);
    }

    @PostMapping("")
    public User createUser(@RequestParam("name") String name) {
        return userService.create(name);
    }
}
```

`HandlerMapping.registerControllers` produces:

| HTTP Method | Path Pattern | Handler |
|---|---|---|
| GET | `/users/{id}` | `UserController#getUser` |
| POST | `/users` | `UserController#createUser` |

This table is exactly what `DispatcherServlet`'s `RequestMappingHandlerMapping` builds internally in real Spring MVC — ours is a deliberately simplified, linear-scan version instead of Spring's trie-based `PathPattern` matcher.

---

# 29. Step 7: The Embedded HTTP Server

Rather than pull in Tomcat/Jetty, we lean on the JDK's built-in `com.sun.net.httpserver.HttpServer` — good enough to demonstrate real request/response handling with zero extra dependencies.

```java
// MiniApplication.java (server bootstrap portion)
HttpServer server = HttpServer.create(new InetSocketAddress(port), 0);
server.createContext("/", new FrontController(handlerMapping));
server.setExecutor(Executors.newFixedThreadPool(8));
server.start();
```

Every inbound request — regardless of path — hits the same `FrontController`, exactly mirroring how Spring routes *all* traffic through a single `DispatcherServlet` mapped to `/*`.

---

# 30. The Front Controller (DispatcherServlet Equivalent)

```java
// web/FrontController.java
public class FrontController implements HttpHandler {
    private final HandlerMapping handlerMapping;

    public FrontController(HandlerMapping handlerMapping) {
        this.handlerMapping = handlerMapping;
    }

    @Override
    public void handle(HttpExchange exchange) throws IOException {
        HttpMethod method = HttpMethod.valueOf(exchange.getRequestMethod());
        String path = exchange.getRequestURI().getPath();

        Optional<MatchedHandler> matched = handlerMapping.resolve(method, path);
        if (matched.isEmpty()) {
            writeResponse(exchange, 404, "{\"error\":\"No handler for " + method + " " + path + "\"}");
            return;
        }

        try {
            Object result = invoke(matched.get(), exchange);
            writeResponse(exchange, 200, toJson(result));
        } catch (Exception e) {
            writeResponse(exchange, 500, "{\"error\":\"" + e.getMessage() + "\"}");
        }
    }

    private void writeResponse(HttpExchange exchange, int status, String body) throws IOException {
        byte[] bytes = body.getBytes(StandardCharsets.UTF_8);
        exchange.getResponseHeaders().add("Content-Type", "application/json");
        exchange.sendResponseHeaders(status, bytes.length);
        try (OutputStream os = exchange.getResponseBody()) {
            os.write(bytes);
        }
    }
    // invoke() and toJson() in §31
}
```

---

# 31. Resolving and Invoking Handlers

```java
// web/FrontController.java — invoke()
private Object invoke(MatchedHandler matched, HttpExchange exchange) throws Exception {
    HandlerMethod handler = matched.getHandlerMethod();
    Method method = handler.getMethod();
    Map<String, String> pathVars = matched.getPathVariables();
    Map<String, String> queryParams = parseQuery(exchange.getRequestURI().getRawQuery());

    Object[] args = new Object[method.getParameterCount()];
    Parameter[] params = method.getParameters();

    for (int i = 0; i < params.length; i++) {
        Parameter param = params[i];
        if (param.isAnnotationPresent(PathVariable.class)) {
            String key = param.getAnnotation(PathVariable.class).value();
            args[i] = convert(pathVars.get(key), param.getType());
        } else if (param.isAnnotationPresent(RequestParam.class)) {
            RequestParam rp = param.getAnnotation(RequestParam.class);
            String raw = queryParams.getOrDefault(rp.value(), rp.defaultValue());
            args[i] = convert(raw, param.getType());
        }
    }

    method.setAccessible(true);
    return method.invoke(handler.getControllerBean(), args);
}

private Object convert(String raw, Class<?> targetType) {
    if (targetType == Long.class || targetType == long.class) return Long.valueOf(raw);
    if (targetType == Integer.class || targetType == int.class) return Integer.valueOf(raw);
    return raw; // String and anything else passes through as-is
}
```

This is the reflective core of *every* Java MVC framework: read the method's declared parameters, match each one against an annotation, pull the matching raw value from the request, coerce its type, and `invoke()`.

---

# 32. Serializing the Response (JSON)

We do not hand-roll a JSON library — a handful of lines using reflection is enough for a demo-quality response writer:

```java
private String toJson(Object result) {
    if (result == null) return "null";
    if (result instanceof String) return "\"" + result + "\"";
    if (result instanceof Number || result instanceof Boolean) return result.toString();

    StringBuilder json = new StringBuilder("{");
    Field[] fields = result.getClass().getDeclaredFields();
    for (int i = 0; i < fields.length; i++) {
        fields[i].setAccessible(true);
        try {
            Object value = fields[i].get(result);
            json.append("\"").append(fields[i].getName()).append("\":");
            json.append(value instanceof String ? "\"" + value + "\"" : value);
            if (i < fields.length - 1) json.append(",");
        } catch (IllegalAccessException ignored) { }
    }
    return json.append("}").toString();
}
```

> In a real project you would use Jackson here (as Spring Boot does via `MappingJackson2HttpMessageConverter`). This inline version exists purely so the guide's example runs with **zero external dependencies**.

---

# 33. Step 8: MiniApplication.run() — Wiring It All Together

```java
// MiniApplication.java
public class MiniApplication {

    public static ApplicationContext run(Class<?> primarySource, String[] args) {
        long start = System.currentTimeMillis();

        MiniApplicationContext context = new MiniApplicationContext(primarySource);

        HandlerMapping handlerMapping = new HandlerMapping();
        handlerMapping.registerControllers(context, context.getBeanFactory());

        int port = 8080;
        try {
            HttpServer server = HttpServer.create(new InetSocketAddress(port), 0);
            server.createContext("/", new FrontController(handlerMapping));
            server.setExecutor(Executors.newFixedThreadPool(8));
            server.start();
        } catch (IOException e) {
            throw new RuntimeException("Failed to start embedded server on port " + port, e);
        }

        long elapsed = System.currentTimeMillis() - start;
        System.out.println("MiniSpring started on port " + port + " in " + elapsed + " ms");
        return context;
    }
}
```

---

# 34. Full Working Example

```java
// demo/AppConfig.java
@ComponentScan("com.example.demo")
public class AppConfig { }

// demo/UserRepository.java
@Repository
public class UserRepository {
    private final Map<Long, User> store = new ConcurrentHashMap<>();
    public User findById(Long id) { return store.get(id); }
    public User save(User user) { store.put(user.id, user); return user; }
}

// demo/UserService.java
@Service
public class UserService {
    private final UserRepository repository;

    @Autowired
    public UserService(UserRepository repository) {
        this.repository = repository;
    }

    public User findById(Long id) { return repository.findById(id); }

    public User create(String name) {
        User user = new User(System.nanoTime(), name);
        return repository.save(user);
    }
}

// demo/UserController.java
@Controller
@RequestMapping("/users")
public class UserController {
    @Autowired
    private UserService userService;

    @GetMapping("/{id}")
    public User getUser(@PathVariable("id") Long id) { return userService.findById(id); }

    @PostMapping("")
    public User createUser(@RequestParam("name") String name) { return userService.create(name); }
}

// Main.java
public class Main {
    public static void main(String[] args) {
        MiniApplication.run(AppConfig.class, args);
    }
}
```

```bash
curl -X POST "http://localhost:8080/users?name=Alice"
# {"id":123456789,"name":"Alice"}

curl "http://localhost:8080/users/123456789"
# {"id":123456789,"name":"Alice"}
```

---

# 35. End-to-End Request Flow

```text
main() -> MiniApplication.run(AppConfig.class, args)
             |
             +--> new MiniApplicationContext(AppConfig.class)
             |        +--> ComponentScanner.scan("com.example.demo")
             |        |        finds: UserRepository, UserService, UserController
             |        +--> for each definition: beanFactory.getBean(name)  [eager init]
             |                 UserRepository()          -> cached
             |                 UserService(UserRepository)-> cached
             |                 UserController()           -> cached
             |                    injectFields: userController.userService = <cached UserService>
             |
             +--> new HandlerMapping().registerControllers(context, beanFactory)
             |        builds route table: GET /users/{id}, POST /users
             |
             +--> HttpServer.start() on :8080

   ... time passes ...

HTTP GET /users/42
   -> FrontController.handle(exchange)
        -> handlerMapping.resolve(GET, "/users/42") -> matches "/users/{id}", id=42
        -> invoke(): reads @PathVariable("id") -> converts "42" -> Long 42
        -> method.invoke(userController, 42L) -> userService.findById(42L)
        -> toJson(user) -> HTTP 200 response body
```

---

# 36. Step 9: Testing the Framework

Because the container is just plain objects wired by reflection, it is unit-testable without spinning up HTTP at all:

```java
class DefaultBeanFactoryTest {

    @Test
    void wiresConstructorInjectionCorrectly() {
        DefaultBeanFactory factory = new DefaultBeanFactory();
        factory.registerBeanDefinition(new BeanDefinition("userRepository", UserRepository.class, false));
        factory.registerBeanDefinition(new BeanDefinition("userService", UserService.class, false));

        UserService service = factory.getBean(UserService.class);

        assertNotNull(service);
        assertNotNull(service.findById(1L)); // proves UserRepository was actually injected, not null
    }

    @Test
    void detectsCircularConstructorDependency() {
        DefaultBeanFactory factory = new DefaultBeanFactory();
        factory.registerBeanDefinition(new BeanDefinition("a", A.class, false));
        factory.registerBeanDefinition(new BeanDefinition("b", B.class, false));

        assertThrows(BeanCurrentlyInCreationException.class, () -> factory.getBean(A.class));
    }

    @Test
    void singletonScopeReturnsSameInstance() {
        DefaultBeanFactory factory = new DefaultBeanFactory();
        factory.registerBeanDefinition(new BeanDefinition("userRepository", UserRepository.class, false));

        assertSame(factory.getBean(UserRepository.class), factory.getBean(UserRepository.class));
    }
}
```

For the web layer, test `HandlerMapping.resolve(...)` directly with fabricated `BeanDefinition`s rather than going through a real socket — faster, and it isolates the routing logic from the HTTP transport.

---

# 37. Common Pitfalls and Gotchas

| Pitfall | Why it happens | Fix |
|---|---|---|
| `NoSuchBeanDefinitionException` for a class you *did* annotate | The class isn't under the scanned base package, or the scanner only walked exploded directories and your class is inside a JAR | Verify `basePackage`; extend the scanner to read `JarFile` entries |
| `@Autowired` field stays `null` | You annotated a field in a class the container never scanned as a bean (missing `@Component`/`@Service`/etc.) | Add a stereotype annotation to the class |
| Infinite recursion instead of a clean circular-dependency error | Forgot to track `beansCurrentlyInCreation`, so `getBean` re-enters `createBean` forever | Guard `createBean` with the in-creation `Set`, as in [§17](#17-step-4-the-bean-factory) |
| Two beans of the same type, `getBean(Class)` returns the "wrong" one | Type-based lookup here returns the *first* match found during iteration | Prefer `getBean(String)` by explicit name when a type is ambiguous, or throw on ambiguity like real Spring's `NoUniqueBeanDefinitionException` |
| Route never matches despite looking identical to the pattern | Trailing slash mismatch (`/users/` vs `/users`) — our naive `split("/")` matcher is strict | Normalize trailing slashes before calling `resolve()` |

---

# 38. Extending the Framework: Next Steps

Once the core above works, these are the natural next features to bolt on — each is a well-scoped, self-contained exercise:

- **Prototype scope** — add a `scope` field to `BeanDefinition`; skip the singleton cache for prototype beans.
- **`@PostConstruct` lifecycle callback** — after `injectFields`, reflectively find and invoke a method annotated `@PostConstruct`.
- **AOP-lite / `@Transactional` demo** — wrap a bean in a JDK dynamic proxy (`Proxy.newProxyInstance`) that begins/commits a fake transaction around every method call, to show how Spring AOP intercepts calls without touching your source.
- **Exception handling (`@ExceptionHandler`)** — let `FrontController` catch exceptions thrown by `method.invoke(...)` and route them to a registered handler method instead of a flat 500.
- **JAR-aware scanning** — extend `ComponentScanner.findClasses` to open `JarFile` and iterate `JarEntry` when the classpath resource URL's protocol is `jar` instead of `file`.
- **Interceptors (`HandlerInterceptor`)** — a `preHandle`/`postHandle` chain the `FrontController` runs around `invoke()`, useful for logging or auth demos.
- **Constructor injection ambiguity resolution** — when a class has multiple constructors and none is `@Autowired`, pick the greediest one (most parameters), matching Spring's actual fallback heuristic.

---

# 39. Implementation Roadmap

A sensible build order if you are doing this as a learning project, each phase independently testable:

1. **Phase 1 — Annotations only.** Define every annotation in [§8–§13](#8-step-1-custom-annotations); no logic yet.
2. **Phase 2 — Manual bean factory.** Hand-write `BeanDefinition`s (no scanning), prove `DefaultBeanFactory` can instantiate + wire them.
3. **Phase 3 — Component scanning.** Replace manual `BeanDefinition` registration with `ComponentScanner.scan(...)`.
4. **Phase 4 — ApplicationContext facade + eager init.** Wrap scanner + factory behind `MiniApplicationContext`, add the eager-instantiation loop.
5. **Phase 5 — Circular dependency detection.** Add `beansCurrentlyInCreation` and the custom exception; write a failing test first.
6. **Phase 6 — Handler mapping.** Build the route table from `@Controller` beans, no HTTP yet — test `resolve()` in isolation.
7. **Phase 7 — Embedded HTTP server + front controller.** Wire `HandlerMapping` into `FrontController`, bring the server up.
8. **Phase 8 — Parameter binding + JSON.** `@PathVariable`, `@RequestParam`, and the reflective JSON writer.
9. **Phase 9 — `MiniApplication.run()`.** Collapse all of the above behind the one-line bootstrap.

---

# 40. Final Architecture

```text
┌─────────────────────────────────────────────────────────────────────┐
│                             MiniApplication                          │
│  run(AppConfig.class, args)                                          │
└───────────────────────────────┬───────────────────────────────────--┘
                                 │
                 ┌───────────────┴────────────────┐
                 ▼                                 ▼
   ┌───────────────────────────┐     ┌───────────────────────────────┐
   │   MiniApplicationContext   │     │        HandlerMapping          │
   │  ┌───────────────────────┐│     │  route table:                  │
   │  │  ComponentScanner      ││     │  (GET  /users/{id}) -> method  │
   │  └───────────┬───────────┘│     │  (POST /users)      -> method  │
   │              ▼            │     └───────────────┬─────────────────┘
   │  ┌───────────────────────┐│                     │
   │  │  DefaultBeanFactory     │◄────getBean()───────┘
   │  │  - BeanDefinitions      │
   │  │  - singletonCache       │
   │  │  - beansCurrentlyInCreation (cycle guard)     │
   │  └───────────────────────┘│
   └───────────────────────────┘
                                                       ▼
                                     ┌───────────────────────────────┐
                                     │        FrontController         │
                                     │  (implements HttpHandler)      │
                                     └───────────────┬─────────────────┘
                                                       ▼
                                     ┌───────────────────────────────┐
                                     │  com.sun.net.httpserver         │
                                     │        HttpServer               │
                                     └───────────────────────────────┘
```

---

# 41. Key Takeaways

- **Separation of metadata from objects** (`BeanDefinition` vs. the actual bean instance) is what lets a container validate and reason about a whole dependency graph before creating anything — the same idea shows up in Kubernetes manifests, Terraform plans, and build-system dependency graphs.
- **Reflection + annotations** is the entire trick behind "magic" DI — there is no magic, just `Class`, `Constructor`, `Field`, `Method`, and a disciplined convention for reading them at startup.
- **Eager singleton instantiation at `refresh()` time**, not on first use, is what converts a wiring mistake into a fast, loud startup failure instead of a 2 a.m. production surprise.
- **A single `Set` tracking "beans currently being created"** is the entire mechanism behind circular-dependency detection — no graph library required.
- **Every MVC framework's routing is the same shape:** a table of `(method, path-pattern) -> reflective handler`, built once at startup, matched on every request by a single front controller.
- Building this from scratch is the fastest way to stop treating `@Autowired` and `@GetMapping` as magic — and to debug real Spring issues (bean not found, circular dependency, ambiguous mapping) with an accurate mental model instead of guesswork.
