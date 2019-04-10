## Spring（一）：Ioc,DI

### SPring概述

#### Spring是什么？

分层JavaSE/EE应用、轻量级开源框架、Ioc和AOP为核心。

提供：Spring MVC（展现层）、Spring JDBC（持久层）、业务层

兼容：众多第三方框架和类库

#### 耦合

程序间的依赖关系，包括：

* 类之间的依赖
* 方法之间的依赖

#### 解耦

降低程序之间的耦合性。但不可完全避免。

期望：`编译器不依赖，运行时依赖`

``` java
DriverManager.registerDriver(new com.mysql.jdbc.Driver());
Class.forName("com.mysql.jdbc.Driver");##字符串，如果缺少这个类，编译器无法检测出，程序可以编译成功，运行时才报错。
```

#### 解耦思路

第一步：使用反射创建对象，避免使用new。

2：通过读取配置文件来获取要创建的对象的全限定类名。

#### ->BeanFactory

可重用组件。

一般用于创建service和dao。

##### 1.配置文件  （key,value）形式。

.xml、.properties

bean.properties

``` bas
#accoutService=要创建的类的全名称。
accoutService=com.example.service.impl.AccoutServiceImpl
accountDao=com.exmple.dao.impl.AccoutDaoImpl
```

##### 2.读取配置文件，创建反射对象

用工程模式创建对象。

BeanFactory.java

``` java
public class BeanFactory{
    private static Properties prop;//定义一个Properties对象
    static{
        prop = new Properties();
        //获取Properties的文件流对象
        //InputStream in = new FileInputStream("文件路径");
        InputStream in = BeanFactory.class.getClassLoader().getResourceAsStream("bean.properties");
        prop.load(in);
    }
    
    /**
    *根据Bean名称获取Bean对象
    */
    public static Object getBean(String beanName){
        Object bean = null;
        try{
            String beanPath = prop.getProperty(beanName);
            bean = Class.forName(beanPath).newInstance();//每次都会调用默认构造函数创建新的对象。
        }catch(Exception e){
         	e.printStackTrace();   
        }
        return bean;
    }
    
}
```

这个Factory产生的对象是多例的。每次创建的都是一个独立的对象。

由于多例对象被创建多次，执行效率没有单例高。但是单例有线程安全问题。

##### 3.如何实现单例？

定义一个Map,用于存放BeanFactory产生的对象。——容器。

BeanFactory2.java

``` java
public class BeanFactory2{
    private static Properties prop;//定义一个Properties对象
    private static Map<String,Object> beans;
    static{
        prop = new Properties();
        //获取Properties的文件流对象
        //InputStream in = new FileInputStream("文件路径");
        InputStream in = BeanFactory.class.getClassLoader().getResourceAsStream("bean.properties");
        prop.load(in);
        beans = new HashMap<>();
        Enumeration keys = prop.keys();//枚举
        while(keys.hasMoreElements()){
            String key = keys.nextElement().toString();
            String beanPath = prop.getProperty(key);
            Object value = Class.forName(beanPath).newInstance();//初始化时会调用默认构造函数创建新的对象。
            beans.put(key,bean);//存储
        }
    }
    
    /**
    *根据Bean名称获取Bean对象
    */
    public static Object getBean(String beanName){
        return beans.get(beanName);
    }
    
}
```

#### 工厂模式获取实例与传统方式获取实例的对比：

``` java
A a = new A()
```

![1554871325561](D:\GitPro\TmpeImage\Spring\1554871325561.png)

``` java
A a = (A) BeanFactory.getBean("A");
```



![1554871370204](D:\GitPro\TmpeImage\Spring\1554871370204.png)

实现了控制反转（Ioc），也就是控制转移：把创建对象的权力交给了框架，是框架的重要特征。它包括依赖注入DI和依赖查找

明确Ioc的作用：`降低计算机程序之间的耦合`。

### Spring中的Ioc

配置一个普通工程，使用Maven的方式。创建好工程之后，引入Spring的东西：

1.引入依赖——spring-context

``` xml
<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>5.1.6.RELEASE</version>
    </dependency>
</dependencies>
```

这个里面包含了spring的核心基本内容：

![1554874601156](D:\GitPro\TmpeImage\Spring\1554874601156.png)

2.bean.xml

作用类似于一个工场，并且是单例模式的。

在resourcies目录下创建bean.xml：

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <!--把对象的创建交给Spring-->
    <bean id="accountService" class="com.spring.day01.service.impl.AccountServiceImpl"/>
    <bean id="accountDao" class="com.spring.day01.dao.impl.AccountDaoImpl"/>
</beans>
```

这样就让spring来控制AccountServiceImpl和AccountDaoImpl实例的创建。

3.使用

``` java
//        AccountService accountService = new AccountServiceImpl();
        //1.获取核心容器
        ApplicationContext ac = new ClassPathXmlApplicationContext("bean.xml");
        //2.根据Id获取bean对象——方式一
        AccountService accountService = (AccountService) ac.getBean("accountService");
        //方式二：
        AccountService as = ac.getBean("accountService",AccountService.class);
        accountService.saveAccount();
        System.out.println(accountService);
        System.out.println(as);
```

首先要获取核心容器对象，然后有两种获取bean对象的方式。通过测试发现产生的是**单例**：

``` java
保存成功
com.spring.day01.service.impl.AccountServiceImpl@5b275dab
com.spring.day01.service.impl.AccountServiceImpl@5b275dab
```

#### ApplicationContext

这也是一个接口，并且它还继承于多个接口。

![1554875097659](D:\GitPro\TmpeImage\Spring\1554875097659.png)

再来看它的实现：

![1554875156096](D:\GitPro\TmpeImage\Spring\1554875156096.png)

刚才用的就是`ClassPathXmlApplicationContext`这个。

常用的还有：`AnnotationConfigApplicationContext`和`FileSystemXmlApplicationContext`

* ClassPathXmlApplicationContext：加载类路径下的配置文件，要求配置文件必须在类路径下。
* AnnotationConfigApplicationContext：可以加载磁盘任意路径下的配置文件（需要具有访问权限）。
* FileSystemXmlApplicationContext：用于读取注解创建容器。

ApplicationContext在创建核心容器的时候，创建对象的策略是**立即加载**，只要一读取配置文件就立刻创建配置文件中的所有bean，然后缓存。即声明ApplicationContext ac = new ClassPathXmlApplicationContext("bean.xml");时候就创建对象了。

#### BeanFactory

BeanFactory是ApplicationContext的父接口。它也可以创建对象。但是采取的策略是与ApplicationContext截然不同的策略——**延迟加载**。即用到这个对象的时候才加载。

``` java
Resource resource = new ClassPathResource("bean.xml");
BeanFactory factory = new XmlBeanFactory(resource);
AccountService accountService = (AccountService) factory.getBean("accountService");
```

在`AccountService accountService = (AccountService) factory.getBean("accountService");`时才真正创建对象。

它也是**单例模式**。

创建时间点不相同。

## Spring对Bean管理的细节

### 创建bean的三种方式

#### 1.使用无参默认构造器创建

``` xml
<bean id="accountService" class="com.spring.day01.service.impl.AccountServiceImpl"/>
```

AccountServiceImpl中必须有一个对应的默认构造器，否咋就会出错。所以这种方式要求要创建对象的类必须有默认构造器。

#### 2.使用Factory的方式

假设有一个Factory类，并且无法修改其中的源码。如果要想获得工厂中的方法，那么就必须使用第二种bean方式.

``` java
public class InstanceFactory {
	//工厂中的普通方法，需要通过工厂对象来调用
    public AccountService getAccountService(){
        return new AccountServiceImpl();
    }
}
```

创建bean的方式为：

``` xml
<!--工厂:使用某个类中的方法创建对象-->
    <bean id="instanceFactory" class="com.spring.day01.factory.InstanceFactory"/>
    <bean id="accountService2" factory-bean="instanceFactory" factory-method="getAccountService"/>
```



#### 3.使用工厂中的静态方法创建对象

``` java
public class InstanceFactory {
    //工厂中的静态方法，需要使用工厂名字来调用
    public static IAccountDao getAccoutDao(){
        return new AccountDaoImpl();
    }
}
```

创建bean的方式为：

``` xml
<!--静态工厂方法-->
<bean id="accountDao2" class="com.spring.day01.factory.InstanceFactory" factory-method="getAccoutDao"/>
```

### Bean的作用范围

Spring使用bean创建对象的时候，默认是单例的。如果不需要单例，则可以使用scope属性进行调整。

scope的取值有：

* singleton (默认)：单例
* prototype ：多例
* request ：作用于web应用的请求范围
* session：作用于web应用的会话范围
* global-session：作用于集群环境的会话范围，当不是集群的时候，就是session。

可以简单理解为session对应于一台服务器，global-session对应于所有集群共享的session。

![1554879631162](D:\GitPro\TmpeImage\Spring\1554879631162.png)

### Bean对象的生命周期

#### 单例对象

出生：当容器创建时对象出生  
或者：只要容器还在，对象存活  
死亡：容器销毁，对象消亡  
总结：**生命周期同容器**  

实验：为AccountServiceImpl对象添加两个方法：

``` java
public class AccountServiceImpl implements AccountService {
    public void saveAccount() {
        IAccountDao accountDao = new AccountDaoImpl();
        accountDao.save();
    }
    
    public void init(){
        System.out.println("对象被创建了");
    }
    
    public void destroy(){
        System.out.println("对象被销毁了");
    }
}
```

在bean.xml中绑定这两个方法：

``` xml
<bean id="accountService" class="com.spring.day01.service.impl.AccountServiceImpl"
    init-method="init" destroy-method="destroy"/>
```

表示在这个对象被创建的时候就会调用init方法，在对象被销毁的时候，调用detroy方法。

``` java
ClassPathXmlApplicationContext ac = new ClassPathXmlApplicationContext("bean.xml");
AccountService accountService = (AccountService) ac.getBean("accountService");
ac.close();
```

注意，ApplicationContext没有close方法，它的实现类才有。

``` java
对象被创建了
对象被销毁了
```

#### 多例对象

出生：当使用对象时，spring框架才创建。  
活着：对象在使用过程中就一直活着。  
死亡：当对象没有其他对象引用且长时间不使用的时候  
总结：和普通new出来的对象差不多。

## Spring依赖注入

### 什么是依赖注入？

依赖关系的维护就叫做依赖注入。而依赖关系是指类之间的耦合关系，依赖关系的管理有spring提供。依赖注入能注入的数据类型分为了三类：

* 基本类型（包括包装类）和String类型
* 其他bean类型。也就是其它类对象。例如Date，或者自己定义的类。
* 复杂类型/集合类型

### 注入的方式

#### 使用构造函数

创建一个StudentServiceImpl类，它只有一个三个参数的构造器。

``` java
public class StudentServiceImpl implements StudentService {
    private String name;
    private Integer age;
    private Date birthday;

    public StudentServiceImpl(String name, Integer age, Date birthday) {
        this.name = name;
        this.age = age;
        this.birthday = birthday;
    }

    public void saveStudent() {
        System.out.println("学生被存储了,学生信息为："+name+",年零："+age+",生日："+birthday);
    }
}
```

如果是需要经常变化的数据，不适合依赖注入。

在bean.xml注册的时候，需要使用标签：`constructor-arg`。

这个标签有5个可选参数（每次至少用到两个）：

三个用于指定参数：

* name：指定构造器中参数的名字
* type：指定构造器中该种类型所对应的参数（适用于只有一种这个参数类型）
* index:指定第i个参数。参数的索引从0开始。

两个用于给参数赋值：

* value：可以赋基本类型和String类型
* ref：bean文件中声明的另一个bean

``` xml
<bean id="studentService" class="com.spring.day01.service.impl.StudentServiceImpl">
    <constructor-arg type="java.lang.String" value="小米"/>
    <constructor-arg name="age" value="18"/>
    <constructor-arg index="2" ref="now"/>
</bean>

<bean id="now" class="java.util.Date"/>
```

用的最多的应该是name+value方式。

#### setter方法注入（更常用）

StudentServiceImpl2有属性，但是没有带参构造器，只能通过先创建对象，然后调用setter方法赋值。

``` java
public class StudentServiceImpl2 implements StudentService {
    private String name;
    private Integer age;
    private Date birthday;

    public void saveStudent() {
        System.out.println("学生被存储了,学生信息为："+name+",年零："+age+",生日："+birthday);
    }

    public void setName(String name) {
        this.name = name;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    public void setBirthday(Date birthday) {
        this.birthday = birthday;
    }
}
```

在bean.xml中可以通过`<property>`标签进行调用setter方法，注入参数。

``` xml
<!--依赖注入-setter方式-->
<bean id="studentService2" class="com.spring.day01.service.impl.StudentServiceImpl2">
    <property name="age" value="18"/>
    <property name="name" value="Alice"/>
    <property name="birthday" ref="now"/>
</bean>
```

property只能使用name的方式索引到具体参数。

#### 复杂类型/集合类型的注入

StudentServiceImpl3是一个具有众多复杂类型的属性的类：

``` java
public class StudentServiceImpl3 implements StudentService {
    private String[] myStrs;
    private List<String> myList;
    private Set<String> mySet;
    private Map<String,String> myMap;
    private Properties properties;

    public void saveStudent() {
        System.out.println(Arrays.toString(myStrs));
        System.out.println(myList);
        System.out.println(mySet);
        System.out.println(myMap);
        System.out.println(properties);
    }

    public void setMyStrs(String[] myStrs) {
        this.myStrs = myStrs;
    }

    public void setMyList(List<String> myList) {
        this.myList = myList;
    }

    public void setMySet(Set<String> mySet) {
        this.mySet = mySet;
    }

    public void setMyMap(Map<String, String> myMap) {
        this.myMap = myMap;
    }

    public void setProperties(Properties properties) {
        this.properties = properties;
    }
}
```

它的注入方式为：

``` xml
<!--复杂类型/集合类型注入-->
<bean id="studentService3" class="com.spring.day01.service.impl.StudentServiceImpl3">
    <property name="myStrs">
        <array>
            <value>AAA</value>
            <value>BBB</value>
            <value>CCC</value>
        </array>
    </property>

    <property name="myList">
        <list>
            <value>AAA</value>
            <value>BBB</value>
            <value>CCC</value>
        </list>
    </property>

    <property name="mySet">
        <set>
            <value>AAA</value>
            <value>BBB</value>
            <value>CCC</value>
        </set>
    </property>

    <property name="myMap">
        <map>
            <entry key="aaa" value="AAA"/>
            <entry key="bbb" value="BBB" />
        </map>
    </property>
    
    <property name="properties">
        <props>
            <prop key="aaa">AAA</prop>
            <prop key="bbb">BBB</prop>
        </props>
    </property>
</bean>
```

需要注意的是，实际上复杂类型/集合类型在注入时数据类型只有两类：

* array/list/set 三种标签可以互换，不会产生影响
* map/props 两种标签也可以呼唤，不会产生影响











