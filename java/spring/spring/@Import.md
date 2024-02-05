# 总结
通过Import可以实现多个配置类的一次性导入，SpringBoot的自动装配也是基于此。
# 源码分析
## ConfigurationClassParser
### 解析import
在spring解析@Configuration配置类的时候会通过ConfigurationClassParser进行解析
org.springframework.context.annotation.ConfigurationClassParser#doProcessConfigurationClass
在doProcessConfigurationClass里会执行`processImports`来解析@Import
```java
	protected final SourceClass doProcessConfigurationClass(
			ConfigurationClass configClass, SourceClass sourceClass, Predicate<String> filter)
			throws IOException {
    //....
		// Process any @Import annotations
		processImports(configClass, sourceClass, getImports(sourceClass), filter, true);
    // ...
		return null;
	}
```
在processImports里主要分为四类处理：
1. 只实现了ImportSelector: 直接执行对应的selectImports解析出需要导入的类然后递归执行processImports
2. 同时实现了ImportSelector + 继承了DeferredImportSelector： 将对应的selector放入deferredImportSelectorHandler后续执行
3. 实现了ImportBeanDefinitionRegistrar：将对应的selector放入 importBeanDefinitionRegistrars 后续执行
4. 都没有实现：直接视为标注了@Configuration的类调用processConfigurationClass进行解析执行
```java
private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
    Collection<SourceClass> importCandidates, Predicate<String> exclusionFilter,
    boolean checkForCircularImports) {

  if (importCandidates.isEmpty()) {
    return;
  }

  if (checkForCircularImports && isChainedImportOnStack(configClass)) {
    this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
  }
  else {
    this.importStack.push(configClass);
    try {
      for (SourceClass candidate : importCandidates) {
        
        if (candidate.isAssignable(ImportSelector.class)) {
          // Candidate class is an ImportSelector -> delegate to it to determine imports
          Class<?> candidateClass = candidate.loadClass();
          ImportSelector selector = ParserStrategyUtils.instantiateClass(candidateClass, ImportSelector.class,
              this.environment, this.resourceLoader, this.registry);
          Predicate<String> selectorFilter = selector.getExclusionFilter();
          if (selectorFilter != null) {
            exclusionFilter = exclusionFilter.or(selectorFilter);
          }
          // 同时实现了ImportSelector + 继承了DeferredImportSelector
          if (selector instanceof DeferredImportSelector deferredImportSelector) {
            this.deferredImportSelectorHandler.handle(configClass, deferredImportSelector);
          }
          else {
            // 只实现了ImportSelector
            String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
            Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames, exclusionFilter);
            processImports(configClass, currentSourceClass, importSourceClasses, exclusionFilter, false);
          }
        }
        else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) { // 实现了ImportBeanDefinitionRegistrar
          // Candidate class is an ImportBeanDefinitionRegistrar ->
          // delegate to it to register additional bean definitions
          Class<?> candidateClass = candidate.loadClass();
          ImportBeanDefinitionRegistrar registrar =
              ParserStrategyUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class,
                  this.environment, this.resourceLoader, this.registry);
          configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
        }
        else {
          // 都没有实现视为 @Configuration
          // Candidate class not an ImportSelector or ImportBeanDefinitionRegistrar ->
          // process it as an @Configuration class
          this.importStack.registerImport(
              currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
          processConfigurationClass(candidate.asConfigClass(configClass), exclusionFilter);
        }
      }
    }
    //...
  }
}
```
### 解析deferredImportSelectorHandler
在parse执行`parse(bd.getBeanClassName(), holder.getBeanName());`解析完@Configuration标志的方法后就执行`this.deferredImportSelectorHandler.process();`deferredImportSelectors
org.springframework.context.annotation.ConfigurationClassParser#parse(java.util.Set<org.springframework.beans.factory.config.BeanDefinitionHolder>)
```java
	public void parse(Set<BeanDefinitionHolder> configCandidates) {
		for (BeanDefinitionHolder holder : configCandidates) {
			BeanDefinition bd = holder.getBeanDefinition();
			try {
				if (bd instanceof AnnotatedBeanDefinition annotatedBeanDef) {
					parse(annotatedBeanDef.getMetadata(), holder.getBeanName());
				}
				else if (bd instanceof AbstractBeanDefinition abstractBeanDef && abstractBeanDef.hasBeanClass()) {
					parse(abstractBeanDef.getBeanClass(), holder.getBeanName());
				}
				else {
					parse(bd.getBeanClassName(), holder.getBeanName());
				}
			}
			catch (BeanDefinitionStoreException ex) {
				throw ex;
			}
			catch (Throwable ex) {
				throw new BeanDefinitionStoreException(
						"Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);
			}
		}

		this.deferredImportSelectorHandler.process();
	}

```
在process方法里会执行`handler.processGroupImports();`进行处理处理
org.springframework.context.annotation.ConfigurationClassParser.DeferredImportSelectorHandler#process
```java
public void process() {
  List<DeferredImportSelectorHolder> deferredImports = this.deferredImportSelectors;
  this.deferredImportSelectors = null;
  try {
    if (deferredImports != null) {
      DeferredImportSelectorGroupingHandler handler = new DeferredImportSelectorGroupingHandler();
      deferredImports.sort(DEFERRED_IMPORT_COMPARATOR);
      deferredImports.forEach(handler::register);
      handler.processGroupImports();
    }
  }
  finally {
    this.deferredImportSelectors = new ArrayList<>();
  }
}
```
在processGroupImports里会执行`grouping.getImports()`执行对应实现的解析类方，然后执行`processImports`进行处理
org.springframework.context.annotation.ConfigurationClassParser.DeferredImportSelectorGroupingHandler#processGroupImports
```java
public void processGroupImports() {
  for (DeferredImportSelectorGrouping grouping : this.groupings.values()) {
    Predicate<String> exclusionFilter = grouping.getCandidateFilter();
    grouping.getImports().forEach(entry -> {
      ConfigurationClass configurationClass = this.configurationClasses.get(entry.getMetadata());
      try {
        processImports(configurationClass, asSourceClass(configurationClass, exclusionFilter),
            Collections.singleton(asSourceClass(entry.getImportClassName(), exclusionFilter)),
            exclusionFilter, false);
      }
      catch (BeanDefinitionStoreException ex) {
        throw ex;
      }
      catch (Throwable ex) {
        throw new BeanDefinitionStoreException(
            "Failed to process import candidates for configuration class [" +
                configurationClass.getMetadata().getClassName() + "]", ex);
      }
    });
  }
}
```
在getImports会调用`this.group.process`执行对应实现的`selectImports`
org.springframework.context.annotation.ConfigurationClassParser.DeferredImportSelectorGrouping#getImports
```java
public Iterable<Group.Entry> getImports() {
  for (DeferredImportSelectorHolder deferredImport : this.deferredImports) {
    this.group.process(deferredImport.getConfigurationClass().getMetadata(),
        deferredImport.getImportSelector());
  }
  return this.group.selectImports();
}
```
在process方法会执行对应实现的selectImports方法获取需要注册到容器里的bean
org.springframework.context.annotation.ConfigurationClassParser.DefaultDeferredImportSelectorGroup#process
```java
private static class DefaultDeferredImportSelectorGroup implements Group {

  private final List<Entry> imports = new ArrayList<>();

  @Override
  public void process(AnnotationMetadata metadata, DeferredImportSelector selector) {
    for (String importClassName : selector.selectImports(metadata)) {
      this.imports.add(new Entry(metadata, importClassName));
    }
  }

  @Override
  public Iterable<Entry> selectImports() {
    return this.imports;
  }
}

```

### 解析importBeanDefinitionRegistrars

importBeanDefinitionRegistrars的处理是最晚的。在@Import、ImportSelect之后。
查看processConfigBeanDefinitions解析处理逻辑
1. 先创建ConfigurationClassParser
2. 执行parse方法解析@Configuration类，此时会先注册：只实现了ImportSelector或者都没有实现的
3. 然后在parse里会执行解析deferredImportSelectorHandler
4. 最后执行processConfigBeanDefinitions里的`this.reader.loadBeanDefinitions(configClasses);`解析importBeanDefinitionRegistrars
org.springframework.context.annotation.ConfigurationClassPostProcessor#processConfigBeanDefinitions

```java
    //...
		ConfigurationClassParser parser = new ConfigurationClassParser(
				this.metadataReaderFactory, this.problemReporter, this.environment,
				this.resourceLoader, this.componentScanBeanNameGenerator, registry);

		Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
		Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
		do {
			StartupStep processConfig = this.applicationStartup.start("spring.context.config-classes.parse");
			parser.parse(candidates);
			parser.validate();

			Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
			configClasses.removeAll(alreadyParsed);

			// Read the model and create bean definitions based on its content
			if (this.reader == null) {
				this.reader = new ConfigurationClassBeanDefinitionReader(
						registry, this.sourceExtractor, this.resourceLoader, this.environment,
						this.importBeanNameGenerator, parser.getImportRegistry());
			}
			this.reader.loadBeanDefinitions(configClasses);
  //...
```
执行loadBeanDefinitions中的`loadBeanDefinitionsForConfigurationClass`加载importBeanDefinitionRegistrars里的待处理的类
```java
public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {
  TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();
  for (ConfigurationClass configClass : configurationModel) {
    loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
  }
}
```
loadBeanDefinitionsForConfigurationClass里会执行`loadBeanDefinitionsFromRegistrars`加载importBeanDefinitionRegistrars里的待处理的类
org.springframework.context.annotation.ConfigurationClassBeanDefinitionReader#loadBeanDefinitionsForConfigurationClass
```java
private void loadBeanDefinitionsForConfigurationClass(
    ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {

  if (trackedConditionEvaluator.shouldSkip(configClass)) {
    String beanName = configClass.getBeanName();
    if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
      this.registry.removeBeanDefinition(beanName);
    }
    this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());
    return;
  }

  if (configClass.isImported()) {
    registerBeanDefinitionForImportedConfigurationClass(configClass);
  }
  for (BeanMethod beanMethod : configClass.getBeanMethods()) {
    loadBeanDefinitionsForBeanMethod(beanMethod);
  }

  loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
  // 这里会加载，通过getImportBeanDefinitionRegistrars() 获取之前放入的
  loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
}
```
在loadBeanDefinitionsFromRegistrars里会挨个执行对应实现的ImportBeanDefinitionRegistrar进行注册bean
```java
private void loadBeanDefinitionsFromRegistrars(Map<ImportBeanDefinitionRegistrar, AnnotationMetadata> registrars) {
  registrars.forEach((registrar, metadata) ->
      registrar.registerBeanDefinitions(metadata, this.registry, this.importBeanNameGenerator));
}

```
