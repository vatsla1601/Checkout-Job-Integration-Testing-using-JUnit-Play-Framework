
---

# 🔁 Retry Prevention with Custom Annotations and AOP

This project uses **custom annotations** combined with **Aspect-Oriented Programming (AOP)** to manage and limit retry attempts for operations (like visitor checkouts) based on configurable rules and runtime behavior analysis.

---

## ✅ Key Annotations:

### `@FilterBusObjWithMaxAttempts`

Filters out business objects that have exceeded their maximum retry attempts **before** processing.

```java
@FilterBusObjWithMaxAttempts(retryJobContextMethodArgIndex = 0, searchCriteriaMethodArgIndex = 1)
```

---

### `@UpdateBusObjRetryAttempts`

Automatically updates retry attempt count **after** an operation completes.

```java
@UpdateBusObjRetryAttempts(retryJobContextMethodArgIndex = 0, retryResourceMethodArgIndex = 1)
```

---

## 🔍 How it Works:

These annotations are processed by **Guice AOP interceptors** defined in `RetryPreventionModule`.

Each annotation specifies:

* The **index of method parameters** (e.g., `0`, `1`) to instruct the interceptor:

  * Where to find the **retry context** (e.g., `VisitorJobContext`)
  * Which object to **filter or update** (e.g., `VisitsUIDTO`, `SearchCriteria`)

### Interceptors use this to:

* ✅ **Skip** processing if max attempts are exceeded (`@FilterBusObjWithMaxAttempts`)
* 🔁 **Increment** retry count after operation (`@UpdateBusObjRetryAttempts`)

This design cleanly **separates retry logic from business logic** using annotations and AOP.

---

## 🔁 What does `invocation.proceed()` do?

Inside your `invoke()` method:

```java
Object returnValue = invocation.proceed();
```

This line executes the **actual method** that your interceptor is intercepting.

Think of `invocation.proceed()` as:

> "Now go and run the original method logic (which I'm intercepting), and return the result."

So the aspect:

* ✅ Does something **before** (`e.g. check retry count`)
* ▶️ Proceeds with the **actual method execution**
* 🔄 Does something **after** (e.g., update retry count or log result)

---

## 📦 RetryPreventionModule – Binding Everything Together

```java
@Module
public class RetryPreventionModule extends AbstractModule {
    @Override
    protected void configure() {
        BusObjProactivePreventiveRetryAspect proactivePreventiveRetryAspect = new BusObjProactivePreventiveRetryAspect();
        requestInjection(proactivePreventiveRetryAspect);
        bindInterceptor(Matchers.any(), Matchers.annotatedWith(FilterBusObjWithMaxAttempts.class),
                proactivePreventiveRetryAspect);

        BusObjReactivePreventiveRetryAspect reactivePreventiveRetryAspect = new BusObjReactivePreventiveRetryAspect();
        requestInjection(reactivePreventiveRetryAspect);
        bindInterceptor(Matchers.any(), Matchers.annotatedWith(CheckBusObjWithMaxAttempts.class),
                reactivePreventiveRetryAspect);

        BusObjUpdateRetryAttemptsAspect updateRetryAttemptsAspect = new BusObjUpdateRetryAttemptsAspect();
        requestInjection(updateRetryAttemptsAspect);
        bindInterceptor(Matchers.any(), Matchers.annotatedWith(UpdateBusObjRetryAttempts.class),
                updateRetryAttemptsAspect);
    }
}
```

---

## 🔄 How Everything Works Together

### 1. **Your Custom Annotations**:

* `@FilterBusObjWithMaxAttempts`
* `@CheckBusObjWithMaxAttempts`
* `@UpdateBusObjRetryAttempts`

➡️ These are **metadata** that tell Guice:

> “If a method is annotated with this, attach an aspect to control its behavior.”

---

### 2. **Your Aspect Classes**:

* `BusObjProactivePreventiveRetryAspect` → Handles `@FilterBusObjWithMaxAttempts`
* `BusObjReactivePreventiveRetryAspect` → Handles `@CheckBusObjWithMaxAttempts`
* `BusObjUpdateRetryAttemptsAspect` → Handles `@UpdateBusObjRetryAttempts`

Each of them:

* Intercepts the method call
* Can check retry limits
* Decide whether to continue (`proceed()`) or **skip**
* Can **update state**, log, etc.

---

### 3. **The Bindings in Guice**:

```java
bindInterceptor(Matchers.any(), Matchers.annotatedWith(FilterBusObjWithMaxAttempts.class),
                proactivePreventiveRetryAspect);
```

This means:

> “If any method has `@FilterBusObjWithMaxAttempts`, call `BusObjProactivePreventiveRetryAspect.invoke()` around it.”

---

## 📘 Real-Life Flow Example:

```java
@FilterBusObjWithMaxAttempts(retryJobContextMethodArgIndex = 0, searchCriteriaMethodArgIndex = 1)
public void doFinalCheckoutForAllTroughJob(VisitorJobContext context, SearchCriteria criteria, ...) {
    // your business logic
}
```

**When this method is called:**

* Guice sees the annotation
* Routes the call through:

```java
BusObjProactivePreventiveRetryAspect.invoke(MethodInvocation invocation)
```

Which:

* Extracts method parameters
* Checks retry limits
* Decides to proceed or skip

---

## 🔖 `@Retention` in Java – Technical Explanation

`@Retention` is a **meta-annotation** (i.e., annotation used on other annotations).
It tells the compiler how long the annotation should be **retained** and **accessible**.

---

### 🧠 Retention Policies:

| Retention Policy    | Description                                                             | Use Case                      |
| ------------------- | ----------------------------------------------------------------------- | ----------------------------- |
| `SOURCE`            | Present only in source code, discarded during compilation               | Tools like Lombok             |
| `CLASS`             | Stored in `.class` files but not available at runtime                   | Bytecode analyzers            |
| `RUNTIME` (✅ yours) | Available at runtime via **reflection** (used by frameworks like Guice) | Spring, Guice, Hibernate, AOP |

```java
@Retention(RetentionPolicy.RUNTIME)
public @interface FilterBusObjWithMaxAttempts {
    int retryJobContextMethodArgIndex() default 0;
    int searchCriteriaMethodArgIndex() default 1;
}
```

> You're using `RUNTIME` because Guice AOP needs to read the annotation **at runtime** using reflection.

---

### ⚠️ Without `@Retention(RUNTIME)`:

Your annotation would not be visible at runtime, and **Guice interceptors wouldn't work**.

---

## 🔖 Your Annotations and Their Parameters

### 1. `@FilterBusObjWithMaxAttempts(retryJobContextMethodArgIndex = 0, searchCriteriaMethodArgIndex = 1)`

Applied on:

```java
public void doFinalCheckoutForAllTroughJob(
    VisitorJobContext context,                    // index 0
    SearchCriteria finalCheckOutSearchCriteria,   // index 1
    LocationDTO location,                         // index 2
    int pageSize                                  // index 3
)
```

✅ What it means:

* `retryJobContextMethodArgIndex = 0`: → Use param at index 0 as the **retry context**
* `searchCriteriaMethodArgIndex = 1`: → Use param at index 1 as the **resource to filter**

---

### 2. `@UpdateBusObjRetryAttempts(retryJobContextMethodArgIndex = 0, retryResourceMethodArgIndex = 1)`

Applied on:

```java
public VisitsUIDTO doFinalCheckoutOnVisit(
    VisitorJobContext context,     // index 0
    VisitsUIDTO visitorVisit,      // index 1
    LocationDTO location           // index 2
)
```

✅ What it means:

* `retryJobContextMethodArgIndex = 0`: → Use param 0 for **context**
* `retryResourceMethodArgIndex = 1`: → Use param 1 for **resource to update retry**

---

### 🔍 How parameters are used in aspect logic:

```java
Optional<BusObjRetryAwareJobContext> optContext = getObjectFromArgs(
    methodArgs, updateBusObjRetryAttempts.retryJobContextMethodArgIndex(), BusObjRetryAwareJobContext.class
);

Optional<? extends DTO> optResource = getObjectFromArgs(
    methodArgs, updateBusObjRetryAttempts.retryResourceMethodArgIndex(), DTO.class
);
```

* `methodArgs[]` holds the actual arguments passed to your method.
* The annotation tells the interceptor **which argument to treat as retry context/resource**.

---

## ✅ TL;DR – What Do Annotation Parameters Mean?

| Annotation                     | Parameter Name                  | Index | Description                                    |
| ------------------------------ | ------------------------------- | ----- | ---------------------------------------------- |
| `@FilterBusObjWithMaxAttempts` | `retryJobContextMethodArgIndex` | `0`   | Use argument at index 0 for retry context      |
|                                | `searchCriteriaMethodArgIndex`  | `1`   | Use argument at index 1 for filtering resource |
| `@UpdateBusObjRetryAttempts`   | `retryJobContextMethodArgIndex` | `0`   | Retry context parameter                        |
|                                | `retryResourceMethodArgIndex`   | `1`   | DTO whose retry count should be incremented    |

---

## 🔍 What’s Being Retained in Your Java Files?

✅ The **annotation presence** and its **parameter values** (like `retryJobContextMethodArgIndex = 0`) are retained **at runtime**, because of:

```java
@Retention(RetentionPolicy.RUNTIME)
```

This makes them readable by AOP via reflection.

---

Let me know if you'd like a **diagram**, flowchart, or **code walkthrough GIF** to go with this README!
