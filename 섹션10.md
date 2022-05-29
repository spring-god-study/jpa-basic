## 객체지향 쿼리 언어1 - 기본 문법

### 소개

#### **JPA는 다양한 쿼리 방법을 지원한다**

- JPQL
- JPA Criteria
- QueryDSL
- 네이티브 SQL
- Jdbc API 직접 사용, MyBatis, SpringJdbcTemplate과 함께 사용

#### **JPQL(Java Persistence Query Language) 소개**

- 검색 조건이 포함된 SQL이 필요 (ex. 나이가 18세 이상인 회원 검색)
- **JPQL : SQL을 추상화한 객체지향 쿼리 언어**
- SQL 문법과 유사 (SELECT, FROM, WHERE, GROUP BY, HAVING, JOIN 지원)
- 엔티티 객체 대상 쿼리(JPQL) vs. 데이터베이스 테이블 대상 쿼리(SQL)

```java
String jpql = "select m from Member m where m.name like '%hello%'";
List<Member> result = em.createQuery(jpql, Member.class).getResultList();
```

👉 테이블이 아닌 객체 대상 검색 (여기서 m은 Member 엔티티를 가리킴)

- 단점: 동적 쿼리하기 힘듦 -> JPA Criteria, QueryDSL (자바 코드로 짜서 JPQL을 빌드해주는 generator라고 함)

#### **Criteria 소개**

- 단점: 너무 복잡하고 실용성이 없다
- Criteria <<<<< QueryDSL
  - 공통점: 문자가 아닌 자바 코드로 JPQL 작성 가능, JPQL 빌더 역할, 동적쿼리 작성 가능
- 참고 코드:

```java
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<Member> query = cb.createQuery(Member.class);
Root<Member> m = query.from(Member.class);
CriteriaQuery<Member> cq =  query.select(m).where(cb.equal(m.get("username"), "kim"));
List<Member> resultList = em.createQuery(cq).getResultList();
```

#### **QueryDSL 소개**

```java
//JPQL
//select m from Member m where m.age > 18
JPAFactoryQuery query = new JPAQueryFactory(em);
QMember m = QMember.member;

List<Member> list =
query.selectFrom(m)
        .where(m.age.get(18))
        .orderBy(m.name.desc())
        .fetch();
```

- 단순하고 쉽다는 장점 (-> 영한 님 왈; 모두 QueryDSL 하세요~~)

#### **네이티브SQL 소개**

- JPA가 제공하는 SQL을 직접 사용하는 개념(쌩쿼리)
- JPQL로 해결할 수 없는 특정 DB에 의존적인 기능 (ex. Oracle connect by)

```java
String sql = "SELECT ID, AGE, TEAM_ID, NAME FROM MEMBER WHERE NAME = ‘kim’";
List<Member> resultList = em.createNativeQuery(sql, Member.class).getResultList();
```

#### **JDBC 직접 사용, SpringJdbcTemplate**

- JPA를 사용하면서 JDBC 커넥션을 직접 사용하거나, 스프링 JdbcTemplate, 마이바티스등을 함께 사용 가능
- 주의) 영속성 컨텍스트를 적절한 시점에 강제로 플러시 필요

### 기본문법과 쿼리 API

> select*문 :: = <br> &nbsp;&nbsp;&nbsp;&nbsp; select*절 <br> &nbsp;&nbsp;&nbsp;&nbsp; from*절 <br> &nbsp;&nbsp;&nbsp;&nbsp; [where*절] <br> &nbsp;&nbsp;&nbsp;&nbsp; [groupby_절] <br> &nbsp;&nbsp;&nbsp;&nbsp; [having_절] <br> &nbsp;&nbsp;&nbsp;&nbsp; [orderby_절]

- SQL과 유사: `select m from Member m where m.age > 18` (별칭 m 필수)

- 집합과 정렬:

```sql
select COUNT(m), //회원수
       SUM(m.age), //나이 합
       AVG(m.age), //평균 나이
       MAX(m.age), //최대 나이
       MIN(m.age) //최소 나이
from Member m
```

     GROUP BY, HAVING, ORDER BY

- TypeQuery, Query
  - TypeQuery: 반환 타입이 명확할 때
  - Query: 반환 타입이 명확하지 않을 때

```java
TypedQuery<Member> query = em.createQuery("SELECT m FROM Member m", Member.class);
Query query = em.createQuery("SELECT m.username, m.age from Member m");
```

- 결과 조회 API

  - `query.getResultList()`: 결과가 하나 이상일 때
  - `query.getSingleResult()`: 결과가 정확히 하나일 때 (-> 결과가 없거나 둘 이상일 때 exception 터짐)

- 파라미터 바인딩

<이름 기준>

```java
> SELECT m FROM Member m where m.username=:username (쿼리)
//...
query.setParameter("username", usernameParam);
```

<위치 기준>

```java
> SELECT m FROM Member m where m.username=?1 (쿼리)
//...
query.setParameter(1, usernameParam);
```

    이름 기준 파라미터 바인딩 추천

### 프로젝션 (SELECT)

SELECT절에 조회할 대상을 지정하는 것

- 프로젝션 대상: 엔티티, 임베디드 타입, 스칼라 타입(기본 타입)

```sql
-- 엔티티 프로젝션
SELECT m FROM Member m
SELECT m.team FROM Member m

-- 임베디드 프로젝션
SELECT m.address FROM Member m

-- 스칼라 타입 프로젝션
SELECT m.username, m.age FROM Member m
```

    - DISTINCT로 중복 제거

- 여러값 조회
  1. Query 타입
  2. Object[] 타입
  3. new 명령어 (단순 값을 DTO로 바로 조회) :
     `SELECT new jpabook.jpql.UserDTO(m.username, m.age) FROM Member m`

### 페이징

- `setFirstResult(int startPosition)`: 조회 시작 위치 지정
- `setMaxResults(int maxResult)` : 조회할 데이터 수 지정

```java
//페이징 쿼리
String jpql = "select m from Member m order by m.name desc";
//0번째부터 10개 가져오기
List<Member> resultList
= em.createQuery(jpql, Member.class)
    .setFirstResult(0)
    .setMaxResults(10)
    .getResultList();
```

    > 각각의 방언에 맞게 SQL문으로 변환

### 조인

- 내부 조인: `select m from Member m join m.team t`
- 외부 조인: `select m from Member m left join m.team t`
- 세타 조인: `select count(m) from Member m, Team t where m.username = t.name`

- ON절

  - 조인 대상 필터링
  - ex. 회원과 팀을 조인하면서, 팀 이름이 A인 팀만 조인

  ```sql
  -- JPQL
  SELECT m, t FROM Member m LEFT JOIN m.team t on t.name = 'A'

  -- SQL
  SELECT m.*, t.* FROM Member m LEFT JOIN Team t ON m.TEAM_ID=t.id and t.name='A'
  ```

  - 연관관계 없는 엔티티 외부 조인
  - ex. 회원의 이름과 팀의 이름이 같은 대상 외부 조인

  ```sql
  -- JPQL
  SELECT m, t FROM Member m LEFT JOIN Team t on m.username = t.name

  --SQL
  SELECT m.*, t.* FROM Member m LEFT JOIN Team t ON m.username = t.name
  ```

### 서브쿼리

- ex1. 나이가 평균보다 많은 회원:

```sql
select m from Member m
where m.age > (select avg(m2.age) from Member m2)
```

- ex2. 한건이라도 주문한 고객:

```sql
select m from Member m
where (select count(o) from Order o where m = o.member) > 0
```

- 서브쿼리 지원 함수

  - `[NOT] EXISTS (subquery)`: 서브쿼리에 결과가 존재하면 참
    - `{ALL|ANY|SOME} (subquery)`
    - ALL - 모두 만족하면 참, ANY, SOME - 조건을 하나라도 만족하면 참
  - `[NOT] IN (subquery)`: 서브쿼리의 결과 중 하나라도 같은 것이 있으면 참

- 예제

```sql
--팀A 소속인 회원
select m from member m
where exists (select t from m.team t where t.name = '팀A')

-- 전체 상품 각각의 재고보다 주문량이 많은 주문들
select o from Order o
where o.orderAmount > (select p.stockAmount from Product p)

-- 어떤 팀이든 팀에 소속된 회원
select m from Member m
where m.team = ANY(select t from Team t)
```

- 서브쿼리 한계: WHERE, HAVING, SELECT 절만 가능, FROM 절의 서브쿼리는 현재 불가

### JPQL 타입 표현식과 기타식

- 문자, 숫자, Boolean, ENUM(패키지명 포함), 엔티티 타입
- JPQL 기타
  - EXISTS, IN
  - AND, OR, NOT
  - =, >, >=, <, <=, <>
  - BETWEEN, LIKE, IS NULL

### 조건식 - CASE식

- 기본 CASE 식

```sql
select
    case when m.age <= 10 then '학생요금'
         when m.age >= 60 then '경로요금'
         else '일반요금'
    end
from Member m
```

- 단순 CASE 식

```sql
select
    case t.name
        when '팀A' then '인센티브110%'
        when '팀B' then '인센티브120%'
        else '인센티브105%'
    end
from Team t
```

- COALESCE: 하나씩 조회해서 null이 아니면 반환

```sql
--사용자 이름이 없으면 이름 없는 회원을 반환
select coalesce(m.username, '이름 없는 회원') from Member m
```

- NULLIF: 두값이 같으면 null, 다르면 첫번째 값 반환

```sql
-- 사용자 이름이 관리자면 null 반환, 나머지는 본인 이름 반환
select nullif(m.username, '관리자') from Member m
```

### JPQL 함수

- CONCAT, SUBSTRING, TRIM, LOWER, UPPER, LENGTH, LOCATE, ABS, SORT, MOD, SIZE, INDEX
- 여기서 해결이 안되면? -> 사용자 정의 함수 등록하기
