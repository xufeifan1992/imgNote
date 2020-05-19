### SpringBoot启动流程分析原理



现在pom.xml中添加一个web工程依赖，方便后面观察容器

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```



```java
@SpringBootApplication
//扫描mabits通用mapper所在的包
@MapperScan(basePackages = "com.**.mapper")
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class,args);
    }

}
```



启动之后，F5进入run方法

```java
public static ConfigurableApplicationContext run(Class<?>[] primarySources,String[] args) {
	return new SpringApplication(primarySources).run(args);
}
```

到这里会执行`new SpringApplication(primarySources)`创建spring应用对象，继续F5往下跟会执行`SpringApplication`构造器





```java
// SpringApplication构造器
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    // 资源加载器
		this.resourceLoader = resourceLoader;
		Assert.notNull(primarySources, "PrimarySources must not be null");
		this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    // 1. 可能的web应用程序类型的类型。
		this.webApplicationType = WebApplicationType.deduceFromClasspath();
    // 2. 设置初始化应用context
		setInitializers((Collection) getSpringFactoriesInstances(
				ApplicationContextInitializer.class));
	// 3.设置初始化监听	
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
	// 4. 推演主程序类	
    this.mainApplicationClass = deduceMainApplicationClass();
}
```

```java
static WebApplicationType deduceFromClasspath() {
		if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null)
				&& !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
				&& !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
			return WebApplicationType.REACTIVE;
		}
		for (String className : SERVLET_INDICATOR_CLASSES) {
			if (!ClassUtils.isPresent(className, null)) {
				return WebApplicationType.NONE;
			}
		}
    // 这里是我们测试web容器
		return WebApplicationType.SERVLET;
	}
```

```java
public enum WebApplicationType {

	/**
	 * The application should not run as a web application and should not start an
	 * embedded web server.
	 */
	NONE, // 不是web应用

	/**
	 * The application should run as a servlet-based web application and should start an
	 * embedded servlet web server.
	 */
	SERVLET, // servlet容器

	/**
	 * The application should run as a reactive web application and should start an
	 * embedded reactive web server.
	 */
	REACTIVE;  // 反应型web应用（webflux）
```



在设置初始化应用context的时候 ，是先执行了`getSpringFactoriesInstances(ApplicationContextInitializer.class)方法，参数是ApplicationContextInitializer.class字节码对象



```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type,
		Class<?>[] parameterTypes, Object... args) {
		ClassLoader classLoader = getClassLoader();
		// Use names and ensure unique to protect against duplicates
		Set<String> names = new LinkedHashSet<>(
      // 加载ApplicationContextInitializer.class类型的类
       // 这里传入就是参数 ApplicationContextInitializer.clas
	 	SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    // 实例化加载到的类
		List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
				classLoader, args, names);
		AnnotationAwareOrderComparator.sort(instances);
	// 返回
    return instances;
}
```



```java
public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader) {
		String factoryClassName = factoryClass.getName();
		return loadSpringFactories(classLoader).getOrDefault(factoryClassName, Collections.emptyList());
}
```



```java
private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
    // 先从缓存中拿
		MultiValueMap<String, String> result = cache.get(classLoader);
		if (result != null) {
			return result;
		}
		try {
// 去资源路径下加载
public static final String ACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories"; 
public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories"; 
	Enumeration<URL> urls = (classLoader != null ?	
 classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :     ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION);			result = new LinkedMultiValueMap<>();
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
             // 返回所有的加载的类
			return result;
		}
		catch (IOException ex) {
			throw new IllegalArgumentException("Unable to load factories from location [" +
					FACTORIES_RESOURCE_LOCATION + "]", ex);
		}
}
```

里有两个加载配置类的地方其实都指向了`META-INF/spring.factories`，通过断点我们可以看到应用程序是加载了以下几个jar下的`spring.factores`文件。

   

​    

从Map中根据据`org.springframework.context.ApplicationContextInitializer`的类型拿到需要的类初始化类，断点进入`getOrDefault(factoryClassName, Collections.emptyList());`方法





```java
public void setInitializers(
			Collection<? extends ApplicationContextInitializer<?>> initializers) {
		this.initializers = new ArrayList<>();
		this.initializers.addAll(initializers);
	}
```

