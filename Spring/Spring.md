![image-20231116210422235](typora文档图片/image-20231116210422235.png)

## IOC/DI

IOC：Inversion of control，控制反转。

- 在类的内部使用对象时，由**主动**`new`产生对象**转换为由外部提供**对象，此过程中**对象创建控制权**由程序转移到外部，此思想称为**控制反转**。

Spring技术实现IoC思想：

- Spring提供了一个容器，称为**IoC容器**，用来充当IoC思想中的**外部**；
- **IoC容器**负责对象的**创建、初始化**等一系列工作，被创建或被管理的**对象**在IoC容器中统称为**Bean**。

DI：Dependency Injection，依赖注入。

- 在**容器中**建立**bean与bean之间的依赖关系**的整个过程，称为**依赖注入**。

**IoC本质**：将自身对象中的一个内置对象的控制反转，反转后不再由自己本身的对象进行控制这个内置对象的创建，而是由第三方系统去控制这个内置对象的创建。简单来说就是把**本来在类内部控制的对象**，**反转到类外部进行创建后注入**，不在由类本身镜像控制，这就是IoC的本质。

**DI**：**自身对象中的内置对象**是通过**注入的方式**进行**创建**。

**IoC和DI的关系**：IoC就是容器，DI就是注入这一行为，那么DI确实就是IoC的具体功能的实现。而IoC则是DI发挥的平台和空间。所以说。IoC和DI即是相辅相成的拍档。他们都是为了实现解耦而服务的

### **入门案例**

#### IoC

1、创建spring项目，然后在`pom.xml`文件中添加`Spring`依赖：

```xml
<dependency>
	<groupId>org.springframework</groupId>
	<artifactId>spring-context</artifactId>
	<version>5.2.10.RELEASE</version> 
</dependency>
```

版本自定。

2、添加案例中需要的类

创建`BookService`,`BookServiceImpl`，`BookDao`和`BookDaoImpl`四个类

```java
public interface BookDao {
    public void save();
}
public class BookDaoImpl implements BookDao {
    public void save() {
        System.out.println("book dao save ...");
    }
}
public interface BookService {
    public void save();
}
public class BookServiceImpl implements BookService {
    private BookDao bookDao = new BookDaoImpl();
    public void save() {
        System.out.println("book service save ...");
        bookDao.save();
    }
}
```

3、添加spring配置文件

resources下添加spring配置文件applicationContext.xml，并完成bean的配置

![1629734336440](typora文档图片/1629734336440.png)

4、在配置文件中完成bean的配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
<!--    1、导入spring坐标-->
<!--    2、配置bean-->
<!--
    id 表示bean的名字
    class表示bean类型，即所属类
-->
    <bean id="bookDap" class="com.lxb.spring.dao.impl.BookDaoImpl"/>
    <bean id="bookService" class="com.lxb.spring.service.impl.BookServiceImpl"/>

</beans>
```

**==注意事项：bean定义时id属性在同一个上下文中(配置文件)不能重复==**

5、获取IOC容器

使用Spring提供的接口完成IoC容器的创建，创建App类，编写main方法

```java
public class App {
    public static void main(String[] args) {
        //获取IOC容器
		ClassPathXmlApplicationContext ctx = new ClassPathXmlApplicationContext("applicationContext.xml"); 
    }
}
```

6、从容器中获取对象进行方法调用

```java
public class App {
    public static void main(String[] args) {
        //获取IOC容器
		ApplicationContext ctx = new ClassPathXmlApplicationContext("applicationContext.xml"); 
//        BookDao bookDao = (BookDao) ctx.getBean("bookDao");
//        bookDao.save();
        BookService bookService = (BookService) ctx.getBean("bookService");
        bookService.save();
    }
}
```

#### DI

上面的控制反转只是将对象的创建的过程交给了spring容器，但是如果**需要创建的对象内部也有对象**，那么内部的对象还是只能由spring创建的对象自己在内部创建，所以为了将**内部的对象也交给spring创建**，提出了**DI**，即Dependency Injection，依赖注入。

`Spring`创建的对象是通过`bean`绑定，所以如果`bean`所指代的对象内部也有对象需要创建，就在`bean`标签里通过`dependency`创建，`dependency`绑定的是`bean`对象的`Setter`方法，可以自动创建对象并并传递给`bean`对象的`Setter`方法。

**代码**

```xml
    <bean id="bookService" class="com.lxb.spring.service.impl.BookServiceImpl">
    <!--
        为bean配置属性（即成员变量）：property表示一个属性
         name：属性里面的引用类型，即类里面对应的变量名
         ref：属性具体对应的bean
    -->
        <property name="xxxxx" ref="bookDao"/>
    </bean>
```

`bean`代表的**对象里面的属性名**对应`dependency`的`name`，`ref`表示该属性对应的`bean`（因为对象创建都由spring管理，所以这个`dependency`对应的对象也是一个`bean`，只不过这个对象是另一个`bean`的成员变量，需要通过依赖注入的方式创建）。

对应的类需要有`Setter`方法：

```java
public void setXxxxx(BookDao bookDao) { 
    this.xxxxx = bookDao;
}
```

### 个人理解

项目开发时，类里面的属性包含对象，所以会在类里面创建对象然后再使用，当类多了之后，互相之间联系密切，就会出现“牵一发而动全身”的问题。

为了解决这个问题，可以使用**控制反转（IoC）思想**，即将类的创建、初始化等操作**交给Spring容器**来完成，类只需要调用spring容器接收创建好的对象去干自己的事就行了。类的创建、初始化等操作由类在内部实现变成由Spring容器实现，即对类的控制权发生了变化，所以叫：控制反转。

#### IoC容器

首先在pom配置文件里面添加Spring依赖，然后创建Spring配置文件，通过bean绑定需要控制的对象。在对应的类里使用Spring来进行对象创建。

#### DI

Spring用`bean`绑定对象，实现创建、初始化等操作。但是如果类的成员变量也是对象，就需要用DI（依赖注入）来完成。即在对应的`bean`里面添加`dependency`标签，绑定对应的成员变量，这样就可以通过Setter方法实现由spring创建对象成员变量。

#### Spring配置文件解读

被spring管理的对象统称为`bean`；通过Spring配置文件告诉Spring容器需要管理哪些对象（即哪些bean）。

##### 书写规则

标签/元素定义：

```xml
<label> </label>
// 如果标签内部为空，也可以直接写成单标签
<label/>
// 举例
<bean></bean>
<bean/>
```

##### bean元素

用来定义一个bean对象，指代需要控制的对象。

```xml
<bean id="bean唯一标识" name="bean名称" class="类的完整名称" factory-bean="工厂bean名称" factory-method="工厂方法"/>
```

**id**：bean的名称，在一个spring容器中必须唯一，通过bean名称可以从spring容器获取对应的bean对象。`context.getBeanDefinitionNames();` 获取所有bean的id名，以`String[]`数组形式返回

**name**：别名，即外号，也可以用来从spring容器获取对应的bean对象。`context.getAliases(id);`根据id名获取对应bean的所有别名

**bean名称别名定义规则**

1. 当**id存在**的时候，不管name有没有，**取id为bean的名称**；

2. 当**id不存在**，此时需要**看name**，name的值可以通过“,”或者“;”或者“ ”分割，最后会按照分隔符得到一个String数组，数组的**第一个元素作为bean的名称**，**其他**的作为bean的**别名**；

3. 当id和name**都存在**的时候，id为bean名称，name用来定义多个别名；

4. 当id和name**都不指定**的时候，bean名称**自动生成**，格式：bean的class的完整类名#编号

    上面的编号是从0开始的，同种类型的没有指定名称的依次递增。示例：`com.lxb.spring.service.impl.BookServiceImpl#0`。

###### property子元素

元素的子元素，用于调用 Bean **实例中的 setter 方法**对**属性进行赋值**，从而完成属性的注入。该元素的 `name` 属性用于指定 Bean 实例中**相应的属性名**。

```xml
<property name="xxxxx" ref="bookDao"/>
```

**name**：表示该依赖是bean对象的哪个成员变量。

**ref**：该依赖是哪个`bean`，可以用对应`bean`的`id`或者`name`绑定。

###### scope属性

表示`bean`的作用范围：

- singleton：单例（默认）；
- prototype：非单例；

即创建bean对象时，是单例对象还是非单例对象。

**验证方法**

```java
// 获取同一个bean对象两次，看两个对象是否是同一个
BookService bookDao1 = (BookService) context.getBean("bookService");
BookService bookDao2 = (BookService) context.getBean("bookService");       System.out.println(bookDao2);
System.out.println(bookDao1);
```

###### scope属性

**为什么`bean`默认单例？**

单例的意思是在Spring的IoC容器中只会有该类的一个对象，避免了对象的频繁创建与销毁，达到了bean对象的复用，性能高。

**单例会不会产生线程安全问题？**

如果对象是有状态对象，即该对象有成员变量可以用来存储数据的，因为所有请求线程共用一个bean对象，所以会存在线程安全问题。

如果对象是无状态对象，即该对象没有成员变量没有进行数据存储的，因方法中的局部变量在方法调用完成后会被销毁，所以不会存在线程安全问题。

**哪些bean对象适合交给容器进行管理?**

表现层对象、业务层对象、数据层对象、工具对象

**哪些bean对象不适合交给容器进行管理?**

封装实例的域对象，因为会引发线程安全问题，所以不适合。

###### init-method属性



###### destroy-method属性



##### alias元素

```xml
<alias name="需要定义别名的bean的id" alias="别名" />
```

即根据name给对应的bean设置别名alias。name可以是id或者bean里的name，即别名。

##### import元素

通过import元素引入其他bean配置文件。

```xml
<import resource="applicationContext.xml"/>
```

#### IoC实例化方式

Spring容器是如何通过bean完成对象创建的。

三种方式：通过构造方法来实例化、通过静态工厂实例化、通过实例工厂和FactoryBean

- **构造方法实例化**

    通过Spring容器中的`bean`创建对象的时候，会自动**调用类的无参构造方法**进行对象创建，如果重写了构造方法而没有重写无参构造方法则会报错。

- **静态工厂实例化**

    什么是静态工厂实例化？

    即在工厂类中配置一个静态方法来创建对象并返回。

    ```java
    public class OrderDaoFactory {
        public static OrderDao getOrderDao() {
            return new OrderDaoImpl();
        }
    }
    ```

    如何交给Spring管理？

    1. 在spring的配置文件xml中添加以下内容:

        ```xml
        <bean id="orderDao" class="com.itheima.factory.OrderDaoFactory" factory-method="getOrderDao"/>
        ```

        `class`:工厂类的类全名；`factory-mehod`:具体工厂类中创建对象的方法名；

    2. 在运行类中使用IoC容器创建对应的bean对象。（与前面类似）

    使用静态工厂实例化的**好处**：可以在创建对象的时候在静态方法中添加其他的业务操作。

- **FactoryBean**

    实例工厂实例化：即先创建工厂的对象，然后用工厂的对象执行方法，该方法返回需要创建的对象。

    ```java
    // 工厂类
    public class UserDaoFactory {
        public UserDao getUserDao() {
            return new UserDaoImpl();
        }
    }
    // 创建工厂类对象 
    UserDaoFactory userDaoFactory = new UserDaoFactory();
    // 调用工厂类的方法构造对象并返回
    UserDao userDao = userDaoFactory.getUserDao();
    userDao.save();
    ```
    
    如何将上述实例化过程交给Spring实现？
    
    1. spring配置文件。
    
        先将工厂类绑定`bean`（因为要创建工厂类对象），然后将要创建的对象也绑定`bean`，通过`factory-bean`属性指明工厂类的实例对象，`factory-method`为工厂类中创建对象的方法。
    
        ```xml
        <bean id="userFactory" class="com.lxb.spring.factory.UserDaoFactory"/>
        <bean id="userDao" factory-method="getUserDao" factory-bean="userFactory"/> 
        ```
    
    2. 在运行类app中获取对象。
    
        ```java
        UserDao userDao = (UserDao) context.getBean("userDao");
        userDao.save();
        ```
    
    **简化方法**：直接使用`FactoryBean`。
    
    1. 创建`UserDaoFactoryBean`类，实现`FactoryBean`接口，并重写接口的方法：
    
        ```java
        public class UserDaoFactoryBean implements FactoryBean<UserDao> {
            // 代替原始实例工厂中创建对象的方法
            @Override
            public UserDao getObject() throws Exception {
                return new UserDaoImpl();
            }
        
            // 返回所创建类的Class对象
            @Override
            public Class<?> getObjectType() {
                return UserDao.class;
            }
        
        }
        ```
    
    2. 配置Spring文件：
    
        ```xml
        <bean id="userDao" class="com.lxb.spring.factoryBean.UserDaoFactoryBean"/>
        ```
    
    3. 在app运行类中直接根据id获取该FactoryBean的对象，会自动返回需要创建的对象。
    
    
    
    **FactoryBean**
    
    - 源码
    
        ```java
        public interface FactoryBean<T> {
            String OBJECT_TYPE_ATTRIBUTE = "factoryBeanObjectType";
        
            @Nullable
            T getObject() throws Exception;
        
            @Nullable
            Class<?> getObjectType();
        
            default boolean isSingleton() {
                return true;
            }
        }
        ```
    
        `getObject`：重写用于**在方法中创建对象并返回**；
    
        `getObjectType`：返回被创建的类的`class`；
    
        `isSingleton`：默认为单例，可以重写设置为非单例，即返回`false`。
    
    - 

#### bean的声明周期

##### 手动提供

设计测试类，分别添加两个方法：

```java
public class BookDaoImpl implements BookDao {
    public BookDaoImpl() {
        System.out.println("构造函数执行.....");
    }

    @Override
    public void save() {
        System.out.println("book dao.....");
    }

    public void init() {
        System.out.println("init...");
    }

    public void destroy() {
        System.out.println("destroy...");
    }
}
```

在spring配置文件中配置：

```xml
<bean id="bookDao" class="com.lxb.spring.dao.impl.BookDaoImpl" init-method="init" destroy-method="destroy"/>
```

在app运行类中根据bean获取对象，并执行save方法：

```java
ClassPathXmlApplicationContext context =new ClassPathXmlApplicationContext("applicationContext.xml");
BookDao bookDao = (BookDao) context.getBean("bookDao");
bookDao.save();
```

输出结果：

```tex
构造函数执行.....
init...
book dao.....
```

**结果分析**

`init`方法执行了，但是`destroy`方法却未执行。原因：

- Spring的IoC容器是运行在JVM中；
- 运行 main 方法后，JVM启动，Spring加载配置文件生成 IoC 容器，从容器获取bean对象，然后调方法执行
- main 方法执行完后，JVM退出，这个时候 IoC 容器中的 bean 还没有来得及销毁就已经结束了，所以没有调用对应的destroy方法；



**close关闭容器**

在main方法执行完成之前通过手动调用`close`关闭IoC容器，这样就会执行destroy方法：

```java
BookDao bookDao = (BookDao) context.getBean("bookDao");
bookDao.save();
context.close();

// 输出
构造函数执行.....
init...
book dao.....
destroy...
```

注意：IoC容器关闭后就无法再获取bean对象了。

**注册钩子关闭容器**

在容器未关闭之前，**提前设置好回调函数**，让JVM在退出之前**回调此函数来关闭容器**。

钩子注册的方法：`registerShutdownHook()`。

```java
BookDao bookDao = (BookDao) context.getBean("bookDao");
bookDao.save();
context.registerShutdownHook();

// 输出

构造函数执行.....
init...
book dao.....
destroy...
```

**二者关系**

- 相同点：都能关闭IoC容器。
- 不同点：`close`()是在**调用的时候**关闭，`registerShutdownHook`()是**在JVM退出前**调用关闭。 

##### 使用Spring提供的接口

创建的类：

```java
public class BookDaoImpl implements BookDao, InitializingBean, DisposableBean {
    @Override
    public void save() {
        System.out.println("save...");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("init...");
    }

    @Override
    public void destroy() throws Exception {
        System.out.println("destroy...");
    }
}
```

spring配置文件：

```xml
<bean id="bookDao" class="com.lxb.spring.dao.impl.BookDaoImpl" />
```

输出：

```java
init...
save...
destroy...
```

通过使用Spring提供的接口可以不需要在类里面重新设计启动和销毁方法，而是直接继承两个接口然后使用对应启动和销毁方法即可。

注意启动方法`afterPropertiesSet`：实在属性设置完成之后执行，即在setter方法将属性赋值之后再执行（当然该setter方法也在spring里面进行了绑定，用来创建对象属性）。

##### 总结

bean的生命周期控制在bean的整个生命周期中所处的位置如下：

- 初始化容器
    1. 创建对象(内存分配) 
    2. 执行构造方法 
    3. 执行属性注入(**set操作**) 
    4. 执行**bean初始化**方法：afterPropertiesSet
- 使用bean
    - 执行业务操作
- 关闭/销毁容器
    - 执行bean销毁方法

**关闭容器的两种方式**： `close`()方法 `registerShutdownHook`()方法。

#### 依赖注入——DI

dependency injection。

**依赖注入**描述了在容器中**建立bean与bean之间的依赖关系**的过程。

**注入方式**

- **setter注入**：适用于简单类型和对象引用类型；
- **构造器注入**：适用于简单类型和引用类型

##### setter注入

在`bean`标签内进行注入：

- 对于引用数据类型使用的是：`<property name="" ref=""/>` 或者 `dependency< name="" ref=""/>`；
- 对于简单数据类型使用的是：`<property name="" value=""/>`；

###### 注入引用类型

对类里面的属性，提供可以访问的setter方法由IoC调用：

```java
public class BookServiceImpl implements BookService {
    private BookDao bookDao;
    @Override
    public void save() {
        System.out.println("book service.....");
        bookDao.save();
    }

    public void setBookDao(BookDao bookDao) {
        this.bookDao = bookDao;
    }
}
```

配置spring文件：

- 类的属性对应的类也要绑定bean；
- 类的属性是引用类型，使用`dependency`标签绑定，`ref`属性注入引用类型的对象对应的bean，`name`属性对应类中的属性名

```xml
    <bean id="bookDao" class="com.lxb.spring.dao.impl.BookDaoImpl" />
    <bean id="bookService" class="com.lxb.spring.service.impl.BookServiceImpl">
    <!--
        为bean配置属性（即成员变量）：property表示一个属性
         name：bean的id或者别名，表示对应的属性，即类里面对应的变量名，
         ref：属性具体对应的bean
    -->
        <property name="demoName" ref="bookDao"/>
    </bean>
```

###### 注入简单类型

设置类的属性：

```java
public class BookServiceImpl implements BookService {
    private String name;
    private int age;
    
    @Override
    public void save() {
        System.out.println(name + " is " + age + " years.");
    }

    public void setName(String name) {
        this.name = name;
    }

    public void setAge(int age) {
        this.age = age;
    }
}
```

使用`property`标签在`bean`内部进行简单类型的注入：

```xml
    <bean id="bookService" class="com.lxb.spring.service.impl.BookServiceImpl">
        <property name="name" value="apple"/>
        <property name="age" value="12"/>
    </bean>
```

##### 构造器注入

- 简单数据类型：

    ```xml
    <bean>
    <constructor-arg name="" value=""/> 
    <constructor-arg type="" value=""/>
    <constructor-arg index="" value=""/>
    </bean>
    ```

- 引用数据类型：

    ```xml
    <bean>
    <constructor-arg name="" ref=""/>
    <constructor-arg type="" ref=""/>
    <constructor-arg index="" ref=""/>
    </bean>
    ```

###### 注入引用类型

类里面需要添加相应的构造方法对引用变量进行赋值，参数是引用变量：

```java
public class BookServiceImpl implements BookService {
    private BookDao bookDao;

    public BookServiceImpl(BookDao bookDao) {
        this.bookDao = bookDao;
    }

    @Override
    public void save() {
        System.out.println("service save");
        bookDao.save();
    }
}
```

spring配置文件：

使用`constructor-arg`标签绑定构造方法：

- `name`属性对应的值为**构造方法中形参的参数名**，必须要保持一致；
- `ref`属性指向的是spring的IOC容器中其他bean对象；

```xml
<bean id="bookDao" class="com.lxb.spring.dao.impl.BookDaoImpl" />
<bean id="bookService" class="com.lxb.spring.service.impl.BookServiceImpl">
        <constructor-arg name="bookDao" ref="bookDao"/>
    </bean>
```

多个属性都是引用类型变量就用多个`constructor-arg`标签，同时要在构造方法中同时传入。

###### 注入简单类型

类中添加简单的构造方法：

```java
public class BookServiceImpl implements BookService {

    private String name;
    private int age;

    public BookServiceImpl(String name, int age) {
        this.name = name;
        this.age = age;
    }

    @Override
    public void save() {
        System.out.println(name + " is " + age + " years.");
    }
}
```

spring配置：

```xml
    <bean id="bookService" class="com.lxb.spring.service.impl.BookServiceImpl">
        <constructor-arg name="name" value="apple"/>
        <constructor-arg name="age" value="1"/>
    </bean>
```

**问题**：

上面的配置文件对应的是构造方法，但是当构造方法中的参数名发生变化之后，就需要更改有关的配置文件，导致“紧耦合”，怎么解决?

1. **删除**`name`属性，**添加**`type`属性，按照类型注入。

    即将上面的name属性改成type属性，然后填入类的属性对应的类型。

2. **删除**`type`属性，**添加**`index`属性，按照索引下标注入，下标从0开始。

    但是如果构造方法参数顺序发生变化后，这种方式又带来了耦合。

##### 注入方式选择

待定

##### 自动配置

前面的方式需要反复的更改配置文件，麻烦。

**依赖自动装配**：IoC容器根据`bean`**所依赖的资源**在容器中**自动查找并注入到bean中**的过程称为自动装配。

**自动装配方式**

- 按类型（常用） 
- 按名称 
- 按构造方法 
- 不启用自动装配

###### 实现

**按照类型注入**

类定义：

```java
public class BookServiceImpl implements BookService {

    private BookDao  bookDao;

    public void setBookDao(BookDao bookDao) {
        this.bookDao = bookDao;
    }

    @Override
    public void save() {
        System.out.println("service save...");
        bookDao.save();
    }
}
```

配置文件：

```xml
    <bean class="com.lxb.spring.dao.impl.BookDaoImpl"/>
    <bean id="bookService" class="com.lxb.spring.service.impl.BookServiceImpl" autowire="byType"></bean>
```

**按照名称注入**

一个类型在IOC中有多个对象，还想要注入成功，这个时候就需要按照名称注入。

类定义同上。

配置文件：

```xml
    <bean id="bookDao" class="com.lxb.spring.dao.impl.BookDaoImpl"/>
    <bean id="bookService" class="com.lxb.spring.service.impl.BookServiceImpl" autowire="byName"></bean>
```

IoC会自动查找到bean的id和类的属性（即类里面要属性名）相同的bean，然后进行注入。根据setter识别，因为setter默认格式是：set+属性名首字母大写

找不到则注入`null`。

**注意**

- 需要注入属性的类中对应属性的`setter`**方法不能省略**；而且得按默认格式：set+属性名首字母大写；
- **被注入的对象**必须要被Spring的IoC容器管理；
- 按照类型如果找到多个对象，会报`NoUniqueBeanDefinitionException`；

关于自动装配

- 自动装配**用于引用类型**依赖注入，**不能对简单类型**进行操作；
-  使用**按类型装配时**（`byType`）必须保障容器中**相同类型的bean**唯一，推荐使用
- 使用**按名称装配时**（`byName`）必须保障容器中**具有指定名称的bean**，因变量名与配置耦合，不推荐使用 
- **自动装配优先级低于setter注入与构造器注入**，同时出现时自动装配配置失效；

##### 集合注入

**即类里面的属性是集合**。

常见集合：数组、List、Map、Set、Properties。

###### 实现

类：

```java
package com.lxb.spring.service.impl;

import com.lxb.spring.dao.BookDao;
import com.lxb.spring.dao.impl.BookDaoImpl;
import com.lxb.spring.service.BookService;

import java.util.*;

/**
 * @Author: Daniel
 * @Description:
 * @Date: 2023/11/16 22:17
 * @Email: lxb2000m@gmail.com
 */

public class BookServiceImpl implements BookService {

    private int[] nums;
    private List<String> list;
    private Set<String> set;
    private Map<String, String> map;
    private Properties properties;

    public void setNums(int[] nums) {
        this.nums = nums;
    }

    public void setList(List<String> list) {
        this.list = list;
    }

    public void setSet(Set<String> set) {
        this.set = set;
    }

    public void setMap(Map<String, String> map) {
        this.map = map;
    }

    public void setProperties(Properties properties) {
        this.properties = properties;
    }

    @Override
    public void save() {
        System.out.println("service ...");
        System.out.println(Arrays.toString(nums));
        System.out.println(list);
        System.out.println(set);
        System.out.println(map);
        System.out.println(properties);
    }
}
```

配置文件：

```xml
    <bean id="bookService" class="com.lxb.spring.service.impl.BookServiceImpl">
        <property name="nums">
            <array>
                <value>12</value>
                <value>12</value>
                <value>12</value>
            </array>
        </property>
        <property name="list">
            <list>
                <value>asdasd</value>
                <value>aaa</value>
                <value>asdasd</value>
            </list>
        </property>
        <property name="map">
            <map>
                <entry key="212" value="1212"> </entry>
                <entry key="12" value="1212"> </entry>
                <entry key="21" value="1212"> </entry>
            </map>
        </property>
        <property name="set">
            <set>
                <value>asdasd</value>
                <value>asd</value>
                <value>add</value>
            </set>
        </property>
        <property name="properties">
            <props>
                <prop key="asdsa">111</prop>
                <prop key="assa">222</prop>
                <prop key="asa">333</prop>
            </props>
        </property>
    </bean>
```

###### 注意

- `property`标签**表示setter方式**注入，构造方式注入`constructor-arg`标签内部也可以写`<array>、<list>、<map>、<props>、<set>`标签。

- `List`的底层也是通过数组实现的，所以`<List>`和`<array>`标签是可以混用；
- 集合中要添加引用类型，只需要把标签`<value>`改成`<ref>`标签；

### IoC源码理解

