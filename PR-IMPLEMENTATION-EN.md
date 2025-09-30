# AssertJ Protobuf Module Implementation

## Overview

This PR adds Protocol Buffers message assertion support to AssertJ. It enables Java developers working with gRPC and Protobuf to write tests using AssertJ's fluent API style.

**Related Issue**: #3736

## Implemented Features

### 1. Core Module: `assertj-protobuf`

A new Maven module providing:

#### Main Classes

**`org.assertj.protobuf.api.Assertions`**
- Entry point for AssertJ Protobuf
- Provides `assertThat(Message)` static method

**`org.assertj.protobuf.api.MessageAssert<T extends Message>`**
- Main assertion class for Protobuf messages
- Extends `AbstractAssert` following AssertJ standards

**`org.assertj.protobuf.internal.MessageDifferencer`**
- Internal message comparison logic
- Supports repeated fields, nested messages, field exclusion, etc.

#### Provided Methods

```java
// Basic comparison
assertThat(actual).isEqualTo(expected);

// Ignore repeated field order
assertThat(actual).ignoringRepeatedFieldOrder().isEqualTo(expected);

// Exclude specific fields
assertThat(actual).ignoringFields("timestamp").isEqualTo(expected);

// Partial comparison (only expected fields)
assertThat(actual).comparingExpectedFieldsOnly().isEqualTo(expected);

// Field presence checks
assertThat(actual).hasField("name");
assertThat(actual).doesNotHaveField("deletedAt");
```

### 2. Integration Test Module: `assertj-protobuf-tests`

#### Test Structure
- `src/test/proto/test.proto`: Test Protobuf definitions
- 4 test classes with 15 test cases
- All tests passing ✅

#### Test Coverage
- `MessageAssert_isEqualTo_Test`: 5 tests
- `MessageAssert_ignoringRepeatedFieldOrder_Test`: 5 tests
- `MessageAssert_ignoringFields_Test`: 2 tests
- `MessageAssert_hasField_Test`: 3 tests

## Technical Design

### Dependency Management
```xml
<!-- Compile -->
<dependency>
  <groupId>org.assertj</groupId>
  <artifactId>assertj-core</artifactId>
  <version>${project.version}</version>
</dependency>

<!-- Provided - users choose version -->
<dependency>
  <groupId>com.google.protobuf</groupId>
  <artifactId>protobuf-java</artifactId>
  <version>4.29.3</version>
  <scope>provided</scope>
</dependency>
```

**Key Decisions**:
- `protobuf-java` with `provided` scope allows users to choose their version
- **No Guava dependency**: Unlike Curiostack implementation, follows minimal dependency principle
- Java Module System support (`module-info.java`)
- OSGi bundle support (bnd-maven-plugin)

### Package Structure
```
org.assertj.protobuf/
├── api/              # Public API (backward compatibility guaranteed)
│   ├── Assertions
│   └── MessageAssert
└── internal/         # Internal implementation (excluded from japicmp)
    └── MessageDifferencer
```

### Core Implementation Logic

#### Repeated Field Handling
Protobuf's repeated fields don't support `hasField()`, requiring special handling:

```java
if (field.isRepeated()) {
  // Use getField() instead of hasField()
  actualValue = actual.getField(field);
  expectedValue = expected.getField(field);
  
  // Treat empty lists as null
  List<Object> actualList = (List<Object>) actualValue;
  if (actualList.isEmpty()) actualValue = null;
}
```

#### AS_SET Mode (Order-Insensitive)
Algorithm for comparing repeated fields like sets:

1. For each item in expected list
2. Find matching item in actual list
3. Remove matched item from actual list (prevent duplicates)
4. Success if all items matched

#### Recursive Message Comparison
Nested messages are compared recursively, tracking field paths:
- `"field_name"` → `"nested.field_name"` → `"nested.child.field_name"`

## Usage Examples

### Basic Usage
```java
import static org.assertj.protobuf.api.Assertions.assertThat;

@Test
void should_compare_protobuf_messages() {
  // GIVEN
  UserProto expected = UserProto.newBuilder()
      .setName("John")
      .setAge(30)
      .build();
  
  UserProto actual = userService.getUser(userId);
  
  // WHEN/THEN
  assertThat(actual).isEqualTo(expected);
}
```

### Advanced Patterns
```java
@Test
void should_ignore_timestamp_and_repeated_order() {
  // GIVEN
  OrderProto expected = OrderProto.newBuilder()
      .setOrderId("ORD-123")
      .addItems("ITEM-1")
      .addItems("ITEM-2")
      .build();
  
  OrderProto actual = orderService.getOrder("ORD-123");
  
  // WHEN/THEN
  assertThat(actual)
      .ignoringFields("createdAt", "updatedAt")
      .ignoringRepeatedFieldOrder()
      .isEqualTo(expected);
}
```

## Build & Test

### Build
```bash
# Build protobuf module only
./mvnw clean install -pl assertj-protobuf

# Include tests
./mvnw clean install -pl assertj-protobuf,assertj-tests/assertj-integration-tests/assertj-protobuf-tests
```

### Run Tests
```bash
./mvnw test -pl assertj-tests/assertj-integration-tests/assertj-protobuf-tests
```

### Test Results
```
[INFO] Tests run: 15, Failures: 0, Errors: 0, Skipped: 0
```

## Checklist

### Code Quality
- [x] Spotless formatting applied
- [x] Apache 2.0 license headers added
- [x] Javadoc written for all public methods
- [x] Success/failure examples included

### Testing
- [x] JUnit 5 used
- [x] GIVEN/WHEN/THEN structure
- [x] Naming convention followed (`should_pass_xxx`, `should_fail_xxx`)
- [x] Package-private visibility
- [x] All tests passing (15/15)

### Dependencies
- [x] Minimal dependency principle followed
- [x] Provided scope utilized
- [x] No Guava dependency

### Compatibility
- [x] Java Module System support
- [x] OSGi bundle support
- [x] japicmp configuration (internal package excluded)

## Future Work

The following features can be added in separate PRs:

1. **Additional Assertion Methods**
   - `extracting()` - Field extraction
   - `ignoringFieldsMatchingRegex()` - Regex-based field exclusion

2. **Special Protobuf Type Support**
   - `google.protobuf.Any`
   - `google.protobuf.Timestamp`
   - `google.protobuf.Struct`

3. **Error Message Improvements**
   - More detailed difference display
   - Color output support

## References

- **Issue**: #3736
- **Google Truth Protobuf**: https://github.com/google/truth/tree/master/extensions/proto
- **Curiostack assertj-proto**: https://github.com/curioswitch/curiostack/tree/master/tools/assertj-proto (MIT License)

## Attribution

This implementation was inspired by:
- Google Truth's Protobuf extension (Apache 2.0)
- Curiostack's assertj-proto (MIT License)

All code has been written from scratch following AssertJ's coding style and principles.

---

**Author**: JongJun Kim  
**Closes**: #3736
