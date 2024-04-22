## Java事务管理

事务的ACID属性：原子性(Atomicity )、一致性( Consistency )、隔离性或独立性( Isolation)和持久性(Durabilily)

-  A（原子性）事务的原子操作单元，对数据的修改，要么全部执行，要么全部不执行； 

-  C（一致性）在事务开始和完成时，数据必须保持一致状态，相关的数据规则必须应用于事务的修改，以保证数据的完整性，事务结束时，所有的内部数据结构必须正确； 

-  I（隔离性）保证事务不受外部并发操作影响的独立环境执行； 

-  D（持久性）事务完成之后，对于数据的修改是永久的，即使系统出现故障也能够保持；

  事务分为声明式事务和编程事务

  声明事务是通过配置方式来管理事务，编程事务通过编写代码显式地管理事务的开始、提交和回滚。（比较少用）

java事务管理类型

 JDBC事务、JTA（Java Transaction API）事务、容器事务

#### 1.1 JDBC事务

JDBC的一切行为包括事务是基于一个Connection的，JDBC通过Connection对象进行事务管理。

常用的事物相关方法是：setAutoCommit    commit    rollback等

JDBC事务的优点： 接口较为简单，性能较好 缺点： 不支持多[数据库](https://cloud.tencent.com/solution/database?from_column=20065&from=20065)的事务

#### 1.2 JTA事务

Java事务API（Java Transaction API，简称JTA） Java事务服务（Java Transaction Service，简称JTS）

JTA和JTS一起，为J2EE平台提供了[分布式事务](https://cloud.tencent.com/product/dtf?from_column=20065&from=20065)服务。JTA只提供接口，没有具体的实现，需要J2EE服务提供商根据JTS规范提供，

扩展： 标准的分布式事务（Distributed Transaction） 一个事务管理器（Transaction）

### spring声明事务实现

开启声明事务用的是`@Transactional` ，本质上是使用的 代理和AOP来实现。

​	**事务拦截器链（Interceptor Chain）**：Spring的声明式事务依赖于AOP技术，在运行时动态生成代理对象并创建事务拦截器链。在方法调用链中，每个事务拦截器都会被依次调用，并根据事务属性的定义决定是否开启、提交或回滚事务。
​	**事务切点（Transaction Pointcut）**：事务切点定义了哪些方法需要被事务拦截器拦截并应用事务逻辑。
​	**事务属性解析**：在声明式事务中，事务属性可以通过注解（如@Transactional）或配置文件来指定。事务属性包括隔离级别、传播行为、超时设置等。
​	**事务管理器（Transaction Manager）**：事务管理器是Spring框架的核心组件之一。它负责处理实际的事务管理操作，与底层的数据访问技术（如JDBC、Hibernate等）进行交互。事务管理器负责事务的创建、提交和回滚，并与当前线程进行绑定。
​	**事务同步器（Transaction Synchronization）**：事务同步器用于在事务的不同阶段注册回调方法。在事务提交或回滚时，事务同步器会触发注册的回调方法，以执行一些额外的操作。例如，清理数据库连接、提交缓存数据等。Spring利用事务同步器来确保与事务相关的资源的正确管理和释放。
​	**事务切面（Transaction Aspect）**：事务切面是由事务拦截器和事务切点组成的，它定义了在目标方法执行前后应用事务逻辑的规则。事务切面通过AOP技术将事务管理逻辑与业务逻辑进行解耦。当目标方法被调用时，事务切面会根据事务属性的定义，决定是否开启、提交。

### @Transactional 注解的参数

#### rollbackFor：捕获到异常就回滚事务

~~~java
@Transactional(rollbackFor = Exception.class}
~~~

#### propagation (事务传播行为)

事务的传播行为是指：当前事务方法被调用的时候，需要做什么样的操作

<img src="C:\Users\86147\AppData\Roaming\Typora\typora-user-images\image-20240417105332416.png" alt="image-20240417105332416" style="zoom: 67%;" />

#### isolation（事务隔离级别）

数据库隔离级别

![image-20240417105309091](C:\Users\86147\AppData\Roaming\Typora\typora-user-images\image-20240417105309091.png)

### @Transactional 实践

在深入理解事务的传播行为之前，我们需要理解三个基本的概念，理解了它们我们就理解了事务传播行为。它们分别是：嵌套事务、新事务、当前事务

A事务的定义如下不会改变，B事务的传播行为可能会变。

~~~java
@Transactional(rollbackFor = Exception.class)
public void A() {
    userMapper.insertUser("A",1);
    sqlTestService.B();
}

public void B() {
    userMapper.insertUser("B",2);
}
~~~

##### 嵌套事务

修改B事务的传播行为，让它生成嵌套事务

~~~java
@Transactional(rollbackFor = Exception.class, propagation = Propagation.NESTED)
~~~

嵌套事务和父事务是有关联的，当A事务回滚的时候，B事务一定回滚。

当B事务异常回滚的时候，要判断在A里面是否try了B事务，如果try就A不会回滚，只是B回滚。

##### 新事务

修改B事务的传播行为，让它生成新事务

~~~java
@Transactional(rollbackFor = Exception.class, propagation = Propagation.REQUIRES_NEW)
~~~

A、B事务没有什么必然的关系

##### 当前事务

修改B事务的传播行为，让它加入当前事务

~~~java
@Transactional(rollbackFor = Exception.class, propagation = Propagation.REQUIRED)

@Transactional(rollbackFor = Exception.class, propagation = Propagation.SUPPORTS)
~~~

A、B方法都是一起提交、或一起回滚。

## Java主流ORM框架

ORM 是 Object Relational Mapping 的缩写，译为 “对象关系映射” 框架。

ORM 框架就是一种为了解决面向对象与关系型数据库中数据类型不匹配的技术，它通过描述 Java 对象与数据库表之间的映射关系，自动将 Java 应用程序中的对象持久化到关系型数据库的表中。

程序员使用 API 直接操作 JavaBean 对象就可以实现数据的存储、查询、更改和删除等操作。

<img src="http://c.biancheng.net/uploads/allimg/200714/1-200G415225NI.gif" alt="img" style="zoom: 67%;" />

常见的框架有 Hibernate 和 MyBatis

MyBatis 框架是一个半自动映射的框架。这里所谓的 “半自动” 是相对于 Hibernate 框架全表映射而言的，MyBatis 框架需要手动匹配提供 POJO、SQL 和映射关系，而 Hibernate 框架只需提供 POJO 和映射关系即可。

## Java基础

#### [JVM](https://javaguide.cn/java/basis/java-basic-questions-01.html#jvm)

Java 虚拟机（JVM）是运行 Java 字节码（即.class文件）的虚拟机。一次编译，随处可以运行

![运行在 Java 虚拟机之上的编程语言](https://oss.javaguide.cn/github/javaguide/java/basis/java-virtual-machine-program-language-os.png)

JIT为编译器把文件编译过一次就会保存下来机器码，后面再运行时效率就会提高

![Java程序转变为机器代码的过程](https://oss.javaguide.cn/github/javaguide/java/basis/java-code-to-machine-code.png)

### 基本类型和包装类型

基本类型包括：

1. 整型：byte, short, int, long
2. 浮点型：float, double
3. 字符型：char
4. 布尔型：boolean

包装类型

1. 整型：Byte, Short, Integer, Long
2. 浮点型：Float, Double
3. 字符型：Character
4. 布尔型：Boolean

包装类型主要用于集合类（如`ArrayList`、`LinkedList`等）和泛型中，因为这些类只能存储对象，无法存储基本类型。

- **默认值**：成员变量包装类型不赋值就是 `null` ，而基本类型有默认值且不是 `null`。

**几乎所有对象实例都存在于堆中**

**基本数据类型存放在栈中是一个常见的误区。如果它们是局部变量，那么它们会存放在栈中；如果它们是成员变量，那么它们会存放在堆中。！** 

### 包装类型的缓存机制

包装类型的==比较的是对象的内存地址

Java 基本数据类型的包装类型的大部分都用到了缓存机制来提升性能，

例如 Byte`,`Short`,`Integer`,`Long` 这 4 种包装类默认创建了数值 **[-128，127]** 的相应类型的缓存数据，

例如

Integer i1 = 40; 相当于Integer i1=Integer.valueOf(40)
Integer i2 = new Integer(40); 相当于新创建一个对象
System.out.println(i1==i2);  结果为false；

~~~java
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
private static class IntegerCache {
    static final int low = -128;
    static final int high;
    static {
        // high value may be configured by property
        int h = 127;
    }
}
~~~

`Float`,`Double` 并没有实现缓存机制。如果超出对应范围仍然会去创建新的对象

### 自动装箱与拆箱

- **装箱**：将基本类型用它们对应的引用类型包装起来；
- **拆箱**：将包装类型转换为基本数据类型；

并且频繁的拆装箱会损害系统性能

下面就是拆装箱的示例

~~~java
Integer i = 10;  //装箱 等价于Integer i = Integer.valueOf(10)
int n = i;   //拆箱 等价于int n = i.intValue()
~~~

### 如何解决浮点数运算的精度丢失问题

`BigDecimal` 可以实现对浮点数的运算，不会造成精度丢失。

~~~
BigDecimal a = new BigDecimal("1.0");
BigDecimal b = new BigDecimal("0.9");
BigDecimal c = new BigDecimal("0.8");

BigDecimal x = a.subtract(b);
BigDecimal y = b.subtract(c);

System.out.println(x); /* 0.1 */
System.out.println(y); /* 0.1 */
System.out.println(Objects.equals(x, y)); /* true */
~~~

### 静态变量

静态变量也就是被 `static` 关键字修饰的变量。它可以被类的所有实例共享，无论一个类创建了多少个对象，它们都共享同一份静态变量。

静态变量是通过类名来访问的，例如`StaticVariableExample.staticVar`（如果被 `private`关键字修饰就无法这样访问了）。

### 字符常量和字符串常量

字符常量只占 2 个字节; 字符串常量占若干个字节。

字符常量相当于一个整型值( ASCII 值),可以参加表达式运算; 字符串常量代表一个地址值(该字符串在内存中存放位置)

### 静态方法为什么不能调用非静态成员

1.静态方法是属于类的，在类加载的时候就会分配内存，可以通过类名直接访问。而非静态成员属于实例对象，只有在对象实例化之后才存在，需要通过类的实例对象去访问。

2.在类的非静态成员不存在的时候静态方法就已经存在了，此时调用在内存中还不存在的非静态成员，属于非法操作。

静态方法调用无需创建对象，实例则不行。但是实例可以访问静态成员和实例成员

### 重载和重写的区别

重载

发生在同一个类中（或者父类和子类之间），==方法名必须相同==，参数类型不同、个数不同、顺序不同，方法返回值和访问修饰符可以不同。

#### [重写](#重写)

重写发生在运行期，是子类对父类的允许访问的方法的实现过程进行重新编写。

1. 方法名、参数列表必须相同，子类方法返回值类型应比父类方法返回值类型更小或相等，抛出的异常范围小于等于父类，访问修饰符范围大于等于父类。
2. 如果父类方法访问修饰符为 `private/final/static` 则子类就不能重写该方法，但是被 `static` 修饰的方法能够被再次声明。
3. 构造方法无法被重写

**重写就是子类对父类方法的重新改造，外部样子不能改变，内部逻辑可以改变。**

## 面向对象基础

创建对象的运算符

new 运算符，new 创建对象实例（对象实例在堆内存中），对象引用指向对象实例（对象引用存放在栈内存中）

### 面向对象三大特征

#### 封装

封装是指把一个对象的状态信息（也就是属性）隐藏在对象内部，不允许外部对象直接访问对象的内部信息。

#### 继承

子类拥有父类对象所有的属性和方法（包括私有属性和私有方法），但是父类中的私有属性和方法子类是无法访问，**只是拥有**。

子类可以拥有自己属性和方法，即子类可以对父类进行扩展。

子类可以用自己的方式实现父类的方法。（以后介绍）。

#### 多态

多态，顾名思义，表示一个对象具有多种的状态，具体表现为父类的引用指向子类的实例。

~~~Java
// 父类 Animal
class Animal {
    public void makeSound() {
        System.out.println("动物发出声音");
    }
}

// 子类 Dog
class Dog extends Animal {
    @Override
    public void makeSound() {
        System.out.println("狗叫：汪汪汪");
    }
}

// 子类 Cat
class Cat extends Animal {
    @Override
    public void makeSound() {
        System.out.println("猫叫：喵喵喵");
    }
}

public class Main {
    public static void main(String[] args) {
        // 使用多态
        Animal animal1 = new Dog(); // Dog 对象
        Animal animal2 = new Cat(); // Cat 对象
        
        animal1.makeSound(); // 调用 Dog 类的 makeSound() 方法
        animal2.makeSound(); // 调用 Cat 类的 makeSound() 方法
    }
}

~~~

### 深拷贝和浅拷贝

1. **浅拷贝（Shallow Copy）**：
   - 浅拷贝创建一个新对象{==会在堆上创建一个新的对象，区别于引用拷贝的一点==}，然后将原始对象的非静态字段的值复制到新对象中。如果字段是基本类型，那么就会复制其值；如果字段是引用类型，则复制引用，而不是引用的对象本身。因此，新对象和原始对象共享相同的引用对象。
   - 修改新对象中的引用对象会影响到原始对象中相应的引用对象，因为它们指向同一块内存地址。
2. **深拷贝（Deep Copy）**：
   - 深拷贝创建一个新对象，并且递归地将原始对象的所有字段以及其引用的对象都复制到新对象中。这意味着新对象和原始对象是完全独立的，对新对象的修改不会影响到原始对象，反之亦然。
   - 在深拷贝中，即使字段是引用类型，也会复制引用的对象本身，而不仅仅是引用。

![浅拷贝、深拷贝、引用拷贝示意图](https://oss.javaguide.cn/github/javaguide/java/basis/shallow&deep-copy.png)

### Object

Object 类是一个特殊的类，是所有类的父类

#### ==和equals的区别

- 对于基本数据类型来说，`==` 比较的是值。
- 对于引用数据类型来说，`==` 比较的是对象的内存地址。
- equals不重写object类的话也是比较内存地址

`String` 中的 `equals` 方法是被重写过的，因为 `Object` 的 `equals` 方法是比较的对象的内存地址，而 `String` 的 `equals` 方法比较的是对象的值。

#### hashCode()的作用

`hashCode()` 的作用是获取哈希码（`int` 整数），也称为散列码。这个哈希码的作用是确定该对象在哈希表中的索引位置

`hashCode()` 定义在 JDK 的 `Object` 类中，这就意味着 Java 中的任何类都包含有 `hashCode()` 函数。

当你把对象加入 `HashSet` 时，`HashSet` 会先计算对象的 `hashCode` 值来判断对象加入的位置，同时也会与其他已经加入的对象的 `hashCode` 值作比较，如果没有相符的 `hashCode`，`HashSet` 会假设对象没有重复出现。但是如果发现有相同 `hashCode` 值的对象，这时会调用 `equals()` 方法来检查 `hashCode` 相等的对象是否真的相同。如果两者相同，`HashSet` 就不会让其加入操作成功。如果不同的话，就会重新散列到其他位置。这样我们就大大减少了 `equals` 的次数，相应就大大提高了执行速度。

#### 重写 equals() 时也要重写 hashCode() 方法

如果重写 `equals()` 时没有重写 `hashCode()` 方法的话就可能会导致 `equals` 方法判断是相等的两个对象，`hashCode` 值却不相等。

### String

#### String、StringBuffer、StringBuilder 的区别？

`String` 是不可变的

`StringBuilder` 与 `StringBuffer` 都继承自 `AbstractStringBuilder` 类，在 `AbstractStringBuilder` 中也是使用字符数组保存字符串，用append可以修改字符串

`String` 中的对象是不可变的，也就可以理解为常量，线程安全。`StringBuilder` 并没有对方法进行加同步锁，所以是非线程安全的。stringBuffer是安全的

性能方面

每次对 `String` 类型进行改变的时候，都会生成一个新的 `String` 对象，然后将指针指向新的 `String` 对象。

但使用StringBuilder要承担多线程不安全的风险

#### String为什么不可变

因为`String` 类中使用 `final` 关键字修饰字符数组来保存字符串

~~~java
public final class String implements java.io.Serializable, Comparable<String>, CharSequence {
    private final char value[];
  //...
}
~~~

保存字符串的数组被 `final` 修饰且为私有的，并且`String` 类没有提供/暴露修改这个字符串的方法。

`String` 类被 `final` 修饰导致其不能被继承，进而避免了子类破坏 `String` 不可变。

### 字符串拼接用到+

“+”和“+=”是专门为 String 类重载过的运算符，也是 Java 中仅有的两个重载过的运算符。

底层实际上是通过 `StringBuilder` 调用 `append()` 方法实现的，拼接完成之后调用 `toString()` 得到一个 `String` 对象 。

### 字符串常量池

**字符串常量池** 是 JVM 为了提升性能和减少内存消耗针对字符串（String 类）专门开辟的一块区域，主要目的是为了避免字符串的重复创建。如果要创建的字符串存在，则从池中取出引用

~~~java
// 在堆中创建字符串对象”ab“
// 将字符串对象”ab“的引用保存在字符串常量池中
String aa = "ab";
// 直接返回字符串常量池中字符串对象”ab“的引用
String bb = "ab";
System.out.println(aa==bb);// true
//会在堆中创建一个单独的对象"ab"
String s = new String("ab");
~~~

## 异常

<img src="https://oss.javaguide.cn/github/javaguide/java/basis/types-of-exceptions-in-java.png" alt="Java 异常类层次结构图" style="zoom:50%;" />

Exception和Error区别

**`Exception`** :程序本身可以处理的异常，可以通过 `catch` 来进行捕获。

又分为两类

**Checked Exception** 即 受检查异常 和**Unchecked Exception** (不受检查异常，可以不处理也可以正常通过编译)。

**`Error`**：`Error` 属于程序无法处理的错误，Java 虚拟机（JVM）一般会选择线程终止

#### try-catch-finally

**不要在 finally 语句块中使用 return!** 当 try 语句和 finally 语句中都有 return 语句时，try 语句块中的 return 语句会被忽略。

## 泛型

编译器可以对泛型参数进行检测，并且通过泛型参数可以指定传入的对象类型。

有三种使用方式，泛型类，泛型接口，泛型方法

实例化泛型

~~~
ArrayList<E> extends AbstractList<E>
~~~

## 反射

通过反射你可以获取任意一个类的所有属性和方法，你还可以调用这些方法和属性。是框架的灵魂

Spring/Spring Boot、MyBatis **大量使用了动态代理，而动态代理的实现也依赖反射**

 JDK 实现动态代理的示例代码，其中就使用了反射类 `Method` 来调用指定的方法。

~~~java
public class DebugInvocationHandler implements InvocationHandler {
    /**
     * 代理类中的真实对象
     */
    private final Object target;

    public DebugInvocationHandler(Object target) {
        this.target = target;
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws InvocationTargetException, IllegalAccessException {
        System.out.println("before method " + method.getName());
        Object result = method.invoke(target, args);
        System.out.println("after method " + method.getName());
        return result;
    }
}
~~~

#### 反射实战

要动态获取信息，需要依靠class对象（字节码对象）

有四种方式

![image-20240414191006796](C:\Users\86147\AppData\Roaming\Typora\typora-user-images\image-20240414191006796.png)

使用反射操作

~~~java
package cn.javaguide;

import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;

public class Main {
    public static void main(String[] args) throws ClassNotFoundException, NoSuchMethodException, IllegalAccessException, InstantiationException, InvocationTargetException, NoSuchFieldException {
        /**
         * 获取 TargetObject 类的 Class 对象并且创建 TargetObject 类实例
         */
        Class<?> targetClass = Class.forName("cn.javaguide.TargetObject");
        TargetObject targetObject = (TargetObject) targetClass.newInstance();
        /**
         * 获取 TargetObject 类中定义的所有方法
         */
        Method[] methods = targetClass.getDeclaredMethods();
        for (Method method : methods) {
            System.out.println(method.getName());
        }

        /**
         * 获取指定方法并调用
         */
        Method publicMethod = targetClass.getDeclaredMethod("publicMethod",
                String.class);

        publicMethod.invoke(targetObject, "JavaGuide");

        /**
         * 获取指定参数并对参数进行修改
         */
        Field field = targetClass.getDeclaredField("value");
        //为了对类中的参数进行修改我们取消安全检查
        field.setAccessible(true);
        field.set(targetObject, "JavaGuide");

        /**
         * 调用 private 方法
         */
        Method privateMethod = targetClass.getDeclaredMethod("privateMethod");
        //为了调用private方法我们取消安全检查
        privateMethod.setAccessible(true);
        privateMethod.invoke(targetObject);
    }
}
~~~

## 注解

注解本质是一个继承了`Annotation` 的特殊接口

注解只有被解析之后才会生效，常见的解析方法有两种：

**编译期直接扫描**和**运行期通过反射处理**

## SPI

Service Provider Interface 

专门提供给服务提供者或者扩展框架功能的开发者去使用的一个接口。

### 和API区别

<img src="https://oss.javaguide.cn/github/javaguide/java/basis/spi/1ebd1df862c34880bc26b9d494535b3dtplv-k3u1fbpfcp-watermark.png" alt="img" style="zoom:50%;" />

当接口存在于调用方这边时，就是 SPI ，由接口调用方确定接口规则，然后由不同的厂商去根据这个规则对这个接口进行实现，从而提供服务。



## 序列化和反序列化

如果我们需要持久化 Java 对象比如将 Java 对象保存在文件中，或者在网络传输 Java 对象，这些场景都需要用到序列化。

- **序列化**：将数据结构或对象转换成二进制字节流的过程
- **反序列化**：将在序列化过程中所生成的二进制字节流转换成数据结构或者对象的过程

**序列化的主要目的是通过网络传输对象或者说是将对象存储到文件系统、数据库、内存中。**

对于不想进行序列化的变量，使用 `transient` 关键字修饰。

- `transient` 只能修饰变量，不能修饰类和方法。
- `transient` 修饰的变量，在反序列化后变量值将会被置成类型的默认值。例如，如果是修饰 `int` 类型，那么反序列后结果就是 `0`。
- `static` 变量因为不属于任何对象(Object)，所以无论有没有 `transient` 关键字修饰，均不会被序列化。

Json是一种序列化协议

JDK自带的序列化方式，只需实现 `java.io.Serializable`接口即可。然后最后手动设置**serialVersionUID**。

在反序列化时，会检查当前类和**serialVersionUID**是否一致，不一致会抛出异常。

## I/O

Java IO 流的 40 多个类都是从如下 4 个抽象类基类中派生出来的。

- `InputStream`/`Reader`: 所有的输入流的基类，前者是字节输入流，后者是字符输入流。
- `OutputStream`/`Writer`: 所有输出流的基类，前者是字节输出流，后者是字符输出流。

Java只有值传递（只会拷贝实参的地址或者实参的值然后创建一个副本），没有引用传递，

## Java代理模式

代理模式是一种比较好理解的设计模式。简单来说就是 **我们使用代理对象来代替对真实对象(real object)的访问，这样就可以在不修改原目标对象的前提下，提供额外的功能操作，扩展目标对象的功能。**

静态代理在实际中应用效果不佳。**静态代理在编译时就将接口、实现类、代理类这些都变成了一个个实际的 class 文件。**

~~~java
public class SmsProxy implements SmsService {

    private final SmsService smsService;

    public SmsProxy(SmsService smsService) {
        this.smsService = smsService;
    }
	//重写了send方法，相当于加强了原有目标的功能
    @Override
    public String send(String message) {
        //调用方法之前，我们可以添加自己的操作
        System.out.println("before method send()");
        smsService.send(message);
        //调用方法之后，我们同样可以添加自己的操作
        System.out.println("after method send()");
        return null;
    }
}

~~~

动态代理

**动态代理是在运行时动态生成类字节码，并加载到 JVM 中的**

#### JDK动态代理机制

**在 Java 动态代理机制中 `InvocationHandler` 接口和 `Proxy` 类是核心。**

`Proxy` 类中使用频率最高的方法是：`newProxyInstance()` ，这个方法主要用来生成一个代理对象。

~~~java
    public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException
    {
        ......
    }
/*
loader :类加载器，用于加载代理对象。
interfaces : 被代理类实现的一些接口；
h : 实现了 InvocationHandler 接口的对象
*/
~~~



**通过`Proxy` 类的 `newProxyInstance()` 创建的代理对象在调用方法的时候，实际会调用到实现`InvocationHandler` 接口的类的 `invoke()`方法。** 你可以在 `invoke()` 方法中自定义处理逻辑，比如在方法执行前后做什么事情。

~~~java
public interface InvocationHandler {

    /**
     * 当你使用代理对象调用方法的时候实际会调用到这个方法
     proxy :动态生成的代理类
	method : 与代理类对象调用的方法相对应
	args : 当前 method 方法的参数
     */
    public Object invoke(Object proxy, Method method, Object[] args)
        throws Throwable;
}
~~~

使用步骤

1. 定义一个接口及其实现类；
2. 自定义 `InvocationHandler`（implements InvocationHandler接口） 并重写`invoke`方法，在 `invoke` 方法中我们会调用原生方法（被代理类的方法）并自定义一些处理逻辑；
3. 通过 `Proxy.newProxyInstance(ClassLoader loader,Class<?>[] interfaces,InvocationHandler h)` 方法创建代理对象；

### 语法糖

只是方便使用 ，JVM并不支持这些语法糖**在编译阶段就会被还原成简单的基础语法结构，这个过程就是解语法糖**

##### 可变长参数

它允许一个方法把任意数量的值作为参数。

~~~java
 print("Holis", "公众号:Hollis", "博客：www.hollischuang.com", "QQ：907607222");
~~~

##### 内部类

内部类又称为嵌套类，可以把内部类理解为外部类的一个普通成员。内部类之所以也是语法糖，是因为它仅仅是一个编译时的概念，`outer.java`里面定义了一个内部类`inner`，一旦编译成功，就会生成两个完全不同的`.class`文件了，分别是`outer.class`和`outer$inner.class`。所以内部类的名字完全可以和它的外部类名字相同。

lambda 表达式也是语法糖

**lambda 表达式的实现其实是依赖了一些底层的 api，在编译阶段，编译器会把 lambda 表达式进行解糖，转换成调用内部 api 的方式。**

### Unsafe

==TODO 待了解==

Java语言先比较与C和C++有一个非常大的不同点在于Java语言无法直接操作内存，Unsafe提供了通过Java直接操作内存的API。

`Unsafe` 提供的这些功能的实现需要依赖本地方法（Native Method）。你可以将本地方法看作是 Java 中使用其他编程语言编写的方法。

但是使用这个类有很多限制，由`Bootstrap classLoader`加载不会报错。

有两个方案调动

1、利用反射获得 Unsafe 类中已经实例化完成的单例对象 `theUnsafe` 

~~~java
private static Unsafe reflectGetUnsafe() {
    try {
      Field field = Unsafe.class.getDeclaredField("theUnsafe");
      field.setAccessible(true);
      return (Unsafe) field.get(null);
    } catch (Exception e) {
      log.error(e.getMessage(), e);
      return null;
    }
}
~~~

## Java集合

Java 集合，也叫作容器，主要是由两大接口派生而来

一个是 `Collection`接口（单列值集合），一个是 `Map` （键值对集合）接口

<img src="https://oss.javaguide.cn/github/javaguide/java/collection/java-collection-hierarchy.png" alt="Java 集合框架概览" style="zoom: 50%;" />

### [说说 List, Set, Queue, Map 四者的区别？](#说说-list-set-queue-map-四者的区别)

- `List`(对付顺序的好帮手): 存储的元素是有序的、可重复的。
- `Set`(注重独一无二的性质): 存储的元素不可重复的。
- `Queue`(实现排队功能的叫号机): 按特定的排队规则来确定先后顺序，存储的元素是有序的、可重复的。
- `Map`(用 key 来搜索的专家): 使用键值对（key-value）存储，类似于数学上的函数 y=f(x)，"x" 代表 key，"y" 代表 value，key 是无序的、不可重复的，value 是无序的、可重复的，每个键最多映射到一个值。

### ArrayList 源码分析

`ArrayList` 的底层是数组队列，相当于动态数组，容量可以动态增长，可以存储null值，不过不建议

~~~java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable{

  }
~~~

`List` : 表明它是一个列表，支持添加、删除、查找等操作，并且可以通过下标进行访问。

`RandomAccess` ：这是一个标志接口，表明实现这个接口的 `List` 集合是支持 **快速随机访问** 的。在 `ArrayList` 中，我们即可以通过元素的序号快速获取元素对象，这就是快速随机访问。

`Cloneable` ：表明它具有拷贝能力，可以进行深拷贝或浅拷贝操作。

`Serializable` : 表明它可以进行序列化操作，也就是可以将对象转换为字节流进行持久化存储或网络传输，非常方便。

### Arraylist 与 LinkedList 区别

都不是线程安全的

**底层数据结构：**`ArrayList` 底层使用的是 **`Object` 数组**；`LinkedList` 底层使用的是 **双向链表** 数据结构

**插入和删除是否受元素位置的影响**：

`ArrayList` 采用数组存储，所以插入和删除元素的时间复杂度受元素位置的影响。

`LinkedList` 采用链表存储，所以在头尾插入或者删除元素不受元素位置的影响（`add(E e)`、`addFirst(E e)`、`addLast(E e)`、`removeFirst()`、 `removeLast()`），时间复杂度为 O(1)，如果是要在指定位置 `i` 插入和删除元素的话（`add(int index, E element)`，`remove(Object o)`,`remove(int index)`）， 时间复杂度为 O(n) ，因为需要先移动到指定位置再插入和删除。

**LinkerList不支持快速随机访问**

**内存空间占用：**

`ArrayList` 的空间浪费主要体现在在 list 列表的结尾会预留一定的容量空间，而 LinkedList 的空间花费则体现在它的每一个元素都需要消耗比 ArrayList 更多的空间

### 源码分析

#### 扩容机制

~~~java
/**
 * 默认初始容量大小
 */
private static final int DEFAULT_CAPACITY = 10;

private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

/**
 * 默认构造函数，使用初始容量10构造一个空列表(无参数构造)
 */
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}

/**
 * 带初始容量参数的构造函数。（用户自己指定容量）
 */
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {//初始容量大于0
        //创建initialCapacity大小的数组
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {//初始容量等于0
        //创建空数组
        this.elementData = EMPTY_ELEMENTDATA;
    } else {//初始容量小于0，抛出异常
        throw new IllegalArgumentException("Illegal Capacity: " + initialCapacity);
    }
}
/**
 *构造包含指定collection元素的列表，这些元素利用该集合的迭代器按顺序返回
 *如果指定的集合为null，throws NullPointerException。
 */
public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    if ((size = elementData.length) != 0) {
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        // replace with empty array.
        this.elementData = EMPTY_ELEMENTDATA;
    }
}
~~~

**以无参数构造方法创建 `ArrayList` 时，实际上初始化赋值的是一个空数组。当真正对数组进行添加元素操作时，才真正分配容量。即向数组中添加第一个元素时，数组容量扩为 10。**

> 补充：JDK6 new 无参构造的 `ArrayList` 对象时，直接创建了长度是 10 的 `Object[]` 数组 `elementData` 。

add方法

~~~java
/**
* 将指定的元素追加到此列表的末尾。
*/
public boolean add(E e) {
    // 加元素之前，先调用ensureCapacityInternal方法
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    // 这里看到ArrayList添加元素的实质就相当于为数组赋值
    elementData[size++] = e;
    return true;
}
~~~

- 当我们要 `add` 进第 1 个元素到 `ArrayList` 时，`elementData.length` 为 0 （因为还是一个空的 list），因为执行了 `ensureCapacityInternal()` 方法 ，所以 `minCapacity` 此时为 10。此时，`minCapacity - elementData.length > 0`成立，所以会进入 `grow(minCapacity)` 方法。
- 当 `add` 第 2 个元素时，`minCapacity` 为 2，此时 `elementData.length`(容量)在添加第一个元素后扩容成 `10` 了。此时，`minCapacity - elementData.length > 0` 不成立，所以不会进入 （执行）`grow(minCapacity)` 方法。
- 添加第 3、4···到第 10 个元素时，依然不会执行 grow 方法，数组容量都为 10。

直到添加第 11 个元素，`minCapacity`(为 11)比 `elementData.length`（为 10）要大。进入 `grow` 方法进行扩容

#### grow方法扩容

`int newCapacity = oldCapacity + (oldCapacity >> 1)`,所以 ArrayList 每次扩容之后容量都会变为原来的 1.5 倍左右（oldCapacity 为偶数就是 1.5 倍，否则是 1.5 倍左右）

~~~java
/**
 * 要分配的最大数组大小
 */
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

/**
 * ArrayList扩容的核心方法。
 */
private void grow(int minCapacity) {
    // oldCapacity为旧容量，newCapacity为新容量
    int oldCapacity = elementData.length;
    // 将oldCapacity 右移一位，其效果相当于oldCapacity /2，
    // 我们知道位运算的速度远远快于整除运算，整句运算式的结果就是将新容量更新为旧容量的1.5倍，
    int newCapacity = oldCapacity + (oldCapacity >> 1);

    // 然后检查新容量是否大于最小需要容量，若还是小于最小需要容量，那么就把最小需要容量当作数组的新容量，
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;

    // 如果新容量大于 MAX_ARRAY_SIZE,进入(执行) `hugeCapacity()` 方法来比较 minCapacity 和 MAX_ARRAY_SIZE，
    // 如果minCapacity大于最大容量，则新容量则为`Integer.MAX_VALUE`，否则，新容量大小则为 MAX_ARRAY_SIZE 即为 `Integer.MAX_VALUE - 8`。
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);

    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
~~~



当 `add` 第 1 个元素时，`oldCapacity` 为 0，经比较后第一个 if 判断成立，`newCapacity = minCapacity`(为 10)。但是第二个 if 判断不会成立，即 `newCapacity` 不比 `MAX_ARRAY_SIZE` 大，则不会进入 `hugeCapacity` 方法。数组容量为 10，`add` 方法中 return true,size 增为 1。

当 `add` 第 11 个元素进入 `grow` 方法时，`newCapacity` 为 15，比 `minCapacity`（为 11）大，第一个 if 判断不成立。新容量没有大于数组最大 size，不会进入 huge`C`apacity 方法。数组容量扩为 15，add 方法中 return true,size 增为 11。

以此类推······

**这里补充一点比较重要，但是容易被忽视掉的知识点：**

- Java 中的 `length`属性是针对数组说的,比如说你声明了一个数组,想知道这个数组的长度则用到了 length 这个属性.
- Java 中的 `length()` 方法是针对字符串说的,如果想看这个字符串的长度则用到 `length()` 这个方法.
- Java 中的 `size()` 方法是针对泛型集合说的,如果想看这个泛型有多少个元素,就调用此方法来查看

#### hugeCapacity() 方法

从上面 `grow()` 方法源码我们知道：如果新容量大于 `MAX_ARRAY_SIZE`,进入(执行) `hugeCapacity()` 方法来比较 `minCapacity` 和 `MAX_ARRAY_SIZE`，如果 `minCapacity` 大于最大容量，则新容量则为`Integer.MAX_VALUE`，否则，新容量大小则为 `MAX_ARRAY_SIZE` 即为 `Integer.MAX_VALUE - 8`。

~~~java
private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    // 对minCapacity和MAX_ARRAY_SIZE进行比较
    // 若minCapacity大，将Integer.MAX_VALUE作为新数组的大小
    // 若MAX_ARRAY_SIZE大，将MAX_ARRAY_SIZE作为新数组的大小
    // MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}

~~~

### [`System.arraycopy()` 和 `Arrays.copyOf()`方法](https://javaguide.cn/java/collection/arraylist-source-code.html#system-arraycopy-和-arrays-copyof-方法)

`ArrayList` 中大量调用了这两个方法，我们上面讲的扩容操作以及`add(int index, E element)`、`toArray()` 等方法中都用到了该方法

#### [`System.arraycopy()` 方法](https://javaguide.cn/java/collection/arraylist-source-code.html#system-arraycopy-方法)

~~~java
    // 我们发现 arraycopy 是一个 native 方法,接下来我们解释一下各个参数的具体意义
    /**
    *   复制数组
    * @param src 源数组
    * @param srcPos 源数组中的起始位置
    * @param dest 目标数组
    * @param destPos 目标数组中的起始位置
    * @param length 要复制的数组元素的数量
    */
    public static native void arraycopy(Object src,  int  srcPos,
                                        Object dest, int destPos,
                                        int length);
~~~

#### [`Arrays.copyOf()`方法](https://javaguide.cn/java/collection/arraylist-source-code.html#arrays-copyof-方法)

~~~java
    public static int[] copyOf(int[] original, int newLength) {
      // 申请一个新的数组
        int[] copy = new int[newLength];
  // 调用System.arraycopy,将源数组中的数据进行拷贝,并返回新的数组
        System.arraycopy(original, 0, copy, 0,
                         Math.min(original.length, newLength));
        return copy;
    }
~~~

#### [两者联系和区别](https://javaguide.cn/java/collection/arraylist-source-code.html#两者联系和区别)

`copyOf()`内部实际调用了 `System.arraycopy()` 方法

**区别：**

`arraycopy()` 需要目标数组，将原数组拷贝到你自己定义的数组里或者原数组，而且可以选择拷贝的起点和长度以及放入新数组中的位置 

`copyOf()` 是系统自动在内部新建一个数组，并返回该数组。

## LinkedList

### [LinkedList 为什么不能实现 RandomAccess 接口？](#linkedlist-为什么不能实现-randomaccess-接口)

`RandomAccess` 是一个标记接口，用来表明实现该接口的类支持随机访问（即可以通过索引快速访问元素）。由于 `LinkedList` 底层数据结构是链表，内存地址不连续，只能通过指针来定位，不支持随机快速访问，所以不能实现 `RandomAccess` 接口。

### 源码分析

~~~java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
{
  //...
}
~~~

`LinkedList` 继承了 `AbstractSequentialList` ，而 `AbstractSequentialList` 又继承于 `AbstractList` 。

阅读过 `ArrayList` 的源码我们就知道，`ArrayList` 同样继承了 `AbstractList` ， 所以 `LinkedList` 会有大部分方法和 `ArrayList` 相似。

实现一下接口

- `List` : 表明它是一个列表，支持添加、删除、查找等操作，并且可以通过下标进行访问。
- `Deque` ：继承自 `Queue` 接口，具有双端队列的特性，支持从两端插入和删除元素，方便实现栈和队列等数据结构。需要注意，`Deque` 的发音为 "deck" [dɛk]，这个大部分人都会读错。
- `Cloneable` ：表明它具有拷贝能力，可以进行深拷贝或浅拷贝操作。
- `Serializable` : 表明它可以进行序列化操作，也就是可以将对象转换为字节流进行持久化存储或网络传输，非常方便。

LinkedList的元素定义

~~~java
private static class Node<E> {
    E item;// 节点值
    Node<E> next; // 指向的下一个节点（后继节点）
    Node<E> prev; // 指向的前一个节点（前驱结点）

    // 初始化参数顺序分别是：前驱结点、本身节点值、后继节点
    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
~~~

### [初始化](https://javaguide.cn/java/collection/linkedlist-source-code.html#初始化)

`LinkedList` 中有一个无参构造函数和一个有参构造函数。

~~~java
// 创建一个空的链表对象
public LinkedList() {
}

// 接收一个集合类型作为参数，会创建一个与传入集合相同元素的链表对象
public LinkedList(Collection<? extends E> c) {
    this();
    addAll(c);
}
~~~

### [插入元素](https://javaguide.cn/java/collection/linkedlist-source-code.html#插入元素)

`LinkedList` 除了实现了 `List` 接口相关方法，还实现了 `Deque` 接口的很多方法，所以我们有很多种方式插入元素。既可以头插也可以尾插

### [获取元素](#获取元素)

`LinkedList`获取元素相关的方法一共有 3 个：

1. `getFirst()`：获取链表的第一个元素。
2. `getLast()`：获取链表的最后一个元素。
3. `get(int index)`：获取链表指定位置的元素。

`get(int index)` 或 `remove(int index)` 等方法内部都调用了该方法来获取对应的节点。

从这个方法的源码可以看出，该方法通过比较索引值与链表 size 的一半大小来确定从链表头还是尾开始遍历。如果索引值小于 size 的一半，就从链表头开始遍历，反之从链表尾开始遍历。这样可以在较短的时间内找到目标节点，充分利用了双向链表的特性来提高效率

### [删除元素](#删除元素)

`LinkedList`删除元素相关的方法一共有 5 个：

1. `removeFirst()`：删除并返回链表的第一个元素。

2. `removeLast()`：删除并返回链表的最后一个元素。

3. `remove(E e)`：删除链表中首次出现的指定元素，如果不存在该元素则返回 false。

4. `remove(int index)`：删除指定索引处的元素，并返回该元素的值。

5. `void clear()`：移除此链表中的所有元素

6. ~~~java
   // 删除链表中首次出现的指定元素，如果不存在该元素则返回 false
   public boolean remove(Object o) {
       // 如果指定元素为 null，遍历链表找到第一个为 null 的元素进行删除
       if (o == null) {
           for (Node<E> x = first; x != null; x = x.next) {
               if (x.item == null) {
                   unlink(x);
                   return true;
               }
           }
       } else {
           // 如果不为 null ,遍历链表找到要删除的节点
           for (Node<E> x = first; x != null; x = x.next) {
               if (o.equals(x.item)) {
                   unlink(x);
                   return true;
               }
           }
       }
       return false;
   }
   
   ~~~

删除元素的核心在于 `unlink(Node<E> x)` 这个方法：

~~~java
E unlink(Node<E> x) {
    // 断言 x 不为 null
    // assert x != null;
    // 获取当前节点（也就是待删除节点）的元素
    final E element = x.item;
    // 获取当前节点的下一个节点
    final Node<E> next = x.next;
    // 获取当前节点的前一个节点
    final Node<E> prev = x.prev;

    // 如果前一个节点为空，则说明当前节点是头节点
    if (prev == null) {
        // 直接让链表头指向当前节点的下一个节点
        first = next;
    } else { // 如果前一个节点不为空
        // 将前一个节点的 next 指针指向当前节点的下一个节点
        prev.next = next;
        // 将当前节点的 prev 指针置为 null，，方便 GC 回收
        x.prev = null;
    }

    // 如果下一个节点为空，则说明当前节点是尾节点
    if (next == null) {
        // 直接让链表尾指向当前节点的前一个节点
        last = prev;
    } else { // 如果下一个节点不为空
        // 将下一个节点的 prev 指针指向当前节点的前一个节点
        next.prev = prev;
        // 将当前节点的 next 指针置为 null，方便 GC 回收
        x.next = null;
    }

    // 将当前节点元素置为 null，方便 GC 回收
    x.item = null;
    size--;
    modCount++;
    return element;
}
~~~

`unlink()` 方法的逻辑如下：

1. 首先获取待删除节点 x 的前驱和后继节点；
2. 判断待删除节点是否为头节点或尾节点： 
   - 如果 x 是头节点，则将 first 指向 x 的后继节点 next
   - 如果 x 是尾节点，则将 last 指向 x 的前驱节点 prev
   - 如果 x 不是头节点也不是尾节点，执行下一步操作
3. 将待删除节点 x 的前驱的后继指向待删除节点的后继 next，断开 x 和 x.prev 之间的链接；
4. 将待删除节点 x 的后继的前驱指向待删除节点的前驱 prev，断开 x 和 x.next 之间的链接；
5. 将待删除节点 x 的元素置空，修改链表长度。

### [遍历链表](https://javaguide.cn/java/collection/linkedlist-source-code.html#遍历链表)

推荐使用`for-each` 循环来遍历 `LinkedList` 中的元素， `for-each` 循环最终会转换成迭代器形式

迭代器 `ListItr` 中的核心方法进行详细介绍

这里面的next不是当前节点的next节点，而是下一个要遍历的节点

~~~java
// 双向迭代器
private class ListItr implements ListIterator<E> {
    // 表示上一次调用 next() 或 previous() 方法时经过的节点；
    private Node<E> lastReturned;
    // 表示下一个要遍历的节点；
    private Node<E> next;
    // 表示下一个要遍历的节点的下标，也就是当前节点的后继节点的下标；
    private int nextIndex;
    // 表示当前遍历期望的修改计数值，用于和 LinkedList 的 modCount 比较，判断链表是否被其他线程修改过。
    private int expectedModCount = modCount;
}
~~~

下面是从头到尾方向的迭代

~~~java
// 判断还有没有下一个节点
public boolean hasNext() {
    // 判断下一个节点的下标是否小于链表的大小，如果是则表示还有下一个元素可以遍历
    return nextIndex < size;
}
// 获取下一个节点
public E next() {
    // 检查在迭代过程中链表是否被修改过
    checkForComodification();
    // 判断是否还有下一个节点可以遍历，如果没有则抛出 NoSuchElementException 异常
    if (!hasNext())
        throw new NoSuchElementException();
    // 将 lastReturned 指向当前节点
    lastReturned = next;
    // 将 next 指向下一个节点
    next = next.next;
    nextIndex++;
    return lastReturned.item;
}
~~~

如果需要删除或插入元素，也可以使用迭代器进行操作。

#### RandomAccess 接口

~~~java
public interface RandomAccess {
}
~~~

==这个接口什么都没有实现，只是一个标识，用来标识这个类有随机访问的能力==

### Set

#### Comparable 和 Comparator 的区别

`Comparable` 接口和 `Comparator` 接口都是 Java 中用于排序的接口

`Comparable` 接口实际上是出自`java.lang`包 它有一个 `compareTo(Object obj)`方法用来排序

`Comparator`接口实际上是出自 `java.util` 包它有一个`compare(Object obj1, Object obj2)`方法用来排序

~~~java
Comparator 定制排序
Collections.sort(arrayList, new Comparator<Integer>() {
    @Override
    public int compare(Integer o1, Integer o2) {
        return o2.compareTo(o1);
    }
});

~~~

#### 重写 compareTo 方法实现按年龄来排序

~~~java
// person对象没有实现Comparable接口，所以必须实现，这样才不会出错，才可以使treemap中的数据按顺序排列
// 前面一个例子的String类已经默认实现了Comparable接口，详细可以查看String类的API文档，另外其他
// 像Integer类等都已经实现了Comparable接口，所以不需要另外实现了
public  class Person implements Comparable<Person> {
    private String name;
    private int age;

    public Person(String name, int age) {
        super();
        this.name = name;
        this.age = age;
    }
    /**
     * T重写compareTo方法实现按年龄来排序
     */
    @Override
    public int compareTo(Person o) {
        if (this.age > o.getAge()) {
            return 1;
        }
        if (this.age < o.getAge()) {
            return -1;
        }
        return 0;
    }
}
~~~

无序性：不等于随机性，是根据对象的hashcode值来进行决定

不可重复性：指添加的元素按照 `equals()` 判断时 ，返回 false，需要同时重写 `equals()` 方法和 `hashCode()` 方法。

以确保相等的对象具有相等的哈希码，并且哈希码相等的对象也必须根据`equals()`方法判断为相等

###  HashSet、LinkedHashSet 和 TreeSet 三者的异同

都不是线程安全的，并且元素唯一

`HashSet`、`LinkedHashSet` 和 `TreeSet` 的主要区别在于底层数据结构不同。`HashSet` 的底层数据结构是哈希表（基于 `HashMap` 实现）。`LinkedHashSet` 的底层数据结构是链表和哈希表，元素的插入和取出顺序满足 FIFO。`TreeSet` 底层数据结构是红黑树，元素是有序的，排序的方式有自然排序和定制排序。

### Queue

#### Queue 与 Deque 的区别

- `Queue` 扩展了 `Collection` 的接口，根据 **因为容量问题而导致操作失败后处理方式的不同** 可以分为两类方法: 一种在操作失败后会抛出异常，另一种则会返回特殊值。
- `Deque` 扩展了 `Queue` 的接口, 增加了在队首和队尾进行插入和删除的方法，同样根据失败后处理方式的不同分为两类：
- `Deque` 是双端队列，在队列的两端均可以插入或删除元素。
- 事实上，`Deque` 还提供有 `push()` 和 `pop()` 等其他方法，可用于模拟栈。

### [ArrayDeque 与 LinkedList 的区别](#arraydeque-与-linkedlist-的区别)

`ArrayDeque` 和 `LinkedList` 都实现了 `Deque` 接口，两者都具有队列的功能，但两者有什么区别呢？

- `ArrayDeque` 是基于可变长的数组和双指针来实现，而 `LinkedList` 则通过链表来实现。
- `ArrayDeque` 不支持存储 `NULL` 数据，但 `LinkedList` 支持。
- `ArrayDeque` 是在 JDK1.6 才被引入的，而`LinkedList` 早在 JDK1.2 时就已经存在。
- `ArrayDeque` 插入时可能存在扩容过程, 不过均摊后的插入操作依然为 O(1)。虽然 `LinkedList` 不需要扩容，但是每次插入数据时均需要申请新的堆空间，均摊性能相比更慢。

从性能的角度上，选用 `ArrayDeque` 来实现队列要比 `LinkedList` 更好。此外，`ArrayDeque` 也可以用于实现栈。

#### PriorityQueue

1. `PriorityQueue` 利用了==二叉堆的数据结构来实现的==，==底层使用可变长的数组来存储数据==
2. `PriorityQueue` 通过堆元素的上浮和下沉，实现了==在 O(logn) 的时间复杂度内插入元素和删除堆顶元素===。
3. `PriorityQueue` 是非线程安全的，且不支持存储 `NULL` 和 `non-comparable` 的对象。
4. ==`PriorityQueue` 默认是小顶堆==

### BlockingQueue

`BlockingQueue` （阻塞队列）是一个接口，继承自 `Queue`。`BlockingQueue`阻塞的原因是其支持当队列没有元素时一直阻塞，直到有元素；还支持如果队列已满，一直等到队列可以放入新元素时再放入

常用阻塞队列

ArrayBlockingQueue，LinkedBlockingQueue，PriorityBlockingQueue

### ArrayBlockingQueue 和 LinkedBlockingQueue 有什么区别

`它们都是线程安全的

区别：

1. 底层实现：`ArrayBlockingQueue` 基于数组实现，而 `LinkedBlockingQueue` 基于链表实现。
2. 是否有界：`ArrayBlockingQueue` 是有界队列，必须在创建时指定容量大小。`LinkedBlockingQueue` 创建时可以不指定容量大小，默认是`Integer.MAX_VALUE`，也就是无界的。但也可以指定队列大小，从而成为有界的。
3. 锁是否分离： `ArrayBlockingQueue`中的锁是没有分离的，即生产和消费用的是同一个锁；`LinkedBlockingQueue`中的锁是分离的，即生产用的是`putLock`，消费是`takeLock`，这样可以防止生产者和消费者线程之间的锁争夺。
4. 内存占用：`ArrayBlockingQueue` 需要提前分配数组内存，而 `LinkedBlockingQueue` 则是动态分配链表节点内存。这意味着，`ArrayBlockingQueue` 在创建时就会占用一定的内存空间，且往往申请的内存比实际所用的内存更大，而`LinkedBlockingQueue` 则是根据元素的增加而逐渐占用内存空间。

### ArrayBlockingQueue 源码分析

阻塞队列思想

​	阻塞队列就说基于非空和非满两个条件实现生产者和消费者之间的交互，尽管这些交互流程和等待通知的机制实现非常复杂，好在 Doug Lea 的操刀之下已将阻塞队列的细节屏蔽，我们只需调用 `put`、`take`、`offer`、`poll` 等 API 即可实现多线程之间的生产和消费。

![ArrayBlockingQueue 类图](https://oss.javaguide.cn/github/javaguide/java/collection/arrayblockingqueue-class-diagram.png)

它通过 `AbstractCollection` 获得了集合的常见操作方法，然后通过 `Queue` 接口获得了队列的特性。

#### 核心构造方法

~~~java
// capacity 表示队列初始容量，fair 表示 锁的公平性
public ArrayBlockingQueue(int capacity, boolean fair) {
  //如果设置的队列大小小于0，则直接抛出IllegalArgumentException
  if (capacity <= 0)
      throw new IllegalArgumentException();
  //初始化一个数组用于存放队列的元素
  this.items = new Object[capacity];
  //创建阻塞队列流程控制的锁
  lock = new ReentrantLock(fair);
  //用lock锁创建两个条件控制队列生产和消费
  notEmpty = lock.newCondition();
  notFull =  lock.newCondition();
}
~~~

默认使用非公平锁，即各个生产者或者消费者线程收到通知后，对于锁的争抢是随机的。

阻塞式获取和新增元素方法：

阻塞式就是在进行操作时判断的notEmpty和notFull锁的状态是否符合。

`put(E e)`：将元素插入队列中，如果队列已满，则该方法会一直阻塞，直到队列有空间可用或者线程被中断。

`take()` ：获取并移除队列头部的元素，如果队列为空，则该方法会一直阻塞，直到队列非空或者线程被中断。

#### put方法源码：

~~~java
public void put(E e) throws InterruptedException {
    //确保插入的元素不为null
    checkNotNull(e);
    //加锁
    //消费和生产用的是同一把锁
    final ReentrantLock lock = this.lock;
    //这里使用lockInterruptibly()方法而不是lock()方法是为了能够响应中断操作，如果在等待获取锁的过程中被打断则该方法会抛出InterruptedException异常。
    lock.lockInterruptibly();
    try {
            //如果count等数组长度则说明队列已满，当前线程将被挂起放到AQS队列中，等待队列非满时插入（非满条件）。
       //在等待期间，锁会被释放，其他线程可以继续对队列进行操作。
        while (count == items.length)
            notFull.await();
           //如果队列可以存放元素，则调用enqueue将元素入队
        enqueue(e);
    } finally {
        //释放锁
        lock.unlock();
    }
}
~~~

~~~java
private void enqueue(E x) {
   //获取队列底层的数组
    final Object[] items = this.items;
    //将putindex位置的值设置为我们传入的x
    items[putIndex] = x;
    //更新putindex，如果putindex等于数组长度，则更新为0
    if (++putIndex == items.length)
        putIndex = 0;
    //队列长度+1
    count++;
    //通知队列非空，那些因为获取元素而阻塞的线程可以继续工作了
    notEmpty.signal();
}
~~~

执行步骤为

1. 获取 `ArrayBlockingQueue` 底层的数组 `items`。
2. 将元素存到 `putIndex` 位置。
3. 更新 `putIndex` 到下一个位置，如果 `putIndex` 等于队列长度，则说明 `putIndex` 已经到达数组末尾了，下一次插入则需要 0 开始。(`ArrayBlockingQueue` 用到了循环队列的思想，即从头到尾循环复用一个数组)
4. 更新 `count` 的值，表示当前队列长度+1。
5. 调用 `notEmpty.signal()` 通知队列非空，消费者可以从队列中获取值了。

非阻塞式获取和新增

1. `offer(E e)`：将元素插入队列尾部。如果队列已满，则该方法会直接返回 false，不会等待并阻塞线程。
2. `poll()`：获取并移除队列头部的元素，如果队列为空，则该方法会直接返回 null，不会等待并阻塞线程。
3. `add(E e)`：将元素插入队列尾部。如果队列已满则会抛出 `IllegalStateException` 异常，底层基于 `offer(E e)` 方法。
4. `remove()`：移除队列头部的元素，如果队列为空则会抛出 `NoSuchElementException` 异常，底层基于 `poll()`。
5. `peek()`：获取但不移除队列头部的元素，如果队列为空，则该方法会直接返回 null，不会等待并阻塞线程。

### 相关面试题目

#### ArrayBlockingQueue是什么，特点是什么

`ArrayBlockingQueue` 是 `BlockingQueue` 接口的有界队列实现类，常用于多线程之间的数据共享，底层采用数组实现。

==`ArrayBlockingQueue` 的容量有限，一旦创建，容量不能改变。==

为了保证线程安全，`ArrayBlockingQueue` 的并发控制采用可重入锁 `ReentrantLock` ，不管是插入操作还是读取操作，都需要获取到锁才能进行操作。并且，它还支持公平和非公平两种方式的锁访问机制，默认是非公平锁。

`ArrayBlockingQueue` 虽名为阻塞队列，但也支持非阻塞获取和新增元素（例如 `poll()` 和 `offer(E e)` 方法），==只是队列满时添加元素会抛出异常，队列为空时获取的元素为 null==

#### ArrayBlockingQueue 和 LinkedBlockingQueue 有什么区别？

底层数据结构不一样，是否拥有边界界限

锁是否分离：`ArrayBlockingQueue`中的锁是没有分离的，即生产和消费用的是同一个锁；`LinkedBlockingQueue`中的锁是分离的，即生产用的是`putLock`，消费是`takeLock`，这样可以防止生产者和消费者线程之间的锁争夺。

内存占用不一样

#### ArrayBlockingQueue 的实现原理

1. `ArrayBlockingQueue` 内部维护一个定长的数组用于存储元素。
2. 通过使用 `ReentrantLock` 锁对象对读写操作进行同步，即通过锁机制来实现线程安全。
3. 通过 `Condition` 实现线程间的等待和唤醒操作。



### Map

### HashMap和Hashtable的区别

线程安全性：HashMap是不安全的，HashTable是安全的，`Hashtable` 内部的方法基本都经过`synchronized` 修饰。

**效率：** 因为线程安全的问题，`HashMap` 要比 `Hashtable` 效率高一点。

HashMap**对 Null key 和 Null value 支持**，Hashtable则会抛出异常

**初始容量大小和每次扩充容量大小的不同**：HashMap默认大小为16，扩容策略为2倍。如果不使用默认大小容量，`HashMap` 会将其扩充为 2 的幂次方大小

这个方法保证了大小为2的幂次方。先将cap-1，通过连续进行按位或和无符号右移操作，确保在目标容量的二进制表示中，除了最高位的1之外，所有的位都被设置为1，最后加一返回就变成了2的幂次方了

~~~java
    /**
     * Returns a power of two size for the given target capacity.
     */
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }

~~~

**底层数据结构：** `HashMap` 在解决哈希冲突时有了较大的变化，当链表长度大于阈值（默认为 8）时，将链表转化为红黑树（将链表转换成红黑树前会判断，如果当前数组的长度小于 64，那么会选择先进行数组扩容，而不是转换为红黑树），以减少搜索时

### 和TreeMap的区别

`TreeMap`它还实现了`NavigableMap`接口和`SortedMap` 接口。

实现 `NavigableMap` 接口让 `TreeMap` 有了对集合内元素的搜索的能力。

基于红黑树数据结构的属性实现的，红黑树保持平衡状态，从而保证了搜索操作的时间复杂度为 O(log n)

实现`SortedMap`接口让 `TreeMap` 有了对集合中的元素根据键排序的能力。

**相比于`HashMap`来说， `TreeMap` 主要多了对集合中的元素根据键排序的能力以及对集合内元素的搜索的能力**

### HashSet如何查重

在 JDK1.8 中，`HashSet`的`add()`方法只是简单的调用了**HashMap**的`put()`方法，并且判断了一下返回值以确保是否有重复元素。

~~~java
// Returns: true if this set did not already contain the specified element
// 返回值：当 set 中没有包含 add 的元素时返回真
public boolean add(E e) {
        return map.put(e, PRESENT)==null;
}
~~~

JDK1.8 中，实际上无论`HashSet`中是否已经存在了某元素，`HashSet`都会直接插入，只是会在`add()`方法的返回值处告诉我们插入前是否存在相同元素。

### HashMap底层实现

JDK1.8 之前 `HashMap` 底层是 **数组和链表** 结合在一起使用也就是 **链表散列**，得到key的hash值，

然后通过 `(n - 1) & hash` 判断当前元素存放的位置。（n 指的是数组的长度，因为n等于2的幂次方，所以n为111...）

并且Hash 值的范围值-2147483648 到 2147483647太大了，数组放不下，所以采用Hash%n的操作

 **hash%length==hash&(length-1)的前提是 length 是 2 的 n 次方”** 并且 **采用二进制位操作 &，相对于%能够提高运算效率，这就解释了 HashMap 的长度为什么是 2 的幂次方。**

jdk1.8的底层，当链表长度大于8时就会转换成红黑树

![jdk1.8之后的内部结构-HashMap](https://oss.javaguide.cn/github/javaguide/java/collection/jdk1.8_hashmap.png)

多线程下建议不要使用hashMap来操作数据，可能会出现覆盖数据的问题。并发环境下，推荐使用 `ConcurrentHashMap` 

举个例子

两个线程 1,2 同时进行 put 操作，并且发生了哈希冲突（hash 函数计算出的插入下标是相同的）。

不同的线程可能在不同的时间片获得 CPU 执行的机会，当前线程 1 执行完哈希冲突判断后，由于时间片耗尽挂起。线程 2 先完成了插入操作。

随后，线程 1 获得时间片，由于之前已经进行过 hash 碰撞的判断，所有此时会直接进行插入，这就导致线程 2 插入的数据被线程 1 覆盖了。

~~~java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
    // ...
    // 判断是否出现 hash 碰撞
    // (n - 1) & hash 确定元素存放在哪个桶中，桶为空，新生成结点放入桶中(此时，这个结点是放在数组中)
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    // 桶中已经存在元素（处理hash冲突）
    else {
    // ...
}
~~~

还有一种情况是这两个线程同时 `put` 操作导致 `size` 的值不正确，进而导致数据覆盖的问题：

1. 线程 1 执行 `if(++size > threshold)` 判断时，假设获得 `size` 的值为 10，由于时间片耗尽挂起。

2. 线程 2 也执行 `if(++size > threshold)` 判断，获得 `size` 的值也为 10，并将元素插入到该桶位中，并将 `size` 的值更新为 11。

3. 随后，线程 1 获得时间片，它也将元素放入桶位中，并将 size 的值更新为 11。

4. 线程 1、2 都执行了一次 `put` 操作，但是 `size` 的值只增加了 1，也就导致实际上只有一个元素被添加到了 `HashMap` 中

   ~~~java
   public V put(K key, V value) {
       return putVal(hash(key), key, value, false, true);
   }
   
   final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                      boolean evict) {
       // ...
       // 实际大小大于阈值则扩容
       if (++size > threshold)
           resize();
       // 插入后回调
       afterNodeInsertion(evict);
       return null;
   }
   ~~~

### ConcurrentHashMap 和 Hashtable 的区别

ConcurrentHashMap 底层数据结构和HashMap一样，Hashtable 则是数组加链表

**实现线程安全的方式（重要）**

**`Hashtable`(同一把锁)** :使用 `synchronized` 来保证线程安全，效率非常低下。当一个线程访问同步方法时，其他线程也访问同步方法，可能会进入阻塞或轮询状态，如使用 put 添加元素，另一个线程不能使用 put 添加元素，也不能使用 get，竞争会越来越激烈效率越低。

到了 JDK1.8 的时候，`ConcurrentHashMap` 已经摒弃了 `Segment`（分段锁，只锁容器的一部分数据） 的概念，而是直接用 `Node` 数组+链表+红黑树的数据结构来实现，并发控制使用 `synchronized` 和 CAS 来操作。（JDK1.6 以后 `synchronized` 锁做了很多优化） 整个看起来就像是优化过且线程安全的 `HashMap`，虽然在 JDK1.8 中还能看到 `Segment` 的数据结构，但是已经简化了属性，只是为了兼容旧版本；

#### ConcurrentHashMap线程安全具体实现

#### JDK1.8以前

![Java7 ConcurrentHashMap 存储结构](https://oss.javaguide.cn/github/javaguide/java/collection/java7_concurrenthashmap.png)

**`ConcurrentHashMap` 是由 `Segment` 数组结构和 `HashEntry` 数组结构组成**

`Segment` 继承了 `ReentrantLock`,所以 `Segment` 是一种可重入锁，扮演锁的角色。`HashEntry` 用于存储键值对数据。

`Segment` 的个数一旦**初始化就不能改变**。 `Segment` 数组的大小默认是 16，也就是说默认可以同时支持 16 个线程并发写。

一个 `Segment` 包含一个 `HashEntry` 数组每个 `Segment` 守护着一个 `HashEntry` 数组里的元素，当对 `HashEntry` 数组的数据进行修改时，必须首先获得对应的 `Segment` 的锁。也就是说，对同一 `Segment` 的并发写入会被阻塞，不同 `Segment` 的写入是可以并发执行的。

#### JDK1.8 之后

![Java8 ConcurrentHashMap 存储结构](https://oss.javaguide.cn/github/javaguide/java/collection/java8_concurrenthashmap.png)

`ConcurrentHashMap` 取消了 `Segment` 分段锁，采用 `Node + CAS + synchronized` 来保证并发安全。

`synchronized` 只锁定当前链表或红黑二叉树的首节点，这样只要 hash 不冲突，就不会产生并发，就不会影响其他 Node 的读写，效率大幅提升。

CAS操作：

CAS是Compare And Swap的缩写，即比较与交换，通常指的==是一种原子操作==，它用于解决多线程并发访问共享数据时的线程安全问题。CAS操作包含三个操作数：需要修改的内存位置V、预期原值A和新值B。当且仅当预期原值与内存位置的当前值相同时，才会将内存位置的值修改为新值；否则，CAS操作会失败，并返回内存位置的当前值。

**Java 语言并没有直接实现 CAS，。因此， CAS 的具体实现和操作系统以及 CPU 都有关系。**

**`sun.misc`包下的`Unsafe`类提供了`compareAndSwapObject`、`compareAndSwapInt`、`compareAndSwapLong`方法来实现的对`Object`、`int`、`long`类型的 CAS 操作**

JDK1.8相比于JDK1.7并发度更大（为Node数组大小），线程安全实现方式不一样，Hash碰撞解决方法不一样，1.7为拉链法，1.8为拉链法结合红黑树。**并且他们都不能存储null值key和value，会产生二义性（get返回null无法判断是否有）**

`ConcurrentHashMap` 提供了一些原子性的复合操作，如 `putIfAbsent`、`compute`、`computeIfAbsent` 、`computeIfPresent`、`merge`等。

可以优化这些操作

~~~java
// 线程 A
if (!map.containsKey(key)) {
map.put(key, value);
}
// 线程 B
if (!map.containsKey(key)) {
map.put(key, anotherValue);
}
--------------------------------------
    // 线程 A
map.putIfAbsent(key, value);
// 线程 B
map.putIfAbsent(key, anotherValue);

~~~



## Java并发面试

### 线程和进程定义

进程是系统运行程序的基本单位。

当我们启动 main 函数时其实就是启动了一个 JVM 的进程，而 main 函数所在的线程就是这个进程中的一个线程，也称主线程。

线程为一个更小的执行单位。

与进程不同的是同类的多个线程共享进程的**堆**和**方法区**资源，但每个线程有自己的**程序计数器**、**虚拟机栈**和**本地方法栈**，线程切换工作负担比进程小

### Java线程和操作系统的线程区别

在 JDK 1.2 及以后，Java 线程改为基于原生线程（Native Threads）实现，也就是说 JVM 直接使用操作系统原生的内核级线程（内核线程）来实现 Java 线程，由操作系统内核进行线程的调度和管理

### 用户线程和内核线程区别

- 用户线程：由用户空间程序管理和调度的线程，运行在用户空间（专门给应用程序使用）。
- 内核线程：由操作系统内核管理和调度的线程，运行在内核空间（只有内核程序可以访问）

用户线程和内核线程的区别和特点：用户线程创建和切换成本低，但不可以利用多核。内核态线程，创建和切换成本高，可以利用多核。

### 线程模型是用户线程和内核线程之间的关联方式，常见的线程模型有这三种：

1. 一对一（一个用户线程对应一个内核线程）
2. 多对一（多个用户线程映射到一个内核线程）
3. 多对多（多个用户线程映射到多个内核线程）

![常见的三种线程模型](https://oss.javaguide.cn/github/javaguide/java/concurrent/three-types-of-thread-models.png)

Java线程采用的是一对一的线程模型

### java内存区域

<img src="https://oss.javaguide.cn/github/javaguide/java/jvm/java-runtime-data-areas-jdk1.8.png" alt="Java 运行时数据区域（JDK1.8 之后）" style="zoom: 67%;" />

一个进程中可以有多个线程，多个线程共享进程的**堆**和**方法区 (JDK1.8 之后的元空间)资源，但是每个线程有自己的\**程序计数器**、**虚拟机栈** 和 **本地方法栈**。

#### 程序计数器**、**虚拟机栈和 **本地方法栈**为什么私有

程序计数器用于记录当前线程执行的位置，依次读取指令。程序计数器私有主要是为了**线程切换后能恢复到正确的执行位置**

**虚拟机栈：** 每个 Java 方法在执行之前会创建一个栈帧用于存储局部变量表、操作数栈、常量池引用等信息。从方法调用直至执行完成的过程，就对应着一个栈帧在 Java 虚拟机栈中入栈和出栈的过程。

**本地方法栈：** 和虚拟机栈所发挥的作用非常相似，区别是：**虚拟机栈为虚拟机执行 Java 方法 （也就是字节码）服务，而本地方法栈则为虚拟机使用到的 Native 方法服务。** 在 HotSpot 虚拟机中和 Java 虚拟机栈合二为一。

为了**保证线程中的局部变量不被别的线程访问到**，虚拟机栈和本地方法栈是线程私有的。

### 堆和方法区定义

堆和方法区是所有线程共享的资源，堆是最大的一块内存，主要用于存放新的创建对象。

方法区主要用于存放已被加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。

异步就是调用发出之后，不用等待结果就可以直接返回

### 为什么使用多线程

- **从计算机底层来说：** 线程可以比作是轻量级的进程，是程序执行的最小单位,线程间的切换和调度的成本远远小于进程。另外，多核 CPU 时代意味着多个线程可以同时运行，这减少了线程上下文切换的开销。

​	并且可以提高Cpu的使用率

线程安全需要在多线程环境下保证数据访问的正确性和一致性

#### 线程的类型和任务的性质分为两种

Cpu密集型和IO密集型

#### 严格来说，Java 就只有一种方式可以创建线程，那就是通过`new Thread().start()`创建。不管是哪种方式，最终还是依赖于`new Thread().start()`。

### 线程生命周期和状态

有6种状态：运行(Thread.run)，等待(wait)，终止(执行完run后)，初始（Thread.start），超时等待(sleep)，

阻塞（当线程进入 `synchronized` 方法/块或者调用 `wait` 后（被 `notify`）重新进入 `synchronized` 方法/块，但是锁被其它线程占有，这个时候线程就会进入 **BLOCKED（阻塞）** 状态。

<img src="https://oss.javaguide.cn/github/javaguide/java/concurrent/640.png" alt="Java 线程状态变迁图" style="zoom:67%;" />

在操作系统层面，线程有 READY 和 RUNNING 状态；而在 JVM 层面，只能看到 RUNNABLE 状态

JVM没有去分这两种状态，是因为线程得到的时间分片非常小，时间片用后就要被切换下来放入调度队列的末尾等待再次调度。（也即回到 ready 状态）

### 线程上下文切换

线程从占用 CPU 状态中退出的情况：

1.cpu时间片用完

2.主动让出CPU，调用sleep，wait等

3.调用了阻塞类型的系统中断，例如请求IO

4.被结束或终止

线程上下文意味着下次切换回线程是恢复线程的运行现场

### 线程死锁

四个条件

1.互斥条件：资源任意时刻只有一个线程占用

2.不可抢夺

3.循环等待

4.请求与保持：一个线程因请求资源而阻塞时，对已获得的资源保持不放。

#### 预防和避免死锁

破坏死锁条件

**破坏请求与保持条件**：一次性申请所有的资源。

**破坏不剥夺条件**：占用部分资源的线程进一步申请其他资源时，如果申请不到，可以主动释放它占有的资源。

**破坏循环等待条件**：靠按序申请资源来预防。按某一顺序申请资源，释放资源则反序释放。破坏循环等待条件。



#### 避免死锁

避免死锁就是在资源分配时，借助于算法（比如银行家算法）对资源分配进行计算评估，使其进入安全状态。

### sleep和wait方法对比

**共同点**：两者都可以暂停线程的执行。

**区别**：

- **`sleep()` 方法没有释放锁，而 `wait()` 方法释放了锁** 。
- `wait()` 通常被用于线程间交互/通信，`sleep()`通常被用于暂停执行。
- `wait()` 方法被调用后，线程不会自动苏醒，需要别的线程调用同一个对象上的 `notify()`或者 `notifyAll()` 方法。`sleep()`方法执行完成后，线程会自动苏醒，或者也可以使用 `wait(long timeout)` 超时后线程会自动苏醒。
- `sleep()` 是 `Thread` 类的静态本地方法，`wait()` 则是 `Object` 类的本地方法。因为object类方法可以使线程种的对象放弃对象锁，而thread类只是让当前线程暂停

如果直接使用Thread.run并不会以多线程执行，只有start是启动多线程

## JMM（Java 内存模型）详解

JMM(Java 内存模型)主要定义了对于一个共享变量，当另一个线程对这个共享变量执行写操作后，这个线程对这个共享变量的可见性。

## 从 CPU 缓存模型说起

 **CPU 缓存则是为了解决 CPU 处理速度和内存处理速度不对等的问题。内存缓存的是硬盘数据用于解决硬盘访问速度过慢的问题。**

## 指令重排序

简单来说就是系统在执行代码的时候并不一定是按照你写的代码的顺序依次执行。

有两种情况

**编译器优化重排**：编译器（包括 JVM、JIT 编译器等）在不改变单线程程序语义的前提下，重新安排语句的执行顺序。

**指令并行重排**：现代处理器采用了指令级并行技术(Instruction-Level Parallelism，ILP)来将多条指令重叠执行。如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序。

### JMM（Java Memory Model）java内存模型

#### JMM如何抽象线程和主内存之间的关系

在 JDK1.2 之前，Java 的内存模型实现总是从 **主存** （即共享内存）读取变量，是不需要进行特别的注意的。而在当前的 Java 内存模型下，线程可以把变量保存 **本地内存** （比如机器的寄存器）中，而不是直接在主存中进行读写。

**什么是主内存？什么是本地内存？**

**主内存**：所有线程创建的实例对象都存放在主内存中，不管该实例对象是成员变量，还是局部变量，类信息、常量、静态变量都是放在主内存中。

**本地内存**：每个线程都有一个私有的本地内存，本地内存存储了该线程以读 / 写共享变量的副本

JMM内存模型

![JMM(Java 内存模型)](https://oss.javaguide.cn/github/javaguide/java/concurrent/jmm.png)

线程 1 与线程 2 之间如果要进行通信的话，必须要经历下面 2 个步骤：

1. 线程 1 把本地内存中修改过的共享变量副本的值同步到主内存中去。
2. 线程 2 到主存中读取对应的共享变量的值。

关于主内存与工作内存直接的具体交互协议，即一个变量如何从主内存拷贝到工作内存，如何从工作内存同步到主内存之间的实现细节，Java 内存模型定义来以下八种同步操作（了解即可，无需死记硬背）：

- **锁定（lock）**: 作用于主内存中的变量，将他标记为一个线程独享变量。

- **解锁（unlock）**: 作用于主内存中的变量，解除变量的锁定状态，被解除锁定状态的变量才能被其他线程锁定。

- **read（读取）**：作用于主内存的变量，它把一个变量的值从主内存传输到线程的工作内存中，以便随后的 load 动作使用。

- **load(载入)**：把 read 操作从主内存中得到的变量值放入工作内存的变量的副本中。

- **use(使用)**：把工作内存中的一个变量的值传给执行引擎，每当虚拟机遇到一个使用到变量的指令时都会使用该指令。

- **assign（赋值）**：作用于工作内存的变量，它把一个从执行引擎接收到的值赋给工作内存的变量，每当虚拟机遇到一个给变量赋值的字节码指令时执行这个操作。

- **store（存储）**：作用于工作内存的变量，它把工作内存中一个变量的值传送到主内存中，以便随后的 write 操作使用。

- **write（写入）**：作用于主内存的变量，它把 store 操作从工作内存中得到的变量的值放入主内存的变量中。

  **除了这 8 种同步操作之外，还规定了下面这些同步规则来保证这些同步操作的正确执行（了解即可，无需死记硬背）：**

  - 不允许一个线程无原因地（没有发生过任何 assign 操作）把数据从线程的工作内存同步回主内存中。
  - 一个新的变量只能在主内存中 “诞生”，不允许在工作内存中直接使用一个未被初始化（load 或 assign）的变量，换句话说就是对一个变量实施 use 和 store 操作之前，必须先执行过了 assign 和 load 操作。
  - 一个变量在同一个时刻只允许一条线程对其进行 lock 操作，但 lock 操作可以被同一条线程重复执行多次，多次执行 lock 后，只有执行相同次数的 unlock 操作，变量才会被解锁。
  - 如果对一个变量执行 lock 操作，将会清空工作内存中此变量的值，在执行引擎使用这个变量前，需要重新执行 load 或 assign 操作初始化变量的值。
  - 如果一个变量事先没有被 lock 操作锁定，则不允许对它执行 unlock 操作，也不允许去 unlock 一个被其他线程锁定住的变量

### [Java 内存区域和 JMM 有何区别？](#java-内存区域和-jmm-有何区别)

 **Java 内存区域和内存模型是完全不一样的两个东西**：

- JVM 内存结构和 Java 虚拟机的运行时区域相关，定义了 JVM 在运行时如何分区存储程序数据，就比如说堆主要用于存放对象实例。
- **Java 内存模型和 Java 的并发编程相关，抽象了线程和主内存之间的关系就比如说线程之间的共享变量必须存储在主内存中，规定了从 Java 源代码到 CPU 可执行指令的这个转化过程要遵守哪些和并发相关的原则和规范**，其主要目的是为了简化多线程编程，增强程序可移植性的。

### happens-before原则是什么

**逻辑时钟并不度量时间本身，仅区分事件发生的前后顺序，其本质就是定义了一种 happens-before 关系。**

**原则设计思想**

为了对编译器和处理器的约束尽可能少，只要不改变程序的执行结果（单线程程序和正确执行的多线程程序），编译器和处理器怎么进行重排序优化都行。

对于会改变程序执行结果的重排序，JMM 要求编译器和处理器必须禁止这种重排序。

**原则定义**

如果一个操作 happens-before 另一个操作，那么第一个操作的执行结果将对第二个操作可见，并且第一个操作的执行顺序排在第二个操作之前。

两个操作之间存在 happens-before 关系，并不意味着 Java 平台的具体实现必须要按照 happens-before 关系指定的顺序来执行。如果重排序之后的执行结果，与按 happens-before 关系来执行的结果一致，那么 JMM 也允许这样的重排序。

**happens-before 原则表达的意义其实并不是一个操作发生在另外一个操作的前面，虽然这从程序员的角度上来说也并无大碍。更准确地来说，它更想表达的意义是前一个操作的结果对于后一个操作是可见的，无论这两个操作是否在同一个线程里。**

**原则常见规则**

1. **程序顺序规则**：一个线程内，按照代码顺序，书写在前面的操作 happens-before 于书写在后面的操作；
2. **解锁规则**：解锁 happens-before 于加锁；
3. **volatile 变量规则**：对一个 volatile 变量的写操作 happens-before 于后面对这个 volatile 变量的读操作。说白了就是对 volatile 变量的写操作的结果对于发生于其后的任何操作都是可见的。
4. **传递规则**：如果 A happens-before B，且 B happens-before C，那么 A happens-before C；
5. **线程启动规则**：Thread 对象的 `start()`方法 happens-before 于此线程的每一个动作。

**happen-before和JMM的关系**

![happens-before 与 JMM 的关系](https://oss.javaguide.cn/github/javaguide/java/concurrent/image-20220731084604667.png)

并发编程三个特性

原子性，可见性，有序性

### 总结

- Java 是最早尝试提供内存模型的语言，其主要目的是为了简化多线程编程，增强程序可移植性的。
- CPU 可以通过制定缓存一致协议（比如 [MESI 协议open in new window](https://zh.wikipedia.org/wiki/MESI协议)）来解决内存缓存不一致性问题。
- **指令重排序可以保证串行语义一致，但是没有义务保证多线程间的语义也一致** ，所以在多线程下，指令重排序可能会导致一些问题。
- 你可以把 JMM 看作是 Java 定义的并发编程相关的一组规范，除了抽象了线程和主内存之间的关系之外，其还规定了从 Java 源代码到 CPU 可执行指令的这个转化过程要遵守哪些和并发相关的原则和规范，其主要目的是为了简化多线程编程，增强程序可移植性的。
- JSR 133 引入了 happens-before 这个概念来描述两个操作之间的内存可见性。

## volatile关键字

### 保证变量的可见性

如果我们将变量声明为 **`volatile`** ，这就指示 JVM，这个变量是共享且不稳定的，每次使用它都到主存中进行读取。会强制线程每次都从主内存中读取

<img src="https://oss.javaguide.cn/github/javaguide/java/concurrent/jmm.png" alt="JMM(Java 内存模型)" style="zoom:50%;" />

<img src="https://oss.javaguide.cn/github/javaguide/java/concurrent/jmm2.png" alt="JMM(Java 内存模型)强制在主存中进行读取" style="zoom:50%;" />

`volatile` 关键字能保证数据的可见性，但不能保证数据的原子性。除了保证数据可见性还可以**防止 JVM 的指令重排序。**

`volatile` 关键字禁止指令重排序的效果

**双重校验锁实现对象单例（线程安全）**：

~~~java
public class Singleton {

    private volatile static Singleton uniqueInstance;

    private Singleton() {
    }

    public  static Singleton getUniqueInstance() {
       //先判断对象是否已经实例过，没有实例化过才进入加锁代码
        if (uniqueInstance == null) {
            //类对象加锁
            synchronized (Singleton.class) {
                if (uniqueInstance == null) {
                    uniqueInstance = new Singleton();
                }
            }
        }
        return uniqueInstance;
    }
}
~~~

 `uniqueInstance = new Singleton();` 这段代码分为三步执行：

1. 为 `uniqueInstance` 分配内存空间
2. 初始化 `uniqueInstance`
3. 将 `uniqueInstance` 指向分配的内存地址

但是由于 JVM 具有指令重排的特性，执行顺序有可能变成 1->3->2。指令重排在单线程环境下不会出现问题，但是在多线程环境下会导致一个线程获得还没有初始化的实例。例如，线程 T1 执行了 1 和 3，此时 T2 调用 `getUniqueInstance`() 后发现 `uniqueInstance` 不为空，因此返回 `uniqueInstance`，但此时 `uniqueInstance` 还未被初始化。

### 悲观锁

**共享资源每次只给一个线程使用，其它线程阻塞，用完后再把资源转让给其它线程**。`synchronized`和`ReentrantLock`等独占锁就是悲观锁思想的实现。

### 乐观锁

线程可以不停地执行，无需加锁也无需等待，只是在提交修改的时候去验证对应的资源（也就是数据）是否被其它线程修改了（版本号机制或CAS算法）（`AtomicInteger`、`LongAdder`）就是使用了乐观锁的一种实现方式 **CAS** 实现的。

悲观锁通常多用于写比较多的情况（多写场景，竞争激烈），乐观锁通常多用于写比较少的情况（多读场景，竞争较少）。不过，乐观锁主要针对的对象是单个共享变量

**乐观锁实现的CAS有什么问题**

**ABA问题，误认为A值没有被修改过。解决思路是可以加上版本号机制**

JDK 1.5 以后的 `AtomicStampedReference` 类就是用来解决 ABA 问题的，其中的 `compareAndSet()` 方法就是首先检查当前引用是否等于预期引用，并且当前标志是否等于预期标志，如果全部相等，则以原子方式将该引用和该标志的值设置为给定的更新值。

**循环时间开销大**

CAS 经常会用到自旋操作来进行重试，也就是不成功就一直循环执行直到成功。一直不成功的话，会增加系统开销。

**只能保证一个共享变量的原子操作**

### synchronized关键字

主要解决的是多个线程之间访问资源的同步性，可以保证被它修饰的方法或者代码块在任意时刻只能有一个线程执行。

主要来修饰

1. 实例方法：进入同步代码前要获得 **当前对象实例的锁** 。

2. 静态方法：进入同步代码前要获得 **当前 class 的锁**。给类加锁，影响的是静态同步方法的访问，

   这是因为静态成员不属于任何一个实例对象，归整个类所有，不依赖于类的特定实例，被类的所有实例共享。

   静态 `synchronized` 方法和非静态 `synchronized` 方法之间的调用不会互斥。对象实例锁和类锁是分开的

3. 代码块：锁指定对象/类

   - `synchronized(object)` 表示进入同步代码库前要获得 **给定对象的锁**。
   - `synchronized(类.class)` 表示进入同步代码前要获得 **给定 Class 的锁**

总结：

- `synchronized` 关键字加到 `static` 静态方法和 `synchronized(class)` 代码块上都是是给 Class 类上锁；
- `synchronized` 关键字加到实例方法上是给对象实例上锁；
- 尽量不要使用 `synchronized(String a)` 因为 JVM 中，字符串常量池具有缓存功能。

**构造方法不能使用synchronized 关键字修饰。**，因为构造方法本身是线程安全的。

#### 底层原理

~~~java
//示例代码
//修饰同步代码块
public class SynchronizedDemo {
    public void method() {
        synchronized (this) {
            System.out.println("synchronized 代码块");
        }
    }
}
//下面是字节码
~~~

![image-20240420122844002](C:\Users\15282\AppData\Roaming\Typora\typora-user-images\image-20240420122844002.png)

`synchronized` 同步语句块的实现使用的是 `monitorenter` 和 `monitorexit` 指令，其中 `monitorenter` 指令指向同步代码块的开始位置，`monitorexit` 指令则指明同步代码块的结束位置。

字节码中包含一个 `monitorenter` 指令以及两个 `monitorexit` 指令，这是为了保证锁在同步代码块代码正常执行以及出现异常的这两种情况下都能被正确释放。

在执行`monitorenter`时，会尝试获取对象的锁，如果锁的计数器为 0 则表示锁可以被获取，获取后将锁计数器设为 1 也就是加 1。

<img src="https://oss.javaguide.cn/github/javaguide/java/concurrent/synchronized-get-lock-code-block.png" alt="执行 monitorenter 获取锁" style="zoom: 67%;" />

只有锁对象拥有者可以在执行 `monitorexit` 指令后，将锁计数器设为 0，表明锁被释放，其他线程可以尝试获取锁。

~~~java
//修饰方法
public class SynchronizedDemo2 {
    public synchronized void method() {
        System.out.println("synchronized 方法");
    }
}
~~~

![image-20240420123427010](C:\Users\15282\AppData\Roaming\Typora\typora-user-images\image-20240420123427010.png)

修饰的方法并没有 `monitorenter` 指令和 `monitorexit` 指令，用的是`ACC_SYNCHRONIZED` 标识。该标识指明了该方法是一个同步方法，由JVM来判别标识。

注意锁可以升级不可降级，这种策略是为了提高获得锁和释放锁的效率。

#### `synchronized` 关键字和 `volatile` 关键字的区别

两者是互补的存在

- `volatile` 关键字是线程同步的轻量级实现，所以 `volatile`性能肯定比`synchronized`关键字要好 。但是 `volatile` 关键字只能用于变量而 `synchronized` 关键字可以修饰方法以及代码块 。
- `volatile` 关键字能保证数据的可见性，但不能保证数据的原子性。`synchronized` 关键字两者都能保证。
- `volatile`关键字主要用于解决变量在多个线程之间的可见性，而 `synchronized` 关键字解决的是多个线程之间访问资源的同步性。

#### ReentrantLock定义

`ReentrantLock` 实现了 `Lock` 接口，是一个可重入且独占式的锁，比synchronized更灵活和强大

ReentrantLock` 里面有一个内部类 `Sync，主要锁的操作在这个类实现。这个类有公平锁和非公平锁两个子类

`ReentrantLock` 默认使用非公平锁，也可以通过构造器来显式的指定使用公平锁。

~~~java
// 传入一个 boolean 值，true 时为公平锁，false 时为非公平锁
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
~~~

### [公平锁和非公平锁有什么区别？](#公平锁和非公平锁有什么区别)

- **公平锁** : 锁被释放之后，先申请的线程先得到锁。性能较差一些，因为公平锁为了保证时间上的绝对顺序，上下文切换更频繁。
- **非公平锁**：锁被释放之后，后申请的线程可能会先获取到锁，是随机或者按照其他优先级排序的。性能更好，但可能会导致某些线程永远无法获取到锁

### [synchronized 和 ReentrantLock 有什么区别？](https://javaguide.cn/java/concurrent/java-concurrent-questions-02.html#synchronized-和-reentrantlock-有什么区别)

#### 两者都是可重入锁

**可重入锁** 也叫递归锁，指的是线程可以再次获取自己的内部锁

#### synchronized 依赖于 JVM 而 ReentrantLock 依赖于 API

`ReentrantLock` 是 JDK 层面实现的（也就是 API 层面，需要 lock() 和 unlock() 方法配合 try/finally 语句块来完成

`ReentrantLock`增加了一些高级功能。

- **等待可中断** : `ReentrantLock`提供了一种能够中断等待锁的线程的机制，也就是说正在等待的线程可以选择放弃等待，改为处理其他事情。
- **可实现公平锁** : `ReentrantLock`可以指定是公平锁还是非公平锁。而`synchronized`只能是非公平锁。所谓的公平锁就是先等待的线程先获得锁。`ReentrantLock`默认情况是非公平的，可以通过 `ReentrantLock`类的`ReentrantLock(boolean fair)`构造方法来指定是否是公平的。
- **可实现选择性通知（锁可以绑定多个条件）**: `synchronized`关键字与`wait()`和`notify()`/`notifyAll()`方法相结合可以实现等待/通知机制。`ReentrantLock`类当然也可以实现，但是需要借助于`Condition`接口与`newCondition()`方法

关于 `Condition`接口的补充：

> `Condition`是 JDK1.5 之后才有的，可以实现多路通知功能也就是在一个`Lock`对象中可以创建多个`Condition`实例（即对象监视器），**线程对象可以注册在指定的`Condition`中，从而可以有选择性的进行线程通知，在调度线程上更加灵活。 在使用`notify()/notifyAll()`方法进行通知时，被通知的线程是由 JVM 选择的，用`ReentrantLock`类结合`Condition`实例可以实现“选择性通知”** ，而`synchronized`关键字就相当于整个 `Lock` 对象中只有一个`Condition`实例，所有的线程都注册在它一个身上。如果执行`notifyAll()`方法的话就会通知所有处于等待状态的线程，这样会造成很大的效率问题。而`Condition`实例的`signalAll()`方法，只会唤醒注册在该`Condition`实例中的所有等待线程。

- **可中断锁**：获取锁的过程中可以被中断，不需要一直等到获取锁之后 才能进行其他逻辑处理。`ReentrantLock` 就属于是可中断锁。
- **不可中断锁**：一旦线程申请了锁，就只能等到拿到锁以后才能进行其他的逻辑处理。 `synchronized` 就属于是不可中断锁。

- **共享锁**：一把锁可以被多个线程同时获得。
- **独占锁**：一把锁只能被一个线程获得。

#### 线程有读锁情况下不能获取写锁

在线程持有读锁的情况下，该线程不能取得写锁(因为获取写锁的时候，如果发现当前的读锁被占用，就马上获取失败，不管读锁是不是被当前线程持有)。

在线程持有写锁的情况下，该线程可以继续获取读锁（获取读锁时如果发现写锁被占用，只有写锁没有被当前线程占用的情况才会获取失败）

### ThreadLocal

实现每个线程有自身的本地变量。

#### 原理

从 `Thread`类源代码入手。

~~~java
public class Thread implements Runnable {
    //......
    //与此线程有关的ThreadLocal值。由ThreadLocal类维护
    ThreadLocal.ThreadLocalMap threadLocals = null;

    //与此线程有关的InheritableThreadLocal值。由InheritableThreadLocal类维护
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
    //......
}
~~~

`Thread` 类中有一个 `threadLocals` 和 一个 `inheritableThreadLocals` 变量，它们都是 `ThreadLocalMap` 类型的变量。,我们可以把 `ThreadLocalMap` 理解为`ThreadLocal` 类实现的定制化的 `HashMap`。只有当前线程调用 `ThreadLocal` 类的 `set`或`get`方法时才创建它们（threadLocalMap）。

**因为`ThreadLocalMap`是`ThreadLocal`的静态内部类**

**`ThreadLocal`类的`set()`方法**

~~~java
public void set(T value) {
    //获取当前请求的线程
    Thread t = Thread.currentThread();
    //取出 Thread 类内部的 threadLocals 变量(哈希表结构)
    ThreadLocalMap map = getMap(t);
    if (map != null)
        // 将需要存储的值放入到这个哈希表中
        map.set(this, value);
    else
        createMap(t, value);
}
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
~~~

**最终的变量是放在了当前线程的 `ThreadLocalMap` 中，并不是存在 `ThreadLocal` 上，`ThreadLocal` 可以理解为只是`ThreadLocalMap`的封装，传递了变量值。**

**每个`Thread`中都具备一个`ThreadLocalMap`，而`ThreadLocalMap`可以存储以`ThreadLocal`为 key ，Object 对象为 value 的键值对。**

~~~java
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    //......
}
~~~

`ThreadLocalMap`的 key 就是 `ThreadLocal`对象，value 就是 `ThreadLocal` 对象调用`set`方法设置的值。

<img src="https://oss.javaguide.cn/github/javaguide/java/concurrent/threadlocal-data-structure.png" alt="ThreadLocal 数据结构" style="zoom:67%;" />



#### ThreadLocal的内存泄漏

`ThreadLocalMap` 中使用的 key 为 `ThreadLocal` 的弱引用，而 value 是强引用。所以，如果 `ThreadLocal` 没有被外部强引用的情况下，在垃圾回收的时候，key 会被清理掉，而 value 不会被清理掉。

这样一来value 永远无法被 GC 回收，这个时候就可能会产生内存泄露。`ThreadLocalMap` 实现中已经考虑了这种情况，在调用 `set()`、`get()`、`remove()` 方法的时候，会清理掉 key 为 null 的记录。使用完 `ThreadLocal`方法后最好手动调用`remove()`方法

**弱引用介绍：**

> 弱引用与软引用的区别在于：只具有弱引用的对象拥有更短暂的生命周期。在垃圾回收器线程扫描它 所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。不过，由于垃圾回收器是一个优先级很低的线程， 因此不一定会很快发现那些只具有弱引用的对象。
>
> 弱引用可以和一个引用队列（ReferenceQueue）联合使用，如果弱引用所引用的对象被垃圾回收，Java 虚拟机就会把这个弱引用加入到与之关联的引用队列中。

### 线程池

线程池就是管理一系列线程的资源池。当有任务要处理时，直接从线程池中获取线程来处理，处理完之后线程并不会立即被销毁，而是等待下一个任务。

池化技术的思想主要是为了减少每次获取资源的消耗，提高对资源的利用率。

线程池好处

- **降低资源消耗**。通过重复利用已创建的线程降低线程创建和销毁造成的消耗。
- **提高响应速度**。当任务到达时，任务可以不需要等到线程创建就能立即执行。
- **提高线程的可管理性**。线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，调优和监控。

#### 创建线程池

**方式一：通过`ThreadPoolExecutor`构造函数来创建（推荐）。**

![通过构造方法实现](https://javaguide.cn/assets/threadpoolexecutor%E6%9E%84%E9%80%A0%E5%87%BD%E6%95%B0-BR-2Ub-c.png)

**方式二：通过 `Executor` 框架的工具类 `Executors` 来创建。**

![img](https://oss.javaguide.cn/github/javaguide/java/concurrent/executors-new-thread-pool-methods.png)

1. `FixedThreadPool`：固定线程数量的线程池。
2. `SingleThreadExecutor`： 只有一个线程的线程池。若多余一个任务被提交到该线程池，任务会被保存在一个任务队列中，待线程空闲，按先入先出的顺序执行队列中的任务。
3. `CachedThreadPool`： 可根据实际情况调整线程数量的线程池。
4. `ScheduledThreadPool`：给定的延迟后运行任务或者定期执行任务的线程池

#### 线程池常见参数

- `corePoolSize` : 任务队列未达到队列容量时，最大可以同时运行的线程数量。
- `maximumPoolSize` : 任务队列中存放的任务达到队列容量的时候，当前可以同时运行的线程数量变为最大线程数。
- `workQueue`: 新任务来的时候会先判断当前运行的线程数量是否达到核心线程数，如果达到的话，新任务就会被存放在队列中。

~~~java
    /**
     * 用给定的初始参数创建一个新的ThreadPoolExecutor。
     */
    public ThreadPoolExecutor(int corePoolSize,//线程池的核心线程数量
                              int maximumPoolSize,//线程池的最大线程数
                              long keepAliveTime,//当线程数大于核心线程数时，多余的空闲线程存活的最长时间
                              TimeUnit unit,//时间单位
                              BlockingQueue<Runnable> workQueue,//任务队列，用来储存等待执行任务的队列
                              ThreadFactory threadFactory,//线程工厂，用来创建线程，一般默认即可
                              RejectedExecutionHandler handler//拒绝策略，当提交的任务过多而不能及时处理时，我们可以定制策略来处理任务
                               ) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
~~~

#### 饱和策略

如果当前同时运行的线程数量达到最大线程数量并且队列也已经被放满了任务时，`ThreadPoolExecutor` 定义一些策略:

- `ThreadPoolExecutor.AbortPolicy`：抛出 `RejectedExecutionException`来拒绝新任务的处理。这个是默认的
- `ThreadPoolExecutor.CallerRunsPolicy`：调用执行自己的线程运行任务，也就是直接在调用`execute`方法的线程中运行(`run`)被拒绝的任务，如果执行程序已关闭，则会丢弃该任务。因此这种策略会降低对于新任务提交速度，影响程序的整体性能。如果您的应用程序可以承受此延迟并且你要求任何一个任务请求都要被执行的话，你可以选择这个策略。
- `ThreadPoolExecutor.DiscardPolicy`：不处理新任务，直接丢弃掉。
- `ThreadPoolExecutor.DiscardOldestPolicy`：此策略将丢弃最早的未处理的任务请求。

#### 常用的阻塞队列

新任务来的时候会先判断当前运行的线程数量是否达到核心线程数，如果达到的话，新任务就会被存放在队列中。

- 容量为 `Integer.MAX_VALUE` 的 `LinkedBlockingQueue`（无界队列）：

  `FixedThreadPool` 和 `SingleThreadExector` 。

  `FixedThreadPool`最多只能创建核心线程数的线程（核心线程数和最大线程数相等），`SingleThreadExector`只能创建一个线程（核心线程数和最大线程数都是 1），二者的任务队列永远不会被放满。

- `SynchronousQueue`（同步队列）：`CachedThreadPool` 。`SynchronousQueue` 没有容量，不存储元素，目的是保证对于提交的任务，如果有空闲线程，则使用空闲线程来处理；否则新建一个线程来处理任务。也就是说，`CachedThreadPool` 的最大线程数是 `Integer.MAX_VALUE` ，可以理解为线程数是可以无限扩展的，可能会创建大量线程，从而导致 OOM。

- `DelayedWorkQueue`（延迟阻塞队列）：

  `ScheduledThreadPool` 和 `SingleThreadScheduledExecutor` 。

  `DelayedWorkQueue` 的内部元素并不是按照放入的时间排序，而是会按照延迟的时间长短对任务进行排序，内部采用的是“堆”的数据结构，可以保证每次出队的任务都是当前队列中执行时间最靠前的。`DelayedWorkQueue` 添加元素满了之后会自动扩容原来容量的 1/2，即永远不会阻塞，最大扩容可达 `Integer.MAX_VALUE`，所以最多只能创建核心线程数的线程。

#### 处理任务流程

![图解线程池实现原理](https://oss.javaguide.cn/github/javaguide/java/concurrent/thread-pool-principle.png)

#### 如何设定线程池的大小

如果我们设置线程数量太大，大量线程可能会同时在争取 CPU 资源，这样会导致大量的上下文切换，从而增加线程的执行时间，影响了整体执行效率

有一个简单并且适用面比较广的公式：

- **CPU 密集型任务(N+1)**： N（CPU 核心数）+1，比 CPU 核心数多出来的一个线程是为了防止线程偶发的缺页中断，或者其它原因导致的任务暂停而带来的影响。一旦任务暂停，CPU 就会处于空闲状态，而在这种情况下多出来的一个线程就可以充分利用 CPU 的空闲时间。
- **I/O 密集型任务(2N)：** 这种任务应用起来，系统会用大部分的时间来处理 I/O 交互，而线程在处理 I/O 的时间段内不会占用 CPU 来处理，这时就可以将 CPU 交出给其它线程使用。因此在 I/O 密集型任务的应用中，我们可以多配置一些线程，具体的计算方法是 2N。

CPU 密集型简单理解就是利用 CPU 计算能力的任务比如你在内存中对大量数据进行排序。但凡涉及到网络读取，文件读取这类都是 IO 密集型，

#### 动态修改线程池参数

![img](https://oss.javaguide.cn/github/javaguide/java/concurrent/threadpoolexecutor-methods.png)

格外需要注意的是`corePoolSize`， 程序运行期间的时候，我们调用 `setCorePoolSize()`这个方法的话，线程池会首先判断当前工作线程数是否大于`corePoolSize`，如果大于的话就会回收工作线程。

### [如何设计一个能够根据任务的优先级来执行的线程池？](https://javaguide.cn/java/concurrent/java-concurrent-questions-03.html#如何设计一个能够根据任务的优先级来执行的线程池)

假如我们需要实现一个优先级任务线程池的话，那可以考虑使用 `PriorityBlockingQueue` （优先级阻塞队列）作为任务队列。这个队列是线程安全的

![ThreadPoolExecutor构造函数](https://oss.javaguide.cn/github/javaguide/java/concurrent/common-parameters-of-threadpool-workqueue.jpg)

要想让 `PriorityBlockingQueue` 实现对任务的排序，传入其中的任务必须是具备排序能力的，方式有两种：

1. 提交到线程池的任务实现 `Comparable` 接口，并重写 `compareTo` 方法来指定任务之间的优先级比较规则。
2. 创建 `PriorityBlockingQueue` 时传入一个 `Comparator` 对象来指定任务之间的排序规则(推荐)

不过，这存在一些风险和问题，比如：

- `PriorityBlockingQueue` 是无界的，可能堆积大量的请求，从而导致 OOM。
- 可能会导致饥饿问题，即低优先级的任务长时间得不到执行。
- 由于需要对队列中的元素进行排序操作以及保证线程安全（并发控制采用的是可重入锁 `ReentrantLock`），因此会降低性能。

对于 OOM 这个问题的解决比较简单粗暴，就是继承`PriorityBlockingQueue` 并重写一下 `offer` 方法(入队)的逻辑，当插入的元素数量超过指定值就返回 false 。

饥饿问题这个可以通过优化设计来解决（比较麻烦），比如等待时间过长的任务会被移除并重新添加到队列中，但是优先级会被提升。

对于性能方面的影响，是没办法避免的，毕竟需要对任务进行排序操作。并且，对于大部分业务场景来说，这点性能影响是可以接受的。

### Future

当我们执行某一耗时的任务时，可以将这个耗时任务交给一个子线程去异步执行，同时我们可以干点其他事情，不用傻傻等待耗时任务执行完成。等我们的事情干完后，我们再通过 `Future` 类获取到耗时任务的执行结果。异步思想的运用，也是多线程中Future模式。

在 Java 中，`Future` 类只是一个泛型接口，位于 `java.util.concurrent` 包下，其中定义了 5 个方法，主要包括下面这 4 个功能：

- 取消任务；
- 判断任务是否被取消;
- 判断任务是否已经执行完成;
- 获取任务执行结果。

简单理解就是：我有一个任务，提交给了 `Future` 来处理。任务执行期间我自己可以去做任何想做的事情。并且，在这期间我还可以取消任务以及获取任务的执行状态。一段时间之后，我就可以 `Future` 那里直接取出任务执行结果。

#### Callable 和 Future 有什么关系？

可以通过 `FutureTask` 来理解 `Callable` 和 `Future` 之间的关系

`FutureTask` 提供了 `Future` 接口的基本实现，常用来封装 `Callable` 和 `Runnable`，具有取消任务、查看任务是否执行完成以及获取任务执行结果的方法。`ExecutorService.submit()` 方法返回的其实就是 `Future` 的实现类 `FutureTask` 。

`FutureTask` 还实现了`Runnable` 接口，可以作为任务直接被线程执行。

![img](https://oss.javaguide.cn/github/javaguide/java/concurrent/completablefuture-class-diagram.jpg)

`FutureTask` 有两个构造函数，可传入 `Callable` 或者 `Runnable` 对象。实际上，传入 `Runnable` 对象也会在方法内部转换为`Callable` 对象。

`FutureTask`相当于对`Callable` 进行了封装，管理着任务执行的情况，存储了 `Callable` 的 `call` 方法的任务执行结果。

### CompletableFuture 类有什么用？

![img](https://oss.javaguide.cn/github/javaguide/java/concurrent/completablefuture-class-diagram.jpg)

`CompletionStage` 接口描述了一个异步计算的阶段。很多计算可以分成多个阶段或步骤，此时可以通过它将所有步骤组合起来，形成异步计算的流水线。

`CompletionStage` 接口中的方法比较多，`CompletableFuture` 的函数式能力就是这个接口赋予的。从这个接口的方法参数你就可以发现其大量使用了 Java8 引入的函数式编程。



### AQS

全称为 `AbstractQueuedSynchronizer` ，抽象队列同步器。**AQS 就是一个抽象类**，**主要用来构建锁和同步器。**

我们提到的 `ReentrantLock`，`Semaphore`，其他的诸如 `ReentrantReadWriteLock`，`SynchronousQueue`等等皆是基于 AQS 的。

### 原理

AQS 核心思想是，如果被请求的共享资源空闲，则将当前请求资源的线程设置为有效的工作线程，并且将共享资源设置为锁定状态。如果被请求的共享资源被占用，那么就需要一套线程阻塞等待以及被唤醒时锁分配的机制，这个机制 AQS 是用 **CLH 队列锁** 实现的，即将暂时获取不到锁的线程加入到队列中。

CLH(Craig,Landin,and Hagersten) 队列是一个虚拟的双向队列。AQS 是将**每条请求共享资源的线程**封装成一个 CLH 锁队列的一个结点（Node）来实现锁的分配。

![img](https://oss.javaguide.cn/github/javaguide/java/CLH.png)

AQS中的变量

AQS 使用 **int 成员变量 `state` 表示同步状态**，通过内置的 **线程等待队列** 来完成获取资源线程的排队工作。

`state` 变量由 `volatile` 修饰，用于展示当前临界资源的获锁情况。

~~~java
// 共享变量，使用volatile修饰保证线程可见性
private volatile int state;
~~~

资源状态信息 state 可以通过 protected 类型的getState()、setState()和compareAndSetState() 进行操作。并且，这几个方法都是 final 修饰的，在子类中无法被重写。

以 `ReentrantLock` 为例，`state` 初始值为 0，表示未锁定状态。A 线程 `lock()` 时，会调用 `tryAcquire()` 独占该锁并将 `state+1` 。此后，其他线程再 `tryAcquire()` 时就会失败，直到 A 线程 `unlock()` 到 `state=`0（即释放锁）为止，其它线程才有机会获取该锁。当然，释放锁之前，A 线程自己是可以重复获取此锁的（`state` 会累加），这就是可重入的概念。但要注意，获取多少次就要释放多少次，这样才能保证 state 是能回到零态的。

这里为什么能行是因为**state**是一个共享变量 ，是一个资源。

再以 `CountDownLatch` 以例，任务分为 N 个子线程去执行，`state` 也初始化为 N（注意 N 要与线程个数一致）。这 N 个子线程是并行执行的，每个子线程执行完后`countDown()` 一次，state 会 CAS(Compare and Swap) 减 1。等到所有子线程都执行完后(即 `state=0` )，会 `unpark()` 主调用线程，然后主调用线程就会从 `await()` 函数返回，继续后余动作

### [Semaphore 有什么用？](https://javaguide.cn/java/concurrent/java-concurrent-questions-03.html#semaphore-有什么用)

`synchronized` 和 `ReentrantLock` 都是一次只允许一个线程访问某个资源，而`Semaphore`(信号量)可以用来控制同时**访问特定资源**的线程数量。

当初始的资源个数为 1 的时候，`Semaphore` 退化为排他锁。

`Semaphore` 有两种模式：。

- **公平模式：** 调用 `acquire()` 方法的顺序就是获取许可证的顺序，遵循 FIFO；
- **非公平模式：** 抢占式的。

下面的代码表示同一时刻 N 个线程中只有 5 个线程能获取到共享资源，其他线程都会阻塞，只有获取到共享资源的线程才能执行。等到有线程释放了共享资源，其他阻塞的线程才能获取到。

~~~java
// 初始共享资源数量
final Semaphore semaphore = new Semaphore(5);
// 获取1个许可
semaphore.acquire();
// 释放1个许可
semaphore.release();
~~~

#### Semaphore原理

Semaphore` 是共享锁的一种实现，它默认构造 AQS 的 `state` 值为 `permits，只有拿到许可证的线程才能执行。

要使用CAS操作修改state值，许可证数量不足。此时会创建一个 Node 节点加入阻塞队列，挂起当前线程。

### [CountDownLatch 有什么用？](#countdownlatch-有什么用)

`CountDownLatch` 允许 `count` 个线程阻塞在一个地方，直至所有线程的任务都执行完毕。

`CountDownLatch` 是一次性的，计数器的值只能在构造方法中初始化一次，之后没有任何机制再次对其设置值，当 `CountDownLatch` 使用完毕后，它不能再次被使用。

### [CountDownLatch 的原理是什么？](#countdownlatch-的原理是什么)

`CountDownLatch` 是共享锁的一种实现,它默认构造 AQS 的 `state` 值为 `count`。当线程使用 `countDown()` 方法时,其实使用了`tryReleaseShared`方法以 CAS 的操作来减少 `state`,直至 `state` 为 0 。当调用 `await()` 方法的时候，如果 `state` 不为 0，那就证明任务还没有执行完毕，`await()` 方法就会一直阻塞，也就是说 `await()` 方法之后的语句不会被执行。直到`count` 个线程调用了`countDown()`使 state 值被减为 0，或者调用`await()`的线程被中断，该线程才会从阻塞中被唤醒，`await()` 方法之后的语句得到执行。

~~~java
//应用场景，await等待阻塞的线程执行
public class CountDownLatchExample1 {
    // 处理文件的数量
    private static final int threadCount = 6;

    public static void main(String[] args) throws InterruptedException {
        // 创建一个具有固定线程数量的线程池对象（推荐使用构造方法创建）
        ExecutorService threadPool = Executors.newFixedThreadPool(10);
        final CountDownLatch countDownLatch = new CountDownLatch(threadCount);
        for (int i = 0; i < threadCount; i++) {
            final int threadnum = i;
            threadPool.execute(() -> {
                try {
                    //处理文件的业务操作
                    //......
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    //表示一个文件已经被完成
                    countDownLatch.countDown();
                }

            });
        }
        countDownLatch.await();
        threadPool.shutdown();
        System.out.println("finish");
    }
}

~~~

### CyclicBarrier 有什么用

`CyclicBarrier` 和 `CountDownLatch` 非常类似，它也可以实现线程间的技术等待，但是它的功能比 `CountDownLatch` 更加复杂和强大。主要应用场景和 `CountDownLatch` 类似。

> `CountDownLatch` 的实现是基于 AQS 的，而 `CycliBarrier` 是基于 `ReentrantLock`(`ReentrantLock` 也属于 AQS 同步器)和 `Condition` 的。

`CyclicBarrier` 的字面意思是可循环使用（Cyclic）的屏障（Barrier）。它要做的事情是：让一组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续干活。

由一个count来计数，达到规定才放行

#### 原理

`CyclicBarrier` 默认的构造方法是 `CyclicBarrier(int parties)`，其参数表示屏障拦截的线程数量，每个线程调用 `await()` 方法告诉 `CyclicBarrier` 我已经到达了屏障，然后当前线程被阻塞。

当调用 `CyclicBarrier` 对象调用 `await()` 方法时，实际上调用的是 `dowait(false, 0L)`方法。 `await()` 方法就像树立起一个栅栏的行为一样，将线程挡住了，当拦住的线程数量达到 `parties` 的值时，栅栏才会打开，线程才得以通过执行