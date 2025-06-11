`@Profile` is a **Spring annotation** used to **control which beans are registered** in the application context depending on the **active profile**. It's a powerful feature for managing different configurations for different environments (like `dev`, `test`, `prod`).

---

## 🧠 1. What is `@Profile`?

### ➤ Definition:

`@Profile` is a Spring annotation that allows you to specify that a **bean should only be created when a specific profile is active**.

### ➤ Declaration:

```java
@Profile("dev")
@Component
public class DevDataSourceConfig implements DataSourceConfig {
    // Dev-specific configuration
}
```

---

## ⚙️ 2. Why Is `@Profile` Needed?

Profiles help you:

| Use Case                      | Why Use `@Profile`                          |
| ----------------------------- | ------------------------------------------- |
| 🔧 Environment-specific Beans | Create different beans for dev/test/prod    |
| 🧪 Testing Configurations     | Load mock beans in test profile             |
| 💼 Deployment Configurations  | Use real services in prod                   |
| 🔄 Switching Behavior Easily  | No code changes needed, just profile change |

---

## ⚙️ 3. How `@Profile` Works

### 🔹 Step-by-step Flow:

1. **Spring starts** and checks the active profile.
2. It evaluates all beans and classes annotated with `@Profile`.
3. Only beans with a matching profile are registered in the context.
4. Others are ignored/skipped.

---

## 🔍 4. Setting Active Profile

### ✅ Using `application.properties`:

```properties
spring.profiles.active=dev
```

### ✅ Using command-line:

```bash
java -jar app.jar --spring.profiles.active=prod
```

### ✅ Programmatically:

```java
new SpringApplicationBuilder(MyApp.class)
    .profiles("test")
    .run(args);
```

---

## 🧪 5. Practical Example

```java
public interface MailService {
    void sendEmail(String to, String body);
}

@Profile("dev")
@Service
public class DummyMailService implements MailService {
    public void sendEmail(String to, String body) {
        System.out.println("Pretending to send email: " + body);
    }
}

@Profile("prod")
@Service
public class SmtpMailService implements MailService {
    public void sendEmail(String to, String body) {
        // Actual SMTP sending logic
    }
}
```

➡ Depending on the active profile (`dev` or `prod`), the right `MailService` will be injected.

---

## 🧠 6. Internal Mechanism

* The `@Profile` annotation is processed by `ProfileCondition` during the bean registration phase.
* Spring Environment (`Environment#getActiveProfiles()`) is queried.
* If the profile on the bean matches an active profile, the bean is included.

---

## ✅ 7. Best Practices

* Use `@Profile` to **avoid if-else logic** based on environments.
* Combine `@Profile` with `@Configuration` classes to load different property files or beans.
* Avoid putting `@Profile` inside business logic classes directly—keep them in config classes for clean separation.

---

## ⚠️ 8. Common Pitfalls

| Pitfall                    | Explanation                                               |
| -------------------------- | --------------------------------------------------------- |
| ❌ No default profile set   | Leads to no bean being registered                         |
| ❌ Misnamed profile         | Typos in profile names cause beans not to load            |
| ❌ Multiple active profiles | Can cause conflicts if multiple beans match the same type |

---

Excellent follow-up! `@Profile` and `@ConditionalOnProperty` are **both used to conditionally create beans**, but they serve different purposes and operate at different levels of **control and granularity**.

---

## 🔍 `@Profile` vs `@ConditionalOnProperty`

| Feature               | `@Profile`                                        | `@ConditionalOnProperty`                                                       |
| --------------------- | ------------------------------------------------- | ------------------------------------------------------------------------------ |
| **Purpose**           | Activate beans based on active Spring **profile** | Activate beans based on specific **property key/value**                        |
| **Scope**             | Coarse-grained (profile-wide)                     | Fine-grained (single or multiple properties)                                   |
| **Declaration**       | `@Profile("dev")`                                 | `@ConditionalOnProperty(name = "feature.order.enabled", havingValue = "true")` |
| **Typical Use**       | Different environments: `dev`, `prod`, `test`     | Feature toggles, optional components                                           |
| **Activation Source** | `spring.profiles.active=dev`                      | `application.properties`, YAML, env vars, or command-line                      |
| **Default Behavior**  | Bean is skipped if profile not active             | Can be made conditional with `matchIfMissing`                                  |

---

## 🎯 When to Use Each?

### ✅ Use `@Profile` when:

* You want to **group configuration or beans by environment**.
* Example:

  * `dev` profile uses `H2DatabaseConfig`
  * `prod` profile uses `PostgresDatabaseConfig`

### ✅ Use `@ConditionalOnProperty` when:

* You want to **enable/disable a specific feature or bean** using a property flag.
* Example:

  * Enable an email service only when `feature.email.enabled=true`

---

## 🧪 Example: `@ConditionalOnProperty`

```java
@Configuration
public class NotificationConfig {

    @Bean
    @ConditionalOnProperty(name = "notifications.sms.enabled", havingValue = "true", matchIfMissing = false)
    public SmsService smsService() {
        return new TwilioSmsService();
    }
}
```

### application.properties:

```properties
notifications.sms.enabled=true
```

If `notifications.sms.enabled=true`, the `smsService` bean will be created. If not, it will be skipped.

---

## ⚙️ Internal Working

* `@ConditionalOnProperty` is part of **Spring Boot’s conditional annotations**.
* It is processed during auto-configuration via the **`ConditionEvaluationReport`**.
* It checks the **Environment** for a matching property and value.

---

## ✅ Use Together?

Absolutely! In advanced setups:

```java
@Profile("prod")
@ConditionalOnProperty(name = "feature.online.order", havingValue = "true")
@Bean
public OrderService onlineOrderService() {
    return new OnlineOrderService();
}
```

This bean will only be created **if**:

1. The `prod` profile is active **AND**
2. `feature.online.order=true` is set

---

## 🧠 Summary

| Use Case                               | Use Annotation           |
| -------------------------------------- | ------------------------ |
| Different environments                 | `@Profile`               |
| Feature flags, toggles, optional beans | `@ConditionalOnProperty` |
| Combined control                       | Use both together        |

---



Absolutely! This is a **core feature in Spring Boot**—using **profile-specific property files** like `application-dev.properties`, `application-qa.properties`, etc. Let’s cover this deeply.

---

## 1️⃣ **How Profile-specific Property Files Work**

### **Naming Convention**

* Spring Boot looks for property files named:

  * `application.properties` (default)
  * `application-{profile}.properties` (profile-specific)

**Example:**

* `application.properties` – base/default config for all environments
* `application-dev.properties` – overrides for `dev` profile
* `application-qa.properties` – overrides for `qa` profile

---

## 2️⃣ **How Spring Loads These Files**

### **Load Order**

1. Loads `application.properties`
2. If a profile is active (e.g., `dev`), **loads and overrides** with values from `application-dev.properties`
3. You can have **multiple profiles active**—Spring merges and resolves precedence

---

## 3️⃣ **How to Activate a Profile**

* **Via application.properties:**

  ```properties
  spring.profiles.active=dev
  ```
* **Or via command line:**

  ```bash
  java -jar app.jar --spring.profiles.active=qa
  ```
* **Or via environment variable:**

  ```bash
  export SPRING_PROFILES_ACTIVE=prod
  ```

---

## 4️⃣ **Practical Example**

### **File Structure**

```
src/main/resources/
├── application.properties
├── application-dev.properties
└── application-qa.properties
```

### **application.properties**

```properties
logging.level=INFO
app.title=MyApp
```

### **application-dev.properties**

```properties
logging.level=DEBUG
app.title=MyApp [DEV]
database.url=jdbc:h2:mem:devdb
```

### **application-qa.properties**

```properties
logging.level=INFO
app.title=MyApp [QA]
database.url=jdbc:mysql://qa-db-server/app
```

---

### **Usage**

* When you run with **dev** profile active:

  * `logging.level` → `DEBUG`
  * `app.title` → `MyApp [DEV]`
  * `database.url` → `jdbc:h2:mem:devdb`

* When you run with **qa** profile active:

  * `logging.level` → `INFO`
  * `app.title` → `MyApp [QA]`
  * `database.url` → `jdbc:mysql://qa-db-server/app`

* If **no profile is active**:

  * Only `application.properties` is used

---

## 5️⃣ **Advanced: YAML Format**

You can also use YAML:

```yaml
# application.yml
spring:
  profiles:
    active: dev

---
spring:
  profiles: dev
database:
  url: jdbc:h2:mem:devdb

---
spring:
  profiles: qa
database:
  url: jdbc:mysql://qa-db-server/app
```

---

## 6️⃣ **How it Helps with Profiles**

* Property values are **switched automatically** depending on the active profile.
* Great for managing **database URLs, logging levels, API endpoints, feature flags**, etc.
* Combine this with `@Profile` or `@ConditionalOnProperty` to wire up beans/configurations and **complete environment separation**.

---

## 7️⃣ **Best Practices**

* Keep secrets (like passwords) out of VCS—use environment variables or externalized configs for sensitive info.
* Keep `application.properties` for defaults; **only override what’s necessary** in profile-specific files.

---

## 8️⃣ **Common Pitfall**

* If `spring.profiles.active` is not set, **profile-specific files won’t be loaded**.
* Always double-check file naming: must be `application-{profile}.properties`.

---

### **Summary Table**

| Profile | Config File Loaded         | Usage Scenario            |
| ------- | -------------------------- | ------------------------- |
| Default | application.properties     | Always loaded             |
| Dev     | application-dev.properties | Only when `dev` is active |
| QA      | application-qa.properties  | Only when `qa` is active  |

---

**In short:**
Profile-specific property files let you manage environment-specific configs with zero code change—just set the profile and Spring Boot does the rest!

---
