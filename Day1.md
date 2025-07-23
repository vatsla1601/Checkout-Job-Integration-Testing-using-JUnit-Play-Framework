Here’s a **Q\&A formatted summary** of your Java test framework concepts and doubts that you can directly use in your **GitHub README** section under `🧩 Doubts, Questions & Concepts with Answers`:

---

## 🧩 Doubts, Questions & Concepts with Answers (Java + Play Framework)

---

### ❓1. Is my test class starting the application/server?

✅ **Answer:**
Yes. The method `Helpers.start(application)` is starting the Play application in **TEST** mode.
It’s used in your `@BeforeClass` block, which means the application is bootstrapped once before all tests run.

📌 Why? Because integration tests need a real running environment with configs, services, and DB access.

---

### ❓2. What is `GuiceApplicationBuilder`?

✅ **Answer:**
It’s a Play Framework utility to **manually build** an application instance with **custom configuration or modules**, especially useful during tests.

🔧 Used when:

* You need to override modules (e.g., inject mock DB)
* You want test-specific configuration
* You need to control what gets loaded

---

### ❓3. What does `Helpers.start(app)` do?

✅ **Answer:**
It **starts** the application instance created with `GuiceApplicationBuilder`.

📌 Analogy:

* `GuiceApplicationBuilder` → builds the car
* `Helpers.start(app)` → turns the ignition and starts it

🧪 This is crucial in **test setup** to make the app ready for integration tests.

---

### ❓4. What does this line of code mean?

```java
builder = (new CustomApplicationLoader()).builder(
  new ApplicationLoader.Context(
    new Environment(
      new File("."), 
      AbstractBaseTest.class.getClassLoader(), 
      Mode.TEST
    )
  )
);
```

✅ **Answer:** You're creating an environment to run Play app in TEST mode.

* `new File(".")` → current working directory (project root)
* `getClassLoader()` → loads test classes and resources
* `Mode.TEST` → configures app to behave like a test environment

---

### ❓5. Why must `@BeforeClass` be static in JUnit 4?

✅ **Answer:**
JUnit creates a **new test class object for each test method**.
At the time `@BeforeClass` runs, no test object exists — so it must be **static**.

📌 Summary:

| Annotation     | Static? | Runs When?              | Why?                         |
| -------------- | ------- | ----------------------- | ---------------------------- |
| `@BeforeClass` | ✅ Yes   | Before all test methods | No object created yet        |
| `@Before`      | ❌ No    | Before each test        | New object created each time |

---

### ❓6. Is this integration testing?

✅ **Answer:** Yes. You're running integration tests because:

* You're starting the app (`Helpers.start`)
* You insert real test data into DB
* You use actual service classes (not mocks)
* You verify full end-to-end logic

---

### ❓7. Why use mocks?

✅ **Answer:**
Mocks are used in **unit testing** to:

* Avoid slow or external dependencies (e.g., real DB, APIs)
* Focus on testing a single class
* Control behavior of collaborators

🧱 Example:

```java
@Mock PaymentService paymentService;

@Before
public void setUp() {
    MockitoAnnotations.openMocks(this);
    when(paymentService.charge(anyString(), anyInt())).thenReturn(true);
}
```

---

### ❓8. What does `verifyResult()` method do?

✅ **Answer:**
This is a **template-based output verification** approach.

* Converts actual object (e.g., DTO) to JSON
* Compares it against a pre-saved expected JSON file
* If the file is missing, it writes the actual as the initial template

🧱 Example:

```java
boolean res = verifyResult(identityDTO, "/expected/output.json");
```

---

### ✅ Summary Table

| Concept                   | Purpose/Explanation                                                    |
| ------------------------- | ---------------------------------------------------------------------- |
| `GuiceApplicationBuilder` | Build custom Play app instance for tests                               |
| `Helpers.start(app)`      | Start the built app so tests can interact with it                      |
| `Environment(...)`        | Defines app context: root path, class loader, mode                     |
| `@BeforeClass`            | Must be static; runs once before test class methods                    |
| **Integration Test**      | Tests app behavior across modules using real services and DB           |
| **Mocks**                 | Replace real services in unit tests for speed and isolation            |
| `verifyResult(...)`       | Verifies actual output (converted to JSON) matches saved template JSON |

---

📌 **Note:** These understandings can evolve as the framework or your use cases grow.

