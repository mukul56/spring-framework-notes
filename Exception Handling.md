https://notebook.zohopublic.in/public/notes/9wn9o7260a3af90164c31941f061955edb73d

**stepwise, structured handwritten-notes-friendly summary**, starting from `@ExceptionHandler`, then `@ControllerAdvice`/global handler, then `@ResponseStatus`, and ending with their interactions and best practices.

---

# **Spring Exception Handling: Handwritten Notes**

---

## 1. **@ExceptionHandler**

* **Purpose:**
  Handle specific exceptions thrown in a controller method.
* **Where:**

  * Inside controller class (local handler)
  * Inside `@ControllerAdvice`/`@RestControllerAdvice` (global handler)
* **How:**
  Annotate a method to handle exceptions of a specific type.

**Example:**

```java
@ExceptionHandler(UserNotFoundException.class)
public ResponseEntity<ErrorResponse> handleUserNotFound(UserNotFoundException ex) {
    // Build and return error response
}
```

* Can return:

  * `ResponseEntity<T>` (best for REST, sets status)
  * POJO + `@ResponseStatus` on method (sets status)
  * POJO (defaults to 200 OK if no status given)
* **If handler exists, it “wins”—no other resolver is called.**

---

## 2. **Global Handler (`@ControllerAdvice` / `@RestControllerAdvice`)**

* **Purpose:**
  Apply exception handling, data binding, or model attributes globally to multiple controllers.
* **Usage:**

  * `@ControllerAdvice`: for both MVC (HTML) and REST (with `@ResponseBody` or method return type)
  * `@RestControllerAdvice`: for REST APIs (all responses as JSON/XML)
* **Benefits:**
  Centralizes error handling; avoids duplicating handlers in every controller.
* **Where:**
  Separate class annotated with `@ControllerAdvice` or `@RestControllerAdvice`.

**Example:**

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(UserNotFoundException.class)
    public ErrorResponse handleUserNotFound(UserNotFoundException ex) { ... }
}
```

* Catches exceptions thrown by any controller method.
* **Return type logic is the same as for local handlers.**

---

## 3. **@ResponseStatus**

* **Purpose:**
  Set the HTTP response status code for an exception or handler method.
* **Where:**

  * On **exception class**: tells Spring what status to use if this exception is not otherwise handled.
  * On **handler method**: tells Spring what status to use for this handler’s response.

**A. On Exception Class:**

```java
@ResponseStatus(HttpStatus.NOT_FOUND)
public class UserNotFoundException extends RuntimeException {}
```

* Used if exception is thrown and **no handler** exists.
* Handled by `ResponseStatusExceptionResolver`.

**B. On Handler Method:**

```java
@ExceptionHandler(UserNotFoundException.class)
@ResponseStatus(HttpStatus.NOT_FOUND)
public ErrorResponse handle(UserNotFoundException ex) { ... }
```

* Tells Spring to return 404 for this handler, even if you just return a POJO.

---

## 4. **Resolution Chain and Interactions**

**Resolution Order:**

1. **ExceptionHandlerExceptionResolver**

   * Looks for `@ExceptionHandler` methods (local/global)
   * If found, uses handler and stops chain
2. **ResponseStatusExceptionResolver**

   * Looks for:

     * Exception class with `@ResponseStatus`
     * `ResponseStatusException` thrown
   * Sets status and message accordingly
3. **DefaultHandlerExceptionResolver**

   * Handles Spring MVC exceptions (400, 405, etc.)
4. **(Spring Boot) BasicErrorController/DefaultErrorAttributes**

   * Handles any remaining unhandled exceptions

**Key Rules:**

* **If a handler method exists, it always takes priority.**

  * Handler’s return type/status/annotation is used
  * `@ResponseStatus` on exception class is ignored if handler exists
* **If no handler method,** Spring tries `@ResponseStatus` on exception class or `ResponseStatusException`.
* **If nothing matches,** fallback error JSON/page.

---

## 5. **Best Practices and Summary Table**

| Where?                    | How?                                                              | Status set by                                          | Return type            | Note                                      |
| ------------------------- | ----------------------------------------------------------------- | ------------------------------------------------------ | ---------------------- | ----------------------------------------- |
| Local handler             | `@ExceptionHandler`                                               | By handler (RespEntity or `@ResponseStatus` on method) | POJO or ResponseEntity | Highest priority; 200 OK if no status set |
| Global handler            | `@ControllerAdvice`/`@RestControllerAdvice` + `@ExceptionHandler` | By handler                                             | POJO/ResponseEntity    | Same as above                             |
| No handler                | `@ResponseStatus` on class                                        | By annotation                                          | None                   | Next in chain                             |
| No handler, not annotated | DefaultHandlerExceptionResolver                                   | By Spring                                              | None                   | For built-in exceptions                   |
| Nothing matches           | BasicErrorController/DefaultErrorAttributes                       | 500 or other                                           | JSON/HTML              | Fallback                                  |

---

## 6. **Pitfalls**

* **Returning POJO from handler without setting status** → sends 200 OK (even for errors)
* **Both handler and `@ResponseStatus` on exception class** → **handler wins** (class annotation ignored)
* **Using `@ControllerAdvice` for REST but forgetting `@ResponseBody`** → may return view name, not JSON

---

## 7. **Rule of Thumb**

* **Always use `@RestControllerAdvice` for REST APIs**
* **Set HTTP status explicitly in handler** (either `ResponseEntity` or `@ResponseStatus`)
* **Don’t rely on exception class annotation if handler exists**

---

### **One-Line Summary**

> The exception handler (`@ExceptionHandler`, local or global) always takes priority.
> If no handler, `@ResponseStatus` on the exception class or a thrown `ResponseStatusException` is used.
> Always explicitly set your error status in the handler for REST APIs!

---

Excellent question—this is a real-world scenario and a **critical point** for mastering Spring exception handling in layered architectures.

Let’s break it down thoroughly:

---

# **What About Exceptions Not Thrown Directly from Controllers?**

---

## 1. **Typical Spring Layered Architecture**

```
[Controller] ---> [Service] ---> [Repository/DAO] ---> [DB/External API]
                       |
                  [DTOs, Mappers, etc.]
```

* **Exceptions can originate in any layer**: service, repository, DTO mapping, etc.—not just in controller methods.

---

## 2. **How Spring Handles Exceptions from Other Layers**

### **A. Propagation Up the Call Stack**

* If a method in a service/repository/mapper throws an exception and that method is ultimately invoked by a controller (directly or indirectly), **the exception propagates up the call stack**.
* **If unhandled**, the exception will “bubble up” into the controller layer, as if the controller itself had thrown it.

**Example:**

```java
@RestController
public class UserController {
    @GetMapping("/users/{id}")
    public User getUser(@PathVariable int id) {
        return userService.getUserById(id); // Suppose this throws UserNotFoundException
    }
}
```

* If `userService.getUserById(id)` throws `UserNotFoundException`, and you have a handler for that exception (local or global), **it will be caught and handled by Spring’s exception handling mechanism**.

---

### **B. Exception Handling Flow Remains the Same**

* Spring’s resolver chain works **regardless of which layer the exception was originally thrown in**, **as long as it propagates to the controller boundary**.
* The exception does NOT need to be thrown or caught in the controller for global handling to work.

---

## 3. **What About Exceptions Thrown Outside the DispatcherServlet?**

* **Spring MVC’s exception handling (resolvers, advice, etc.) only works for requests handled by the DispatcherServlet**.
* If exceptions occur in background threads, scheduled jobs, or components not participating in HTTP request flow, **ControllerAdvice and the handler chain will NOT catch them**.

  * For those, you must use:

    * Custom exception handlers (e.g., try/catch in task runners)
    * Asynchronous error handlers
    * Global exception logging frameworks (e.g., via `@EventListener` for `ApplicationFailedEvent`, or integration with `Thread.UncaughtExceptionHandler`)

---

## 4. **Best Practices for Exception Propagation in Layers**

* **Let domain/service/repository exceptions bubble up:**
  Don’t try to catch and handle every exception in the lower layers unless you need to convert/transform it (e.g., rethrow a `SQLException` as a domain-specific exception).
* **Convert infrastructure exceptions to meaningful business exceptions:**
  For example, catch a `DataAccessException` and throw a `UserNotFoundException`.
* **Use global exception handling (`@ControllerAdvice`/`@RestControllerAdvice`) to map exceptions to appropriate HTTP responses**.

---

## 5. **Example: Service Layer Exception Propagation**

**Service:**

```java
@Service
public class UserService {
    public User getUserById(int id) {
        // throws UserNotFoundException if not found
    }
}
```

**Controller:**

```java
@RestController
public class UserController {
    @GetMapping("/users/{id}")
    public User getUser(@PathVariable int id) {
        return userService.getUserById(id); // exception propagates
    }
}
```

**Global Handler:**

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(UserNotFoundException.class)
    public ErrorResponse handleNotFound(UserNotFoundException ex) { ... }
}
```

**No try/catch needed in controller.**

---

## 6. **Edge Cases & Gotchas**

* **Checked exceptions:**
  If you declare a checked exception and don’t handle it, you must declare it in the method signature, or wrap it in an unchecked exception.
* **Asynchronous code:**
  Exceptions thrown in `@Async` methods will NOT be handled by global exception handlers. You need to handle them via `Future`, `CompletableFuture`, or by setting up custom error callbacks.

---

## 7. **Summary Table**

| Where Exception Is Thrown    | Will Global Handler Work? | Why/Why Not                               |
| ---------------------------- | ------------------------- | ----------------------------------------- |
| Service called by controller | Yes                       | Exception bubbles up to DispatcherServlet |
| Repository called by service | Yes                       | Same as above                             |
| DTO/mapper called by service | Yes                       | Same as above                             |
| Scheduled/background task    | No                        | Not handled by DispatcherServlet          |
| @Async method                | No                        | Not handled by DispatcherServlet          |

---

## 8. **Key Takeaway (Write in Box)**

> **Any exception (checked or unchecked) thrown in any layer—service, repository, DTO, etc.—that propagates to the DispatcherServlet via a controller method will be handled by Spring’s global exception handling mechanism. Only exceptions outside the HTTP request flow need separate handling.**

---


Great advanced question!
**What happens if you define multiple `@ExceptionHandler` methods for the same exception?**
Let’s explore all possible scenarios and Spring’s behavior, including best practices and edge cases.

---

# **Multiple `@ExceptionHandler` for Same Exception**

---

## **1. Within the Same Class**

You **cannot** have two `@ExceptionHandler` methods for the **exact same exception type** in the same class (controller or advice).

* **Spring will fail to start** with a bean creation error:
  *“Ambiguous mapping: Cannot map ‘handler’ method ... There is already ‘handler’ bean method ...”*

**Example (illegal, will fail):**

```java
@RestControllerAdvice
public class MyAdvice {
    @ExceptionHandler(SomeException.class)
    public ErrorResponse handler1(SomeException ex) { ... }

    @ExceptionHandler(SomeException.class)
    public ErrorResponse handler2(SomeException ex) { ... }
}
```

* **Result:** Application context fails to start.
* **Reason:** Spring needs to know which method to call. Duplicate mappings are not allowed.

---

## **2. Across Different Classes (Controller + Global Advice)**

Suppose you have:

* A controller with a local handler:

  ```java
  @RestController
  public class MyController {
      @ExceptionHandler(SomeException.class)
      public ErrorResponse localHandler(SomeException ex) { ... }
  }
  ```
* And a global handler:

  ```java
  @RestControllerAdvice
  public class GlobalAdvice {
      @ExceptionHandler(SomeException.class)
      public ErrorResponse globalHandler(SomeException ex) { ... }
  }
  ```

**What happens if `SomeException` is thrown?**

* **Spring prioritizes local handlers over global ones.**
* If the exception is thrown inside `MyController`,
  `localHandler()` is called.
* If thrown elsewhere, and only global handler matches,
  `globalHandler()` is called.

**You cannot have two matching local handlers in the same controller** (same as scenario 1—Spring fails to start).

---

## **3. Different Exception Types in Hierarchy**

You can have:

* One handler for a **parent** exception (e.g., `Exception.class`)
* One handler for a **specific** exception (e.g., `SomeException.class`)
* **Spring chooses the most specific handler**.

---

## **4. What About Method Parameters?**

Spring will match the `@ExceptionHandler` method whose parameter type matches (or is closest to) the thrown exception.

---

## **5. Summary Table**

| Scenario                                             | Allowed?                                                                                                                                                 | Who Handles?                          | Result               |
| ---------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------- | -------------------- |
| Multiple handlers for same exception in same class   | ❌                                                                                                                                                        | -                                     | Spring startup error |
| Local + global handler for same exception            | ✅                                                                                                                                                        | Local handler wins (if in controller) | Local handler called |
| Multiple global handlers in different advice classes | ❌ (ambiguous, Spring fails to start if same exception in same advice class; if in separate, depends on order and basePackages/assignableTypes filtering) | Ambiguity if not filtered             |                      |
| Handlers for parent and child exception types        | ✅                                                                                                                                                        | Most specific handler                 | Closest match called |

---

## **6. Best Practice**

* **Never define two handlers for the same exception in the same advice/controller class.**
* **Local controller handlers take precedence over global advice.**
* **Use exception hierarchy for fallback/global error handling.**
* **Filter advice classes with `basePackages`, `assignableTypes`, or `annotations` to avoid ambiguity in large apps.**

---

## **Key Takeaway**

> **Spring will throw an error if multiple handlers for the same exception exist in the same class.
> Local handlers always win over global ones for that controller.
> For exception hierarchies, the most specific handler is chosen.**

---

 
 Let’s do a **real-world REST API example** that covers all Spring exception handling features **in a single, relatable scenario**.

### **Scenario:**

Imagine a “User Management” REST API with endpoints to **get a user** and **register a new user**. We’ll show:

* Service throws custom exceptions from the business layer
* Some exceptions have `@ResponseStatus`, some do not
* Both local and global handlers
* Fallback/global error handling

---

## **1. Custom Exception Classes**

```java
// Exception with @ResponseStatus for not found
@ResponseStatus(HttpStatus.NOT_FOUND)
public class UserNotFoundException extends RuntimeException {
    public UserNotFoundException(String msg) { super(msg); }
}

// Exception for duplicate user, without @ResponseStatus
public class DuplicateUserException extends RuntimeException {
    public DuplicateUserException(String msg) { super(msg); }
}
```

---

## **2. Service Layer (Where Exceptions Originate)**

```java
@Service
public class UserService {
    private Map<String, String> userDb = new HashMap<>();

    public String getUser(String username) {
        if (!userDb.containsKey(username)) {
            throw new UserNotFoundException("User '" + username + "' not found.");
        }
        return userDb.get(username);
    }

    public void registerUser(String username, String details) {
        if (userDb.containsKey(username)) {
            throw new DuplicateUserException("User '" + username + "' already exists.");
        }
        userDb.put(username, details);
    }
}
```

---

## **3. Controller Layer**

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    @Autowired
    private UserService userService;

    // A. Local Exception Handler for DuplicateUserException
    @ExceptionHandler(DuplicateUserException.class)
    public ResponseEntity<ErrorResponse> handleDuplicate(DuplicateUserException ex) {
        return new ResponseEntity<>(
            new ErrorResponse("DUPLICATE: " + ex.getMessage()),
            HttpStatus.CONFLICT // 409
        );
    }

    // B. Endpoints
    @GetMapping("/{username}")
    public String getUser(@PathVariable String username) {
        return userService.getUser(username); // May throw UserNotFoundException
    }

    @PostMapping
    public String registerUser(@RequestParam String username, @RequestParam String details) {
        userService.registerUser(username, details); // May throw DuplicateUserException
        return "User registered!";
    }
}
```

---

## **4. Global Exception Handler**

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    // Handles all other exceptions, e.g., NullPointerException, etc.
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleAll(Exception ex) {
        return new ErrorResponse("UNEXPECTED: " + ex.getClass().getSimpleName() + " - " + ex.getMessage());
    }
}
```

---

## **5. ErrorResponse DTO**

```java
public class ErrorResponse {
    private String message;
    public ErrorResponse(String message) { this.message = message; }
    public String getMessage() { return message; }
}
```

---

# **How It Works in Real Use**

### **A. GET `/api/users/john` (when “john” not in system)**

* **Service throws:** `UserNotFoundException` (annotated with `@ResponseStatus(HttpStatus.NOT_FOUND)`)
* **No local handler, no global handler for this**
* **Spring uses:** `@ResponseStatus` from exception class via `ResponseStatusExceptionResolver`
* **HTTP 404**

  ```json
  {
    "timestamp": "...",
    "status": 404,
    "error": "Not Found",
    "message": "User 'john' not found.",
    "path": "/api/users/john"
  }
  ```

---

### **B. POST `/api/users?username=john&details=hello` (when “john” already exists)**

* **Service throws:** `DuplicateUserException`
* **Controller has local handler for this exception**
* **Handler called:** `handleDuplicate()`, returns status 409 and custom message

  ```json
  {
    "message": "DUPLICATE: User 'john' already exists."
  }
  ```

  **HTTP 409 CONFLICT**

---

### **C. GET `/api/users/exception-test` (suppose this endpoint throws `NullPointerException`)**

* **No local handler**
* **Global handler for `Exception.class` handles it**
* **Returns:**

  ```json
  {
    "message": "UNEXPECTED: NullPointerException - Something went wrong"
  }
  ```

  **HTTP 500 INTERNAL SERVER ERROR**

---

# **Summary Table**

| Exception              | Where Thrown | Which Handler Handles?                | Status | Message Source            |
| ---------------------- | ------------ | ------------------------------------- | ------ | ------------------------- |
| UserNotFoundException  | Service      | @ResponseStatus on exception          | 404    | Default Spring error JSON |
| DuplicateUserException | Service      | Local @ExceptionHandler in controller | 409    | Handler return            |
| NullPointerException   | Anywhere     | Global @ExceptionHandler(Exception)   | 500    | Handler return            |

---

# **Why This Is “Real World”**

* **Service-layer exceptions propagate up to controllers.**
* **Different error handling for business errors (404, 409) vs unknown errors (500).**
* **Mix of local, global, and fallback error handling.**
* **Shows separation of concerns: controller does not “catch” exceptions from service.**

---

## **Want a more advanced scenario?**

* Add validation (`@Valid`) and handler for `MethodArgumentNotValidException`
* Add database-layer (JPA) exception mapping
* Return error codes or trace IDs in the error response

Let me know!


