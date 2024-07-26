# SpringBoot
一、什么是Spring Boot的自动配置？

Spring Boot的最大的特点就是简化了各种xml配置内容，还记得曾经使用SSM框架时我们在spring-mybatis.xml配置了多少内容吗？数据源、连接池、会话工厂、事务管理···，而现在Spring Boot告诉你这些都不需要了，一切交给它的自动配置吧！

所以现在能大概明白什么是Spring Boot的自动配置了吗？简单来说就是用注解来对一些常规的配置做默认配置，简化xml配置内容，使你的项目能够快速运行。

二、Spring Boot自动配置核心原理

在分析原理之前先来整体的看一下自动配置的核心原理图，作一个简单的了解。

简单来说，Spring Boot通过@EnableAutoConfiguration注解开启自动配置，对jar包下的spring.factories文件进行扫描，这个文件中包含了可以进行自动配置的类，当满足@Condition注解指定的条件时，便在依赖的支持下进行实例化，注册到Spring容器中。

下面我们将浅析Spring Boot自动配置原理的过程。 

三、三大注解

在启动类中可以看到@SpringBootApplication注解，它是SpringBoot的核心注解，也是一个组合注解。我们进入这个注解可以看到里面又包含了其它很多注解，但其中@SpringBootConfiguration、@EnableAutoConfiguration、@ComponentScan三个注解尤为重要。

	
@SpringBootConfiguration：继承自Configuration，支持JavaConfig的方式进行配置。

@EnableAutoConfiguration：本文重点讲解，主要用于开启自动配置。

@ComponentScan：自动扫描组件，默认扫描该类所在包及其子包下所有带有指定注解的类，将它们自动装配到bean容器中，会被自动装配的注解包括@Controller、@Service、@Component、@Repository等。也可以指定扫描路径。

四、@EnableAutoConfiguration

看到这个名字，大家也应该能猜到这个注解的作用，没错，这个注解可以帮助我们自动加载程序所需的默认配置。要想搞清楚自动配置的原理，我们就要拿它开刀。

继续进入@EnableAutoConfiguration，注意到这两个注解：@AutoConfigurationPackage和@Import，这里我们需要关心的是这个@Import注解

五、@Import

观察@Import(AutoConfigurationImportSelector.class)注解，这里导入了AutoConfigurationImportSelector类。这个类中有一个非常重要的方法——selectImports（），它几乎涵盖了组件自动装配的所有处理逻辑，包括获得候选配置类、配置类去重、排除不需要的配置类、过滤等，最终返回符合条件的自动配置类的

全限定名数组。下面源码图中对该方法进行了详尽的注释。

	@Override
	public String[] selectImports(AnnotationMetadata annotationMetadata) {
		//检查自动配置功能是否开启，默认开启
		if (!isEnabled(annotationMetadata)) {
			return NO_IMPORTS;
		}
		//加载自动配置的元信息
		AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
				.loadMetadata(this.beanClassLoader);
		AnnotationAttributes attributes = getAttributes(annotationMetadata);
		//获取候选配置类
		List<String> configurations = getCandidateConfigurations(annotationMetadata,
				attributes);
		//去掉重复的配置类
		configurations = removeDuplicates(configurations);
		//获得注解中被exclude和excludeName排除的类的集合
		Set<String> exclusions = getExclusions(annotationMetadata, attributes);
		//检查被排除类是否可实例化、是否被自动注册配置所使用，不符合条件则抛出异常
		checkExcludedClasses(configurations, exclusions);
		//从候选配置类中去除掉被排除的类
		configurations.removeAll(exclusions);
		//过滤
		configurations = filter(configurations, autoConfigurationMetadata);
		//将配置类和排除类通过事件传入到监听器中
		fireAutoConfigurationImportEvents(configurations, exclusions);
		//最终返回符合条件的自动配置类的全限定名数组
		return StringUtils.toStringArray(configurations);
	}

这里需要关注的的是如何得到候选的配置类，可以看到所有的配置信息通过getCandidateConfigurations()得到，并最终由一个列表保存。我们继续查看getCandidateConfigurations()方法。


继续进入loadFactoryNames()方法，可以发现一个获取资源的可疑地址：FACTORIES_RESOURCE_LOCATION

再进入FACTORIES_RESOURCE_LOCATION，发现值为：META-INF/spring.factories

public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";

简述以上过程就是：getCandidateConfigurations()方法通过SpringFactoriesLoader加载器加载META-INF/spring.factories文件，首先通过这个文件获取到每个配置类的url，再通过这些url将它们封装成Properties对象，最后解析内容存于Map<String,List<String>>中。

下面是spring.factories文件的内容格式，根据它我们可以清晰地了解Map<String,List<String>>中都存了些什么。其中Key值为：org.springframework.boot.autoconfigure.EnableAutoConfiguration，Value值为后面的各种XXXAutoConfiguration类。

最后通过loadFactoryNames传递过来的class名称作为Key从Map中获得该类的配置列表，而这个class名称是什么呢？回到之前的SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader())方法，注意第一个参数，是一个方法，我们进入这个方法，发现返回的是

EnableAutoConfiguration.class，这即是我们需要的class名称。

	/**
	 * Return the class used by {@link SpringFactoriesLoader} to load configuration
	 * candidates.
	 * @return the factory class
	 */
	protected Class<?> getSpringFactoriesLoaderFactoryClass() {
		return EnableAutoConfiguration.class;
	}
 
最终，getCandidateConfigurations（）方法获取到了候选配置类并存于List中。之所以说是“候选”，是因为它们后续还需要经过一系列的去重、排除、过滤等操作，最终会通过selectImports（）方法返回一个自动配置类的全限定名数组。

六、@Condition

在上面的步骤中我们得到了一个动配置类的全限定名数组，这些配置类需要在满足@Condition后才能真正的被注册到Spring容器之中。但在Spring Boot项目中我们更多的是看到@Condition注解的衍生注解，如下：

@ConditionOnBean	在容器中有指定Bean的条件下。
@ConditionalOnMissingBean	在容器中没有指定Bean的条件下。
@ConditionOnClass	在classpath类路径下有指定类的条件下。
@ConditionalOnMissingClass	在classpath类路径下没有指定类的条件下。
@ConditionalOnResource	类路径是否有指定的值。
@ConditionalOnWebApplication	在项目是一个Web项目的条件下。
@ConditionOnProperty	在指定的属性有指定的值条件下。
