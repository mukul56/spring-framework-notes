---

# **1. `@Id` Annotation**

## **1.1. Introduction & Definition**

* `@Id` is a **JPA annotation** used to specify the **primary key** property or field of an entity.
* It tells the JPA provider (e.g., Hibernate) which field should be used as the **unique identifier** for records in the database table.

```java
@Entity
public class User {
    @Id
    private Long id;
    private String name;
    // ...
}
```

* **Primary key** is mandatory for every JPA entity (except for special cases like Embeddables, explained later).

---

## **1.2. Core Principles**

* **Uniqueness**: Each entity instance must have a unique value for the `@Id` field.
* **Non-null**: The primary key must not be null.
* **Immutability** (recommended): Once set, the primary key value should not change (though JPA does not enforce this).
* **Mapping**: Can be on a single field or multiple fields (composite keys).

---

## **1.3. Practical Usage & Code Examples**

### **a) Simple Primary Key**

```java
@Entity
public class Product {
    @Id
    private Long productId;
    private String name;
}
```

### **b) With @GeneratedValue**

You can use `@GeneratedValue` to auto-generate the ID:

```java
@Entity
public class Customer {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
}
```

* Strategies: `AUTO`, `IDENTITY`, `SEQUENCE`, `TABLE`.

---

### **c) Composite Primary Key**

For tables where the primary key is made up of multiple columns.

#### **Option 1: `@IdClass`**

```java
@Entity
@IdClass(OrderPK.class)
public class Order {
    @Id
    private Long orderId;
    @Id
    private Long productId;
    private int quantity;
}
// Separate PK class
public class OrderPK implements Serializable {
    private Long orderId;
    private Long productId;
    // equals & hashCode
}
```

#### **Option 2: `@EmbeddedId`**

```java
@Entity
public class Enrollment {
    @EmbeddedId
    private EnrollmentId id;
    private String grade;
}

@Embeddable
public class EnrollmentId implements Serializable {
    private Long studentId;
    private Long courseId;
    // equals & hashCode
}
```

> **Note:** This is where `@Embeddable` comes into play. More below!

---

## **1.4. Advanced Internals & Configuration**

* JPA tracks entities by their `@Id` value in the **persistence context**.
* The `@Id` field **must be mapped** to a column that has a unique constraint (PRIMARY KEY in SQL).
* **Proxying**: Hibernate uses the ID to lazily fetch objects (`getReference` returns a proxy with just the ID).

---

## **1.5. Best Practices**

* Use a **surrogate key** (`Long`, `UUID`, etc.) rather than natural keys for simplicity.
* Always override `equals()` and `hashCode()` for entities (especially with composite keys).
* Never change the value of the ID after the entity is persisted.
* Prefer `@GeneratedValue` for numeric IDs to avoid managing IDs yourself.

---

## **1.6. Real-World Applications**

* **CRUD Operations**: All entity lookups, updates, and deletes rely on the primary key.
* **Relationships**: Foreign keys in associations reference the `@Id` of the target entity.

---

## **1.7. Common Pitfalls & Edge Cases**

* **No @Id**: JPA will throw a `PersistenceException` if no `@Id` is defined.
* **Duplicate @Id**: If two entities have the same ID in the persistence context, unexpected behavior (usually exceptions).
* **Changing @Id**: Modifying the primary key after persisting will break entity tracking and relations.
* **Composite Keys**: Forgetting `equals()`/`hashCode()` on `@Embeddable` IDs will break cache and retrieval logic.
* **Null IDs**: Attempting to persist an entity with a null ID (without auto-generation) will fail.

---

# **2. `@Embeddable` Annotation**

## **2.1. Introduction & Definition**

* `@Embeddable` marks a class whose **fields are mapped as part of another entity**.
* **Embeddables are NOT entities:** They have **no identity** of their own, and cannot be queried independently.
* Often used for:

  * **Composite primary keys** (with `@EmbeddedId`)
  * **Value objects** (e.g., Address, Money)

```java
@Embeddable
public class Address {
    private String street;
    private String city;
}
```

---

## **2.2. Core Concepts**

* **Reusability:** Encapsulates a set of fields to be reused across entities.
* **Composition over inheritance:** Embeddables are used to **compose** entities.

---

## **2.3. Practical Usage & Code Examples**

### **a) Embedded Value Object**

```java
@Entity
public class Employee {
    @Id
    @GeneratedValue
    private Long id;

    @Embedded
    private Address address;
}
```

### **b) EmbeddedId for Composite Key**

```java
@Entity
public class BookAuthor {
    @EmbeddedId
    private BookAuthorId id;

    private String role;
}

@Embeddable
public class BookAuthorId implements Serializable {
    private Long bookId;
    private Long authorId;
    // equals() and hashCode()!
}
```

---

## **2.4. Advanced Configuration & Internals**

* **AttributeOverrides:** Allows you to customize column names for embedded fields.

```java
@Embedded
@AttributeOverrides({
    @AttributeOverride(name = "street", column = @Column(name = "home_street")),
    @AttributeOverride(name = "city", column = @Column(name = "home_city"))
})
private Address homeAddress;
```

* **Embeddable in Embeddable**: Embeddable classes can contain other embeddables.
* **Lifecycle**: Embeddables don’t have their own lifecycle, they're persisted with the owning entity.
* **Nullability**: All embeddable fields must be serializable, and are stored in the same table as the owning entity.

---

## **2.5. Best Practices**

* Use for **small value objects** that don't need their own table.
* Always implement `Serializable` (required for composite keys).
* For composite keys: implement **equals() and hashCode()** using all fields.

---

## **2.6. Real-World Applications**

* **Address** objects used in multiple entities (User, Order, Company, etc).
* **Money, GeoLocation, Period (start/end date)**, etc.
* **Audit Fields**: CreatedBy, UpdatedBy, Timestamps as a reusable embeddable.

---

## **2.7. Common Pitfalls & Edge Cases**

* **No @Embeddable/@Embedded**: Fields won’t be mapped, or mapping errors will occur.
* **Column name clashes**: Two embedded objects with the same field names—fix with `@AttributeOverrides`.
* **Cyclic Embedding**: An embeddable containing itself (directly or indirectly) causes mapping errors.
* **Embeddable cannot be an Entity**: Do not use `@Entity` and `@Embeddable` on the same class.

---

# **3. Putting It All Together: Example with Both**

Suppose you want to model a `StudentCourseEnrollment` where each row is uniquely identified by `(studentId, courseId)` and you want to store an `Address` as well.

```java
@Entity
public class StudentCourseEnrollment {
    @EmbeddedId
    private EnrollmentId id;

    @Embedded
    private Address address;

    private String status;
}

@Embeddable
public class EnrollmentId implements Serializable {
    private Long studentId;
    private Long courseId;

    // equals & hashCode (all fields)
}

@Embeddable
public class Address {
    private String street;
    private String city;
}
```


Great clarifying questions! Let’s break down both:

---

## **1. Does `@Embeddable` Make the Object a Composite Primary Key?**

### **Short Answer:**

Not **automatically**.
`@Embeddable` **defines** a class whose fields can be embedded into other entities, but **just annotating a class as `@Embeddable` does NOT make it a composite primary key by itself**.

#### **Composite Primary Key Use**

* To make an embeddable object act as a composite primary key, you must use it **with** the `@EmbeddedId` annotation inside your entity.

### **How It Works:**

```java
@Embeddable
public class EnrollmentId implements Serializable {
    private Long studentId;
    private Long courseId;
    // equals and hashCode
}

@Entity
public class Enrollment {
    @EmbeddedId
    private EnrollmentId id; // THIS makes it the PK!
    // other fields
}
```

* Here, `EnrollmentId` (marked with `@Embeddable`) is used as a **composite key** via `@EmbeddedId` in the `Enrollment` entity.

### **Summary Table:**

| Annotation    | On What?     | Purpose                                |
| ------------- | ------------ | -------------------------------------- |
| `@Embeddable` | Class        | Marks class as embeddable value object |
| `@EmbeddedId` | Entity Field | Makes the embeddable the composite PK  |

**So, just `@Embeddable` = Value Object; `@Embeddable` + `@EmbeddedId` = Composite PK.**

---

## **2. Can We Search/Query Entities Using Embeddable Composite Keys?**

### **Yes**, but you need to use the embeddable key as the identifier when searching.

### **How You Query:**

#### **Find by ID (Composite Key):**

```java
EnrollmentId eid = new EnrollmentId();
eid.setStudentId(1L);
eid.setCourseId(10L);

Enrollment enrollment = entityManager.find(Enrollment.class, eid);
```

* **Spring Data JPA Example:**

```java
public interface EnrollmentRepository extends JpaRepository<Enrollment, EnrollmentId> {}

// Usage
EnrollmentId eid = new EnrollmentId(1L, 10L);
Optional<Enrollment> enrollment = enrollmentRepository.findById(eid);
```

#### **JPQL Example:**

```java
@Query("SELECT e FROM Enrollment e WHERE e.id.studentId = :sid AND e.id.courseId = :cid")
Enrollment findByStudentAndCourse(@Param("sid") Long studentId, @Param("cid") Long courseId);
```

---

## **Best Practices**

* Always override `equals()` and `hashCode()` in your `@Embeddable` key class!
* Use a single embeddable for your composite key, **do not mix** `@Id` and `@EmbeddedId` in the same entity.

---

## **Edge Cases & Pitfalls**

* **Cannot query the embeddable on its own:** Embeddables have no separate table, no independent queries—**they exist only inside an entity.**
* **Composite key required in full:** To fetch by PK, you must specify all parts of the composite key.

---

### **Summary**

* `@Embeddable` by itself = value object.
* `@Embeddable` + `@EmbeddedId` in an entity = composite primary key.
* You **search** by providing an instance of the embeddable key as the ID.

---


**how JPA/Hibernate optimize sequence generation** using *allocationSize* and how IDs are cached to avoid round-trips to the database.

---

## **How Sequence Generation Works in JPA/Hibernate**

### **1. Default Approach (allocationSize = 1)**

* When you use `@GeneratedValue(strategy = GenerationType.SEQUENCE)`, by default, every time you persist an entity, Hibernate/JPA will:

  1. Query the sequence from the database: `SELECT nextval('sequence_name')`
  2. Use that returned value as the entity’s ID in the INSERT statement.
* **One database call per entity**.
  **No caching.**

**Example:**

```java
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "emp_seq")
@SequenceGenerator(name = "emp_seq", sequenceName = "employee_seq", allocationSize = 1)
private Long id;
```

* Every `persist()` → 1 call to DB for sequence.

---

### **2. Optimized Approach (allocationSize > 1, e.g. 50)**

* **Problem:** One database call per entity is slow for batch inserts.
* **Solution:** Use `allocationSize > 1` (e.g., 50 or 100).
* Now, **Hibernate requests a block of IDs** from the sequence in a single call:

  * `SELECT nextval('sequence_name')` returns (say) 100.
* Hibernate then computes a **range of IDs** in memory using that value as the "high value."
* **All inserts for the next 100 entities use those IDs—no DB call needed for each!**

**Example:**

```java
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "emp_seq")
@SequenceGenerator(name = "emp_seq", sequenceName = "employee_seq", allocationSize = 50)
private Long id;
```

* **First persist:** Hibernate calls the DB for nextval, gets (e.g.) 101.
* **Hibernate knows** it can assign IDs from 52 up to 101 (if allocationSize=50, and the DB sequence was at 51).
* **Subsequent persists:** Hibernate assigns IDs in memory, without calling the DB, until the cache is exhausted.

---

### **3. How the "Cache" Works Internally**

* **Hibernate maintains a local pool/cache of IDs.**
* For each new persist, it just hands out the next number in the cached block.
* When the block is exhausted, it asks the DB for a new one (by getting nextval from the sequence again, and recalculating the range).
* **allocationSize** controls the size of the block/cache.

---

### **4. Real-World Example Walkthrough**

Suppose:

* Sequence is at 100.
* `allocationSize = 50`.

**First block:**

* Hibernate requests nextval (gets 150).
* It knows it can use IDs from 101 to 150.
* Persists 50 entities with IDs 101..150.
* Only when it needs ID 151 (the 51st entity) does Hibernate ask DB again.

---

### **5. Benefits of Sequence Block Allocation**

* **Major performance boost** for batch inserts.
* Fewer DB calls, less network overhead.

### **6. Trade-offs & Edge Cases**

* If the application/server **crashes** after pre-allocating a block, unused IDs in the range are **skipped** (gap in sequence).
* Not a problem for technical PKs, but for business PKs that must be strictly sequential (like invoice numbers), it’s not suitable.
* Always ensure the `allocationSize` in your entity matches the actual increment on the sequence in the DB! (Otherwise you’ll get ID collisions or skipped values.)

---

### **7. How Do You Control This?**

```java
@SequenceGenerator(
  name = "my_seq",
  sequenceName = "my_seq",
  allocationSize = 50   // This sets the size of the ID cache
)
```

---

## **Summary Table**

| allocationSize | DB Calls per N Inserts | IDs Cached | Best For              | Possible Issue         |
| -------------- | ---------------------- | ---------- | --------------------- | ---------------------- |
| 1              | N                      | None       | Simple/rare inserts   | Slow for batch inserts |
| 50             | N/50                   | Yes (50)   | Batch/high throughput | Gaps if crash occurs   |

---

## **Internal Flow (Hibernate):**

1. Persist entity.
2. If ID cache is empty:

   * Get new "high" value from sequence (`nextval`).
   * Compute ID range using allocationSize.
   * Cache range.
3. Assign ID from cache.
4. Repeat for each persist until cache is exhausted.

---

### **In Summary**

* **Who manages the ID cache?**
  Hibernate/JPA, in-memory, per `@SequenceGenerator`.

* **When does Hibernate call the DB for IDs?**
  Only when the local cache is exhausted (after allocationSize inserts).

* **What happens on crash?**
  Unused IDs from the current block will be lost (sequence jumps ahead).

---



