# Spring Boot 应用 “Connection is closed” 及 MySQL 空闲超时断开连接解决方案

在使用 Spring Boot + MySQL + HikariCP 的组合时，可能会在生产或测试环境中遭遇类似如下异常信息：

```ruby
org.springframework.jdbc.UncategorizedSQLException: PreparedStatementCallback; uncategorized SQLException for SQL [SELECT ...]; 
SQL state [null]; error code [0]; Connection is closed; 
nested exception is java.sql.SQLException: Connection is closed

org.springframework.jdbc.UncategorizedSQLException: PreparedStatementCallback; uncategorized SQLException for SQL [SELECT ...];
SQL state [HY000]; error code [4031]; 
The client was disconnected by the server because of inactivity. 
See wait_timeout and interactive_timeout for configuring this behavior.
...
Caused by: java.sql.SQLException: The client was disconnected by the server because of inactivity.
```

对于初次遇到此类问题的开发者来说，这往往令人困惑：连接池不是已经帮我们管理连接了吗？为什么还会出现连接被关闭的异常？本文将从问题现象出发，深入分析原因，并给出一些行之有效的解决方案，帮助你解决该问题。

## 问题现象

在一个正常运行的 Spring Boot 应用中，有时在调用数据库查询接口时会突然报出上述错误信息。其典型特征如下：

1. **Connection is closed**：表示在执行 SQL 时发现当前使用的数据库连接已经被关闭，无法继续使用。
2. **The client was disconnected by the server because of inactivity**：明确指出 MySQL 服务端在无操作的情况下，主动断开了该连接。

从报错信息可知，**客户端（即应用侧）认为自己的连接还在池中可用，但当真正执行 SQL 时却发现服务器端已经将连接回收或断开**。这通常意味着**连接池中的空闲连接在长时间未使用后失效**，当再次被应用取用时引发异常。

## 原因分析

问题的源头在于 **数据库空闲连接的超时与连接池管理机制的交互**。

1. **MySQL 服务端空闲超时机制**：MySQL 会针对长时间处于空闲状态的连接进行自动回收。决定这一过程的关键参数是 `wait_timeout` 和 `interactive_timeout`。如果连接在一段时间（默认通常是8小时）内未发生任何交互，MySQL 就会主动关闭该连接。
2. **连接池未定期验证连接可用性**：虽然使用了 HikariCP 这样优秀的连接池工具，但如果连接池在将连接归还到池中后，没有执行定期的心跳检测或验证查询，那么当这些连接在池中 “沉睡” 超过 MySQL 的超时时间后，它们会在 MySQL 端被关闭，但连接池本身仍认为这条连接是有效的空闲连接。
3. **连接重用导致的意外报错**：当应用再次从连接池取出这条被 MySQL 已断开的连接试图执行查询时，就会触发上述异常。

## 解决方案

### 调整 MySQL 服务端的超时参数（可选）

根据实际业务场景，如果长时间的空闲连接是业务特性（例如夜间业务量很低，白天又有高峰请求），可以适当增大 MySQL 的 `wait_timeout` 和 `interactive_timeout` 来减少连接被过早回收的概率。

```sql
SET GLOBAL wait_timeout = 28800;        -- 设置空闲超时为8小时
SET GLOBAL interactive_timeout = 28800; -- 同上
```

不过，这只是缓解手段，并非最终方案。过度延长空闲时间并不总是明智，因为这会占用数据库资源。

### 在 HikariCP 层面实施连接保活策略（推荐）

HikariCP 已经为我们提供了对连接健康检查的良好支持。通过适当地配置，它可以定期验证连接是否仍然有效，并在其被回收前主动进行一个类似 “心跳” 的查询。

关键配置项可包括：

* `connection-test-query`：用于验证连接有效性的查询，一般为简单的 `SELECT 1` 或 `SELECT 1 FROM DUAL`（根据数据库类型决定）。
* `validation-timeout`：验证连接可用性的超时时间。
* `keepalive-time`（HikariCP 3.4.1+ 提供）：为空闲连接添加心跳检测，即使在没有请求使用时，HikariCP 也会周期性地发送校验查询，以防止连接因空闲被断开。

示例配置（`application.yml`）：

```yaml
spring:
  datasource:
    hikari:
      # 测试连接的查询
      connection-test-query: SELECT 1
      # 每5分钟对空闲连接进行一次心跳测试
      keepalive-time: 300000
      # 验证连接有效性的超时时间为5秒
      validation-timeout: 5000
      # 连接池中的最大连接数（建议 < MySQL max.connections）
      maximum-pool-size: 100
      # 最小空闲连接数
      minimum-idle: 10
      # 连接在池中存活30分钟后重建，防止意外长连接问题
      max-lifetime: 1800000
      # 客户端从池中获取连接的最大等待时间（以毫秒为单位）, 超过此时间将抛出异常 30s
      connection-timeout: 30000
      # 启用自动提交模式
      auto-commit: true
      # 连接在池中允许空闲的最长时间（毫秒为单位） 10min
      idle-timeout: 600000
      # 连接池的名称
      pool-name: DatebookHikariCP
```

通过增加 `keepalive-time` 和 `connection-test-query` 等参数，当连接空闲达到一定时长时，HikariCP 会自动执行一次 “心跳” 查询，确保该连接始终处于活跃状态。如果该连接已被 MySQL 回收，心跳检测会及时发现并主动从池中移除该连接并新建连接，以确保在真正使用时不出故障。

> connection-test-query 说明：
>
> HikariCP 在较新版本中会在需要时自动使用 `Connection.isValid()` 来检测连接，无需手动指定 `connection-test-query`。如果你的数据库版本和 JDBC 驱动都支持 `isValid()` 方法，可以考虑去掉 `connection-test-query`，让 HikariCP 的内置机制发挥作用，从而减少不必要的测试查询开销。

> keepalive-time 说明：
>
> 上面将 `keepalive-time` 设置为5分钟(300000 ms)。这会在连接闲置5分钟后对其进行一次心跳检测，以确保其保持可用。如果你的数据库超时时间较长或者系统较为稳定，你也可以适度延长该时间间隔，减少心跳查询的开销。不过，这需要在业务闲置周期和数据库超时策略之间找到平衡点。

> maximum-pool-size 与 minimum-idle 说明：
>
> `maximum-pool-size: 100` 和 `minimum-idle: 10` 已算是较为通用的配置。但实际最优值要依据你的并发量、数据库负载、服务器资源等多方面因素来定。如果你的系统在高峰期请求量很大，且需要保持较多的“预热”连接，可以适当提高 `minimum-idle`，这样在高峰来临时连接池能更快提供足够连接。反之，如果系统较少需要瞬时高并发，你可以适当降低 `minimum-idle` 减少资源占用。

> 连接寿命与超时时间说明：
>
> `max-lifetime: 1800000` (30分钟)和 `idle-timeout: 600000` (10分钟) 通常是合理的默认值。`max-lifetime` 可以确保连接不会在池中永久存在（避免连接状态异常积累），`idle-timeout` 可以减少长期闲置连接占用资源。根据具体情况可考虑微调。例如，如果数据库 wait_timeout 为8小时，而你每隔5分钟有keepalive，会在连接超时前刷新连接，不见得需要30分钟就重建连接。但此设置能在更长周期内避免意外的连接状态问题。

> connection-timeout 说明：
>
> `connection-timeout: 30000` ms 表示当请求获取连接超过30秒未成功就抛出异常。这对多数场景足够宽裕，但如果你的系统对请求延迟敏感，希望快速降级，也可稍微缩短该时间。

> auto-commit: true 说明：
>
> 通常大多数查询和更新操作在 Spring 事务管理下是可以自动提交的。如果你的代码使用事务注解（@Transactional）来控制提交/回滚，这个设置一般没有问题。但请确保应用程序层面事务管理逻辑清晰。

### 业务侧定期 “热身”（非必要）

在大多数情况下，通过 HikariCP 的保活机制已经足够。如果业务场景特殊，可以在非高峰期通过定期任务（例如定时执行一次简单查询）来保持连接活性。但这通常是冗余且不太优雅的做法。