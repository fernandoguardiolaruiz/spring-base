# Spring Base Framework

[![Java](https://img.shields.io/badge/Java-17+-orange.svg)](https://openjdk.java.net/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.x-green.svg)](https://spring.io/projects/spring-boot)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Modules](#modules)
  - [Commons](#commons)
  - [View System](#view-system)
  - [MongoDB Integration](#mongodb-integration)
  - [Redis Integration](#redis-integration)
  - [JPA Integration](#jpa-integration)
  - [Cache Module](#cache-module)
  - [Application Core](#application-core)
  - [API Module](#api-module)
  - [Model Module](#model-module)
- [Key Features](#key-features)
- [Getting Started](#getting-started)
- [Usage Examples](#usage-examples)
- [Design Patterns](#design-patterns)

---

## 🎯 Overview

**Spring Base** is a modular framework designed to accelerate Spring Boot application development by providing reusable, production-ready components. It abstracts common patterns and provides powerful utilities for:

- **Dynamic querying** (MongoDB, JPA/Hibernate)
- **View transformation** (Model-to-DTO mapping with security)
- **Redis operations** (transparent data structure management)
- **Error recovery** (automatic retry mechanisms)
- **JMS/RabbitMQ integration** (message-driven architectures)
- **Request tracking** (distributed tracing)
- **Cache management** (Redis-based caching)

The framework follows **modular architecture** principles, with each module separated into `api` (contracts) and `core` (implementations) to respect **Dependency Inversion Principle**.

---

## 🏗️ Architecture

```
spring-base/
├── parent/
│   ├── api/                    # Core API contracts
│   ├── commons/                # Shared utilities
│   ├── view/                   # View transformation system
│   │   ├── api/               # View annotations & interfaces
│   │   └── core/              # View resolution engine
│   ├── mongo/                  # MongoDB dynamic search
│   │   ├── api/               # Search annotations
│   │   ├── core/              # Synchronous implementation
│   │   ├── core-commons/      # Shared MongoDB utilities
│   │   └── core-react/        # Reactive implementation
│   ├── redis/                  # Redis abstraction layer
│   │   ├── api/               # Redis contracts
│   │   ├── core/              # Synchronous implementation
│   │   └── core-react/        # Reactive implementation
│   ├── jpa/                    # JPA/Hibernate dynamic search
│   │   ├── api/               # Search annotations
│   │   └── core/              # HQL query builder
│   ├── cache/                  # Cache management
│   ├── app/                    # Application core utilities
│   └── model/                  # Domain model utilities
```

**Design Philosophy:**
- **Separation of Concerns**: API contracts separated from implementations
- **Dependency Inversion**: Consumers depend on abstractions (`api`), not concrete implementations (`core`)
- **Single Responsibility**: Each module has a focused, well-defined purpose
- **Plugin Architecture**: Modules are independent and can be used individually

---

## 📦 Modules

### 🔧 Commons

**Location:** `parent/commons/`

Shared utilities used across all modules.

#### Key Components:

| Class | Description |
|-------|-------------|
| `ReflectionUtils` | Advanced reflection operations (field introspection, method resolution) |
| `TestUtils` | JSON file loading for integration tests |
| `JsonConverter` | Jackson-based serialization with MixIn support |
| `ExceptionUtils` | Exception handling utilities |
| `Pagination` | Pagination data structures |

#### Exception Hierarchy:

```java
ServiceException (parent)
├── BadRequestException (400 errors)
└── NotFoundException (404 errors)
```

#### Controllers:

- **`CommonsController`**: Base controller with common response patterns
- **`ErrorMessageFactory`**: Standardized error response builder

---

### 🎨 View System

**Location:** `parent/view/`

Annotation-driven **model-to-DTO transformation** with security, filtering, and lazy-loading support.

#### Core Annotations:

##### `@ViewProperty`
Maps entity fields to view properties with filtering capabilities.

```java
@ViewProperty(
    value = "customFieldName",           // Custom name in view
    filter = MyFilter.class,              // Runtime filter predicate
    filterContext = MyContextFilter.class, // Reactive context-based filter
    excludeProperties = {"password"},     // Exclude nested properties
    includeProperties = {"id", "name"}    // Include only specific nested properties
)
private User user;
```

##### `@ViewType`
Defines view classes for polymorphic entities.

```java
@ViewType(types = {
    @ViewFromClass(clazz = Admin.class, view = AdminView.class),
    @ViewFromClass(clazz = Customer.class, view = CustomerView.class)
})
```

##### `@PropertyRolesAllowed`
Role-based field access control.

```java
@PropertyRolesAllowed(
    value = {"ADMIN", "MANAGER"},
    authorizer = CustomAuthorizer.class
)
private BigDecimal salary;
```

##### `@FillerProperty` (View Enrichment)
Used by `FillerResolver` to populate fields **after** the initial view transformation. This effectively decouples the View transformation from external data fetching, enabling efficient **Cross-Microservice Composition**.

**Use Cases:**
- Hydrating a View with data from another microservice (e.g., populating `UserView` in `OrderView`).
- Bulk loading data to avoid N+1 queries.
- Merging data from different data sources (e.g., SQL + Mongo).

```java
// 1. View Definition
public class OrderView implements View {
    private String userId; // Stored as String, but IDs must be Integer for FillerResolver
    
    // Configures the field to be filled by the "user-service-fetcher" function
    // using "userId" as the key.
    @FillerProperty(fillerFunctionName = "user-service-fetcher", supplier = "userId")
    private UserView userDetails;
    
    public String getUserId() { return this.userId; }
}

// 2. Define the Filler Function (e.g., in a @Configuration class)
public FillerFunction<OrderView, UserView> createUserFillerFunction(UserClient userClient) {
    final FillerFunction<OrderView, UserView> filler = new FillerFunction<>();
    filler.setFunctionName("user-service-fetcher");
    
    // A. Define how to extract the ID from the source (OrderView)
    // Note: SupplierId expects Function<View, Integer>
    filler.setSuppliersId(List.of(
        new SupplierId<>(view -> Integer.parseInt(view.getUserId()), "userId")
    ));
    
    // B. Define how to match the result back (extract ID from UserView)
    filler.setResponsesId(UserView::getId);
    
    // C. Define the Bulk Fetch Logic (Batch API Call)
    // Input: Collection<Integer> ids -> Output: Collection<UserView>
    filler.setFillerFunction(ids -> userClient.getUsersByIds(ids));
    
    return filler;
}

// 3. Usage in Business Logic
List<OrderView> orders = viewMapper.map(orderEntities, OrderView.class);

// Create or inject the list of functions
List<FillerFunction<?, ?>> fillers = List.of(this.createUserFillerFunction(userClient));

// Bulk fetch user details for ALL orders in one go
FillerResolver.resolve(orders, fillers);
```

**Key Components:**
- **`FillerResolver`**: The engine that coordinates the enrichment.
- **`FillerFunction`**: Defines the bulk fetching logic (IDs -> Views).
- **`FillerProperty`**: Links the View field to a specific `FillerFunction`.

##### `@DataAdapter`
Custom data transformation during view resolution.

```java
@DataAdapter(adaptor = DateToStringAdapter.class)
private LocalDateTime createdAt;
```

#### Key Features:

✅ **Lazy-loading aware**: Handles Hibernate proxies without `LazyInitializationException`  
✅ **Polymorphism support**: Resolves views based on entity runtime type  
✅ **Nested object mapping**: Recursive transformation with circular reference detection  
✅ **Collection handling**: Transforms `List`, `Set`, `Map` structures  
✅ **Security**: Role-based field visibility  
✅ **Context-aware filtering**: Reactive context propagation  

#### Usage Example:

```java
// Entity
@Entity
public class Order {
    private Long id;
    private User customer;
    private List<OrderItem> items;
    private BigDecimal totalAmount;
}

// View
public class OrderView implements View {
    @ViewProperty
    private Long id;
    
    @ViewProperty(includeProperties = {"id", "email"})
    private UserView customer;
    
    @ViewProperty
    private List<OrderItemView> items;
    
    @ViewProperty
    @PropertyRolesAllowed({"ADMIN", "FINANCE"})
    private BigDecimal totalAmount;
}

// Usage
OrderView view = ViewResolver.resolveEntity(OrderView.class, order);
```

---

### 🍃 MongoDB Integration

**Location:** `parent/mongo/`

**Annotation-driven dynamic search** for MongoDB with aggregation pipeline support.

#### Core Annotations:

##### `@MongoSearchProperty`
Maps request fields to MongoDB criteria.

```java
@MongoSearchProperty(
    value = "email",                      // MongoDB field path
    operationType = OperationType.LIKE,   // Search operation
    conditionType = ConditionType.AND     // Logical operator
)
private String userEmail;
```

**Supported Operations:**
- `IS`: Exact match (`field = value`)
- `LIKE`: Regex match (`field ~= /value/i`)
- `IN`: Inclusion (`field in [values]`)
- `GTE`: Greater than or equal (`field >= value`)
- `LTE`: Less than or equal (`field <= value`)
- `EXISTS`: Field existence check

##### `@MongoSearchClass`
Nested object search.

```java
@MongoSearchClass(value = "address")
private AddressSearch address;
```

##### `@MongoSearchProperties`
Multiple field search (OR condition).

```java
@MongoSearchProperties(
    value = {"firstName", "lastName", "email"},
    operationType = OperationType.LIKE
)
private String searchTerm;
```

##### `@MongoConcatProperty`
Search on concatenated fields.

```java
@MongoConcatProperty(
    properties = {"firstName", "lastName"},
    separator = " ",
    operationType = OperationType.LIKE
)
private String fullName;
```

##### `@MongoAddFieldProperty`
Add computed fields using aggregation expressions.

```java
@MongoAddFieldProperty(
    name = "discountedPrice",
    function = "$multiply",
    properties = {"price", "discountRate"}
)
```

#### Repository Integration:

```java
public interface ProductRepository extends 
    MongoRepository<Product, String>, 
    MongoSearchRepository<Product, ProductSearch> { }
```

**Custom Methods:**
```java
List<Product> search(ProductSearch searchCriteria);
Page<Product> search(ProductSearch searchCriteria, Pageable pageable);
long count(ProductSearch searchCriteria);
```

**Reactive Support:**
```java
public interface ReactiveProductRepository extends 
    ReactiveMongoRepository<Product, String>,
    MongoSearchReactiveRepository<Product, ProductSearch> { }

Flux<Product> search(ProductSearch searchCriteria);
Mono<Long> count(ProductSearch searchCriteria);
```

---

### 🔴 Redis Integration

**Location:** `parent/redis/`

**Type-safe, high-level abstraction** over Redis data structures with transparent serialization.

#### Core Concepts:

##### 1. **Simple Key-Value Operations**

```java
@Service
public class MyService {
    
    @Autowired
    private RedisService redisService;

    @Autowired
    private RedisConnectionManager redisConnectionManager;
    
    public void processUser(User userObject) {
        final RedisCommands redisCommands = this.redisConnectionManager.getConnection();
        try {
            // Store object
            this.redisService.set(redisCommands, "user:123", userObject, 3600);
            
            // Retrieve object
            final User user = this.redisService.get(redisCommands, "user:123", User.class);
            
            // Delete
            this.redisService.delete(redisCommands, "user:123");
        } finally {
            this.redisConnectionManager.closeConnection(redisCommands);
        }
    }
}
```

##### 2. **Hash Operations**

```java
RedisCommands redisCommands = redisConnectionManager.getConnection();

try {
    // Store hash
    Map<String, String> userData = Map.of(
        "name", "John",
        "email", "john@example.com"
    );
    redisService.setHash(redisCommands, "user:123", userData);

    // Get single field
    String email = redisService.getHashField(this.redisCommands, "user:123", "email");

    // Get all fields
    Map<String, String> allData = redisService.getHash(this.redisCommands, "user:123");
} finally {
    redisConnectionManager.closeConnection(redisCommands);
}
```



#### Distributed Locking:

```java
RedisLock lock = redisLockFactory.create("resource:123", 30000, 10000);

try {
    if (lock.acquire()) {
        // Critical section
        performAtomicOperation();
    }
} finally {
    lock.release();
}
```

#### Functional API (Bulk Operations Strategy)

The framework uses a functional strategy pattern for handling complex bulk operations (get, set, update) efficiently on **Key-Value** pairs.

**1. Implement `GetFunctions` for Read Operations:**

```java
public class OrderGetFunctions implements GetFunctions<OrderKey, OrderValue, OrderKeyValue> {
    
    private final JsonConverter<OrderValue> converter = new JsonConverter<>(OrderValue.class);

    @Override
    public ToValueFunction<OrderKey, OrderValue> toValueFunction() {
        return (key, rawJson) -> rawJson != null ? this.converter.toObject(rawJson) : null;
    }

    @Override
    public ToKeyValueFunction<OrderKey, OrderValue, OrderKeyValue> toKeyValueFunction() {
        return (key, value) -> new OrderKeyValue(key, value);
    }
}
```

**2. Implement `SetFunctions` for Write Operations:**

```java
public class OrderSetFunctions implements SetFunctions<OrderKey, OrderValue, OrderKeyValue> {

    private final Collection<OrderUpdateData> updates;

    public OrderSetFunctions(Collection<OrderUpdateData> updates) {
        this.updates = updates;
    }

    @Override
    public UpdateValueFunction<OrderKey, OrderValue> updateValueFunction() {
        return (key, currentOrder) -> {
            // Business logic to update the order
            findUpdate(key).ifPresent(update -> currentOrder.setStatus(update.getStatus()));
            return currentOrder;
        };
    }

    @Override
    public NewValueFunction<OrderKey, OrderValue> newValueFunction() {
        return (key) -> {
             // Logic to create a new order if not found
             return new OrderValue(findUpdate(key).orElse(null));
        };
    }
    
    // ... implement other methods
}
```

**3. Usage:**

```java
@Autowired
private RedisService<OrderKey, OrderValue, OrderKeyValue> redisService;

public void processOrders(List<String> orderIds) {
    final List<OrderKey> keys = orderIds.stream().map(OrderKey::new).toList();
    
    // Bulk read
    final Collection<OrderKeyValue> orders = this.redisService.getKeyValues(keys, new OrderGetFunctions());
    
    // Bulk update/insert
    this.redisService.setKeyValues(keys, new OrderSetFunctions(incomingUpdates));
}
```

#### 🔴 Redis Units (Advanced Hash Management)

**Location:** `parent/redis/`

`RedisUnitService` simplifies managing hierarchical data where a **Container** (e.g., Order) holds a collection of **Units** (e.g., OrderItems) stored as a Redis **Hash**.
- **Redis Key**: The Container ID (`OrderKey`).
- **Hash Field**: The Unit Key (`OrderItemKey`).
- **Hash Value**: The Unit Data (`OrderItem`).

**1. Define the Structure:**

```java
// 1. Container Key (The Redis Key)
public class OrderKey implements RedisKey {
    private String id;
    @Override public String getKey() { return "order:" + this.id; }
}

// 2. Unit Key (The Hash Field)
public class OrderItemKey implements RedisUnitKey {
    private String sku;
    @Override public String getKey() { return this.sku; } // Unique within the container
}

// 3. Unit (The data stored in Hash Value)
public class OrderItem implements RedisUnit<OrderItemKey> {
    private OrderItemKey key;
    private int quantity;
    @Override public OrderItemKey getKey() { return this.key; }
}

// 4. Container (The Object representing the whole structure)
public class Order implements RedisUnitContainer<OrderKey, OrderItem> {
    private OrderKey key;
    private Collection<OrderItem> items;
    // ... getters/setters/constructor
    @Override public OrderKey getRedisKey() { return this.key; }
    @Override public Collection<OrderItem> getRedisUnits() { return this.items; }
}
```

**2. Usage (`RedisUnitService`):**

```java
@Autowired
private RedisUnitService<OrderKey, OrderItemKey, OrderItem, Order> redisUnitService;

public void manageOrderItems() {
    final List<OrderKey> keys = List.of(new OrderKey("ORD-1"));

    // READ: Get Orders with their Items
    // Requires: Mapping Function (Hash Map -> Units) + Creation Function (Units -> Container)
    final Collection<Order> orders = this.redisUnitService.getRedisUnitContainers(
        keys, 
        RedisFunctions.mappingUnitFunction(), 
        RedisFunctions.redisUnitContainerCreationFunction(), 
        item -> item.getQuantity() > 0 // Optional: Filter units
    );

    // WRITE: Save Orders and their Items
    // Requires: Mapping Attribute Function (Units -> Hash Map) + Merger Function
    this.redisUnitService.setRedisUnitContainers(
        orders,
        RedisFunctions.mappingUnitFunction(),       // Map -> Unit
        RedisFunctions.mappingAttributeFunction(),  // Unit -> Map
        RedisFunctions.mergerUnitFunction(),        // Merge Strategy
        RedisFunctions.redisUnitContainerCreationFunction()
    );
}
```

**3. Required Functions Helper:**

You need to implement the translation functions. Typically encapsulated in a utility class:

```java
public class RedisFunctions {
    // Defines how to convert Redis Hash Data (Map<String, String>) to Unit Objects
    public static MappingUnitFunction<OrderKey, OrderItemKey, OrderItem> mappingUnitFunction() {
        return (redisKeyMapValues) -> {
            // logic to convert map values to Set<OrderItem>
            return redisKeyMapValues.getValues().values().stream()
                .map(json -> jsonConverter.toObject(json))
                .collect(Collectors.toSet());
        };
    }
    
    // Defines how to convert Unit Objects to Redis Hash Data
    public static MappingAttributeFunction<OrderKey, OrderItemKey, OrderItem> mappingAttributeFunction() {
        return (key, units) -> {
            final Map<String, String> map = units.stream()
                .collect(Collectors.toMap(u -> u.getKey().getKey(), u -> jsonConverter.toString(u)));
            return new RedisKeyMapValues<>(key, map);
        };
    }
    
    // ... implement mergerUnitFunction and redisUnitContainerCreationFunction
}
```

---

### 🗄️ JPA Integration

**Location:** `parent/jpa/`

**Annotation-driven dynamic HQL query generation** for JPA/Hibernate.

#### Core Annotations:

##### `@SearchProperty`
Maps request fields to JPA entity properties.

```java
@SearchProperty(
    value = "email",                        // Entity property path
    isLike = true,                          // Use LIKE operator
    conditionType = ConditionType.AND,      // Logical operator
    compareOperator = CompareOperator.EQUAL, // Comparison operator
    join = @Join(path = "user", alias = "u") // Join configuration
)
private String userEmail;
```

##### `@SearchProperties`
Search across multiple properties (OR condition).

```java
@SearchProperties(
    value = {"firstName", "lastName", "email"},
    isLike = true,
    conditionType = ConditionType.OR
)
private String searchTerm;
```

##### `@SearchPropertyExists`
Check for null/non-null values.

```java
@SearchPropertyExists(
    value = "deletedAt",
    exists = false  // WHERE deletedAt IS NULL
)
private Boolean activeOnly;
```

##### `@SubSearch`
Nested search criteria.

```java
@SubSearch(conditionType = ConditionType.AND)
private AddressSearch address;
```

##### `@Join`
Configure entity joins.

```java
@Join(
    path = "orders",           // Navigation property
    alias = "o",               // Join alias
    type = JoinType.LEFT_JOIN  // Join type
)
```

#### Repository Integration:

```java
public interface OrderRepository extends 
    JpaRepository<Order, Long>,
    SearchRepository<Order, OrderSearch> { }
```

**Custom Methods:**
```java
List<Order> search(OrderSearch searchCriteria);
Page<Order> search(OrderSearch searchCriteria, Pageable pageable);
long count(OrderSearch searchCriteria);
```

---

### 💾 Cache Module

**Location:** `parent/cache/`

**Redis-based caching** with Spring Cache abstraction and management endpoints.

#### Configuration:

```java
@Service
public class UserService {
    
    @Cacheable(value = "users", key = "#userId")
    @RedisCacheConfiguration(ttl = 3600) // 1 hour TTL
    public User getUserById(String userId) {
        return userRepository.findById(userId);
    }
    
    @CacheEvict(value = "users", key = "#user.id")
    public void updateUser(User user) {
        userRepository.save(user);
    }
}
```

#### Management API:

```java
@RestController
@RequestMapping("/admin/cache")
public class CacheController {
    
    @Autowired
    private CacheService cacheService;
    
    @DeleteMapping("/{cacheName}")
    public boolean clearCache(@PathVariable String cacheName) {
        return this.cacheService.clearCache(cacheName);
    }
    
    @GetMapping("/statistics")
    public Collection<CacheInformationStatistics> getStats() {
        return this.cacheService.getCacheInformationStadistics(filter);
    }
}
```

---

### ⚙️ Application Core

**Location:** `parent/app/`

**Production-ready utilities** for Spring Boot applications.

#### 1. Error Recovery System

**Recovery mechanism** for failed operations. Errors are registered in an external service, and a background task polls for pending errors to trigger the recovery logic.

```java
@Service
@ErrorRecovery(maxRetries = "3", fixedDelayMilis = "5000")
public class OrderService implements Recoverable {
    
    @Autowired
    private ErrorRegister errorRegister;
    
    public void processOrder(Order order) {
        try {
            // Business logic that might fail
            paymentGateway.charge(order);
        } catch (final Exception ex) {
            // Register error for later recovery
            this.errorRegister.registryErrorAsync(
                ex,
                "processOrder", 
                this.getClass().getName(), // Class name matches the bean for recovery
                Map.of("orderId", order.getId()), 
                true
            );
        }
    }

    @Override
    public void recover(ErrorView errorView) {
        // Recovery logic executed by the background task
        final String orderId = errorView.getData().get("orderId");
        final Order order = orderRepository.findById(orderId);
        this.processOrder(order); 
    }
}
```

#### 2. Request Tracking

**Distributed tracing** with request ID propagation.

```java
@Component
public class RequestTrackerFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
        final String requestId = UUID.randomUUID().toString();
        MDC.put("REQUEST_ID", requestId);
        response.addHeader("X-Request-ID", requestId);
        chain.doFilter(request, response);
    }
}
```

#### 3. JMS/RabbitMQ Integration

**Integration with `rabbitmq-java-queue`** for message-driven architecture.

The framework leverages the separate **`rabbitmq-java-queue`** library to provide robust messaging capabilities. Within `spring-base`, this is primarily used for:

1.  **Error Reporting:** Automatically sending exceptions to an error queue via `ErrorRegister`.
2.  **Event Propagation:** Publishing business events using producers like `EventProducerResource`.

**Example 1: Event Producer**

```java
@Component
@JmsProducer
@NotifyErrorHandler
@JmsDestination(name = "event-queue", clazzSuffix = JmsActiveProfileSuffix.class)
public class EventProducerResource extends JmsProducerResource<EventRequest> {
    // Methods to publish events are inherited from JmsProducerResource (e.g., send(event))
}
```

**Example 2: Error Reporting (Internal)**

The `ErrorRegister` component uses an internal producer (`JmsErrorProducer`) to send error details to the configured error queue.

```java
@Component
public class ErrorRegister {

    @Autowired
    private JmsErrorProducer jmsErrorProducer;

    public void registryError(Throwable exception, Map<String, String> data) {
        // Sends error details to the DLQ/Error Queue
        this.jmsErrorProducer.send(ErrorRequest.builder()
            .errorMessage(exception.getMessage())
            .data(data)
            .build());
    }
}
```

#### 4. Exception Handling

**Centralized exception management:**

```java
@RestControllerAdvice
public class ExceptionControllerHandler {
    
    @ExceptionHandler(BadRequestException.class)
    public ResponseEntity<ErrorMessage> handleBadRequest(BadRequestException ex) {
        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(ErrorMessage.builder()
                .code(ex.getCode())
                .message(ex.getMessage())
                .level(ErrorLevel.ERROR)
                .build());
    }
}
```

#### 5. Mapping Utilities

**Request-to-Model mapping with view-model pattern:**

```java
@Component
public class ViewModelMapper<R, VM, M> {
    
    public M map(R request, Class<VM> viewModelClass, M model) {
        // Maps request -> view model -> model
        // Supports nested objects and collections
    }
}
```

#### 6. Refreshable Singleton Scope

**Custom Spring scope** for beans that can be refreshed on demand.

```java
@Service
@Scope("refreshable-singleton")
public class ConfigService {
    
    @InitMethod
    public void initialize() {
        // Called on bean creation and refresh
        loadConfiguration();
    }
}
```

#### 7. Management Controllers

Operational endpoints for controlling application state at runtime.

| Controller | Base Path | Description |
|------------|-----------|-------------|
| **JmsController** | `/jms` | Manage JMS consumers (start, stop, list). Useful for operating on specific consumers without restart. |
| **BeanController** | `/bean` | Refresh beans annotated with `@Scope("refreshable-singleton")` via `/bean/refresh?name={beanName}`. |
| **ErrorRecoveryController** | `/errorRecovery` | Manually trigger the error recovery process based on provided search criteria. |
| **CommonsController** | N/A | Base controller providing standardized `@ExceptionHandler` responses for `NotFoundException`, `BadRequest`, etc. |

---

### 🌐 API Module

**Location:** `parent/api/`

**Core contracts** for service APIs.

#### Base Request/Response:

```java
public abstract class BaseRequest {
    private String requestId;
    private LocalDateTime timestamp;
}

public abstract class BaseResponse {
    private String requestId;
    private LocalDateTime timestamp;
    private ErrorMessage error;
}

public class SingleBaseResponse<T> extends BaseResponse {
    private T data;
}

public class CollectionBaseResponse<T> extends BaseResponse {
    private Collection<T> data;
    private Pagination pagination;
}
```

---

### 📊 Model Module

**Location:** `parent/model/`

**Domain model utilities** and audit support.

#### Audit Interface:

```java
public interface Audit {
    AuditInfo getAuditInfo();
    void setAuditInfo(AuditInfo auditInfo);
}

@Embeddable
public class AuditInfo {
    private LocalDateTime createdAt;
    private String createdBy;
    private LocalDateTime updatedAt;
    private String updatedBy;
}
```

---

## 🎯 Key Features

### ✅ **Dynamic Querying**
- **MongoDB**: Annotation-driven aggregation pipelines with support for **Geospatial search**, computed fields (`$multiply`, etc.), and field concatenation.
- **JPA/Hibernate**: Dynamic HQL generation with automatic Join handling.
- **Advanced Filtering**: Support for `IS`, `LIKE`, `IN`, `GTE`, `LTE`, `EXISTS` and nested criteria.

### ✅ **View Transformation & Composition**
- **Cross-Microservice Composition**: Hydrate views with data from external services using `FillerResolver`.
- **Annotation-based**: Minimal boilerplate for Model-to-DTO mapping.
- **Security-aware**: Role-based field filtering (`@PropertyRolesAllowed`).
- **Lazy-loading safe**: Intelligent Hibernate proxy handling to avoid N+1 and initialization exceptions.
- **Polymorphic**: Runtime type resolution for inheritance hierarchies.

### ✅ **Redis Ecosystem**
- **Hierarchical Data**: Manage complex "Container-Unit" structures (e.g., Orders -> Items) with `RedisUnitService`.
- **Functional Strategies**: Type-safe `GetFunctions`/`SetFunctions` for efficient bulk operations.
- **Distributed Locking**: Robust mutex implementation for cluster-safe operations.
- **Reactive Support**: Fully non-blocking implementations for high-throughput scenarios.

### ✅ **Resilience & Recovery**
- **Error Recovery System**: Interface-based (`Recoverable`) scheduled recovery for failed operations.
- **Message-Driven Architecture**: Integration with `rabbitmq-java-queue` for robust Event-Driven patterns.
- **Automatic Error Reporting**: Seamless integration with error queues and DLQ strategies.

### ✅ **Application Operations**
- **Refreshable Scope**: Custom `@Scope("refreshable-singleton")` for runtime bean reloading without restart.
- **Management Endpoints**: Built-in controllers for managing JMS consumers, clearing caches, and triggering manual recovery.
- **Cache Management**: Method-level TTL configuration (`@RedisCacheConfiguration`) and runtime statistics.
- **Request Tracing**: Distributed tracing with Request ID propagation across HTTP and JMS borders.

### ✅ **Modular Architecture**
- **Dependency Inversion**: Strict separation between `api` (contracts) and `core` (implementations).
- **Plug-and-Play**: Modules (Mongo, Redis, View, etc.) can be used independently.

---

## 🚀 Getting Started

### Prerequisites

- **Java 17+**
- **Maven 3.8+**
- **Spring Boot 3.x**

### Installation

Add the required modules to your `pom.xml`:

```xml
<dependencies>
    <!-- Commons -->
    <dependency>
        <groupId>com.core</groupId>
        <artifactId>commons</artifactId>
        <version>1.0.0</version>
    </dependency>
    
    <!-- View System -->
    <dependency>
        <groupId>com.core</groupId>
        <artifactId>view-api</artifactId>
        <version>1.0.0</version>
    </dependency>
    <dependency>
        <groupId>com.core</groupId>
        <artifactId>view-core</artifactId>
        <version>1.0.0</version>
    </dependency>
    
    <!-- MongoDB Integration -->
    <dependency>
        <groupId>com.core</groupId>
        <artifactId>mongo-api</artifactId>
        <version>1.0.0</version>
    </dependency>
    <dependency>
        <groupId>com.core</groupId>
        <artifactId>mongo-core</artifactId>
        <version>1.0.0</version>
    </dependency>
</dependencies>
```

### Build

```bash
mvn clean install
```

---

## 📖 Usage Examples

### Example 1: MongoDB Dynamic Search

```java
// 1. Define search request
@Data
public class ProductSearch implements MongoSortedSearch {
    
    @MongoSearchProperty(value = "name", operationType = OperationType.LIKE)
    private String name;
    
    @MongoSearchProperty(value = "category", operationType = OperationType.IS)
    private String category;
    
    @MongoSearchProperty(value = "price", operationType = OperationType.GTE)
    private BigDecimal minPrice;
    
    private SortCriteria sortCriteria;
}

// 2. Create repository
public interface ProductRepository extends 
    MongoRepository<Product, String>,
    MongoSearchRepository<Product, ProductSearch> { }

// 3. Use in service
@Service
public class ProductService {
    
    @Autowired
    private ProductRepository productRepository;
    
    public List<Product> searchProducts(ProductSearch search) {
        return this.productRepository.search(search);
    }
}
```

### Example 2: View Transformation with Security

```java
// 1. View with role-based filtering
public class EmployeeView implements View {
    
    @ViewProperty
    private Long id;
    
    @ViewProperty
    private String firstName;
    
    @ViewProperty
    @PropertyRolesAllowed({"ADMIN", "HR"})
    private BigDecimal salary;  // Only visible to ADMIN/HR
    
    @ViewProperty(includeProperties = {"id", "name"})
    private DepartmentView department;
}

// 2. Transform
@Service
public class EmployeeService {
    
    public EmployeeView getEmployeeView(Long id) throws Exception {
        final Employee employee = employeeRepository.findById(id)
            .orElseThrow(() -> new NotFoundException("Employee not found"));
        
        return ViewResolver.resolveEntity(EmployeeView.class, employee);
    }
}
```

### Example 3: JPA Dynamic Search with Joins

```java
// 1. Search request
@Data
@SearchForClass(entityClass = Order.class)
public class OrderSearch implements Search {
    
    @SearchProperty(
        value = "status",
        inclusionOperator = IncusionOperator.IN
    )
    private List<OrderStatus> statuses;
    
    @SearchProperty(
        value = "c.email",
        isLike = true,
        join = @Join(value = "customer c", left = true)
    )
    private String customerEmail;
    
    @SearchProperties(
        value = {"c.firstName", "c.lastName"},
        isLike = true,
        conditionType = ConditionType.OR
    )
    private String customerName;
    
    private OrderBy orderBy;
}

// 2. Repository
public interface OrderRepository extends 
    JpaRepository<Order, Long>,
    SearchRepository<Order, OrderSearch> { }

// 3. Usage
Page<Order> orders = orderRepository.search(search, PageRequest.of(0, 20));
```

### Example 4: Advanced JPA Search (SubSearch & Complex Joins)

Shows complex querying with chained joins, sub-searches (nested logic), and string concatenation.

```java
@Data
@SearchForClass(value = Booking.class, distinct = true)
public class BookingSearch extends Search {

    // Complex search on multiple fields with conditions
    @SearchProperties({
        @SearchProperty(
            join = @Join("c.bookingPaxes p"),
            preCondition = @PreCondition(condition = "p.isLeadPax = true"), // Only search lead pax
            concat = @Concat({"p.name", "p.surname"}),                       // Search 'Name Surname'
            isLike = true
        ),
        @SearchProperty("bookingReference"),
        @SearchProperty(
            join = @Join("c.bookingProductItems bp"), 
            value = "bp.productName", 
            isLike = true
        )
    })
    private String searchText;

    // Chained Joins: Booking -> ProductItems -> ServiceItems
    @SearchProperty(
        join = @Join("c.bookingProductItems bp JOIN bp.bookingServiceItems bi"),
        value = "bi.startDate"
    )
    private Date eventDate;

    // SubSearch: Groups these conditions with custom logic (e.g., OR)
    // AND ( ... main search conditions ... ) AND ( ... companyOrSearch conditions ... )
    @SubSearch
    private BookingCompanyOrSearch companyOrSearch;
}

@Data
public class BookingCompanyOrSearch extends Search {
    
    // This allows finding a booking if ANY of these company IDs match (OR logic)
    
    @SearchProperty(value = "promoterCompanyId", conditionType = ConditionType.OR)
    private Integer promoterCompanyId;

    @SearchProperty(
        join = @Join("c.bookingProductItems bp JOIN bp.bookingServiceItems bi"),
        value = "bi.providerCompanyId", 
        conditionType = ConditionType.OR
    )
    private Integer providerCompanyId;

    @SearchProperty(
        join = @Join("c.bookingProductItems bp"),
        value = "bp.sellerCompanyId", 
        conditionType = ConditionType.OR
    )
    private Integer sellerCompanyId;
}
```

### Example 5: Redis with Distributed Lock

```java
@Service
public class InventoryService {
    
    @Autowired
    private RedisService redisService;
    
    @Autowired
    private RedisLockFactory redisLockFactory;
    
    public void reserveProduct(String productId, int quantity) {
        final RedisLock lock = this.redisLockFactory.create("inventory:" + productId, 30000, 10000);
        
        try {
            if (lock.acquire()) {
                // Get current inventory
                final Inventory inventory = this.redisService.get(
                    commands, 
                    "inventory:" + productId, 
                    Inventory.class
                );
                
                // Update inventory
                inventory.reserve(quantity);
                this.redisService.set(commands, "inventory:" + productId, inventory, 3600);
            }
        } finally {
            lock.release();
        }
    }
}
```

---

## 🎨 Design Patterns

### Patterns Implemented:

| Pattern | Module | Description |
|---------|--------|-------------|
| **Specification** | JPA, MongoDB | Dynamic query construction |
| **Builder** | JPA, MongoDB | Fluent query building |
| **Strategy** | All search modules | Different search implementations |
| **Factory** | Redis, Error Recovery | Object creation abstraction |
| **Proxy** | View System | Hibernate proxy handling |
| **Template Method** | HTTP Client | WebClient abstraction |
| **Chain of Responsibility** | View System | Annotation processing pipeline |
| **Decorator** | Redis | Functional transformation chains |
| **Repository** | JPA, MongoDB | Data access abstraction |
| **Unit of Work** | Redis Units | Aggregate operations |
| **Mutex** | Redis | Distributed locking |

---

## 👤 Author

**Fernando Guardiola Ruiz**  
Software Architect | Backend Specialist

- GitHub: [@fguardiola](https://github.com/fguardiola)
- LinkedIn: [Fernando Guardiola](https://www.linkedin.com/in/fernando-guardiola)

---

**Built with ❤️ for the developer community**
