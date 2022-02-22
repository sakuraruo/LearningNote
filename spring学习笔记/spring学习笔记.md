1.什么是BeanDefinition？

BeanDefinition是JavaBean的声明，Spring根据BeanDefinition来创建Bean对象。

2.BeanDefinition中的重要属性

beanClass、scope、isLazy、dependsOn、primary、initMethodName

3.什么是BeanFactory？

BeanFactory是一种Spring容器，是Bean工厂，是顶层容器接口，下面有很多实现，主要用来获取Bean、创建bean。

4.BeanFactory的核心子接口和实现类

ListableBeanFactory、ConfigurableBeanFactory、AutowireCapableBeanFactory、AbstractBeanFactory、DefaultListableBeanFactory

5.DefaultListableBeanFactory的功能

支持单例Bean、支持Bean别名、支持父子BeanFactory、支持Bean类型转换、支持Bean后置处理、支持FactoryBean、支持自动装配等。

6.什么是Bean生命周期？

Bean生命周期描述的是Spring中的一个Bean创建过程和销毁过程中所经历的步骤。可以利用Bean生命周期机制对Bean进行自定义加工。

7.Bean创建生命周期的重要阶段

BeanDefinition(Bean定义)=>构造方法推断(选出一个构造方法)=>实例化(构造方法反射得到对象)=>属性填充(对属性进行自动填充)=>初始化(对其他属性赋值、校验)=>初始化后(AOP、生成代理对象)

8.实例化

通过构造方法反射得到一个实例化对象，在Spring中，可以通过BeanPostProcessor机制大队实例化进行干预。

9.属性填充

实例化所得到的对象，是不完整的对象。属性填充则是自动给某些属性进行赋值，属性填充就是自动注入和依赖注入。

10.初始化

在对一个对象进行属性填充之后，Spring提供了初始化机制，可以在初始化阶段对Bean进行自定义加工，比如可以利用InitializingBean接口来对Bean中的其他属性进行赋值，或者对Bean中的某些属性进行校验。
11.初始化后

初始化后是Bean创建生命周期中最后一个步骤，常说的AOP机制就是在这个步骤通过BeanPostProcessor机制实现的，初始化之后得到的对象才是真正的Bean对象。

12.@autowired是什么？

@autowired表示某个属性是否需要进行依赖注入，可以写在属性和方法上。注解中的required属性默认为true，表示如果没有对象可以注入给属性则抛异常。Spring会先根据属性的类型去Spring容器中找出该类型所有的Bean对象，如果找出来多个，则再根据属性的名字从多个中再确认一个。如果required属性为true，并且根据属性信息找不到对象，则直接抛异常。

当@autowired注解写在某个方法上时，spirng在Bean生命周期的属性填充阶段，会根据方法的参数类型、参数名字从Spring容器找到对象当作方法入参，自动反射调用该方法。

@AutoWired加在构造方法上时，Spring会在推断构造方法阶段，选择该构造方法来进行实例化，在反射调用构造方法之前，会先根据构造方法参数类型、参数名从Spring容器中找到Bean对象，当做构造方法入参。

13.@Resource是什么？

@Resource注解与@Autowired类似，也是用来进行依赖注入的，@Resouce是Java层面所提供的注解，@Autowired是Spring所提供的注解，它们依赖注入的底层实现逻辑也不同。

@Resource注解中有一个name属性，针对name属性是否有值，@Resource的依赖注入底层流程有所不同。如果name有值，则直接去Spring容器中找Bean对象，找不到则报错，找到了则成功；@Resource中的name属性没有值，则：

1）先判断该属性名字在Spring容器中是否存在Bean对象 2）如果存在，则成功找到Bean的对象进行注入 3）如果不存在，则根据属性类型去Spring容器找Bean对象，找到一个则进行注入。

14.@value是如何工作的

@Value注解和@Resouce、@Autowired类似，也是用来对属性进行依赖注入的。只不过@Value是用来从properties文件中获取值的，并且@Value可以解析spEl(Spring表达式)

会将${}中的字符串当作key，从properties文件中找出的对应的value赋值给属性，如果没找到就会把表达式中的值注入给属性

会将#{}中的字符串当作Spring表达式进行解析，Spring会把{}中的值当作beanName并从Spring容器中找对应的bean，如果找到则进行属性注入，没找到则报错。

15.FactoryBean是什么？

FactoryBean是Spring所提供的一种较灵活的创建Bean的方式，可以通过实现FactoryBean接口中的getObject方法来返回一个对象，这个对象就是最终的Bean对象。

FactoryBean接口中的方法：Object getObject()返回的是Bean对象；boolean isSingleton()返回的是否是单例对象；Class getObjectType(); 返回的是Bean对象的类型。

FactoryBean对象本身也是一个Bean，同时它相当于一个小型工厂，可以生产出另外的Bean。BeanFactory是一个Spring容器，是一个大型工厂，它可以生产出各种各样的bean。

FactoryBean机制被广泛的应用在Spring内部和Spring与第三方框架或组件的整合过程中。

16.什么是ApplicationContext？

ApplicationContext是比BeanFactory更加强大的Spring容器，它既可以创建Bean、获取Bean还支持国际化、事件广播、获取资源等BeanFactory不具备的功能。

ApplicationContext所继承的接口：EnvironmentCapable、ListableBeanFactory、HierarchicalBeanFactory、MessageSource、ApplicationEventPublisher、ResourcePatternResolver。

EnvironmentCapable继承这个接口，表示拥有了获取环境变量的功能，可以通过ApplicationContext获取操作系统环境变量和JVM环境变量。

ListableBeanFactory继承了这个接口，就拥有了获取所有BeanNames、判断某个beanName是否存在BeanDefinition对象、统计BeanDefinition个数、获取某个类型所有对应BeanNames等功能。

HierarchicalBeanFactory继承了这个接口，就拥有了获取父BeanFactory、判断某个name是否存在bean对象的功能。

MessageSource继承了这个接口，就拥有了国际化功能，比如可以直接利用MessageSource对象获取某个国际化资源(比如不同国家语言所对应的字符）

ApplicationEventPublisher继承了这个接口，就拥有了事件发布功能，可以发布事件，这是Application相对于BeanFactory比较突出、常用的功能。

ResourcePatternResolver继承了这个接口，就拥有了加载并获取资源的功能，这里的资源可以是文件，图片等某个URL资源都可以。

17.什么是BeanPostProcessor？

BeanPostProcessor是Spring所提供的一种扩展机制，可以利用该机制对Bean进行定制化加工，在Spring底层源码实现中，也广泛的用到了该机制，BeanPostProcessor通常也叫做Bean后置处理器。

BeanPostProcessor在Spring中的一个接口，我们定义一个后置处理器，就是提供一个类实现该接口，在Spring中还存在一些接口集成了BeanPostprocessor，这些子接口是在BeanPostProcessor的基础上增加了一些其他功能。

BeanPostProcessor中的方法：postProcessBeforeInitialization() 初始化前方法，表示可以利用这个方法来对Bean在初始化前进行自定义加工；postProcessAfterInitialization()初始化后方法，表示可以利用这个方法来对Bean在初始化后进行自定义加工。

InstantiationAwareBeanPostProcessor，BeanPostProcessor的一个子接口，postProcessBeforeInstantiation()：实例化前；postProcessAfterInstantiation():实例化后；postProcessProperties():属性注入后

18.什么是AOP？

AOP是面向切面编程，是一种非常适合在无需修改业务代码的前提下，对某个或某些业务增加统一的功能，比如日志记录、权限控制、事务管理等，能很好的使得代码解耦，提高开发效率。

AOP中的核心概念：Advice、PointCut、Advisor、Weaving、Target、JoinPoint

Advice可以理解为通知、建议，在Spring中通过定义Advice来定义代理逻辑。

Pointcut是切点，表示Advice对应的代理逻辑应用在哪个类、哪个方法上。

Advisor等于advice+pointcut，表示代理逻辑和切点的一个整体，可以通过定义或封装一个advisor，来定义切点和代理逻辑。

Weaving表示织入，将Advice的代理逻辑在源代码级别嵌入到切点的过程，就叫做织入。

Target表示目标对象，也就是被代理对象，在AOP生成的代理对象中会持有目标对象。

JoinPoint表示连接点，在Spring-AOP中，就是方法的执行点。

AOP的工作原理：

AOP是发生在Bean的生命周期过程中的：

1.Spring生成Bean对象时，先实例化出来一个对象，也就是target对象。

2.再去target对象进行填充。

3.在初始化后步骤中，会判断target对象有没有对应的切面。

4.如果有切面，就表示当前target对象需要进行aop。

5.通过cglib、jdk动态代理机制生成一个代理对象，作为最终的bean对象。



19.JavaBean、SpringBean、对象之间的区别

对象指Java语言中的类实例、类对象。

JavaBean指的是属性为私有，只提供get set方法的类。

SpringBean则是代表由Spring创建管理的对象。



20.Bean标签、@Bean、@component等注解的区别。

如果是从xml中读取则是使用ClassPathXmlApplicationContext；如果是注解，则是使用AnnotationConfigApplicationContext。

21.BeanDefinition定义一个Bean

声明一个BeanDefinition然后注入入Spring容器。



22.@componentScan注解之扫描索引

当META-INF目录下spring-component文件中存放需要扫描的类，如果配置了该类，则Spring不会再通过路径进行全局扫描。

23.@Conditional注解

@conditional注解中有一个数组属性，类型为Condition类的子类，实现match方法。该方法的入参中包含了很多属性。

24.@autowired注解

@AutoWired注解是首先根据类型去匹配找到多个的话再根据名称去找。

25.@lazy注解

@Lazy注解加在Bean上面，则代表Bean初始化实际放在getBean的时候。

可以通过@Lazy注解解决循环依赖报错的问题









































