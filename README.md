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

##### `@FillerProperty`
Post-processing enrichment using Spring beans.

```java
@FillerProperty(value = "priceCalculator")
private BigDecimal calculatedPrice;
```

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
    
    // Store object
    redisService.set(redisCommands, "user:123", userObject, 3600);
    
    // Retrieve object
    User user = this.redisService.get(redisCommands, "user:123", User.class);
    
    // Delete
    redisService.delete(redisCommands, "user:123");
}
```

##### 2. **Hash Operations**

```java
// Store hash
Map<String, String> userData = Map.of(
    "name", "John",
    "email", "john@example.com"
);
redisService.setHash(redisCommands, "user:123", userData);

// Get single field
String email = redisService.getHashField(redisCommands, "user:123", "email");

// Get all fields
Map<String, String> allData = redisService.getHash(redisCommands, "user:123");
```

##### 3. **Redis Units**

**Pattern for managing complex hierarchical data structures.**

```java
// Define key structure
public class OrderKey implements RedisUnitKey {
    private String orderId;
    
    @Override
    public String getKey() {
        return "order:" + this.orderId;
    }
}

// Define unit structure
public class OrderItemUnit implements RedisUnit<OrderItemKey> {
    private OrderItemKey key;
    private String productId;
    private Integer quantity;
    
    @Override
    public OrderItemKey getKey() {
        return this.key;
    }
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

#### Functional API:

```java
// GetFunctions - Read transformations
String value = redisService.execute(
    redisCommands,
    GetFunctions.getAndTransform("user:123", User.class)
        .andThen(user -> user.getEmail())
        .andThen(String::toUpperCase)
);

// SetFunctions - Write transformations
redisService.execute(
    redisCommands,
    SetFunctions.transformAndSet(
        "user:123",
        user -> {
            user.setLastLogin(LocalDateTime.now());
            return user;
        },
        3600
    )
);
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
@RedisCacheConfiguration(
    cacheName = "users",
    ttl = 3600  // 1 hour TTL
)
@Service
public class UserService {
    
    @Cacheable(value = "users", key = "#userId")
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

**Automatic retry mechanism** for failed operations with exponential backoff.

```java
@Service
public class OrderService {
    
    @Recoverable(
        maxRetries = 3,
        backoffMs = 1000,
        jmsQueue = "error.recovery.queue"
    )
    public void processOrder(Order order) {
        // If this fails, it will be retried automatically
        externalPaymentService.charge(order);
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

**Message-driven architecture** with automatic context propagation.

```java
@Component
@JmsListener(value = OrderCreatedConsumer.class)
public class OrderCreatedConsumer implements JmsResourceListener {
    
    @Autowired
    private OrderService orderService;
    
    @Override
    public void onProcessingMessage(Message message) {
        final OrderCreatedEvent event = fromJson(message);
        this.orderService.processNewOrder(event.getOrderId());
    }
    
    @Override
    @NotifyErrorHandler(jmsQueue = "error.queue")
    public void onErrorMessage(Message message, Exception ex) {
        // Automatic error handling and DLQ
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
- **MongoDB**: Annotation-driven aggregation pipelines
- **JPA/Hibernate**: Dynamic HQL generation
- **Type-safe**: Compile-time validation with annotations

### ✅ **View Transformation**
- **Annotation-based**: Minimal boilerplate
- **Security-aware**: Role-based field filtering
- **Lazy-loading safe**: Hibernate proxy handling
- **Polymorphic**: Runtime type resolution

### ✅ **Redis Abstraction**
- **High-level API**: Transparent data structure management
- **Functional style**: Elegant transformation pipelines
- **Distributed locking**: Mutex and lock support
- **Reactive support**: Non-blocking operations

### ✅ **Error Recovery**
- **Automatic retries**: Exponential backoff
- **Error tracking**: Searchable error registry
- **Manual intervention**: Admin recovery endpoints

### ✅ **Request Tracking**
- **Distributed tracing**: Request ID propagation
- **Context management**: Thread-local storage
- **JMS integration**: Context propagation across messages

### ✅ **Modular Architecture**
- **API/Core separation**: Clear contracts
- **Dependency Inversion**: Depend on abstractions
- **Plugin system**: Use modules independently

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
        value = "customer.email",
        isLike = true,
        join = @Join(path = "customer", alias = "c")
    )
    private String customerEmail;
    
    @SearchProperties(
        value = {"customer.firstName", "customer.lastName"},
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

### Example 4: Redis with Distributed Lock

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
