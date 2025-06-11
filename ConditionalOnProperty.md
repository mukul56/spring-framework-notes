The annotation `@ConditionalOnProperty` in Spring Boot is **used to conditionally enable or disable a Spring bean or configuration class based on the presence and value of a property in the `application.properties` or `application.yml` file.**

---

### 🔹 1. **Why is `@ConditionalOnProperty` required?**

It is **required** to control bean registration **based on external configuration**, allowing you to:

* **Toggle features** without code changes
* **Enable or disable beans** based on flags in config
* Support **multiple environments** (dev, test, prod) with different behaviors
* Write **flexible and modular configurations**

---

### 🔹 2. **Basic Definition**

```java
@ConditionalOnProperty(
    name = "feature.enabled",     // property key
    havingValue = "true",         // property value to match
    matchIfMissing = false        // default behavior if property is missing
)
```

➡️ This means the bean is **only created if `feature.enabled=true`** in the config.

---

### 🔹 3. **Use Case Example**

#### application.properties

```properties
feature.enabled=true
```

#### Conditional Configuration

```java
@Configuration
public class MyFeatureConfig {

    @Bean
    @ConditionalOnProperty(name = "feature.enabled", havingValue = "true")
    public MyService myService() {
        return new MyService();
    }
}
```

➡️ `MyService` will only be registered as a bean if `feature.enabled=true`.

---

### 🔹 4. **Parameters Explained**

| Parameter        | Purpose                                                      |
| ---------------- | ------------------------------------------------------------ |
| `name`           | Property key to check (required)                             |
| `havingValue`    | Expected value for bean to be loaded                         |
| `prefix`         | Optional prefix for grouped properties                       |
| `matchIfMissing` | Load the bean if the property is missing (defaults to false) |

---

### 🔹 5. **Advanced Usage**

#### With Prefix

```java
@ConditionalOnProperty(prefix = "service.feature", name = "enabled", havingValue = "true")
```

And in `application.properties`:

```properties
service.feature.enabled=true
```

---

### 🔹 6. **Best Practices**

* Use it to conditionally load beans based on **config flags**
* Combine with **profiles**, `@Profile`, or `@Conditional` for more control
* Avoid making too many beans conditional—it can make debugging harder

---

### 🔹 7. **Common Pitfalls**

| Issue                                         | Reason                           |
| --------------------------------------------- | -------------------------------- |
| Bean not loaded                               | Property not set or wrong value  |
| Typo in property name                         | Property name must match exactly |
| `matchIfMissing=true` unintentionally enables | Be explicit when needed          |

---
To **conditionally create only one bean** — either `OnlineOrderService` or `OfflineOrderService` — based on a property like `order.mode=online|offline`, we can use `@ConditionalOnProperty` on both classes.

---

## ✅ Step-by-Step Solution

### 🔹 1. **Define the Common Interface**

```java
public interface OrderService {
    void placeOrder();
}
```

---

### 🔹 2. **Create `OnlineOrderService` Implementation**

```java
@Service
@ConditionalOnProperty(name = "order.mode", havingValue = "online")
public class OnlineOrderService implements OrderService {
    @Override
    public void placeOrder() {
        System.out.println("Placing online order");
    }
}
```

---

### 🔹 3. **Create `OfflineOrderService` Implementation**

```java
@Service
@ConditionalOnProperty(name = "order.mode", havingValue = "offline")
public class OfflineOrderService implements OrderService {
    @Override
    public void placeOrder() {
        System.out.println("Placing offline order");
    }
}
```

---

### 🔹 4. **Add Property in `application.properties` or `application.yml`**

```properties
order.mode=online   # or offline
```

---

### 🔹 5. **Inject the Interface (Only One Bean Will Be Present)**

```java
@RestController
public class OrderController {

    private final OrderService orderService;

    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @GetMapping("/order")
    public String placeOrder() {
        orderService.placeOrder();
        return "Order placed!";
    }
}
```

➡️ Spring will only inject the available implementation. If `order.mode=online`, then only `OnlineOrderService` is created, and vice versa.

---

## ✅ What Happens Behind the Scenes?

* `@ConditionalOnProperty` uses `OrderCondition` internally.
* It checks if the property exists and matches the given `havingValue`.
* If match found, the bean is created; otherwise, it's ignored.

---

## 🧠 Advanced: Handle Missing Property Gracefully

You can allow a **default fallback** using `matchIfMissing`:

```java
@ConditionalOnProperty(name = "order.mode", havingValue = "offline", matchIfMissing = true)
```

This means if `order.mode` is **missing**, use `OfflineOrderService` by default.

---

## ❗️Common Pitfall

If both beans are registered due to misconfiguration (e.g., typo in property name), you’ll get:

```
NoUniqueBeanDefinitionException: No qualifying bean of type 'OrderService' available: expected single matching bean but found 2
```

So validate config names carefully.

---
