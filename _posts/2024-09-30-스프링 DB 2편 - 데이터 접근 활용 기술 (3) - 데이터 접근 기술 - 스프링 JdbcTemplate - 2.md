---
title: 스프링 DB 2편 - 데이터 접근 활용 기술 (3) - 데이터 접근 기술 - 스프링 JdbcTemplate - 2
aliases: 
tags:
  - spring
  - db
  - JdbcTemplate
categories:
  - spring
toc: true
toc_label: 목차
date: 2024-09-30
last_modified_at: 2024-09-30
---
>  인프런 스프링 DB 2편 - 데이터 접근 활용 기술편을 학습하고 정리한 내용 입니다.


## JdbcTemplate - 이름 지정 파라미터 1

### 순서대로 바인딩

JdbcTemplate을 기본으로 사용하면 파라미터를 순서대로 바인딩한다.

예를 들어서 

```java
String sql = "update item set item_name=?, price=?, quantity=? where id=?";  
template.update(sql,  
        itemName,  
        price,  
        quantity,  
        itemId);
```

여기서는 `itemName`, `price` , `quantity`가 SQL에 있는 `?`에 순서대로 바인딩 된다. 따라서 순서만 잘 지키면 문제가 될 것은 없다. 

그런데 문제는 변경 시점에 발생한다.

누군가 다음과 같이 SQL 코드의 순서를 변경했다고 가정해보자. (`price`와 `quantity`의 순서를 변경했다.)

```java
String sql = "update item set item_name=?, quantity=?, price=? where id=?"; template.update(sql, itemName, price, quantity, itemId);
```

이렇게 되면 다음과 같은 순서로 데이터가 바인딩 된다.

`item_name=itemName, quantity=price, price=quantity`


결과적으로 `price`와 `quantity`가 바뀌는 매우 심각한 문제가 발생한다. 이럴일이 없을 것 같지만, 실무에서는 파라미터가 10~20개가 넘어가는 일도 아주 많다. 그래서 미래에 필드를 추가하거나, 수정하면서 이런 문제가 충분히 발생할 수 있다.

버그 중에서 가장 고치기 힘든 버그는 데이터베이스에 데이터가 잘못 들어가는 버그다. 이것은 코드만 고치는 수준이 아니라 데이터베이스의 데이터를 복구해야 하기 때문에 버그를 해결하는데 들어가는 리소스가 어마어마하다.


**개발을 할 때는 코드를 몇줄 줄이는 편리함도 중요하지만, 모호함을 제거해서 코드를 명확하게 만드는 것이 유지보수 관점에서 매우 중요하다.**

이처럼 파라미터를 순서대로 바인딩 하는 것은 편리하기는 하지만, 순서가 맞지 않아서 버그가 발생할 수도 있으므로 주의해서 사용해야 한다.

### 이름 지정 바인딩

JdbcTemplate은 이런 문제를 보완하기 위해 `NamedParameterJdbcTemplate`라는 이름을 지정해서 파라미터를 바인딩 하는 기능을 제공한다.

지금부터 코드로 알아보자

`JdbcTemplateItemRepositoryV2`
```java
/**  
 * NamedParameterJdbcTemplate * SqlParameterSource * - BeanPropertySqlParameterSource * - MapSqlParameterSource * Map * * BeanPropertyRowMapper * */@Slf4j  
public class JdbcTemplateItemRepositoryV2 implements ItemRepository {  
  
    //private final JdbcTemplate template;  
    private final NamedParameterJdbcTemplate template;  
  
    public JdbcTemplateItemRepositoryV2(DataSource dataSource) {  
        this.template = new NamedParameterJdbcTemplate(dataSource);  
    }  
  
    @Override  
    public Item save(Item item) {  
        String sql = "insert into item(item_name, price, quantity)" +  
                " values(:itemName, :price, :quantity)";  
        KeyHolder keyHolder = new GeneratedKeyHolder();  
  
        SqlParameterSource param = new BeanPropertySqlParameterSource(item);  
        template.update(sql, param, keyHolder);  
  
        long key = keyHolder.getKey().longValue();  
        item.setId(key);  
        return item;  
    }  
  
    @Override  
    public void update(Long itemId, ItemUpdateDto updateParam) {  
        String sql = "update item " +  
                "set item_name = :itemName, price = :price, quantity = :quantity " +  
                "where id = :id";  
        SqlParameterSource param = new MapSqlParameterSource()  
                .addValue("itemName", updateParam.getItemName())  
                .addValue("price", updateParam.getPrice())  
                .addValue("quantity", updateParam.getQuantity())  
                .addValue("id", itemId);  
  
        template.update(sql, param);  
  
    }  
  
    @Override  
    public Optional<Item> findById(Long id) {  
        String sql = "select id, item_name, price, quantity from item where id = :id";  
  
        Map<String, Long> param = Map.of("id", id);  
        Item item = template.queryForObject(sql, param,  itemRowMapper());  
        return Optional.ofNullable(item);  
    }  
  
    private RowMapper<Item> itemRowMapper() {  
        return BeanPropertyRowMapper.newInstance(Item.class); //camel 변환 지원  
    }  
    @Override  
    public List<Item> findAll(ItemSearchCond cond) {  
        String itemName = cond.getItemName();  
        Integer maxPrice = cond.getMaxPrice();  
  
        SqlParameterSource param = new BeanPropertySqlParameterSource(cond);  
  
        String sql = "select id, item_name, price, quantity from item";  
  
        //동적 쿼리  
        if (StringUtils.hasText(itemName) || maxPrice != null) {  
            sql += " where";  
        }  
        boolean andFlag = false;  
        if (StringUtils.hasText(itemName)) {  
            sql += " item_name like concat('%',:itemName,'%')";  
            andFlag = true;  
        }  
        if (maxPrice != null) {  
            if (andFlag) {  
                sql += " and";  
            }  
            sql += " price <= :maxPrice";  
        }  
        log.info("sql={}", sql);  
  
        return template.query(sql, param, itemRowMapper());  
    }  
}
```

- `JdbcTemplateItemRepositoryV2`는 `ItemRepository`인터페이스를 구현했다.
- `this.template = new NamedParameterJdbcTemplate(dataSource)`
	- `NamedParameterJdbcTemplate`도 내부에 `dataSource`가 필요하다.
	- `JdbcTemplateItemRepositoryV2`생성자를 보면 의존관계 주입은 `dataSource`를 받고 내부에서 `NamedParameterJdbcTemplate`을 생성해서 가지고 있다. 스프링에서는 `JdbcTemplate`관련 기능을 사용할 때 관례상 이 방법을 많이 사용한다.
	- 물론 `NamedParameterJdbcTemplate`을 스프링 빈으로 직접 등록하고 주입받아도 된다.

**save()**
SQL에서 다음과 같이 `?` 대신에 `:파라미터이름`을 받는 것을 확인할 수 있다. 
```sql
insert into item (item_name, price, quantity) values (:itemName, :price, :quantity)
```

추가로 `NamedParameterJdbcTemplate`은 데이터베이스가 생성해주는 키를 매우 쉽게 조회하는 기능도 제공해준다.


## JdbcTemplate - 이름 지정 파라미터 2

### 이름 지정 파라미터

파라미터를 전달하려면 `Map`처럼 `key`, `value`데이터 구조를 만들어서 전달해야 한다.

여기서 `key`는 `:파리이터이름`으로 지정한, 파라미터의 이름이고 , `value`는 해당 파라미터의 값이 된다.

다음 코드를 보면 이렇게 만든 파라미터(`param`)를 전달하는 것을 확인할 수 있다.
- `template.update(sql, param, keyHolder);`

이름 지정 바인딩에서 자주 사용하는 파라미터의 종류는 크게 3가지가 있다.
- `Map`
- `SqlParameterSource`
	- `MapSqlParameterSource`
	- `BeanPropertySqlParameterSource`

#### 1. Map

단순히 `Map`을 사용한다. `findById()`코드에서 확인할 수 있다.

```java
Map param = Map.of("id", id); 
Item item = template.queryForObject(sql, param, itemRowMapper());
```



#### 2. MapSqlParameterSource

`Map`과 유사한데, SQL 타입을 지정할 수 있는 등 SQL에 좀 더 특화된 기능을 제공한다.
`SqlParameterSource`인터페이스의 구현체이다.
`MapSqlParameterSource`는 메서드 체인을 통해 편리한 사용법도 제공한다.

`update()` 코드에서 확인할 수 있다. 
```java
SqlParameterSource param = new MapSqlParameterSource()  
		.addValue("itemName", updateParam.getItemName())  
		.addValue("price", updateParam.getPrice())  
		.addValue("quantity", updateParam.getQuantity())  
		.addValue("id", itemId);  

template.update(sql, param)
```


#### 3. BeanPropertySqlParameterSource

자바빈 프로퍼티 규약을 통해서 자동으로 파라미터 객체를 생성한다.

예) (`getXxx() -> xxx, getItemName() -> itemName`)

예를 들어서 `getItemName()`, `getPrice()`가 있으면 다음과 같은 데이터를 자동으로 만들어낸다.
- `key=itemName, value=상품명 값`
- `key=price, value=가격 값`

`SqlParameterSource` 인터페이스의 구현체이다.

`save()`, `findAll()` 코드에서 확인할 수 있다. 
```java
SqlParameterSource param = new BeanPropertySqlParameterSource(item); 
KeyHolder keyHolder = new GeneratedKeyHolder(); 
template.update(sql, param, keyHolder);
```

- 여기서 보면 `BeanPropertySqlParameterSource`가 많은 것을 자동화 해주기 때문에 가장 좋아보이지만, `BeanPropertySqlParameterSource`를 항상 사용할 수 있는 것은 아니다.
- 예를 들어서 `update()`에서는 SQL에 `:id`를 바인딩 해야 하는데, `update()`에서 사용하는 `ItemUpdateDto`에는 `itemId`가 없다. 따라서 `BeanPropertySqlParameterSource`를 사용할 수 없고, 대신에 `MapSqlParameterSource`를 사용했다.


### BeanPropertyRowMapper

이번 코드에서 [V1](https://iamminseongkim.github.io/spring/%EC%8A%A4%ED%94%84%EB%A7%81-DB-2%ED%8E%B8-%EB%8D%B0%EC%9D%B4%ED%84%B0-%EC%A0%91%EA%B7%BC-%ED%99%9C%EC%9A%A9-%EA%B8%B0%EC%88%A0-(2)-%EB%8D%B0%EC%9D%B4%ED%84%B0-%EC%A0%91%EA%B7%BC-%EA%B8%B0%EC%88%A0-%EC%8A%A4%ED%94%84%EB%A7%81-JdbcTemplate-1/#jdbctemplate-%EC%A0%81%EC%9A%A9-1---%EA%B8%B0%EB%B3%B8)과 비교해서 변화된 부분이 하나 더 있다. 바로 `BeanPropertyRowMapper`를 사용한 것이다.

**JdbcTemplateItemRepositoryV1 - itemRowMapper()**
```java
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
```

**JdbcTemplateItemRepositoryV2 - itemRowMapper()**
```java
private RowMapper<Item> itemRowMapper() {  
    return BeanPropertyRowMapper.newInstance(Item.class); //camel 변환 지원  
}
```

`BeanPropertyRowMapper`는 `ResultSet`의 결과를 받아서 자바빈 규약에 맞추어 데이터를 변환한다.

예를 들어서 데이터베이스에서 조회한 결과가 `select id, price`라고 하면 다음과 같은 코드를 작성해준다. (실제로는 리플렉션 같은 기능을 사용한다.) 

```java
Item item = new Item(); 
item.setId(rs.getLong("id")); 
item.setPrice(rs.getInt("price"));
```

데이터베이스에서 조회한 결과 이름을 기반으로 `setId()`, `setPrice()`처럼 자바빈 프로퍼티 규약에 맞춘 메서드를 호출하는 것이다.

#### 별칭

그런데 `select item_name`의 경우 `setItem_name()`이라는 메서드가 없기 때문에 골치가 아프다.

이런 경우 개발자가 조회 SQL을 다음과 같이 고치면 된다.

`select item_name as itemName`

별칭 `as`를 사용해서 SQL 조회 결과의 이름을 변경하는 것이다. 실제로 이 방법은 자주 사용된다. 특히 데이터베이스 컬럼 이름과 객체 이름이 완전히 다를 때 문제를 해결할 수 있다. 예를 들어서 데이터베이스에는 `member_name`이라고 되어 있는데 객체에 `username`이라고 되어 있다면 다음과 같이 해결할 수 있다.

`select member_name as username`

이렇게 데이터베이스 컬럼 이름과 객체의 이름이 다를 때 별칭(`as`)을 사용해서 문제를 많이 해결한다.

`JdbcTemplate`은 물론이고, `MyBatis`같은 기술에서도 자주 사용된다.

#### 관례의 불일치

자바 객체는 카멜(`camelCase`) 표기법을 사용한다. `itemName`처럼 중간에 낙타 봉이 올라와 있는 표기법이다.

반면에 관계형 데이터베이스에서는 주로 언더스코어를 사용하는 `snake_case`표기법을 사용한다. `item_name`처럼 중간에 언더스코어를 사용하는 표기법이다.

이 부분을 관례로 많이 사용하다 보니 `BeanPropertyRowMapper`는 언더스코어 표기법을 카멜로 자동 변환해준다. 따라서 `select item_name`으로 조회해도 `setItemName()`에 문제 없이 값이 들어간다.

정리하면 `snake_case`는 자동으로 해결되니 그냥 두면 되고, 컬럼 이름과 객체 이름이 완전히 다른 경우에는 조회 SQL에서 별칭을 사용하면 된다.



## JdbcTemplate - 이름 지정 파라미터 3

이제 이름 지정 파라미터를 사용하도록 구성하고 실행해보자.

`JdbcTemplateV2Config`
```java
@RequiredArgsConstructor  
@Configuration  
public class JdbcTemplateV2Config {  
  
    private final DataSource dataSource;  
  
    @Bean  
    public ItemService itemService() {  
        return new ItemServiceV1(itemRepository());  
    }  
  
    @Bean  
    public ItemRepository itemRepository() {  
        return new JdbcTemplateItemRepositoryV2(dataSource);  
    }  
  
}
```

`ItemServiceApplication - 변경`
```java
//@Import(MemoryConfig.class)  
//@Import(JdbcTemplateV1Config.class)  
@Import(JdbcTemplateV2Config.class)  
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


![](https://i.imgur.com/u7FIZP0.png){: .align-center}

조회 등록 수정 삭제가 잘 작동한다.



## JdbcTemplate - SimpleJdbcInsert


JdbcTemplate은 INSERT SQL를 직접 작성하지 않아도 되도록 `SimpleJdbcInsert`라는 편리한 기능을 제공한다.

`JdbcTemplateItemRepositoryV3`
```java
/**  
 * SimpleJdbcInsert 
 */
 @Slf4j  
public class JdbcTemplateItemRepositoryV3 implements ItemRepository {  
  
    //private final JdbcTemplate template;  
    private final NamedParameterJdbcTemplate template;  
    private final SimpleJdbcInsert jdbcInsert;  
  
    public JdbcTemplateItemRepositoryV3(DataSource dataSource) {  
        this.template = new NamedParameterJdbcTemplate(dataSource);  
        this.jdbcInsert = new SimpleJdbcInsert(dataSource)  
                .withTableName("item")  
                .usingGeneratedKeyColumns("id");  
                //.usingColumns("item_name", "price", "quantity"); //생략 가능  
  
    }  
  
    @Override  
    public Item save(Item item) {  
        SqlParameterSource param = new BeanPropertySqlParameterSource(item);  
        Number key = jdbcInsert.executeAndReturnKey(param);  
        item.setId(key.longValue());  
        return item;  
    }  
	...
}
```

- `JdbcTemplateItemRepositoryV3`은 `ItemRepository`인터페이스를 구현했다.
- `this.jdbcInsert = new SimpleJdbcInsert(dataSource)` : 생성자를 보면 의존관계 주입은 `dataSource`를 받고 내부에서 `SimpleJdbcInsert`을 생성해서 가지고 있다. 스프링에서는 `JdbcTemplate`관련 기능을 사용할 때 관례상 이 방법을 많이 사용한다.
	- 물론 `SimpleJdbcInsert`을 스프링 빈으로 직접 등록하고 주입받아도 된다.

### SimpleJdbcInsert


```java
this.jdbcInsert = new SimpleJdbcInsert(dataSource)  
                .withTableName("item")  
                .usingGeneratedKeyColumns("id");  
                //.usingColumns("item_name", "price", "quantity"); //생략 가능
```

- `withTableName` : 데이터를 저장할 테이블 명을 지정한다.
- `usingGeneratedKeyColumns` : `key`를 생성하는 PK 컬럼 명을 지정한다.
- `usingColumns` : INSERT SQL에 사용할 컬럼을 지정한다. 특정 값만 저장하고 싶을 때 사용한다. 생략할 수 있다.

`SimpleJdbcInsert`는 생성 시점에 데이터베이스 테이블의 메타 데이터를 조회한다. 따라서 어떤 컬럼이 있는지 확인 할 수 있으므로 `usingColumns`을 생략할 수 있다. 만약 특정 컬럼만 지정해서 저장하고 싶다면 `usingColumns`를 사용하면 된다.


### save()
`jdbcInsert.executeAndReturnKey(param)`을 사용해서 INSERT SQL을 실행하고, 생성된 키 값도 매우 편리하게 조회할 수 있다. 
```java
public Item save(Item item) {  
	SqlParameterSource param = new BeanPropertySqlParameterSource(item);  
	Number key = jdbcInsert.executeAndReturnKey(param);  
	item.setId(key.longValue());  
	return item;  
}
```

나머지는 코드 부분은 기존과 같다.

이제 config등록하고 실행해보자.


![](https://i.imgur.com/WSNleXa.png){: .align-center}

잘 작동한다.

## JdbcTemplate 기능 정리

JdbcTemplate의 기능을 간단히 정리해보자.

### 주요 기능

JdbcTemplate이 제공하는 주요 기능은 다음과 같다.

- `JdbcTemplate`
	- 순서 기반 파라미터 바인딩을 지원한다.
- `NamedParameterJdbcTemplate`
	- 이름 기반 파라미터 바인딩을 지원한다. (권장)
- `SimpleJdbcInsert`
	- INSERT SQL을 편리하게 사용할 수 있다.
- `SimpleJdbcCall`
	- 스토어드 프로시저를 편리하게 호출할 수 있다.

> 참고 : 스토어드 프로시저를 사용하기 위한 `SimpleJdbcCall`에 대한 자세한 내용은 다음 [스프링 공식 메뉴얼](https://docs.spring.io/spring-framework/reference/data-access/jdbc/simple.html#jdbc-simple-jdbc-call-1)을 참고하자. 


### JdbcTemplate 사용법 정리

JdbcTemplate에 대한 사용법은 스프링 공식 메뉴얼에 자세히 소개되어 있다. 여기서는 스프링 공식 메뉴얼이 제공하는 예제를 통해 JdbcTemplate의 기능을 간단히 정리해보자.

> **참고** : [스프링 JdbcTemplate 사용 방법 공식 메뉴얼](https://docs.spring.io/spring-framework/reference/data-access/jdbc/core.html#jdbc-JdbcTemplate)


### 조회

#### 단건 조회 - 숫자 문자
```java
int rowCount = jdbcTemplate.queryForObject("select count(*) from t_actor", Integer.class);
```

하나의 로우를 조회할 때는 `queryForObject()`를 사용하면 된다. 지금처럼 조회 대상이 객체가 아니라 단순 데이터하나라면 타입을 `Integer.class`, `String.class`와 같이 지정해주면 된다.


#### 단건 조회 - 객체 조회 
```java
Actor actor = jdbcTemplate.queryForObject(  
       "select first_name, last_name from t_actor where id = ?",  
       (resultSet, rowNum) -> {  
          Actor newActor = new Actor();  
          newActor.setFirstName(resultSet.getString("first_name"));  
          newActor.setLastName(resultSet.getString("last_name"));  
          return newActor;  
       },  
       1212L);
```

객체 하나를 조회한다. 결과를 객체로 매핑해야 하므로 `RowMapper`를 사용해야 한다. 여기서는 람다를 사용했다.

#### 목록 조회 - 객체 
```java
List<Actor> actors = jdbcTemplate.query(  
       "select first_name, last_name from t_actor",  
       (resultSet, rowNum) -> {  
          Actor actor = new Actor();  
          actor.setFirstName(resultSet.getString("first_name"));  
          actor.setLastName(resultSet.getString("last_name"));  
          return actor;  
       });
```


여러 로우를 조회할 때는 `query()`를 사용하면 된다. 결과를 리스트로 반환한다.

결과를 객체로 매핑해야 하므로 `RowMapper`를 사용해야 한다. 여기서는 람다를 사용했다.


```java
private final RowMapper<Actor> actorRowMapper = (resultSet, rowNum) -> {  
    Actor actor = new Actor();  
    actor.setFirstName(resultSet.getString("first_name"));  
    actor.setLastName(resultSet.getString("last_name"));  
    return actor;  
};  
public List<Actor> findAllActors() {  
    return this.jdbcTemplate.query("select first_name, last_name from t_actor",  
          actorRowMapper);  
}
```

여기서는 `RowMapper`를 분리했다. 이렇게 하면 여러 곳에서 재사용 할 수 있다.


### 변경(INSERT, UPDATE, DELETE)

데이터를 변경할 때는 `jdbcTemplate.update()`를 사용하면 된다. 참고로 `int`반환값을 반환하는데, SQL 실행결과에 영향받은 로우 수를 반환한다.

#### 등록

```java
jdbcTemplate.update(  
	 "insert into t_actor (first_name, last_name) values (?, ?)",  
        "Leonor", "Watling");
```

#### 수정 

```java
jdbcTemplate.update(  
	 "update t_actor set last_name = ? where id = ?",  
	    "Banjo", 5276L);
```

#### 삭제

```java
jdbcTemplate.update(  
	 "delete from t_actor where id = ?",  
	 Long.valueOf(actorId));
```

### 기타 기능

임의의 SQL을 실행할 때는 `execute()`를 사용하면 된다. 테이블을 생성하는 DDL에 사용할 수 있다.

#### DDL

```java
jdbcTemplate.execute("create table mytable (id integer, name varchar(100))");
```

#### 스토어드 프로시저 호출 
```java
jdbcTemplate.update( 
	"call SUPPORT.REFRESH_ACTORS_SUMMARY(?)", 
	Long.valueOf(unionId));
```



## 정리

실무에서 가장 간단하고 실용적인 방법으로 SQL을 사용하려면 JdbcTemplate을 사용하면 된다. JPA와 같은 ORM 기술을 사용하면서 동시에 SQL을 직접 작성해야 할 때가 있는데, 그때도 JdbcTemplate을 함께 사용하면 된다.

그런데 JdbcTemplate의 최대 단점이 있는데, 바로 동적 쿼리 문제를 해결하지 못한다는 점이다. 그리고 SQL을 자바코드로 작성하기 때문에 SQL 라인이 코드를 넘어갈 때 마다 문자 더하기를 해주어야 하는 단점도 있다.

동적 쿼리 문제를 해결하면서 동시에 SQL도 편리하게 작성할 수 있게 도와주는 기술이 바로 `MyBatis`이다.