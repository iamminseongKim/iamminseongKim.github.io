---
title: 스프링 DB 2편 - 데이터 접근 활용 기술 (2) - 데이터 접근 기술 - 스프링 JdbcTemplate - 1
aliases: 
tags:
  - spring
  - db
  - JdbcTemplate
categories:
  - spring
toc: true
toc_label: 목차
date: 2024-09-27
last_modified_at: 2024-09-27
---
>  인프런 스프링 DB 2편 - 데이터 접근 활용 기술편을 학습하고 정리한 내용 입니다.

## JdbcTemplate 소개와 설정

SQL을 직접 사용하는 경우에 스프링이 제공하는 JdbcTemplate은 아주 좋은 선택지다. JdbcTemplate은 JDBC를 매우 편리하게 사용할 수 있게 도와준다.

### 장점
- 설정의 편리함
	- JdbcTemplate은 `spring-jdbc`라이브러리에 포함되어 있는데, 이 라이브러리는 스프링으로 JDBC를 사용할 때 기본으로 사용되는 라이브러리이다. 그리고 별도의 복잡한 설정 없이 바로 사용할 수 있다.
- 번복 문제 해결
	- JdbcTemplate은 **템플릿 콜백 패턴**을 사용해서, JDBC를 직접 사용할 때 발생하는 대부분의 반복 작업을 대신 처리해준다.
	- 개발자는 SQL을 작성하고, 전달할 파리미터를 정의하고, 응답 값을 매핑하기만 하면 된다.
	- 우리가 생각할 수 있는 대부분의 반복 작업을 대신 처리해준다.
		- 커넥션 획득
		- `statment`를 준비하고 실행
		- 결과를 반복하도록 루프를 실행
		- 커넥션 종료, `statement`, `resultset`종료
		- 예외 발생시 스프링 예외 변환기 실행

### 단점

- 동적 SQL을 해결하기 어렵다.

### JdbcTemplate 설정

해당 내용을 `build.gradle`에 추가하자

```gradle
//JdbcTemplate 추가 
implementation 'org.springframework.boot:spring-boot-starter-jdbc' 
//H2 데이터베이스 추가 
runtimeOnly 'com.h2database:h2'
```
- `org.springframework.boot:spring-boot-starter-jdbc` 를 추가하면 `JdbcTemplate`이 들어있는 `spring-jdbc`가 라이브러리에 포함된다.
- 여기서는 H2 데이터베이스에 접속해야 하기 때문에 H2 데이터베이스의 클라이언트 라이브러리(Jdbc Driver)도 추가하자.
- `runtimeOnly 'com.h2database:h2'`

## JdbcTemplate 적용 1 - 기본

이제부터 본격적으로 JdbcTemplate을 사용해서 메모리에 저장하던 데이터를 데이터베이스에 저장해보자.

`ItemRepository`인터페이스가 있으니 이 인터페이스를 기반으로 JdbcTemplate을 사용하는 새로운 구현체를 개발하자.


![](https://i.imgur.com/qAlBGJR.png)

다음과 같이 `hello.itemservice.repository.jdbctemplate` 패키지를 만들었다.


`JdbcTemplateItemRepositoryV1`
```java
@Slf4j  
public class JdbcTemplateItemRepositoryV1 implements ItemRepository {  
  
    private final JdbcTemplate template;  
  
    public JdbcTemplateItemRepositoryV1(DataSource dataSource) {  
        this.template = new JdbcTemplate(dataSource);  
    }  
  
    @Override  
    public Item save(Item item) {  
        String sql = "insert into item(item_name, price, quantity) values(?, ?, ?)";  
        KeyHolder keyHolder = new GeneratedKeyHolder();  
  
        template.update(connection -> {  
            // 자동 증가 키  
            PreparedStatement ps = connection.prepareStatement(sql, new String[]{"id"});  
            ps.setString(1, item.getItemName());  
            ps.setInt(2, item.getPrice());  
            ps.setInt(3, item.getQuantity());  
            return ps;  
        }, keyHolder);  
  
        long key = keyHolder.getKey().longValue();  
        item.setId(key);  
  
        return item;  
    }  
  
    @Override  
    public void update(Long itemId, ItemUpdateDto updateParam) {  
        String sql = "update item set item_name = ?, price = ?, quantity = ? where id = ?";  
        template.update(sql,  
                updateParam.getItemName(),  
                updateParam.getPrice(),  
                updateParam.getQuantity(),  
                itemId);  
    }  
  
    @Override  
    public Optional<Item> findById(Long id) {  
        String sql = "select id, item_name, price, quantity from item where id = ?";  
        Item item = template.queryForObject(sql, itemRowMapper(), id);  
        return Optional.ofNullable(item);  
    }  
  
    private RowMapper<Item> itemRowMapper() {  
        return ((rs, rowNum) -> {  
            Item item = new Item();  
            item.setId(rs.getLong("id"));  
            item.setItemName(rs.getString("item_name"));  
            item.setPrice(rs.getInt("price"));  
            item.setQuantity(rs.getInt("quantity"));  
            return item;  
        });  
    }  
  
  
    @Override  
    public List<Item> findAll(ItemSearchCond cond) {  
        String itemName = cond.getItemName();  
        Integer maxPrice = cond.getMaxPrice();  
  
        String sql = "select id, item_name, price, quantity from item";  
  
        //동적 쿼리  
        if (StringUtils.hasText(itemName) || maxPrice != null) {  
            sql += " where";  
        }  
        boolean andFlag = false;  
        List<Object> param = new ArrayList<>();  
        if (StringUtils.hasText(itemName)) {  
            sql += " item_name like concat('%',?,'%')";  
            param.add(itemName);  
            andFlag = true;  
        }  
        if (maxPrice != null) {  
            if (andFlag) {  
                sql += " and";  
            }  
            sql += " price <= ?";  
            param.add(maxPrice);  
        }  
        log.info("sql={}", sql);  
  
        return template.query(sql, itemRowMapper(), param.toArray());  
    }  
}
```

- `JdbcTemplateItemRepositoryV1`은 `ItemRepository`인터페이스를 구현했다.
- `this.template = new JdbcTemplate(dataSource)`
	- `JdbcTemplate`은 데이터소스(`dataSource`)가 필요하다.
	- `JdbcTemplateItemRepositoryV1()`생성자를 보면 `dataSource`를 의존 관계 주입 받고 생성자 내부에서 `JdbcTemplate`을 생성한다. 스프링에서는 `JdbcTemplate`을 사용할 때 관례상 이 방법을 많이 사용한다.
	- 물론 `JdbcTemplate`을 스프링 빈으로 직접 등록하고 주입받아도 된다.


**save()** : 데이터를 저장한다.
- `template.update()` : 데이터를 변경할 때는 `update()`를 사용하면 된다.
	- `INSERT`, `UPDATE` , `DELETE` SQL에 사용한다.
	- `template.update()`의 반환 값은 `int`인데, 영향 받은 로우 수를 반환한다.
- 데이터를 저장할 때 PK 생성에 `identity`(auto increment) 방식을 사용하기 때문에, PK인 ID 값을 개발자가 직접 지정하는 것이 아니라 **비워두고 저장**해야 한다. 그러면 데이터베이스가 PK인 ID를 대신 생성해준다.
- 문제는 이렇게 데이터베이스가 대신 생성해주는 PK ID 값은 데이터베이스가 생성하기 때문에, 데이터베이스에 INSERT가 완료 되어야 생성된 PK ID 값을 확인할 수 있다.
- `KeyHolder`와 `connection.prepareStatement(sql, new String[]{"id"})`를 사용해서 id를 지정해주면 INSERT 쿼리 실행 이후에 데이터베이스에서 생성된 ID 값을 조회할 수 있다.
- 참고로 JdbcTemplate이 제공하는 `SimpleJdbcInsert`라는 훨씬 편리한 기능이 있으므로 대략 이렇게 사용한다 정도로만 알아두면 된다.

**update()** : 데이터를 업데이트 한다.
- `template.update()` : 데이터를 변경할 때는 `update()`를 사용하면 된다.
	- `?`에 바인딩할 파라미터를 순서대로 전달하면 된다.
	- 반환 값은 해당 쿼리의 영향을 받은 로우 수 이다. 여기서는 `where id=?`를 지정했기 때문에 영향 받은 로우수는 최대 1개이다.

**findById()** : 데이터를 하나 조회한다.
- `template.queryForObject()`
	- 결과 로우가 하나일 때 사용한다.
	- `RowMapper`는 데이터베이스의 반환 결과인 `ResultSet`을 객체로 변환한다.
	- 결과가 없으면 `EmptyResultDataAccessException`예외가 발생한다.
	- 결과가 둘 이상이면 `IncorrectResultSizeDataAccessException`예외가 발생한다.
- `ItemRepository.findById()` 인터페이스는 결과가 없을 때 `Optional`을 반환해야 한다. 따라서 결과가 없으면 예외를 잡아서 `Optional.empty`를 대신 반환하면 된다.

![](https://i.imgur.com/SJHiTZl.png){: .align-center}


`스프링 jdbc 6.1.12`를 사용중인데 여기서는 따로 `null`일때 `null`로 리턴해줘서 `try~catch`를 쓰지 않아도 될 것 같다. 

```java
@Override
public Optional<Item> findById(Long id) {  
	String sql = "select id, item_name, price, quantity from item where id = ?";  
	Item item = template.queryForObject(sql, itemRowMapper(), id);  
	return Optional.ofNullable(item);  
}  
```


**findAll()** : 데이터를 리스트로 조회한다. 그리고 검색 조건으로 적절한 데이터를 찾는다.
- `template.query()`
	- 결과가 하나 이상일 때 사용한다.
	- `RowMapper`는 데이터베이스의 반환 결과인 `ResultSet`을 객체로 변환한다.
	- 결과가 없으면 빈 컬렉션을 반환한다.
	- 동적 쿼리에 대한 부분은 바로 다음에 다룬다.


**itemRowMapper()** : 데이터베이스의 조회 결과를 객체로 변환할 때 사용한다.
- JDBC를 직접 사용할 때 `ResultSet`를 사용했던 부분을 떠올리면 된다.
- 차이가 있다면 다음과 같이 JdbcTemplate이 다음과 같은 루프를 돌려주고, 개발자는 `RowMapper`를 구현해서 그 내부 코드만 채운다고 이해하면 된다. 

```java
while(resultSet 이 끝날 때 까지) { 
	rowMapper(rs, rowNum) 
}
```


## JdbcTemplate 적용 2 - 동적 쿼리 문제

결과를 검색하는 `findAll()`에서 어려운 부분은 사용자가 검색하는 값에 따라서 실행하는 SQL이 동적으로 달려져야 한다는 점이다.

예를 들어서 다음과 같다.

1. 검색 조건이 없음

```sql
select id, item_name, price, quantity from item
```

2. 상품명(`itemName`)으로 검색
```sql
select id, item_name, price, quantity from item 
where item_name like concat('%',?,'%')
```

3. 최대 가격(`maxPrice`)으로 검색 
```sql
select id, item_name, price, quantity from item
where price <= ?
```

4. 상품명(`itemName`), 최대 가격(`maxPrice`) 둘다 검색
```sql
select id, item_name, price, quantity from item
where item_name like concat('%',?,'%')
and price <= ?
```

결과적으로 4가지 상황에 따른 SQL을 동적으로 생성해야 한다. 동적 쿼리가 언듯 보면 쉬워 보이지만, 막상 개발해보면 생각보다 다양한 상황을 고민해야 한다. 예를 들어서 어떤 경우에는 `where`를 앞에 넣고 어떤 경우에는 `and`를 넣어야 하는지 등을 모두 계산해야 한다.

그리고 각 상황에 맞추어 파라미터도 생성해야 한다. 물론 실무에서는 이보다 훨씬 더 복잡한 동적 쿼리들이 사용된다.

참고로 이후에 설명할 MyBatis의 가장 큰 장점은 SQL을 직접 사용할 때 동적 쿼리를 쉽게 작성할 수 있다는 점이다.


## JdbcTemplate 적용 3 - 구성과 실행

실제 코드가 동작하도록 구성하고 실행해보자.

### JdbcTemplateV1Config

```java
@RequiredArgsConstructor  
@Configuration  
public class JdbcTemplateV1Config {  
  
    private final DataSource dataSource;  
  
    @Bean  
    public ItemService itemService() {  
        return new ItemServiceV1(itemRepository());  
    }  
  
    @Bean  
    public ItemRepository itemRepository() {  
        return new JdbcTemplateItemRepositoryV1(dataSource);  
    }  
  
}
```

- `ItemRepository`구현체로 `JdbcTemplateItemRepositoryV1`이 사용되도록 했다. 이제 메모리 저장소가 아니라 실제 DB에 연결하는 JdbcTemplate이 사용된다.

### ItemServiceApplication - 변경 

```java
//@Import(MemoryConfig.class)  
@Import(JdbcTemplateV1Config.class)  
@SpringBootApplication(scanBasePackages = "hello.itemservice.web")  
public class ItemServiceApplication {  
  
    public static void main(String[] args) {  
       SpringApplication.run(ItemServiceApplication.class, args);  
    }  
  
    @Bean  
    @Profile("local")  
    public TestDataInit testDataInit(ItemRepository itemRepository) {  
       return new TestDataInit(itemRepository);  
    }    
}
```

`@Import(JdbcTemplateV1Config.class)`로 해당 config를 등록했다.

### 데이터베이스 접근 설정

```properties
spring.profiles.active=local  
spring.datasource.url=jdbc:h2:tcp://localhost/~/test  
spring.datasource.username=sa  
spring.datasource.password=
```

- 이렇게 설정만 하면 스프링 부트가 해당 설정을 사용해서 커넥션 풀과 DataSource , 트랜잭션 매니저를 스프링 빈으로 자동 등록한다.


![](https://i.imgur.com/RppsO95.png){: .align-center}

![](https://i.imgur.com/0ApBdEI.png){: .align-center}


testinit 때문에 itemA B가 계속 들어가긴 하는데 검색 등록 수정 삭제 잘 된다.

 