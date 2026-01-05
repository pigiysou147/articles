# Spring Boot 3：从 2.x 平滑升级踩坑记录

## 背景
Spring Boot 3.0 基于 Spring Framework 6.0，是新一代的 Java 开发标准。虽然新特性很诱人（AOT 编译、虚拟线程支持等），但升级过程可谓“步步惊心”。
将一个 Spring Boot 2.7 的老项目升级到 3.1 时，通常会遇到以下核心坑点。

## 1. 基础环境升级
Spring Boot 3 最硬性的要求：**Java 17+**。
如果你还在用 Java 8，必须先升级 JDK。
- **Jakarta EE 迁移**：这是最大的工作量。
  `javax.*` 包名全部改为了 `jakarta.*`。
  - `javax.servlet` -> `jakarta.servlet`
  - `javax.persistence` -> `jakarta.persistence`
  *   *操作*：IDEA 全局替换，但要注意 `javax.sql` 等 JDK 自带的包不能换。

## 2. 依赖库的不兼容
很多老牌库还没发布适配 Jakarta EE 的版本。
- **MySQL Driver**: 建议升级到 `com.mysql:mysql-connector-j`。
- **Swagger/SpringFox**: SpringFox 已经死透了，**必须** 迁移到 `SpringDoc`。
  ```xml
  <dependency>
      <groupId>org.springdoc</groupId>
      <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
      <version>2.2.0</version>
  </dependency>
  ```
- **Redis**: 底层 jedis/lettuce 版本变动，建议直接依赖 `spring-boot-starter-data-redis` 管理版本。

## 3. 配置项变更
`application.yml` 里很多配置 key 变了。
- `spring.redis.*` -> `spring.data.redis.*`
- 这是一个非常繁琐的过程，推荐使用 **Spring Boot Migrator (SBM)** 工具，或者引入 `spring-boot-properties-migrator` 依赖，它会在启动时打印出过期的配置提示。

## 4. 核心代码改动
- **Spring Security 6.0**: 变化巨大。
  `WebSecurityConfigurerAdapter` 被移除。
  现在需要定义 `SecurityFilterChain` Bean。
  ```java
  @Bean
  public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
      http
          .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
          .httpBasic(withDefaults());
      return http.build();
  }
  ```
- **URL 匹配**: 默认不再支持尾部斜杠匹配（`/api/users` 和 `/api/users/` 被视为不同）。

## 总结
升级 Spring Boot 3 是大势所趋，不仅为了新特性，更为了安全性。
建议路线：Java 8 -> Java 17 -> Spring Boot 2.7 -> Spring Boot 3.x。
先解决 Jakarta EE 的包名问题，再逐步攻克第三方依赖。
