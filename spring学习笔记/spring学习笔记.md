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



















