---
layout: post
title: Spring Boot - 自动配置原理
date: 2019-03-21
categories:
    - Spring
comments: true
permalink: spring-auto-config.html
---

# 1. 自动配置原理

Spring Boot 中的配置体系是一套强大而复杂的体系，其中最基础、最核心的要数自动配置（AutoConfiguration）机制了。我们可以先从`@SpringBootApplication`了解一下

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
...
}
```

查看源码可以发现 `@SpringBootApplication` 注解实际上是一个组合注解，它由三个注解组合而成，分别是 `@SpringBootConfiguration`、`@EnableAutoConfiguration` 和 `@ComponentScan`。

- **@ComponentScan **

`@ComponentScan` 注解就是扫描基于 `@Component` 等注解所标注的类所在包下的所有需要注入的类，并把相关 Bean 定义批量加载到容器中。

- ** @SpringBootConfiguration**

`@SpringBootConfiguration` 注解比较简单，事实上它是一个空注解，只是使用了 Spring 中的 `@Configuration` 注解。`@Configuration` 注解比较常见，提供了 JavaConfig 配置类实现。所以我们可以认为：`@SpringBootConfiguration` = `@Configuration`

- **@EnableAutoConfiguration**

`@EnableAutoConfiguration` 注解是我们需要重点剖析的对象

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {

	String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

	Class<?>[] exclude() default {};

	String[] excludeName() default {};

}
```

这里我们关注两个新注解，`@AutoConfigurationPackage` 和 `@Import(AutoConfigurationImportSelector.class)`。

- **@AutoConfigurationPackage**

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage {

	String[] basePackages() default {};

	Class<?>[] basePackageClasses() default {};

}
```

从命名上讲，在这个注解中我们对该注解所在包下的类进行自动配置，而在实现方式上用到了 Spring 中的 `@Import` 注解。

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Import {

	Class<?>[] value();

}
```

在 `@Import` 注解的属性中可以设置需要引入的类名，例如  `@AutoConfigurationPackage` 注解上的  `@Import(AutoConfigurationPackages.Registrar.class)`。根据该类的不同类型，Spring 容器针对 `@Import` 注解有以下四种处理方式：

- 如果该类实现了 `ImportSelector` 接口，Spring 容器就会实例化该类，并且调用其 selectImports 方法；
- 如果该类实现了 `DeferredImportSelector` 接口，则 Spring  容器也会实例化该类并调用其 selectImports方法。`DeferredImportSelector` 继承了  ImportSelector，区别在于 `DeferredImportSelector` 实例的 selectImports 方法调用时机晚于  `ImportSelector` 的实例，要等到 `@Configuration` 注解中相关的业务全部都处理完了才会调用；
- 如果该类实现了 `ImportBeanDefinitionRegistrar` 接口，Spring 容器就会实例化该类，并且调用其 registerBeanDefinitions 方法；
- 如果该类没有实现上述三种接口中的任何一个，Spring 容器就会直接实例化该类。

我们在回过头来看`AutoConfigurationPackages.Registrar`

```
static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {

	@Override
	public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
		register(registry, new PackageImports(metadata).getPackageNames().toArray(new String[0]));
	}

	@Override
	public Set<Object> determineImports(AnnotationMetadata metadata) {
		return Collections.singleton(new PackageImports(metadata));
	}

}
```

可以看到这个 Registrar 类实现了 `ImportBeanDefinitionRegistrar` 接口并重写了 registerBeanDefinitions 方法，这个方法会调用``AutoConfigurationPackages`自身的register方法

```
public static void register(BeanDefinitionRegistry registry, String... packageNames) {
	if (registry.containsBeanDefinition(BEAN)) {
		BeanDefinition beanDefinition = registry.getBeanDefinition(BEAN);
		ConstructorArgumentValues constructorArguments = beanDefinition.getConstructorArgumentValues();
		constructorArguments.addIndexedArgumentValue(0, addBasePackages(constructorArguments, packageNames));
	}
	else {
		GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
		beanDefinition.setBeanClass(BasePackages.class);
		beanDefinition.getConstructorArgumentValues().addIndexedArgumentValue(0, packageNames);
		beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
		registry.registerBeanDefinition(BEAN, beanDefinition);
	}
}
```

这个方法的逻辑是先判断整个 Bean 有没有被注册，如果已经注册则获取 Bean 的定义，通过 Bean  获取构造函数的参数并添加参数值；如果没有，则创建一个新的 Bean 的定义，设置 Bean 的类型为  AutoConfigurationPackages 类型并进行 Bean 的注册。

- **AutoConfigurationImportSelector**

`AutoConfigurationImportSelector`实现的是`@Import`的第二种情况`DeferredImportSelector` 接口，所以会执行如下所示的 selectImports 方法

```
@Override
public String[] selectImports(AnnotationMetadata annotationMetadata) {
	if (!isEnabled(annotationMetadata)) {
		return NO_IMPORTS;
	}
	AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(annotationMetadata);
	return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
}

protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
	if (!isEnabled(annotationMetadata)) {
		return EMPTY_ENTRY;
	}
	AnnotationAttributes attributes = getAttributes(annotationMetadata);
	List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
	configurations = removeDuplicates(configurations);
	Set<String> exclusions = getExclusions(annotationMetadata, attributes);
	checkExcludedClasses(configurations, exclusions);
	configurations.removeAll(exclusions);
	configurations = getConfigurationClassFilter().filter(configurations);
	fireAutoConfigurationImportEvents(configurations, exclusions);
	return new AutoConfigurationEntry(configurations, exclusions);
}
```

这段代码的核心是通过 getCandidateConfigurations 方法获取 configurations 集合并进行过滤。

```
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
   List<String> configurations = SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(),
         getBeanClassLoader());
   Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you "
         + "are using a custom packaging, make sure that file is correct.");
   return configurations;
}

protected Class<?> getSpringFactoriesLoaderFactoryClass() {
	return EnableAutoConfiguration.class;
}
```

`SpringFactoriesLoader` 会查找所有  META-INF/spring.factories 文件夹中的配置文件，并把 Key 为 `EnableAutoConfiguration`  所对应的配置项通过反射实例化为配置类并加载到容器中。

`EnableAutoConfiguration` 项中包含了各式各样的配置项，这些配置项在 Spring Boot 启动过程中都能够通过 `SpringFactoriesLoader` 加载到运行时环境，从而实现自动化配置。

分析到这里，我们就可以得出一个完整的结论了：

当我们的SpringBoot项目启动的时候，会先导入AutoConfigurationImportSelector

这个类会帮我们选择所有候选的配置，我们需要导入的配置都是SpringBoot帮我们写好的一个一个的配置类，那么这些配置类的位置，存在与META-INF/spring.factories文件中

通过这个文件，Spring可以找到这些配置类的位置，于是去加载其中的配置。

![](/assets/images/posts/spring-auto-config/spring-auto-config-1.png)

# 2. @ConditionalOn 条件注解

Spring Boot 默认提供了 100 多个 AutoConfiguration 类，显然我们不可能会全部引入。所以在自动装配时，系统会去类路径下寻找是否有对应的配置类。如果有对应的配置类，则按条件进行判断，决定是否需要装配。

Spring Boot 中提供了一系列的条件注解，常见的包括：

- `@ConditionalOnProperty`：只有当所提供的属性属于 true 时才会实例化 Bean
- `@ConditionalOnBean`：只有在当前上下文中存在某个对象时才会实例化 Bean
- `@ConditionalOnClass`：只有当某个 Class 位于类路径上时才会实例化 Bean
- `@ConditionalOnExpression`：只有当表达式为 true 的时候才会实例化 Bean
- `@ConditionalOnMissingBean`：只有在当前上下文中不存在某个对象时才会实例化 Bean
- `@ConditionalOnMissingClass`：只有当某个 Class 在类路径上不存在的时候才会实例化 Bean
- `@ConditionalOnNotWebApplication`：只有当不是 Web 应用时才会实例化 Bean

 Spring Boot 还提供了一些不大常用的 @ConditionalOnXXX 注解，这些注解都定义在 org.springframework.boot.autoconfigure.condition 包中。

我们根据`@ConditionalOnClass` 注解详细了解下实现原理

```
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnClassCondition.class)
public @interface ConditionalOnClass {

	Class<?>[] value() default {};

	String[] name() default {};

}
```

`@ConditionalOnClass` 注解本身带有两个属性，一个 Class 类型的  value，一个 String 类型的 name，所以我们可以采用这两种方式中的任意一种来使用该注解。同时 ConditionalOnClass 注解本身还带了一个 `@Conditional(OnClassCondition.class)` 注解。所以，  ConditionalOnClass 注解的判断条件其实就包含在 OnClassCondition 这个类中。

OnClassCondition 是 SpringBootCondition 的子类，而 SpringBootCondition 又实现了Condition 接口。Condition 接口只有一个 matches 方法

```
@FunctionalInterface
public interface Condition {

	boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);

}
```

SpringBootCondition 中的 matches 方法实现如下：

```
@Override
public final boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
	String classOrMethodName = getClassOrMethodName(metadata);
	try {
		ConditionOutcome outcome = getMatchOutcome(context, metadata);
		logOutcome(classOrMethodName, outcome);
		recordEvaluation(context, classOrMethodName, outcome);
		return outcome.isMatch();
	}
	catch (NoClassDefFoundError ex) {
		throw new IllegalStateException("Could not evaluate condition on " + classOrMethodName + " due to "
				+ ex.getMessage() + " not found. Make sure your own configuration does not rely on "
				+ "that class. This can also happen if you are "
				+ "@ComponentScanning a springframework package (e.g. if you "
				+ "put a @ComponentScan in the default package by mistake)", ex);
	}
	catch (RuntimeException ex) {
		throw new IllegalStateException("Error processing condition on " + getName(metadata), ex);
	}
}
```

这里的 getClassOrMethodName 方法获取被添加了@ConditionalOnClass 注解的类或者方法的名称，而  getMatchOutcome 方法用于获取匹配的输出。我们看到 getMatchOutcome 方法实际上是一个抽象方法，需要交由  SpringBootCondition 的各个子类完成实现。

```
@Override
public ConditionOutcome getMatchOutcome(ConditionContext context, AnnotatedTypeMetadata metadata) {
   ClassLoader classLoader = context.getClassLoader();
   ConditionMessage matchMessage = ConditionMessage.empty();
   List<String> onClasses = getCandidates(metadata, ConditionalOnClass.class);
   if (onClasses != null) {
      List<String> missing = filter(onClasses, ClassNameFilter.MISSING, classLoader);
      if (!missing.isEmpty()) {
         return ConditionOutcome.noMatch(ConditionMessage.forCondition(ConditionalOnClass.class)
               .didNotFind("required class", "required classes").items(Style.QUOTE, missing));
      }
      matchMessage = matchMessage.andCondition(ConditionalOnClass.class)
            .found("required class", "required classes")
            .items(Style.QUOTE, filter(onClasses, ClassNameFilter.PRESENT, classLoader));
   }
   List<String> onMissingClasses = getCandidates(metadata, ConditionalOnMissingClass.class);
   if (onMissingClasses != null) {
      List<String> present = filter(onMissingClasses, ClassNameFilter.PRESENT, classLoader);
      if (!present.isEmpty()) {
         return ConditionOutcome.noMatch(ConditionMessage.forCondition(ConditionalOnMissingClass.class)
               .found("unwanted class", "unwanted classes").items(Style.QUOTE, present));
      }
      matchMessage = matchMessage.andCondition(ConditionalOnMissingClass.class)
            .didNotFind("unwanted class", "unwanted classes")
            .items(Style.QUOTE, filter(onMissingClasses, ClassNameFilter.MISSING, classLoader));
   }
   return ConditionOutcome.match(matchMessage);
}
```

首先通过 getCandidates 方法获取了 ConditionalOnClass 的 name 属性和 value 属性。然后通过  getMatches  方法将这些属性值进行比对，得到这些属性所指定的但在类加载器中不存在的类。如果发现类加载器中应该存在但事实上又不存在的类，则返回一个匹配失败的  Condition；反之，如果类加载器中存在对应类的话，则把匹配信息进行记录并返回一个 ConditionOutcome。

# 3. 自定义@Import

前面提过，Spring 容器针对 `@Import` 注解有以下四种处理方式：

- 如果该类实现了 `ImportSelector` 接口，Spring 容器就会实例化该类，并且调用其 selectImports 方法；
- 如果该类实现了 `DeferredImportSelector` 接口，则 Spring  容器也会实例化该类并调用其 selectImports方法。`DeferredImportSelector` 继承了  ImportSelector，区别在于 `DeferredImportSelector` 实例的 selectImports 方法调用时机晚于  `ImportSelector` 的实例，要等到 `@Configuration` 注解中相关的业务全部都处理完了才会调用；
- 如果该类实现了 `ImportBeanDefinitionRegistrar` 接口，Spring 容器就会实例化该类，并且调用其 registerBeanDefinitions 方法；
- 如果该类没有实现上述三种接口中的任何一个，Spring 容器就会直接实例化该类。

下面我们来逐一测试这几种种处理方式

## 3.1. ImportSelector

```
public class MyEnableAutoConfig implements ImportSelector {
  @Override
  public String[] selectImports(AnnotationMetadata importingClassMetadata) {
    return new String[]{"com.github.edgar615.spring.enable.Role",
            "com.github.edgar615.spring.enable.User"};
  }
}

@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(MyEnableAutoConfig.class)
public @interface EnableBean {
}
```

## 3.2. ImportBeanDefinitionRegistrar

```
public class CustomEnableAutoConfig implements ImportBeanDefinitionRegistrar {
  @Override
  public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
                                      BeanDefinitionRegistry registry) {
    registry.registerBeanDefinition("custom", createBeanDefinition(CustomBean.class));
  }

  private GenericBeanDefinition createBeanDefinition(Class<?> beanClass) {
    GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
    beanDefinition.setBeanClass(beanClass);
    beanDefinition.setAutowireMode(GenericBeanDefinition.AUTOWIRE_NO);
    return beanDefinition;
  }
}

@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(CustomEnableAutoConfig.class)
public @interface EnableCustomBean {
}
```

## 3.3. 未实现上述接口

```
public class SomeBeanConfiguration {

  @Bean
  public String aBean1() {
    return "aBean1";
  }

  @Bean
  @ConditionalOnProperty(name = "aBean2.enabled", matchIfMissing = true)
  public String aBean2() {
    return "aBean2";
  }
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Import(SomeBeanConfiguration.class)
@interface EnableSomeBeans {}
```

