# Self-typed generic interfaces

Read when the user shows inclination for this pattern. The general Java rules are in `java.md`.

- **Self-Typed Generic Interfaces Pattern**

	- **Self-Typed Generic Interfaces**: Use self-typed generics to achieve type-safe method chaining. Pattern: `public interface MyInterface<T extends MyInterface<T>>` implemented as `public class MyClass implements MyInterface<MyClass>`.

	- **When to Use**: Apply this pattern when (1) you have more than 1 implementation of an interface AND (2) implementations are datatype/pojo/builder-pattern classes (not service/feature classes). This enables fluent API with proper return types.

	- the most mature implementation of this is luvml, see `../../luvml/luvml/luvml_tutorial.md`

	- **Example Pattern**:
	```java
	// Interface with self-typed generic
	public interface BlockXml<B extends BlockXml<B>> {
		String bid();
		B bid(String bid);
	}
	
	// Implementation with Lombok
	@Data
	@Accessors(fluent = true)
	public class ParagraphXml implements BlockXml<ParagraphXml> {
		String bid; // notice this is package-private not private.
		// Lombok generates: public String bid() and public ParagraphXml bid(String bid)
		// NO manual implementation needed!
	}
	```
	- **Naming Convention Pattern - Implementation Priority**: For class/interface pairs, use the pattern `MyClass` (implementation) and `MyClass_I` (interface). This creates clear one-to-one mapping where the interface name is the class name plus `_I` suffix. Examples: `Block` class implements `Block_I`, `Annotation` class implements `Annotation_I`.
