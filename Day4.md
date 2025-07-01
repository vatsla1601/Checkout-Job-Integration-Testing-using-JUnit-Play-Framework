⚙️ How @Builder Works Internally
Lombok’s @Builder does not use the class’s constructor or field initializers.
Instead, it generates a separate static inner class — the Builder — and sets fields manually based on what you call during builder().xxx(...).

❌ Why This Causes Your Field Initializers to be Ignored
When you do this:

java
Copy
Edit
VisitorDTO dto = VisitorDTO.builder().build();
The builder calls:

java
Copy
Edit
return new VisitorDTO(this.customFields, this.visitorAccess, ...);
But if you didn’t explicitly set customFields via builder().customFields(...), then:

this.customFields == null

So VisitorDTO.customFields = null — the field initializer (new HashMap<>()) is ignored

✅ Solution: Use @Builder.Default
Lombok will only apply your initializer expression if you annotate it like this:

java
Copy
Edit
@Builder.Default
private Map<String, String> customFields = new HashMap<>();
This tells Lombok:

“Hey! If the user doesn't set this value in the builder, use this default.”

🔁 In Summary
Behavior	With @Builder only	With @Builder.Default
Field initializer used?	❌ No	✅ Yes
Field gets default value?	❌ No (set to null)	✅ Yes (your initializer runs)

❓ If I mark fields as @Mapping(..., ignore = true) in MapStruct, do I still need @Builder.Default in Lombok?
✅ Yes — but only if you use the builder in your test code or anywhere else.

✅ Why?
MapStruct and Lombok serve different purposes:

Tool	Purpose
MapStruct	Maps one object to another (e.g., Entity → DTO)
Lombok	Generates constructors, getters, setters, and the Builder pattern

So when you do:

java
Copy
Edit
@Mapping(target = "visitorAccess", ignore = true)
MapStruct says:

“I will skip mapping this field when I create a new DTO.”

But Lombok says:

“When someone uses VisitorDTO.builder().build(), I need to know whether to set a default or leave it null.”

✅ You Need @Builder.Default If:
You are using .builder() to create test data or actual DTOs anywhere.

You want fields like visitorAccess to be auto-initialized (instead of null).

❌ You Can Skip @Builder.Default If:
You never use .builder() to build VisitorDTO instances.

You always set visitorAccess manually or through setters after object creation.

✅ 2. Where and how to add Lombok in the project?
Added this to libraryDependencies in the correct module (alertserver, because VisitorDTO is defined there):

scala
Copy
Edit
"org.projectlombok" % "lombok" % "1.18.30" % Provided
Also, enabled annotation processing in IntelliJ:
File → Settings → Build, Execution, Deployment → Compiler → Annotation Processors → Enable

✅ These are MapStruct errors, not Builder (@Builder) errors.

Let’s break them down so you understand each part clearly:

✅ 1. Builder Warnings (Not Errors)
text
Copy
Edit
[warn] @Builder will ignore the initializing expression entirely. 
If you want the initializing expression to serve as default, add @Builder.Default.
This is a Lombok @Builder warning about this kind of code:

java
Copy
Edit
private Set<VisitorSystemDTO> visitorSystems = new HashSet<>();
🔧 Fix it like this:

java
Copy
Edit
@Builder.Default
private Set<VisitorSystemDTO> visitorSystems = new HashSet<>();
❌ 2. MapStruct Errors
These are real compile-time errors caused by MapStruct:

text
Copy
Edit
[error] Unknown property "id" in result type com.alnt.visitormgmt.domain.dto.VisitorDTO
[error] Unknown property "id_str" in result type com.alnt.visitormgmt.domain.dto.VisitorDTO
They happen because in your mapper:

java
Copy
Edit
@Mapper(config = MapperCentralConfig.class)
public interface VisitorMapper extends BaseMapper<VisitorDTO, Visitor> {

    @Override
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "id_str", ignore = true)
    @Mapping(target = "visitorAccess", ignore = true)
    @Mapping(target = "visitorAssets", ignore = true)
    @Mapping(target = "visitorSystems", ignore = true)
    VisitorDTO createCopyFrom(VisitorDTO source);
}
MapStruct is trying to ignore "id" and "id_str" but cannot find them in VisitorDTO.

✅ 5. Safe workaround: Just remove this line if you're ignoring it anyway
java
Copy
Edit
// Remove this
@Mapping(target = "id_str", ignore = true)
If you're not using or mapping it, and just ignoring it — removing the line entirely is safest.

If you add @SuperBuilder (from Lombok) to all parent classes (e.g., BaseMasterDTO, PersonDTO, and then VisitorDTO), here’s exactly what will happen:

✅ What @SuperBuilder Does
@SuperBuilder is Lombok's solution for builder pattern with inheritance.

Unlike @Builder, which only supports flat classes, @SuperBuilder:

Generates builder classes that support inherited fields

Ensures you can build a child class while also setting values from the parent

Supports method chaining across parent and child builders

🧱 Example
📦 If you do this:
java
Copy
Edit
@SuperBuilder
public class BaseMasterDTO {
    private Long id;
}

@SuperBuilder
public class PersonDTO extends BaseMasterDTO {
    private String name;
}

@SuperBuilder
public class VisitorDTO extends PersonDTO {
    private String masterId;
}
✅ You can then write:
java
Copy
Edit
VisitorDTO visitor = VisitorDTO.builder()
    .id(123L)                // inherited from BaseMasterDTO
    .name("John Doe")        // from PersonDTO
    .masterId("vatsla.adhikari")  // from VisitorDTO
    .build();
So now all fields across the hierarchy are included in the builder.

⚙️ What Happens With Lombok’s @EqualsAndHashCode (Without callSuper = true)
When you write:

java
Copy
Edit
@Data
@EqualsAndHashCode // (implicitly callSuper = false)
public class VisitorDTO extends BaseDTO {
    private String name;
}
Lombok generates equals() and hashCode() only using the fields defined in VisitorDTO, which is:

java
Copy
Edit
private String name;
It ignores inherited fields like id from BaseDTO, because callSuper = false by default.

🔍 So yes:
It checks name ✅

It does not check id ❌ (because id is in the superclass)

🧪 Concrete Example
👨‍👧 Classes:
java
Copy
Edit
public class BaseDTO {
    private String id;
    // Getters and setters...
}

@Data
@EqualsAndHashCode // 👈 callSuper = false
public class VisitorDTO extends BaseDTO {
    private String name;
}
🧪 Test code:
java
Copy
Edit
VisitorDTO v1 = new VisitorDTO();
v1.setId("123");     // from BaseDTO
v1.setName("Alice");

VisitorDTO v2 = new VisitorDTO();
v2.setId("456");     // different ID!
v2.setName("Alice"); // same name
🧾 Result:
java
Copy
Edit
System.out.println(v1.equals(v2)); // ✅ true!
This may surprise you, but it’s because:

equals() only checks name, not id

Both v1 and v2 have name = "Alice"

So they’re considered equal, even though their ids are different — which could be a logic bug.

✅ Fix: Add callSuper = true
java
Copy
Edit
@EqualsAndHashCode(callSuper = true)
Now equals() will also call super.equals(), which (if overridden properly or inherited from Lombok-generated code) includes id, making:

java
Copy
Edit
System.out.println(v1.equals(v2)); // ❌ false
Which is correct, since their ids are different.

💡 What are equals() and hashCode()?
In Java:

equals() checks if two objects are "logically equal".

hashCode() returns an integer used by hash-based collections (like HashMap, HashSet) to organize objects efficiently.

Together, they define how objects are compared and stored.

📌 Real-Life Analogy
Imagine two visitor badges:

java
Copy
Edit
VisitorDTO v1 = new VisitorDTO("123", "Alice");
VisitorDTO v2 = new VisitorDTO("123", "Alice");
You want Java to say:

java
Copy
Edit
v1.equals(v2) => true
Because they represent the same logical visitor — even if they're different objects in memory.

✅ equals() – Checks content equality
Example:

java
Copy
Edit
VisitorDTO v1 = new VisitorDTO("123", "Alice");
VisitorDTO v2 = new VisitorDTO("123", "Alice");

System.out.println(v1.equals(v2)); // true
But without overriding equals(), it will default to memory address comparison — meaning:

java
Copy
Edit
v1 == v2 => false (always, unless same object)
Overriding equals() lets Java check field values instead.

✅ hashCode() – Used in collections
java
Copy
Edit
HashSet<VisitorDTO> set = new HashSet<>();
set.add(v1);
set.add(v2); // This won't be added if equals() and hashCode() work properly

System.out.println(set.size()); // Should be 1 if v1.equals(v2) == true
Without correct hashCode() override, Java thinks they're different.

❌ Problem if you override only one
Java’s contract:

If a.equals(b) is true, then a.hashCode() == b.hashCode() must also be true.

If you override equals() but not hashCode(), weird bugs happen in collections like HashMap, HashSet.

✅ With Lombok
If you use:

java
Copy
Edit
@Data
@EqualsAndHashCode(callSuper = true)
Lombok generates equals() and hashCode() based on all fields — including parent fields.

🧪 Example: BaseDTO + VisitorDTO
java
Copy
Edit
public class BaseDTO {
    private String id;
}

@Data
@EqualsAndHashCode(callSuper = true)
public class VisitorDTO extends BaseDTO {
    private String name;
}
Now:

java
Copy
Edit
VisitorDTO v1 = new VisitorDTO();
v1.setId("123");
v1.setName("Alice");

VisitorDTO v2 = new VisitorDTO();
v2.setId("123");
v2.setName("Alice");

System.out.println(v1.equals(v2));  // ✅ true
System.out.println(v1.hashCode() == v2.hashCode());  // ✅ true



