
#SpringBoot项目构建

###在项目中使用SpringBoot

 Spring官方发布的每一个SpringBoot版本下都会提供一个版本依赖目录，在实际项目中引入即可使用Spring官方做好的版本在maven中引入spring-boot-starter-parent作为顶级项目，spring-boot-starter-parent中为springboot项目规定一些项目构建的变量，以2.0.3.RELEASE版本为例，spring-boot-starter-parent-2.0.3.RELEASE.pom 中定义变量：
 
``` xml
 <properties>
	 <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
	 <java.version>1.8</java.version>
	 <resource.delimiter>@</resource.delimiter>
	 <maven.compiler.source>${java.version}</maven.compiler.source>
	 <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
	 <maven.compiler.target>${java.version}</maven.compiler.target>
 </properties>
```
和应用配置文件的定义：
```xml
<resource>
	<directory>${basedir}/src/main/resources</directory>
	<excludes>
		<exclude>**/application*.yml</exclude>
		<exclude>**/application*.yaml</exclude>
		<exclude>**/application*.properties</exclude>
	</excludes>
</resource>
```

spring-boot-starter-parent具体的依赖声明在其父级项目spring-boot-dependencies中，pom中声明了绝大部分的官方配套组件，包括 日志、数据持久化、监控、web服务器等。在项目当中只需要把项目的parent设置为 spring-boot-starter-parent  同时引入依赖不需要指定框架的版本即可使用。如果starter-parent中的依赖不能满足项目需求，可以单独指定所需组件的版本，通过properties标签指定。加入所有项目依赖之后，通过Spring官方提供的maven插件来打包成一个可执行的jar包。  

```xml
<build>
	<plugins>
		<plugin>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-maven-plugin</artifactId>
		</plugin>
	</plugins>
</build>
```

###Springboot启动创建  
Spring-boot通常是以SpringApplication.run()的方式去启动一个Spring-boot的项目。
`run()` 方法中新建了一个 `SpringApplication` 对象。

SpringApplication是Spring-boot的程序主控对象也可以理解为是对整个Spring-boot工程的抽象，大多数情况下都可以通过静态的run()方法来启动一个Spring-boot项目。

``` java
public static ConfigurableApplicationContext run(Class<?>[] primarySources,String[] args) 
{   
	return new SpringApplication(primarySources).run(args);
}
```

SpringApplication对象实例化的过程代码如下：
```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {   
	this.resourceLoader = resourceLoader;   
	Assert.notNull(primarySources, "PrimarySources must not be null");   
	this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));   
	this.webApplicationType = deduceWebApplicationType();   
	setInitializers((Collection) getSpringFactoriesInstances(         ApplicationContextInitializer.class));   
	setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));   
	this.mainApplicationClass = deduceMainApplicationClass();}
```

创建SpringApplication过程中主要完成了以下工作：

	 1. web类型检测（webApplicationType）
>this.webApplicationType = deduceWebApplicationType();

之后会根据检测到的webApplicationType来创建对应不同的Spring应用上下文（ApplicationContext）	

	2. 初始化协助对象创建setInitializers（Initializer）
>setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));

这些对象在boot程序初始化时候会被委托相应的任务。

	3. 设置事件监听器
>setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));

用于监听Application生命周期。

####web类型检测

``` java
private WebApplicationType deduceWebApplicationType() {   
	if (ClassUtils.isPresent(REACTIVE_WEB_ENVIRONMENT_CLASS, null)         && !ClassUtils.isPresent(MVC_WEB_ENVIRONMENT_CLASS, null)) {      
		return WebApplicationType.REACTIVE;   
	}   
	for (String className : WEB_ENVIRONMENT_CLASSES) {      
		if (!ClassUtils.isPresent(className, null)) {         
			return WebApplicationType.NONE;      
		}  
	}   
	return WebApplicationType.SERVLET;
}
```
在检测项目类型的`deduceWebApplicationType()`方法中。    如果项目类路径下存在 `org.springframework.web.reactive.DispatcherHandler` 且不存在 `org.springframework.web.servlet.DispatcherServlet` 则使用 REACTIVE 环境（reactive web 环境）。    

如果不存在 javax.servlet.Servlet 或者 `org.springframework.web.context.ConfigurableWebApplicationContext` 类则认为是一个NONE环境(非web环境，将不启动内置的web容器)。

默认情况下则是一个 SERVLET 的环境 (标准的web环境，基于JDK servlet)


####初始化协助对象创建setInitializers((Collection) 
>getSpringFactoriesInstances(ApplicationContextInitializer.class));

在创建这些协助对象的时候会通过 `getSpringFactoriesInstances()` 方法获取boot项目的初始化协助对象。
``` java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type,      Class<?>[] parameterTypes, Object... args) {   
	ClassLoader classLoader = Thread.currentThread().getContextClassLoader();   
	// Use names and ensure unique to protect against duplicates   Set<String> names = new LinkedHashSet<>(         SpringFactoriesLoader.loadFactoryNames(type, classLoader));   
	List<T> instances = createSpringFactoriesInstances(type, parameterTypes,classLoader, args, names);   
	AnnotationAwareOrderComparator.sort(instances);   
	return instances;
}
```

在 getSpringFactoriesInstances() 当中，从源码看出Spring-boot是通过类加载器去加载并创建一些Spring-boot初始化协助对象，最终的加载过程是委托给 SpringFactoriesLoader 来完成，实际的加载过程是在 `loadSpringFactories()`方法中。
```java
private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {   
	MultiValueMap<String, String> result = cache.get(classLoader);   
	if (result != null) {      
		return result;   
	}   
	try {      
		Enumeration<URL> urls = (classLoader != null ?classLoader.getResources(FACTORIES_RESOURCE_LOCATION) : ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));      
		result = new LinkedMultiValueMap<>();      
		while (urls.hasMoreElements()) {         
			URL url = urls.nextElement();         
			UrlResource resource = new UrlResource(url);         
			Properties properties = PropertiesLoaderUtils.loadProperties(resource);         
			for (Map.Entry<?, ?> entry : properties.entrySet()) {
				List<String> factoryClassNames = Arrays.asList(                  StringUtils.commaDelimitedListToStringArray((String) entry.getValue()));            
				result.addAll((String) entry.getKey(), factoryClassNames);         
			}      
		}      
		cache.put(classLoader, result);      
		return result;   
	}catch (IOException ex) {      
		throw new IllegalArgumentException("Unable to load factories from location [" +FACTORIES_RESOURCE_LOCATION + "]", ex);   
		}
	}
```
>其中 `FACTORIES_RESOURCE_LOCATION` 的值就是 `META-INF/spring.factories`。

META-INF这个目录下的spring.factories是Spring-boot项目必须要包含的文件，因为许多默认的配置文件都是从这个文件中读取的，这个文件也是Spring-boot项目也是实现自动配置的基础，是Spring-boot项目的一个很重要的文件。

在Spring-boot包中的META-INF/spring.factories 定义了以下配置信息


><font face="黑体" color="#A72550">#PropertySource Loaders  (属性配置加载器)</font>
org.springframework.boot.env.PropertySourceLoader
<font face="黑体" color="#A72550"># Run Listeners   (运行监听者)</font>
org.springframework.boot.SpringApplicationRunListener
<font face="黑体" color="#A72550"># Error Reporters     (错误回调器)</font>
org.springframework.boot.SpringBootExceptionReporter
<font face="黑体" color="#A72550"># Application Context Initializers     (初始化协助对象)</font>
org.springframework.context.ApplicationContextInitializer
<font face="黑体" color="#A72550"># Application Listeners   (应用监听对象)</font>
org.springframework.context.ApplicationListener
<font face="黑体" color="#A72550"># Environment Post Processors      (环境加工对象)</font>
org.springframework.boot.env.EnvironmentPostProcessor
<font face="黑体" color="#A72550"># Failure Analyzer</font>
sorg.springframework.boot.diagnostics.FailureAnalyzer


这些在Spring-boot工程中定义了一些boot工程相关的组件，而在boot启动过程中则根据spring.factories中获取相应的初始化协助对象 `ApplicationContextInitializer` ，默认使用的对象有:
<table><tr><td bgcolor="F6F6F6"> Application Context Initializersorg.springframework.context.ApplicationContextInitializer=
\org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer,
\org.springframework.boot.context.ContextIdApplicationContextInitializer,
\org.springframework.boot.context.config.DelegatingApplicationContextInitializer,
\org.springframework.boot.web.context.ServerPortInfoApplicationContextInitializer</td></tr></table>
+ ConfigurationWarningsApplicationContextInitializer
+ ContextIdApplicationContextInitializer
+ DelegatingApplicationContextInitializer
+ ServerPortInfoApplicationContextInitializer


这些ApplicationContextInitializer中`DelegatingApplicationContextInitializer`是留给对Srping扩展的一个预留接口。通过`context.initializer.classes`属性可以自定义项目中需要用到的 ApplicationContextInitializer ，DelegatingApplicationContextInitializer 会加载定义的 ApplicationContextInitializer 来执行一定的逻辑，有兴趣可以自行了解其过程。

#### 设置事件监听器setListeners((Collection) 
同样的，在设置事件监听器时候是获取spring.factories中配置的 ApplicationListener 对象
	>1. ClearCachesApplicationListener
	>2. ParentContextCloserApplicationListener
	>3. FileEncodingApplicationListener
	>4. AnsiOutputApplicationListener
	>5. ConfigFileApplicationListener
	>6. DelegatingApplicationListener
	>7. ClasspathLoggingApplicationListener
	>8. LoggingApplicationListener
	>9. LiquibaseServiceLocatorApplicationListener

这些监听器在整个boot工程启动的过程中都会参与到其中。现在我们还是看回 SpringApplication 的 run 方法中，boot工程所有的启动流程都包含在这里。

####run()方法启动工程
在创建完SpringApplicsation之后，亦即对boot工程有了一个抽象的表达对象，会调用SpringApplicsation的`run()`方法来启动整个Spring-boot工程。
```java
/** Run the Spring application, creating and refreshing a new
 * {@link ApplicationContext}.
 * @param args the application arguments (usually passed from a Java main method)
 * @return a running {@link ApplicationContext}
 */
public ConfigurableApplicationContext run(String... args) {   
	StopWatch stopWatch = new StopWatch();   
	stopWatch.start();   
	ConfigurableApplicationContext context = null;   
	Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();   
	configureHeadlessProperty();   
	SpringApplicationRunListeners listeners = getRunListeners(args);   listeners.starting();   
	try {      
		ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);      
		ConfigurableEnvironment environment = prepareEnvironment(listeners,            applicationArguments);      
		configureIgnoreBeanInfo(environment);      
		Banner printedBanner = printBanner(environment);      
		context = createApplicationContext();      
		exceptionReporters = getSpringFactoriesInstances(            SpringBootExceptionReporter.class,new Class[] { ConfigurableApplicationContext.class }, context);      
		prepareContext(context, environment, listeners, applicationArguments,printedBanner);      
		refreshContext(context);      
		afterRefresh(context, applicationArguments);      
		stopWatch.stop();      
		if (this.logStartupInfo) {         
			new StartupInfoLogger(this.mainApplicationClass)               .logStarted(getApplicationLog(), stopWatch);      
		}      
		listeners.started(context);      
		callRunners(context, applicationArguments);   
	}catch (Throwable ex) {      
		handleRunFailure(context, ex, exceptionReporters, listeners);      
		throw new IllegalStateException(ex);   
	}
	try {      
		listeners.running(context);   
	}catch (Throwable ex) {     
		handleRunFailure(context, ex, exceptionReporters, null);      
		throw new IllegalStateException(ex);   
	}   
	return context;
}
```
Spring-boot工程的启动流程，大致可以归纳为4个步骤主要流程如下：
	1. 创建 SpringApplicationRunListeners。 这个监听器是专门用在run方法中监听整个过程，run方法是启动boot工程的主体方法，所以也可以认为SpringApplicationRunListeners就是专门用来监听boot启动事件的监听器。在实际过程中，SpringApplicationRunListeners 则会承担部分的boot工程启动任务。
	2. 准备环境。这是一个很重要的步骤，准备环境的过程包括准备一些工程运行的环境信息，和通知ApplicationListener 监听器，亦即在spring.factories中配置的，实例化 SpringApplication 时候注入的那些监听器。
	3. 创建刷新应用上下文(ApplicationContext)。完成所有bean的配装，这下回到了Spring核心内容。
	4. 完成boot工程启动，广播启动完成的事件。

####初始化SpringApplicationRunListeners

`SpringApplicationRunListeners` 是`SpringApplicationRunListener`的集合对象(加了一个s)，先来看看SpringApplicationRunListener是一个什么样的监听器。

``` text
/*** Listener for the {@link SpringApplication} {@code run} method.
 * {@link SpringApplicationRunListener}s are loaded via the {@link SpringFactoriesLoader}
 * and should declare a public constructor that accepts a {@link SpringApplication}
 * instance and a {@code String[]} of arguments. A new
 * {@link SpringApplicationRunListener} instance will be created for each run.
 * @author Phillip Webb* @author Dave Syer
 * @author Andy Wilkinson
 */
```
在 SpringApplicationRunListener 接口顶部有简单的描述 ，大意是讲 SpringApplicationRunListener  主要用于监听 SpringApplication 的 run() 方法，从名字也可以大概看出来\，并且规定了这些 `SpringApplicationRunListener` 都应该有一个以 `SpringApplication` 和 `String[]` 作为入参的构造方法。这些 SpringApplicationRunListener 是在 boot 工程正式启动前就需要加载的，过程如下：
``` java
SpringApplicationRunListeners listeners = getRunListeners(args);
listeners.starting();
```
初始化 SpringApplicationRunListeners 只有两样工作

	1. 创建Listener
	2. 传播 boot 工程开始启动事件

创建的过程非常简单，是那个熟悉的 META-INF/spring.factories 中SpringApplicationRunListener 的配置好的类。
```java
private SpringApplicationRunListeners getRunListeners(String[] args) {  
	Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };   
	return new SpringApplicationRunListeners(logger, getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args));
}
```
在Spring-boot工程的 META-INF/spring.factories 中定义的 SpringApplicationRunListener 配置只有一个EventPublishingRunListener：
> RunListenersorg.springframework.boot.SpringApplicationRunListener=
\org.springframework.boot.context.event.EventPublishingRunListener

EventPublishingRunListener 的实例化过程会注入 SpringApplication 的引用，并且会将 SpringApplication 中的 `ApplicationListener` ，即是在SpringApplication实例化的时候根据spring.factories中配置 ApplicationListener ，传入到 EventPublishingRunListener 的广播对象中。即 `ApplicationListener`实际上不仅仅在`SpringApplication`
```java
public EventPublishingRunListener(SpringApplication application, String[] args) {   
	this.application = application;   
	this.args = args;   
	this.initialMulticaster = new SimpleApplicationEventMulticaster();   
	for (ApplicationListener<?> listener : application.getListeners()) {      
		this.initialMulticaster.addApplicationListener(listener);   
	}
}
```
加载完监听器之后，开始传播 boot 启动的事件，`listeners.starting()`这个方法通知 SpringApplicationRunListeners 下的所有 SpringApplicationRunListener 去执行 starting() 方法
```java
public void starting() {   
	for (SpringApplicationRunListener listener : this.listeners) {      
		listener.starting();   
	}
}
```

因为通过上面发现如果单单只是 spring-boot 工程的话我们只有注册了一个EventPublishingRunListener，那我们直接来看 EventPublishingRunListener 的 starting() 方法。
```java
public void starting() {   
	this.initialMulticaster.multicastEvent(         
		new ApplicationStartingEvent(this.application, this.args));
}
```
当 EventPublishingRunListener 接收到事件之后会将广播出去，然后注册在 SpringApplication 中的所有 ApplicationListener 都会接受到 SpringApplication 初始化的广播事件，

再次列出spring-boot项目中默认包含的`ApplicationListener`
> Application Listenersorg.springframework.context.ApplicationListener=
>\org.springframework.boot.ClearCachesApplicationListener,
>\org.springframework.boot.builder.ParentContextCloserApplicationListener,
>\org.springframework.boot.context.FileEncodingApplicationListener,
>\org.springframework.boot.context.config.AnsiOutputApplicationListener,
>\org.springframework.boot.context.config.ConfigFileApplicationListener,
>\org.springframework.boot.context.config.DelegatingApplicationListener,
>\org.springframework.boot.context.logging.ClasspathLoggingApplicationListener,
>\org.springframework.boot.context.logging.LoggingApplicationListener,
>\org.springframework.boot.liquibase.LiquibaseServiceLocator

ApplicationListener目前 SpringApplication 初始化的事件各个 ApplicationListener ，这里不详细介绍全部，只挑一些关键的ApplicationListener 。
当 SpringApplication 的初始化的广播事件广播完成之后，就会开始着手准备 SpringApplication 的运行环境。

#### SpringApplication的environment环境对象创建与配置

SpringApplication环境创建和配置都在prepareEnvironment方法中，这个Environment环境变量代表着Spring-boot工程的运行环境，主要包括一些运行参数和配置的属性参数。

```java
ConfigurableEnvironment environment =prepareEnvironment(listeners,applicationArguments);
private ConfigurableEnvironment prepareEnvironment(      
	SpringApplicationRunListeners listeners,      
	ApplicationArguments applicationArguments) {   // Create and configure the environment   
	ConfigurableEnvironment environment = getOrCreateEnvironment();   
	configureEnvironment(environment, applicationArguments.getSourceArgs());   
	listeners.environmentPrepared(environment);   
	bindToSpringApplication(environment);   
	if (this.webApplicationType == WebApplicationType.NONE) {      
		environment = new EnvironmentConverter(getClassLoader())            .convertToStandardEnvironmentIfNecessary(environment);   
	}   
	ConfigurationPropertySources.attach(environment);   
	return environment;
}
```
SpringApplication环境的准备主要是创建一个环境和配置该环境。
```java
private ConfigurableEnvironment getOrCreateEnvironment() {   
	if (this.environment != null) {      
		return this.environment;   
	}   
	if (this.webApplicationType == WebApplicationType.SERVLET) {      
		return new StandardServletEnvironment();   
	}   
	return new StandardEnvironment();
}
```
环境也是由 SpringApplication 方法创建并记录在environment成员变量中，的创建过程并没有复杂的地方，只是根据情况创建 servlet 环境还是普通环境。先来看看 StandardEnvironment 包含了什么。
``` text
/***
 * {@link Environment} implementation suitable for use in 'standard' (i.e. non-web)
 * applications.
 *  <p>In addition to the usual functions of a {@link ConfigurableEnvironment} such as
 *  property resolution and profile-related operations, this implementation configures two
 *  default property sources, to be searched in the following order:
 *  <ul>
 *   <li>{@linkplain AbstractEnvironment#getSystemProperties() system properties}
 *   <li>{@linkplain AbstractEnvironment#getSystemEnvironment() system environment variables}
 *  </ul>
 *  That is, if the key "xyz" is present both in the JVM system properties as well as in
 *  the set of environment variables for the current process, the value of key "xyz" from
 *  system properties will return from a call to {@code environment.getProperty("xyz")}.
 *  This ordering is chosen by default because system properties are per-JVM, while
 *  environment variables may be the same across many JVMs on a given system. Giving
 *  system properties precedence allows for overriding of environment variables on a
 *  per-JVM basis.
 *  <p>These default property sources may be removed, reordered, or replaced; and
 *  additional property sources may be added using the {@link MutablePropertySources}
 *  instance available from {@link #getPropertySources()}. See
 *  {@link ConfigurableEnvironment} Javadoc for usage examples.
 *  <p>See {@link SystemEnvironmentPropertySource} javadoc for details on special handling
 *  of property names in shell environments (e.g. Bash) that disallow period characters in
 *  variable names.
 */
```
根据顶头部分的介绍知道 `StandardEnvironment` 是一个用来解析和存储`系统属性 (system properties)` 和 `系统环境变量(system environment variables)` ,而StandardEnvironment只是重构了父类 customizePropertySources 方法，而  customizePropertySources 方法则是在父类的无参构造函数中调用。StandardEnvironment 中的 customizePropertySources方法是把系统属性和系统环境变量存储到 环境(environment)对象 的属性集(propertySources) 属性中。

```java
protected void customizePropertySources(MutablePropertySources propertySources) {   
	propertySources.addLast(new MapPropertySource(SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME, getSystemProperties()));   
	propertySources.addLast(new SystemEnvironmentPropertySource(SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME, getSystemEnvironment()));
}
```
而如果是一个 SERVLET 环境，亦即在创建 SpringApplication 时候通过检测得到的web环境类型是 WebApplicationType.SERVLET 则会创建 StandardServletEnvironment 环境对象，该对象也是同样重构了父类 customizePropertySources 方法。
```java
protected void customizePropertySources(MutablePropertySources propertySources) {   	
	propertySources.addLast(new StubPropertySource(SERVLET_CONFIG_PROPERTY_SOURCE_NAME));   
	propertySources.addLast(new StubPropertySource(SERVLET_CONTEXT_PROPERTY_SOURCE_NAME));   
	if (JndiLocatorDelegate.isDefaultJndiEnvironmentAvailable()) {      
	propertySources.addLast(new JndiPropertySource(JNDI_PROPERTY_SOURCE_NAME));   
	}   
	super.customizePropertySources(propertySources);
}
```
值得一提，StandardServletEnvironment是继承StandardEnvironment类的，所以如果是一个 StandardServletEnvironment 的环境也会加载 系统属性 (system properties) 和 系统环境变量(system environment variables) 到属性集中，StandardServletEnvironment 则是额外再添加了 SERVLET 的上下文属性和 SERVLET 的配置属性。到此环境创建完毕。    

#### 环境配置
```java
protected void configureEnvironment(ConfigurableEnvironment environment,      String[] args) {   
	configurePropertySources(environment, args);  configureProfiles(environment, args);}
```
环境配置看得出只有两步，配置PropertySources和Profiles，这两个都是获取配置属性的方法。而PropertySources是上文讲到的在新建 Environment 时候就会把一些必要的 
PropertySources 保存到 Environment 的中。 先来看看PropertySources属性是怎样配置的。
```java
/*** Add, remove or re-order any {@link PropertySource}s in this application's* environment.
 * @param environment this application's environment
 * @param args arguments passed to the {@code run} method
 * @see #configureEnvironment(ConfigurableEnvironment, String[])
 * */
protected void configurePropertySources(ConfigurableEnvironment environment,      String[] args) {   
	MutablePropertySources sources = environment.getPropertySources();   
	if (this.defaultProperties != null && !this.defaultProperties.isEmpty()) {      
		sources.addLast(new MapPropertySource("defaultProperties",this.defaultProperties));   
	}   
	if (this.addCommandLineProperties && args.length > 0) {      
		String name = CommandLinePropertySource.COMMAND_LINE_PROPERTY_SOURCE_NAME;      
		if (sources.contains(name)) {         
			PropertySource<?> source = sources.get(name);         
			CompositePropertySource composite = new CompositePropertySource(name);         
			composite.addPropertySource(new SimpleCommandLinePropertySource(               "springApplicationCommandLineArgs", args));         
			composite.addPropertySource(source);         
			sources.replace(name, composite);      
		}else {         
			sources.addFirst(new SimpleCommandLinePropertySource(args));      
		}  
	}
}
```
configurePropertySources 首先会额外添加默认的PropertySources 如果有的话，这个默认的 PropertySources 需要在调用 SpringApplication 的run方法之前调用setDefaultProperties将其注入。而CommandLinePropertySource是指启动时传入的参数，就是熟悉的 “java project --server.port = 80” 中的--server.port = 80，这些附加参数封装成CompositePropertySource并一起加到环境environment的属性资源集PropertySources中。在设置完资源集之后，就会去加载 boot 激活的属性文件

```java
/*** Configure which profiles are active (or active by default) for this application
 * environment. Additional profiles may be activated during configuration file
 * processing via the {@code spring.profiles.active} property.
 * @param environment this application's environment
 * @param args arguments passed to the {@code run} method
 * @see #configureEnvironment(ConfigurableEnvironment, String[])
 * @see org.springframework.boot.context.config.ConfigFileApplicationListener
 * */

protected void configureProfiles(ConfigurableEnvironment environment, String[] args) {   
	environment.getActiveProfiles(); // ensure they are initialized   // But these ones should go first (last wins in a property key clash)   
	Set<String> profiles = new LinkedHashSet<>(this.additionalProfiles);   
	profiles.addAll(Arrays.asList(environment.getActiveProfiles()));   
	environment.setActiveProfiles(StringUtils.toStringArray(profiles));
}
```
如上所示，在注释中已经大致说明了configureProfiles的过程，就是加载 spring.profiles.active 属性指定的 profiles 文件，spring.profiles.active就是在实际开发中用于指定不同环境下使用哪个属性配置文件的一个属性。在代码中是通过`environment.getActiveProfiles()`来激活指定的属性文件profiles，并且会额外增加属性文件Set<String> profiles = new LinkedHashSet<>(this.additionalProfiles);这个额外文件是留给boot的扩展时使用，比如说在spring-cloud启动的时候就会把cloud自己的Profiles写到 spring-cloud的应用启动引导对象中，而最终Profiles文件中定义的属性最后会一并合并到主环境的 SpringApplication 的中。配置Environment环境的属性集到这里已经完成，因为Environment实际上是Spring-boot工程的环境的一个抽象，所以Environment中配置的属性最后都会在SpringApplication中应用到。
####环境配置完成事件传播
在环境创建完成和对环境配置完成之后，就会通知注册到SpringApplication的监听器，广播一个环境创建完成事件 `ApplicationEnvironmentPreparedEvent`代表环境配置完成，而在这个过程中事件广播器和事件接收器都是与 SpringApplication 开始创建事件的广播器和事件接收器一样。
```java
//环境配置完成事件传播代码
listeners.environmentPrepared(environment);
bindToSpringApplication(environment);
```
这次SpringApplication传播的是环境准备完成的事件，最后接收消息的还是在factories文件中配置的listener。
	1. ClearCachesApplicationListener
	2. ParentContextCloserApplicationListener
	3. FileEncodingApplicationListener
	4. AnsiOutputApplicationListener
	5. ConfigFileApplicationListener
	6. DelegatingApplicationListener
	7. ClasspathLoggingApplicationListener
	8. LoggingApplicationListener
	9. LiquibaseServiceLocatorApplicationListener

这了说几个值得关注的ApplicationListener，
1. `DelegatingApplicationListener`主要留给boot扩展应用去在这个时候有机会去改变 SpringApplication 的一些内容，跟`DelegatingApplicationContextInitializer`有点相像。
2.  ConfigFileApplicationListener ，处理过程代码如下：
```java
private void onApplicationEnvironmentPreparedEvent(      ApplicationEnvironmentPreparedEvent event) {   
	List<EnvironmentPostProcessor> postProcessors = loadPostProcessors();   
	postProcessors.add(this);   
	AnnotationAwareOrderComparator.sort(postProcessors);   
	for (EnvironmentPostProcessor postProcessor : postProcessors) {      
		postProcessor.postProcessEnvironment(event.getEnvironment(),            event.getSpringApplication());   
	}
}
```

其中
```java
loadPostProcessorsList<EnvironmentPostProcessor> loadPostProcessors() {   
	return SpringFactoriesLoader.loadFactories(EnvironmentPostProcessor.class,         getClass().getClassLoader());
}
```
同样的，loadPostProcessors也是获取META-INF/spring.factories中的EnvironmentPostProcessor，boot工程项目中的EnvironmentPostProcessor配置列表如下：
```text
# Environment Post Processorsorg.springframework.boot.env.EnvironmentPostProcessor=
\org.springframework.boot.env.SpringApplicationJsonEnvironmentPostProcessor,
\org.springframework.boot.env.SystemEnvironmentPropertySourceEnvironmentPostProcessor
```

`SpringApplicationJsonEnvironmentPostProcessor`主要是对JSON格式的配置做解析，`SystemEnvironmentPropertySourceEnvironmentPostProcessor`这个处理器的作用是对环境中的操作系统环境变量作一个修正。不多做分析，有兴趣可以细读这些Processor的postProcessEnvironment方法。在环境事件传播完成之后环境的建立就基本完成。

####创建ApplicationContext

在环境的建立完成之后就可以创建Spring的应用上下文ApplicationContext， 
```java
context = createApplicationContext();
```
在createApplicationContext中，创建了一个ConfigurableApplicationContext实例。
```java
protected ConfigurableApplicationContext createApplicationContext() {   
	Class<?> contextClass = this.applicationContextClass;   
	if (contextClass == null) {      
		try {         
			switch (this.webApplicationType) {         
				case SERVLET:            
					contextClass = Class.forName(DEFAULT_WEB_CONTEXT_CLASS);            
					break;         
				case REACTIVE:            
					contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);            
					break;         
				default:            
					contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);         
			}      
		}catch (ClassNotFoundException ex) {         
			throw new IllegalStateException("Unable create a default ApplicationContext, "+ "please specify an ApplicationContextClass",               ex);      
		}   
	}   
	return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
}
```
不同的环境会使用不同的ApplicationContext：
	1. SERVLET环境会使用AnnotationConfigServletWebServerApplicationContext
	2. REACTIVE环境会使用AnnotationConfigReactiveWebServerApplicationContext
	3. 默认环境下会使用AnnotationConfigApplicationContext

创建过程是通过反射机制来调用无参构造函数创建
```java
prepareContext(context, environment, listeners, applicationArguments,      printedBanner);
```
创建完之后，跟创建环境一样的逻辑开始对ApplicationContext做些准备。在prepareContext方法中，我们可以看到在主要的流程大致分为：
	1. 对应用上下文做一些额外功能的初始化
	2. 增加一些特殊用途的bean
	3. 加载资源

```java
private void prepareContext(ConfigurableApplicationContext context,      ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,ApplicationArguments applicationArguments, Banner printedBanner) {   
	context.setEnvironment(environment);   
	postProcessApplicationContext(context);   
	applyInitializers(context);   
	listeners.contextPrepared(context);   
	if (this.logStartupInfo) {      
		logStartupInfo(context.getParent() == null);      
		logStartupProfileInfo(context);   
	}   // Add boot specific singleton beans   
	context.getBeanFactory().registerSingleton("springApplicationArguments",applicationArguments);   
	if (printedBanner != null) {      
		context.getBeanFactory().registerSingleton("springBootBanner", printedBanner);   
	}   // Load the sources   
	Set<Object> sources = getAllSources();   
	Assert.notEmpty(sources, "Sources must not be empty");   
	load(context, sources.toArray(new Object[0]));   
	listeners.contextLoaded(context);
}
```
一个个来看，先从postProcessApplicationContext方法
```java
protected void postProcessApplicationContext(ConfigurableApplicationContext context) {   
	if (this.beanNameGenerator != null) {      
		context.getBeanFactory().registerSingleton(AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR,this.beanNameGenerator);   
	} 
	if (this.resourceLoader != null) {      
		if (context instanceof GenericApplicationContext) {         
			((GenericApplicationContext) context).setResourceLoader(this.resourceLoader);      
		}      
		if (context instanceof DefaultResourceLoader) {         
			((DefaultResourceLoader) context).setClassLoader(this.resourceLoader.getClassLoader());      
		}   
	}
}
```
postProcessApplicationContext方法会注册一个默认的beanNameGenerator名字生成器，这个名字生成器只有在我们定义Bean的时候没有给定bean的名字才用得上，此时会根据一定的策略来生成一个默认的名字。第二个则是添加一个resourceLoader资源加载器，是用于加载Spring相关资源的组件，比如application.properties。

####初始化上下文
```java
protected void applyInitializers(ConfigurableApplicationContext context) {   	
	for (ApplicationContextInitializer initializer : getInitializers()) {      
		Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(initializer.getClass(), ApplicationContextInitializer.class);
		Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");      
		initializer.initialize(context);   
	}
}
```
在初始化上下文的步骤中实际上则是把注册到SpringApplication中的ApplicationContextInitializer对生成的ApplicationContext进行一个处理。在ApplicationContext实际refreshe方法调用之前留给SpringApplication一个机会来对ApplicationContext做一些额外的修改或者增加某些功能。单纯的spring-boot工程会应用到的是以下
```text
ApplicationContextInitializer# Application Context Initializersorg.springframework.context.ApplicationContextInitializer=
\org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer,
\org.springframework.boot.context.ContextIdApplicationContextInitializer,
\org.springframework.boot.context.config.DelegatingApplicationContextInitializer
```

单纯的spring-boot工程使用到的ApplicationContextInitializer并没有太多的处理过程，DelegatingApplicationContextInitializer这个名字或多或少有点印象，这个以`Delegating-`开头的类都是预留给用户去扩展的接口，这里`DelegatingApplicationContextInitializer`会读取环境的配置属性context.initializer.classes设置的实现了ApplicationContextInitializer的类，`ApplicationContextInitializer`接口只有一个`initialize`方法，会以创建好的`ApplicationContext`作为入参。

####contextPrepared事件传播

在众多的`ApplicationContextInitializer`对Spring应用上下文`ApplicationContext`做完修改之后会广播一个应用上下文创建完成的事件，这个事件目前并没有做太多处理是一个空方法，暂时应该是做预留的一个方法。

####加载资源 <a id="load"></a>

回到prepareContext方法中，prepareContext方法的末端都有
```java
// Load the sources
Set<Object> sources = getAllSources();
Assert.notEmpty(sources, "Sources must not be empty");
load(context, sources.toArray(new Object[0]));
listeners.contextLoaded(context);
```
这里的资源并非指上文说的在准备环境的时候那些资源，在环境准备阶段加载的资源都是一些属性集的资源，像 `spring.config.location = PATH`之类的属性集合。但是这里资源加载的并不是指这种环境加载的属性，这里的资源在官方的解释是：一个class名字、一个包路径名字或者是一个XML文件的位置。(A source can be: a class name, package name, or an XML resource location)。

通过getAllSources方法获得出的Sources会包含两部分，第一部分是主资源，第二部分是额外资源。主资源是我们在构造SpringApplication这个对整个Boot工程作抽象的对象时，传入的参数Class对象，而额外资源是在执行SpringApplication对象的run方法之前需要我们通过setSources方法来注入我们希望额外加载的资源。

其实这里的资源sources其实是指我们bean配置的文件，而在随后的 load 方法中，获得到的sources如果是一个class名字那么将会把这个class转换成BeanDefinition并且注册到ApplicationContext中( 实际上是ApplicationContext下的bean管理容器BeanDefinitionRegistry中，可能ApplicationContext本身就是一个bean管理容器，否则就是ApplicationContext中的BeanFactory )；

如果是一个包路径名字，那么将会该包路径下的bean全部封装成BeanDefinition注册到ApplicationContext中。

如果是一个XML文件，那么会把XML中定义的bean全部注册到ApplicationContext中。

####刷新上下文

上下文刷新是正式启动Spring容器的一步 ，`AnnotationConfigServletWebServerApplicationContext`是SERVLET环境下的ApplicationContext实例对象，简单阅读该类的创建过程。
![Alt text](./AnnotationConfigServletWebServerApplicationContext.png)

通过上面的类图可以看出`AnnotationConfigServletWebServerApplicationContext`使用了许多接口去修饰，这里列出一些关键的接口和父类及其无参构造方法，因为创建AnnotationConfigServletWebServerApplicationContext上下文是通过无参方法实例化的：
	>1. `AbstractApplicationContext`：AbstractApplicationContext中主要是提供一些对BeanFactory管理的方法和消息传播相关的方法，AbstractApplicationContext还实现了ResourceLoader所以还会提供资源解析相关的方法，AbstractApplicationContext当中最重要的还是refresh()方法，refresh方法决定了Spring核心的启动流程。而AbstractApplicationContext的无参构造方法只是生成了一个默认的资源解析器ResourcePatternResolver。
	>2. `GenericApplicationContext`：GenericApplicationContext提供了更多的BeanDefinition的管理方法，BeanDefinition是bean在Spring中的抽象定义。GenericApplicationContext的无参构造方法只是创建一个DefaultListableBeanFactory。
	>3. `GenericWebApplicationContext`：GenericWebApplicationContext是在  GenericApplicationContext 的基础上增加了对ServletContext的支持。
	>4. `ServletWebServerApplicationContext`：ServletWebServerApplicationContext进一步完善了对web环境的功能和加入了对内嵌的web server的支持。
	>5. `AnnotationConfigServletWebServerApplicationContext`：AnnotationConfigServletWebServerApplicationContext继承了以上的父类同时支持注解方式配置需要配装的bean。

上文也说过WEB环境下Spring-boot工程默认启动的ApplicationContext是`AnnotationConfigServletWebServerApplicationContext`。在`AnnotationConfigServletWebServerApplicationContext`的构造方法中注入了两个组件，一个是`BeanDefinitionReader`，另外一个是`BeanDefinitionScanner`。Scanner是根据一定的策略去扫描项目中定义的Bean组件，而Reader则是注册指定的配置方式中配置的bean。一般而言，Scanner通常使用的是基于类路径(ClassPath)上的扫描方式，对给定的类路径下的全部加上 `@Component` 且符合过滤规则的类都生成对应的`BeanDefinition`并且注册到Spring容器中。

虽然`AnnotationConfigServletWebServerApplicationContext`中通过扫描或者注册主类(添加了`@SpringBootApplication`的类)来读取装载应用需要的Bean，但是在Spring-boot中并没有使用 AnnotationConfigServletWebServerApplicationContext 来读取扫描。而是通过上文 <a href="#load">加载资源</a> 一节提到过的 BeanDefinitionLoader 来读取装载。

不知为何是这样设计，不过在`BeanDefinitionLoader`中不但支持BeanDefinitionReader或者是BeanDefinitionScanner来读取bean配置同时还支持以XML的形式来读取bean，同时还支持Groovy脚本来读取，可能是为了适配多种环境。不管是哪种类型的ApplicationContext几乎都是通过AbstractApplicationContext的refresh()来刷新启动Spring容器，这里不对Spring-core的内容做详细解读，只对refresh中流程简单介绍：
	1. 销毁已有的beanfactory。
	2. 创建一个新的beanfactory，类型由子类决定。
	3. 子类修改beanfactory，留给一个机会去准备beanfactory的一些参数
	4. 执行beanfactory后置处理器(BeanFactoryPostProcessor)，这些后置处理器会在加载bean之前调用，留给ApplicationContext最后的机会对beanfactory做修改的机会。
	5. 注册Bean后置器BeanPostProcessors，Bean后置器是在beanfactory配装bean完成之后对Bean进行进一步修改。
	6. 广播beanfactory刷新事件。
	7. 配装非懒加载的bean。

在ApplicationContext启动过程中有一个很重要的beanfactory后置处理器`ConfigurationClassPostProcessor`，这个后置处理器是在准备ApplicationContext上下文时，SpringApplication 针对 ApplicationContext 上下文加载资源创建 <a href="#load" >`BeanDefinitionLoader`</a>(上文提到)时候就已经把ConfigurationClassPostProcessor加入到ApplicationContext中。

在应用上下文ApplicationContext应用BeanFactoryPostProcessor后置处理器时，由于实现了BeanDefinitionRegistryPostProcessors(BeanFactoryPostProcessor子类)的`ConfigurationClassPostProcessor`会被优先执行，BeanDefinitionRegistryPostProcessors接口会在ApplicationContext刷新时调用其postProcessBeanDefinitionRegistry方法，在ConfigurationClassPostProcessor这个后置处理器中会通过ConfigurationClassParser对在ApplicationContext中所有注解了`@Configuration`的类进行处理。如果注解了`@Configuration`的类同时还注解了`@Import`、`@ComponentScan`、`@ImportResource`、`@Bean`会一一解析并且实现这些注解对应的功能。
>我们经常使用的starter就是通过这种途径来实现自动配置。

####ApplicationContext刷新完成后处理
回到SpringApplication的run方法中，走到这里相信Spring-boot已经启动起来了，剩下的都是一些后续的工作。

```java
afterRefresh(context, applicationArguments);
stopWatch.stop();
if (this.logStartupInfo) {   
	new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
}
listeners.started(context);
callRunners(context, applicationArguments);
```
其中afterRefresh()为空方法，是留给以后做扩展用的预留函数。值得关注的是listeners.started()，在SpringApplication启动之时就会实例化项目路径下的META-INF/spring.factories中配置的SpringApplicationRunListener，并且广播了SpringApplication启动的事件。在这里listeners再次广播SpringApplication启动完成的事件 `ApplicationStartedEvent`。
实际上SpringApplication启动完成的事件是通过ApplicationContext来发布事件，详细过程可以参考相关文章。跑到这里基本上可以认为Spring-boot项目已经启动完成。之后会回调一些Runner，这些Runner实际意义是让我们在Spring-boot启动完的第一时间可以去执行一些我们希望系统做的一些事情。
```java
private void callRunners(ApplicationContext context, ApplicationArguments args) {   
	List<Object> runners = new ArrayList<>();  
	runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());   
	runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());   
	AnnotationAwareOrderComparator.sort(runners);   
	for (Object runner : new LinkedHashSet<>(runners)) {      
		if (runner instanceof ApplicationRunner) {         
			callRunner((ApplicationRunner) runner, args);      
		}      
		if (runner instanceof CommandLineRunner) {         
			callRunner((CommandLineRunner) runner, args);      
		}   
	}
}
```
这里是获取ApplicationContext中实现了ApplicationRunner或者CommandLineRunner接口的bean。这两个接口都只需要实现一个run方法，在Spring-boot项目启动完成之后，就会执行。

####后记
Spring-boot给我的感觉是对Spring的更高一层的抽象封装，利用Spring预留的众多扩展接口对Spring进行无代码入侵的扩展，使用起来Spring-boot就像是一个Spring应用一样，其中并没有很多新的技术融合到Spring-boot中，对实现简化配置的实现方法简单明了，只要在对应的starter项目中配置META-INF/spring.factories文件引入对应的类即可以实现自动配置。

在以前使用Spring-boot的时候并没有太多关注开箱即用的实现过程，只是觉得用起来很方便，在研究了Spring-cloud之后发现毕竟是基于Spring-boot的，研究源码的时候就顺便看了一眼，大概弄懂来龙去脉，也积累一点点代码阅读量，技术债迟早要还的。

看完之后如果喜欢的朋友点个心心呗。