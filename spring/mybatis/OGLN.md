# MyBatis OGNL (Object-Graph Navigation Language)

## OGNL이란?

OGNL(Object-Graph Navigation Language)은 객체 그래프를 탐색하고 조작하기 위한 표현식 언어입니다. MyBatis에서는 동적 SQL을 작성할 때 OGNL을 사용하여 객체의 속성에 접근하고 조건을 평가합니다.

## OGNL의 주요 특징

1. **객체 속성 접근**
   - 점(.) 표기법을 사용하여 객체의 속성에 접근
   - 예: `user.name`, `order.items[0].price`

2. **메서드 호출**
   - 객체의 메서드를 호출할 수 있음
   - 예: `user.getName()`, `list.size()`

3. **연산자 지원**
   - 산술 연산자: `+`, `-`, `*`, `/`, `%`
   - 비교 연산자: `==`, `!=`, `>`, `<`, `>=`, `<=`
   - 논리 연산자: `&&`, `||`, `!`

## MyBatis에서의 OGNL 사용 예시

### 1. 동적 SQL에서의 사용
```xml
<select id="findUsers" resultType="User">
  SELECT * FROM users
  <where>
    <if test="name != null">
      AND name = #{name}
    </if>
    <if test="age > 18">
      AND age > #{age}
    </if>
  </where>
</select>
```

### 2. 컬렉션 처리
```xml
<select id="findUsersByIds" resultType="User">
  SELECT * FROM users
  WHERE id IN
  <foreach collection="ids" item="id" open="(" separator="," close=")">
    #{id}
  </foreach>
</select>
```

### 3. 조건문에서의 사용
```xml
<select id="findUsers" resultType="User">
  SELECT * FROM users
  <choose>
    <when test="role == 'admin'">
      WHERE role = 'admin'
    </when>
    <when test="role == 'user'">
      WHERE role = 'user'
    </when>
    <otherwise>
      WHERE role IS NOT NULL
    </otherwise>
  </choose>
</select>
```

## 주의사항

1. **null 체크**
   - OGNL 표현식에서 null 체크는 중요
   - `test="name != null"`과 같이 명시적으로 체크 필요

2. **문자열 비교**
   - 문자열 비교 시 작은따옴표(') 사용
   - 예: `test="role == 'admin'"`

3. **컬렉션 접근**
   - List나 Map과 같은 컬렉션에 접근할 때는 적절한 인덱스나 키 사용
   - 예: `list[0]`, `map['key']`

## 결론

OGNL은 MyBatis에서 동적 SQL을 작성할 때 매우 유용한 표현식 언어입니다. 객체의 속성에 쉽게 접근하고, 조건을 평가하며, 복잡한 쿼리를 동적으로 생성할 수 있게 해줍니다. 올바르게 사용하면 더 유연하고 강력한 SQL 매핑을 구현할 수 있습니다.

## 특이 케이스: OGNL의 비교 연산자 동작 방식

```xml
<select id="findUsers" resultType="User">
  SELECT * FROM users
  <where>
    <!-- name이 "48"인 경우 해당 if문은 false가 되어 name 조건이 적용되지 않는다 -->
    <if test="name != null and name != '0'">
      AND name = #{name}
    </if>
    <if test="age > 18">
      AND age > #{age}
    </if>
  </where>
</select>
```

위의 예시에서 `name`이 "48"인 경우, `AND name = #{name}` 조건이 SQL에 포함되지 않는 예상치 못한 동작이 발생할 수 있습니다. 이는 OGNL의 비교 연산자 처리 방식 때문입니다.

### OGNL 비교 연산자의 내부 동작

1. **파싱 과정**
   - `org.apache.ibatis.scripting.xmltags.ExpressionEvaluator`가 `name != '0'` 표현식을 파싱
   - 좌항(`name`)과 우항(`'0'`)을 분리하여 처리

2. **비교 연산 처리**
   - `org.apache.ibatis.ognl.OgnlOps.equal` 메서드에서 비교 수행
   - 내부적으로 다음과 같은 타입 변환 발생:
     - String "48" → double 48.0
     - char '0' → double 48.0 (ASCII 코드 값)

3. **문제 발생 원인**
   - String과 char 타입을 비교할 때 자동으로 double로 변환
   - "48"과 '0'이 모두 48.0으로 변환되어 동등 비교 시 true 반환
   - 결과적으로 `name != '0'` 조건이 false가 되어 의도하지 않은 동작 발생

### 해결 방안

문자열 비교 시에는 OGNL의 비교 연산자 대신 Java의 String 메서드를 사용하는 것이 안전합니다:

```xml
<if test='name != null and !"0".equals(name)'>
  AND name = #{name}
</if>
```

이렇게 하면 타입 변환 문제 없이 정확한 문자열 비교가 가능합니다.