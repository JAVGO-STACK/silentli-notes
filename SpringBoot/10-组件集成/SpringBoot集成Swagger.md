# SpringBoot 集成 Swagger

在现代微服务架构或分布式服务中，API 管理和可视化已成为必备能力。无论是内部服务之间的调用，还是对外公开的接口，都需要一份清晰、准确且可在线实时更新的接口文档。Swagger（OpenAPI）加上 Knife4j 的增强特性，能够为我们的 API 提供自动化的文档生成与可视化展示，更直观地展示接口的请求参数、响应模型、认证方式和示例数据，从而提高开发、测试、运维、前后端协作的效率和准确性。

本篇将以 **Spring Boot 3.0.x** 为基础，结合 **DDD 分层架构**，给出从配置、依赖、实现到使用的完整方案。目标是实现企业级生产可用的标准参考架构，包括：

1. 清晰的分层设计：application、domain、infrastructure、interface 层次明确，代码组织合理。
2. 完整的 Swagger 配置及与 Knife4j 的无缝集成。
3. 安全可靠：配合权限控制、鉴权校验等机制（可在后续扩展中实现），对文档访问进行保护。
4. 可继承与可扩展的代码结构及注解使用说明，便于快速复制到新项目中使用。

## 技术选型与版本

* **Spring Boot**: 3.0.x
* **OpenAPI + Springdoc**: 使用 `springdoc-openapi-starter-webmvc-ui` 来替代传统的 swagger 2.x 方案。
* **Knife4j**: 在 OpenAPI 上进一步增强界面交互、Markdown 文档说明等特性。
* **JDK**: 17 及以上（Spring Boot 3.0.x 基本要求）

## 依赖与基础配置

### Maven 依赖

在 `pom.xml` 中加入所需依赖（请根据实际版本灵活调整）：

```xml
<dependencies>
    <!-- Springdoc OpenAPI Starter -->
    <dependency>
        <groupId>org.springdoc</groupId>
        <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
        <version>2.1.0</version>
    </dependency>
    
    <!-- Knife4j UI -->
    <dependency>
        <groupId>com.github.xiaoymin</groupId>
        <artifactId>knife4j-openapi3-starter</artifactId>
        <version>4.1.0</version>
    </dependency>
</dependencies>
```

### Spring 配置文件

在 `application.yml` 中添加 OpenAPI 和 Knife4j 配置项（根据需求灵活调整）：

```yaml
springdoc:
  api-docs:
    enabled: true
    path: /api-docs
  swagger-ui:
    enabled: true
    path: /swagger-ui.html
    operationsSorter: method
    tagsSorter: alpha
  group-configs:
    - group: default
      paths-to-match: /api/**

knife4j:
  enable: true
  setting:
    showTag: true
    enableSearch: true
    enableFilter: true
```

说明：

- `api-docs`：默认提供 json 格式的 API 文档信息。
- `swagger-ui`：Swagger UI 的访问路径和样式定制。
- `knife4j`：启用增强功能，如搜索、筛选、分组展示等。

## 工程分层结构

### 配置类（infrastructure/config/OpenApiConfig.java）

```java
package com.example.project.infrastructure.config;

import io.swagger.v3.oas.models.Components;
import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Info;
import io.swagger.v3.oas.models.security.*;
import org.springdoc.core.models.GroupedOpenApi;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * OpenAPI & Knife4j 配置类
 */
@Configuration
public class OpenApiConfig {

    /**
     * 定义全局OpenAPI基础信息
     */
    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
                .components(new Components()
                        .addSecuritySchemes("Authorization",
                                new SecurityScheme()
                                        .type(SecurityScheme.Type.HTTP)
                                        .scheme("bearer")
                                        .bearerFormat("JWT")
                                        .in(SecurityScheme.In.HEADER)
                                        .name("Authorization")
                        ))
                .info(new Info().title("企业级API文档")
                        .description("通过 Spring Boot 3.0.x 集成 OpenAPI & Knife4j 实现的在线API文档")
                        .version("1.0.0"));
    }

    /**
     * 分组API配置示例（可选）
     */
    @Bean
    public GroupedOpenApi defaultGroup() {
        return GroupedOpenApi.builder()
                .group("default")
                .packagesToScan("com.example.project.interface.controller")
                .pathsToMatch("/api/**")
                .build();
    }

}
```

此配置类：

- 为 OpenAPI 文档设定基本信息（标题、描述、版本）。
- 定义了安全方案（如 JWT Bearer）。
- 通过 `GroupedOpenApi` 对某些特定路径和包下的控制器进行分组管理。

### 控制器（interface/controller/UserController.java）

```java
package com.example.project.interface.controller;

import com.example.project.application.dto.UserDTO;
import com.example.project.interface.param.UserCreateParam;
import com.example.project.interface.vo.UserVO;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import org.springframework.web.bind.annotation.*;

/**
 * 用户管理相关接口
 */
@Tag(name = "用户管理", description = "提供用户的增删改查接口")
@RestController
@RequestMapping("/api/user")
public class UserController {

    @Operation(summary = "新增用户", description = "创建一个新用户")
    @PostMapping("/create")
    public UserVO createUser(@RequestBody UserCreateParam param) {
        // 这里调用 application 层的 service 完成用户创建逻辑
        // 最终返回一个 UserVO 对象
        // ...
        return new UserVO("1", param.getUsername(), param.getEmail());
    }

    @Operation(summary = "查询用户信息", description = "通过用户ID查询用户详情")
    @GetMapping("/{userId}")
    public UserVO getUserById(@PathVariable("userId") String userId) {
        // 这里通过 application 层查询用户信息（UserDTO）再转换为 UserVO 返回
        // ...
        return new UserVO(userId, "testUser", "testUser@example.com");
    }

    @Operation(summary = "列表查询", description = "查询用户列表")
    @GetMapping("/list")
    public List<UserVO> listUsers() {
        // 返回用户列表
        // ...
        return List.of(new UserVO("1", "UserA", "a@example.com"),
                       new UserVO("2", "UserB", "b@example.com"));
    }

}
```

注解说明：

- `@Tag` 为接口控制类分组，用于在文档中对控制器进行逻辑分块。
- `@Operation` 为接口操作级注解，用于描述接口的功能、参数、响应说明等。

### 接收参数对象（interface/param/UserCreateParam.java）

```java
package com.example.project.interface.param;

import io.swagger.v3.oas.annotations.media.Schema;

@Schema(name = "UserCreateParam", description = "创建用户时提交的参数")
public class UserCreateParam {

    @Schema(description = "用户名", example = "JohnDoe")
    private String username;

    @Schema(description = "用户邮箱地址", example = "john.doe@example.com")
    private String email;

    // ... getter/setter ...
}
```

### 返回对象（interface/vo/UserVO.java）

```java
package com.example.project.interface.vo;

import io.swagger.v3.oas.annotations.media.Schema;

@Schema(name = "UserVO", description = "用户视图对象")
public class UserVO {

    @Schema(description = "用户ID", example = "1")
    private String userId;

    @Schema(description = "用户名", example = "JohnDoe")
    private String username;

    @Schema(description = "邮箱地址", example = "john.doe@example.com")
    private String email;

    public UserVO(String userId, String username, String email) {
        this.userId = userId;
        this.username = username;
        this.email = email;
    }

    // ... getter/setter ...
}
```

## 实现思路与逻辑

1. **自动文档生成**：通过引入 `springdoc-openapi` 相关依赖，扫描 `controller` 中的 `@RestController`、`@Controller` 并结合 `@Operation`、`@Tag`、`@Schema` 等注解，自动生成符合 OpenAPI 规范的 API 文档数据。
2. **Knife4j 增强**：在生成的基础文档之上，Knife4j 提供更美观友好的 UI，增强了搜索、分组、快速测试等特性。通过配置 `application.yml` 的 `knife4j` 部分和在 `OpenApiConfig` 中定义安全方案等，可以使文档更加直观和可交互。
3. **可扩展性**：
   * **分组管理**：利用 `GroupedOpenApi` 可以对不同业务域的接口进行分组，减少文档冗余，清晰分类。
   * **安全配置**：在 `OpenAPI` 配置中嵌入安全策略（如 JWT），实现文档访问权限控制和接口测试时自动注入 Authorization Header。
   * **版本控制**：可通过 `info.version` 字段设定 API 版本，配合 `pathsToMatch` 切分新旧版本接口。
4. **DDD 分层与文档**：
   * 接口层（interface）仅负责处理 HTTP 请求和响应，清晰定义参数和返回值的 DTO/VO 对象，并借助注解让文档自动生成。
   * 应用层（application）封装业务用例（service）和数据转换（assembler），不直接关心文档生成，但会通过 DTO 对象间接影响文档的模型结构。
   * 领域层（domain）和基础设施层（infrastructure）不直接参与文档生成，但通过模型和仓储的设计确保代码逻辑与参数定义的清晰、稳定性，从而使得接口文档也更具有可演化性。

## 使用说明（集成后的使用）

在完成上述配置和代码放置后，新接入的开发者只需：

1. **Controller 开发**：在 `com.example.project.interface.controller` 包下新增 `Controller` 类，使用 `@Tag` 对控制器分组，使用 `@Operation` 对接口进行说明，入参和出参的类使用 `@Schema` 补充字段描述、示例值。
2. **DTO/VO 增强**：在定义应用层 DTO 或接口层 VO 时，为关键字段添加 `@Schema` 注解，以便文档自动识别和展示字段含义、示例。接入新的业务模块时，只需遵循相同的注解使用规范和包结构放置。
3. **访问文档**：启动应用后，访问 `http://localhost:8080/swagger-ui.html` 或 `http://localhost:8080/doc.html` (Knife4j)
   即可查看生成的增强文档页面。
4. **扩展与定制**：如需新增全局请求参数描述、安全策略或过滤器，可在 `OpenApiConfig` 中定义新的 `Bean` 或扩展 `OperationCustomizer`、`OpenApiCustomiser` 等接口。

## 常用注解说明

* **`@Tag(name = "...", description = "...")`**：为控制器提供分组和描述。
* **`@Operation(summary = "...", description = "...")`**：为每个接口方法添加操作说明，包括摘要和详细描述。
* **`@Schema(name = "...", description = "...", example = "...")`**：为入参与出参对象及其字段添加模型说明，让文档中更直观地展现字段含义。
* **`@Parameter`**（可选）：可用于方法参数级别的描述，如路径参数、请求头参数、查询参数。
* **`@RequestBody`** 和 `@RequestParam`、`@PathVariable`、`@RequestHeader`：结合 `@Operation` 和 `@Schema` 可以清晰定义请求参数类型。
* **`@SecurityRequirement(name = "Authorization")`**：在需要鉴权的接口方法上添加此注解，以在文档中展示需要传入授权 Token。
