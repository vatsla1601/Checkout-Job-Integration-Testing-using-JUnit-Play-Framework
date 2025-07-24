
---

## 🚀 Smart Dependency Injection & Retry Logic with Guice

This project uses **[Google Guice](https://github.com/google/guice)** — a powerful dependency injection (DI) framework — to wire components together **automatically**, and to apply **cross-cutting retry logic** behind the scenes with zero boilerplate in your core business logic.

---

### 🧠 What’s Going On Behind the Scenes?

Rather than manually writing retry logic or cluttering your methods with checks, we use **Guice’s AOP (Aspect-Oriented Programming)** capabilities to hook into specific methods and apply behaviors like retry prevention — transparently.

#### 📦 It starts with a module: `RetryPreventionModule`

This class extends Guice’s `AbstractModule`, and registers method-level interceptors based on custom annotations:

```java
bindInterceptor(
    Matchers.any(),
    Matchers.annotatedWith(FilterBusObjWithMaxAttempts.class),
    new BusObjProactivePreventiveRetryAspect()
);
```

You can think of it as telling Guice:

> “Hey, whenever you see a method annotated with `@FilterBusObjWithMaxAttempts`, don’t just run it directly — wrap it with this retry-prevention logic first.”

---

### ⚙️ How the Whole Thing Gets Wired

At startup, your application calls:

```java
Guice.createInjector(
    new RetryPreventionModule(),
    ...
);
```

This tells Guice:

> “Load all the configurations from my module, apply any interceptors, and manage all injections from now on.”

Guice then builds an `Injector` — a kind of central factory — which injects dependencies and wraps annotated methods in their corresponding interceptors.

---

### 🎯 How It Works at Runtime

Let’s say you’ve written a method like this:

```java
@FilterBusObjWithMaxAttempts(retryJobContextMethodArgIndex = 0, searchCriteriaMethodArgIndex = 1)
public void doFinalCheckoutForAllTroughJob(...) {
    // Your actual business logic
}
```

Because of the module configuration, this method will automatically be wrapped by the `BusObjProactivePreventiveRetryAspect`, which may:

* Check if the job already failed too many times
* Skip it if retries are exhausted
* Log retry attempts or trigger alternative actions

You write your logic. The retry protection just happens.

---

### ✅ Why This Rocks

* 🧼 **No boilerplate** — Your core business logic stays clean and focused
* 🧠 **Centralized retry strategy** — Easy to configure and extend
* 🧪 **Test-friendly** — You can mock or verify retry behavior independently
* 🛠 **Fully pluggable** — You can add more annotations (`@CheckBusObjWithMaxAttempts`, `@UpdateBusObjRetryAttempts`, etc.) and hook them in the same way

---

### 🔍 TL;DR

Guice intercepts annotated methods like `@FilterBusObjWithMaxAttempts` and injects retry-prevention behavior defined in classes like `BusObjProactivePreventiveRetryAspect`.
All of this is bootstrapped via `RetryPreventionModule` during app startup using `Guice.createInjector()`.

Your methods stay clean. Your retries stay smart. Your dev life stays happier. ✅

---

