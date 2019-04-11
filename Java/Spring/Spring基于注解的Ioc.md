## Spring(二)：基于注解的Ioc

Spring的Ioc可以通过xml配置的方式实现控制反转和依赖注入，也可以通过注解的方式。

### 基本使用

#### 1.用于创建对象

##### 使用注解：`@Component`

它的作用是将当前类对象存入到spring容器中。这个注解可以指定属性：

* value：指定这个bean的id，如果不指定默认是注解类的名称，并且首字母小写。
* ....

在AccountServiceImpl4上使用注解。没有参数value时，id默认是：accountServiceImpl4

``` java
@Component
public class AccountServiceImpl4 implements AccountService {
    public void saveAccount() {
        System.out.println("我被创建了");
    }
    public void init(){
        System.out.println("对象被创建了");
    }

    public void destroy(){
        System.out.println("对象被销毁了");
    }

}
```

如果Component中只有一个参数，那么也可以不写value，直接**@Component("accountServiceImpl4")**

然后告知spring在创建容器时要扫描的包，配置所需要的标签在context名称空间和约束中：

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd">

    <context:component-scan base-package="com.spring.day01"/>
</beans>
```

这就相当于在bean.xml中注册了这个类，因为这个标签将扫面这个package下所有用**@Component**标注的类，并进行相应的注册，然后就可以通过构造核心容器的方式取到相应的实例。

这种方式对应于原来依赖注入的以下方式：

``` xml
<bean id="accountService" class="com.spring.day01.service.impl.AccountServiceImpl"/>
```

##### 其他注解

除了@Component这个注解以外，还有另外3个注解，并且这三个注解的作用和属性和@Component都是一样的。他们是spring框架为我们提供的明确三层的注解，使三层更加清晰。

* @Controller：一般用于表现层
* @Service：一般用在业务层——service
* @Repository：一般用在持久层——dao
* @Component：通用层

``` java
@Repository
public class AccountDaoImpl implements IAccountDao {
    public void save() {
        System.out.println("保存成功");
    }
}
```

#### 2.用于注入数据

##### @AutoWired

它的作用是**自动按照类型**注入，只要容器中有唯一的一个bean对象和要注入的对象类型匹配，就可以注入成功。

出现的位置：可以是在类 变量上，也可以是在方法上。

AccountServiceImpl4类中有一个IAccountDao的接口对象，它被`@Autowired`标注了。

``` java
@Component(value = "accountServiceImpl")
public class AccountServiceImpl4 implements AccountService {

    @Autowired
    private IAccountDao accountDao;

    public void saveAccount() {
        accountDao.save();
    }

}
```

又由于此时容器中只有一个bean是它或者它的实现类，所以此时可以自动注入成功。

``` java
@Repository
public class AccountDaoImpl implements IAccountDao {
    public void save() {
        System.out.println("保存成功");
    }
}
```

1）当没有一个可以注入的对象时，则报错。

2）当有多个可以注入的对象时，且spring无法区分应该注入哪一个bean时，也会报错。

例如有两个用Repository标注的dao实现.

``` java
@Repository("accountDao1")
public class AccountDaoImpl implements IAccountDao {
    public void save() {
        System.out.println("保存成功1");
    }
}

@Repository("accountDao2")
public class AccountDaoImp2 implements IAccountDao {
    public void save() {
        System.out.println("保存成功2");
    }
}
```

此时如使用下面的注入：

``` java
@Autowired
private IAccountDao accountDao;
```

则无法注入成功，因为有两个符合条件的接口的实现类都可以注入进来。这是会报错。

如果修改一下变量名称和Repository标注的value相同，则又可以注入成功：

``` java
@Autowired
private IAccountDao accountDao1;
```

此时将注入AccountDaoImpl这个类的对象。

##### @Qualifier

作用：在按照类中注入的基础上再按照名称注入。它在给类成员注入时不能单独使用，但是在给方法注入的时候可以单独使用。

属性：value：用于指定bean的id。

``` java
@Autowired
@Qualifier(value = "accountDao1")
private AccountDaoImpl accountDao;
```

##### @Resource

作用：直接按照bean的id注入，它可以单独使用

属性：name：用于指定bean的id。

``` java
@Resource(name = "accountDao1")
private AccountDaoImpl accountDao;
```

**注意：**

@AutoWired、@Qualifier和@Resource三个注解都只能注入其他bean类型数据，而基本数据类型和String类型无法使用它们。

另外，集合类型的注入只能通过XML来实现。

##### @Value

作用：用于注入基本数据类型和String类型。

属性：value：数据的值。它可以使用spring中的Spring的EL表达式。



#### 3.用于指定作用范围的

##### @Scope

作用：指定bean的作用范围。

属性：value:singleton、prototype

``` java
@Component(value = "accountServiceImpl")
@Scope(value = "singleton")
public class AccountServiceImpl4 implements AccountService {
    //.....
}
```

#### 4.和生命周期相关的

##### @PostConstruct

对象被创建之后，执行的初始化方法。

##### @PreDestroy

对象被销毁时，执行的销毁方法。但是我们不能主动的去销毁一个对象，只能等虚拟机自己去销毁(多例时)。

``` java
@Service("accountService2")
@Scope(value = "prototype")
public class AccountServiceImpl2 implements AccountService {
    public AccountServiceImpl2(){
        System.out.println("对象被创建了");
    }

    @Resource(name = "accountDao2")
    private IAccountDao accountDao;

    public void saveAccount() {
        accountDao.save();
    }

    @PostConstruct
    public void init(){
        System.out.println("对象被初始化了了");
    }

    @PreDestroy
    public void destroy(){
        System.out.println("对象被销毁了");
    }

}
```

#### 注入jar文件中的类

在上面使用数据注入的时候，注入的数据都是我们自己写好的类，那该如何注入jar包中的类对象呢？

```xml
<bean id="runner" class="org.apache.commons.dbutils.QueryRunner">
    <constructor-arg name="ds" ref="dataSource"/>
</bean>
```

可以通过在xml中配置的方式，也可以通过注解：



### 配置类

用@Configuration标注，它可以实现bean.xml的所有功能。用它标注的类是一个配置类

#### @Configuration

作用：指定当前类是一个配置类。

#### @ComponentScan/@ComponentScans

作用：指定要扫描的包。

参数：basePackages或value。两个等价，用于指定创建容器时要扫描的包。

```java
@Configuration
@ComponentScan(basePackages = "com.spring.day02")
public class SpringConfiguration {
    
}
```

basePackeges指定的是一个数组类型的，因此多个的包的话，可以如下：

``` java
@Configuration
@ComponentScan(value = {"com.spring.day02","com.spring.day03"})
public class SpringConfiguration {

}
```

如果只有一个参数的时候，还可以省略掉basePackages或者value。

此时就可以把bean.xml中配置扫描的路径给去掉了：

``` xml
<!--告知spring容器创建时要扫描的包-->
<context:component-scan base-package="com.spring.day02"/>
```

然后使用`AnnotationConfigApplicationContext`注解配置的容器读取类：

##### AnnotationConfigApplicationContext

提供对Configuration配置类的容器读入。

参数：直接放入配置类的`类名.class`。可以放入多个class，用逗号分开。

注意：其实这个函数可以为指定的类默认配置一个@Configuration的注解，所以若采用这种方式激活容器，则对应的类可以把@Configuration省略掉。

``` java
ApplicationContext ac = new AnnotationConfigApplicationContext(SpringConfiguration.class);
```

但是由于此时使用的时候，使用的容器不是由bean.xml产生的，因此bean.xml中的QueryRunner以及DataSource都不能再被容器创建：

``` xml
<bean id="runner" class="org.apache.commons.dbutils.QueryRunner">
        <constructor-arg name="ds" ref="dataSource"/>
    </bean>

    <bean id="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource">
        <property name="driverClass" value="com.mysql.jdbc.Driver"/>
        <property name="jdbcUrl" value="jdbc:mysql://47.106.179.33:3306/test"/>
        <property name="user" value="root"/>
        <property name="password" value="1234"/>
    </bean>
```

解决办法：

在配置类中创建这两个实例。

首先思考如下创建方式：

``` java
@Configuration
@ComponentScan(value = "com.spring.day02")
public class SpringConfiguration {

    public QueryRunner queryRunner(){
        return new QueryRunner();
    }
}
```

这种方式确实产生了一个QueryRunner实例，但它不属于Spring容器，因此不能在容器中查找到，和上面的bean.xml创建的方式有很大的差别。

#### @Bean(方法上)

作用：把当前方法的返回值作为bean对象存入spring的ioc容器中。

属性：name：用于指定bean的id。当不写时，默认值是当前方法的名称。

注意：当用@Bean注解的方法具有参数的时候，当前的配置类容器中必须要有一个该类型的bean。不然就会报错。和@AutoWired的检查机制一致。

``` java
@Bean
public QueryRunner queryRunner(DataSource dataSource){
    return new QueryRunner(dataSource);
}

@Bean
public DataSource createDataSource(){
    ComboPooledDataSource ds = new ComboPooledDataSource();
    try {
        ds.setDriverClass("com.mysql.jdbc.Driver");
        ds.setJdbcUrl("jdbc:mysql://47.106.179.33:3306/test");
        ds.setUser("root");
        ds.setPassword("1234");
        return ds;
    } catch (PropertyVetoException e) {
        throw new RuntimeException(e);
    }
}
```

这样再通过：

``` java
ApplicationContext ac = new AnnotationConfigApplicationContext(SpringConfiguration.class);
AccountServiceImpl2 accountService = ac.getBean("accountService2", AccountServiceImpl2.class);
```

便可以成功激活容器并自动注入成功相应的对象。

#### @Scope

作用：指定bean对象的范围。

值：prototype（多例），默认是singleton（单例）。

``` java
@Bean(name = "runner")
@Scope("prototype")
public QueryRunner queryRunner(DataSource dataSource){
    return new QueryRunner(dataSource);
}
```

#### @Import

作用：用于在当前配置类导入另一个配置类（可以不用@Configuration标注），并且这个inport的配置类从属于当前配置类。

属性：value：指定其他配置类的字节码

例如：JdbcConfiguration.java

``` java
@Configuration
public class JdbcConfiguration {
    @Bean(name = "runner")
    @Scope("prototype")
    public QueryRunner queryRunner(DataSource dataSource){
        return new QueryRunner(dataSource);
    }

    @Bean("dataSource")
    public DataSource createDataSource(){
        ComboPooledDataSource ds = new ComboPooledDataSource();
        try {
            ds.setDriverClass("com.mysql.jdbc.Driver");
            ds.setJdbcUrl("jdbc:mysql://47.106.179.33:3306/test");
            ds.setUser("root");
            ds.setPassword("1234");
            return ds;
        } catch (PropertyVetoException e) {
            throw new RuntimeException(e);
        }
    }
}
```

SpringConfiguration.java

``` java
@Configuration
@Import(JdbcConfiguration.class)
@ComponentScan(value = "com.spring.day02")
public class SpringConfiguration {

}
```

使用@Import注解后，JdbcConfiguration从属于SpringConfiguration了。

-------------------

上面代码中的一个问题就是耦合的DataSource的配置，可以考虑使用properties文件进行抽离。

#### @PropertySource/@PropertySources

作用：用于指定properties文件位置

属性：value：指定文件的名称和路径。关键字：classpath：表示类路径下。

新建一个jdbcConfig.properties文件：

```properties
jdbc.Driver=com.mysql.jdbc.Driver
jdbc.Url=jdbc:mysql://47.106.179.33:3306/test
jdbc.User=root
jdbc.Password=1234
```
然后在需要使用properties文件的地方，使用@PropertySource标注。
``` java
@Configuration
@PropertySource("classpath:jdbcConfig.properties")
public class JdbcConfiguration {
}
```

然后使用Value注入。

完整的JdbcCOnfiguration.java类如下：

``` java

/**
 * 和连接数据库相关的类
 */
@Configuration
@PropertySource("classpath:jdbcConfig.properties")
public class JdbcConfiguration {
    @Value("${jdbc.Driver}")
    private String driver;
    @Value("${jdbc.Url}")
    private String url;
    @Value("${jdbc.User}")
    private String user;
    @Value("${jdbc.Password}")
    private String password;

    @Bean(name = "runner")
    @Scope("prototype")
    public QueryRunner queryRunner(DataSource dataSource){
        return new QueryRunner(dataSource);
    }

    @Bean("dataSource")
    public DataSource createDataSource(){
        ComboPooledDataSource ds = new ComboPooledDataSource();
        try {
            ds.setDriverClass(driver);
            ds.setJdbcUrl(url);
            ds.setUser(user);
            ds.setPassword(password);
            return ds;
        } catch (PropertyVetoException e) {
            throw new RuntimeException(e);
        }
    }
}
```



### Spring整合Junit

#### @Before

作用：每次在执行@Test的时候，都会先执行@Before标注的方法。

``` java
@Before
public void init(){
    ac = new AnnotationConfigApplicationContext(SpringConfiguration.class);
    as = ac.getBean("accountService2",AccountServiceImpl2.class);
}
```

不可取，不适合测试工程师。

##### 问题：

junit单元中虽没有main方法，但实际上jnuit集成了一个main方法。该方法会判断测试类中哪些方法有@Test注解，junit就会让有Test注解的方法执行。

但junit不知道用户使用了什么框架(spring)，所以运行@Test的时候就不会读取配置文件/配置类创建spring容器，所以在Test方法执行时，没有Ioc容器，就算写了AutoWired注解，也不起作用。

``` java
@Autowired
private ApplicationContext ac;//无效注解
```

可以使用Spring提供的测试单元：

1.引入spring-test

``` xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-test</artifactId>
    <version>5.1.6.RELEASE</version>
</dependency>
```

但是spring-test也是依赖于Junit的，并且要求junit测试单元的版本大于等于`4.12`.

2.使用Junit提供的一个注解把原有的main方法替换了，替换为spring提供的main。

3.告知spring的运行器，spring和ioc创建的是基于xml还是注解的，并且说明位置。

#### @RunWith

作用：替换原来Junit实现的main方法。

参数：`SpringJUnit4ClassRunner.class`

#### @ContextConfiguration

作用：告知spring和ioc创建的容器是基于xml，还是注解的。

属性：classes：SpringConfiguration.class（基于注解）。locations：“classpath:bean,xml”（基于xml）

``` java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = SpringConfiguration.class)
//或者
//@ContextConfiguration(locations = "classpath:bean.xml")
public class AccountServiceTest3 {
    
    @Autowired
    private IAccountService as = null;

    @Test
    public void TestFindAll() {
        List<Account> allAccount = as.findAllAccount();
        for (Account account : allAccount){
            System.out.println(account);
        }
    }
}
```





