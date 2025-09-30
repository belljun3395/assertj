# AssertJ Protobuf 모듈 구현

## 개요

이 PR은 AssertJ에 Protocol Buffers 메시지를 위한 assertion 지원을 추가합니다. gRPC와 Protobuf를 사용하는 Java 개발자들이 유창한(fluent) API 스타일로 테스트를 작성할 수 있습니다.

**관련 이슈**: #3736

## 구현된 기능

### 1. 핵심 모듈: `assertj-protobuf`

새로운 Maven 모듈로 다음을 제공합니다:

#### 주요 클래스

**`org.assertj.protobuf.api.Assertions`**
- AssertJ Protobuf의 진입점
- `assertThat(Message)` static 메서드 제공

**`org.assertj.protobuf.api.MessageAssert<T extends Message>`**
- Protobuf 메시지를 위한 주요 assertion 클래스
- `AbstractAssert`를 상속하여 AssertJ 표준 준수

**`org.assertj.protobuf.internal.MessageDifferencer`**
- 내부 메시지 비교 로직 처리
- Repeated 필드, 중첩 메시지, 필드 무시 등 지원

#### 제공 메서드

```java
// 기본 비교
assertThat(actual).isEqualTo(expected);

// Repeated 필드 순서 무시
assertThat(actual).ignoringRepeatedFieldOrder().isEqualTo(expected);

// 특정 필드 제외
assertThat(actual).ignoringFields("timestamp").isEqualTo(expected);

// 부분 비교 (기대값의 필드만)
assertThat(actual).comparingExpectedFieldsOnly().isEqualTo(expected);

// 필드 존재 확인
assertThat(actual).hasField("name");
assertThat(actual).doesNotHaveField("deletedAt");
```

### 2. 통합 테스트 모듈: `assertj-protobuf-tests`

#### 테스트 구조
- `src/test/proto/test.proto`: 테스트용 Protobuf 정의
- 4개 테스트 클래스, 15개 테스트 케이스
- 모든 테스트 통과 ✅

#### 테스트 커버리지
- `MessageAssert_isEqualTo_Test`: 5개 테스트
- `MessageAssert_ignoringRepeatedFieldOrder_Test`: 5개 테스트
- `MessageAssert_ignoringFields_Test`: 2개 테스트
- `MessageAssert_hasField_Test`: 3개 테스트

## 기술적 설계

### 의존성 관리
```xml
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
```

**주요 결정사항**:
- `protobuf-java`를 `provided` scope로 설정하여 사용자가 버전 선택 가능
- **Guava 의존성 제거**: Curiostack 구현과 달리 최소 의존성 원칙 준수
- Java Module System 지원 (`module-info.java`)
- OSGi 번들 지원 (bnd-maven-plugin)

### 패키지 구조
```
org.assertj.protobuf/
├── api/              # 공개 API (하위 호환성 보장)
│   ├── Assertions
│   └── MessageAssert
└── internal/         # 내부 구현 (japicmp 제외)
    └── MessageDifferencer
```

### 핵심 구현 로직

#### Repeated 필드 처리
Protobuf의 repeated 필드는 `hasField()`를 지원하지 않아 특별한 처리가 필요합니다:

```java
if (field.isRepeated()) {
  // hasField() 대신 getField() 사용
  actualValue = actual.getField(field);
  expectedValue = expected.getField(field);
  
  // 빈 리스트는 null로 취급
  List<Object> actualList = (List<Object>) actualValue;
  if (actualList.isEmpty()) actualValue = null;
}
```

#### AS_SET 모드 (순서 무시)
Repeated 필드를 Set처럼 비교하는 알고리즘:

1. 기대값의 각 항목에 대해
2. 실제값 리스트에서 매칭되는 항목 찾기
3. 찾으면 실제값 리스트에서 제거 (중복 방지)
4. 모든 항목이 매칭되면 성공

#### 재귀적 메시지 비교
중첩 메시지는 재귀적으로 비교하며, 필드 경로를 추적합니다:
- `"field_name"` → `"nested.field_name"` → `"nested.child.field_name"`

## 사용 예제

### 기본 사용법
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

### 고급 사용 패턴
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

## 빌드 및 테스트

### 빌드
```bash
# Protobuf 모듈만 빌드
./mvnw clean install -pl assertj-protobuf

# 테스트 포함
./mvnw clean install -pl assertj-protobuf,assertj-tests/assertj-integration-tests/assertj-protobuf-tests
```

### 테스트 실행
```bash
./mvnw test -pl assertj-tests/assertj-integration-tests/assertj-protobuf-tests
```

### 테스트 결과
```
[INFO] Tests run: 15, Failures: 0, Errors: 0, Skipped: 0
```

## 체크리스트

### 코드 품질
- [x] Spotless 포맷팅 적용
- [x] Apache 2.0 라이선스 헤더 추가
- [x] 모든 공개 메서드에 Javadoc 작성
- [x] 성공/실패 예제 포함

### 테스트
- [x] JUnit 5 사용
- [x] GIVEN/WHEN/THEN 구조
- [x] 명명 규칙 준수 (`should_pass_xxx`, `should_fail_xxx`)
- [x] Package-private visibility
- [x] 모든 테스트 통과 (15/15)

### 의존성
- [x] 최소 의존성 원칙 준수
- [x] Provided scope 활용
- [x] Guava 의존성 없음

### 호환성
- [x] Java Module System 지원
- [x] OSGi 번들 지원
- [x] japicmp 설정 (internal 패키지 제외)

## 향후 작업

다음 기능들은 별도 PR로 추가될 수 있습니다:

1. **추가 Assertion 메서드**
   - `extracting()` - 필드 추출
   - `ignoringFieldsMatchingRegex()` - 정규식 기반 필드 무시

2. **특수 Protobuf 타입 지원**
   - `google.protobuf.Any`
   - `google.protobuf.Timestamp`
   - `google.protobuf.Struct`

3. **에러 메시지 개선**
   - 더 상세한 차이점 표시
   - 컬러 출력 지원

## 참고 자료

- **이슈**: #3736
- **Google Truth Protobuf**: https://github.com/google/truth/tree/master/extensions/proto
- **Curiostack assertj-proto**: https://github.com/curioswitch/curiostack/tree/master/tools/assertj-proto (MIT License)

## 기여

이 구현은 다음을 참고했습니다:
- Google Truth의 Protobuf 확장 (Apache 2.0)
- Curiostack의 assertj-proto (MIT License)

모든 코드는 AssertJ의 코딩 스타일과 원칙을 따르도록 새로 작성되었습니다.

---

**Author**: JongJun Kim  
**Closes**: #3736
