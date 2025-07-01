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

