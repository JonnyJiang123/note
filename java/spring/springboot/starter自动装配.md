# 总结
SpringBoot自定义导入本质都是配置类上添加的@Import的变种
# 常用方式
![image](https://github.com/JonnyJiang123/note/assets/56102991/768ca141-934a-4544-987e-f9169ce760fc)
1. resources/META-INF/spring.factories 在SpringBoot低版本支持，高版本不支持（比如说3.2.1）
2. resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports 所有版本都支持
## spring.factories

使用spring.factories需要使用org.springframework.boot.autoconfigure.EnableAutoConfiguration作为key，value为需要自动注入到容器里的bean，如果有多个通过“,”分割
比如
```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.alibaba.demo.domain.order.Order
```
## org.springframework.boot.autoconfigure.AutoConfiguration.imports
使用org.springframework.boot.autoconfigure.AutoConfiguration.imports直接在文件内容里写明需要自动注入的bean即可，应该bean一行
比如：
```plaintext
com.alibaba.demo.domain.order.Order
```
# 源码解读
核心在于<b>org.springframework.boot.autoconfigure.SpringBootApplication</b>这个超级注解
## SpringBootApplication
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
}
```
在SpringBootApplication注解上标注了许多注解，其中跟自动注入相关的为EnableAutoConfiguration注解
## EnableAutoConfiguration
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
}
```
在EnableAutoConfiguration标注了许多注解，其中最核心的注解就是`@Import(AutoConfigurationImportSelector.class)`。在执行ConfigurationClassParser的时候会解析<b>@Import</b>注解，
同时解析AutoConfigurationImportSelector执行对应的import方法来进行导入
## AutoConfigurationImportSelector
```java
public class AutoConfigurationImportSelector implements DeferredImportSelector, BeanClassLoaderAware,
		ResourceLoaderAware, BeanFactoryAware, EnvironmentAware, Ordered {
}
```
可以看到AutoConfigurationImportSelector实现了DeferredImportSelector
```java
public interface DeferredImportSelector extends ImportSelector {
}
```
查看DeferredImportSelector可以知道实现了ImportSelector这个最原始的spring提供自动注入bean的接口
回到AutoConfigurationImportSelector实现的<b>selectImports</b>方法
```java
	public String[] selectImports(AnnotationMetadata annotationMetadata) {
		if (!isEnabled(annotationMetadata)) {
			return NO_IMPORTS;
		}
    // 执行getAutoConfigurationEntry方法来实现装配需要自动注入的bean
		AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(annotationMetadata);
		return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
	}
```
在selectImports中核心时通过getAutoConfigurationEntry来实现自动装配的
```java
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
在getAutoConfigurationEntry中核心是通过<b>getCandidateConfigurations</b>来获取需要自动装配的bean的

## getCandidateConfigurations
getCandidateConfigurations在不同的版本实现有点不一样。本质都是通过spi技术去获取需要自动装配的bean
### 较低版本
```java
	protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
		List<String> configurations = new ArrayList<>(
				SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader())); // 解析spring.factories里需要装配的bean
		ImportCandidates.load(AutoConfiguration.class, getBeanClassLoader()).forEach(configurations::add);  // 解析org.springframework.boot.autoconfigure.AutoConfiguration.imports需要装配的bean
		Assert.notEmpty(configurations,
				"No auto configuration classes found in META-INF/spring.factories nor in META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports. If you "
						+ "are using a custom packaging, make sure that file is correct.");
		return configurations;
	}
```
在较低版本里会先去<b>spring.factories</b>获取，再去<b>org.springframework.boot.autoconfigure.AutoConfiguration.imports</b>获取
### 较高版本(以3.2.1为例)
```java
	protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
		List<String> configurations = ImportCandidates.load(AutoConfiguration.class, getBeanClassLoader()) // 解析org.springframework.boot.autoconfigure.AutoConfiguration.imports需要装配的bean
			.getCandidates();
		Assert.notEmpty(configurations,
				"No auto configuration classes found in "
						+ "META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports. If you "
						+ "are using a custom packaging, make sure that file is correct.");
		return configurations;
	}
```
在较高版本里只会去<b>org.springframework.boot.autoconfigure.AutoConfiguration.imports</b>获取
