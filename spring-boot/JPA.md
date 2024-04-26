# JPA

## Intro

## 依赖&配置

```xml
//in pom.xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>
```

```properties
#in application properties

#设置本地数据库的url
spring.datasource.url=jdbc:mysql://localhost:3306/<数据库>?serverTimezone=UTC&useUnicode=true&characterEncoding=utf-8&useSSL=true
#用户名&密码
spring.datasource.username=root
spring.datasource.password=root
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
#jpa配置
#spring.jpa.properties.hibernate.hbm2ddl.auto： 配置实体类维护数据库表结构的具体行为
#update：启动时会检查数据库表结构，并根据实体类的定义进行更新。它不会删除已存在的表，也不会删除或修改现有数据，但会新增或修改表的字段、索引等。
#create：启动时会删除现有的数据库表，然后根据实体类的定义重新创建表结构。这会删除所有数据。
spring.jpa.properties.hibernate.hbm2ddl.auto=create

spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL5InnoDBDialect
#SQL 输出
#spring.jpa.show-sql ：在操作的时候在控制台打印真实的sql语句，方便调试。
spring.jpa.show-sql=true
#format 下 SQL 进输出
spring.jpa.properties.hibernate.format_sql=true
```

### Hint

* 加入数据库依赖的时候需要配置好数据库，否则可能会无法正常启动项目
* 配置了JPA的时候要编写相应的实体类文件才能正常启动项目(如何判断已经正确整合JPA了：设置好依赖和配置、创建一个实体类并尝试启动项目，启动成功说明已经整合成功)

## 实体类

### 作用

* 定义实体类，项目启动时系统会根据实体类来创建对应于实体类的**数据库中的表**

### 重要注解

* **@Entity** | 声明这个类是一个实体类，会自动在数据库中创建对应于这个**实体类**的**一张表**
* **@Table** | 描述这个实体类对应的数据库中表的信息(表名称等)，没有声明的话，默认表明和实体类的名称一样
* **@Id** | 表示这一个属性是这个实体类的唯一标识，相当于数据库中表里面的**主键**，用来**唯一标识**每一行**数据实体**
* **@GeneratedValue** | 设置数据的生成机制
* **@Column** |声明实体类中的属性是对应于数据库中表的一个字段(也就是要创建的数据库表里面要有**与这个属性对应的存储字段**)，类型默认是一样的，字符串长度可以进行指定
  * nullable 设置字段不能为空
  * unique 设置字段必须唯一，如果对某一字段设置了unique，在表中插入已经存在相同内容，就会导致报错！
  * length 设置字符串长度
* **@Data** | 用来自动为类中的属性**生成getter和setter**，减少了大量的无意义重复工作(由Lombok库提供)

### 配置关联

#### 级联操作

​	级联操作发生在实体与实体之间。当一个实体A(主实体)配置了对另一个实体B(关联实体)的级联，那么对A执行的某种操作(保存、更新、删除等等)，就会影响到被级联的实体B

​	cascade属性用来指定级联行为

#### 加载策略

​	当加载主实体的时候，在一些情况下我们希望关联实体也立马被一起加载；而有些情况因为关联实体过于庞大，我们希望只有当关联实体被使用到的时候才去真正的加载。

* EAGER 急加载

  当加载主实体的时候，就会同时加载关联实体

* LAZY 懒加载

  当加载主实体的时候，不会立马加载关联实体，只有当明确访问关联实体时，关联实体才会被加载

​	fetch属性用来指定加载行为

#### 关系

##### 一对一

* @OneToOne：一对一的关系
* @JoinColumn：JoinColumn用来在一端设置外键

​	**配置了JoinColumn的实体类，对应的表中就会有一个外键用来指示关联的那个实体类**

##### 一对多

* @OneToMany & @ManyToOne
* @JoinColumn：被设置JoinColumn的一端会生成对另一端的外键，一般放在Many端
* @MappedBy：放在One端

​	**配置了JoinColumn的实体类，对应的表中就会有一个外键用来指示关联的那个实体类**

##### 多对多

* @ManyToMany

  ManyToMany指明了两个实体类A、B之间，一个A的实例可以对应多个B，一个B的实例可以对应多个A

* @JoinTable:设置多对多关系的关联表

## Repository类

Repository是JPA提供的数据访问接口，只需要继承JpaRepository类来建立一个接口，就可以利用许多JpaRepository已经内置好的方法来对某个实体类生成的数据库中的表进行查询，此外，还可以在这个接口中自定义方法利用SQL(数据库语言)进行查询，最重要的是，这些都只需要在接口中声明，不需要我们去写什么具体的实现逻辑。

### Repository类创建要求

* 命名：一般来说Repository命名要对应于要查询的实体类的名称，比如查询实体类User的repository接口一般就是命名为UserRepository，见名知意，这并非强制要求，但这是一个**很好的工程实践**
* 继承：继承Spring Data JPA提供的JpaRepository接口
* 指定泛型参数：继承的JpaRepository接口需要传入参数来指明要查询的实体类(在这里repository与实体类发生了关联)

```java
// 注解，声明这是一个repo接口
//  ↓
@Repository
//
//                                                  实体类   实体类主键类型
//                                                    ↓       ↓
public interface UserRepository extends JpaRepository<User, Long> {
//        ↑                        ↑             ↑
//        |                        |         Spring Data JPA提供的进行数据访问的接口
//        |                        |
//      声明一个接口               继承
//可以自定义查询方法
}
```

## Controller类

### 返回Json

* 利用ResponseEntity<Object>来返回一个对象，而声明了RequestMapping会转换为返回Json

```java
@PostMapping("/create")
public ResponseEntity<Object> createNewUser(@RequestBody User user){
    user.setArticle_num(0);
    user.setFollow_num(0);
    userRepository.save(user);
    
    return ResponseEntity.ok().body("{\"status\":\"ok\"}");
}
```

