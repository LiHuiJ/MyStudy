**springboot学习**

# 自动装配

## 相关注解

### @Configuration与@Bean

@Configuration： 配置于类，该配置类也是组件，告诉springboot该类为配置类
@Bean：给容器添加组件，以方法名作为组件id，返回类型即组件类型，返回值即为组件在	  容器中的实例。亦可@bean(“bean-name”),此时bean-name为组件id。

获取容器中组件：

```java
ConfigurableApplicationContext context = SpringApplication.run(启动类.class, args);
Bean tom = context.getBean("bean-name", Bean-class.class);
```

注：

以上方法获取到的bean是单实例的

### **@Import**

@Import({A.class, B.class}): 在容器中自动创建出A、B两个类型的组件，默认组件名称为A、B的全类名

### **@Conditional**

按条件装配，满足条件则装配，其包含许多派生注解，比如：@ConditionalOnBean,@ConditionalOnMissingBean,@ConditionalOnClass等等。

## **自动配置入门**

核心注解@SpringApplication,其相当于@SpringBootConfiguration，@EnableAutoConfiguration与@ComponentScan的组合。

```java
//@SpringBootConfiguration 代表当前类为一个配置类
//@ComponentScan 指定扫描哪些包
```

核心注解为**@EnableAutoConfiguration**

```java
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
	//....
}
```

### @AutoConfigurationPackage

```java
@Import(AutoConfigurationPackages.Registrar.class)//给容器中导入组件
public @interface AutoConfigurationPackage {
}
//利用Registrar给容器中导入一系列组件，Registrar如下：
```

#### AutoConfigurationPackages.Registrar.class

```java
static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {

    	/**
         * 将指定的一个包下面的所有组件导入进来
         * 即：主启动类所在包下的所有组件
         */
		@Override
		public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
			register(registry, new PackageImport(metadata).getPackageName());
		}

		@Override
		public Set<Object> determineImports(AnnotationMetadata metadata) {
			return Collections.singleton(new PackageImport(metadata));
		}

	}
```

### @Import(AutoConfigurationImportSelector.class)

#### AutoConfigurationImportSelector.class

核心为selectImports方法中调用的getAutoConfigurationEntry

```java
	@Override
	public String[] selectImports(AnnotationMetadata annotationMetadata) {
		if (!isEnabled(annotationMetadata)) {
			return NO_IMPORTS;
		}
		AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
				.loadMetadata(this.beanClassLoader);
		AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(autoConfigurationMetadata,
				annotationMetadata);
		return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
	}

	protected AutoConfigurationEntry getAutoConfigurationEntry(AutoConfigurationMetadata autoConfigurationMetadata,
			AnnotationMetadata annotationMetadata) {
		if (!isEnabled(annotationMetadata)) {
			return EMPTY_ENTRY;
		}
		AnnotationAttributes attributes = getAttributes(annotationMetadata);
        //获取所有候选的配置，configurations为127个默认导入容器中的配置类
		List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
		configurations = removeDuplicates(configurations);
		Set<String> exclusions = getExclusions(annotationMetadata, attributes);
		checkExcludedClasses(configurations, exclusions);
		configurations.removeAll(exclusions);
		configurations = filter(configurations, autoConfigurationMetadata);
		fireAutoConfigurationImportEvents(configurations, exclusions);
		return new AutoConfigurationEntry(configurations, exclusions);
	}

	
	protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
        //利用工厂加载
		List<String> configurations = SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(),
				getBeanClassLoader());
		Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you "
				+ "are using a custom packaging, make sure that file is correct.");
		return configurations;
	}
```

利用工厂加载Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader)得到所有组件

```java
public final class SpringFactoriesLoader {
    
    public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";
    
    //.....
    public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader) {
		String factoryClassName = factoryClass.getName();
		return loadSpringFactories(classLoader).getOrDefault(factoryClassName, Collections.emptyList());
	}
    
    
    private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
		MultiValueMap<String, String> result = cache.get(classLoader);
		if (result != null) {
			return result;
		}

		try {
            //从FACTORIES_RESOURCE_LOCATION（即：“META-INF/spring.factories”）获取资源文件
            //即默认扫描所有META-INF/spring.factories位置的文件
			Enumeration<URL> urls = (classLoader != null ?
					classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
					ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
			result = new LinkedMultiValueMap<>();
			while (urls.hasMoreElements()) {
				URL url = urls.nextElement();
				UrlResource resource = new UrlResource(url);
				Properties properties = PropertiesLoaderUtils.loadProperties(resource);
				for (Map.Entry<?, ?> entry : properties.entrySet()) {
					String factoryClassName = ((String) entry.getKey()).trim();
					for (String factoryName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
						result.add(factoryClassName, factoryName.trim());
					}
				}
			}
			cache.put(classLoader, result);
			return result;
		}
		catch (IOException ex) {
			throw new IllegalArgumentException("Unable to load factories from location [" +
					FACTORIES_RESOURCE_LOCATION + "]", ex);
		}
	}
    
    //.....
}
```

springboot包下的META-INF/spring.factories

![](E:\my-projects\MyStudy\images\springboot-release-fac.jpg)

spring-boot-autoconfigure包下的META-INF/spring.factories

![](E:\my-projects\MyStudy\images\springboot-autoconfig-fac.jpg)

从spring-boot-autoconfigure包下的META-INF/spring.factories可以看到127个自动配置项是在该文件中写死的，但是这么多自动配置类不是全部加载进去的，是按需加载的。

## 按需加载

根据@Conditional及其派生注解按需加载

比如批处理的包：

```java
@Configuration
//当依赖中存在JobLauncher.class, DataSource.class且存在JobLauncher.class类的Bean时才自动配置
@ConditionalOnClass({ JobLauncher.class, DataSource.class })
@AutoConfigureAfter(HibernateJpaAutoConfiguration.class)
@ConditionalOnBean(JobLauncher.class)
@EnableConfigurationProperties(BatchProperties.class)
@Import(BatchConfigurerConfiguration.class)
public class BatchAutoConfiguration {
	//......
}
```

servlet包：

```java
/**
*给容器中加入文件上传解析器
*/
@Bean
//容器中有MultipartResolver类型的组件
@ConditionalOnBean(MultipartResolver.class)
////容器中没有名称为DispatcherServlet.MULTIPART_RESOLVER_BEAN_NAME（multipartResolver）的组件
@ConditionalOnMissingBean(name = DispatcherServlet.MULTIPART_RESOLVER_BEAN_NAME)
public MultipartResolver multipartResolver(MultipartResolver resolver) {
    // Detect if the user has created a MultipartResolver but named it incorrectly
    return resolver;
}
```













