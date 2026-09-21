# Build Your Own CRUD Admin Framework From Scratch — MiniAdmin Step-by-Step Guide

> **Goal:** Build **MiniAdmin** — a metadata-driven CRUD framework on top of [MiniSpring](MiniSpring-Step-by-Step-Guide.md) that behaves like Django's admin site or Flask-Admin: define a model (as a Java class, with annotations — no boilerplate CRUD code) and get a working list view, create/edit form, and delete action automatically, including dropdowns for many-to-one relationships and related lists for one-to-many. On top of that, add a **Salesforce-style Custom Object builder** — the ability to define an entirely new "object" (name, fields, types) **from the UI, at runtime, with no code deployment at all**.
>
> This guide assumes familiarity with the companion guides: [MiniSpring-Step-by-Step-Guide.md](MiniSpring-Step-by-Step-Guide.md) (the IoC container, component scanning, and MVC layer MiniAdmin is built on top of) and, optionally, [Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) (MiniAdmin's generic repository talks to any JDBC database, TinyDB included, exactly the way a MiniSpring `@Repository` does).

---

# 1. What We Are Building

We are building **MiniAdmin** — a framework that turns a data model into a working CRUD UI **without writing a controller, a form, or a table per entity**. By the end of this guide you will have:

- A **metadata model** (`ObjectMetadata`/`FieldMetadata`) that describes any "object" — its fields, their types, and its relationships — regardless of how that object was defined.
- **Two ways to define an object**, both producing the same metadata: a **Java class with annotations** (`@AdminEntity`, `@AdminField`, `@ManyToOne`, `@OneToMany` — Django-model-style), and a **Salesforce-style Custom Object builder** reachable entirely from the UI, with no code or redeploy.
- **Two storage strategies** behind one interface: real physical tables (`CREATE TABLE`) for class-based models, and an **Entity-Attribute-Value (EAV)** store for UI-defined custom objects — the same technique Salesforce's own platform uses internally, chosen specifically so adding a field from the UI never requires a schema migration.
- A **single generic CRUD REST controller** that serves list/create/read/update/delete for *any* registered object, driven entirely by its metadata — not one controller per entity.
- An **auto-generated admin UI**: a list (data table) view and a create/edit form for every object, with many-to-one fields rendered as searchable dropdowns and one-to-many relationships rendered as related-list panels — built and refreshed purely from metadata, the same way Django's admin site introspects `Model._meta.fields`.
- **Customization hooks** so a team can override the defaults (custom widgets, hidden fields, custom actions, a simple page-layout editor) without forking the generated behavior.

```text
Class-based model                     UI-based Custom Object
(@AdminEntity, Java code)             ("New Object" screen, no code)
        |                                       |
        v                                       v
  ClassModelScanner                    CustomObjectService
        |                                       |
        +------------------> MetadataRegistry <-+
                                    |
                    +---------------+---------------+
                    |                               |
           PhysicalTableStorage            EavStorage
           (real CREATE TABLE)             (generic value rows)
                    |                               |
                    +---------------+---------------+
                                    v
                        GenericRepository (metadata-driven)
                                    |
                                    v
                    GenericCrudController (one controller, any object)
                                    |
                                    v
                  Auto-generated Admin UI (list, form, dropdowns, related lists)
```

---

# 2. Learning Objectives

By the end of this guide you should be able to answer, from first principles:

- How does Django's admin site (or Flask-Admin) generate a full CRUD UI from a model class it has never seen before, without a developer writing a single view for it?
- What is the actual difference between defining a schema in code (a compiled class) and defining it as data (rows describing fields), and why does Salesforce choose the latter for its Custom Objects?
- What is Entity-Attribute-Value storage, why does it avoid the "every new field needs a migration" problem, and what does it cost you in return?
- How can one controller class correctly serve create/read/update/delete for objects it knows nothing about at compile time?
- How does a many-to-one relationship become a `<select>` dropdown, and a one-to-many relationship become a related-records panel, purely from metadata?
- Where do you draw the line between "generated by the framework" and "customized by a developer," so customization doesn't mean forking the generator?

---

# 3. Why Build This? (Interview Motivation)

> **"Design a system where defining a data model automatically produces a working CRUD UI — list, create, edit, delete — including relationship fields rendered as dropdowns and related lists, similar to Django's admin site. Additionally, support defining a completely new object from a UI, at runtime, without deploying code, similar to Salesforce Custom Objects. Explain how both paths converge into one system, and how you'd let a team customize the generated UI without losing the automation."**

This is a strong **platform/framework-design interview question** — distinct from a typical CRUD-app question — because it tests:

- **Metadata-driven design** — building software that describes *other* software's shape, and drives behavior from that description rather than from hard-coded per-entity logic.
- **Reflection** — extracting a data model from annotated Java classes, the same mechanism [MiniSpring's component scanner](MiniSpring-Step-by-Step-Guide.md) uses to find `@Component` classes.
- **Schema flexibility trade-offs** — physical tables vs. EAV, and knowing exactly when each is the right call rather than reaching for one reflexively.
- **API design for the unknown** — a single controller and a single set of UI components that must correctly handle any object, including ones that don't exist yet.
- **Extensibility discipline** — customization that composes with the generated defaults instead of replacing them, the same "strategy/decorator over special-casing" discipline the [TinyDB guide's design-pattern sections](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) argued for.

---

# 4. Recommended Technology Stack

| Concern | Choice | Why |
|---|---|---|
| Application framework | [MiniSpring](MiniSpring-Step-by-Step-Guide.md) | MiniAdmin is a layer *on top of* MiniSpring's `ApplicationContext`, component scanning, and `@Controller`/`@GetMapping` MVC layer — not a replacement for it. |
| Persistence | Any JDBC database — [TinyDB](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) or a real one | MiniAdmin's `GenericRepository` talks to `java.sql.Connection`, so it works unmodified against TinyDB or Postgres/MySQL. |
| Metadata storage | The same JDBC database, in two dedicated tables (`admin_object_def`, `admin_field_def`) | Object/field *definitions* need the same durability a real table's schema gets — see §18. |
| UI rendering | A single, generic, vanilla-JS admin frontend consuming a JSON metadata + CRUD API | No external UI framework — matches this series' "no unexplained dependency" discipline, and makes the metadata-drives-everything design visible in the code, not hidden inside a framework. |
| Reflection | `java.lang.reflect` (`Class`, `Field`, `Annotation`) | Same toolset [MiniSpring's `ComponentScanner`](MiniSpring-Step-by-Step-Guide.md) already uses. |

---

# 5. Project Structure

```text
miniadmin/
├── pom.xml
├── src/main/java/com/example/miniadmin/
│   ├── metadata/
│   │   ├── FieldType.java
│   │   ├── FieldMetadata.java
│   │   ├── RelationshipMetadata.java
│   │   ├── ObjectMetadata.java
│   │   └── MetadataRegistry.java
│   ├── classmodel/
│   │   ├── AdminEntity.java
│   │   ├── AdminField.java
│   │   ├── AdminId.java
│   │   ├── ManyToOne.java
│   │   ├── OneToMany.java
│   │   └── ClassModelScanner.java
│   ├── customobject/
│   │   ├── CustomObjectService.java
│   │   ├── CustomObjectController.java
│   │   └── AdminObjectDefRepository.java     // persists object/field DEFINITIONS
│   ├── storage/
│   │   ├── ObjectStorageStrategy.java
│   │   ├── PhysicalTableStorage.java
│   │   └── EavStorage.java
│   ├── repository/
│   │   └── GenericRepository.java
│   ├── web/
│   │   ├── GenericCrudController.java
│   │   ├── ObjectMetadataController.java
│   │   └── LookupController.java             // powers many-to-one dropdown search
│   └── customization/
│       ├── ObjectUiConfig.java
│       ├── FieldWidget.java
│       ├── WidgetRegistry.java
│       └── AdminAction.java
├── src/main/resources/static/admin/
│   ├── admin.html                            // the generic admin shell (single page)
│   ├── admin.js                              // metadata-driven list/form/dropdown rendering
│   └── admin.css
└── src/test/java/com/example/miniadmin/
    ├── ClassModelScannerTest.java
    ├── EavStorageTest.java
    ├── GenericRepositoryTest.java
    └── GenericCrudControllerTest.java
```

---

# 6. Two Ways to Define a Model: Class-Based vs UI-Based (Salesforce-Style)

Django models are Python classes; Salesforce Custom Objects are records created by clicking "New Object" in Setup. MiniAdmin deliberately supports **both**, because they serve different moments in a system's life:

| | Class-based (`@AdminEntity`) | UI-based (Custom Object builder) |
|---|---|---|
| Who defines it | A developer, in Java code | Anyone with admin access, from a browser |
| When it takes effect | On the next deploy | Immediately, no deploy |
| Where it's stored | The class file itself is the definition | A row in `admin_object_def`/`admin_field_def` (§18) |
| Backing storage | A real table, `CREATE TABLE`-defined | An EAV structure — no schema change per field (§16, §19) |
| Best for | Core domain entities that evolve with the codebase | Fast-moving, ad-hoc data needs — exactly Salesforce's own pitch for Custom Objects |

The design decision that makes this guide coherent rather than "two separate half-features": **both paths produce the exact same `ObjectMetadata`/`FieldMetadata` shape** (§8), and everything downstream of that — the generic repository (§23), the generic CRUD controller (§24), and the entire auto-generated UI (§28 onward) — is written once, against that shared metadata, and never needs to know or care which path an object came from.

---

# 7. High-Level Architecture

```text
                     +------------------+       +---------------------------+
                     |  @AdminEntity     |       |  "New Object" / "New Field" |
                     |  Java classes      |       |  UI screens (§20-§21)      |
                     +--------+----------+       +--------------+------------+
                              |                                 |
                     ClassModelScanner (§13)          CustomObjectService (§19)
                              |                                 |
                              +----------------+----------------+
                                               v
                                    MetadataRegistry (§11)
                                    ObjectMetadata + FieldMetadata (§8)
                                               |
                       +------------------------+------------------------+
                       |                                                 |
              PhysicalTableStorage (§17)                        EavStorage (§19)
              one real table per object                    one shared value table
                       |                                                 |
                       +------------------------+------------------------+
                                               v
                                  GenericRepository (§23)
                                  (talks to ObjectStorageStrategy, never
                                   a concrete table structure directly)
                                               |
                                               v
                                  GenericCrudController (§24)
                                  one controller, driven by {objectName}
                                               |
                                               v
                             Auto-generated Admin UI (§28-§36)
                             list view, form view, dropdowns, related lists
```

---

# 8. Phase 1 — The Metadata Model: ObjectMetadata and FieldMetadata

This is the single most important design decision in the whole guide: **everything else is a producer or a consumer of this one shape.**

```java
// metadata/FieldType.java
public enum FieldType {
    STRING, TEXT, INTEGER, LONG, DOUBLE, BOOLEAN, DATE, DATETIME,
    MANY_TO_ONE,   // a foreign-key-style reference to one record of another object
    ONE_TO_MANY    // the inverse side — a virtual field, never stored, only computed for display (§27)
}
```

```java
// metadata/FieldMetadata.java
public class FieldMetadata {
    private final String name;              // "customerId", "orderNumber" — the physical/logical field name
    private final String label;             // "Customer", "Order Number" — what the UI shows
    private final FieldType type;
    private final boolean required;
    private final boolean primaryKey;
    private final int displayOrder;
    private final RelationshipMetadata relationship; // null unless type is MANY_TO_ONE / ONE_TO_MANY

    // constructor / getters omitted for brevity
}

// metadata/RelationshipMetadata.java
public class RelationshipMetadata {
    private final String relatedObjectName;  // e.g. "Customer"
    private final String displayField;       // for MANY_TO_ONE: which field of the related record to show, e.g. "name"
    private final String mappedByField;      // for ONE_TO_MANY: the MANY_TO_ONE field, on the related object, that points back here

    // constructor / getters omitted for brevity
}
```

```java
// metadata/ObjectMetadata.java
public class ObjectMetadata {
    private final String objectName;   // "Customer", "Order", "Invoice__c" — unique, used as the API/table identifier
    private final String label;        // "Customer", "Sales Order" — what the UI shows
    private final List<FieldMetadata> fields;
    private final String origin;       // "CLASS" or "CUSTOM" — purely observational (§22 explains why it must stay that way)

    public FieldMetadata getField(String name) {
        return fields.stream().filter(f -> f.getName().equals(name)).findFirst()
            .orElseThrow(() -> new IllegalArgumentException("No such field: " + name));
    }
    // constructor / other getters omitted for brevity
}
```

This is deliberately close in shape to Django's own `Model._meta` (the object every `ModelAdmin` introspects to build its UI) and to Salesforce's Metadata API's `DescribeSObjectResult` (`fields`, `label`, relationship info) — MiniAdmin is reimplementing a well-established pattern, not inventing a new one.

---

# 9. Field Types and Their UI Widgets

Deciding the widget for a field is a pure function of its `FieldType` — this table is, in effect, the entire "auto-generate a form" feature, stated once, referenced everywhere:

| `FieldType` | Default UI widget | Notes |
|---|---|---|
| `STRING` | Single-line text input | |
| `TEXT` | Multi-line textarea | For longer free text (descriptions, notes) |
| `INTEGER` / `LONG` | Number input | |
| `DOUBLE` | Number input, decimal-enabled | |
| `BOOLEAN` | Checkbox | |
| `DATE` | Date picker | |
| `DATETIME` | Date + time picker | |
| `MANY_TO_ONE` | Searchable `<select>` dropdown (§33) | Options come from the related object's records, shown via `displayField` |
| `ONE_TO_MANY` | A read-only related-list panel (§34), never a form input | This field is never written to directly — it's computed by querying the *other* side's `MANY_TO_ONE` |

§32 turns this table directly into rendering code — naming the mapping explicitly here is what makes that later section a straightforward lookup instead of a pile of special cases.

---

# 10. Relationship Metadata: Many-to-One and One-to-Many

A **many-to-one** field (`Order.customer`) is the "real," stored side: the `Order` record physically holds a reference to one `Customer`. A **one-to-many** field (`Customer.orders`) is its mirror image — "every `Order` whose `customer` points at me" — and is never stored on `Customer` at all; it's computed on read by querying `Order` for `customer = thisCustomerId` (§27).

```java
Customer:
  orders: ONE_TO_MANY  { relatedObjectName: "Order", mappedByField: "customer" }

Order:
  customer: MANY_TO_ONE { relatedObjectName: "Customer", displayField: "name" }
```

Both class-based (`@OneToMany(mappedBy=...)`, §12) and UI-based (§21) definitions produce exactly this same `RelationshipMetadata` shape — which is precisely why §27's "resolve the related records for display" logic, and §34's "render a related-list panel" UI code, only need to be written once.

---

# 11. Phase 2 — The MetadataRegistry

```java
// metadata/MetadataRegistry.java
@Component // a MiniSpring bean, injectable anywhere via @Autowired — see MiniSpring guide §17
public class MetadataRegistry {
    private final Map<String, ObjectMetadata> objectsByName = new ConcurrentHashMap<>();

    public void register(ObjectMetadata metadata) {
        objectsByName.put(metadata.getObjectName(), metadata);
    }

    public ObjectMetadata get(String objectName) {
        ObjectMetadata metadata = objectsByName.get(objectName);
        if (metadata == null) throw new IllegalArgumentException("No such object: " + objectName);
        return metadata;
    }

    public boolean exists(String objectName) { return objectsByName.containsKey(objectName); }
    public Collection<ObjectMetadata> allObjects() { return objectsByName.values(); }
}
```

This is structurally identical to [TinyDB's `SchemaRegistry`](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) — a single-purpose, in-memory catalog that is the one place "what objects/tables exist" can be answered from. That's not a coincidence: both are the same pattern (a metadata catalog) solving the same problem (many consumers need a consistent view of "what exists") at two different layers of the same overall stack.

---

# 12. Phase 3 — Custom Annotations for Class-Based Models

```java
// classmodel/AdminEntity.java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface AdminEntity {
    String label() default ""; // defaults to the class's simple name if left blank
}

// classmodel/AdminField.java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface AdminField {
    String label() default "";
    boolean required() default false;
    int order() default 0;
}

// classmodel/AdminId.java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface AdminId { }

// classmodel/ManyToOne.java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface ManyToOne {
    Class<?> target();       // the related @AdminEntity class
    String displayField();   // which field of the related record to show in dropdowns/labels (§33)
}

// classmodel/OneToMany.java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface OneToMany {
    Class<?> target();
    String mappedBy();       // the name of the @ManyToOne field, ON the target class, that points back here
}
```

```java
// A worked example that §42 will use end to end
@AdminEntity(label = "Customer")
public class Customer {
    @AdminId private Long id;
    @AdminField(label = "Full Name", required = true, order = 1) private String name;
    @AdminField(label = "Email", order = 2) private String email;
    @OneToMany(target = Order.class, mappedBy = "customer") private List<Order> orders; // never persisted directly
}

@AdminEntity(label = "Order")
public class Order {
    @AdminId private Long id;
    @AdminField(label = "Order Number", required = true, order = 1) private String orderNumber;
    @ManyToOne(target = Customer.class, displayField = "name") private Customer customer;
    @AdminField(label = "Total", order = 2) private Double total;
}
```

This mirrors Django's own `models.py` almost line for line — a `CharField`/`ForeignKey`-style declaration is exactly what `@AdminField`/`@ManyToOne` are standing in for, and, like Django, **the class itself is both the domain model and the admin's source of truth** — there's no separate "admin config file" for the default behavior.

---

# 13. Phase 4 — Reflective Metadata Extraction From Annotated Classes

```java
// classmodel/ClassModelScanner.java
@Component
public class ClassModelScanner {

    public ObjectMetadata extract(Class<?> entityClass) {
        AdminEntity entityAnnotation = entityClass.getAnnotation(AdminEntity.class);
        if (entityAnnotation == null) {
            throw new IllegalArgumentException(entityClass.getName() + " is not annotated with @AdminEntity");
        }
        String objectName = entityClass.getSimpleName();
        String label = entityAnnotation.label().isBlank() ? objectName : entityAnnotation.label();

        List<FieldMetadata> fields = new ArrayList<>();
        for (Field field : entityClass.getDeclaredFields()) {
            fields.add(extractFieldMetadata(field));
        }
        fields.sort(Comparator.comparingInt(FieldMetadata::getDisplayOrder));

        return new ObjectMetadata(objectName, label, fields, "CLASS");
    }

    private FieldMetadata extractFieldMetadata(Field field) {
        boolean isId = field.isAnnotationPresent(AdminId.class);
        ManyToOne manyToOne = field.getAnnotation(ManyToOne.class);
        OneToMany oneToMany = field.getAnnotation(OneToMany.class);
        AdminField adminField = field.getAnnotation(AdminField.class);

        if (manyToOne != null) {
            String relatedName = manyToOne.target().getSimpleName();
            RelationshipMetadata relationship = new RelationshipMetadata(relatedName, manyToOne.displayField(), null);
            return new FieldMetadata(field.getName(), relatedName, FieldType.MANY_TO_ONE, false, false, 0, relationship);
        }
        if (oneToMany != null) {
            String relatedName = oneToMany.target().getSimpleName();
            RelationshipMetadata relationship = new RelationshipMetadata(relatedName, null, oneToMany.mappedBy());
            return new FieldMetadata(field.getName(), relatedName, FieldType.ONE_TO_MANY, false, false, 0, relationship);
        }
        if (isId) {
            return new FieldMetadata(field.getName(), "ID", inferScalarType(field), false, true, -1, null);
        }
        String label = (adminField != null && !adminField.label().isBlank()) ? adminField.label() : field.getName();
        boolean required = adminField != null && adminField.required();
        int order = adminField != null ? adminField.order() : 0;
        return new FieldMetadata(field.getName(), label, inferScalarType(field), required, false, order, null);
    }

    private FieldType inferScalarType(Field field) {
        Class<?> t = field.getType();
        if (t == String.class) return FieldType.STRING;
        if (t == Long.class || t == long.class) return FieldType.LONG;
        if (t == Integer.class || t == int.class) return FieldType.INTEGER;
        if (t == Double.class || t == double.class) return FieldType.DOUBLE;
        if (t == Boolean.class || t == boolean.class) return FieldType.BOOLEAN;
        if (t == LocalDate.class) return FieldType.DATE;
        if (t == LocalDateTime.class) return FieldType.DATETIME;
        throw new IllegalArgumentException("Unsupported field type: " + t.getName());
    }
}
```

This is the exact same reflective technique [MiniSpring's `ComponentScanner`](MiniSpring-Step-by-Step-Guide.md) uses to turn `@Component`-annotated classes into `BeanDefinition`s — walk `getDeclaredFields()`/`getAnnotations()`, build a plain data object, register it. MiniAdmin isn't introducing a new reflection technique here, only pointing an existing one at a new kind of annotation.

---

# 14. Phase 5 — Registering Class-Based Models as a MiniSpring Component

```java
// classmodel/ClassModelRegistrar.java
@Component
public class ClassModelRegistrar {
    private final ClassModelScanner scanner;
    private final MetadataRegistry metadataRegistry;
    private final List<Class<?>> entityClasses; // discovered by scanning the classpath for @AdminEntity, same technique as §13

    @Autowired
    public ClassModelRegistrar(ClassModelScanner scanner, MetadataRegistry metadataRegistry) {
        this.scanner = scanner;
        this.metadataRegistry = metadataRegistry;
        this.entityClasses = findAllAdminEntityClasses(); // reuses the classpath-walking code from MiniSpring §15
    }

    @PostConstruct // MiniSpring guide §33-§34 — runs once, right after this bean's dependencies are wired
    public void registerAllClassModels() {
        for (Class<?> entityClass : entityClasses) {
            metadataRegistry.register(scanner.extract(entityClass));
        }
    }
}
```

`@PostConstruct` is doing real, deliberate work here: it guarantees every class-based `ObjectMetadata` is registered **before** the `GenericCrudController` (§24) or the admin UI (§28) ever serves a request — the same "resolve everything at startup, fail fast" discipline [MiniSpring's `ApplicationContext.refresh()`](MiniSpring-Step-by-Step-Guide.md) already uses for bean wiring, applied here to model metadata instead.

---

# 15. Detecting and Resolving Relationships Between Class-Based Models

One subtlety `ClassModelScanner` (§13) glosses over: `Order.customer`'s `@ManyToOne(target = Customer.class, ...)` refers to `Customer` **by Java `Class`**, but `Customer.orders`'s `@OneToMany(mappedBy = "customer")` only knows the *field name* `"customer"` — there's no compile-time link forcing these two annotations to agree with each other. A validation pass at registration time catches a mismatch before it becomes a confusing runtime bug in the UI:

```java
// Added to ClassModelRegistrar.registerAllClassModels() (§14), after all objects are registered
private void validateRelationships() {
    for (ObjectMetadata object : metadataRegistry.allObjects()) {
        for (FieldMetadata field : object.getFields()) {
            if (field.getType() != FieldType.ONE_TO_MANY) continue;
            RelationshipMetadata rel = field.getRelationship();
            ObjectMetadata related = metadataRegistry.get(rel.getRelatedObjectName());
            FieldMetadata backReference = related.getField(rel.getMappedByField()); // throws if missing (§8)
            if (backReference.getType() != FieldType.MANY_TO_ONE
                    || !backReference.getRelationship().getRelatedObjectName().equals(object.getObjectName())) {
                throw new IllegalStateException(
                    object.getObjectName() + "." + field.getName() + " mappedBy=\"" + rel.getMappedByField()
                    + "\" does not point back to " + object.getObjectName());
            }
        }
    }
}
```

Catching this at startup — not the first time a user opens `Customer`'s related-orders panel (§34) — is the same "fail at boot, not in production traffic" principle [MiniSpring's eager bean instantiation](MiniSpring-Step-by-Step-Guide.md) and [TinyDB's schema validation](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) both already commit to.

---

# 16. Two Storage Strategies: Physical Tables vs Entity-Attribute-Value (Why Salesforce Uses EAV)

A class-based `Customer` object has a fixed shape known at compile time — a real table with real `name`/`email` columns (§17) is the obvious, efficient choice, and it's exactly what [TinyDB's `CREATE TABLE`](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) already supports.

A UI-defined Custom Object is different in one crucial way: **its shape can change at any moment, from a browser, with no deploy** — someone can click "Add Field" on a live system at 2pm. If every Custom Object were backed by its own real table, adding a field would mean issuing an `ALTER TABLE ... ADD COLUMN` against a live production table on every single field addition — workable at small scale, but exactly the kind of schema-migration risk (locking, replication lag while the DDL propagates, no easy per-tenant isolation) that a *multi-tenant* platform serving thousands of orgs, each defining their own objects, cannot take on for every customer's every field click.

This is, concretely, why Salesforce's real platform does not give every org's every Custom Object its own physical table. Salesforce's actual architecture (publicly documented as the "Universal Data Dictionary") stores **all** custom field values from **every** org in a small number of enormous, generic, pre-allocated tables, and uses metadata to know which generic column means what for which object. MiniAdmin's `EavStorage` (§19) is a simplified version of exactly that idea:

| | `PhysicalTableStorage` (§17) | `EavStorage` (§19) |
|---|---|---|
| Backing structure | One real table per object, real typed columns | One shared table: `(object_name, record_id, field_name, field_value)` |
| Adding a field | Requires `ALTER TABLE` (or, for MiniAdmin, isn't supported at all for class-based models — you add a field in code and redeploy) | Just start writing rows with a new `field_name` — no schema change |
| Query performance | Fast — native columns, native types, indexable directly | Slower — a "get all fields for this record" query is a self-join or multiple lookups against the shared table |
| Used for | Class-based (`@AdminEntity`) models | UI-based Custom Objects |

Naming this trade-off explicitly is the point: EAV is not "the modern way to do it" — it's a deliberate trade of query performance for schema agility, taken **only** where that agility is the actual requirement (a field added from a UI, live), which is exactly the condition class-based models don't have.

---

# 17. Phase 6 — PhysicalTableStorage for Class-Based Models

```java
// storage/ObjectStorageStrategy.java — the interface BOTH storage strategies implement
public interface ObjectStorageStrategy {
    String save(ObjectMetadata metadata, Map<String, Object> fieldValues) throws SQLException; // returns the record id
    Map<String, Object> findById(ObjectMetadata metadata, String id) throws SQLException;
    List<Map<String, Object>> findAll(ObjectMetadata metadata) throws SQLException;
    List<Map<String, Object>> findWhere(ObjectMetadata metadata, String fieldName, String value) throws SQLException; // used by §27
    void deleteById(ObjectMetadata metadata, String id) throws SQLException;
}
```

```java
// storage/PhysicalTableStorage.java
public class PhysicalTableStorage implements ObjectStorageStrategy {
    private final MiniConnectionPool pool; // the same pool built in the MiniTomcat companion guide's §32-§34

    public PhysicalTableStorage(MiniConnectionPool pool) { this.pool = pool; }

    @Override
    public String save(ObjectMetadata metadata, Map<String, Object> fieldValues) throws SQLException {
        boolean isUpdate = fieldValues.get("id") != null;
        String sql = isUpdate ? buildUpdateSql(metadata) : buildInsertSql(metadata);
        try (Connection connection = pool.borrow(); PreparedStatement ps = connection.prepareStatement(sql)) {
            bindParameters(ps, metadata, fieldValues);
            ps.executeUpdate();
            return String.valueOf(fieldValues.getOrDefault("id", generatedId(ps)));
        } catch (InterruptedException e) {
            throw new SQLException(e);
        }
    }

    @Override
    public List<Map<String, Object>> findAll(ObjectMetadata metadata) throws SQLException {
        String sql = "SELECT * FROM " + metadata.getObjectName(); // the object's real table, created once at CREATE TABLE time (§18)
        try (Connection connection = pool.borrow(); PreparedStatement ps = connection.prepareStatement(sql);
             ResultSet rs = ps.executeQuery()) {
            return mapRows(rs, metadata);
        } catch (InterruptedException e) {
            throw new SQLException(e);
        }
    }
    // findById/findWhere/deleteById/buildInsertSql/buildUpdateSql/bindParameters/mapRows omitted for brevity —
    // straightforward JDBC, one real column per FieldMetadata, exactly like §26 of the TinyDB companion guide.
}
```

`PhysicalTableStorage` is, deliberately, nothing more than a thin, metadata-driven JDBC repository — it earns its keep entirely from **not being written per entity**: the same class serves `Customer`, `Order`, and any future `@AdminEntity` class, because every SQL statement it builds is derived from `ObjectMetadata.getFields()`, never from a hard-coded column list.

---

# 18. Phase 7 — The Custom Object Admin API (Salesforce-Style Object Builder)

Before a Custom Object can be defined *from the UI*, MiniAdmin needs somewhere durable to store that definition — the UI-driven equivalent of a `.java` file. Two tables, created once via MiniAdmin's own bootstrap SQL, hold this:

```sql
CREATE TABLE admin_object_def (
    object_name VARCHAR PRIMARY KEY,
    label VARCHAR
);
CREATE TABLE admin_field_def (
    id VARCHAR PRIMARY KEY,
    object_name VARCHAR,      -- foreign key, by convention, to admin_object_def
    field_name VARCHAR,
    label VARCHAR,
    field_type VARCHAR,
    required BOOLEAN,
    related_object_name VARCHAR,  -- null unless field_type is MANY_TO_ONE/ONE_TO_MANY
    display_field VARCHAR,
    mapped_by_field VARCHAR,
    display_order INT
);
```

```java
// customobject/AdminObjectDefRepository.java
@Repository
public class AdminObjectDefRepository {
    private final MiniConnectionPool pool;

    public void saveObjectDef(String objectName, String label) throws SQLException { /* INSERT INTO admin_object_def */ }
    public void saveFieldDef(String objectName, FieldMetadata field) throws SQLException { /* INSERT INTO admin_field_def */ }
    public List<ObjectMetadata> loadAllCustomObjects() throws SQLException {
        // SELECT from admin_object_def, join admin_field_def, reassemble into ObjectMetadata objects — the
        // mirror image of ClassModelScanner (§13): that one reads Java annotations, this one reads database rows,
        // and BOTH produce the exact same ObjectMetadata/FieldMetadata shape (§8).
        return List.of();
    }
}
```

These two tables are themselves ordinary class-based `@AdminEntity`-shaped data — MiniAdmin's own metadata is stored the same way any application's data would be, just consumed by MiniAdmin itself rather than rendered in its own generated UI.

---

# 19. Phase 8 — EavStorage for UI-Defined Custom Objects

```sql
CREATE TABLE eav_record (
    id VARCHAR PRIMARY KEY,
    object_name VARCHAR
);
CREATE TABLE eav_value (
    record_id VARCHAR,
    field_name VARCHAR,
    field_value VARCHAR,       -- everything stored as text; FieldMetadata.type governs how it's parsed back (§25)
    PRIMARY KEY (record_id, field_name)
);
```

```java
// storage/EavStorage.java
public class EavStorage implements ObjectStorageStrategy {
    private final MiniConnectionPool pool;

    @Override
    public String save(ObjectMetadata metadata, Map<String, Object> fieldValues) throws SQLException {
        String recordId = (String) fieldValues.getOrDefault("id", UUID.randomUUID().toString());
        try (Connection connection = pool.borrow()) {
            connection.setAutoCommit(false); // every field write for one record is one atomic transaction
            try (PreparedStatement upsertRecord = connection.prepareStatement(
                    "INSERT INTO eav_record (id, object_name) VALUES (?, ?) ON CONFLICT DO NOTHING")) {
                upsertRecord.setString(1, recordId);
                upsertRecord.setString(2, metadata.getObjectName());
                upsertRecord.executeUpdate();
            }
            for (FieldMetadata field : metadata.getFields()) {
                if (field.getType() == FieldType.ONE_TO_MANY) continue; // never stored — computed on read, §27
                Object value = fieldValues.get(field.getName());
                if (value == null) continue;
                try (PreparedStatement upsertValue = connection.prepareStatement(
                        "INSERT INTO eav_value (record_id, field_name, field_value) VALUES (?, ?, ?) "
                        + "ON CONFLICT (record_id, field_name) DO UPDATE SET field_value = ?")) {
                    upsertValue.setString(1, recordId);
                    upsertValue.setString(2, field.getName());
                    upsertValue.setString(3, String.valueOf(value));
                    upsertValue.setString(4, String.valueOf(value));
                    upsertValue.executeUpdate();
                }
            }
            connection.commit();
            return recordId;
        } catch (InterruptedException e) {
            throw new SQLException(e);
        }
    }

    @Override
    public Map<String, Object> findById(ObjectMetadata metadata, String id) throws SQLException {
        // SELECT field_name, field_value FROM eav_value WHERE record_id = ?  -- one row per field, pivoted
        // back into a single Map<String,Object> keyed by field name, exactly like a real row would look
        // to PhysicalTableStorage's caller — GenericRepository (§23) never has to know which strategy answered it.
        return Map.of();
    }
    // findAll/findWhere/deleteById omitted for brevity — findAll in particular needs one query against eav_record
    // filtered by object_name, then a per-record (or batched) join against eav_value.
}
```

`ON CONFLICT ... DO UPDATE` — an upsert — is what lets `save()` handle both "first time this field is set" and "this field already had a value" without the caller needing to know which case applies; adding a brand-new field to a Custom Object later (§21) requires **zero** change to this class, because `eav_value` never had a fixed set of columns to begin with.

---

# 20. Phase 9 — The Custom Object Builder: Creating an Object From the UI

```java
// customobject/CustomObjectService.java
@Service
public class CustomObjectService {
    private final MetadataRegistry metadataRegistry;
    private final AdminObjectDefRepository objectDefRepository;

    public ObjectMetadata createObject(String objectName, String label) throws SQLException {
        if (metadataRegistry.exists(objectName)) {
            throw new IllegalArgumentException("Object already exists: " + objectName);
        }
        FieldMetadata idField = new FieldMetadata("id", "ID", FieldType.STRING, true, true, -1, null);
        ObjectMetadata metadata = new ObjectMetadata(objectName, label, List.of(idField), "CUSTOM");

        objectDefRepository.saveObjectDef(objectName, label); // durable — survives a restart (§18)
        metadataRegistry.register(metadata);                   // visible immediately — no deploy, no restart needed
        return metadata;
    }
}
```

```java
// customobject/CustomObjectController.java
@Controller
@RequestMapping("/api/admin/objects")
public class CustomObjectController {
    private final CustomObjectService customObjectService;

    @PostMapping("")
    public ResponseEntity<ObjectMetadata> createObject(@RequestBody CreateObjectRequest request) throws SQLException {
        return ResponseEntity.ok(customObjectService.createObject(request.objectName(), request.label()));
    }
    // record CreateObjectRequest(String objectName, String label) {}
}
```

```bash
curl -X POST http://localhost:8080/api/admin/objects \
  -H "Content-Type: application/json" \
  -d '{"objectName": "Product", "label": "Product"}'
```

The moment that `POST` returns, `Product` exists as a real, queryable object — the admin UI's object list (§29) shows it on the next page load, with **no server restart, no deploy, no `CREATE TABLE`** — precisely because `EavStorage` (§19) never needed a physical table to exist in the first place.

---

# 21. Phase 10 — Adding Fields to a Custom Object Without a Schema Migration

```java
// customobject/CustomObjectService.java — continued
public ObjectMetadata addField(String objectName, FieldMetadata newField) throws SQLException {
    ObjectMetadata existing = metadataRegistry.get(objectName);
    if (!"CUSTOM".equals(existing.getOrigin())) {
        throw new IllegalArgumentException("Cannot add fields to a class-based object at runtime: " + objectName);
    }
    List<FieldMetadata> updatedFields = new ArrayList<>(existing.getFields());
    updatedFields.add(newField);
    ObjectMetadata updated = new ObjectMetadata(objectName, existing.getLabel(), updatedFields, "CUSTOM");

    objectDefRepository.saveFieldDef(objectName, newField); // durable
    metadataRegistry.register(updated);                      // replaces the old ObjectMetadata — visible immediately
    return updated;
}
```

```bash
curl -X POST http://localhost:8080/api/admin/objects/Product/fields \
  -H "Content-Type: application/json" \
  -d '{"name": "price", "label": "Price", "type": "DOUBLE", "required": true}'
```

Compare this to what the same operation would require under `PhysicalTableStorage`: an `ALTER TABLE Product ADD COLUMN price DOUBLE`, run against a live table, with all the locking/replication considerations the [TinyDB guide's DDL section](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) discusses for `CREATE TABLE`. Under `EavStorage`, adding a field is **only a metadata change** — the very first `Product` row that sets a `price` simply inserts a new `eav_value` row with `field_name = 'price'`; no existing data, and no existing table structure, is touched at all. This is the concrete payoff of §16's trade-off, not just a claim about it.

**Explicitly guarding against adding fields to a class-based object at runtime** (the `IllegalArgumentException` above) matters: that would create metadata (`Customer.newField`) with no matching Java field and no matching table column — a state `PhysicalTableStorage` (§17) has no way to honor. The check keeps each storage strategy's contract intact rather than letting the two paths silently corrupt each other's assumptions.

---

# 22. Unifying Both Paths Into One MetadataRegistry and One ObjectStorageStrategy Interface

With §14 and §20 both calling `metadataRegistry.register(...)`, and both `PhysicalTableStorage` and `EavStorage` implementing the same `ObjectStorageStrategy` interface (§17), everything built from §23 onward can be written against exactly two abstractions — `ObjectMetadata` and `ObjectStorageStrategy` — and genuinely never needs an `if (origin.equals("CLASS"))` branch anywhere in the CRUD or UI layers:

```java
// The ONE place that decides which storage strategy backs a given object
@Component
public class StorageStrategyResolver {
    private final PhysicalTableStorage physicalTableStorage;
    private final EavStorage eavStorage;

    public ObjectStorageStrategy resolve(ObjectMetadata metadata) {
        return "CLASS".equals(metadata.getOrigin()) ? physicalTableStorage : eavStorage;
    }
}
```

`ObjectMetadata.origin` (§8) exists **only** so this one class can make this one decision — every other consumer of `ObjectMetadata` in this guide (§23's `GenericRepository`, §24's `GenericCrudController`, §30–§32's UI rendering) reads `objectName`, `fields`, and relationships, and is never given a reason to branch on origin at all. That containment is deliberate: it's the difference between "one clean seam" and "the CLASS/CUSTOM distinction leaking into forty files."

---

# 23. Phase 11 — A Generic Repository Driven by Metadata and Storage Strategy

```java
// repository/GenericRepository.java
@Repository
public class GenericRepository {
    private final MetadataRegistry metadataRegistry;
    private final StorageStrategyResolver strategyResolver;

    @Autowired
    public GenericRepository(MetadataRegistry metadataRegistry, StorageStrategyResolver strategyResolver) {
        this.metadataRegistry = metadataRegistry;
        this.strategyResolver = strategyResolver;
    }

    public String save(String objectName, Map<String, Object> fieldValues) throws SQLException {
        ObjectMetadata metadata = metadataRegistry.get(objectName);
        validate(metadata, fieldValues); // §25
        return strategyResolver.resolve(metadata).save(metadata, fieldValues);
    }

    public Map<String, Object> findById(String objectName, String id) throws SQLException {
        ObjectMetadata metadata = metadataRegistry.get(objectName);
        return strategyResolver.resolve(metadata).findById(metadata, id);
    }

    public List<Map<String, Object>> findAll(String objectName) throws SQLException {
        ObjectMetadata metadata = metadataRegistry.get(objectName);
        return strategyResolver.resolve(metadata).findAll(metadata);
    }

    public void deleteById(String objectName, String id) throws SQLException {
        ObjectMetadata metadata = metadataRegistry.get(objectName);
        strategyResolver.resolve(metadata).deleteById(metadata, id);
    }
}
```

This single class is **the entire data-access layer for every object MiniAdmin will ever serve** — there is no `CustomerRepository`, no `OrderRepository`, no `ProductRepository`. Every one of those would, in a hand-written MiniSpring app, look almost identical to every other one (per the [MiniSpring guide's own `@Repository` examples](MiniSpring-Step-by-Step-Guide.md)) — `GenericRepository` exists specifically because that repetition is exactly what metadata-driven design is for.

---

# 24. Phase 12 — A Generic CRUD REST Controller for Any Object

```java
// web/GenericCrudController.java
@Controller
@RequestMapping("/api/data/{objectName}")
public class GenericCrudController {
    private final GenericRepository repository;

    @Autowired
    public GenericCrudController(GenericRepository repository) { this.repository = repository; }

    @GetMapping("")
    public ResponseEntity<List<Map<String, Object>>> list(@PathVariable("objectName") String objectName) throws SQLException {
        return ResponseEntity.ok(repository.findAll(objectName));
    }

    @GetMapping("/{id}")
    public ResponseEntity<Map<String, Object>> get(@PathVariable("objectName") String objectName,
                                                     @PathVariable("id") String id) throws SQLException {
        Map<String, Object> record = repository.findById(objectName, id);
        return record == null ? ResponseEntity.status(404).build() : ResponseEntity.ok(record);
    }

    @PostMapping("")
    public ResponseEntity<Map<String, Object>> create(@PathVariable("objectName") String objectName,
                                                        @RequestBody Map<String, Object> fieldValues) throws SQLException {
        String id = repository.save(objectName, fieldValues);
        return ResponseEntity.ok(repository.findById(objectName, id));
    }

    @PutMapping("/{id}")
    public ResponseEntity<Map<String, Object>> update(@PathVariable("objectName") String objectName,
                                                        @PathVariable("id") String id,
                                                        @RequestBody Map<String, Object> fieldValues) throws SQLException {
        fieldValues.put("id", id);
        repository.save(objectName, fieldValues);
        return ResponseEntity.ok(repository.findById(objectName, id));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable("objectName") String objectName,
                                        @PathVariable("id") String id) throws SQLException {
        repository.deleteById(objectName, id);
        return ResponseEntity.noContent().build();
    }
}
```

Five `@GetMapping`/`@PostMapping`/`@PutMapping`/`@DeleteMapping` methods, registered **once**, at `/api/data/{objectName}`, serve `Customer`, `Order`, `Product`, and every object created after this guide is finished. `{objectName}` is resolved by [MiniSpring's `@PathVariable` mechanism](MiniSpring-Step-by-Step-Guide.md) exactly like any other path variable — nothing about MiniSpring's routing needed to change to make a *variable* segment mean "which object," because that was already the framework's job.

---

# 25. Phase 13 — Metadata-Driven Validation

```java
// repository/GenericRepository.java — the validate() method referenced in §23
private void validate(ObjectMetadata metadata, Map<String, Object> fieldValues) {
    List<String> errors = new ArrayList<>();
    for (FieldMetadata field : metadata.getFields()) {
        if (field.getType() == FieldType.ONE_TO_MANY) continue; // never submitted, never validated as an input
        Object value = fieldValues.get(field.getName());
        if (field.isRequired() && (value == null || value.toString().isBlank())) {
            errors.add(field.getLabel() + " is required");
            continue;
        }
        if (value != null && !matchesType(value, field.getType())) {
            errors.add(field.getLabel() + " must be a " + field.getType());
        }
    }
    if (!errors.isEmpty()) throw new ValidationException(errors);
}

private boolean matchesType(Object value, FieldType type) {
    String s = value.toString();
    return switch (type) {
        case INTEGER, LONG -> s.matches("-?\\d+");
        case DOUBLE -> s.matches("-?\\d+(\\.\\d+)?");
        case BOOLEAN -> s.equalsIgnoreCase("true") || s.equalsIgnoreCase("false");
        case DATE, DATETIME -> isParsableDate(s, type);
        default -> true; // STRING/TEXT/MANY_TO_ONE accept any non-blank string here
    };
}
```

This is the same "required" this guide already saw once, in [MiniSpring's `Schema.validate(row)`](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) — a `FieldMetadata.required` flag rejecting an invalid write **before** it reaches storage — except here it runs identically for *every* object, because it reads the requirement from metadata rather than from a hand-written `if` per entity. §36 reuses this exact same rule set on the client side, so a user sees "Full Name is required" before ever submitting the form, not just after.

---

# 26. Phase 14 — Saving Many-to-One Foreign Keys

A `MANY_TO_ONE` field's *value*, from the UI's perspective, is just the id of the related record — the dropdown (§33) submits `{"customer": "17"}`, not a nested `Customer` object. `PhysicalTableStorage`/`EavStorage` (§17, §19) already handle this correctly with **no special-casing**, because a foreign key is, at the storage layer, indistinguishable from any other string/id-shaped value:

```java
// PhysicalTableStorage's buildInsertSql (§17) treats a MANY_TO_ONE field exactly like a STRING/LONG column —
// the column is named after the field ("customer") and holds the related record's id, nothing more.
INSERT INTO Order (id, orderNumber, customer, total) VALUES (?, ?, ?, ?)
--                                    ^^^^^^^^ just the customer's id, e.g. "17"
```

The relationship *metadata* (§10) only matters at the two points that need to interpret that id as a reference: rendering it as a dropdown pre-selected to the current value (§33), and resolving it into a human-readable label (`displayField`) instead of showing a raw id in the list view (§30). Storage itself stays completely relationship-agnostic — which is precisely why `GenericRepository` (§23) never needed a special code path for `MANY_TO_ONE` fields at all.

---

# 27. Phase 15 — Resolving One-to-Many Related Records for Display

Unlike `MANY_TO_ONE`, a `ONE_TO_MANY` field is **never** part of `fieldValues` on save (§19, §25 both skip it explicitly) — it's computed only when a record is being *displayed*, by querying the other side:

```java
// web/GenericCrudController.java — a dedicated endpoint for one-to-many related records
@GetMapping("/{id}/related/{relatedFieldName}")
public ResponseEntity<List<Map<String, Object>>> related(@PathVariable("objectName") String objectName,
                                                           @PathVariable("id") String id,
                                                           @PathVariable("relatedFieldName") String relatedFieldName) throws SQLException {
    ObjectMetadata metadata = metadataRegistry.get(objectName);
    FieldMetadata field = metadata.getField(relatedFieldName); // e.g. Customer.orders
    RelationshipMetadata rel = field.getRelationship();
    // "find every Order whose `customer` field equals this Customer's id" — ObjectStorageStrategy.findWhere (§17)
    return ResponseEntity.ok(repository.findWhere(rel.getRelatedObjectName(), rel.getMappedByField(), id));
}
```

```bash
curl http://localhost:8080/api/data/Customer/17/related/orders
# -> every Order where Order.customer = "17" — Customer.orders itself was never stored anywhere
```

This is the mechanical answer to "how does a one-to-many relationship show up in the UI": it's a **query**, triggered by opening a specific record's detail view (§34), never a stored value — exactly mirroring how Django's admin computes a reverse foreign-key's inline list, and how Salesforce computes a Related List panel.

---

# 28. Choosing the UI Approach: A Metadata-Driven Admin Frontend

Django's admin site is server-rendered Python templates that introspect `ModelAdmin`/`Model._meta` on every request. MiniAdmin makes a different but equivalent choice: a **single, generic, static HTML+JS admin shell** (`admin.html`/`admin.js`) that fetches `ObjectMetadata` (§29) and record data (§24) as JSON and renders the list/form/dropdown UI **entirely in the browser**, driven by the same metadata every other layer of this guide already uses.

The reasoning is the same one that shaped every earlier decision in this guide: **write the rendering logic once, against `ObjectMetadata`, and never again per object.** A server-templated approach (Django's route) would work too — the metadata-to-widget mapping (§9) is identical either way — but a JSON API plus a generic client keeps MiniAdmin's UI reusable by something other than a browser (a native app, a CLI, another service) for free, the same benefit [TinyDB's wire protocol](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) got from not being HTML-shaped in the first place.

```java
// web/ObjectMetadataController.java — the endpoint the admin frontend calls before rendering anything
@Controller
@RequestMapping("/api/meta")
public class ObjectMetadataController {
    private final MetadataRegistry metadataRegistry;

    @GetMapping("/objects")
    public ResponseEntity<Collection<ObjectMetadata>> listObjects() {
        return ResponseEntity.ok(metadataRegistry.allObjects());
    }

    @GetMapping("/objects/{objectName}")
    public ResponseEntity<ObjectMetadata> getObject(@PathVariable("objectName") String objectName) {
        return ResponseEntity.ok(metadataRegistry.get(objectName));
    }
}
```

---

# 29. Phase 16 — The Admin Shell and Object Navigation

```html
<!-- static/admin/admin.html — the entire admin UI lives in this one page -->
<!doctype html>
<html>
<head><link rel="stylesheet" href="admin.css"></head>
<body>
  <nav id="object-nav"></nav>       <!-- populated from GET /api/meta/objects, §28 -->
  <main id="content"></main>        <!-- list view (§30), form view (§31), or detail view (§34) -->
  <script src="admin.js"></script>
</body>
</html>
```

```javascript
// static/admin/admin.js
async function renderNav() {
    const objects = await fetch('/api/meta/objects').then(r => r.json());
    const nav = document.getElementById('object-nav');
    nav.innerHTML = objects.map(o =>
        `<a href="#/${o.objectName}" data-object="${o.objectName}">${o.label}</a>`
    ).join('');
}

window.addEventListener('hashchange', route);
window.addEventListener('load', () => { renderNav(); route(); });

function route() {
    const [, objectName, id] = location.hash.split('/'); // "#/Customer" or "#/Customer/17"
    if (!objectName) return;
    id ? renderDetail(objectName, id) : renderList(objectName); // §30, §34
}
```

This sidebar is the direct equivalent of Django admin's app index page (the list of registered models) — built from `GET /api/meta/objects`, it lists `Customer`, `Order`, and `Product` (§20) side by side, with **no code distinguishing which one is class-based and which is a Custom Object**, exactly per §22's design goal.

---

# 30. Phase 17 — Auto-Generating the List View

```javascript
// static/admin/admin.js — continued
async function renderList(objectName) {
    const metadata = await fetch(`/api/meta/objects/${objectName}`).then(r => r.json());
    const records = await fetch(`/api/data/${objectName}`).then(r => r.json());
    const columns = metadata.fields.filter(f => f.type !== 'ONE_TO_MANY'); // never a list column — §9, §27

    const content = document.getElementById('content');
    content.innerHTML = `
        <h1>${metadata.label}</h1>
        <button onclick="location.hash='#/${objectName}/new'">New ${metadata.label}</button>
        <table>
            <thead><tr>${columns.map(c => `<th>${c.label}</th>`).join('')}</tr></thead>
            <tbody>${records.map(r => renderRow(objectName, r, columns)).join('')}</tbody>
        </table>`;
}

function renderRow(objectName, record, columns) {
    const cells = columns.map(c => `<td>${formatCellValue(record[c.name], c)}</td>`).join('');
    return `<tr onclick="location.hash='#/${objectName}/${record.id}'">${cells}</tr>`;
}

function formatCellValue(value, column) {
    if (column.type === 'BOOLEAN') return value ? 'Yes' : 'No';
    if (column.type === 'MANY_TO_ONE') return value?.displayLabel ?? value; // resolved server-side, §33
    return value ?? '';
}
```

Every column header, every row's cell rendering, and the very existence of a "New" button are all derived from the `metadata` object fetched a moment earlier — swap `Customer` for `Product` in the URL hash and the exact same function renders a completely different table, with zero MiniAdmin code aware that `Product` exists.

---

# 31. Phase 18 — Auto-Generating the Create/Edit Form

```javascript
async function renderForm(objectName, existingRecord) {
    const metadata = await fetch(`/api/meta/objects/${objectName}`).then(r => r.json());
    const fields = metadata.fields.filter(f => f.type !== 'ONE_TO_MANY'); // computed-only fields never appear as inputs

    const content = document.getElementById('content');
    content.innerHTML = `
        <h1>${existingRecord ? 'Edit' : 'New'} ${metadata.label}</h1>
        <form id="record-form">
            ${(await Promise.all(fields.map(f => renderFieldWidget(f, existingRecord)))).join('')}
            <button type="submit">Save</button>
        </form>`;

    document.getElementById('record-form').addEventListener('submit', e => {
        e.preventDefault();
        submitForm(objectName, fields, existingRecord?.id);
    });
}

async function submitForm(objectName, fields, existingId) {
    const formData = new FormData(document.getElementById('record-form'));
    const fieldValues = Object.fromEntries(fields.map(f => [f.name, formData.get(f.name)]));
    if (!validateClientSide(fields, fieldValues)) return; // §36 — mirrors §25's server-side rules

    const url = existingId ? `/api/data/${objectName}/${existingId}` : `/api/data/${objectName}`;
    const method = existingId ? 'PUT' : 'POST';
    await fetch(url, { method, headers: {'Content-Type': 'application/json'}, body: JSON.stringify(fieldValues) });
    location.hash = `#/${objectName}`;
}
```

§32 fills in `renderFieldWidget` — the function that turns §9's type-to-widget table into actual `<input>` markup.

---

# 32. Mapping Field Types to Input Widgets

```javascript
// static/admin/admin.js — renderFieldWidget, directly implementing §9's table
async function renderFieldWidget(field, existingRecord) {
    const value = existingRecord ? existingRecord[field.name] : '';
    const requiredAttr = field.required ? 'required' : '';
    const label = `<label>${field.label}${field.required ? ' *' : ''}</label>`;

    switch (field.type) {
        case 'TEXT':
            return `${label}<textarea name="${field.name}" ${requiredAttr}>${value}</textarea>`;
        case 'BOOLEAN':
            return `${label}<input type="checkbox" name="${field.name}" ${value ? 'checked' : ''}>`;
        case 'INTEGER': case 'LONG': case 'DOUBLE':
            return `${label}<input type="number" name="${field.name}" value="${value}" ${requiredAttr}>`;
        case 'DATE':
            return `${label}<input type="date" name="${field.name}" value="${value}" ${requiredAttr}>`;
        case 'DATETIME':
            return `${label}<input type="datetime-local" name="${field.name}" value="${value}" ${requiredAttr}>`;
        case 'MANY_TO_ONE':
            return await renderManyToOneDropdown(field, value); // §33
        default: // STRING and the primary key field
            return `${label}<input type="text" name="${field.name}" value="${value}" ${requiredAttr}>`;
    }
}
```

This function is, in a very literal sense, §9's table transcribed into code — extending MiniAdmin with a brand-new scalar type (say, `EMAIL`, rendered as `<input type="email">` with built-in browser validation) means adding one row to §9's table and one `case` here, nowhere else.

---

# 33. Phase 19 — Rendering Many-to-One Fields as Dropdowns

```java
// web/LookupController.java — powers the dropdown's option list
@Controller
@RequestMapping("/api/lookup")
public class LookupController {
    private final GenericRepository repository;
    private final MetadataRegistry metadataRegistry;

    @GetMapping("/{objectName}")
    public ResponseEntity<List<Map<String, String>>> options(@PathVariable("objectName") String objectName,
                                                               @RequestParam(value = "q", defaultValue = "") String search) throws SQLException {
        ObjectMetadata metadata = metadataRegistry.get(objectName);
        String displayField = inferDisplayField(metadata); // first non-id STRING field, unless overridden (§37)
        return ResponseEntity.ok(repository.findAll(objectName).stream()
            .filter(record -> search.isBlank() || String.valueOf(record.get(displayField)).toLowerCase().contains(search.toLowerCase()))
            .map(record -> Map.of("id", String.valueOf(record.get("id")), "label", String.valueOf(record.get(displayField))))
            .toList());
    }
}
```

```javascript
// static/admin/admin.js — renderManyToOneDropdown
async function renderManyToOneDropdown(field, currentValue) {
    const options = await fetch(`/api/lookup/${field.relationship.relatedObjectName}`).then(r => r.json());
    const optionTags = options.map(o =>
        `<option value="${o.id}" ${o.id === currentValue ? 'selected' : ''}>${o.label}</option>`
    ).join('');
    return `<label>${field.label}</label><select name="${field.name}">${optionTags}</select>`;
}
```

This is the direct answer to "any one-to-many or many-to-one relationship will also be visible on the UI, like the dropdown": `Order`'s form renders `customer` as a `<select>` populated from `GET /api/lookup/Customer`, showing each customer's `name` (§8's `displayField`) instead of a meaningless numeric id — and because `LookupController` is, like `GenericCrudController`, driven entirely by `{objectName}`, the exact same endpoint and the exact same dropdown-rendering code serve *any* `MANY_TO_ONE` field on *any* object.

---

# 34. Phase 20 — Rendering One-to-Many Relationships as Related Lists

```javascript
async function renderDetail(objectName, id) {
    const metadata = await fetch(`/api/meta/objects/${objectName}`).then(r => r.json());
    const record = await fetch(`/api/data/${objectName}/${id}`).then(r => r.json());
    await renderForm(objectName, record); // the same form from §31, pre-filled

    const relatedFields = metadata.fields.filter(f => f.type === 'ONE_TO_MANY');
    for (const field of relatedFields) {
        const related = await fetch(`/api/data/${objectName}/${id}/related/${field.name}`).then(r => r.json()); // §27
        document.getElementById('content').insertAdjacentHTML('beforeend', renderRelatedListPanel(field, related));
    }
}

function renderRelatedListPanel(field, relatedRecords) {
    return `<section class="related-list">
        <h2>${field.relatedObjectLabel ?? field.name} (${relatedRecords.length})</h2>
        <ul>${relatedRecords.map(r => `<li>${r.id}: ${JSON.stringify(r)}</li>`).join('')}</ul>
    </section>`;
}
```

Opening a `Customer` record now shows an "Orders (3)" panel underneath the edit form, populated by §27's query — this is MiniAdmin's version of a Salesforce Related List or a Django admin `TabularInline`, and like both of those, it required no field on `Customer` to be written to, because `ONE_TO_MANY` was never a storable value (§10, §19) to begin with.

---

# 35. Phase 21 — Wiring the UI to the Generic CRUD API

At this point every piece from §29–§34 composes into the full loop with no gaps: the nav (§29) lists objects from `/api/meta/objects`; clicking one renders a list (§30) from `/api/data/{objectName}`; clicking "New" or a row renders a form (§31) whose widgets (§32–§33) are chosen from the same metadata; submitting the form `POST`s or `PUT`s to `/api/data/{objectName}` (§24), which validates (§25) and saves through the correct storage strategy (§22) without the browser ever knowing which one; and opening an existing record additionally renders related-list panels (§34) from a second metadata-driven query (§27). Nothing in this loop mentions `Customer`, `Order`, or `Product` by name anywhere in the JavaScript or the generic Java controllers — every name-specific detail lives entirely in metadata.

---

# 36. Phase 22 — Client-Side Validation Mirrored From Metadata

```javascript
// static/admin/admin.js — validateClientSide, called from submitForm (§31) before the network request fires
function validateClientSide(fields, fieldValues) {
    const errors = [];
    for (const field of fields) {
        const value = fieldValues[field.name];
        if (field.required && (!value || value.trim() === '')) {
            errors.push(`${field.label} is required`);
        }
    }
    if (errors.length > 0) {
        alert(errors.join('\n')); // a real UI would render these inline next to each field
        return false;
    }
    return true;
}
```

This is a deliberately **duplicated** rule set, not a shortcut around §25's server-side validation — the client-side check exists purely for a fast, no-round-trip user experience; §25's server-side check is the one that actually protects data integrity, because the client-side copy can always be bypassed (a direct `curl` to `/api/data/Customer`, for instance). Both read from the *same* `required` flag in the *same* `ObjectMetadata`, so the two checks can never drift into disagreeing with each other about which fields are mandatory.

---

# 37. Phase 23 — Customization Hooks: Per-Object UI Configuration

Everything through §36 produces *sensible defaults* — the same value proposition Django admin and Flask-Admin make. Real usage always needs to override some of it: hide an internal field, change a column's label, pick a non-default `displayField`. `ObjectUiConfig` is the one seam every customization in this guide goes through, rather than each need inventing its own mechanism:

```java
// customization/ObjectUiConfig.java
public class ObjectUiConfig {
    private final String objectName;
    private final Set<String> hiddenFields = new HashSet<>();
    private final Map<String, String> fieldLabelOverrides = new HashMap<>();
    private final List<String> listViewColumnOrder = new ArrayList<>(); // empty = use FieldMetadata.displayOrder
    private final String displayFieldOverride; // used by §33's lookup instead of "first STRING field"

    // fluent builder-style setters, e.g. hideField(String), overrideLabel(String, String) — omitted for brevity
}

// customization/ObjectUiConfigRegistry.java
@Component
public class ObjectUiConfigRegistry {
    private final Map<String, ObjectUiConfig> configs = new ConcurrentHashMap<>();
    public void register(ObjectUiConfig config) { configs.put(config.getObjectName(), config); }
    public ObjectUiConfig getOrDefault(String objectName) {
        return configs.getOrDefault(objectName, ObjectUiConfig.defaultsFor(objectName));
    }
}
```

```java
// A developer customizes Customer's admin UI with a small, declarative @Component — no generated code touched
@Component
public class CustomerUiConfig {
    @PostConstruct
    public void configure(ObjectUiConfigRegistry registry) {
        registry.register(ObjectUiConfig.forObject("Customer")
            .hideField("internalNotes")
            .overrideLabel("email", "Email Address"));
    }
}
```

`ObjectMetadataController` (§28) and the list/form renderers (§30–§32) simply consult `ObjectUiConfigRegistry` after reading `ObjectMetadata`, filtering/relabeling before responding — the generator's output and the customization layer are two separate, composable steps, not one entangled one.

---

# 38. Phase 24 — Custom Field Widgets and Renderers

§32's `switch` on `FieldType` covers the defaults; a specific field sometimes needs its *own* widget regardless of type — a `status` `STRING` field that should render as a colored badge dropdown, not a plain text box. A small `WidgetRegistry`, keyed by `objectName.fieldName`, lets that override slot in without touching §32 at all:

```java
// customization/WidgetRegistry.java
@Component
public class WidgetRegistry {
    private final Map<String, String> customWidgetTemplates = new ConcurrentHashMap<>(); // "Order.status" -> a JS template id

    public void register(String objectName, String fieldName, String widgetTemplateId) {
        customWidgetTemplates.put(objectName + "." + fieldName, widgetTemplateId);
    }
    public Optional<String> lookup(String objectName, String fieldName) {
        return Optional.ofNullable(customWidgetTemplates.get(objectName + "." + fieldName));
    }
}
```

```javascript
// admin.js — renderFieldWidget (§32) checks the registry FIRST, falling back to the type-based default
async function renderFieldWidget(field, existingRecord, objectName) {
    const customWidget = await fetch(`/api/meta/widgets/${objectName}/${field.name}`).then(r => r.json());
    if (customWidget.templateId) return renderCustomWidget(customWidget.templateId, field, existingRecord);
    // ... falls through to §32's type-based switch ...
}
```

This is the **Strategy pattern** applied to rendering, the same design tool the [TinyDB guide's architecture section](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) used for pluggable indexes and concurrency control: a field's widget is chosen by looking up a strategy, with a well-defined default, rather than by editing the one function that renders every field in the system.

---

# 39. Phase 25 — Custom Actions on List and Detail Views

Some operations aren't CRUD at all — "Resend Invoice," "Mark as Shipped." `AdminAction` is a small extension point that adds a button to a list or detail view without MiniAdmin needing to know what the button does:

```java
// customization/AdminAction.java
public interface AdminAction {
    String getLabel();
    boolean appliesTo(String objectName);
    void execute(String objectName, String recordId) throws Exception;
}

@Component
public class ResendInvoiceAction implements AdminAction {
    public String getLabel() { return "Resend Invoice"; }
    public boolean appliesTo(String objectName) { return "Order".equals(objectName); }
    public void execute(String objectName, String recordId) { /* application-specific logic */ }
}
```

```java
// A generic endpoint exposes whatever actions apply to the object being viewed — the UI just renders buttons for them
@GetMapping("/api/meta/objects/{objectName}/actions")
public List<String> availableActions(@PathVariable String objectName) {
    return actions.stream().filter(a -> a.appliesTo(objectName)).map(AdminAction::getLabel).toList();
}
```

Registering a new `AdminAction` as a `@Component` is additive — exactly the Open/Closed shape the [TinyDB guide's SOLID audit](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) argued for: MiniAdmin's generic controllers and UI never change to support a new action, they only ever iterate over whatever's registered.

---

# 40. Phase 26 — Field-Level Visibility and Read-Only Rules

`ObjectUiConfig` (§37) already hides a field entirely; a lighter-weight rule — "show `createdAt`, but never let it be edited" — is a small addition to `FieldMetadata`'s consumers, not to `FieldMetadata` itself:

```java
// ObjectUiConfig — one more collection alongside hiddenFields (§37)
private final Set<String> readOnlyFields = new HashSet<>();
```

```javascript
// admin.js — renderFieldWidget (§32), one line added at the top
if (uiConfig.readOnlyFields.includes(field.name)) {
    return `<label>${field.label}</label><span class="readonly-value">${value}</span>`;
}
```

A read-only field still appears in the form (unlike a hidden one) and still comes back from `GET /api/data/{objectName}/{id}` — it simply never renders as an editable input, and the server-side `save()` path (§23) can additionally reject an incoming change to it, closing the same "don't trust the client" gap §36 already named for required-field validation.

---

# 41. Phase 27 — A Simple Layout Editor (Salesforce Page-Layout-Style)

Salesforce's Page Layout editor lets an admin drag fields into sections and reorder them, entirely from the UI, per object. MiniAdmin's version is a small, honestly-scoped slice of that idea: a persisted field **order** and **section grouping**, stored as more metadata, consumed by the exact same form renderer:

```java
// customization/LayoutMetadata.java
public class LayoutMetadata {
    public record Section(String title, List<String> fieldNames) { }
    private final String objectName;
    private final List<Section> sections;
}
```

```java
// customization/LayoutController.java
@PutMapping("/api/admin/objects/{objectName}/layout")
public ResponseEntity<Void> saveLayout(@PathVariable String objectName, @RequestBody LayoutMetadata layout) {
    layoutRegistry.register(layout); // a ConcurrentHashMap-backed registry, same shape as every other registry in this guide
    return ResponseEntity.noContent().build();
}
```

```javascript
// admin.js — renderForm (§31) groups fields into sections when a layout exists, falls back to a flat list otherwise
async function renderForm(objectName, existingRecord) {
    const layout = await fetch(`/api/meta/objects/${objectName}/layout`).then(r => r.json()).catch(() => null);
    const sections = layout ? layout.sections : [{ title: null, fieldNames: metadata.fields.map(f => f.name) }];
    // ... render one <fieldset> per section, using §32's renderFieldWidget for each field inside it ...
}
```

This is deliberately the smallest version of the feature that still demonstrates the pattern — a drag-and-drop editor is a UI-effort problem, not a design problem, once the underlying model ("an object has an ordered list of named sections, each with an ordered list of field names, stored as data") already exists.

---

# 42. Full End-to-End Example: Defining Customer and Order (Class-Based)

Starting from §12's `Customer`/`Order` classes, with no further code:

1. On startup, `ClassModelRegistrar` (§14) scans, extracts, and registers both objects — `MetadataRegistry` now knows about `Customer` and `Order` before the first HTTP request is served.
2. Opening `/admin/admin.html#/Customer` renders a list view (§30) with columns `Full Name`, `Email` — pulled from `GET /api/meta/objects/Customer` and `GET /api/data/Customer`.
3. Clicking "New Customer" renders a form (§31–§32) with a text input for `name` (marked required, per `@AdminField(required = true)`) and one for `email`.
4. Saving posts to `POST /api/data/Customer`, which validates (§25), routes to `PhysicalTableStorage` (§17, since `origin == "CLASS"`), and inserts a real row into a real `Customer` table.
5. Opening `/admin/admin.html#/Order/new` renders `customer` as a searchable dropdown (§33), populated from `GET /api/lookup/Customer`, showing each customer's `name`.
6. Opening that same `Customer` record afterward shows an "Orders (1)" related-list panel (§34), computed by querying `Order` for `customer = thisId` (§27) — a field that was never written to `Customer` at all.

---

# 43. Full End-to-End Example: Defining a Custom Object From the UI

No Java code, no deploy, starting from a running MiniAdmin instance:

1. `POST /api/admin/objects {"objectName": "Product", "label": "Product"}` (§20) — `Product` now exists, backed by `EavStorage` (`origin == "CUSTOM"`).
2. `POST /api/admin/objects/Product/fields {"name": "price", "type": "DOUBLE", "required": true}` and a second call for `{"name": "sku", "type": "STRING"}` (§21) — two fields added, no schema migration, no restart.
3. Reloading `/admin/admin.html` shows "Product" in the nav (§29) — `ObjectMetadataController` (§28) now returns it alongside `Customer` and `Order`, with no code distinguishing it from them.
4. Creating a `Product` record through the auto-generated form (§31) writes two rows into `eav_value` (`field_name = 'price'`, `field_name = 'sku'`) plus one row into `eav_record` (§19) — no `Product` table ever existed.
5. A third `POST /api/admin/objects/Product/fields` call, adding `{"name": "description", "type": "TEXT"}`, takes effect immediately for every *future* `Product` record — existing records simply have no `eav_value` row for `description` until it's set, which `findById` (§19) already handles by returning `null` for a missing field rather than erroring.

---

# 44. Tracing a Create Request End to End

Following one `POST /api/data/Order` submitted from the auto-generated form, through every layer built in this guide:

```text
Browser: submitForm() (§31) serializes the form -> POST /api/data/Order { orderNumber, customer, total }
   |
   v
GenericCrudController.create("Order", fieldValues)                              (§24)
   |
   v
GenericRepository.save("Order", fieldValues)                                    (§23)
   |    -> metadataRegistry.get("Order")                                        (§11)
   |    -> validate(metadata, fieldValues)                                      (§25) — rejects a missing orderNumber
   |    -> strategyResolver.resolve(metadata)                                   (§22) -> PhysicalTableStorage (origin=CLASS)
   v
PhysicalTableStorage.save(metadata, fieldValues)                                (§17)
   |    -> builds "INSERT INTO Order (id, orderNumber, customer, total) VALUES (?, ?, ?, ?)"
   |    -> customer's value is just the id string the dropdown (§33) submitted  (§26)
   v
A real row in a real Order table, via the MiniConnectionPool from the MiniTomcat companion guide
```

The same trace for a `Product` create request is identical through `GenericRepository.save` and then diverges at exactly one line — `strategyResolver.resolve(metadata)` returns `EavStorage` instead — which is the entire, contained cost of supporting two fundamentally different storage models.

---

# 45. Tracing How a Many-to-One Dropdown Gets Populated

```text
Browser: renderForm("Order", null) (§31) -> renderFieldWidget(customerField, null) (§32)
   |    -> field.type === 'MANY_TO_ONE' -> renderManyToOneDropdown(field, null)     (§33)
   v
GET /api/lookup/Customer
   |
   v
LookupController.options("Customer", "")                                        (§33)
   |    -> metadataRegistry.get("Customer")                                     (§11)
   |    -> inferDisplayField(metadata) -> "name" (Customer's first STRING field, or ObjectUiConfig's override, §37)
   |    -> repository.findAll("Customer")                                       (§23) -> PhysicalTableStorage.findAll (§17)
   v
[{ id: "17", label: "Alice" }, { id: "18", label: "Bob" }, ...]
   |
   v
Browser renders <select><option value="17">Alice</option>...</select>
```

Every relationship dropdown in the entire generated UI — regardless of which two objects it connects — runs through exactly this one path, because `LookupController` (like `GenericCrudController`) takes `{objectName}` as a path variable rather than being written per relationship.

---

# 46. Design Patterns Used in MiniAdmin

| Pattern | Applied to | Where |
|---|---|---|
| **Strategy** | Choosing storage backend (`PhysicalTableStorage` vs `EavStorage`), choosing a field's widget (`WidgetRegistry`) | §17/§19/§22, §38 |
| **Facade** | `GenericRepository` hiding `MetadataRegistry` + `ObjectStorageStrategy` from every controller | §23 |
| **Template Method** (implicit) | `GenericCrudController`'s five HTTP methods define a fixed request-handling shape (resolve metadata → validate → delegate to storage) that every object follows identically | §24 |
| **Registry** | `MetadataRegistry`, `ObjectUiConfigRegistry`, `WidgetRegistry`, the layout registry — every extension point in this guide is "look up by key, fall back to a default" | §11, §37, §38, §41 |
| **Composite** | `LayoutMetadata`'s sections, each holding an ordered list of fields, rendered recursively into nested `<fieldset>`s | §41 |
| **Command-ish plugin object** | `AdminAction` — a self-describing unit of behavior (`getLabel`, `appliesTo`, `execute`) registered as a bean and iterated over generically | §39 |

This list intentionally echoes the [TinyDB guide's own design-pattern section](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) — not because every system needs the same patterns, but because "one generic engine, several pluggable strategies, one facade in front of it" is the same underlying shape both guides converge on once "many similar-but-not-identical things" (tables, or objects) needs to be handled without a class-per-thing explosion.

---

# 47. SOLID Principles Applied to MiniAdmin

| Principle | Where MiniAdmin honors it | Where it would bend under real growth |
|---|---|---|
| **S**ingle Responsibility | `MetadataRegistry` only holds metadata; `ObjectStorageStrategy` implementations only persist; `GenericCrudController` only translates HTTP to repository calls | `GenericRepository.save()` both validates *and* persists — a `Validator` extracted alongside `ObjectStorageStrategy` would separate those two jobs as the validation ruleset grows past simple required/type checks |
| **O**pen/Closed | Adding a new `FieldType`, a new `AdminAction`, or a new storage strategy is additive (§9, §38, §39) | `StorageStrategyResolver.resolve()` (§22) branches on a hard-coded `"CLASS"`/`"CUSTOM"` string — a third storage strategy (e.g. a document-store-backed one) would need a third `if`, not a plugin registration |
| **L**iskov Substitution | Both `PhysicalTableStorage` and `EavStorage` honor `ObjectStorageStrategy`'s contract identically from every caller's point of view (§17, §19, §22) | Solid as built — the interface was designed narrowly enough (five methods, all CRUD-shaped) that both implementations can genuinely promise the same behavior |
| **I**nterface Segregation | `ObjectStorageStrategy`, `AdminAction`, and `MiniFilter`-style hooks are all small, single-purpose interfaces | None identified yet — worth re-checking once `ObjectStorageStrategy` grows a `count()`/`aggregate()` method that `EavStorage` can't efficiently support, which would be the same "split the interface" story as [TinyDB's ISP discussion](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) |
| **D**ependency Inversion | `GenericCrudController` depends on `GenericRepository`, which depends on `MetadataRegistry`/`StorageStrategyResolver` — all constructor-injected MiniSpring beans (§14, §23, §24) | Fully honored — this is, not coincidentally, the exact same constructor-injection discipline [MiniSpring's own `@Autowired`](MiniSpring-Step-by-Step-Guide.md) was built to enforce, and MiniAdmin is itself just a set of MiniSpring beans |

---

# 48. Common Mistakes

- **Mistake 1 — Letting `origin` leak past `StorageStrategyResolver`.** The moment a second class (a controller, a UI renderer) starts checking `if (metadata.getOrigin().equals("CLASS"))`, §22's entire containment goal is broken — fix it by adding a method to `ObjectStorageStrategy` instead of branching on origin elsewhere.
- **Mistake 2 — Skipping server-side validation because the client already checks it (§36).** The client-side copy is a UX nicety, never a security boundary — every write must still pass §25's checks, since any HTTP client can bypass the browser entirely.
- **Mistake 3 — Storing a `ONE_TO_MANY` field's "value."** There is no value to store (§10, §19) — a bug that tries to persist `Customer.orders` directly signals a misunderstanding of the relationship direction, not a missing feature.
- **Mistake 4 — Allowing `addField` on a class-based object (§21).** This creates metadata with no backing table column — guard it explicitly, as §21 does, rather than letting `PhysicalTableStorage` fail confusingly later.
- **Mistake 5 — Building `EavStorage.findAll()` as N+1 queries per record.** A naive "one query per record to fetch its fields" implementation is correct but scales badly (§50) — batch the `eav_value` lookup for every record in the result set into one query instead.
- **Mistake 6 — A custom widget (§38) or action (§39) that bypasses `GenericRepository`.** Talking to `ObjectStorageStrategy` directly from a customization reintroduces the per-object special-casing this entire design was built to avoid.
- **Mistake 7 — Treating `ObjectUiConfig`'s absence as an error instead of a default.** §37's registry must return sensible defaults for any object that hasn't been explicitly configured — most objects, especially freshly created Custom Objects, never will be.

---

# 49. Testing Strategy

| Layer | What to test | How |
|---|---|---|
| `ClassModelScanner` (§13) | An `@AdminEntity` class with every field type and both relationship annotations produces the expected `ObjectMetadata` | Unit test with a small fixture class, assert field names/types/relationships |
| `ClassModelRegistrar`'s validation (§15) | A mismatched `mappedBy` throws at startup, not silently | Unit test with a deliberately broken pair of fixture classes |
| `EavStorage` (§19) | Save then `findById` round-trips every field correctly; a missing field returns `null`, not an error; adding a "new" field name after records already exist doesn't break existing reads | Integration test against a real (or TinyDB) database |
| `GenericRepository` (§23, §25) | Validation rejects a missing required field for *any* registered object, not just one hard-coded test entity | Parameterized test run against every `ObjectMetadata` currently registered |
| `GenericCrudController` (§24) | Full CRUD round-trip via HTTP for a class-based object AND a Custom Object, asserting both work through the identical endpoint shape | Integration test hitting `/api/data/{objectName}` for two different `objectName`s |
| `LookupController` (§33) | Options are filtered by `q`, and `label` reflects the correct `displayField` | Unit test with a fixture `Customer` dataset |
| End-to-end UI | Creating a `Custom Object` from the UI, adding a field, and seeing it appear in a new record's form without a page-framework change | A browser-driven test (or the manual walkthrough in §43) |

The `ClassModelRegistrar` validation test and the `EavStorage` round-trip test are the two most likely to catch a real bug early — everything downstream (§23–§45) trusts that metadata is internally consistent and that storage round-trips faithfully.

---

# 50. Performance Considerations at Scale

- **`EavStorage.findAll()` is the first thing that gets slow.** A table scan across `eav_value` filtered by `object_name` via a join to `eav_record`, for an object with thousands of records, does not benefit from the same simple indexing a physical table's real columns would — a composite index on `(object_name, record_id)` in `eav_record` and `(record_id, field_name)` in `eav_value` (already the primary key in §19) is the minimum; a genuinely large Custom Object may need `EavStorage` to fall back to `PhysicalTableStorage`-style materialization once its record count crosses a threshold — noted as a real limitation, not solved here.
- **`LookupController.options()` (§33) loading every record to filter in Java** (as written) is a correctness-first sketch, not a scalable one — a real implementation pushes the `search` filter into the SQL `WHERE` clause so the database, not the JVM, does the filtering, and paginates rather than returning every match.
- **The admin UI's related-list panel (§34) issues one extra request per `ONE_TO_MANY` field on every detail-view load** — acceptable for a handful of relationships per object, worth lazy-loading (only fetch when a user expands the panel) if an object accumulates many reverse relationships.
- **`ClassModelRegistrar`'s classpath scan (§14) and `@PostConstruct` validation (§15) both run once, at startup** — exactly the right place for this cost, mirroring [MiniSpring's own eager-instantiation discipline](MiniSpring-Step-by-Step-Guide.md): pay a fixed cost once at boot rather than a repeated cost per request.

---

# 51. Final Architecture

```text
                     Class-based (@AdminEntity)          UI-based (Custom Object builder)
                              |                                       |
                     ClassModelScanner (§13)                CustomObjectService (§20-§21)
                     ClassModelRegistrar (§14-§15)                    |
                              +-------------------+-------------------+
                                                  v
                                       MetadataRegistry (§11)
                                                  |
                              +-------------------+-------------------+
                              v                                       v
                    StorageStrategyResolver (§22)          ObjectUiConfigRegistry (§37)
                              |                              WidgetRegistry (§38)
                    +---------+---------+                    Layout registry (§41)
                    v                   v
        PhysicalTableStorage   EavStorage
              (§17)              (§19)
                    +---------+---------+
                              v
                    GenericRepository (§23, §25)
                              |
              +---------------+---------------+
              v                               v
    GenericCrudController (§24)     LookupController (§33)
    ObjectMetadataController (§28)
              |
              v
    Auto-generated Admin UI (§29-§36)
    list, form, dropdowns, related lists, custom widgets/actions (§37-§41)
```

---

# 52. Suggested V2 Enhancements

| Enhancement | What it adds | Where it plugs in |
|---|---|---|
| Role-based field/object permissions | "Only admins can see `internalNotes`," per-role read/write rules | Extends `ObjectUiConfig` (§37) and `GenericRepository`'s validate/save path (§23, §25) with a permission check |
| Search and pagination on list views | Usable list views past a few dozen records | `GenericCrudController.list()` (§24) gains query parameters, `ObjectStorageStrategy.findAll` (§17) gains `LIMIT`/`OFFSET` support |
| Audit history per record | "Who changed this field, and when" — a Salesforce-standard feature | A `ChangeEventPublisher`-style hook (see the [TinyDB guide's change-stream design](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>)) attached to every `GenericRepository.save()` call |
| Bulk import/export (CSV) | Loading many records at once, a common Custom Object workflow | A new endpoint that streams rows through the same `validate()`/`save()` path (§23, §25), one row at a time |
| Formula/rollup fields | A field whose value is computed from related records (e.g. `Customer.totalOrderValue`), Salesforce's Roll-Up Summary Fields | A new `FieldType.FORMULA`, computed at read time the same way `ONE_TO_MANY` already is (§27) |
| Workflow automation (simple triggers) | "When an Order's status changes to Shipped, send an email" — Salesforce Flow's minimal ancestor | An `AdminAction`-shaped hook (§39) invoked automatically on save, rather than only from a UI button |
| Many-to-many relationships | A third relationship type beyond one-to-many/many-to-one | A join-table-backed `FieldType.MANY_TO_MANY`, resolved similarly to §27 but via an intermediate table |
| Moving EavStorage-backed objects to physical tables at scale | Escaping §50's EAV performance ceiling once a Custom Object "graduates" to high volume | A migration tool that reads all `eav_value` rows for an object and replays them as `INSERT`s into a newly `CREATE TABLE`-d physical table, then flips `StorageStrategyResolver`'s decision for that object |

---

# 53. Progressive Interview Question Set

**Level 1 — Metadata fundamentals**
1. Why does one `ObjectMetadata`/`FieldMetadata` shape need to be produced by *both* the class-based and UI-based paths, rather than each path having its own model?
2. Walk through what `ClassModelScanner` does with a `@ManyToOne` field, step by step.

**Level 2 — Storage strategy**
3. Why does a UI-defined Custom Object use EAV storage instead of a real table, and what does that trade away?
4. What would break if a Custom Object were allowed to use `PhysicalTableStorage`, given how §21 adds fields to it?

**Level 3 — Generic APIs**
5. Explain how one controller class can correctly serve CRUD for an object it has never seen at compile time.
6. Why does validation (§25) need to run inside `GenericRepository`, not just in the browser (§36)?

**Level 4 — Relationships**
7. Trace exactly what happens, end to end, when a `MANY_TO_ONE` dropdown is rendered for a brand-new form.
8. Why is a `ONE_TO_MANY` field never part of a save request's payload?

**Level 5 — Extensibility**
9. A team wants to add a `FieldType.EMAIL` with built-in format validation and a specialized widget. Name every file this guide built that needs to change, and why nothing else does.
10. How does `ObjectUiConfig` (§37) avoid becoming a second, competing source of truth alongside `ObjectMetadata` (§8)?

**Final challenge:** Design the migration path for turning a heavily-used, EAV-backed Custom Object into a physical-table-backed one without downtime — for a system already serving live CRUD traffic against it through `GenericRepository`. What has to happen in what order, and how does `StorageStrategyResolver` (§22) need to change to support a mid-flight cutover rather than an instantaneous one?

---

# 54. Final Takeaway

Every feature in this guide — the auto-generated list view, the relationship dropdown, the Custom Object builder, the customization hooks — is a consumer or a producer of exactly one shape: `ObjectMetadata`/`FieldMetadata` (§8). Class-based models and UI-defined Custom Objects are two different **producers** of that shape; `PhysicalTableStorage` and `EavStorage` are two different **strategies** for persisting it; `GenericRepository`, `GenericCrudController`, and the admin frontend are **consumers** that were written exactly once and never touched again as new objects, fields, and relationships were added. That's the entire trick behind "define a model, get a UI for free" — not a large amount of generated code, but a small amount of code that reads a description of the model instead of being written against a specific one.

Layered on top of [MiniSpring's `ApplicationContext`](MiniSpring-Step-by-Step-Guide.md) for wiring, [MiniTomcat](<Build Your Own Web Server From Scratch — MiniTomcat Step-by-Step Guide.md>) for the HTTP transport underneath it, and (optionally) [TinyDB](<Build Your Own Database From Scratch — TinyDB Step-by-Step Guide.md>) for the storage underneath *that*, MiniAdmin completes this guide series' arc: a request now travels from a browser, through a hand-built web server, into a hand-built dependency-injection container, through a metadata-driven CRUD engine a developer never had to write per entity, and into a hand-built database — with every layer in between built from first principles, and none of it a black box.

