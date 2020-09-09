#### 1、Spring有什么好处（特性），怎么管理对象的？

- IOC：Spring的IOC容器，将对象之间的创建和依赖关系交给Spring，降低组件之间的耦合性。即Spring来控制对象的整个生命周期。其实就是平常说的DI或者IOC。
- AOP：面向切面编程。可以将应用各处的功能分离出来形成可重用的组件。核心业务逻辑与安全、事务、日志等这些非核心业务逻辑分离，使得业务逻辑更简洁清晰。

- 使用模板消除了样板式的代码。比如使用JDBC访问数据库。
- 提供了对像关系映射（ORM）、事务管理、远程调用和Web应用的支持。

Spring使用IOC容器创建和管理对象，比如在XML中配置了类的全限定名，然后Spring使用反射+工厂来创建Bean。BeanFactory是最简单的容器，只提供了基本的DI支持，ApplicationContext基于BeanFactory创建，提供了完整的框架级的服务，因此一般使用应用上下文。

#### 2、什么是IOC？

IOC(Inverse of Control)即控制反转。可以理解为控制权的转移。传统的实现中，对象的创建和依赖关系都是在程序进行控制的。而现在由Spring容器来统一管理、对象的创建和依赖关系，控制权转移到了Spring容器，这就是控制反转。

#### 3、什么是DI？DI的好处是什么？

DI(Dependency Injection)依赖注入。对象的依赖关系由负责协调各个对象的第三方组件在创建对象的时候进行设定，对象无需自行创建或管理它们的依赖关系。通俗点说就是Spring容器为对象注入外部资源，设置属性值。DI的好处是使得各个组件之间松耦合，一个对象如果只用接口来表明依赖关系，这种依赖可以在对象毫不知情的情况下，用不同的具体类进行替换。

**IOC和DI其实是对同一种的不同表述**。

#### 4、什么是AOP，AOP的好处？

AOP(Aspect-Oriented Programming)面向切面编程，可以将遍布在应用程序各个地方的功能分离出来，形成可重用的功能组件。系统的各个功能会重复出现在多个组件中，各个组件存在于核心业务中会使得代码变得混乱。使用AOP可以将这些多处出现的功能分离出来，不仅可以在任何需要的地方实现重用，还可以使得核心业务变得简单，实现了将核心业务与日志、安全、事务等功能的分离。

具体来说，散布于应用中多处的功能被称为*横切关注点*，这些横切关注点从概念上与应用的业务逻辑是相分离的，但是又常常会直接嵌入到应用的业务逻辑中，AOP把这些横切关注点从业务逻辑中分离出来。安全、事务、日志这些功能都可以被认为是应用中的横切关注点。

通常要重用功能，可以使用继承或者委托的方式。但是继承往往导致一个脆弱的对像体系；委托带来了复杂的调用。面向切面编程仍然可以在一个地方定义通用的功能，但是可以用声明的方法定义这个功能要在何处出现，而无需修改受到影响的类。**横切关注点可以被模块化为特殊的类，这些类被称为切面（Aspect）**。好处在于：

- 每个关注点都集中在一个地方，而非分散在多处代码中；
- 使得业务逻辑更简洁清晰，因为这样可以只关注核心业务，次要的业务被分离成关注点转移到切面中了。

AOP术语介绍

**通知**：切面所做的工作称为通知。通知定义了切面是什么，以及在何时使用。Spring切面可以应用5种类型的通知

- 前置通知（Before）：在目标方法被调用之前调用通知功能；
- 后置通知（After）：在目标方法被调用或者抛出异常之后都会调用通知功能；
- 返回通知（After-returning）：在目标方法成功执行之后调用通知；
- 异常通知（After-throwing）：在目标方法抛出异常之后调用通知；
- 环绕通知（Around）：通知包裹了被通知的方法，在目标方法被调用之前和调用之后执行自定义的行为。
  

**连接点**：可以被通知的方法

**切点**：实际被通知的方法

**切面**：即通知和切点的结合，它是什么，在何时何处完成其功能。

**引入**：允许向现有的类添加新方法或属性，从而可以在无需修改这些现有的类情况下，让它们具有新的行为和状态。

**织入**：把切面应用到目标对象并创建新的代理对象的过程。切面在指定的连接点被织入到目标对象中。在目标对象的生命周期里有多个点可以进行织入：

- 编译期，切面在目标类编译时织入。
- 类加载期，切面在目标类加载到JVM时被织入。
- 运行期，切面在应用运行的某个时刻被织入，**在织入切面时，AOP容器会为目标对象动态地创建一个代理对象。Spring AOP就是以这种方式织入切面的**。

Spring AOP构建在动态代理基础之上，所以Spring对AOP的支持仅限于方法拦截。

Spring的切面是由包裹了目标对象的代理类实现的。代理类封装了目标类，并拦截被通知方法的调用，当代理拦截到方法调用时，在调用目标bean方法之前，会执行切面逻辑。**其实切面只是实现了它们所包装bean相同接口的代理**。

#### 5、AOP的实现原理：Spring AOP使用的动态代理。

Spring AOP中的动态代理主要有两种方式，JDK动态代理和CGLIB动态代理。JDK动态代理通过反射来接收被代理的类，并且要求被代理的类必须实现一个接口。JDK动态代理的核心是InvocationHandler接口和Proxy类。

如果目标类没有实现接口，那么Spring AOP会选择使用CGLIB来动态代理目标类。CGLIB（Code Generation Library），是一个代码生成的类库，可以在运行时动态的生成某个类的子类，注意，CGLIB是通过继承的方式做的动态代理，因此如果某个类被标记为final，那么它是无法使用CGLIB做动态代理的。

Spring使用动态代理，代理类封装了目标类，当代理拦截到方法调用时，在调用目标bean的方法之前，会执行切面逻辑。

#### 6、Spring的生命周期？

Spring创建、管理对象。Spring容器负责创建对象，装配它们，配置它们并管理它们的整个生命周期。

- 实例化：Spring对bean进行实例化
- 填充属性：Spring将值和bean的引用注入到bean对应的属性中
- 调用BeanNameAware的setBeanName()方法：若bean实现了BeanNameAware接口，Spring将bean的id传递给setBeanName方法
- 调用BeanFactoryAware的setBeanFactory()方法：若bean实现了BeanFactoryAware接口，Spring调用setBeanFactory方法将BeanFactory容器实例传入
- 调用ApplicationContextAware的setApplicationContext方法：如果bean实现了ApplicationContextAware接口，Spring将调用setApplicationContext方法将bean所在的应用上下文传入
- 调用BeanPostProcessor的预初始化方法：如果bean实现了BeanPostProcessor，Spring将调用它们的叛postProcessBeforeInitialization方法
- 调用InitalizingBean的afterPropertiesSet方法：如果bean实现了InitializingBean接口，Spring将调用它们的afterPropertiesSet方法
- 如果bean实现了BeanPostProcessor接口，Spring将调用它们的postProcessAfterInitialzation方法
- 此时bean已经准备就绪，可以被应用程序使用，它们将一直驻留在应用杀死那个下文中，直到该应用的上下文被销毁。
- 如果bean实现了DisposableBean接口，Spring将调用它的destroy方法。

####  7、Spring的配置方式，如何装配bean？bean的注入方法有哪些？

- XML配置，如`<bean id="">`
- Java配置即JavaConfig，使用`@Bean`注解
- 自动装配，组件扫描（component scanning）和自动装配（autowiring），`@ComponentScan`和`@AutoWired`注解

bean的注入方式有：

- 构造器注入
- 属性的setter方法注入
  

推荐对于强依赖使用构造器注入，对于弱依赖使用属性注入。

#### 8、bean的作用域？

- 单例（Singleton）：在整个应用中，只创建bean一个实例。
- 原型（Prototype）：每次注入或通过Spring应用上下文获取时，都会创建一个新的bean实例。
- 会话（Session）：在Web应用中，为每个会话创建一个bean实例。
- 请求（Request）：在Web应用中，为每个请求创建一个bean实例。

默认情况下Spring中的bean都是单例的。

#### 9、Spring中涉及到哪些设计模式？

- 工厂方法模式。在各种BeanFactory以及ApplicationContext创建中都用到了；
- 单例模式。在创建bean时用到，Spring默认创建的bean是单例的；
- 代理模式。在AOP中使用Java的动态代理；
- 策略模式。比如有关资源访问的Resource类
- 模板方法。比如使用JDBC访问数据库，JdbcTemplate。
- 观察者模式。Spring中的各种Listener，如ApplicationListener
- 装饰者模式。在Spring中的各种Wrapper和Decorator
- 适配器模式。Spring中的各种Adapter，如在AOP中的通知适配器AdvisorAdapter

#### 10、MyBatis和Hibernate的区别和应用场景？

Hibernate :是一个标准的ORM(对象关系映射) 框架； SQL语句是自己生成的，程序员不用自己写SQL语句。因此要对SQL语句进行优化和修改比较困难。适用于中小型项目。

MyBatis： 程序员自己编写SQL， SQL修改和优化比较自由。 MyBatis更容易掌握，上手更容易。主要应用于需求变化较多的项目，如互联网项目等。