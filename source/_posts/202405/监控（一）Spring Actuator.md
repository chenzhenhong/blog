---
title: 监控（一）Spring Actuator
date: 2024-05-02 12:23:23
category:
- 实践
tags: 
- 监控
- Spring Acutator
---

> [Spring Actuator官方文档](https://docs.spring.io/spring-boot/reference/actuator/enabling.html)

Spring Actuator提供了监控了管理的功能，可通过JMX或HTTP暴露端点，开箱即用。

本文主要描述HTTP，JMX类似。

### 一、引入Acutator
```xml
<dependencies>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-actuator</artifactId>
	</dependency>
</dependencies>
```
导入以上依赖后，启动Spring Boot项目，日志会如下打印
```
INFO 11310 --- [           main] o.s.b.a.e.web.EndpointLinksResolver      : Exposing 1 endpoint(s) beneath base path '/actuator'
```
默认情况下，只会自动暴露`/health`端点，可以通过访问`/actuator`获取当前已暴露的端点（需配置management.endpoints.web.discovery.enabled=true，默认为true），例如：
```json
{
  "_links": {
    "self": {
      "href": "http://localhost:8080/actuator",
      "templated": false
    },
    "health": {
      "href": "http://localhost:8080/actuator/health",
      "templated": false
    },
    "health-path": {
      "href": "http://localhost:8080/actuator/health/{*path}",
      "templated": true
    }
  }
}
```

### 二、自定义配置
1. 控制暴露的端点
```properties
# exclude优先级高于include，可以使用*通配符表示全部
management.endpoints.web.exposure.exclude=
management.endpoints.web.exposure.include=health,info

# 也可以通过编程的方法控制，实现EndpointFilter
```

2. 配置每个端点
```properties
# 端点的配置项可以通过management.endpoint.<id>.<option>=xxx设置
management.endpoint.health.show-details=always  # health端点显示详细信息
```

### 三、端点安全
在没有引入Spring Security的情况下，exposure的端点是完全公开的；如果引入Spring Security则会保护除/health之外的端点。

1. Spring security自定义安全策略
```java
import org.springframework.boot.actuate.autoconfigure.security.servlet.EndpointRequest;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

import static org.springframework.security.config.Customizer.withDefaults;

@Configuration(proxyBeanMethods = false)
public class MySecurityConfiguration {

	@Bean
	public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
		http.securityMatcher(EndpointRequest.toAnyEndpoint());
		http.authorizeHttpRequests((requests) -> requests.anyRequest().hasRole("ENDPOINT_ADMIN"));
		http.httpBasic(withDefaults());
		return http.build();
	}
}
```

2. 无Spring Security自定义安全策略

通过添加一个`Filter`实现拦截（`/actuator`不是常规Controller，无法通过`HandlerInterceptor`拦截）

```java
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.util.Base64;

@Component
public class ActuatorBasicAuthFilter extends OncePerRequestFilter {

    private static final String USERNAME = "admin";
    private static final String PASSWORD = "123456";

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        String path = request.getRequestURI();
        if (!path.startsWith("/actuator")) {
            filterChain.doFilter(request, response); // 非 /actuator 跳过
            return;
        }

        String authHeader = request.getHeader("Authorization");
        if (authHeader != null && authHeader.startsWith("Basic ")) {
            String base64Credentials = authHeader.substring("Basic ".length());
            String credentials = new String(Base64.getDecoder().decode(base64Credentials), StandardCharsets.UTF_8);
            String[] values = credentials.split(":", 2);

            if (values.length == 2 && USERNAME.equals(values[0]) && PASSWORD.equals(values[1])) {
                filterChain.doFilter(request, response); // 验证通过
                return;
            }
        }

        // 验证失败
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        response.setHeader("WWW-Authenticate", "Basic realm=\"Actuator\""); // 控制浏览器弹输入账户密码的窗口
        response.getWriter().write("Unauthorized");
    }
}
```
3. 独立端口
```properties
management.server.port=8081         # 独立端口
```

4. 启用审计
```java
import org.springframework.boot.actuate.audit.AuditEventRepository;
import org.springframework.boot.actuate.audit.InMemoryAuditEventRepository;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AuditEventConfigurer {
    @Bean
    public AuditEventRepository auditEventRepository() {
        // 测试用，生产应持久化保存
        return new InMemoryAuditEventRepository();
    }
}
```

### 四、自定义端点
通过`@Endpoint`、`@ReadOperation`、`@WriteOperation`、`@DeleteOperation`注解定义个Bean。

不同注解对应不同HTTP Method：
|      Operation    | HTTP method |
|:------------------|:-------|
| @ReadOperation    | GET     |
| @WriteOperation   | POST    |
| @DeleteOperation  | DELETE  |


```java
import org.springframework.boot.actuate.endpoint.annotation.Endpoint;
import org.springframework.boot.actuate.endpoint.annotation.ReadOperation;
import org.springframework.boot.actuate.endpoint.annotation.WriteOperation;
import org.springframework.stereotype.Component;

@Component
@Endpoint(id = "test") // 通过/actuator/test访问
public class MyEndpoint {
    private String display = "hello";

    @ReadOperation
    public String getDisplay() {
        return display;
    }

    @WriteOperation
    public void setDisplay(String display) {
        this.display = display;
    }
}
```

### 五、自定义健康检查
实现`HealthIndicator`，其中Bean名字会作为访问路径，如下代码可通过`/actuator/health/my`访问。

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.boot.actuate.health.Status;
import org.springframework.stereotype.Component;

@Component
public class MyHealthIndicator implements HealthIndicator {
    @Autowired
    private MyEndpoint myEndpoint;

    @Override
    public Health health() {
        String display = myEndpoint.getDisplay();

        Status status;
        if (display == null || display.isEmpty()) {
            status = Status.UNKNOWN;
        } else if (display.equals("ok")) {
            status = Status.UP;
        } else {
            status = Status.DOWN;
        }

        return new Health.Builder()
                .withDetail("display", display)
                .status(status)
                .build();
    }
}
```

输出如下：
```json
{
  "status": "DOWN",
  "details": {
    "display": "hello"
  }
}
```

### 六、应用信息
应用信息通过`/actuator/info`访问，信息来源于全部定义的`InfoContributor` Bean，Spring Boot自定配置了`BuildInfoContributor`、`EnvironmentInfoContributor`、`GitInfoContributor`、`JavaInfoContributor`、`OsInfoContributor`、`ProcessInfoContributor`、`SslInfoContributor`。

以下演示`GItInfoContributor`使用：

1. 添加`git-commit-id-maven-plugin`插件用于生成`git.properties`文件
```xml
<plugin>
    <groupId>io.github.git-commit-id</groupId>
    <artifactId>git-commit-id-maven-plugin</artifactId>
    <configuration>
        <runOnlyOnce>true</runOnlyOnce>
        <generateGitPropertiesFile>true</generateGitPropertiesFile>
        <commitIdGenerationMode>full</commitIdGenerationMode>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>revision</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```
点击编译，会在classes中生成一个`git.properties`文件
```properties
#Generated by Git-Commit-Id-Plugin
git.branch=master
git.build.host=primary-pc
git.build.time=2025-07-11T11\:22\:40+08\:00
git.build.user.email=2246467700@qq.com
git.build.user.name=sanne
git.build.version=0.0.1-SNAPSHOT
git.closest.tag.commit.count=
git.closest.tag.name=
git.commit.author.time=2025-07-11T11\:05\:09+08\:00
git.commit.committer.time=2025-07-11T11\:05\:09+08\:00
git.commit.id.abbrev=d5cb677
git.commit.id.describe=d5cb677-dirty
git.commit.id.describe-short=d5cb677-dirty
git.commit.id.full=d5cb6779c9705fa6291d7c87cd29aeedd70dbd6a
git.commit.message.full=init
git.commit.message.short=init
git.commit.time=2025-07-11T11\:05\:09+08\:00
git.commit.user.email=2246467700@qq.com
git.commit.user.name=sanne
git.dirty=true
git.local.branch.ahead=NO_REMOTE
git.local.branch.behind=NO_REMOTE
git.remote.origin.url=Unknown
git.tag=
git.tags=
git.total.commit.count=1
```
查看`/actuator/info`，输出如下（显示详细信息需配置management.info.git.mode=full）：
```json
{
  "git": {
    "branch": "master",
    "commit": {
      "id": "d5cb677",
      "time": "2025-07-11T11:05:09+08:00"
    }
  }
}
```