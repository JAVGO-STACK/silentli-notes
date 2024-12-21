# MP - 多数据源

在当今的企业级应用中，随着业务规模的不断扩大和复杂性提升，单一数据源已无法满足高并发、大数据量、多样化的业务需求。为了有效地管理和切换多个数据源，实现读写分离、分库分表、数据迁移等功能，MyBatis-Plus 提供了强大的多数据源支持。本文将深入解析两款备受关注的多数据源扩展插件：开源生态的 **dynamic-datasource** 和企业级生态的 **mybatis-mate**，帮助开发者更全面地理解和应用多数据源技术。

## 为何需要多数据源？

在大型系统中，使用多数据源已成为一种常见的架构选择，其主要原因包括：

- **读写分离**：通过将读操作和写操作分配到不同的数据库，提高系统的吞吐量和性能。
- **负载均衡**：在多台数据库服务器之间分配请求，避免单点瓶颈。
- **分库分表**：将数据按一定规则拆分到不同的数据库或表中，提升查询效率和可扩展性。
- **多类型数据库支持**：同时使用关系型数据库和非关系型数据库，以满足不同的数据存储需求。
- **数据迁移与备份**：在不影响业务的情况下，实现数据的平滑迁移和备份。

## dynamic-datasource

### 基本概述

**dynamic-datasource** 是一个基于 Spring Boot 的开源多数据源启动器，旨在简化多数据源的配置和使用。它支持多种数据源类型，提供了丰富的特性，如数据源分组、动态增删数据源、分布式事务等，适用于各种复杂的业务场景。

### 主要特性

- **数据源分组**：支持按组管理数据源，便于实现读写分离、一主多从等策略。
- **敏感信息加密**：通过 `ENC()` 方法加密数据库配置信息，提升安全性。
- **独立初始化**：每个数据源可独立初始化，避免初始化冲突。
- **自定义注解**：支持继承 `@DS` 注解，实现个性化的数据源切换方式。
- **简化集成**：快速集成 Druid、HikariCP 等主流连接池，降低配置复杂度。
- **组件集成**：与 MyBatis-Plus、Quartz 等常用组件无缝集成。
- **动态数据源**：运行时动态增加或移除数据源，满足灵活的业务需求。
- **分布式事务**：基于 Seata 实现分布式事务控制，保证数据一致性。

### 使用约定

- **专注数据源切换**：框架只负责数据源的切换，不干涉具体的数据库操作。
- **数据源命名规范**：配置文件中，以下划线 `_` 分隔的数据源前缀为组名。
  - 例如：`slave_1` 和 `slave_2` 均属于 `slave` 组。
- **数据源切换方式**：支持通过组名或具体数据源名称进行切换。
- **默认数据源**：默认数据源名为 `master`，可通过 `spring.datasource.dynamic.primary` 修改。
- **注解优先级**：方法上的 `@DS` 注解优先于类上的 `@DS` 注解。

### 使用指南

#### (1) 引入依赖

在项目的 `pom.xml` 中添加依赖：

```XML
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>dynamic-datasource-spring-boot-starter</artifactId>
    <version>3.5.0</version> <!-- 请根据实际情况选择版本 -->
</dependency>
```

#### (2) 配置数据源

在 `application.yml` 文件中配置多个数据源：

```YAML
spring:
  datasource:
    dynamic:
      primary: master  # 设置默认数据源
      strict: false    # 严格模式，找不到数据源时是否抛出异常
      datasource:
        master:
          url: jdbc:mysql://localhost:3306/master_db
          username: root
          password: root123
          driver-class-name: com.mysql.cj.jdbc.Driver
        slave_1:
          url: jdbc:mysql://localhost:3306/slave_db1
          username: root
          password: root123
          driver-class-name: com.mysql.cj.jdbc.Driver
        slave_2:
          url: ENC(jdbc:mysql://localhost:3306/slave_db2)  # 使用 ENC() 加密
          username: ENC(root)
          password: ENC(root123)
          driver-class-name: com.mysql.cj.jdbc.Driver
```

#### (3) 切换数据源

使用 `@DS` 注解在类或方法上切换数据源：

```Java
@Service
@DS("slave")  // 默认使用 slave 组的数据源
public class UserServiceImpl implements UserService {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Override
    @DS("slave_1")  // 使用具体的 slave_1 数据源
    public List<Map<String, Object>> getUsers() {
        // 查询年龄大于 18 岁的用户
        return jdbcTemplate.queryForList("SELECT * FROM users WHERE age > 18");
    }
}
```

#### (4) 动态增删数据源

在运行时动态添加或移除数据源：

```Java
@Autowired
private DynamicRoutingDataSource dynamicRoutingDataSource;

// 添加数据源
DataSource newDataSource = createDataSource();
dynamicRoutingDataSource.addDataSource("newDS", newDataSource);

// 移除数据源
dynamicRoutingDataSource.removeDataSource("oldDS");
```

**dynamic-datasource** 的优势在于其高度的灵活性和丰富的特性，尤其是在读写分离和分布式事务方面表现出色。通过简单的注解和配置，即可实现复杂的数据源管理，大大降低了开发成本。此外，其开源免费的特性，使得它在社区中拥有广泛的用户基础和良好的口碑。

## mybatis-mate

### 基本概述

**mybatis-mate** 是 MyBatis-Plus 的企业级增强组件，内置了多数据源、数据权限、审计日志等高级特性。作为付费产品，它为企业用户提供专业的技术支持和更全面的功能，适用于对性能和安全性要求较高的项目。

### 主要特性

- **注解** **`@Sharding`**：通过注解方式灵活切换数据源。
- **多数据源配置**：支持复杂的数据源配置和分组。
- **动态加载卸载**：运行时动态加载或卸载数据源，方便运维管理。
- **分布式事务**：支持 JTA Atomikos 分布式事务，保证数据一致性。
- **高级功能集成**：内置数据权限、审计日志等功能，提升数据安全性。

### 使用指南

#### (1) 配置数据源

在 `application.yml` 中配置：

```YAML
mybatis-mate:
  sharding:
    primary: mysql  # 设置默认数据源
    datasource:
      mysql:
        - key: node1
          url: jdbc:mysql://localhost:3306/mysql_node1
          username: root
          password: root123
          driver-class-name: com.mysql.cj.jdbc.Driver
        - key: node2
          cluster: slave
          url: jdbc:mysql://localhost:3306/mysql_node2
          username: root
          password: root123
          driver-class-name: com.mysql.cj.jdbc.Driver
      postgresql:
        - key: node1
          url: jdbc:postgresql://localhost:5432/postgres_node1
          username: postgres
          password: postgres123
          driver-class-name: org.postgresql.Driver
```

#### (2) 切换数据源

使用 `@Sharding` 注解：

```Java
@Mapper
@Sharding("mysql")  // 默认使用 mysql 数据源
public interface UserMapper extends BaseMapper<User> {

    @Sharding("postgresql")  // 使用 postgresql 数据源
    Long countUsersByName(String name);

}
```

#### (3) 动态切换数据库节点

```Java
// 切换到 mysql 的从库 node2
ShardingKey.change("mysqlnode2");

try {
    // 执行数据库操作
    List<User> users = userMapper.selectList(null);
} finally {
    // 清除数据源上下文，避免影响后续操作
    ShardingKey.clear();
}
```

**mybatis-mate** 作为企业级组件，除了多数据源支持外，还提供了丰富的高级特性，如数据权限控制、审计日志、分布式锁等。这些功能对于大型企业项目尤为重要，能够满足高并发、高安全性的业务需求。付费服务也意味着更及时的技术支持和持续的功能更新，为企业用户提供了可靠的保障。

## dynamic-datasource VS mybatis-mate

| 特性               | dynamic-datasource | mybatis-mate         |
| :----------------- | :----------------- | :------------------- |
| **开源/付费**      | 开源免费           | 付费企业级组件       |
| **数据源切换**     | `@DS` 注解         | `@Sharding` 注解     |
| **动态增删数据源** | 支持               | 支持                 |
| **分布式事务**     | 基于 Seata         | 基于 JTA Atomikos    |
| **高级功能**       | 基本功能           | 数据权限、审计日志等 |
| **技术支持**       | 社区支持           | 官方技术支持         |
| **适用场景**       | 中小型项目         | 大型企业项目         |

> **总结**：
>
> - **dynamic-datasource** 更适合预算有限、需求相对简单的项目，具有上手快、配置简单的优势。
> - **mybatis-mate** 适用于对功能完善性和技术支持有高要求的企业级项目，能够提供更全面的解决方案。