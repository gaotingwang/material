三大特性：

- 组件自动装配：Web MVC、Web Flux、JDBC等
- 嵌入式web容器：Tomcat、Jetty及Undertow
- 生产准备特性：指标、健康检查、外部化配置等



## 自动装配

### 模式注解

是一种用于声明在应用中扮演“组件”角色的注解，实现单个功能。当任何组件标注它时，被视作组件扫描的候选对象。

| 注解          | 场景说明          |
| ------------- | ----------------- |
| `@Repository` | 数据仓储模式注解  |
| `@Component`  | 通用组件模式注解  |
| `@Service`    | 服务模式注解      |
| `@Controller` | Web控制器模式注解 |



### 模块装配`@Enable`

所谓"模块"是指具备相同领域的功能组件集合，组合形成一个独立的单元，是一群功能。比如 Caching模块、JMX模块。

#### 实现方式

##### 注解驱动方式

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
@Documented
@Import({DelegatingWebMvcConfiguration.class})
public @interface EnableWebMvc {
}
```

```java
// 加载单一配置内容,只有一种模式
@Configuration
public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport {
    ...
}
```



##### 接口编程方式

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
@Documented
@Import({CacheConfigurationImportSelector.class})
public @interface EnableCaching {
}
```

```java
static class CacheConfigurationImportSelector implements ImportSelector {
    CacheConfigurationImportSelector() {
    }
	
    // 可条件化，支持多种模式的配置
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        CacheType[] types = CacheType.values();
        String[] imports = new String[types.length];

        for(int i = 0; i < types.length; ++i) {
            imports[i] = CacheConfigurations.getConfigurationClass(types[i]);
        }

        return imports;
    }
}
```



### 条件配置

允许在Bean装配时增加前置条件判断

| Spring注解     | 场景说明       | 起始版本 |
| -------------- | -------------- | -------- |
| `@Profile`     | 配置化条件装配 | 3.1      |
| `@Conditional` | 编程条件装配   | 4.0      |

#### 实现方式

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional({OnClassCondition.class}) // 实现方式在这里
public @interface ConditionalOnClass {
    Class<?>[] value() default {};

    String[] name() default {};
}
```

## Spring Boot 自动装配

底层装配技术

- Spring 模式注解装配
- Spring `@Enable`模块装配
- Spring 条件装配
- Spring 工厂加载机制
  - 实现类：`SpringFactoriesLoader`
  - 配置资源：`META-INF/spring.factories`



















































