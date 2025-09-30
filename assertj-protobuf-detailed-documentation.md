# AssertJ Protobuf Module 구현 상세 문서

## 📋 목차
1. [프로젝트 개요](#프로젝트-개요)
2. [구현 배경](#구현-배경)
3. [아키텍처 설계](#아키텍처-설계)
4. [구현 상세](#구현-상세)
5. [테스트 전략](#테스트-전략)
6. [기술적 의사결정](#기술적-의사결정)
7. [사용 가이드](#사용-가이드)
8. [향후 계획](#향후-계획)

---

## 프로젝트 개요

### 목적
Java 개발자들이 Protocol Buffers (Protobuf) 메시지를 테스트할 때 AssertJ의 유창한(fluent) API 스타일로 assertion을 작성할 수 있도록 지원하는 모듈을 구현합니다.

### 관련 이슈
- GitHub Issue: [#3736](https://github.com/assertj/assertj/issues/3736)
- Milestone: 4.0.0-M3
- Label: `status: ideal for contribution`

### 주요 이해관계자
- **제안자**: alanktwong
- **AssertJ 팀**: joel-costigliola, scordio
- **구현자**: JongJun Kim
- **수혜자**: gRPC/Protobuf 사용 Java 개발자 커뮤니티

---

## 구현 배경

### 문제 정의

#### 현재 상황
1. **Google Truth의 Protobuf 지원**: Google은 Truth assertion 라이브러리에 Protobuf 확장을 제공
2. **AssertJ의 부재**: AssertJ는 Guava, Joda Time 등 다양한 라이브러리를 지원하지만 Protobuf는 없음
3. **Curiostack의 시도**: Curiostack이 Google Truth의 Protobuf 기능을 AssertJ로 포팅했으나 현재 관리되지 않음 (abandoned)

#### 해결해야 할 문제
- Protobuf 메시지의 동등성 비교 시 일반 `equals()`로는 부족
- Repeated 필드의 순서 차이 처리 필요
- 부분 비교(특정 필드만) 기능 필요
- 타임스탬프 등 특정 필드 제외 기능 필요

### 요구사항 분석

#### 기능 요구사항
1. **기본 비교**: Protobuf 메시지 간 동등성 비교
2. **유연한 비교**: 
   - Repeated 필드 순서 무시
   - 특정 필드 제외
   - 부분 메시지 비교
3. **필드 검증**: 필드 존재 여부 확인

#### 비기능 요구사항
1. **최소 의존성**: AssertJ 원칙에 따라 외부 의존성 최소화
2. **성능**: 대형 메시지도 효율적으로 비교
3. **확장성**: 향후 기능 추가가 용이한 구조
4. **호환성**: Java Module System, OSGi 지원

---

## 아키텍처 설계

### 전체 구조

```
assertj/
├── assertj-protobuf/              # 핵심 모듈
│   ├── pom.xml                    # Maven 설정
│   ├── verify.bndrun              # OSGi 검증
│   └── src/main/java/
│       ├── module-info.java       # Java 모듈 정의
│       └── org/assertj/protobuf/
│           ├── api/               # 공개 API
│           │   ├── Assertions.java
│           │   └── MessageAssert.java
│           └── internal/          # 내부 구현
│               └── MessageDifferencer.java
│
└── assertj-tests/assertj-integration-tests/
    └── assertj-protobuf-tests/    # 통합 테스트
        ├── pom.xml
        └── src/test/
            ├── proto/             # 테스트용 .proto
            │   └── test.proto
            └── java/org/assertj/tests/protobuf/
                ├── api/           # API 테스트
                └── testkit/       # 테스트 유틸리티
```

### 클래스 다이어그램

```
┌─────────────────────────┐
│   Assertions            │
│  (Entry Point)          │
└───────┬─────────────────┘
        │ creates
        ▼
┌─────────────────────────────────────┐
│   MessageAssert<T extends Message>  │
│   extends AbstractAssert            │
├─────────────────────────────────────┤
│ - differencer: MessageDifferencer   │
├─────────────────────────────────────┤
│ + isEqualTo(Object)                 │
│ + ignoringRepeatedFieldOrder()      │
│ + ignoringFields(String...)         │
│ + comparingExpectedFieldsOnly()     │
│ + hasField(String)                  │
│ + doesNotHaveField(String)          │
└───────┬─────────────────────────────┘
        │ uses
        ▼
┌─────────────────────────────────────┐
│   MessageDifferencer                │
│   (Internal)                        │
├─────────────────────────────────────┤
│ - repeatedFieldComparison           │
│ - scope                             │
│ - ignoredFields: Set<String>        │
├─────────────────────────────────────┤
│ + compare(Message, Message): bool   │
│ + setRepeatedFieldComparison(...)   │
│ + setScope(Scope)                   │
│ + ignoreField(String)               │
└─────────────────────────────────────┘
```

### 패키지 구조 설계 원칙

#### 1. **api 패키지** (`org.assertj.protobuf.api`)
- **목적**: 공개 API 제공
- **안정성**: 하위 호환성 보장
- **내용**:
  - `Assertions`: 진입점
  - `MessageAssert`: 주요 assertion 클래스
  - (향후) `InstanceOfAssertFactories`: 타입 안전 변환

#### 2. **internal 패키지** (`org.assertj.protobuf.internal`)
- **목적**: 내부 구현 로직
- **안정성**: 변경 가능, japicmp에서 제외
- **내용**:
  - `MessageDifferencer`: 메시지 비교 로직
  - (향후) 기타 유틸리티 클래스

#### 3. **error 패키지** (향후 추가 예정)
- **목적**: 에러 메시지 생성
- **예**: `MessageShouldBeEqual`, `MessageShouldHaveField`

---

## 구현 상세

### 1. Maven 프로젝트 설정

#### assertj-protobuf/pom.xml

```xml
<parent>
  <groupId>org.assertj</groupId>
  <artifactId>assertj-parent</artifactId>
  <version>4.0.0-SNAPSHOT</version>
</parent>

<artifactId>assertj-protobuf</artifactId>
<name>AssertJ Protobuf</name>

<dependencies>
  <!-- Compile -->
  <dependency>
    <groupId>org.assertj</groupId>
    <artifactId>assertj-core</artifactId>
    <version>${project.version}</version>
  </dependency>
  
  <!-- Provided - 사용자가 버전 선택 -->
  <dependency>
    <groupId>com.google.protobuf</groupId>
    <artifactId>protobuf-java</artifactId>
    <version>4.29.3</version>
    <scope>provided</scope>
  </dependency>
</dependencies>
```

**핵심 설계 결정**:
- `protobuf-java`를 `provided` scope로 설정
  - 이유: 사용자가 프로젝트에 맞는 Protobuf 버전 선택 가능
  - 장점: 버전 충돌 방지
- Guava 의존성 제거
  - Curiostack 구현과의 차별점
  - AssertJ 원칙: 최소 의존성

### 2. 핵심 클래스 구현

#### 2.1 Assertions.java - 진입점

```java
package org.assertj.protobuf.api;

import com.google.protobuf.Message;

/**
 * Entry point for assertions for Protocol Buffers.
 */
public class Assertions {

  /**
   * Creates a new instance of {@link MessageAssert}.
   *
   * @param <T> the type of the actual value
   * @param actual the actual Protobuf message
   * @return the created assertion object
   */
  public static <T extends Message> MessageAssert<T> assertThat(T actual) {
    return new MessageAssert<>(actual);
  }

  protected Assertions() {
    // 직접 인스턴스화 방지
  }
}
```

**설계 포인트**:
- Static factory method 패턴 사용
- Generic 타입으로 타입 안전성 보장
- Protected 생성자로 상속은 허용하되 직접 생성은 방지

#### 2.2 MessageAssert.java - 주요 Assertion 클래스

##### 기본 구조
```java
public class MessageAssert<T extends Message> 
    extends AbstractAssert<MessageAssert<T>, T> {

  private final MessageDifferencer differencer;

  public MessageAssert(T actual) {
    super(actual, MessageAssert.class);
    this.differencer = new MessageDifferencer();
  }
  
  // assertion 메서드들...
}
```

**설계 포인트**:
- `AbstractAssert` 상속으로 AssertJ 표준 메서드 활용
- `MessageDifferencer` 인스턴스를 유지하여 설정 보존
- Fluent API를 위해 `this` 반환

##### isEqualTo() 구현

```java
@Override
public MessageAssert<T> isEqualTo(Object expected) {
  if (actual == expected) return this;

  if (expected == null) {
    failWithMessage("Expected message to be null but was:<%s>", actual);
  }

  if (!(expected instanceof Message)) {
    failWithMessage("Expected a Protobuf Message but was:<%s>", 
                    expected.getClass());
  }

  Message expectedMessage = (Message) expected;

  if (!differencer.compare(actual, expectedMessage)) {
    String differences = differencer.getLastDifferences();
    failWithMessage("Protobuf messages are not equal:%n%s", differences);
  }

  return this;
}
```

**구현 세부사항**:
1. **동일성 체크**: `actual == expected`로 빠른 반환
2. **타입 검증**: `instanceof Message` 체크
3. **비교 위임**: `MessageDifferencer`에 실제 비교 로직 위임
4. **상세 에러**: 차이점을 포함한 명확한 실패 메시지

##### ignoringRepeatedFieldOrder() 구현

```java
public MessageAssert<T> ignoringRepeatedFieldOrder() {
  differencer.setRepeatedFieldComparison(
      MessageDifferencer.RepeatedFieldComparison.AS_SET);
  return this;
}
```

**설계 포인트**:
- Fluent API 체인을 위해 `this` 반환
- 내부 상태 변경만 수행, 실제 비교는 `isEqualTo()`에서 발생

##### ignoringFields() 구현

```java
public MessageAssert<T> ignoringFields(String... fieldPaths) {
  for (String fieldPath : fieldPaths) {
    differencer.ignoreField(fieldPath);
  }
  return this;
}
```

**설계 포인트**:
- Varargs로 여러 필드 한 번에 지정 가능
- 필드 경로 형식: `"field_name"`, `"nested.field"`

##### hasField() / doesNotHaveField() 구현

```java
public MessageAssert<T> hasField(String fieldName) {
  isNotNull();
  
  FieldDescriptor field = 
      actual.getDescriptorForType().findFieldByName(fieldName);
  
  if (field == null) {
    failWithMessage("Field <%s> does not exist in message type <%s>", 
                    fieldName, 
                    actual.getDescriptorForType().getFullName());
  }
  
  if (!actual.hasField(field)) {
    failWithMessage("Expected message to have field <%s> but it was not set", 
                    fieldName);
  }
  
  return this;
}
```

**구현 세부사항**:
1. **Null 체크**: `isNotNull()` 먼저 호출
2. **필드 존재 확인**: Descriptor에서 필드 조회
3. **필드 설정 확인**: `hasField()` 호출
4. **명확한 에러**: 필드가 없는지, 설정 안 된 것인지 구분

#### 2.3 MessageDifferencer.java - 비교 로직

##### 열거형 정의

```java
public enum RepeatedFieldComparison {
  AS_LIST,   // 순서 포함 비교 (기본값)
  AS_SET     // 순서 무시 비교
}

public enum Scope {
  FULL,      // 모든 필드 비교 (기본값)
  PARTIAL    // 기대값의 필드만 비교
}
```

##### 상태 관리

```java
private RepeatedFieldComparison repeatedFieldComparison = 
    RepeatedFieldComparison.AS_LIST;
private Scope scope = Scope.FULL;
private final Set<String> ignoredFields = new HashSet<>();
private String lastDifferences = "";
```

##### 주요 메서드: compare()

```java
public boolean compare(Message actual, Message expected) {
  if (actual == expected) return true;
  if (actual == null || expected == null) return false;
  
  // 타입 검증
  if (!actual.getDescriptorForType()
             .equals(expected.getDescriptorForType())) {
    lastDifferences = String.format(
        "Message types differ: <%s> vs <%s>",
        actual.getDescriptorForType().getFullName(),
        expected.getDescriptorForType().getFullName());
    return false;
  }

  StringBuilder differences = new StringBuilder();
  boolean isEqual = compareMessages(actual, expected, "", differences);
  
  if (!isEqual) {
    lastDifferences = differences.toString();
  }
  
  return isEqual;
}
```

##### 재귀적 메시지 비교: compareMessages()

```java
private boolean compareMessages(Message actual, Message expected, 
                               String path, StringBuilder differences) {
  boolean isEqual = true;

  // 비교할 필드 결정
  Set<FieldDescriptor> fieldsToCompare = new HashSet<>();
  
  if (scope == Scope.FULL) {
    // 양쪽 메시지의 모든 필드
    fieldsToCompare.addAll(actual.getAllFields().keySet());
    fieldsToCompare.addAll(expected.getAllFields().keySet());
  } else {
    // 기대값의 필드만
    fieldsToCompare.addAll(expected.getAllFields().keySet());
  }

  for (FieldDescriptor field : fieldsToCompare) {
    String fieldPath = path.isEmpty() ? 
        field.getName() : path + "." + field.getName();
    
    if (ignoredFields.contains(fieldPath)) {
      continue;
    }

    // 필드 값 가져오기
    Object actualValue;
    Object expectedValue;

    if (field.isRepeated()) {
      // Repeated 필드 특수 처리
      actualValue = actual.getField(field);
      expectedValue = expected.getField(field);
      
      @SuppressWarnings("unchecked")
      List<Object> actualList = (List<Object>) actualValue;
      @SuppressWarnings("unchecked")
      List<Object> expectedList = (List<Object>) expectedValue;
      
      // 빈 리스트는 null로 취급
      if (actualList.isEmpty()) actualValue = null;
      if (expectedList.isEmpty()) expectedValue = null;
    } else {
      // 일반 필드
      actualValue = actual.hasField(field) ? 
          actual.getField(field) : null;
      expectedValue = expected.hasField(field) ? 
          expected.getField(field) : null;
    }

    // Null 처리
    if (actualValue == null && expectedValue == null) {
      continue;
    }

    if (actualValue == null || expectedValue == null) {
      differences.append(String.format(
          "Field <%s>: expected <%s> but was <%s>%n",
          fieldPath, expectedValue, actualValue));
      isEqual = false;
      continue;
    }

    // 타입별 비교
    if (field.isRepeated()) {
      if (!compareRepeatedField(actualValue, expectedValue, 
                               field, fieldPath, differences)) {
        isEqual = false;
      }
    } else if (field.getJavaType() == 
               FieldDescriptor.JavaType.MESSAGE) {
      // 중첩 메시지 재귀 비교
      if (!compareMessages((Message) actualValue, 
                          (Message) expectedValue, 
                          fieldPath, differences)) {
        isEqual = false;
      }
    } else {
      // 기본 타입 비교
      if (!actualValue.equals(expectedValue)) {
        differences.append(String.format(
            "Field <%s>: expected <%s> but was <%s>%n",
            fieldPath, expectedValue, actualValue));
        isEqual = false;
      }
    }
  }

  return isEqual;
}
```

**핵심 로직 설명**:

1. **필드 수집**: Scope에 따라 비교할 필드 결정
2. **Repeated 필드 처리**: `hasField()` 사용 불가 → `getField()` 직접 사용
3. **재귀 비교**: 중첩 메시지는 재귀적으로 비교
4. **경로 추적**: 필드 경로를 문자열로 추적 (`parent.child.field`)

##### Repeated 필드 비교: compareRepeatedField()

###### AS_LIST 모드 (순서 포함)

```java
private boolean compareRepeatedFieldAsList(
    List<Object> actualList, List<Object> expectedList,
    FieldDescriptor field, String fieldPath,
    StringBuilder differences) {
  
  if (actualList.size() != expectedList.size()) {
    differences.append(String.format(
        "Repeated field <%s>: size differs, expected %d but was %d%n",
        fieldPath, expectedList.size(), actualList.size()));
    return false;
  }

  boolean isEqual = true;
  for (int i = 0; i < actualList.size(); i++) {
    Object actualItem = actualList.get(i);
    Object expectedItem = expectedList.get(i);

    if (field.getJavaType() == FieldDescriptor.JavaType.MESSAGE) {
      if (!compareMessages((Message) actualItem, 
                          (Message) expectedItem,
                          fieldPath + "[" + i + "]", 
                          differences)) {
        isEqual = false;
      }
    } else {
      if (!actualItem.equals(expectedItem)) {
        differences.append(String.format(
            "Repeated field <%s>[%d]: expected <%s> but was <%s>%n",
            fieldPath, i, expectedItem, actualItem));
        isEqual = false;
      }
    }
  }

  return isEqual;
}
```

###### AS_SET 모드 (순서 무시)

```java
private boolean compareRepeatedFieldAsSet(
    List<Object> actualList, List<Object> expectedList,
    FieldDescriptor field, String fieldPath,
    StringBuilder differences) {
  
  if (actualList.size() != expectedList.size()) {
    differences.append(String.format(
        "Repeated field <%s>: size differs, expected %d but was %d%n",
        fieldPath, expectedList.size(), actualList.size()));
    return false;
  }

  // 복사본 생성 (원본 수정 방지)
  List<Object> actualCopy = new ArrayList<>(actualList);
  List<Object> expectedCopy = new ArrayList<>(expectedList);

  for (Object expectedItem : expectedCopy) {
    boolean found = false;
    
    for (int i = 0; i < actualCopy.size(); i++) {
      Object actualItem = actualCopy.get(i);
      
      boolean matches;
      if (field.getJavaType() == FieldDescriptor.JavaType.MESSAGE) {
        // 메시지는 재귀 비교
        StringBuilder tempDiff = new StringBuilder();
        matches = compareMessages((Message) actualItem, 
                                 (Message) expectedItem, 
                                 fieldPath, tempDiff);
      } else {
        // 기본 타입은 equals
        matches = actualItem.equals(expectedItem);
      }

      if (matches) {
        actualCopy.remove(i);  // 매칭된 항목 제거
        found = true;
        break;
      }
    }

    if (!found) {
      differences.append(String.format(
          "Repeated field <%s>: expected to contain <%s> but was not found%n",
          fieldPath, expectedItem));
      return false;
    }
  }

  return true;
}
```

**알고리즘 설명**:
1. 기대값의 각 항목에 대해
2. 실제값 리스트에서 매칭되는 항목 찾기
3. 찾으면 실제값 리스트에서 제거 (중복 방지)
4. 모든 항목이 매칭되면 성공

---

## 테스트 전략

### 테스트 모듈 구조

```
assertj-protobuf-tests/
├── pom.xml
└── src/test/
    ├── proto/
    │   └── test.proto              # 테스트용 Protobuf 정의
    └── java/org/assertj/tests/protobuf/
        ├── api/
        │   ├── MessageAssert_isEqualTo_Test.java
        │   ├── MessageAssert_ignoringRepeatedFieldOrder_Test.java
        │   ├── MessageAssert_ignoringFields_Test.java
        │   └── MessageAssert_hasField_Test.java
        └── testkit/
            └── AssertionErrors.java # 테스트 유틸리티
```

### 테스트용 Protobuf 정의

```protobuf
syntax = "proto3";

package org.assertj.tests.protobuf;

option java_package = "org.assertj.tests.protobuf";
option java_outer_classname = "TestProtos";

// 기본 테스트 메시지
message TestMessage {
  string foo = 1;
  int32 bar = 2;
  repeated int32 numbers = 3;
  NestedMessage nested = 4;
}

// 중첩 메시지 테스트용
message NestedMessage {
  string value = 1;
  int32 count = 2;
}

// Repeated 필드 전용 테스트
message RepeatedFieldMessage {
  repeated string items = 1;
  repeated int32 values = 2;
}
```

### 테스트 케이스 설계

#### 1. MessageAssert_isEqualTo_Test (5개 테스트)

```java
@Test
void should_pass_when_messages_are_equal() {
  // GIVEN
  TestMessage actual = TestMessage.newBuilder()
                                  .setFoo("bar")
                                  .setBar(42)
                                  .build();
  TestMessage expected = TestMessage.newBuilder()
                                    .setFoo("bar")
                                    .setBar(42)
                                    .build();
  // WHEN/THEN
  assertThat(actual).isEqualTo(expected);
}

@Test
void should_pass_when_comparing_same_instance() {
  // GIVEN
  TestMessage message = TestMessage.newBuilder()
                                   .setFoo("bar")
                                   .build();
  // WHEN/THEN
  assertThat(message).isEqualTo(message);
}

@Test
void should_fail_when_messages_are_different() {
  // GIVEN
  TestMessage actual = TestMessage.newBuilder()
                                  .setFoo("bar")
                                  .build();
  TestMessage expected = TestMessage.newBuilder()
                                    .setFoo("baz")
                                    .build();
  // WHEN
  AssertionError error = expectAssertionError(
      () -> assertThat(actual).isEqualTo(expected));
  // THEN
  then(error).hasMessageContaining("foo")
             .hasMessageContaining("bar")
             .hasMessageContaining("baz");
}
```

**테스트 커버리지**:
- ✅ 동등한 메시지 비교
- ✅ 동일 인스턴스 비교
- ✅ 다른 필드 값
- ✅ 추가 필드 존재
- ✅ 누락된 필드

#### 2. MessageAssert_ignoringRepeatedFieldOrder_Test (5개 테스트)

```java
@Test
void should_pass_when_repeated_fields_match_in_different_order() {
  // GIVEN
  RepeatedFieldMessage actual = 
      RepeatedFieldMessage.newBuilder()
                         .addItems("a")
                         .addItems("b")
                         .addItems("c")
                         .build();
  RepeatedFieldMessage expected = 
      RepeatedFieldMessage.newBuilder()
                         .addItems("c")
                         .addItems("a")
                         .addItems("b")
                         .build();
  // WHEN/THEN
  assertThat(actual).ignoringRepeatedFieldOrder()
                    .isEqualTo(expected);
}

@Test
void should_fail_without_ignoring_order() {
  // GIVEN
  RepeatedFieldMessage actual = 
      RepeatedFieldMessage.newBuilder()
                         .addItems("a")
                         .addItems("b")
                         .build();
  RepeatedFieldMessage expected = 
      RepeatedFieldMessage.newBuilder()
                         .addItems("b")
                         .addItems("a")
                         .build();
  // WHEN
  AssertionError error = expectAssertionError(
      () -> assertThat(actual).isEqualTo(expected));
  // THEN
  then(error).hasMessageContaining("items");
}
```

**테스트 커버리지**:
- ✅ 순서 다른 repeated 필드
- ✅ 다른 값의 repeated 필드
- ✅ 크기 다른 repeated 필드
- ✅ 순서 무시 vs 순서 포함 비교

#### 3. MessageAssert_ignoringFields_Test (2개 테스트)

```java
@Test
void should_pass_when_ignoring_different_field() {
  // GIVEN
  TestMessage actual = TestMessage.newBuilder()
                                  .setFoo("bar")
                                  .setBar(42)
                                  .build();
  TestMessage expected = TestMessage.newBuilder()
                                    .setFoo("bar")
                                    .setBar(99)  // 다른 값
                                    .build();
  // WHEN/THEN
  assertThat(actual).ignoringFields("bar")
                    .isEqualTo(expected);
}

@Test
void should_pass_when_ignoring_multiple_fields() {
  // GIVEN
  TestMessage actual = TestMessage.newBuilder()
                                  .setFoo("bar")
                                  .setBar(42)
                                  .build();
  TestMessage expected = TestMessage.newBuilder()
                                    .setFoo("baz")  // 다른 값
                                    .setBar(99)     // 다른 값
                                    .build();
  // WHEN/THEN
  assertThat(actual).ignoringFields("foo", "bar")
                    .isEqualTo(expected);
}
```

**테스트 커버리지**:
- ✅ 단일 필드 무시
- ✅ 복수 필드 무시

#### 4. MessageAssert_hasField_Test (3개 테스트)

```java
@Test
void should_pass_when_field_is_set() {
  // GIVEN
  TestMessage actual = TestMessage.newBuilder()
                                  .setFoo("bar")
                                  .build();
  // WHEN/THEN
  assertThat(actual).hasField("foo");
}

@Test
void should_fail_when_field_is_not_set() {
  // GIVEN
  TestMessage actual = TestMessage.newBuilder().build();
  // WHEN
  AssertionError error = expectAssertionError(
      () -> assertThat(actual).hasField("foo"));
  // THEN
  then(error).hasMessageContaining("foo")
             .hasMessageContaining("not set");
}

@Test
void should_fail_when_field_does_not_exist() {
  // GIVEN
  TestMessage actual = TestMessage.newBuilder().build();
  // WHEN
  AssertionError error = expectAssertionError(
      () -> assertThat(actual).hasField("nonexistent"));
  // THEN
  then(error).hasMessageContaining("nonexistent")
             .hasMessageContaining("does not exist");
}
```

**테스트 커버리지**:
- ✅ 필드 설정됨
- ✅ 필드 설정 안 됨
- ✅ 필드 존재하지 않음

### 테스트 유틸리티

#### AssertionErrors.java

```java
public class AssertionErrors {

  public static AssertionError expectAssertionError(
      ThrowingCallable throwingCallable) {
    AssertionError error = catchThrowableOfType(
        AssertionError.class, throwingCallable);
    assertThat(error)
        .as("The code under test should have raised an AssertionError")
        .isNotNull();
    return error;
  }
}
```

**목적**:
- AssertJ 내부 유틸리티에 의존하지 않음
- 테스트 모듈에서 assertion 실패 테스트 간소화

### Protobuf 코드 생성 설정

```xml
<build>
  <extensions>
    <extension>
      <groupId>kr.motd.maven</groupId>
      <artifactId>os-maven-plugin</artifactId>
      <version>1.7.1</version>
    </extension>
  </extensions>
  
  <plugins>
    <plugin>
      <groupId>org.xolstice.maven.plugins</groupId>
      <artifactId>protobuf-maven-plugin</artifactId>
      <version>0.6.1</version>
      <configuration>
        <protocArtifact>
          com.google.protobuf:protoc:${protobuf.version}:exe:${os.detected.classifier}
        </protocArtifact>
        <outputDirectory>
          ${project.build.directory}/generated-test-sources/protobuf/java
        </outputDirectory>
      </configuration>
      <executions>
        <execution>
          <goals>
            <goal>test-compile</goal>
          </goals>
        </execution>
      </executions>
    </plugin>
    
    <plugin>
      <groupId>org.codehaus.mojo</groupId>
      <artifactId>build-helper-maven-plugin</artifactId>
      <version>3.6.0</version>
      <executions>
        <execution>
          <id>add-test-source</id>
          <phase>generate-test-sources</phase>
          <goals>
            <goal>add-test-source</goal>
          </goals>
          <configuration>
            <sources>
              <source>
                ${project.build.directory}/generated-test-sources/protobuf/java
              </source>
            </sources>
          </configuration>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

**핵심 설정**:
1. **os-maven-plugin**: OS별 protoc 바이너리 자동 선택
2. **protobuf-maven-plugin**: .proto → Java 코드 생성
3. **build-helper-maven-plugin**: 생성된 코드를 소스 경로에 추가

---

## 기술적 의사결정

### 1. Guava 의존성 제거

**배경**:
- Curiostack 구현은 `assertj-guava`에 의존
- Google Truth도 Guava를 많이 사용

**결정**: Guava 의존성 완전 제거

**근거**:
1. AssertJ 원칙: 최소 의존성
2. 사용자 프로젝트의 버전 충돌 방지
3. Java 표준 라이브러리로 충분히 구현 가능

**구현**:
```java
// Guava 대신 Java 표준 사용
Set<FieldDescriptor> fieldsToCompare = new HashSet<>();  // Guava ImmutableSet 대신
List<Object> actualCopy = new ArrayList<>(actualList);   // Guava Lists.newArrayList 대신
```

### 2. Repeated 필드의 hasField() 문제

**문제**:
```java
// Repeated 필드에서 hasField() 호출 시 예외 발생
UnsupportedOperationException: hasField() called on a repeated field
```

**해결책**:
```java
if (field.isRepeated()) {
  // hasField() 대신 getField() 사용
  actualValue = actual.getField(field);
  expectedValue = expected.getField(field);
  
  // 빈 리스트는 null로 취급
  @SuppressWarnings("unchecked")
  List<Object> actualList = (List<Object>) actualValue;
  if (actualList.isEmpty()) actualValue = null;
} else {
  // 일반 필드는 hasField() 사용
  actualValue = actual.hasField(field) ? 
      actual.getField(field) : null;
}
```

**이유**:
- Protobuf에서 repeated 필드는 항상 존재 (빈 리스트로)
- `hasField()`는 optional 필드에만 유효
- 빈 리스트와 null을 구분할 필요 없음

### 3. 에러 메시지 생성 방식

**선택지**:
1. AssertJ 내부 API 사용 (`Failures.instance().failure()`)
2. `failWithMessage()` 직접 사용

**결정**: `failWithMessage()` 사용

**근거**:
1. `Failures` 클래스는 모듈에서 export되지 않음
2. `failWithMessage()`가 더 간단하고 직접적
3. 에러 메시지 커스터마이징 용이

**구현**:
```java
if (!differencer.compare(actual, expectedMessage)) {
  String differences = differencer.getLastDifferences();
  failWithMessage("Protobuf messages are not equal:%n%s", differences);
}
```

### 4. Maven 플러그인 선택

**Protobuf 코드 생성 플러그인**:

**옵션 1: Xolstice Plugin** ✅ 선택
- ex-Googler 개발
- 커뮤니티 지원 우수
- OS별 protoc 자동 다운로드

**옵션 2: os72 Plugin**
- executable jar 사용
- 별도 설치 불필요
- 덜 공식적

**결정 근거**:
- GitHub Actions에서 protoc 설치 용이
- 더 안정적이고 널리 사용됨
- OS 감지 플러그인과 조합 시 편리

### 5. 테스트 모듈 위치

**선택지**:
1. `assertj-protobuf/src/test` - 모듈 내부
2. `assertj-tests/assertj-integration-tests/assertj-protobuf-tests` - 별도 모듈

**결정**: 별도 통합 테스트 모듈 ✅

**근거**:
1. AssertJ 프로젝트 구조 준수
2. Protobuf 코드 생성이 필요한 테스트 분리
3. 테스트만 Java 21 사용 가능
4. 부모 POM의 enforcer 규칙 준수

### 6. Java Module System 지원

**결정**: `module-info.java` 제공

```java
module org.assertj.protobuf {
  requires transitive org.assertj.core;
  requires com.google.protobuf;

  exports org.assertj.protobuf.api;
}
```

**이유**:
1. AssertJ 프로젝트 전체가 모듈화
2. 향후 Java 플랫폼 호환성
3. 명확한 의존성 선언

---

## 사용 가이드

### 프로젝트에 추가하기

#### Maven

```xml
<dependency>
  <groupId>org.assertj</groupId>
  <artifactId>assertj-protobuf</artifactId>
  <version>4.0.0-SNAPSHOT</version>
  <scope>test</scope>
</dependency>

<dependency>
  <groupId>com.google.protobuf</groupId>
  <artifactId>protobuf-java</artifactId>
  <version>4.29.3</version>
</dependency>
```

#### Gradle

```groovy
testImplementation 'org.assertj:assertj-protobuf:4.0.0-SNAPSHOT'
implementation 'com.google.protobuf:protobuf-java:4.29.3'
```

### 기본 사용법

```java
import static org.assertj.protobuf.api.Assertions.assertThat;

@Test
void testProtobufMessage() {
  // GIVEN
  MyProto expected = MyProto.newBuilder()
      .setName("John")
      .setAge(30)
      .build();
  
  MyProto actual = service.getUser(userId);
  
  // WHEN/THEN
  assertThat(actual).isEqualTo(expected);
}
```

### 고급 사용 패턴

#### 1. 타임스탬프 필드 제외

```java
@Test
void testUserData_ignoringTimestamp() {
  // GIVEN
  UserProto expected = UserProto.newBuilder()
      .setName("John")
      .setEmail("john@example.com")
      .build();
  
  UserProto actual = userRepository.findById(userId);
  
  // WHEN/THEN - createdAt, updatedAt 무시
  assertThat(actual)
      .ignoringFields("createdAt", "updatedAt")
      .isEqualTo(expected);
}
```

#### 2. 리스트 순서 무시

```java
@Test
void testPermissions_ignoringOrder() {
  // GIVEN
  UserProto expected = UserProto.newBuilder()
      .addPermissions("READ")
      .addPermissions("WRITE")
      .addPermissions("DELETE")
      .build();
  
  UserProto actual = userService.getUserWithPermissions(userId);
  
  // WHEN/THEN - 권한 순서는 중요하지 않음
  assertThat(actual)
      .ignoringRepeatedFieldOrder()
      .isEqualTo(expected);
}
```

#### 3. 부분 비교 (API 응답 테스트)

```java
@Test
void testApiResponse_partialMatch() {
  // GIVEN - 최소한의 필드만 설정
  ResponseProto expectedData = ResponseProto.newBuilder()
      .setStatus("SUCCESS")
      .setUserId(123)
      .build();
  
  ResponseProto actual = apiClient.callEndpoint(request);
  
  // WHEN/THEN - expected에 설정된 필드만 확인
  assertThat(actual)
      .comparingExpectedFieldsOnly()
      .isEqualTo(expectedData);
  
  // 추가 필드 존재 확인
  assertThat(actual).hasField("timestamp");
}
```

#### 4. 복합 조건

```java
@Test
void testComplexScenario() {
  // GIVEN
  OrderProto expected = OrderProto.newBuilder()
      .setOrderId("ORD-123")
      .setCustomerId(456)
      .addItems(createItem("ITEM-1", 2))
      .addItems(createItem("ITEM-2", 1))
      .build();
  
  OrderProto actual = orderService.getOrder("ORD-123");
  
  // WHEN/THEN - 여러 조건 결합
  assertThat(actual)
      .ignoringFields("createdAt", "updatedAt", "version")
      .ignoringRepeatedFieldOrder()  // items 순서 무시
      .isEqualTo(expected);
}
```

### 주의사항

#### 1. Proto3의 기본값
```java
// Proto3에서는 설정하지 않은 필드가 기본값을 가짐
message User {
  string name = 1;
  int32 age = 2;  // 설정 안 하면 0
}

// 주의: age를 설정하지 않은 경우
User user1 = User.newBuilder().build();          // age = 0
User user2 = User.newBuilder().setAge(0).build(); // age = 0

// 이 둘은 동일함!
assertThat(user1).isEqualTo(user2);  // PASS
```

#### 2. Repeated 필드의 순서

```java
// 기본적으로 순서 포함 비교
assertThat(actual).isEqualTo(expected);  // [1,2,3] != [3,2,1]

// 순서 무시하려면 명시적으로 설정
assertThat(actual)
    .ignoringRepeatedFieldOrder()
    .isEqualTo(expected);  // [1,2,3] == [3,2,1]
```

#### 3. 중첩 메시지의 필드 경로

```java
message Order {
  Customer customer = 1;
}

message Customer {
  string name = 1;
  Address address = 2;
}

// 중첩 필드는 점(.)으로 구분
assertThat(actual)
    .ignoringFields("customer.address", "customer.phoneNumber")
    .isEqualTo(expected);
```

---

## 향후 계획

### 단기 계획 (1-2개월)

#### 1. 문서화 강화
- [ ] `assertj-protobuf/README.md` 작성
- [ ] 사용 예제 추가
- [ ] Javadoc 완성도 향상
- [ ] assertj-examples 저장소에 예제 추가

#### 2. 추가 Assertion 메서드
```java
// 필드 추출
assertThat(user).extracting("name", "age")
                .containsExactly("John", 30);

// 조건부 필드 무시
assertThat(actual)
    .ignoringFieldsMatchingRegex(".*Timestamp")
    .isEqualTo(expected);

// 필드별 커스텀 비교
assertThat(actual)
    .usingComparatorForField(timestampComparator, "createdAt")
    .isEqualTo(expected);
```

#### 3. 에러 메시지 개선
```java
// 현재
"Protobuf messages are not equal:
Field <name>: expected <John> but was <Jane>"

// 개선 목표
"Expecting:
  <MyProto{name='Jane', age=30}>
to be equal to:
  <MyProto{name='John', age=30}>
when comparing fields:
  - name: expected 'John' but was 'Jane'"
```

### 중기 계획 (3-6개월)

#### 1. 특수 Protobuf 타입 지원

```java
// google.protobuf.Any 지원
assertThat(anyMessage)
    .containsInstanceOf(UserProto.class)
    .extractingTypeUrl()
    .isEqualTo("type.googleapis.com/User");

// google.protobuf.Timestamp 지원
assertThat(message)
    .hasTimestampCloseTo("updatedAt", Instant.now(), 
                        Duration.ofMinutes(1));

// google.protobuf.Struct/Value 지원
assertThat(struct)
    .hasFieldValue("name", "John")
    .hasFieldValue("age", 30);
```

#### 2. 성능 최적화
- [ ] 대형 메시지 비교 성능 벤치마크
- [ ] Repeated 필드 비교 알고리즘 최적화
- [ ] 메모리 사용량 프로파일링

#### 3. 고급 비교 기능

```java
// Field mask 지원
FieldMask mask = FieldMask.newBuilder()
    .addPaths("name")
    .addPaths("email")
    .build();

assertThat(actual)
    .usingFieldMask(mask)
    .isEqualTo(expected);

// Recursive descent
assertThat(actual)
    .usingRecursiveComparison()
    .ignoringActualNullFields()
    .isEqualTo(expected);
```

### 장기 계획 (6개월+)

#### 1. Proto2 지원 강화
- Required 필드 검증
- Extension 필드 비교
- Unknown 필드 처리

#### 2. gRPC 통합
```java
// gRPC 스텁 assertion
assertThat(stub)
    .whenCalling(UserServiceGrpc::getUser)
    .withRequest(GetUserRequest.newBuilder()
                               .setUserId(123)
                               .build())
    .returnsMessage(expectedUser);

// 스트리밍 assertion
assertThat(responseStream)
    .hasSize(5)
    .allMatch(response -> response.getStatus() == Status.SUCCESS);
```

#### 3. 코드 생성 도구
```java
// Assertion 클래스 자동 생성
// UserProto → UserProtoAssert 생성
assertThat(user)
    .hasName("John")
    .hasAge(30)
    .hasEmail("john@example.com");
```

#### 4. 커뮤니티 피드백 반영
- GitHub Discussions 모니터링
- 사용자 요청 기능 우선순위 결정
- Best practices 문서화

---

## 빌드 및 배포

### 로컬 빌드

```bash
# 전체 프로젝트 빌드
./mvnw clean install

# Protobuf 모듈만 빌드
./mvnw clean install -pl assertj-protobuf

# 테스트 포함 빌드
./mvnw clean install -pl assertj-protobuf,assertj-tests/assertj-integration-tests/assertj-protobuf-tests

# 테스트 스킵
./mvnw clean install -DskipTests
```

### 코드 품질 검증

```bash
# Spotless 포맷팅 확인
./mvnw spotless:check

# Spotless 자동 포맷팅
./mvnw spotless:apply

# 라이선스 헤더 확인
./mvnw license:check

# 라이선스 헤더 추가
./mvnw license:format
```

### 테스트 실행

```bash
# 전체 테스트
./mvnw test

# 특정 모듈 테스트
./mvnw test -pl assertj-tests/assertj-integration-tests/assertj-protobuf-tests

# 특정 테스트 클래스
./mvnw test -Dtest=MessageAssert_isEqualTo_Test

# 테스트 커버리지 리포트
./mvnw clean test jacoco:report
```

### 릴리스 프로세스 (향후)

1. **버전 업데이트**
   ```bash
   ./mvnw versions:set -DnewVersion=4.0.0-M3
   ```

2. **변경사항 문서화**
   - CHANGELOG.md 업데이트
   - Release notes 작성

3. **빌드 및 테스트**
   ```bash
   ./mvnw clean verify
   ```

4. **Maven Central 배포**
   ```bash
   ./mvnw clean deploy -Ppublish
   ```

---

## 기여 방법

### 개발 환경 설정

1. **JDK 설치**
   - Java 17 이상 필요 (테스트는 Java 21)

2. **IDE 설정**
   - IntelliJ IDEA 또는 Eclipse
   - Eclipse Code Formatter 플러그인 설치
   - `eclipse/assertj-eclipse-formatter.xml` 임포트

3. **프로젝트 클론**
   ```bash
   git clone https://github.com/assertj/assertj.git
   cd assertj
   ```

4. **브랜치 생성**
   ```bash
   git checkout -b feature/my-new-feature
   ```

### 코딩 규칙

1. **코드 스타일**
   - Spotless 자동 포맷팅 사용
   - 커밋 전 `./mvnw spotless:apply` 실행

2. **테스트**
   - 모든 공개 메서드에 테스트 작성
   - GIVEN/WHEN/THEN 구조 사용
   - 명명 규칙: `should_pass_when_xxx`, `should_fail_when_xxx`

3. **문서화**
   - 모든 공개 API에 Javadoc 작성
   - 성공/실패 예제 포함

### Pull Request 프로세스

1. **로컬에서 빌드 및 테스트**
   ```bash
   ./mvnw clean verify
   ```

2. **변경사항 커밋**
   ```bash
   git add .
   git commit -m "feat: Add new assertion method"
   ```

3. **원격 저장소에 푸시**
   ```bash
   git push origin feature/my-new-feature
   ```

4. **PR 생성**
   - 명확한 제목과 설명
   - 관련 이슈 링크
   - 스크린샷 또는 예제 코드

---

## 참고 자료

### 공식 문서
- [AssertJ Documentation](https://assertj.github.io/doc/)
- [Protocol Buffers Guide](https://protobuf.dev/)
- [gRPC Java Documentation](https://grpc.io/docs/languages/java/)

### 관련 프로젝트
- [Google Truth Protobuf](https://github.com/google/truth/tree/master/extensions/proto)
- [Curiostack assertj-proto](https://github.com/curioswitch/curiostack/tree/master/tools/assertj-proto)

### 기술 문서
- [Protobuf Java Tutorial](https://protobuf.dev/getting-started/javatutorial/)
- [Maven Protobuf Plugin](https://www.xolstice.org/protobuf-maven-plugin/)
- [AssertJ Core Guide](https://assertj.github.io/doc/#assertj-core-assertions-guide)

---

## 라이선스

이 프로젝트는 Apache License 2.0 하에 배포됩니다.

```
Copyright 2012-2025 the original author or authors.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```

---

## 연락처

- **GitHub Issue**: https://github.com/assertj/assertj/issues
- **Stack Overflow**: Tag `assertj`
- **Discord**: AssertJ Community (if available)

---

**문서 버전**: 1.0  
**최종 수정일**: 2025-09-30  
**작성자**: JongJun Kim
