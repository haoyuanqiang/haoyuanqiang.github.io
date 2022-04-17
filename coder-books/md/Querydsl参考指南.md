
[TOC]

## 前言

Querydsl 是一个框架，可以构建静态类型的 SQL 类查询。无需将查询编写为内联字符串或将它们外部化为 XML 文件，它们可以通过 Querydsl 之类的流畅 API 构建。

与简单字符串相比，使用 fluent API 的好处是：

- IDE 中的代码完成
- 几乎不允许语法上无效的查询
- 可以安全地引用域类型和属性
- 更好地重构领域类型的变化


## 1. 简介

### 1.1. 背景

Querydsl 的诞生是出于以类型安全的方式维护 HQL 查询的需要。HQL 查询的增量构造需要字符串连接，导致代码难以阅读。通过纯字符串对域类型和属性的不安全引用是基于字符串的 HQL 构造的另一个问题。

随着领域模型的变化，类型安全为软件开发带来了巨大的好处。域更改直接反映在查询中，查询构造中的自动完成使查询构造更快、更安全。

Hibernate 的 HQL 是 Querydsl 的第一个目标语言，但现在它支持 JPA、JDO、JDBC、Lucene、Hibernate Search、MongoDB、Collections 和 RDFBean 作为后端。

### 1.2. 原则

*类型安全*是 Querydsl 的核心原则。查询是根据生成的查询类型构建的，这些查询类型反映了您的域类型的属性。函数/方法调用也以完全类型安全的方式构造。

*一致性*是另一个重要原则。查询路径和操作在所有实现中都是相同的，查询接口也有一个通用的基本接口。

要了解 Querydsl 查询和表达式类型的表达能力，请访问 javadocs 并探索 `com.querydsl.core.Query`、`com.querydsl.core.Fetchable` 和 `com.querydsl.core.types.Expression `。

## 2. 教程

我们提供 Querydsl 主要后端的集成指南，而不是一般的入门指南。

### 2.1. 查询 JPA

Querydsl 定义了一种通用的静态类型语法，用于在持久的域模型数据之上进行查询。JDO 和 JPA 是 Querydsl 的主要集成技术。本指南描述了如何将 Querydsl 与 JPA 结合使用。

JPA 的 Querydsl 是 JPQL 和 Criteria 查询的替代方案。它以完全类型安全的方式结合了 Criteria 查询的动态特性和 JPQL 的表现力以及所有这些。

#### 2.1.1. Maven 集成

将以下依赖项添加到您的 Maven 项目中：

```xml
<dependency>
  <groupId>com.querydsl</groupId>
  <artifactId>querydsl-apt</artifactId>
  <version>${querydsl.version}</version>
  <scope>provided</scope>
</dependency>

<dependency>
  <groupId>com.querydsl</groupId>
  <artifactId>querydsl-jpa</artifactId>
  <version>${querydsl.version}</version>
</dependency>

<dependency>
  <groupId>org.slf4j</groupId>
  <artifactId>slf4j-log4j12</artifactId>
  <version>1.6.1</version>
</dependency>
```

现在，配置 Maven APT 插件：

```xml
<project>
  <build>
  <plugins>
    ...
    <plugin>
      <groupId>com.mysema.maven</groupId>
      <artifactId>apt-maven-plugin</artifactId>
      <version>1.1.3</version>
      <executions>
        <execution>
          <goals>
            <goal>process</goal>
          </goals>
          <configuration>
            <outputDirectory>target/generated-sources/java</outputDirectory>
            <processor>com.querydsl.apt.jpa.JPAAnnotationProcessor</processor>
          </configuration>
        </execution>
      </executions>
    </plugin>
    ...
  </plugins>
  </build>
</project>
```

JPAAnnotationProcessor 查找使用 javax.persistence.Entity 注释注释的域类型并为它们生成查询类型。

如果你在你的域类型中使用 Hibernate 注释，你应该使用 APT 处理器 `com.querydsl.apt.hibernate.HibernateAnnotationProcessor` 代替。

运行全新安装，您将在 target/generated-sources/java 中生成查询类型。

如果您使用 Eclipse，请运行 mvn eclipse:eclipse 以更新您的 Eclipse 项目以包含 target/generated-sources/java 作为源文件夹。

现在您可以构建 JPA 查询实例和查询域模型的实例。

#### 2.1.2. 蚂蚁集成

将来自 full-deps 包的 jar 文件放在类路径中，并使用以下任务生成 Querydsl 代码：

```xml
    <!-- 基于 APT 的代码生成 -->
    <javac srcdir="${src}" classpathref="cp">
      <compilerarg value="-proc:only"/>
      <compilerarg value="-processor"/>
      <compilerarg value="com.querydsl.apt.jpa.JPAAnnotationProcessor"/>
      <compilerarg value="-s"/>
      <compilerarg value="${generated}"/>
    </javac>

    <!-- 编译 -->
    <javac classpathref="cp" destdir="${build}">
      <src path="${src}"/>
      <src path="${generated}"/>
    </javac>
```

将 *src* 替换为您的主源文件夹，将 *generated* 替换为生成的源文件夹，将 *build* 替换为目标文件夹。

#### 2.1.3. 在 Roo 中使用 Querydsl JPA

如果您将 Querydsl JPA 与 Spring Roo 一起使用，您可以将 `com.querydsl.apt.jpa.JPAAnnotationProcessor` 替换为 `com.querydsl.apt.roo.RooAnnotationProcessor`，它将处理 `@RooJpaEntity` 和 `@RooJpaActiveRecord` 注释类`@Entity` 注释类。

基于 APT 的代码生成不适用于 AspectJ IDT。

#### 2.1.4. 从 hbm.xml 文件生成模型

如果您使用带有基于 XML 的配置的 Hibernate，您可以使用 XML 元数据来创建您的 Querydsl 模型。

`com.querydsl.jpa.codegen.HibernateDomainExporter` 提供了以下功能：

```java
HibernateDomainExporter exporter = new HibernateDomainExporter(
  "Q",                     // 名称前缀
  new File("target/gen3"), // 目标文件夹
  configuration);          // org.hibernate.cfg.Configuration 的实例

exporter.export();
```

HibernateDomainExporter 需要在域类型可见的类路径中执行，因为属性类型是通过反射解析的。

所有 JPA 注释都将被忽略，但会考虑 @QueryInit 和 @QueryType 等 Querydsl 注释。

#### 2.1.5. 使用查询类型

要使用 Querydsl 创建查询，您需要实例化变量和 Query 实现。我们将从变量开始。

假设您的项目具有以下域类型：

```java
@Entity
public class Customer {
    private String firstName;
    private String lastName;

    public String getFirstName() {
        return firstName;
    }

    public String getLastName() {
        return lastName;
    }

    public void setFirstName(String fn) {
        firstName = fn;
    }

    public void setLastName(String ln) {
        lastName = ln;
    }
}
```

Querydsl 将生成一个简单名称为 QCustomer 的查询类型到与 Customer 相同的包中。QCustomer 可以用作 Querydsl 查询中的静态类型变量，作为 Customer 类型的代表。

QCustomer 有一个可以作为静态字段访问的默认实例变量：

```java
QCustomer customer = QCustomer.customer;
```

或者，您可以像这样定义自己的客户变量：

```java
QCustomer customer = new QCustomer("myCustomer");
```

#### 2.1.6. 查询

Querydsl JPA 模块同时支持 JPA 和 Hibernate API。

要使用 JPA API，您可以使用 `JPAQuery` 实例进行查询，如下所示：

```java
// 其中 entityManager 是 JPA EntityManager
JPAQuery<?> query = new JPAQuery<Void>(entityManager);
```

如果您使用的是 Hibernate API，则可以像这样实例化一个 `HibernateQuery`：

```java
// 其中 session 是一个 Hibernate 会话
HibernateQuery<?> query = new HibernateQuery<Void>(session);
```

`JPAQuery` 和 `HibernateQuery` 都实现了 `JPQLQuery` 接口。

对于本章的示例，查询是通过“JPAQueryFactory”实例创建的。`JPAQueryFactory` 应该是获取 `JPAQuery` 实例的首选选项。

对于 Hibernate API `HibernateQueryFactory` 可以使用

要检索名字为 Bob 的客户，您将构建如下查询：

```java
QCustomer customer = QCustomer.customer;
Customer bob = queryFactory.selectFrom(customer)
  .where(customer.firstName.eq("Bob"))
  .fetchOne();
```

selectFrom 调用定义了查询源和投影，where 部分定义了过滤器，而 fetchOne 告诉 Querydsl 返回单个元素。容易，对吧？

要创建具有多个来源的查询，您可以使用如下查询：

```java
QCustomer customer = QCustomer.customer;
QCompany company = QCompany.company;
query.from(customer, company);
```

并使用多个过滤器像这样使用它

```java
queryFactory.selectFrom(customer)
    .where(customer.firstName.eq("Bob"), customer.lastName.eq("Wilson"));
```

或者像这样

```java
queryFactory.selectFrom(customer)
    .where(customer.firstName.eq("Bob").and(customer.lastName.eq("Wilson")));
```

在原生 JPQL 形式中，查询将这样编写：

```sql
select customer from Customer as customer
where customer.firstName = "Bob" and customer.lastName = "Wilson"
```

如果您想通过“或”组合过滤器，请使用以下模式

```java
queryFactory.selectFrom(customer)
    .where(customer.firstName.eq("Bob").or(customer.lastName.eq("Wilson")));
```

#### 2.1.7. 使用联结

Querydsl 在 JPQL 中支持以下连接变体：内联结、联结、左联结和右联结。连接使用是类型安全的，并遵循以下模式：

```java
QCat cat = QCat.cat;
QCat mate = new QCat("mate");
QCat kitten = new QCat("kitten");
queryFactory.selectFrom(cat)
    .innerJoin(cat.mate, mate)
    .leftJoin(cat.kittens, kitten)
    .fetch();
```

查询的原生 JPQL 版本将是

```sql
select cat from Cat as cat
inner join cat.mate as mate
left outer join cat.kittens as kitten
```

另一个例子

```java
queryFactory.selectFrom(cat)
    .leftJoin(cat.kittens, kitten)
    .on(kitten.bodyWeight.gt(10.0))
    .fetch();
```

使用以下 JPQL 版本

```sql
select cat from Cat as cat
left join cat.kittens as kitten
on kitten.bodyWeight > 10.0
```

#### 2.1.8. 一般用法

像这样使用 JPQLQuery 接口的级联方法

*select:* 设置查询的投影。（如果通过查询工厂创建则不需要）

*from:* 在此处添加查询源。

*innerJoin, join, leftJoin, rightJoin, on:* 使用这些结构添加连接元素。对于连接方法，第一个参数是连接源，第二个是目标（别名）。

*where:* 添加查询过滤器，以可变参数形式通过逗号分隔或通过 and 运算符级联。

*groupBy:* 以可变参数形式添加分组参数。

*have:* 添加具有“group by”分组的过滤器作为谓词表达式的 varags 数组。

*orderBy:* 将结果的顺序添加为顺序表达式的可变参数数组。对数字、字符串和其他可比较的表达式使用 asc() 和 desc() 来访问 OrderSpecifier 实例。

*limit,offset,restrict:* 设置结果的分页。最大结果的限制，跳过行的偏移量和一次调用中定义两者的限制。

#### 2.1.9. Ordering

声明 Ordering 的语法是

```java
QCustomer customer = QCustomer.customer;
queryFactory.selectFrom(customer)
    .orderBy(customer.lastName.asc(), customer.firstName.desc())
    .fetch();
```

相当于下面的原生 JPQL

```sql
select customer from Customer as customer
order by customer.lastName asc, customer.firstName desc
```

#### 2.1.10. Grouping

Grouping 可以按以下形式进行

```java
queryFactory.select(customer.lastName).from(customer)
    .groupBy(customer.lastName)
    .fetch();
```

相当于下面的原生 JPQL

```sql
select customer.lastName
from Customer as customer
group by customer.lastName
```

#### 2.1.11. Delete 子句

Querydsl JPA 中的删除子句遵循简单的 delete-where-execute 形式。这里有些例子：

```java
QCustomer customer = QCustomer.customer;
//删除所有客户
queryFactory.delete(customer).execute();
// 删除所有等级小于 3 的客户
queryFactory.delete(customer).where(customer.level.lt(3)).execute();
```

where 调用是可选的，execute 调用执行删除并返回已删除实体的数量。

JPA 中的 DML 子句不考虑 JPA 级级联规则，也不提供细粒度的二级缓存交互。

#### 2.1.12. Update 子句

Querydsl JPA 中的更新子句遵循简单的更新集/位置执行形式。这里有些例子：

```java
QCustomer customer = QCustomer.customer;
// 将名为 Bob 的客户重命名为 Bobby
queryFactory.update(customer).where(customer.name.eq("Bob"))
    .set(customer.name, "Bobby")
    .execute();
```

集合调用以 SQL 更新样式定义属性更新，执行调用执行更新并返回更新实体的数量。

JPA 中的 DML 子句不考虑 JPA 级级联规则，也不提供细粒度的二级缓存交互。

#### 2.1.13. 子查询

要创建子查询，您可以使用“JPAExpressions”的静态工厂方法并通过 from、where 等定义查询参数。

```java
QDepartment department = QDepartment.department;
QDepartment d = new QDepartment("d");
queryFactory.selectFrom(department)
    .where(department.size.eq(
        JPAExpressions.select(d.size.max()).from(d)))
     .fetch();
```

另一个例子

```java
QEmployee employee = QEmployee.employee;
QEmployee e = new QEmployee("e");
queryFactory.selectFrom(employee)
    .where(employee.weeklyhours.gt(
        JPAExpressions.select(e.weeklyhours.avg())
            .from(employee.department.employees, e)
            .where(e.manager.eq(employee.manager))))
    .fetch();
```

#### 2.1.14. 公开原始查询

如果您需要在执行查询之前调整原始查询，您可以像这样公开它：

```java
Query jpaQuery = queryFactory.selectFrom(employee).createQuery();
// ...
List results = jpaQuery.getResultList();
```

#### 2.1.15。在 JPA 查询中使用原生 SQL

Querydsl 通过 JPASQLQuery 类支持 JPA 中的原生 SQL。

要使用它，您必须为您的 SQL 模式生成 Querydsl 查询类型。例如，这可以通过以下 Maven 配置来完成：

```xml
<project>
  <build>
    <plugins>
      ...
      <plugin>
        <groupId>com.querydsl</groupId>
        <artifactId>querydsl-maven-plugin</artifactId>
        <version>${querydsl.version}</version>
        <executions>
          <execution>
            <goals>
              <goal>export</goal>
            </goals>
          </execution>
        </executions>
        <configuration>
          <jdbcDriver>org.apache.derby.jdbc.EmbeddedDriver</jdbcDriver>
          <jdbcUrl>jdbc:derby:target/demoDB;create=true</jdbcUrl>
          <packageName>com.mycompany.mydomain</packageName>
          <targetFolder>${project.basedir}/target/generated-sources/java</targetFolder>
        </configuration>
        <dependencies>
          <dependency>
            <groupId>org.apache.derby</groupId>
            <artifactId>derby</artifactId>
            <version>${derby.version}</version>
          </dependency>
        </dependencies>
      </plugin>
      ...
    </plugins>
  </build>
</project>
```

当查询类型成功生成到您选择的位置后，您可以在查询中使用它们。

单列查询：

```java
// 序列化模板
SQLTemplates templates = new DerbyTemplates();
// 查询类型（S* 代表 SQL，Q* 代表域类型）
SAnimal cat = new SAnimal("cat");
SAnimal mate = new SAnimal("mate");
QCat catEntity = QCat.cat;

JPASQLQuery<?> query = new JPASQLQuery<Void>(entityManager, templates);
List<String> names = query.select(cat.name).from(cat).fetch();
```

如果您在查询中混合使用实体（例如 QCat）和表（例如 SAnimal）引用，您需要确保它们使用相同的变量名。SAnimal.animal 的变量名为“animal”，因此使用了一个新实例 (new SAnimal("cat"))。

另一种模式可能是

```java
QCat catEntity = QCat.cat;
SAnimal cat = new SAnimal(catEntity.getMetadata().getName());
```

查询多列：

```java
query = new JPASQLQuery<Void>(entityManager, templates);
List<Tuple> rows = query.select(cat.id, cat.name).from(cat).fetch();
```

查询所有列：

```java
List<Tuple> rows = query.select(cat.all()).from(cat).fetch();
```

在 SQL 中查询，但作为实体项目：

```java
query = new JPASQLQuery<Void>(entityManager, templates);
List<Cat> cats = query.select(catEntity).from(cat).orderBy(cat.name.asc()).fetch();
```

使用连接查询：

```java
query = new JPASQLQuery<Void>(entityManager, templates);
cats = query.select(catEntity).from(cat)
    .innerJoin(mate).on(cat.mateId.eq(mate.id))
    .where(cat.dtype.eq("Cat"), mate.dtype.eq("Cat"))
    .fetch();
```

查询并投影到 DTO：

```java
query = new JPASQLQuery<Void>(entityManager, templates);
List<CatDTO> catDTOs = query.select(Projections.constructor(CatDTO.class, cat.id, cat.name))
    .from(cat)
    .orderBy(cat.name.asc())
    .fetch();
```

如果您使用的是 Hibernate API 而不是 JPA API，那么请改用“HibernateSQLQuery”。



### 2.2. 查询 JDO

Querydsl 定义了一种通用的静态类型语法，用于在持久的域模型数据之上进行查询。JDO 和 JPA 是 Querydsl 的主要集成技术。本指南描述了如何将 Querydsl 与 JDO 结合使用。

#### 2.2.1. Maven 集成

将以下依赖项添加到您的 Maven 项目中：

```xml
<dependency>
  <groupId>com.querydsl</groupId>
  <artifactId>querydsl-apt</artifactId>
  <version>${querydsl.version}</version>
  <scope>provided</scope>
</dependency>

<dependency>
  <groupId>com.querydsl</groupId>
  <artifactId>querydsl-jdo</artifactId>
  <version>${querydsl.version}</version>
</dependency>

<dependency>
  <groupId>org.slf4j</groupId>
  <artifactId>slf4j-log4j12</artifactId>
  <version>1.6.1</version>
</dependency>
```

现在，配置生成 Querydsl 使用的查询类型的 Maven APT 插件：

```xml
<project>
  <build>
    <plugins>
      ...
      <plugin>
        <groupId>com.mysema.maven</groupId>
        <artifactId>apt-maven-plugin</artifactId>
        <version>1.1.3</version>
        <executions>
          <execution>
            <goals>
              <goal>process</goal>
            </goals>
            <configuration>
              <outputDirectory>target/generated-sources/java</outputDirectory>
              <processor>com.querydsl.apt.jdo.JDOAnnotationProcessor</processor>
            </configuration>
          </execution>
        </executions>
      </plugin>
    ...
    </plugins>
  </build>
</project>
```

JDOAnnotationProcessor 查找使用 `javax.jdo.annotations.PersistenceCapable` 批注的域类型，并为它们生成查询类型。

运行全新安装，您将在 target/generated-sources/java 中生成查询类型。

如果您使用 Eclipse，请运行 mvn eclipse:eclipse 以更新您的 Eclipse 项目以包含 target/generated-sources/java 作为源文件夹。

现在您可以构造 JDO 查询实例和查询域模型的实例。

#### 2.2.2. Ant 集成

将来自 full-deps 包的 jar 文件放在类路径中，并使用以下任务生成 Querydsl 代码：

```xml
    <!-- 基于 APT 的代码生成 -->
    <javac srcdir="${src}" classpathref="cp">
      <compilerarg value="-proc:only"/>
      <compilerarg value="-processor"/>
      <compilerarg value="com.querydsl.apt.jdo.JDOAnnotationProcessor"/>
      <compilerarg value="-s"/>
      <compilerarg value="${generated}"/>
    </javac>

    <!-- 编译 -->
    <javac classpathref="cp" destdir="${build}">
      <src path="${src}"/>
      <src path="${generated}"/>
    </javac>
```

将 *src* 替换为您的主源文件夹，将 *generated* 替换为生成的源文件夹，将 *build* 替换为目标文件夹。

#### 2.2.3. 使用查询类型

要使用 Querydsl 创建查询，您需要实例化变量和 Query 实现。我们将从变量开始。

假设您的项目具有以下域类型：

```java
@PersistenceCapable
public class Customer {
  private String firstName;
  private String lastName;

  public String getFirstName() {
    return firstName;
  }

  public String getLastName() {
    return lastName;
  }

  public void setFirstName(String fn) {
    firstName = fn;
  }

  public void setLastName(String ln) {
    lastName = ln;
  }
}
```

Querydsl 将生成一个简单名称为 QCustomer 的查询类型到与 Customer 相同的包中。QCustomer 可以用作 Querydsl 中的静态类型变量，作为 Customer 类型的代表。

QCustomer 有一个可以作为静态字段访问的默认实例变量：

```java
QCustomer customer = QCustomer.customer;
```

或者，您可以像这样定义自己的客户变量：

```java
QCustomer customer = new QCustomer("myCustomer");
```

QCustomer 将原始类型 Customer 的所有属性反映为公共字段。可以像这样访问 firstName 字段

```java
customer.firstName;
```

#### 2.2.4. 使用 JDO 查询

对于 JDO 模块，`JDOQuery` 是主要的 Query 实现。它是这样实例化的：

```java
PersistenceManager pm = ...;
JDOQuery<?> query = new JDOQuery<Void>(pm);
```

对于本章的示例，查询是通过 `JDOQueryFactory` 实例创建的。`JDOQueryFactory` 应该是获取 `JDOQuery` 实例的首选选项。

要检索名字为 Bob 的客户，您将构建如下查询：

```java
QCustomer customer = QCustomer.customer;
Customer bob = queryFactory.selectFrom(customer)
                   .where(customer.firstName.eq("Bob"))
                   .fetchOne();
```

selectFrom 调用定义了查询源和投影，where 部分定义了过滤器，而 fetchOne 告诉 Querydsl 返回单个元素。很容易，对吧？

或者你也可以这样表达

```java
QCustomer customer = QCustomer.customer;
Customer bob = queryFactory.select(customer).from(customer)
                   .where(customer.firstName.eq("Bob"))
                   .fetchOne();
```

要创建具有多个源的查询，您只需使用 JDOQuery 类，如下所示：

```java
QCustomer customer = QCustomer.customer;
QCompany company = QCompany.company;
query.from(customer, company);
```

并使用多个过滤器像这样使用它

```java
queryFactory.selectFrom(customer)
    .where(customer.firstName.eq("Bob"), customer.lastName.eq("Wilson"));
```

或者像这样

```java
queryFactory.selectFrom(customer)
    .where(customer.firstName.eq("Bob").and(customer.lastName.eq("Wilson")));
```

如果您想通过“或”组合过滤器，请使用以下模式

```java
queryFactory.selectFrom(customer)
    .where(customer.firstName.eq("Bob").or(customer.lastName.eq("Wilson")));
```

#### 2.2.5. 一般用法

像这样使用 JDOQuery 类的级联方法

*select:* 设置查询的投影。（如果通过查询工厂创建则不需要）

*from:* 在此处添加查询源，第一个参数成为主要来源，其他参数视为变量。

*where:* 添加查询过滤器，以可变参数形式通过逗号分隔或通过 and 运算符级联。

*groupBy:* 以可变参数形式添加分组参数。

*have:* 添加具有“group by”分组的过滤器作为谓词表达式的可变参数数组。

*orderBy:* 将结果的顺序添加为顺序表达式的可变参数数组。对数字、字符串和其他可比较的表达式使用 asc() 和 desc() 来访问 OrderSpecifier 实例。

*limit,offset,restrict:* 设置结果的分页。最大结果的限制，跳过行的偏移量和一次调用中定义两者的限制。

#### 2.2.6. Ordering

声明排序的语法是

```java
QCustomer customer = QCustomer.customer;
queryFactory.selectFrom(customer)
    .orderBy(customer.lastName.asc(), customer.firstName.desc())
    .fetch();
```

#### 2.2.7. Grouping

分组可以按以下形式进行

```java
queryFactory.select(customer.lastName).from(customer)
    .groupBy(customer.lastName)
    .fetch();
```

#### 2.2.8. Delete 子句

Querydsl JDO 中的 Delete 子句遵循简单的 delete-where-execute 形式。这里有些例子：

```java
QCustomer 客户 = QCustomer.customer;
//删除所有客户
queryFactory.delete(customer).execute();
// 删除所有等级小于 3 的客户
queryFactory.delete(customer).where(customer.level.lt(3)).execute();
```

JDODeleteClause 构造函数的第二个参数是要删除的实体。where 调用是可选的，execute 调用执行删除并返回已删除实体的数量。

#### 2.2.9. 子查询

要创建子查询，您可以使用“JDOExpressions”的工厂方法之一，并通过 from、where 等添加查询参数。

```java
QDepartment department = QDepartment.department;
QDepartment d = new QDepartment("d");
queryFactory.selectFrom(department)
    .where(department.size.eq(JDOExpressions.select(d.size.max()).from(d))
    .fetch();
```

表示以下原生 JDO 查询

```sql
SELECT this FROM com.querydsl.jdo.models.company.Department
WHERE this.size ==
(SELECT max(d.size) FROM com.querydsl.jdo.models.company.Department d)

```

另一个例子

```java
QEmployee employee = QEmployee.employee;
QEmployee e = new QEmployee("e");
queryFactory.selectFrom(employee)
    .where(employee.weeklyhours.gt(
        JDOExpressions.select(e.weeklyhours.avg())
                      .from(employee.department.employees, e)
                      .where(e.manager.eq(employee.manager)))
    .fetch();
```

它表示以下原生 JDO 查询

```sql
SELECT this FROM com.querydsl.jdo.models.company.Employee
WHERE this.weeklyhours >
(SELECT avg(e.weeklyhours) FROM this.department.employees e WHERE e.manager == this.manager)

```

#### 2.2.10. 使用原生 SQL

Querydsl 通过 `JDOSQLQuery` 类支持 JDO 中的 Native SQL。

要使用它，您必须为您的 SQL 模式生成 Querydsl 查询类型。例如，这可以通过以下 Maven 配置来完成：

```xml
<project>
  <build>
    <plugins>
      ...
      <plugin>
        <groupId>com.querydsl</groupId>
        <artifactId>querydsl-maven-plugin</artifactId>
        <version>${querydsl.version}</version>
        <executions>
          <execution>
            <goals>
              <goal>export</goal>
            </goals>
          </execution>
        </executions>
        <configuration>
          <jdbcDriver>org.apache.derby.jdbc.EmbeddedDriver</jdbcDriver>
          <jdbcUrl>jdbc:derby:target/demoDB;create=true</jdbcUrl>
          <packageName>com.mycompany.mydomain</packageName>
          <targetFolder>${project.basedir}/target/generated-sources/java</targetFolder>
        </configuration>
        <dependencies>
          <dependency>
            <groupId>org.apache.derby</groupId>
            <artifactId>derby</artifactId>
            <version>${derby.version}</version>
          </dependency>
        </dependencies>
      </plugin>
      ...
    </plugins>
  </build>
</project>
```

当查询类型成功生成到您选择的位置后，您可以在查询中使用它们。

单列查询：

```java
// 序列化模板
SQLTemplates templates = new DerbyTemplates();
// 查询类型（S* 代表 SQL，Q* 代表域类型）
SAnimal cat = new SAnimal("cat");
SAnimal mate = new SAnimal("mate");

JDOSQLQuery<?> query = new JDOSQLQuery<Void>(pm, templates);
List<String> names = query.select(cat.name).from(cat).fetch();
```

查询多列：

```java
query = new JDOSQLQuery<Void>(pm, templates);
List<Tuple> rows = query.select(cat.id, cat.name).from(cat).fetch();
```

查询所有列：

```java
List<Tuple> rows = query.select(cat.all()).from(cat).fetch();

```

使用联结查询：

```java
query = new JDOSQLQuery<Void>(pm, templates);
cats = query.select(catEntity).from(cat)
    .innerJoin(mate).on(cat.mateId.eq(mate.id))
    .where(cat.dtype.eq("Cat"), mate.dtype.eq("Cat"))
    .fetch();
```

查询并投影到 DTO：

```java
query = new JDOSQLQuery<Void>(pm, templates);
List<CatDTO> catDTOs = query.select(Projections.constructor(CatDTO.class, cat.id, cat.name))
    .from(cat)
    .orderBy(cat.name.asc())
    .fetch();
```

### 2.3. 查询 SQL

本章介绍 SQL 模块的查询类型生成和查询功能。

#### 2.3.1. Maven 集成

将以下依赖项添加到您的 Maven 项目中：

```xml
<dependency>
  <groupId>com.querydsl</groupId>
  <artifactId>querydsl-sql</artifactId>
  <version>${querydsl.version}</version>
</dependency>

<dependency>
  <groupId>com.querydsl</groupId>
  <artifactId>querydsl-sql-codegen</artifactId>
  <version>${querydsl.version}</version>
  <scope>provided</scope>
</dependency>

<dependency>
  <groupId>org.slf4j</groupId>
  <artifactId>slf4j-log4j12</artifactId>
  <version>1.6.1</version>
</dependency>
```

如果通过 Maven 生成代码，则可以跳过 querydsl-sql-codegen 依赖项。

#### 2.3.2。通过 Maven 生成代码

此功能应主要通过 Maven 插件使用。这是一个例子：

```xml
<project>
  <build>
    <plugins>
      ...
      <plugin>
        <groupId>com.querydsl</groupId>
        <artifactId>querydsl-maven-plugin</artifactId>
        <version>${querydsl.version}</version>
        <executions>
          <execution>
            <goals>
              <goal>export</goal>
            </goals>
          </execution>
        </executions>
        <configuration>
          <jdbcDriver>org.apache.derby.jdbc.EmbeddedDriver</jdbcDriver>
          <jdbcUrl>jdbc:derby:target/demoDB;create=true</jdbcUrl>
          <packageName>com.myproject.domain</packageName>
          <targetFolder>${project.basedir}/target/generated-sources/java</targetFolder>
        </configuration>
        <dependencies>
          <dependency>
            <groupId>org.apache.derby</groupId>
            <artifactId>derby</artifactId>
            <version>${derby.version}</version>
          </dependency>
        </dependencies>
      </plugin>
      ...
    </plugins>
  </build>
</project>
```

使用目标 *test-export* 将目标文件夹视为用于测试代码的测试源文件夹。



**表 2.1. 参数**

| 姓名                     | 说明                                                         |
| ------------------------ | ------------------------------------------------------------ |
| jdbcDriver               | JDBC 驱动的类名                                              |
| jdbcUrl                  | JDBC 网址                                                    |
| jdbcUser                 | JDBC 用户                                                    |
| jdbcPassword             | JDBC 密码                                                    |
| namePrefix               | 生成的查询类的名称前缀（默认值：Q）                          |
| nameSuffix               | 生成的查询类的名称后缀（默认值：）                           |
| beanPrefix               | 生成的 bean 类的名称前缀                                     |
| beanSuffix               | 生成的 bean 类的名称后缀                                     |
| packageName              | 应生成源文件的包名                                           |
| beanPackageName          | 应生成 bean 文件的包名，（默认值：packageName）              |
| beanInterfaces           | 要添加到 bean 类的接口类名​​数组（默认值：空）                 |
| beanAddToString          | 设置为 true 以创建默认 toString() 实现（默认值：false）      |
| beanAddFullConstructor   | 设置为 true 以创建除公共空之外的完整构造函数（默认值：false） |
| beanPrintSupertype       | 设置为 true 以打印超类型（默认值：false）                    |
| schemaPattern            | LIKE 模式形式的模式名称模式；必须与存储在数据库中的模式名称匹配，多个可以用逗号分隔（默认值：null） |
| tableNamePattern         | LIKE 模式形式的表名模式；必须与存储在数据库中的表名匹配，多个可以用逗号分隔（默认：null） |
| targetFolder             | 应生成源的目标文件夹                                         |
| beansTargetFolder        | 应生成 bean 源的目标文件夹，默认为与 targetFolder            |
| namingStrategyClass      | NamingStrategy 类的类名（默认值：DefaultNamingStrategy）     |
| beanSerializerClass      | BeanSerializer 类的类名（默认：BeanSerializer）              |
| serializerClass          | Serializer 类的类名（默认：MetaDataSerializer）              |
| exportBeans              | 设置为 true 也可以生成 bean，请参阅第 2.14.13 节（默认值：false） |
| innerClassesForKeys      | 设置为 true 以生成键的内部类（默认值：false）                |
| validationAnnotations    | 设置为 true 以启用验证注释的序列化（默认值：false）          |
| columnAnnotations        | 导出列注释（默认值：false）                                  |
| createScalaSources       | 是否导出 Scala 源而不是 Java 源，（默认值：false）           |
| schemaToPackage          | 将架构名称附加到包（默认值：false）                          |
| lowerCase                | 名称的小写转换（默认值：false）                              |
| exportTables             | 导出表（默认：true）                                         |
| exportViews              | 导出视图（默认值：true）                                     |
| exportPrimaryKeys        | 导出主键（默认：true）                                       |
| tableTypesToExport       | 要导出的表类型的逗号分隔列表（允许的值取决于 JDBC 驱动程序）。允许导出任意类型的集合，例如：“TABLE, MATERIALIZED VIEW”。如果设置了此参数，将忽略 exportTables 和 exportViews 参数。（默认：无） |
| exportForeignKeys        | 导出外键（默认：true）                                       |
| exportDirectForeignKeys  | 导出直接外键（默认值：true）                                 |
| exportInverseForeignKeys | 导出反向外键（默认值：true）                                 |
| customTypes              | 自定义用户类型（默认：无）                                   |
| typeMappings             | table.column 到 Java 类型的映射（默认值：无）                |
| numericMappings          | 大小/数字到 Java 类型的映射（默认值：无）                    |
| imports                  | 添加到生成的查询类的 java 导入数组： *com.bar* 用于包（不带 .* 符号），*com.bar.Foo* 用于类（默认值：空） |



自定义类型可用于注册其他类型实现：

```xml
<customTypes>
  <customType>com.querydsl.sql.types.InputStreamType</customType>
</customTypes>
```

类型映射可用于注册 table.column 特定的 java 类型：

```xml
<typeMappings>
  <typeMapping>
    <table>IMAGE</table>
    <column>CONTENTS</column>
    <type>java.io.InputStream</type>
  </typeMapping>
</typeMappings>
```

数字映射的默认值是



**表 2.2。数字映射**

| 总位数 | 小数位 | 类型       |
| ------ | ------ | ---------- |
| > 18   | 0      | BigInteger |
| > 9    | 0      | Long       |
| > 4    | 0      | Integer    |
| > 2    | 0      | Short      |
| > 0    | 0      | Byte       |
| > 0    | > 0    | BigDecimal |



它们可以针对特定的总/小数位数组合进行定制，如下所示：

```xml
<numericMappings>
  <numericMapping>
    <total>1</total>
    <decimal>0</decimal>
    <javaType>java.lang.Byte</javaType>
  </numericMapping>
</numericMappings>
```

导入可用于添加跨模式外键支持。

模式、表和列也可以使用插件重命名。这里有些例子：

重命名架构：

```xml
<renameMappings>
  <renameMapping>
    <fromSchema>PROD</fromSchema>
    <toSchema>TEST</toSchema>
  </renameMapping>
</renameMappings>
```

重命名表：

```xml
<renameMappings>
  <renameMapping>
    <fromSchema>PROD</fromSchema>
    <fromTable>CUSTOMER</fromTable>
    <toTable>CSTMR</toTable>
  </renameMapping>
</renameMappings>
```

重命名列：

```xml
<renameMappings>
  <renameMapping>
    <fromSchema>PROD</fromSchema>
    <fromTable>CUSTOMER</fromTable>
    <fromColumn>ID</fromColumn>
    <toColumn>IDX</toTable>
  </renameMapping>
</renameMappings>
```

注意：重命名表和列时可以省略 fromSchema。

与基于 APT 的代码生成相比，某些功能不可用，例如 QueryDelegate 注释处理。

#### 2.3.3. 通过 ANT 生成代码

querydsl-sql 模块的 ANT 任务 `com.querydsl.sql.codegen.ant.AntMetaDataExporter` 提供与 ANT 任务相同的功能。该任务的配置参数与 Maven 插件相同，但复合类型除外。

复合类型在没有包装元素的情况下使用，如本例所示。

```xml
<project name="testproject" default="codegen" basedir=".">

  <taskdef name="codegen" classname="com.querydsl.sql.codegen.ant.AntMetaDataExporter"/>

  <target name="codegen">
    <codegen
      jdbcDriver="org.h2.Driver"
      jdbcUser="sa"
      jdbcUrl="jdbc:h2:/dbs/db1"
      packageName="test"
      targetFolder="target/generated-sources/java">
      <renameMapping fromSchema="PUBLIC" toSchema="PUB"/>
    </codegen>
  </target>
</project>
```

#### 2.3.4. 创建查询类型

要开始将架构导出为 Querydsl 查询类型，如下所示：

```java
java.sql.Connection conn = ...;
MetaDataExporter exporter = new MetaDataExporter();
exporter.setPackageName("com.myproject.mydomain");
exporter.setTargetFolder(new File("target/generated-sources/java"));
exporter.export(conn.getMetaData());
```

这声明数据库模式将被镜像到 target/generated-sources/java 文件夹中的 com.myproject.domain 包中。

生成的类型将表名转换为混合大小写作为类名，并将类似的混合大小写转换应用于可用作查询类型中的属性路径的列。

除了这个主键和外键约束之外，还提供了可用于紧凑连接声明的字段。

#### 2.3.5. 配置

配置是通过 com.querydsl.sql.Configuration 类完成的，该类将 Querydsl SQL 方言作为参数。对于 H2，您将像这样创建它

```java
SQLTemplates templates = new H2Templates();
Configuration configuration = new Configuration(templates);
```

Querydsl 使用 SQL 方言来自定义不同关系数据库所需的 SQL 序列化。可用的方言是：

- CUBRIDTemplates（使用 CUBRID 8.4 测试）
- DB2Templates（使用 DB2 10.1.2 测试）
- DerbyTemplates（使用 Derby 10.8.2.2 测试）
- FirebirdTemplates（用 Firebird 2.5 测试）
- HSQLDBTemplates（使用 HSQLDB 2.2.4 测试）
- H2Templates（使用 H2 1.3.164 测试）
- MySQLTemplates（用 MySQL 5.5 测试）
- OracleTemplates（使用 Oracle 10 和 11 进行测试）
- PostgreSQLTemplates（用 PostgreSQL 9.1 测试）
- SQLiteTemplates（使用 xerial JDBC 3.7.2 测试）
- SQLServerTemplates（使用 SQL Server 测试）
- SQLServer2005模板（用于 SQL Server 2005）
- SQLServer2008 模板（适用于 SQL Server 2008）
- SQLServer2012模板（适用于 SQL Server 2012 及更高版本）
- TeradataTemplates（使用 Teradata 14 测试）

对于自定义 SQLTemplates 实例，您可以像这样使用构建器模式

```java
  H2Templates.builder()
     .printSchema() // 在输出中包含模式
     .quote()       // 引用名称
     .newLineToSingleSpace() // 在输出中用单个空格替换新行
     .escape(ch)    // 设置转义字符
     .build();      // 获取自定义的 SQLTemplates 实例
```

Configuration 类的方法可用于通过 setUseLiterals(true) 启用文字的直接序列化，覆盖模式和表并注册自定义类型。有关完整的详细信息，请查看 Configuration 的 javadocs。

#### 2.3.6. 查询

对于以下示例，我们将使用 `SQLQueryFactory` 类来创建查询。与基于构造函数的查询创建相比，使用它可以生成更简洁的代码。

```java
SQLQueryFactory queryFactory = new SQLQueryFactory(configuration, dataSource);
```

使用 Querydsl SQL 查询就这么简单：

```java
QCustomer customer = new QCustomer("c");

List<String> lastNames = queryFactory.select(customer.lastName).from(customer)
    .where(customer.firstName.eq("Bob"))
    .fetch();
```

假设相关表名为*customer*，列*first_name*和*last_name*，则转化为如下sql查询：

```sql
SELECT c.last_name
FROM customer c
WHERE c.first_name = 'Bob'
```

#### 2.3.7. 一般用法

像这样使用 SQLQuery 类的级联方法

*select:* 设置查询的投影。（如果通过查询工厂创建则不需要）

*from:* 在此处添加查询源。

*innerJoin, join, leftJoin, rightJoin, fullJoin, on:* 使用这些结构添加连接元素。对于连接方法，第一个参数是连接源，第二个是目标（别名）。

*where:* 添加查询过滤器，以可变参数形式通过逗号分隔或通过 and 运算符级联。

*groupBy:* 以可变参数形式添加分组参数。

*have:* 添加具有“group by”分组的过滤器作为谓词表达式的 varags 数组。

*orderBy:* 将结果的顺序添加为顺序表达式的可变参数数组。对数字、字符串和其他可比较的表达式使用 asc() 和 desc() 来访问 OrderSpecifier 实例。

*limit,offset,restrict:* 设置结果的分页。最大结果的限制，跳过行的偏移量和一次调用中定义两者的限制。

#### 2.3.8. 联结

联结是使用以下语法构造的：

```java
QCustomer customer = QCustomer.customer;
QCompany company = QCompany.company;
queryFactory.select(customer.firstName, customer.lastName, company.name)
    .from(customer)
    .innerJoin(customer.company, company)
    .fetch();
```

对于左联结：

```java
queryFactory.select(customer.firstName, customer.lastName, company.name)
    .from(customer)
    .leftJoin(customer.company, company)
    .fetch();
```

或者，也可以写出联结条件：

```java
queryFactory.select(customer.firstName, customer.lastName, company.name)
    .from(customer)
    .leftJoin(company).on(customer.company.eq(company))
    .fetch();
```

#### 2.3.9. Ordering

声明排序的语法是

```java
queryFactory.select(customer.firstName, customer.lastName)
    .from(customer)
    .orderBy(customer.lastName.asc(), customer.firstName.asc())
    .fetch();
```

相当于下面的原生 SQL

```sql
SELECT c.first_name, c.last_name
FROM customer c
ORDER BY c.last_name ASC, c.first_name ASC
```

#### 2.3.10. Grouping

分组可以按以下形式进行

```java
queryFactory.select(customer.lastName)
    .from(customer)
    .groupBy(customer.lastName)
    .fetch();
```

相当于下面的原生 SQL

```sql
SELECT c.last_name
FROM customer c
GROUP BY c.last_name
```

#### 2.3.11. 使用子查询

要创建子查询，您可以使用“SQLExpressions”的工厂方法之一，并通过 from、where 等添加查询参数。

```java
QCustomer customer = QCustomer.customer;
QCustomer customer2 = new QCustomer("customer2");
queryFactory.select(customer.all())
    .from(customer)
    .where(customer.status.eq(
        SQLExpressions.select(customer2.status.max()).from(customer2)))
    .fetch()
```

另一个例子

```java
QStatus status = QStatus.status;
queryFactory.select(customer.all())
    .from(customer)
    .where(customer.status.in(
        SQLExpressions.select(status.id).from(status).where(status.level.lt(3))))
    .fetch();
```

#### 2.3.12. 选择文字

要选择文字，您需要为它们创建常量实例，如下所示：

```java
queryFactory.select(Expressions.constant(1),
                    Expressions.constant("abc"));
```

`com.querydsl.core.types.dsl.Expressions` 类还为投影、操作和模板创建提供了其他有用的静态方法。

#### 2.3.13. 查询扩展支持

可以通过继承 AbstractSQLQuery 并添加标记方法来创建支持引擎特定语法的自定义查询扩展，例如给定的 MySQLQuery 示例：

```java
public class MySQLQuery<T> extends AbstractSQLQuery<T, MySQLQuery<T>> {

    public MySQLQuery(Connection conn) {
        this(conn, new MySQLTemplates(), new DefaultQueryMetadata());
    }

    public MySQLQuery(Connection conn, SQLTemplates templates) {
        this(conn, templates, new DefaultQueryMetadata());
    }

    protected MySQLQuery(Connection conn, SQLTemplates templates, QueryMetadata metadata) {
        super(conn, new Configuration(templates), metadata);
    }

    public MySQLQuery bigResult() {
        return addFlag(Position.AFTER_SELECT, "SQL_BIG_RESULT ");
    }

    public MySQLQuery bufferResult() {
        return addFlag(Position.AFTER_SELECT, "SQL_BUFFER_RESULT ");
    }


    // ...
}
```

这些标志是自定义 SQL 片段，可以插入到序列化的特定点。支持的位置是 com.querydsl.core.QueryFlag.Position 枚举类的枚举。

#### 2.3.14. 窗口函数

Querydsl 通过 `SQLExpressions` 类中的方法支持窗口函数。

使用示例：

```java
queryFactory.select(SQLExpressions.rowNumber()
        .over()
        .partitionBy(employee.name)
        .orderBy(employee.id))
     .from(employee)
```

#### 2.3.15. 公用表表达式

Querydsl SQL 通过两种语法变体支持公用表表达式

```java
QEmployee employee = QEmployee.employee;
queryFactory.with(employee, SQLExpressions.select(employee.all)
                                          .from(employee)
                                          .where(employee.name.startsWith("A")))
            .from(...)
```

并使用列列表

```java
QEmployee employee = QEmployee.employee;
queryFactory.with(employee, employee.id, employee.name)
            .as(SQLExpressions.select(employee.id, employee.name)
                              .from(employee)
                              .where(employee.name.startsWith("A")))
            .from(...)
```

如果公用表表达式的列是现有表或视图的子集，建议使用生成的路径类型，例如在这种情况下为 QEmployee，但如果列不适合任何现有表，则可以使用 PathBuilder反而。

以下是此类情况的示例

```java
QEmployee employee = QEmployee.employee;
QDepartment department = QDepartment.department;
PathBuilder<Tuple> emp = new PathBuilder<Tuple>(Tuple.class, "emp");
queryFactory.with(emp, SQLExpressions.select(employee.id, employee.name, employee.departmentId,
                                          department.name.as("departmentName"))
                                      .from(employee)
                                      .innerJoin(department).on(employee.departmentId.eq(department.id))))
            .from(...)
```

#### 2.3.16. 其他 SQL 表达式

`SQLExpressions` 类中还提供了其他 SQL 表达式作为静态方法。

#### 2.3.17. 使用数据操作命令

##### 2.3.17.1. 插入

带列

```java
QSurvey survey = QSurvey.survey;

queryFactory.insert(survey)
    .columns(survey.id, survey.name)
    .values(3, "Hello").execute();
```

无列

```java
queryFactory.insert(survey)
    .values(4, "Hello").execute();
```

带子查询

```java
queryFactory.insert(survey)
    .columns(survey.id, survey.name)
    .select(SQLExpressions.select(survey2.id.add(1), survey2.name).from(survey2))
    .execute();
```

有子查询，没有列

```java
queryFactory.insert(survey)
    .select(SQLExpressions.select(survey2.id.add(10), survey2.name).from(survey2))
    .execute();
```

作为列/值用法的替代方法，Querydsl 还提供了一个 set 方法，可以像这样使用

```java
QSurvey survey = QSurvey.survey;

queryFactory.insert(survey)
    .set(survey.id, 3)
    .set(survey.name, "Hello").execute();
```

这相当于第一个例子。set 方法的使用总是在内部扩展到列和值。

请注意

```sql
columns(...).select(...)
```

映射要插入的给定查询的结果集，而

要获取创建的键而不是修改的行数，请使用其中一种 executeWithKey/s 方法。

```sql
set(...)
```

映射单列和空值用于空子查询结果。

要根据 bean 的内容填充子句实例，您可以使用

```java
queryFactory.insert(survey)
    .populate(surveyBean).execute();
```

这将排除空绑定，如果您还需要空绑定使用

```java
queryFactory.insert(survey)
    .populate(surveyBean, DefaultMapper.WITH_NULL_BINDINGS).execute();
```

##### 2.3.17.2. Update

使用 where

```java
QSurvey survey = QSurvey.survey;

queryFactory.update(survey)
    .where(survey.name.eq("XXX"))
    .set(survey.name, "S")
    .execute();
```

没有 where

```java
queryFactory.update(survey)
    .set(survey.name, "S")
    .execute();
```

使用 bean 种群

```java
queryFactory.update(survey)
    .populate(surveyBean)
    .execute();
```

##### 2.3.17.3. Delete

使用 where

```java
QSurvey survey = QSurvey.survey;

queryFactory.delete(survey)
    .where(survey.name.eq("XXX"))
    .execute();
```

没有 where

```java
queryFactory.delete(survey)
    .execute()
```

#### 2.3.18. DML 子句中的批处理支持

Querydsl SQL 支持通过 DML API 使用 JDBC 批量更新。如果您有连续的具有相似结构的 DML 调用，则可以通过 addBatch() 用法将调用捆绑到一个 DMLClause 中。请参阅示例如何处理 UPDATE、DELETE 和 INSERT。

Update：

```java
QSurvey 调查 = QSurvey.survey；

queryFactory.insert(survey).values(2, "A").execute();
queryFactory.insert(survey).values(3, "B").execute();

SQLUpdateClause update = queryFactory.update(survey);
update.set(survey.name, "AA").where(survey.name.eq("A")).addBatch();
update.set(survey.name, "BB").where(survey.name.eq("B")).addBatch();
```

Delete：

```java
queryFactory.insert(survey).values(2, "A").execute();
queryFactory.insert(survey).values(3, "B").execute();

SQLDeleteClause delete = queryFactory.delete(survey);
delete.where(survey.name.eq("A")).addBatch();
delete.where(survey.name.eq("B")).addBatch();
assertEquals(2, delete.execute());
```

Insert：

```java
SQLInsertClause insert = queryFactory.insert(survey);
insert.set(survey.id, 5).set(survey.name, "5").addBatch();
insert.set(survey.id, 6).set(survey.name, "6").addBatch();
assertEquals(2, insert.execute());
```

#### 2.3.19. Bean 类生成

要为模式的表创建 JavaBean DTO 类型，请使用 MetaDataExporter，如下所示：

```java
java.sql.Connection conn = ...;
MetaDataExporter exporter = new MetaDataExporter();
exporter.setPackageName("com.myproject.mydomain");
exporter.setTargetFolder(new File("src/main/java"));
exporter.setBeanSerializer(new BeanSerializer());
exporter.export(conn.getMetaData());
```

现在，您可以将 bean 类型用作 DML 子句中的 populate 方法的参数，并且可以直接投影到查询中的 bean 类型。这是一个 JUnit 形式的简单示例：

```java
QEmployee e = new QEmployee("e");

// 插入
Employee employee = new Employee();
employee.setFirstname("John");
Integer id = queryFactory.insert(e).populate(employee).executeWithKey(e.id);
employee.setId(id);

// 更新
employee.setLastname("Smith");
assertEquals(1l, queryFactory.update(e).populate(employee).where(e.id.eq(employee.getId())).execute());

// 询问
Employee smith = queryFactory.selectFrom(e).where(e.lastname.eq("Smith")).fetchOne();
assertEquals("John", smith.getFirstname());

// 删除
assertEquals(1l, queryFactory.delete(e).where(e.id.eq(employee.getId())).execute());
```

#### 2.3.20. 提取 SQL 查询和绑定

SQL 查询和绑定可以通过 getSQL 方法提取：

```java
SQLBindings bindings = query.getSQL();
System.out.println(bindings.getSQL());
```

如果您还需要 SQL 字符串中的所有文字，您可以通过 setUseLiterals(true) 在查询或配置级别启用文字序列化。

#### 2.3.21. 自定义类型

Querydsl SQL 提供了为 ResultSet/Statement 交互声明自定义类型映射的可能性。自定义类型映射可以在 com.querydsl.sql.Configuration 实例中声明，这些实例作为构造函数参数提供给实际查询：

```java
Configuration configuration = new Configuration(new H2Templates());
// 覆盖 Types.DATE 的映射
configuration.register(new UtilDateType());
```

对于表格列

```java
Configuration configuration = new Configuration(new H2Templates());
// 为 person 表中的性别列声明一个映射
configuration.register("person", "gender", new EnumByNameType<Gender>(Gender.class));
```

要自定义数字映射，您可以像这样使用 registerNumeric 方法

```java
configuration.registerNumeric(5,2,Float.class);
```

这会将 Float 类型映射到 NUMERIC(5,2) 类型。

#### 2.3.22. 侦听查询和子句

SQLListener 是一个侦听器接口，可用于侦听查询和 DML 子句。SQLListener 实例可以通过 addListener 方法在配置和查询/子句级别上注册。

侦听器的用例是数据同步、日志记录、缓存和验证。

#### 2.3.23. Spring 集成

Querydsl SQL 通过 querydsl-sql-spring 模块与 Spring 集成：

```xml
<dependency>
  <groupId>com.querydsl</groupId>
  <artifactId>querydsl-sql-spring</artifactId>
  <version>${querydsl.version}</version>
</dependency>
```

它提供了 Spring 异常翻译和 Spring 连接提供程序，用于将 Querydsl SQL 与 Spring 事务管理器一起使用。下面是一个配置示例：

```java
package com.querydsl.example.config;

import com.querydsl.sql.H2Templates;
import com.querydsl.sql.SQLQueryFactory;
import com.querydsl.sql.SQLTemplates;
import com.querydsl.sql.spring.SpringConnectionProvider;
import com.querydsl.sql.spring.SpringExceptionTranslator;
import com.querydsl.sql.types.DateTimeType;
import com.querydsl.sql.types.LocalDateType;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.PropertySource;
import org.springframework.core.env.Environment;
import org.springframework.jdbc.datasource.DataSourceTransactionManager;
import org.springframework.transaction.PlatformTransactionManager;

import javax.inject.Inject;
import javax.inject.Provider;
import javax.sql.DataSource;
import java.sql.Connection;

@Configuration
public class JdbcConfiguration {

    @Bean
    public DataSource dataSource() {
        // implementation omitted
    }

    @Bean
    public PlatformTransactionManager transactionManager() {
        return new DataSourceTransactionManager(dataSource());
    }

    @Bean
    public com.querydsl.sql.Configuration querydslConfiguration() {
        SQLTemplates templates = H2Templates.builder().build(); //change to your Templates
        com.querydsl.sql.Configuration configuration = new com.querydsl.sql.Configuration(templates);
        configuration.setExceptionTranslator(new SpringExceptionTranslator());
        return configuration;
    }

    @Bean
    public SQLQueryFactory queryFactory() {
        Provider<Connection> provider = new SpringConnectionProvider(dataSource());
        return new SQLQueryFactory(querydslConfiguration(), provider);
    }

}
```

### 2.4. Querydsl 空间

可通过 Querydsl Spatial 模块获得对空间查询的支持，该模块是 SQL 模块的扩展模块。Spatial 模块在查询和对象绑定中支持简单特征访问的对象模型。

[geolatte](http://www.geolatte.org/) 项目用于对象模型。

![空间](https://img2020.cnblogs.com/blog/1053740/202007/1053740-20200715165529823-38908165.png)

#### 2.4.1. Maven 集成

将以下依赖项添加到您的 Maven 项目中：

```xml
<dependency>
  <groupId>com.querydsl</groupId>
  <artifactId>querydsl-sql-spatial</artifactId>
  <version>${querydsl.version}</version>
</dependency>
```

此外，以下数据库特定的额外依赖项：

```xml
<!-- 用于 PostgreSQL 的 -->
<dependency>
  <groupId>org.postgis</groupId>
  <artifactId>postgis-jdbc</artifactId>
  <version>1.3.3</version>
  <scope>provided</scope>
</dependency>

<!-- 供 Oracle 使用 -->
<dependency>
  <groupId>oracle</groupId>
  <artifactId>sdoapi</artifactId>
  <version>11.2.0</version>
  <scope>provided</scope>
</dependency>
```

#### 2.4.2. 通过 Maven 生成代码

Querydsl SQL 的代码生成可以设置为检测数据库模式中空间类型的使用，并在这些情况下通过空间属性使用 geolatte 类型：

```xml
<project>
  <build>
    <plugins>
      ...
      <plugin>
        <groupId>com.querydsl</groupId>
        <artifactId>querydsl-maven-plugin</artifactId>
        <version>${querydsl.version}</version>
        ...
        <configuration>
          ...
          <spatial>true</spatial>
        </configuration>
      </plugin>
      ...
    </plugins>
  </build>
</project>
```

#### 2.4.3. 运行时配置

空间模块的运行时配置方面是使用启用空间的实例而不是普通的 SQLTemplates 实例。下面是启用空间的 SQLTemplates 类的列表。

- GeoDBTemplates（用于 H2）
- MySQL空间模板
- OracleSpatialTemplates（阿尔法阶段）
- PostGIS模板
- SQLServer2008空间模板
- TeradataSpatialTemplates

#### 2.4.4. 查询

通过为空间类型设置代码生成和运行时配置，我们现在可以尝试使用它进行查询。

##### 2.4.4.1. 按距离过滤

```java
Geometry point = Wkt.fromWkt("Point(2 2)");
query.where(table.geo.distance(point).lt(5.0));
```

除了几何之间的直线距离之外，还通过 distanceSphere 和 distanceSpheroid 提供了球面和球面距离。

##### 2.4.4.2. 包含

```java
Geometry point = Wkt.fromWkt("Point(2 2)");
query.where(table.geo.contains(point));
```

##### 2.4.4.3. 相交

```java
Geometry geo = query.select(table.geo1.intersection(table.geo2)).fetchOne();
```

##### 2.4.4.4. 访问 SPATIAL_REF_SYS 表

通过 QSpatialRefSys 和 SpatialRefSys 类提供对 SPATIAL_REF_SYS 标准表的统一访问。SPATIAL_REF_SYS 包含有关支持的空间参考系统的数据。

```java
QSpatialRefSys spatialRefSys = QSpatialRefSys.spatialRefSys;
List<SpatialRefSys> referenceSystems = query.select(spatialRefSys).from(spatialRefSys).fetch();
```

#### 2.4.5. 继承

如果您在数据库模式中仅使用通用几何类型，您可以使用对象模型中的转换方法来转换为更具体的类型。

```java
GeometryPath<Geometry> geometry = shapes.geometry;
PointPath<Point> point = geometry.asPoint();
NumberExpression<Double> pointX = point.x(); // x() 在 GeometryExpression/GeometryPath 上不可用
```

### 2.5. 查询 Lucene

本章介绍 Lucene 模块的查询功能。

#### 2.5.1. Maven 集成

Querydsl Lucene 可以通过 Lucene 3 的 querydsl-lucene3 模块、Lucene 4 的 querydsl-lucene4 和 Lucene 5 的 querydsl-lucene5 使用

Lucene 3：

```xml
<dependency>
  <groupId>com.querydsl</groupId>
  <artifactId>querydsl-lucene3</artifactId>
  <version>${querydsl.version}</version>
</dependency>

<dependency>
  <groupId>org.slf4j</groupId>
  <artifactId>slf4j-log4j12</artifactId>
  <version>1.6.1</version>
</dependency>
```

Lucene 4：

```xml
<dependency>
  <groupId>com.querydsl</groupId>
  <artifactId>querydsl-lucene4</artifactId>
  <version>${querydsl.version}</version>
</dependency>

<dependency>
  <groupId>org.slf4j</groupId>
  <artifactId>slf4j-log4j12</artifactId>
  <version>1.6.1</version>
</dependency>
```

Lucene 5：

```xml
<dependency>
  <groupId>com.querydsl</groupId>
  <artifactId>querydsl-lucene5</artifactId>
  <version>${querydsl.version}</version>
</dependency>

<dependency>
  <groupId>org.slf4j</groupId>
  <artifactId>slf4j-log4j12</artifactId>
  <version>1.6.1</version>
</dependency>
```

#### 2.5.2. 创建查询类型

使用字段 year 和 title 手动创建的查询类型可能如下所示：

```java
public class QDocument extends EntityPathBase<Document> {
    private static final long serialVersionUID = -4872833626508344081L;

    public QDocument(String var) {
        super(Document.class, PathMetadataFactory.forVariable(var));
    }

    public final StringPath year = createString("year");

    public final StringPath title = createString("title");
}
```

QDocument 表示带有字段 year 和 title 的 Lucene 文档。

Lucene 不支持代码生成，因为没有可用的模式数据。

#### 2.5.3. 查询

使用 Querydsl Lucene 进行查询就像这样简单：

```java
QDocument doc = new QDocument("doc");

IndexSearcher searcher = new IndexSearcher(index);
LuceneQuery query = new LuceneQuery(true, searcher);
List<Document> documents = query
    .where(doc.year.between("1800", "2000").and(doc.title.startsWith("Huckle"))
    .fetch();
```

将其转换为以下 Lucene 查询：

```
+year:[1800 TO 2000] +title:huckle*
```

#### 2.5.4. 一般用法

像这样使用 LuceneQuery 类的级联方法

*where:* 添加查询过滤器，以可变参数形式通过逗号分隔或通过 and 运算符级联。支持的操作是对除 *matches* 、 *indexOf* 、 *charAt* 之外的 PStrings 执行的操作。目前不支持 *in*，但将来会支持。

*orderBy:* 将结果的顺序添加为顺序表达式的可变参数数组。对数字、字符串和其他可比较的表达式使用 asc() 和 desc() 来访问 OrderSpecifier 实例。

*limit,offset,restrict:* 设置结果的分页。最大结果的限制，跳过行的偏移量和一次调用中定义两者的限制。

#### 2.5.5. Ordering

声明排序的语法是

```java
query
    .where(doc.title.like("*"))
    .orderBy(doc.title.asc(), doc.year.desc())
    .fetch();
```

相当于下面的 Lucene 查询

```
title:*
```

结果根据标题和年份升序排序。

或者，可以使用排序方法调用将排序逻辑声明为 Sort 实例

```java
Sort sort = ...;
query
    .where(doc.title.like("*"))
    .sort(sort)
    .fetch();
```

#### 2.5.6. Limit

声明限制的语法是

```java
query
    .where(doc.title.like("*"))
    .limit(10)
    .fetch();
```

#### 2.5.7. Offset

声明偏移量的语法是

```java
query
    .where(doc.title.like("*"))
    .offset(3)
    .fetch();
```

#### 2.5.8. 模糊搜索

模糊搜索可以通过 com.querydsl.lucene3.LuceneExpressions 类中的 blurLike 方法表示：

```java
query
    .where(LuceneExpressions.fuzzyLike(doc.title, "Hello"))
    .fetch();
```

#### 2.5.9. 将 Lucene 过滤器应用于查询

可以将单个 Lucene 过滤器应用于查询，如下所示：

```java
query
    .where(doc.title.like("*"))
    .filter(filter)
    .fetch();
```

通过 distinct(Path) 方法提供不同过滤的快捷方式：

```java
query
    .where(doc.title.like("*"))
    .distinct(doc.title)
    .fetch();
```



### 2.6. 查询 Hibernate Search

本章介绍 Hibernate Search 模块的查询功能。

#### 2.6.1. 创建 Querydsl 查询类型

有关如何创建查询类型的说明，请参阅 [查询 JPA/Hibernate 源](http://www.querydsl.com/static/querydsl/latest/reference/html/ch02.html#jpa_integration)。

#### 2.6.2. 查询

使用 Querydsl Hibernate Search 进行查询非常简单：

```java
QUser user = QUser.user;
SearchQuery<User> query = new SearchQuery<User>(session, user);
List<User> list = query
    .where(user.firstName.eq("Bob"))
    .fetch();
```

#### 2.6.3. 一般用法

有关一般使用说明，请参阅 [查询 Lucene 源](http://www.querydsl.com/static/querydsl/latest/reference/html/ch02s05.html)。

在查询序列化中，与 Querydsl Lucene 模块的唯一区别是路径的处理方式不同。对于 org.hibernate.search.annotations.Field 带注释的属性，名称属性与属性名称一起用作字段名称的后备。

### 2.7. 查询MongoDB

本章介绍Mongodb 模块的查询功能。

#### 2.7.1. Maven 集成

将以下依赖项添加到您的 Maven 项目中：

```xml
<dependency>
  <groupId>com.querydsl</groupId>
  <artifactId>querydsl-apt</artifactId>
  <version>${querydsl.version}</version>
  <scope>provided</scope>
</dependency>

<dependency>
  <groupId>com.querydsl</groupId>
  <artifactId>querydsl-mongodb</artifactId>
  <version>${querydsl.version}</version>
</dependency>

<dependency>
  <groupId>org.slf4j</groupId>
  <artifactId>slf4j-log4j12</artifactId>
  <version>1.6.1</version>
</dependency>
```

现在，配置生成 Querydsl 使用的查询类型的 Maven APT 插件：

```xml
<project>
  <build>
    <plugins>
      ...
      <plugin>
        <groupId>com.mysema.maven</groupId>
        <artifactId>apt-maven-plugin</artifactId>
        <version>1.1.3</version>
        <executions>
          <execution>
            <goals>
              <goal>process</goal>
            </goals>
            <configuration>
              <outputDirectory>target/generated-sources/java</outputDirectory>
              <processor>com.querydsl.apt.morphia.MorphiaAnnotationProcessor</processor>
            </configuration>
          </execution>
        </executions>
      </plugin>
    ...
    </plugins>
  </build>
</project>
```

MorphiaAnnotationProcessor 查找使用 com.google.code.morphia.annotations.Entity 注释注释的域类型，并为它们生成 Querydsl 查询类型。

运行全新安装，您将在 target/generated-sources/java 中生成查询类型。

如果您使用 Eclipse，请运行 mvn eclipse:eclipse 以更新您的 Eclipse 项目以包含 target/generated-sources/java 作为源文件夹。

现在您可以构建 Mongodb 查询和查询域模型的实例。

#### 2.7.2。查询

使用 Morphia 使用 Querydsl Mongodb 进行查询就像这样简单：

```java
Morphia morphia;
Datastore datastore;
// ...
QUser user = new QUser("user");
MorphiaQuery<User> query = new MorphiaQuery<User>(morphia, datastore, user);
List<User> list = query
    .where(user.firstName.eq("Bob"))
    .fetch();
```

#### 2.7.3. 一般用法

像这样使用 MongodbQuery 类的级联方法

*where:* 添加查询过滤器，以可变参数形式通过逗号分隔或通过 and 运算符级联。支持的操作是对除 *matches* 、 *indexOf* 、 *charAt* 之外的 PStrings 执行的操作。目前不支持 *in*，但将来会支持。

*orderBy:* 将结果的顺序添加为顺序表达式的可变参数数组。对数字、字符串和其他可比较的表达式使用 asc() 和 desc() 来访问 OrderSpecifier 实例。

*limit,offset,restrict:* 设置结果的分页。最大结果的限制，跳过行的偏移量和一次调用中定义两者的限制。

#### 2.7.4. Ordering

声明排序的语法是

```java
query
    .where(doc.title.like("*"))
    .orderBy(doc.title.asc(), doc.year.desc())
    .fetch();
```

结果根据标题和年份升序排序。

#### 2.7.5. Limit

声明限制的语法是

```java
query
    .where(doc.title.like("*"))
    .limit(10)
    .fetch();
```

#### 2.7.6. Offset

声明偏移量的语法是

```java
query
    .where(doc.title.like("*"))
    .offset(3)
    .fetch();
```

#### 2.7.7. 地理空间查询

Double 类型数组 (Double[]) 通过 near 方法支持地理空间查询：

```java
query
    .where(geoEntity.location.near(50.0, 50.0))
    .fetch();
```

#### 2.7.8. 仅选择相关字段

要仅选择相关字段，您可以使用重载的投影方法 fetch、iterate、fetchOne 和 fetchFirst 方法，如下所示

```java
query
    .where(doc.title.like("*"))
    .fetch(doc.title, doc.path);
```

此查询将仅加载文档的标题和路径字段。

### 2.8. 查询集合

querydsl-collections 模块可以与生成的查询类型一起使用，也可以不使用。第一部分描述了没有生成查询类型的用法：

#### 2.8.1. 没有生成查询类型的用法

要在不生成查询类型的情况下使用 querydsl-collections，您需要使用 Querydsl 别名功能。这里有些例子。

首先，添加以下静态导入：

```java
// 需要访问 Querydsl Collections API
import static com.querydsl.collections.CollQueryFactory.*;
// 需要，如果你使用 $-invocations
import static com.querydsl.core.alias.Alias.*;
```

现在为 Cat 类创建一个别名实例。只能为具有空构造函数的非最终类创建别名实例。确保你的班级有一个。

Cat 类型的别名实例及其 getter 调用通过将它们包装到美元方法调用中而转换为路径。例如，调用 `c.getKittens()` 在内部转换为 $ 方法内的属性路径 `c.kittens`。

```java
Cat c = alias(Cat.class, "cat");
for (String name : select($(c.getName())).from($(c),cats)
  .where($(c.getKittens()).size().gt(0))
  .fetch()) {
    System.out.println(name);
}
```

以下示例是前一个示例的变体，其中对列表大小的访问发生在美元方法调用中。

```java
Cat c = alias(Cat.class, "cat");
for (String name : select($(c.getName())).from($(c),cats)
  .where($(c.getKittens().size()).gt(0))
  .fetch()) {
    System.out.println(name);
}
```

别名的所有非原始和非最终类型属性都是别名本身。因此，您可以级联方法调用，直到在美元方法范围内遇到原始或非最终类型（例如 java.lang.String）。

例如

```java
$(c.getMate().getName())
```

在内部转换为 *c.mate.name*，但是

```java
$(c.getMate().getName().toLowerCase())
```

未正确转换，因为未跟踪 toLowerCase() 调用。

另请注意，您只能在别名类型上调用 getter、size()、contains(Object) 和 get(int)。所有其他调用都会引发异常。

#### 2.8.2. 与生成的查询类型一起使用

上面的例子可以用生成的表达式类型这样表达

```java
QCat cat = new QCat("cat");
for (String name : select(cat.name).from(cat,cats)
  .where(cat.kittens.size().gt(0))
  .fetch()) {
    System.out.println(name);
}
```

当您使用生成的查询类型时，您将实例化表达式而不是别名实例，并直接使用属性路径，而无需任何美元方法包装。

#### 2.8.3. Maven 集成

将以下依赖项添加到您的 Maven 项目中：

```xml
<dependency>
  <groupId>com.querydsl</groupId>
  <artifactId>querydsl-apt</artifactId>
  <version>${querydsl.version}</version>
  <scope>provided</scope>
</dependency>

<dependency>
  <groupId>com.querydsl</groupId>
  <artifactId>querydsl-collections</artifactId>
  <version>${querydsl.version}</version>
</dependency>

<dependency>
  <groupId>org.slf4j</groupId>
  <artifactId>slf4j-log4j12</artifactId>
  <version>1.6.1</version>
</dependency>
```

如果您不使用 JPA 或 JDO，您可以通过使用 `com.querydsl.core.annotations.QueryEntity` 批注并将以下插件配置添加到您的 Maven 配置 (pom.xml) 中来为您的域类型生成表达式类型：

```xml
<project>
  <build>
  <plugins>
    ...
    <plugin>
      <groupId>com.mysema.maven</groupId>
      <artifactId>apt-maven-plugin</artifactId>
      <version>1.1.3</version>
      <executions>
        <execution>
          <goals>
            <goal>process</goal>
          </goals>
          <configuration>
            <outputDirectory>target/generated-sources/java</outputDirectory>
            <processor>com.querydsl.apt.QuerydslAnnotationProcessor</processor>
          </configuration>
        </execution>
      </executions>
    </plugin>
    ...
  </plugins>
  </build>
</project>
```

#### 2.8.4. Ant 集成

将来自 full-deps 包的 jar 文件放在类路径中，并使用以下任务生成 Querydsl 代码：

```xml
    <!-- 基于 APT 的代码生成 -->
    <javac srcdir="${src}" classpathref="cp">
      <compilerarg value="-proc:only"/>
      <compilerarg value="-processor"/>
      <compilerarg value="com.querydsl.apt.QuerydslAnnotationProcessor"/>
      <compilerarg value="-s"/>
      <compilerarg value="${generated}"/>
    </javac>

    <!-- 编译 -->
    <javac classpathref="cp" destdir="${build}">
      <src path="${src}"/>
      <src path="${generated}"/>
    </javac>
```

将 *src* 替换为您的主源文件夹，将 *generated* 替换为生成的源文件夹，将 *build* 替换为目标文件夹。

#### 2.8.5. Hamcrest 匹配器

Querydsl Collections 提供 Hamcrest 匹配器。有了这些进口

```java
import static org.hamcrest.core.IsEqual.equalTo;
import static com.querydsl.collections.PathMatcher.hasValue;
import static org.junit.Assert.assertEquals;
import static org.junit.Assert.assertThat;
```

它们可以这样使用：

```java
Car car = new Car();
car.setHorsePower(123);

assertThat(car, hasValue($.horsePower));
assertThat(car, hasValue($.horsePower, equalTo(123)));
```

Hamcrest 匹配器由 [Jeroen van Schagen](https://github.com/jeroenvs) 贡献。

#### 2.8.6. 与 Java 的 Eclipse 编译器一起使用

如果 Querydsl Collections 与系统编译器不可用的 JRE 一起使用，则还可以将 CollQuery 实例配置为使用 Eclipse Compiler for Java (ECJ)：

```java
DefaultEvaluatorFactory evaluatorFactory = new DefaultEvaluatorFactory(
    CollQueryTemplates.DEFAULT,
    new ECJEvaluatorFactory(getClass().getClassLoader()));
QueryEngine queryEngine = new DefaultQueryEngine(evaluatorFactory);
CollQuery query = new CollQuery(queryEngine);
```

### 2.9. 在 Scala 中查询

可以通过 querydsl-scala 模块获得对 Scala 中 Querydsl 使用的通用支持。要将其添加到您的 Maven 构建中，请使用以下代码段：

```xml
<dependency>
  <groupId>com.querydsl</groupId>
  <artifactId>querydsl-scala</artifactId>
  <version>${querydsl.version}</version>
</dependency>
```

#### 2.9.1. Scala 的 DSL 表达式

Querydsl for Scala 为表达式构造提供了另一种 DSL。Scala DSL 利用诸如运算符重载、函数指针和隐式导入等语言特性来增强可读性和简洁性。

以下是主要替代方案的概述：

```
//标准                  替代

expr isNotNull          expr is not(null)
expr isNull             expr is null
expr eq "Ben"           expr === "Ben"
expr ne "Ben"           expr !== "Ben"
expr append "X"         expr + "X"
expr isEmpty            expr is empty
expr isNotEmpty         expr not empty

// 布尔值
left and right          left && right
left or right           left || right
expr not                !expr

// 比较
expr lt 5               expr < 5
expr loe 5              expr <= 5
expr gt 5               expr > 5
expr goe 5              expr >= 5
expr notBetween(2,6)    expr not between (2,6)
expr negate             -expr

// 数字
expr add 3              expr + 3
expr subtract 3         expr - 3
expr divide 3           expr / 3
expr multiply 3         expr * 3
expr mod 5              expr % 5

// 集合
list.get(0)             list(0)
map.get("X")            map("X")
```

#### 2.9.2. 使用 SQL 查询

与 Querydsl SQL for Java 一样，您需要生成查询类型才能构建查询。以下代码示例显示了这是如何完成的：

没有 Bean 类型的生成：

```java
val directory = new java.io.File("target/jdbcgen1")
val namingStrategy = new DefaultNamingStrategy()
val exporter = new MetaDataExporter()
exporter.setNamePrefix("Q")
exporter.setPackageName("com.querydsl")
exporter.setSchemaPattern("PUBLIC")
exporter.setTargetFolder(directory)
exporter.setSerializerClass(classOf[ScalaMetaDataSerializer])
exporter.setCreateScalaSources(true)
exporter.setTypeMappings(ScalaTypeMappings.create)
exporter.export(connection.getMetaData)
```

使用 Bean 类型生成：

```java
val directory = new java.io.File("target/jdbcgen2")
val namingStrategy = new DefaultNamingStrategy()
val exporter = new MetaDataExporter()
exporter.setNamePrefix("Q")
exporter.setPackageName("com.querydsl")
exporter.setSchemaPattern("PUBLIC")
exporter.setTargetFolder(directory)
exporter.setSerializerClass(classOf[ScalaMetaDataSerializer])
exporter.setBeanSerializerClass(classOf[ScalaBeanSerializer])
exporter.setCreateScalaSources(true)
exporter.setTypeMappings(ScalaTypeMappings.create)
exporter.export(connection.getMetaData)
```

##### 2.9.2.1。代码生成

SQL 元类型和投影的 Scala 源可以使用 querydsl-maven-plugin 生成。这是一个示例配置

```xml
<project>
  <build>
    <plugins>
      ...
      <plugin>
        <groupId>com.querydsl</groupId>
        <artifactId>querydsl-maven-plugin</artifactId>
        <version>${querydsl.version}</version>
        <configuration>
          <jdbcDriver>com.mysql.jdbc.Driver</jdbcDriver>
          <jdbcUrl>jdbc:mysql://localhost:3306/test</jdbcUrl>
          <jdbcUser>matko</jdbcUser>
          <jdbcPassword>matko</jdbcPassword>
          <packageName>com.example.schema</packageName>
          <targetFolder>${project.basedir}/src/main/scala</targetFolder>
          <exportBeans>true</exportBeans>
          <createScalaSources>true</createScalaSources>
        </configuration>
        <dependencies>
          <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.16</version>
          </dependency>
          <dependency>
            <groupId>com.querydsl</groupId>
            <artifactId>querydsl-scala</artifactId>
            <version>${querydsl.version}</version>
          </dependency>
          <dependency>
            <groupId>org.scala-lang</groupId>
            <artifactId>scala-library</artifactId>
            <version>${scala.version}</version>
          </dependency>
        </dependencies>
      </plugin>
      ...
    </plugins>
  </build>
</project>
```

执行的 maven 目标是 querydsl:export。

#### 2.9.3. 使用其他后端查询

当使用其他后端查询时，必须手动创建表达式模型，或者可以使用别名功能。

这是 JPA 的一个最小示例：

```java
@Entity
class User {
  @BeanProperty
  @Id
  var id: Integer = _;
  @BeanProperty
  var userName: String = _;
  @BeanProperty
  @ManyToOne
  var department: Department = _;
}

@Entity
class Department {
  @BeanProperty
  @Id
  var id: Integer = _;
  @BeanProperty
  var name: String = _;
}
```

以下是一些查询示例

列表

```java
val person = Person as "person"

selectFrom(person).where(person.firstName like "Rob%").fetch()
```

唯一的结果

```java
selectFrom(person).where(person.firstName like "Rob%").fetchOne()
```

长的 where 语句

```java
selectFrom(person)
  .where(person.firstName like "Rob%", person.lastName like "An%")
  .fetch()
```

排序

```java
selectFrom(person).orderBy(person.firstName asc).fetch()
```

不为空

```java
selectFrom(person)
  .where(person.firstName isEmpty, person.lastName isNotNull)
  .fetch()
```

查询创建的工厂方法是

```java
def query() = new JPAQuery(entityManager)
```

除了查询之外，您还需要可以像这样创建的变量

```java
val person = Person as "person"
```

注意：如果您将 Hibernate 与基于 XML 的配置一起使用，则 Scala 支持尚不可用。HibernateDomainExporter 目前只输出 Java 源文件。

## 3. 一般用法

一般用法部分涵盖了参考文档的教程部分未涵盖的方面。它遵循面向用例的结构。

### 3.1. 创建查询

Querydsl 中的查询构造涉及使用表达式参数调用查询方法。由于查询方法大多是特定于模块的，并且已经在教程部分中介绍过，因此本部分将重点介绍表达式。

表达式通常是通过访问域模块生成的表达式类型上的字段和调用方法来构造的。对于代码生成不适用的情况，可以改用构造表达式的通用方法。

#### 3.1.1. 复杂谓词

要构造复杂的布尔表达式，请使用 `com.querydsl.core.BooleanBuilder` 类。它实现了 Predicate 并且可以以级联形式使用：

```java
public List<Customer> getCustomer(String... names) {
    QCustomer customer = QCustomer.customer;
    JPAQuery<Customer> query = queryFactory.selectFrom(customer);
    BooleanBuilder builder = new BooleanBuilder();
    for (String name : names) {
        builder.or(customer.name.eq(name));
    }
    query.where(builder); // customer.name eq name1 OR customer.name eq name2 OR ...
    return query.fetch();
}
```

`BooleanBuilder` 是可变的，最初表示 null 并在每个 `and` 或 `or` 调用之后表示操作的结果。

#### 3.1.2. 动态表达式

`com.querydsl.core.types.dsl.Expressions` 类是用于动态表达式构造的静态工厂类。工厂方法由返回的类型命名，并且大多是自记录的。

一般来说，`Expressions` 类应该只在不能使用流畅的 DSL 形式的情况下使用，例如动态路径、自定义语法或自定义操作。

下面的表达式

```java
QPerson person = QPerson.person;
person.firstName.startsWith("P");
```

如果 Q-types 不可用，可以这样构造

```java
Path<Person> person = Expressions.path(Person.class, "person");
Path<String> personFirstName = Expressions.path(String.class, person, "firstName");
Constant<String> constant = Expressions.constant("P");
Expressions.predicate(Ops.STARTS_WITH, personFirstName, constant);
```

Path 实例表示变量和属性，Constant 是常量，Operations 是操作，TemplateExpression 实例可用于将表达式表达为字符串模板。

#### 3.1.3. 动态路径

除了基于 `Expressions` 的表达式创建之外，Querydsl 还为动态路径创建提供了更流畅的 API。

对于动态路径生成，可以使用 com.querydsl.core.types.dsl.PathBuilder 类。它扩展了“EntityPathBase”，可以用作类生成和路径生成别名使用的替代方案。

与 Expressions API 相比，PathBuilder 不提供对未知操作或自定义语法的直接支持，但语法更接近于普通的 DSL。

字符串属性：

```java
PathBuilder<User> entityPath = new
PathBuilder<User>(User.class, "entity");
// 完全通用的访问
entityPath.get("用户名");
// .. 或使用提供的类型
entityPath.get("userName", String.class);
// .. 和正确的签名
entityPath.getString("userName").lower();
```

列出具有组件类型的属性：

```java
entityPath.getList("list", String.class).get(0);
```

使用组件表达式类型：

```java
entityPath.getList("list", String.class, StringPath.class).get(0).lower();
```

具有键和值类型的映射属性：

```java
entityPath.getMap("map", String.class, String.class).get("key");
```

使用组件表达式类型：

```java
entityPath.getMap("map", String.class, String.class, StringPath.class).get("key").lower();
```

对于 PathBuilder 验证，可以使用 PathBuilderValidator。它可以被注入到构造函数中，并将被传递给新的 PathBuilder

```java
PathBuilder<Customer> customer = new PathBuilder<Customer>(Customer.class, "customer", validator);
```

PathBuilderValidator.FIELDS 将验证字段是否存在，PathBuilderValidator.PROPERTIES 验证 Bean 属性，而 JPAPathBuilderValidator 使用 JPA 元模型进行验证。

#### 3.1.4. Case 表达式

要构造 case-when-then-else 表达式，请像这样使用“CaseBuilder”类：

```java
QCustomer customer = QCustomer.customer;
Expression<String> cases = new CaseBuilder()
    .when(customer.annualSpending.gt(10000)).then("Premier")
    .when(customer.annualSpending.gt(5000)).then("Gold")
    .when(customer.annualSpending.gt(2000)).then("Silver")
    .otherwise("Bronze");
// case 表达式现在可以在投影或条件中使用
```

对于具有等号操作的 case 表达式，请改用以下更简单的形式：

```java
QCustomer customer = QCustomer.customer;
Expression<String> cases = customer.annualSpending
    .when(10000).then("Premier")
    .when(5000).then("Gold")
    .when(2000).then("Silver")
    .otherwise("Bronze");
// case 表达式现在可以在投影或条件中使用
```

JDOQL 中尚不支持案例表达式。

#### 3.1.5. Casting 表达式

为了避免表达式类型中的泛型签名，类型层次结构被展平。结果是所有生成的查询类型都是 com.querydsl.core.types.dsl.EntityPathBase 或 com.querydsl.core.types.dsl.BeanPath 的直接子类，并且不能直接转换为它们的逻辑超类型。

超类型引用不是直接的 Java 转换，而是可以通过 `_super` 字段访问。_super-field 在所有生成的查询类型中都可用，只有一个超类型：

```java
// 来自账户
QAccount 扩展 EntityPathBase<Account> {
    // ...
}

// 从 BankAccount 扩展 Account
QBankAccount extends EntityPathBase<BankAccount> {

    public final QAccount _super = new QAccount(this);

    // ...
}
```

要将超类型转换为子类型，您可以使用 EntityPathBase 类的 as 方法：

```java
QAccount account = new QAccount("account");
QBankAccount bankAccount = account.as(QBankAccount.class);
```

#### 3.1.6. 选择文字

可以通过常量表达式引用它们来选择文字。这是一个简单的例子

```java
query.select(Expressions.constant(1),
             Expressions.constant("abc"));
```

常量表达式经常用在子查询中。

### 3.2. 结果处理

Querydsl 提供了两种自定义结果的方法，FactoryExpressions 用于基于行的转换和 ResultTransformer 用于聚合。

`com.querydsl.core.types.FactoryExpression` 接口用于 Bean 创建、构造函数调用和更复杂对象的创建。可以通过 `com.querydsl.core.types.Projections` 类访问 Querydsl 的 FactoryExpression 实现的功能。

对于 `com.querydsl.core.ResultTransformer` 接口，`GroupBy` 是主要实现。

#### 3.2.1. 返回多列

从 Querydsl 3.0 开始，多列结果的默认类型是 `com.querydsl.core.Tuple`。Tuple 提供了一个类型安全的 Map 类接口来访问 Tuple 行对象中的列数据。

```java
List<Tuple> result = query.select(employee.firstName, employee.lastName)
                          .from(employee).fetch();
for (Tuple row : result) {
     System.out.println("firstName " + row.get(employee.firstName));
     System.out.println("lastName " + row.get(employee.lastName));
}}
```

这个例子也可以像这样通过 QTuple 表达式类编写

```java
List<Tuple> result = query.select(new QTuple(employee.firstName, employee.lastName))
                          .from(employee).fetch();
for (Tuple row : result) {
     System.out.println("firstName " + row.get(employee.firstName));
     System.out.println("lastName " + row.get(employee.lastName));
}}
```

#### 3.2.2. Bean 种群

在需要根据查询结果填充 Bean 的情况下，可以像这样使用 Bean 投影

```java
List<UserDTO> dtos = query.select(
    Projections.bean(UserDTO.class, user.firstName, user.lastName)).fetch();
```

当应该直接使用字段而不是设置器时，可以使用以下变体

```java
List<UserDTO> dtos = query.select(
    Projections.fields(UserDTO.class, user.firstName, user.lastName)).fetch();
```

#### 3.2.3. 构造函数用法

可以像这样使用基于构造函数的行转换

```java
List<UserDTO> dtos = query.select(
    Projections.constructor(UserDTO.class, user.firstName, user.lastName)).fetch();
```

作为通用构造函数表达式用法的替代方法，构造函数也可以使用 `QueryProjection` 注释进行注释：

```java
class CustomerDTO {

  @QueryProjection
  public CustomerDTO(long id, String name) {
     ...
  }

}
```

然后你可以在查询中像这样使用它

```java
QCustomer customer = QCustomer.customer;
JPQLQuery query = new HibernateQuery(session);
List<CustomerDTO> dtos = query.select(new QCustomerDTO(customer.id, customer.name))
                              .from(customer).fetch();
```

虽然该示例是特定于 Hibernate 的，但此功能在所有模块中都可用。

如果带有 QueryProjection 注释的类型不是带注释的实体类型，则可以像示例中那样使用构造函数投影，但如果带注释的类型是实体类型，则需要通过调用来创建构造函数投影查询类型的静态创建方法：

```java
@Entity
class Customer {

  @QueryProjection
  public Customer(long id, String name) {
     ...
  }

}
QCustomer customer = QCustomer.customer;
JPQLQuery query = new HibernateQuery(session);
List<Customer> dtos = query.select(QCustomer.create(customer.id, customer.name))
                           .from(customer).fetch();
```

或者，如果代码生成不是一个选项，您可以创建一个构造函数投影，如下所示：

```java
List<Customer> dtos = query
    .select(Projections.constructor(Customer.class, customer.id, customer.name))
    .from(customer).fetch();
```

#### 3.2.4. 结果聚合

`com.querydsl.core.group.GroupBy` 类提供聚合功能，可用于在内存中聚合查询结果。下面是一些使用示例。

聚合父子关系

```java
import static com.querydsl.core.group.GroupBy.*;

Map<Integer, List<Comment>> results = query.from(post, comment)
    .where(comment.post.id.eq(post.id))
    .transform(groupBy(post.id).as(list(comment)));
```

这将返回一个帖子 ID 到相关评论的地图。

多个结果列

```java
Map<Integer, Group> results = query.from(post, comment)
    .where(comment.post.id.eq(post.id))
    .transform(groupBy(post.id).as(post.name, set(comment.id)));
```

这会将帖子 ID 的映射返回到可以访问帖子名称和评论 ID 的 Group 实例。

Group 是 GroupBy 等价于 Tuple 接口。

更多示例可以在 [这里](https://github.com/querydsl/querydsl/blob/master/querydsl-collections/src/test/java/com/querydsl/collections/GroupByTest.java) 找到。

### 3.3. 代码生成

Java 6 APT 注释处理功能在 Querydsl 中用于 JPA、JDO 和 Mongodb 模块中的代码生成。本节介绍代码生成的各种配置选项以及 APT 使用的替代方法。

#### 3.3.1. 路径初始化

默认情况下，Querydsl 只初始化前两个级别的引用属性。如果需要更长的初始化路径，则必须通过 `com.querydsl.core.annotations.QueryInit` 注释在域类型中进行注释。QueryInit 用于需要深度初始化的属性。下面的例子演示了用法。

```java
@Entity
class Event {
    @QueryInit("customer.address")
    Account account;
}

@Entity
class Account {
    Customer customer;
}

@Entity
class Customer {
    String name;
    Address address;
    // ...
}
```

当事件路径被初始化为根路径/变量时，此示例强制执行 account.customer 路径的初始化。路径初始化格式也支持通配符，例如“customer.*”或只是“*”。

自动路径初始化取代了手动初始化，后者要求实体字段是非最终的。声明性格式的好处是可以应用于 Query 类型的所有顶级实例并启用最终实体字段的使用。

自动路径初始化是首选的初始化策略，但可以通过 Config 注解激活手动初始化，如下所述。

#### 3.3.2. 定制

Querydsl 的序列化可以通过包和类型上的 Config 注释来定制。他们自定义带注释的包或类型的序列化。

序列化选项是



**表 3.1. 配置选项**

| 姓名                  | 说明                                                    |
| --------------------- | ------------------------------------------------------- |
| entityAccessors       | 实体路径而不是公共最终字段的访问器方法（默认值：false） |
| listAccessors         | listProperty(int index) 样式方法（默认值：false）       |
| mapAccessors          | mapProperty(Key key) 样式访问器方法（默认值：false）    |
| createDefaultVariable | 生成默认变量（默认值：true）                            |
| defaultVariableName   | 默认变量的名称                                          |



下面是一些例子。

Entity类型序列化的自定义：

```java
@Config(entityAccessors=true)
@Entity
public class User {
    //...
}
```

包内容定制：

```java
@Config(listAccessors=true)
package com.querydsl.core.domain.rel;

import com.querydsl.core.annotations.Config;
```

如果要全局自定义序列化器配置，可以通过以下 APT 选项来实现



**表 3.2。APT 选项**

| 姓名                           | 说明                                                    |
| ------------------------------ | ------------------------------------------------------- |
| querydsl.entityAccessors       | 启用引用字段访问器                                      |
| querydsl.listAccessors         | 为直接索引列表访问启用访问器                            |
| querydsl.mapAccessors          | 为基于直接键的映射访问启用访问器                        |
| querydsl.prefix                | 覆盖查询类型的前缀（默认：Q）                           |
| querydsl.suffix                | 为查询类型设置后缀                                      |
| querydsl.packageSuffix         | 为查询类型包设置后缀                                    |
| querydsl.createDefaultVariable | 设置是否创建默认变量                                    |
| querydsl.unknownAsEmbeddable   | 设置未知的非注释类应被视为可嵌入的位置（默认值：false） |
| querydsl.includedPackages      | 要包含在代码生成中的包的逗号分隔列表（默认值：全部）    |
| querydsl.includedClasses       | 要包含在代码生成中的类名的逗号分隔列表（默认值：全部）  |
| querydsl.excludedPackages      | 要从代码生成中排除的包的逗号分隔列表（默认值：无）      |
| querydsl.excludedClasses       | 要从代码生成中排除的类名的逗号分隔列表（默认值：无）    |
| querydsl.useFields             | 设置字段是否用作元数据源（默认：true）                  |
| querydsl.useGetters            | 设置访问器是否用作元数据源（默认值：true）              |



使用 Maven APT 插件，例如这样工作：

```xml
<project>
  <build>
  <plugins>
    ...
    <plugin>
      <groupId>com.mysema.maven</groupId>
      <artifactId>apt-maven-plugin</artifactId>
      <version>1.1.3</version>
      <executions>
        <execution>
          <goals>
            <goal>process</goal>
          </goals>
          <configuration>
            <outputDirectory>target/generated-sources/java</outputDirectory>
            <processor>com.querydsl.apt.jpa.JPAAnnotationProcessor</processor>
            <options>
              <querydsl.entityAccessors>true</querydsl.entityAccessors>
              <querydsl.useFields>false</querydsl.useFields>
            </options>
          </configuration>
        </execution>
      </executions>
    </plugin>
    ...
  </plugins>
  </build>
</project>
```

或者，可以将 `maven-compiler-plugin` 配置为将 APT 直接挂接到编译中：

```xml
<project>
  <build>
    <plugins>
      ...
      <plugin>
        <artifactId>maven-compiler-plugin</artifactId>
        <configuration>
          <generatedSourcesDirectory>target/generated-sources/java</generatedSourcesDirectory>
          <compilerArgs>
            <arg>-Aquerydsl.entityAccessors=true</arg>
            <arg>-Aquerydsl.useFields=false</arg>
          </compilerArgs>
        </configuration>
        <dependencies>
          <dependency>
            <groupId>com.querydsl</groupId>
            <artifactId>querydsl-apt</artifactId>
            <version>${querydsl.version}</version>
            <classifier>jpa</classifier>
          </dependency>
          <dependency>
            <groupId>org.hibernate.javax.persistence</groupId>
            <artifactId>hibernate-jpa-2.1-api</artifactId>
            <version>1.0.0.Final</version>
          </dependency>
        </dependencies>
      </plugin>
      ...
    </plugins>
  </build>
</project>
```

请注意，在定义对 com.querydsl:querydsl-apt 的依赖时，您需要使用适当的分类器。这些额外的工件定义了要在 `META-INF/services/javax.annotation.processing.Processor` 中使用的注释处理器。

可用的分类器包括：

- `general`
- `hibernate`
- `jdo`
- `jpa`

使用此配置，查询对象可以在域对象的编译期间生成和编译它们的源。这也会自动将生成的源目录添加到 Maven 项目源根目录。

这种方法的最大优点是它还可以使用 `groovy-eclipse` 编译器处理带注释的 Groovy 类：

```xml
<project>
  <build>
  <plugins>
    ...
    <plugin>
      <artifactId>maven-compiler-plugin</artifactId>
      <configuration>
        <compilerId>groovy-eclipse-compiler</compilerId>
        <generatedSourcesDirectory>target/generated-sources/java</generatedSourcesDirectory>
        <compilerArgs>
          <arg>-Aquerydsl.entityAccessors=true</arg>
          <arg>-Aquerydsl.useFields=false</arg>
        </compilerArgs>
      </configuration>
      <dependencies>
        <dependency>
          <groupId>org.codehaus.groovy</groupId>
          <artifactId>groovy-eclipse-compiler</artifactId>
          <version>2.9.1-01</version>
        </dependency>
        <dependency>
          <groupId>org.codehaus.groovy</groupId>
          <artifactId>groovy-eclipse-batch</artifactId>
          <version>2.3.7-01</version>
        </dependency>
        <dependency>
          <groupId>com.querydsl</groupId>
          <artifactId>querydsl-apt</artifactId>
          <version>${querydsl.version}</version>
          <classifier>jpa</classifier>
        </dependency>
        <dependency>
          <groupId>org.hibernate.javax.persistence</groupId>
          <artifactId>hibernate-jpa-2.1-api</artifactId>
          <version>1.0.0.Final</version>
        </dependency>
      </dependencies>
    </plugin>
    ...
  </plugins>
  </build>
</project>
```

#### 3.3.3。自定义类型映射

自定义类型映射可用于属性以覆盖派生的 Path 类型。例如，在某些字符串路径上应阻止比较和字符串操作或需要添加对自定义类型的日期/时间支持的情况下，这可能很有用。内置了对 Joda 时间 API 和 JDK（java.util.Date、Calendar 和子类型）的日期/时间类型的支持，但使用此功能可能需要支持其他 API。

以下示例演示了用法：

```java
@Entity
public class MyEntity {
    @QueryType(PropertyType.SIMPLE)
    public String stringAsSimple;

    @QueryType(PropertyType.COMPARABLE)
    public String stringAsComparable;

    @QueryType(PropertyType.NONE)
    public String stringNotInQuerydsl;
}
```

值 `PropertyType.NONE` 可用于在查询类型生成中跳过属性。这种情况与 @Transient 或 @QueryTransient 注释属性不同，其中属性不会持久化。PropertyType.NONE 只是省略了 Querydsl 查询类型中的属性。

#### 3.3.4. 委托方法

要将静态方法声明为委托方法，请添加具有相应域类型作为值的 QueryDelegate 注释，并提供将相应 Querydsl 查询类型作为第一个参数的方法签名。

下面是一个来自单元测试的简单示例：

```java
@QueryEntity
public static class User {

    String name;

    User manager;

}
@QueryDelegate(User.class)
public static BooleanPath isManagedBy(QUser user, User other) {
    return user.manager.eq(other);
}
```

以及 QUser 查询类型中生成的方法：

```java
public BooleanPath isManagedBy(QUser other) {
    return DelegateTest.isManagedBy(this, other);
}
```

委托方法也可用于扩展内置类型。这里有些例子

```java
public class QueryExtensions {

    @QueryDelegate(Date.class)
    public static BooleanExpression inPeriod(DatePath<Date> date, Pair<Date,Date> period) {
        return date.goe(period.getFirst()).and(date.loe(period.getSecond()));
    }

    @QueryDelegate(Timestamp.class)
    public static BooleanExpression inDatePeriod(DateTimePath<Timestamp> timestamp, Pair<Date,Date> period) {
        Timestamp first = new Timestamp(DateUtils.truncate(period.getFirst(), Calendar.DAY_OF_MONTH).getTime());
        Calendar second = Calendar.getInstance();
        second.setTime(DateUtils.truncate(period.getSecond(), Calendar.DAY_OF_MONTH));
        second.add(1, Calendar.DAY_OF_MONTH);
        return timestamp.goe(first).and(timestamp.lt(new Timestamp(second.getTimeInMillis())));
    }

}
```

当为内置类型声明委托方法时，将创建具有正确委托方法用法的子类：

```java
public class QDate extends DatePath<java.sql.Date> {

    public QDate(BeanPath<? extends java.sql.Date> entity) {
        super(entity.getType(), entity.getMetadata());
    }

    public QDate(PathMetadata<?> metadata) {
        super(java.sql.Date.class, metadata);
    }

    public BooleanExpression inPeriod(com.mysema.commons.lang.Pair<java.sql.Date, java.sql.Date> period) {
        return QueryExtensions.inPeriod(this, period);
    }

}

public class QTimestamp extends DateTimePath<java.sql.Timestamp> {

    public QTimestamp(BeanPath<? extends java.sql.Timestamp> entity) {
        super(entity.getType(), entity.getMetadata());
    }

    public QTimestamp(PathMetadata<?> metadata) {
        super(java.sql.Timestamp.class, metadata);
    }

    public BooleanExpression inDatePeriod(com.mysema.commons.lang.Pair<java.sql.Date, java.sql.Date> period) {
        return QueryExtensions.inDatePeriod(this, period);
    }

}
```

#### 3.3.5. 非注释类型

可以通过创建 `@QueryEntities` 注释来为非注释类型创建 Querydsl 查询类型。只需将 QueryEntities 注释放入您选择的包中，并在 value 属性中镜像类。

要实际创建类型，请使用 `com.querydsl.apt.QuerydslAnnotationProcessor`。在 Maven 中，您可以这样做：

```xml
<project>
  <build>
  <plugins>
    ...
    <plugin>
      <groupId>com.mysema.maven</groupId>
      <artifactId>apt-maven-plugin</artifactId>
      <version>1.1.3</version>
      <executions>
        <execution>
          <goals>
            <goal>process</goal>
          </goals>
          <configuration>
            <outputDirectory>target/generated-sources/java</outputDirectory>
            <processor>com.querydsl.apt.QuerydslAnnotationProcessor</processor>
          </configuration>
        </execution>
      </executions>
    </plugin>
    ...
  </plugins>
  </build>
</project>
```

#### 3.3.6. 基于类路径的代码生成

对于注释 Java 源不可用的情况，例如使用不同的 JVM 语言（如 Scala 或 Groovy）或通过字节码操作添加注释，`GenericExporter` 类可用于扫描类路径以查找带注释的类并生成查询类型他们。

要使 GenericExporter 可用，请将 querydsl-codegen 模块的依赖项添加到您的项目中，或者更准确地说是“com.querydsl:querydsl-codegen:${querydsl.version}”。

以下是 JPA 的示例

```java
GenericExporter exporter = new GenericExporter();
exporter.setKeywords(Keywords.JPA);
exporter.setEntityAnnotation(Entity.class);
exporter.setEmbeddableAnnotation(Embeddable.class);
exporter.setEmbeddedAnnotation(Embedded.class);
exporter.setSupertypeAnnotation(MappedSuperclass.class);
exporter.setSkipAnnotation(Transient.class);
exporter.setTargetFolder(new File("target/generated-sources/java"));
exporter.export(DomainClass.class.getPackage());
```

这会将 DomainClass 类和子包的包中的所有 JPA 注释类导出到 target/generated-sources/java 目录。

##### 3.3.6.1. 通过 Maven 使用

querydsl-maven-plugin 的目标 generic-export、jpa-export 和 jdo-export 可用于通过 Maven 使用 GenericExporter。

不同的目标映射到 Querydsl、JPA 和 JDO 注释。

配置元素是



**表 3.3. Maven 配置**

| 类型     | 元素           | 说明                                             |
| -------- | -------------- | ------------------------------------------------ |
| File     | targetFolder   | 生成源的目标文件夹                               |
| boolean  | scala          | true，如果应该生成 Scala 源代码（默认值：false） |
| String[] | packages       | 为实体类自省的包                                 |
| boolean  | handleFields   | true，如果字段应被视为属性（默认值：true）       |
| boolean  | handleMethods  | true，如果 getter 应被视为属性（默认值：true）   |
| String   | sourceEncoding | 生成的源文件的字符集编码                         |
| boolean  | testClasspath  | true，如果应该使用测试类路径                     |



这是 JPA 注释类的示例

```xml
<project>
  <build>
    <plugins>
      ...
      <plugin>
        <groupId>com.querydsl</groupId>
        <artifactId>querydsl-maven-plugin</artifactId>
        <version>${querydsl.version}</version>
        <executions>
          <execution>
            <phase>process-classes</phase>
            <goals>
              <goal>jpa-export</goal>
            </goals>
            <configuration>
              <targetFolder>target/generated-sources/java</targetFolder>
              <packages>
                <package>com.example.domain</package>
              </packages>
            </configuration>
          </execution>
        </executions>
      </plugin>
      ...
    </plugins>
  </build>
</project>
```

这会将 com.example.domain 包和子包的 JPA 注释类导出到 target/generated-sources/java 目录。

如果您需要在此之后直接编译生成的源代码，那么您可以使用 `compile` 目标。

```xml
<execution>
  <goals>
    <goal>compile</goal>
  </goals>
  <configuration>
    <sourceFolder>target/generated-sources/scala</targetFolder>
  </configuration>
</execution>
```

`compile` 目标具有以下配置元素



**表 3.4. Maven 配置**

| 类型    | 元素            | 说明                         |
| ------- | --------------- | ---------------------------- |
| File    | sourceFolder    | 带有生成源的源文件夹         |
| String  | sourceEncoding  | 来源的字符集编码             |
| String  | source          | -source 编译器选项           |
| String  | target          | -target 编译器选项           |
| boolean | testClasspath   | true，如果应该使用测试类路径 |
| Map     | compilerOptions | 编译器选项                   |



除了 `sourceFolder` 之外的所有选项都是可选的。

##### 3.3.6.2. Scala 支持

如果您需要类的 Scala 输出，请使用以下配置的变体

```xml
<project>
  <build>
    <plugins>
      ...
      <plugin>
        <groupId>com.querydsl</groupId>
        <artifactId>querydsl-maven-plugin</artifactId>
        <version>${querydsl.version}</version>
        <dependencies>
          <dependency>
            <groupId>com.querydsl</groupId>
            <artifactId>querydsl-scala</artifactId>
            <version>${querydsl.version}</version>
          </dependency>
          <dependency>
            <groupId>org.scala-lang</groupId>
            <artifactId>scala-library</artifactId>
            <version>${scala.version}</version>
          </dependency>
        </dependencies>
        <executions>
          <execution>
            <goals>
              <goal>jpa-export</goal>
            </goals>
            <configuration>
              <targetFolder>target/generated-sources/scala</targetFolder>
              <scala>true</scala>
              <packages>
                <package>com.example.domain</package>
              </packages>
            </configuration>
          </execution>
        </executions>
      </plugin>
      ...
    </plugins>
  </build>
</project>
```

### 3.4. 别名用法

在不能选择代码生成的情况下，别名对象可以用作表达式构造的路径引用。它们可以通过 getter 方法调用通过代理 Java Bean 对象使用。

以下示例演示如何使用别名对象作为基于生成的类型创建表达式的替代品。

首先是使用 APT 生成的域类型的示例查询：

```java
QCat cat = new QCat("cat");
for (String name : queryFactory.select(cat.name).from(cat,cats)
    .where(cat.kittens.size().gt(0))
    .fetch()) {
    System.out.println(name);
}
```

现在有了 Cat 类的别名实例。美元方法内的调用 `c.getKittens()` 在内部转换为属性路径 `c.kittens`。

```java
Cat c = alias(Cat.class, "cat");
for (String name : select($(c.getName())).from($(c),cats)
    .where($(c.getKittens()).size().gt(0))
    .fetch()) {
    System.out.println(name);
}
```

要在代码中使用别名功能，请添加以下两个导入

```java
import static com.querydsl.core.alias.Alias.$;
import static com.querydsl.core.alias.Alias.alias;
```

以下示例是前一个示例的变体，其中对列表大小的访问发生在美元方法调用中。

```
Cat c = alias(Cat.class, "cat");
for (String name : queryFactory.select($(c.getName())).from($(c),cats)
    .where($(c.getKittens().size()).gt(0))
    .fetch()) {
    System.out.println(name);
}
```

别名的所有非原始和非最终类型属性都是别名本身。因此，您可以级联方法调用，直到在美元方法范围内达到原始类型或最终类型。例如

```java
$(c.getMate().getName())
```

在内部转换为 *c.mate.name*，但是

```java
$(c.getMate().getName().toLowerCase())
```

未正确转换，因为未跟踪 `toLowerCase()` 调用。

另请注意，您只能在别名类型上调用 getter、size()、contains(Object) 和 get(int)。所有其他调用都会引发异常。

## 4. 故障排除

### 4.1. 类型参数不足

Querydsl 在所有代码生成场景中都需要正确编码的 List Set、Collection 和 Map 属性。

当使用不正确编码的字段或 getter 时，您可能会出现以下堆栈跟踪：

```
java.lang.RuntimeException: Caught exception for field com.querydsl.jdo.testdomain.Store#products
  at com.querydsl.apt.Processor$2.visitType(Processor.java:117)
  at com.querydsl.apt.Processor$2.visitType(Processor.java:80)
  at com.sun.tools.javac.code.Symbol$ClassSymbol.accept(Symbol.java:827)
  at com.querydsl.apt.Processor.getClassModel(Processor.java:154)
  at com.querydsl.apt.Processor.process(Processor.java:191)
  ...
Caused by: java.lang.IllegalArgumentException: Insufficient type arguments for List
  at com.querydsl.apt.APTTypeModel.visitDeclared(APTTypeModel.java:112)
  at com.querydsl.apt.APTTypeModel.visitDeclared(APTTypeModel.java:40)
  at com.sun.tools.javac.code.Type$ClassType.accept(Type.java:696)
  at com.querydsl.apt.APTTypeModel.<init>(APTTypeModel.java:55)
  at com.querydsl.apt.APTTypeModel.get(APTTypeModel.java:48)
  at com.querydsl.apt.Processor$2.visitType(Processor.java:114)
  ... 35 more
```

有问题的字段声明及其更正示例：

```java
    private Collection names; // WRONG

    private Collection<String> names; // RIGHT

    private Map employeesByName; // WRONG

    private Map<String,Employee> employeesByName; // RIGHT
```

### 4.2. Querydsl Q 类型的多线程初始化

当从多个线程初始化 Querydsl Q-types 时，如果 Q-types 具有循环依赖关系，则可能会发生死锁。

一个易于使用的解决方案是在将类用于不同线程之前在单个线程中对其进行初始化。

com.querydsl.codegen.ClassPathUtils 类可以像这样使用：

```java
    ClassPathUtils.scanPackage(Thread.currentThread().getContextClassLoader(), packageToLoad);
```

将 packageToLoad 替换为要初始化的类的包。

### 4.3. JDK5使用

使用 JDK 5 编译项目时，您可能会遇到以下编译失败：

```
[INFO] ------------------------------------------------------------------------
[ERROR] BUILD FAILURE
[INFO] ------------------------------------------------------------------------
[INFO] Compilation failure
...
class file has wrong version 50.0, should be 49.0
```

Java 6.0 使用类文件版本 50.0，Java 5.0 使用 49.0。

Querydsl 仅针对 JDK 6.0 进行测试，因为我们广泛使用 APT，而 APT 仅在 JDK 6.0 之后才可用。

如果您想在 JDK 5.0 中使用它，您可能想尝试自己编译 Querydsl。
