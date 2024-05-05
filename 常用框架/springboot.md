启动流程

1. 首先从main找到run()方法，在执行run()方法之前new一个SpringApplication对象

2. 进入run()方法，创建应用监听器SpringApplicationRunListeners开始监听
3. 然后加载SpringBoot配置环境(ConfigurableEnvironment)，然后把配置环境(Environment)加入监听对象中
4. 然后加载应用上下文(ConfigurableApplicationContext)，当做run方法的返回对象
5. 最后创建Spring容器，refreshContext(context)，实现starter自动化配置和bean的实例化等工作。

### 自动装配原理

通过SPI方式进行优化

SpringBoot 在启动时会扫描外部引用 jar 包中的`META-INF/spring.factories`文件，将文件中配置的类型信息加载到 Spring 容器（此处涉及到 JVM 类加载机制与 Spring 的容器知识），并执行类中定义的各种操作。对于外部 jar 来说，只需要按照 SpringBoot 定义的标准，就能将自己的功能装置进 SpringBoot

自动装配可以简单理解为：**通过注解或者一些简单的配置就能在 Spring Boot 的帮助下实现某块功能**

- `@EnableAutoConfiguration`：是启用 SpringBoot 的自动配置机制
- `EnableAutoConfiguration` 只是一个简单地注解，自动装配核心功能的实现实际是通过 `AutoConfigurationImportSelector`类

`AutoConfigurationImportSelector` 类实现了 `ImportSelector`接口，也就实现了这个接口中的 `selectImports`方法，该方法主要用于**获取所有符合条件的类的全限定类名，这些类需要被加载到 IoC 容器中**

~~~java
public String[] selectImports(AnnotationMetadata annotationMetadata) {
        // <1>.判断自动装配开关是否打开
        if (!this.isEnabled(annotationMetadata)) {
            return NO_IMPORTS;
        } else {
          //<2>.获取所有需要装配的bean
            AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader.loadMetadata(this.beanClassLoader);
            AutoConfigurationImportSelector.AutoConfigurationEntry autoConfigurationEntry = this.getAutoConfigurationEntry(autoConfigurationMetadata, annotationMetadata);
            return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
        }
    }
~~~

重点关注一下`getAutoConfigurationEntry()`方法，负责加载配置类

方法调用了链如下

![img](https://oss.javaguide.cn/github/javaguide/system-design/framework/spring/3c1200712655443ca4b38500d615bb70~tplv-k3u1fbpfcp-watermark.png)

源码详细

~~~java
private static final AutoConfigurationEntry EMPTY_ENTRY = new AutoConfigurationEntry();

AutoConfigurationEntry getAutoConfigurationEntry(AutoConfigurationMetadata autoConfigurationMetadata, AnnotationMetadata annotationMetadata) {
        //<1>.判断自动装配开关是否打开
        if (!this.isEnabled(annotationMetadata)) {
            return EMPTY_ENTRY;
        } else {
            //<2>.用于获取EnableAutoConfiguration注解中的 exclude 和 excludeName
            AnnotationAttributes attributes = this.getAttributes(annotationMetadata);
            //<3>.获取需要自动装配的所有配置类，读取META-INF/spring.factories
            //所有 Spring Boot Starter 下的META-INF/spring.factories都会被读取到
            //但不会全部加载
            List<String> configurations = this.getCandidateConfigurations(annotationMetadata, attributes);
            //<4>先移除重复的
            configurations = this.removeDuplicates(configurations);
            Set<String> exclusions = this.getExclusions(annotationMetadata, attributes);
            this.checkExcludedClasses(configurations, exclusions);
            configurations.removeAll(exclusions);
            //5.通过@ConditionalOnXXX 中的所有条件都满足，该类才会生效
            //这里的filter
            configurations = this.filter(configurations, autoConfigurationMetadata);
            this.fireAutoConfigurationImportEvents(configurations, exclusions);
            return new AutoConfigurationImportSelector.AutoConfigurationEntry(configurations, exclusions);
        }
    }
~~~



~~~java
@Configuration
// 检查相关的类：RabbitTemplate 和 Channel是否存在
// 存在才会加载
@ConditionalOnClass({ RabbitTemplate.class, Channel.class })
@EnableConfigurationProperties(RabbitProperties.class)
@Import(RabbitAnnotationDrivenConfiguration.class)
public class RabbitAutoConfiguration {
}
~~~

Spring Boot 通过`@EnableAutoConfiguration`开启自动装配，通过 SpringFactoriesLoader 最终加载`META-INF/spring.factories`中的自动配置类实现自动装配，自动配置类其实就是通过`@Conditional`按需加载的配置类，想要其生效必须引入`spring-boot-starter-xxx`包实现起步依赖

### 依赖注入

依赖注入是一种消除类之间依赖关系的设计模式。例如，A类要依赖B类，A类不再直接创建B类，而是把这种依赖关系配置在外部xml文件（或java config文件）中，然后由Spring容器根据配置信息创建、管理bean类

Spring 的依赖注入（Dependency Injection，DI）是指将对象的依赖关系由容器在创建对象时动态注入，而不是由对象自己创建或查找依赖对象。其核心原理包括以下几个方面：

1. **容器管理对象**：
   - 在 Spring 中，应用程序的组件（Bean）由 Spring 容器负责管理，包括创建、销毁和维护其生命周期。
   - Spring 容器通过配置文件（XML 配置、注解配置或 Java 配置）来描述应用程序中的组件及其依赖关系。
2. **依赖关系描述**：
   - 在配置文件中，通过声明 Bean 的定义和依赖关系，来描述对象之间的依赖关系。
   - 可以使用构造函数注入、Setter 方法注入、字段注入等方式来描述对象之间的依赖关系。
3. **依赖注入方式**：
   - Spring 提供了多种依赖注入方式，包括构造函数注入、Setter 方法注入、字段注入等。
   - 在对象创建时，Spring 容器会根据配置文件中的定义，自动注入所需的依赖对象，完成对象的初始化。
4. **反射机制**：
   - Spring 使用 Java 的反射机制来动态地创建对象和注入依赖关系，从而实现了对象之间的松耦合。
   - 通过反射机制，Spring 可以在运行时动态地创建对象，并在创建对象时注入所需的依赖对象。
5. **容器初始化**：
   - 在应用程序启动时，Spring 容器会初始化并加载配置文件，解析配置信息，并根据配置信息创建和管理对象。
   - Spring 容器会根据配置文件中的定义，实例化对象并注入所需的依赖对象，完成应用程序的初始化工作。

总的来说，Spring 的依赖注入原理是通过容器管理对象，并根据配置文件中的定义，动态地创建对象并注入所需的依赖对象，从而实现了对象之间的松耦合和灵活性