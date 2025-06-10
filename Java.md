# 目录



# 一、序列化和反序列化

## 1. 介绍

### 1.1 概述

**Java序列化**是指将Java对象转换成字节流，以便存储在文件中或在网络上传输。

**Java反序列化**是指将字节流转换成Java对象。

### 1.2 应用场景

- **实现数据持久化**（保存对象及其状态到内存或磁盘）

  Java 平台允许我们在内存中创建可复用的 Java 对象，但一般情况下，只有当 JVM 处于运行时，这些对象才可能存在，即，这些对象的生命周期不会比 JVM 的生命周期更长。但在现实应用中，就可能要求在 JVM 停止运行之后能够保存(持久化)指定的对象，并在将来重新读取被保存的对象。Java 对象序列化就能够帮助我们实现该功能。

- **实现远程通信**

  除了在持久化对象时会用到对象序列化之外，当使用 RMI(远程方法调用)，或在网络中传递对象时，都会用到对象序列化。Java 序列化 API 为处理对象序列化提供了一个标准机制，该 API 简单易用。

- **实现对象的传递**

  通过序列化在进程间传递对象。

> **场景举例：**
>
> - **序列化输出到文件**
>
> Web 服务器中的 Session 会话对象，当有10万用户并发访问，就有可能出现10万个 Session 对象，显然这种情况内存可能是吃不消的。于是 Web 容器就会把一些 Session 先序列化，让他们离开内存空间，序列化到硬盘中，当需要调用时，再把保存在硬盘中的对象还原到内存中。
>
> - **序列化输出到网络**
>
> 我们知道，当两个进程进行远程通信时，彼此可以发送各种类型的数据，包括文本、图片、音频、视频等， 而这些数据都会以二进制序列的形式在网络上传送。同样的序列化与反序列化则实现了 **进程通信间的对象传送**，发送方需要把这个Java对象转换为字节序列，才能在网络上传送；接收方则需要把字节序列再恢复为Java对象。

### 1.3 序列化协议

- XML&SOAP

- JSON

- Protobuf



## 2. 序列化的实现

JDK提供的序列化API：

- java.io.ObjectOutputStream：表示对象输出流，它的writeObject(Object obj)方法可以对参数指定的obj对象进行序列化，把得到的字节序列写到一个目标输出流中。
- java.io.ObjectInputStream：表示对象输入流，它的readObject()方法源输入流中读取字节序列，再把它们反序列化成为一个对象，并将其返回。

在Java中， 只有实现了**Serializable接口**或者**Externalizable接口**的类的对象才能被序列化为字节序列。

### 2.1 `Serializable` 接口

#### 2.1.2 概述

`Serializable` 是一个标记接口（marker interface），它没有任何方法。实现 `Serializable` 接口的类可以通过 Java 的序列化机制将对象的状态保存到字节流中，并从字节流中恢复对象的状态。

#### 2.1.2 使用方法

1. **实现 `Serializable` 接口** 类需要实现 `Serializable` 接口，以表明它可以被序列化。

   ```java
   import java.io.Serializable;
   
   public class Person implements Serializable {
       private static final long serialVersionUID = 1L;
       private String name;
       private int age;
   
       public Person(String name, int age) {
           this.name = name;
           this.age = age;
       }
   
       // Getters and setters
   }
   ```

   

2. **序列化和反序列化** 使用 `ObjectOutputStream` 和 `ObjectInputStream` 来序列化和反序列化对象。

   ```java
   import java.io.*;
   
   public class Main {
       public static void main(String[] args) throws IOException, ClassNotFoundException {
           // 创建对象
           Person person = new Person("Alice", 30);
   
           // 序列化
           try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("person.ser"))) {
               oos.writeObject(person);
           }
   
           // 反序列化
           try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream("person.ser"))) {
               Person deserializedPerson = (Person) ois.readObject();
               System.out.println("Deserialized Person: " + deserializedPerson.getName() + ", " + deserializedPerson.getAge());
           }
       }
   }
   ```

   

3. **自定义序列化逻辑** 如果需要自定义序列化逻辑，可以在类中定义 `writeObject` 和 `readObject` 方法。

   ```java
   private void writeObject(ObjectOutputStream oos) throws IOException {
       oos.defaultWriteObject(); // 调用默认的序列化方法
       // 自定义逻辑
   }
   
   private void readObject(ObjectInputStream ois) throws IOException, ClassNotFoundException {
       ois.defaultReadObject(); // 调用默认的反序列化方法
       // 自定义逻辑
   }
   ```

   

### 2.2 `Externalizable` 接口

#### 2.2.1 概述

`Externalizable` 是一个扩展了 `Serializable` 接口的接口，它提供了更细粒度的控制，允许类完全自定义其序列化和反序列化逻辑。

#### 2.2.1 使用方法

1. **实现 `Externalizable` 接口** 类需要实现 `Externalizable` 接口，并实现 `writeExternal` 和 `readExternal` 方法。

   ```java
   import java.io.*;
   
   public class Person implements Externalizable {
       private static final long serialVersionUID = 1L;
       private String name;
       private int age;
   
       public Person() {} // 必须提供无参构造函数
   
       public Person(String name, int age) {
           this.name = name;
           this.age = age;
       }
   
       @Override
       public void writeExternal(ObjectOutput out) throws IOException {
           out.writeObject(name);
           out.writeInt(age);
       }
   
       @Override
       public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
           name = (String) in.readObject();
           age = in.readInt();
       }
   
       // Getters and setters
   }
   ```

   

2. **序列化和反序列化** 使用 `ObjectOutputStream` 和 `ObjectInputStream` 来序列化和反序列化对象。

   ```java
   import java.io.*;
   
   public class Main {
       public static void main(String[] args) throws IOException, ClassNotFoundException {
           // 创建对象
           Person person = new Person("Alice", 30);
   
           // 序列化
           try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("person.ser"))) {
               oos.writeObject(person);
           }
   
           // 反序列化
           try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream("person.ser"))) {
               Person deserializedPerson = (Person) ois.readObject();
               System.out.println("Deserialized Person: " + deserializedPerson.getName() + ", " + deserializedPerson.getAge());
           }
       }
   }
   ```

   

### 2.3  `Serializable` 和 `Externalizable` 的区别

| 特性         | `Serializable`               | `Externalizable`                            |
| ------------ | ---------------------------- | ------------------------------------------- |
| **实现方式** | 标记接口，无方法             | 提供 `writeExternal` 和 `readExternal` 方法 |
| **控制粒度** | 默认序列化机制，可部分自定义 | 完全自定义序列化和反序列化逻辑              |
| **性能**     | 默认机制，可能较慢           | 更高效，因为可以优化序列化过程              |
| **安全性**   | 默认机制，可能暴露敏感信息   | 更安全，因为可以控制序列化内容              |
| **兼容性**   | 更通用，适用于大多数场景     | 更灵活，但需要更多代码实现                  |

- **`Serializable`**：适用于大多数场景，简单易用，可以通过自定义 `writeObject` 和 `readObject` 方法进行部分自定义。
- **`Externalizable`**：适用于需要完全控制序列化和反序列化逻辑的场景，性能更高，安全性更好，但需要实现 `writeExternal` 和 `readExternal` 方法。



## 3. 序列化ID

在实现了`Serializable`接口的序列化类中，经常会定义一个序列化ID属性：

```java
private static final long serialVersionUID = 1L;
```

### 3.1 介绍

序列化ID其实就是用来验证序列化的对象和反序列化对应的对象的ID是否是一致的。所以这个ID的数字其实不重要，无论是1L还是idea自动生成的，只要序列化的时候对象的serialVersionUID和反序列化的时候对象的serialVersionUID一致的话就行。

serialVersionUID就是起验证作用，可以认为是一个指纹。所以如果为类定义一个`serialVersionUID`然后列化一个对象之后，反序列化之前把对象的类的结构改了，比如增加了一个成员变量，则此时的反序列化会失败。因为类的结构变了，生成的指纹就变了，所以serialVersionUID就不一致了。

‘’

## 4. 序列化具体流程

1. 调用入口方法：`writeObject`

   用户调用 `ObjectOutputStream` 的 `writeObject` 方法，传入需要序列化的对象。

2. 检查流状态

   序列化开始时，会检查流是否已经关闭。如果流已关闭，抛出 `IOException`。

3. 对象替换

   如果启用了对象替换（`enableReplace`），会调用 `replaceObject` 方法，允许用户自定义对象替换逻辑。这可以用于在序列化之前修改对象。

4. 调用核心方法：`writeObject0`

   `writeObject0` 是序列化的核心方法，负责处理各种特殊情况，并递归序列化对象。

5. 处理特殊情况

   - **检查对象是否为 `null`**：如果是 `null`，写入 `null` 标记。

   - **检查对象是否已经序列化过**：如果对象已经序列化过（共享对象处理），写入对象的句柄，而不是重新序列化。

   - **检查对象是否是特殊类型**：如 `Class` 或 `ObjectStreamClass`，调用相应的序列化方法。

6. 对象替换并判断是否满足上一步中的特殊情况，若满足则直接处理。

7. 普通情况序列化

   - 如果序列化对象是String、Array、Enum类型，则调用相应的专门的序列化方法。

   - 如果不是，但实现了 `Serializable` 接口，调用 `writeOrdinaryObject` 方法。

     - 写入对象头

       写入对象头（`TC_OBJECT`）和类描述符（`ObjectStreamClass`）。

     - 分配句柄

       为对象分配一个句柄，以便在后续引用时可以直接使用句柄，避免重复序列化。

     - 处理自定义序列化逻辑

       - 如果对象实现了 `Externalizable` 接口，调用 `writeExternal` 方法。

       - 如果对象没有实现 `Externalizable` 接口，调用 `writeSerialData` 方法序列化对象的字段值。
         - 如果对象的类定义了 `writeObject` 方法，调用该方法进行自定义序列化。
         - 如果对象的类没有定义`writeObject`方法，则调用对象的默认序列化方法`defaultWriteFields`，该方法先把基本数据类型进行序列化，然后遍历对象的其他字段，对这些字段递归调用 `writeObject0` 方法进行序列化，回到步骤4。

   - 如果如上两种情况都不满足，抛出`NotSerializableException`异常。

> ### 递归的终止条件
>
> - 字段值为 `null`，写入 `null` 标记。
> - 字段值是已经序列化过的对象，写入对象的句柄。
> - 字段是 `Class` 或 `ObjectStreamClass`类型，调用相应的序列化方法。
> - 字段是可替换的，并且替换后的类对象满足前面三点的任意一点。
> - 序列化对象是String、Array、Enum类型。
> - 序列化对象没有实现`Serializable`接口而抛出异常。

9. 恢复状态

   在 `finally` 块中，恢复流的状态，减少嵌套深度计数。



# 二、Java容器



# 三、Java8新特性

## 1. 函数式接口

### 1.1 定义
函数式接口是**只有一个抽象方法的接口**，通常用来和Lambda表达式一起使用，可以使用 @FunctionalInterface 注解进行标注，但这不是必须的。该注解的作用是为了保证该接口符合函数式接口的定义。

```java
@FunctionalInterface
public interface MyFunctionalInterface {
    void execute();
}
```

虽然 @FunctionalInterface 注解不是必须的，但推荐使用它，因为它能使代码更具可读性，并在编译时提供额外的检查。

### 1.2 常用的函数式接口
Java 8 提供了许多内置的函数式接口，这些接口都在 java.util.function 包中。以下是一些常用的函数式接口：

**Predicate**：接收一个参数，返回一个布尔值。

```java
@FunctionalInterface
public interface Predicate<T> {
    boolean test(T t);
}
```



**Function<T, R>**：接收一个参数，返回一个结果。

```java
@FunctionalInterface
public interface Function<T, R> {
    R apply(T t);
}
```



**Supplier**：不接收参数，返回一个结果。

```java
@FunctionalInterface
public interface Supplier<T> {
    T get();
}
```



**Consumer**：接收一个参数，没有返回值。

```java
@FunctionalInterface
public interface Consumer<T> {
    void accept(T t);
}
```



**UnaryOperator**：接收一个参数，返回与该参数类型相同的结果。

```java
@FunctionalInterface
public interface UnaryOperator<T> extends Function<T, T> {
}
```



**BinaryOperator**：接收两个参数，返回与参数类型相同的结果。

```
@FunctionalInterface
public interface BinaryOperator<T> extends BiFunction<T, T, T> {
}
```



## 2. lambda表达式

### 2.1 介绍

Lambda 表达式是 JDK8 的一个新特性，用来简便的创建函数式接口的实现类对象，尤其在集合的遍历和其他集合操作中，可以极大地优化代码结构。

不过Lambda表达式也有限制，即用Lambda表达式创建的匿名类所实现的接口只能有一个抽象方法。

Lambda表达式的本质：

- 一方面，Lambda表达式作为接口的实现类的对象。
- 另一方面，Lambda表达式是一个匿名函数。

### 2.2 格式

Lambda表达式由**三部分**组成：

1. 参数列表（允许没有参数）

2. 箭头，用来标识这串代码为 Lambda 表达式（也就是说，看到 `->` 就知道这是 Lambda）

3. 主体，返回值或者执行代码块

   ![image](https://i-blog.csdnimg.cn/blog_migrate/03b88bfb15e1a46cdbb24aa251b2913d.png)

风格：

- （参数）-> { 代码块 }，此时需要在代码块内显式地使用`return`关键字来作为Lambda表达式的返回值。

  `(parameters) -> { statements; }`

- （参数）-> 表达式，当代码块只有一句代码的时候可以去掉`{}`，此时该表达式不能包含`return`，**表达式的返回值会被自动用作Lambda表达式的返回值**，包括void。

  `(parameters) -> expression`

- 参数 -> 表达式 / { 代码块 }，这种情形适用于参数只有一个的时候，此时可以去掉`()`只写唯一的参数。

  `parameter -> expression / { 代码块 }`

  注意：前两种有`()`的情形中，参数的数据类型可加可不加，第三种情形不可加参数类型。

其中 () 用来描述参数列表，{} 用来描述方法体，-> 为 lambda运算符 ，读作(goes to)。

### 2.3 使用

#### 2.3.1 常用函数式接口

函数式接口通常与 Lambda 表达式和方法引用一起使用。下面是一些使用示例：

Predicate 示例

```java
import java.util.function.Predicate;

public class PredicateExample {
    public static void main(String[] args) {
        Predicate<String> isLongerThan5 = s -> s.length() > 5;
        System.out.println(isLongerThan5.test("hello")); // false
        System.out.println(isLongerThan5.test("hello world")); // true
    }
}
```

Function 示例

```java
import java.util.function.Function;

public class FunctionExample {
    public static void main(String[] args) {
        Function<String, Integer> stringLength = s -> s.length();
        System.out.println(stringLength.apply("hello")); // 5
    }
}
```

Supplier 示例

```java
import java.util.function.Supplier;

public class SupplierExample {
    public static void main(String[] args) {
        Supplier<String> stringSupplier = () -> "Hello, World!";
        System.out.println(stringSupplier.get()); // Hello, World!
    }
}
```

Consumer 示例

```java
import java.util.function.Consumer;

public class ConsumerExample {
    public static void main(String[] args) {
        Consumer<String> printConsumer = s -> System.out.println(s);
        printConsumer.accept("Hello, World!"); // Hello, World!
    }
}
```

UnaryOperator 示例

```java
import java.util.function.UnaryOperator;

public class UnaryOperatorExample {
    public static void main(String[] args) {
        UnaryOperator<Integer> square = x -> x * x;
        System.out.println(square.apply(5)); // 25
    }
}
```

BinaryOperator 示例

```java
import java.util.function.BinaryOperator;

public class BinaryOperatorExample {
    public static void main(String[] args) {
    	BinaryOperator<Integer> add = (a, b) -> a + b;
		System.out.println(add.apply(5, 3)); // 8
	}
}
```

#### 2.3.2 自定义函数式接口
除了使用 Java 提供的函数式接口外，你还可以定义自己的函数式接口。下面是一个自定义函数式接口的示例：

```java
@FunctionalInterface
public interface MyFunctionalInterface {
    void execute(String message);
}

public class FunctionalInterfaceDemo {
    public static void main(String[] args) {
        MyFunctionalInterface myFunc = (message) -> System.out.println(message);
        myFunc.execute("Hello, Functional Interface!"); // Hello, Functional Interface!
    }
}
```

#### 2.3.3 使用方法引用
方法引用是另一种简洁的 Lambda 表达式写法。常见的用法包括引用静态方法、实例方法和构造方法。

**静态方法引用**

```java
import java.util.function.Function;

public class MethodReferenceExample {
    public static void main(String[] args) {
        Function<String, Integer> stringToInt = Integer::parseInt;
        System.out.println(stringToInt.apply("123")); // 123
    }
}
```

**实例方法引用**

```java
import java.util.function.Predicate;

public class InstanceMethodReferenceExample {
    public static void main(String[] args) {
        String str = "Hello";
        Predicate<String> isEqual = str::equals;
        System.out.println(isEqual.test("Hello")); // true
        System.out.println(isEqual.test("World")); // false
    }
}
```

构造方法引用

```java
import java.util.function.Supplier;
import java.util.ArrayList;
import java.util.List;

public class ConstructorReferenceExample {
    public static void main(String[] args) {
        Supplier<List<String>> listSupplier = ArrayList::new;
        List<String> list = listSupplier.get();
        System.out.println(list); // []
    }
}
```

通过掌握 Java 8 的函数式接口及其用法，可以编写出更加简洁和高效的代码，充分利用函数式编程的优势。



## 3. Stream流





# 三、Java的IO

## 1. 概述

在 Java 中，提供了一系列 API，可以供开发者来读写外部数据或文件。我们称这些 API 为 Java IO。IO 是 Java 中比较重要，且比较难的知识点，主要是因为随着 Java 的发展，目前有三种 IO 共存。分别是 BIO、NIO 和 AIO。

| 对比维度           | **BIO**                                                 | **NIO**                                                      | **AIO**                                                      |
| ------------------ | ------------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **I/O 模型**       | 同步阻塞（Blocking）                                    | 同步非阻塞（Non-blocking）                                   | 异步非阻塞（Asynchronous Non-blocking）                      |
| **JDK 引入版本**   | JDK 1.0                                                 | JDK 1.4                                                      | JDK 1.7                                                      |
| **是否阻塞**       | 是                                                      | 否                                                           | 否                                                           |
| **是否异步**       | 否                                                      | 否（需自行调度）                                             | 是（系统完成后回调）                                         |
| **传输方向**       | 面向连接，面向流（TCP）                                 | 面向块（block-oriented）、面向连接                           | 面向事件、支持高并发连接处理                                 |
| **数据处理方式**   | 面向流（Stream）                                        | 面向缓冲区（Buffer）                                         | 面向事件（Event + Buffer）                                   |
| **线程模型**       | 一连接一线程（线程池可优化）                            | 一线程多连接（多路复用）                                     | 注册回调 + 系统通知，少量线程                                |
| **I/O 操作处理**   | 主线程阻塞处理                                          | 主线程轮询事件再处理                                         | 回调函数处理                                                 |
| **面向协议**       | TCP（流协议）为主                                       | TCP（主），UDP（DatagramChannel）也支持                      | TCP（目前主流实现），UDP 不常用                              |
| **核心 API**       | `Socket`, `ServerSocket`, `InputStream`, `OutputStream` | `SocketChannel`, `ServerSocketChannel`, `Selector`, `ByteBuffer` | `AsynchronousSocketChannel`, `AsynchronousServerSocketChannel`, `CompletionHandler` |
| **底层机制依赖**   | Java 阻塞 I/O 封装                                      | 基于 Linux epoll / Windows Selector / kqueue                 | 基于操作系统原生异步 I/O（如 Windows IOCP、Linux AIO）       |
| **系统支持依赖性** | 全平台都支持                                            | 全平台都支持                                                 | 操作系统是否支持异步 I/O 特性决定效率（部分系统实现有限）    |
| **资源消耗**       | 高（线程多）                                            | 中（线程少）                                                 | 低（系统通知 + 回调）                                        |
| **可扩展性**       | 差                                                      | 好                                                           | 很好                                                         |
| **开发难度**       | 简单，易上手                                            | 中等，涉及 Selector、Buffer 管理                             | 较高，涉及异步回调逻辑                                       |
| **编程模型**       | 阻塞、串行                                              | 事件轮询 + 通道                                              | 回调函数（基于事件驱动）                                     |
| **是否支持 UDP**   | 否                                                      | 是（DatagramChannel）                                        | 理论可支持，Java API 中不常见                                |
| **典型应用**       | 控制台程序、小服务端、练手项目                          | Netty、Tomcat NIO 模式、聊天室服务器                         | 高性能异步服务（如 Netty AIO 模式、文件服务器）              |
| **代表框架支持**   | Tomcat BIO、传统 Socket 编程                            | Netty、Tomcat NIO、Spring WebFlux、Akka                      | Netty（部分场景支持）、Spring Boot 2 + WebFlux、异步 RPC 系统 |
| **适用场景**       | 并发连接少，逻辑简单场景                                | 高并发网络服务，连接量大但 I/O 量中等                        | 非常高并发、I/O密集型、高响应要求的场景（如游戏、音视频）    |



## 2. BIO

### 2.1 概述

IO，即in和out，也就是输入和输出，指应用程序和外部设备之间的数据传递，常见的外部设备包括文件、管道、网络连接。Java 中是通过流处理IO 的。

> ### 流介绍
>
> 流（Stream），是一个抽象的概念，是指一连串的数据（字符或字节），是以先进先出的方式发送信息的通道。
>
> 当程序需要读取数据的时候，就会开启一个通向数据源的流，这个数据源可以是文件，内存，或是网络连接。类似的，当程序需要写入数据的时候，就会开启一个通向目的地的流。这时候你就可以想象数据好像在这其中“流”动一样。
>
> 一般来说关于流的特性有下面几点：
>
> - 先进先出：最先写入输出流的数据最先被输入流读取到。
> - 顺序存取：可以一个接一个地往流中写入一串字节，读出时也将按写入顺序读取一串字节，不能随机访问中间的数据。（RandomAccessFile除外）
> - 只读或只写：每个流只能是输入流或输出流的一种，不能同时具备两个功能，输入流只能进行读操作，对输出流只能进行写操作。在一个数据传输通道中，如果既要写入数据，又要读取数据，则要分别提供两个流。

Java的BIO体系架构如下图所示：

![IO_cnajssas](E:\各种资料\Java开发笔记\我的笔记\images\IO_cnajssas.png)

### 2.2 File类

#### 2.2.1 概述

- java.io.File类是文件和文件目录路径的抽象表示形式，与平台无关。
- File 能新建、删除、重命名文件和目录，但 File 不能访问文件内容本身。如果需要访问文件内容本身，则需要使用输入/输出流。
- 想要在Java程序中表示一个真实存在的文件或目录，那么必须有一个File对象，但是Java程序中的一个File对象，可能没有一个真实存在的文件或目录。
- File对象可以作为参数传递给流的构造器。

#### 2.2.2 构造

- ```java
  public File(String pathname)
  ```

  以pathname为路径创建File对象，可以是绝对路径或者相对路径，如果pathname是相对路径，则默认的当前路径在系统属性user.dir中存储。

  - 绝对路径：是一个固定的路径,从盘符开始
  - 相对路径：是相对于某个位置开始

- ```java
  public File(String parent,String child)
  ```

  以parent为父路径，child为子路径创建File对象。

- ```java
  public File(File parent,String child)
  ```

  根据一个父File对象和子文件路径创建File对象

> ### 路径分隔
>
> 路径中的每级目录之间用一个**路径分隔符**隔开。路径分隔符和系统有关：
>
> - windows和DOS系统默认使用“\”来表示
> - UNIX和URL使用“/”来表示
>
> 为了解决这个隐患，File类提供了一个常量：
>
> public static final String separator。根据操作系统，动态的提供分隔符。
>
> 举例：
>
> File file1 = new File("d:\\atguigu\\info.txt");
>
> File file2 = new File("d:" + File.separator + "atguigu" + File.separator + "info.txt");
>
> File file3 = new File("d:/atguigu");

#### 2.2.3 常用方法





## 3. NIO





## 4. AIO





# 四、Java并发编程

## 1. 线程运行状态

### 1.1 线程运行状态介绍

#### 1.1.1 NEW

处于 NEW 状态的线程此时尚未启动。这里的尚未启动指的是还没调用 Thread 实例的`start()`方法。



#### 1.1.2 RUNNABLE

表示当前线程正在运行中。处于 RUNNABLE 状态的线程在 Java 虚拟机中运行，也有可能在等待 CPU 分配资源。

也就是说，Java 线程的**RUNNABLE**状态其实包括了操作系统线程的**ready**和**running**两个状态。



#### 1.1.3 BLOCKED

阻塞状态。处于 BLOCKED 状态的线程正等待锁的释放以进入同步区。



#### 1.1.4 WAITING

等待状态。处于等待状态的线程变成 RUNNABLE 状态需要其他线程唤醒。



#### 1.1.5 TIMED_WAITING

超时等待状态。线程等待一个具体的时间，时间到后会被自动唤醒。

调用如下方法会使线程进入超时等待状态：

- `Thread.sleep(long millis)`：使当前线程睡眠指定时间；
- `Object.wait(long timeout)`：线程休眠指定时间，等待期间可以通过`notify()`/`notifyAll()`唤醒；
- `Thread.join(long millis)`：等待当前线程最多执行 millis 毫秒，如果 millis 为 0，则会一直执行；
- `LockSupport.parkNanos(long nanos)`： 除非获得调用许可，否则禁用当前线程进行线程调度指定时间；[LockSupport](https://javabetter.cn/thread/LockSupport.html) 我们在后面会细讲；
- `LockSupport.parkUntil(long deadline)`：同上，也是禁止线程进行调度指定时间；

我们继续延续上面的例子来解释一下 TIMED_WAITING 状态：

到了第二天中午，又到了饭点，你还是到了窗口前。

突然间想起你的同事叫你等他一起，他说让你等他十分钟他改个 bug。

好吧，那就等等吧，你就离开了窗口。很快十分钟过去了，你见他还没来，你想都等了这么久了还不来，那你还是先去吃饭好了。

这时你还是线程 t1，你改 bug 的同事是线程 t2。t2 让 t1 等待了指定时间，此时 t1 等待期间就属于 TIMED_WATING 状态。

t1 等待 10 分钟后，就自动唤醒，拥有了去争夺锁的资格。



#### 1.1.6 TERMINATED

终止状态。此时线程已执行完毕。



### 1.2 线程运行状态变化

![thread_nvdknj](.\images\thread_nvdknj.png)



#### 1.2.1 NEW->RUNNABLE

- 调用Thread的start()方法即可。

> **注意：**Java不允许一个线程重复使用start()来启动，当且仅当线程状态为NEW时才允许启动线程，而线程启动后线程状态就不再是NEW，且不会再变成NEW状态，start()方法源码如下：
>
> ```java
> // 使用synchronized关键字保证这个方法是线程安全的
> public synchronized void start() {
>  // threadStatus != 0 表示这个线程已经被启动过或已经结束了
>  // 如果试图再次启动这个线程，就会抛出IllegalThreadStateException异常
>  if (threadStatus != 0)
>      throw new IllegalThreadStateException();
> 
>  // 以下代码暂不关注
>  group.add(this);
> 
>  boolean started = false;
>  try {
>      start0();
>      started = true;
>  } finally {
>      try {
>          if (!started) {
>              group.threadStartFailed(this);
>          }
>      } catch (Throwable ignore) {
>      }
>  }
> }
> ```
>
> 可以看到，在`start()`内部，有一个 threadStatus 变量。如果它不等于 0，调用`start()`会直接抛出异常。
>
> 测试代码如下，运行后会因为两次调用一个Thread的start()方法而报错：
>
> ```java
> @Test
> public void testStartMethod() {
>  Thread thread = new Thread(() -> {});
>  thread.start(); // 第一次调用
>  thread.start(); // 第二次调用
> }
> ```



#### 1.2.2 RUNNABLE->BLOCK

- `synchronized` 块或方法：当多个线程尝试获取同一把锁时，未获取到锁的线程将被阻塞。

- `ReentrantLock.lock()`：尝试获取锁，如果锁被其他线程持有，则当前线程被阻塞。



#### 1.2.3 BLOCK -> RUNNABLE

- 锁被释放：当持有锁的线程执行完同步块或方法，释放锁后，等待的线程将有机会获取锁并变为可运行状态。



#### 1.2.4 RUNNABLE -> WATING

- `Object.wait()`：使当前线程处于等待状态直到另一个线程唤醒它；
- `Thread.join()`：等待线程执行完毕，底层调用的是 Object 的 wait 方法；
- `LockSupport.park()`：除非获得调用许可，否则禁用当前线程进行线程调度。



#### 1.2.5 WAITING -> RUNNABLE

- `Object.notify()` 或 `Object.notifyAll()`：唤醒在此对象监视器上等待的单个或所有线程。

- `LockSupport.unpark(Thread)`：唤醒指定的线程。



#### 1.2.6 RUNNABLE -> TIMED_WAITING

- `Thread.sleep(long)`：使当前线程休眠指定的时间。
- `Object.wait(long)`：使当前线程等待，直到其他线程调用此对象的 `notify()` 或 `notifyAll()` 方法，或者超过指定的时间。
- `Thread.join(long)`：等待线程执行完毕或超过指定的时间。
- `LockSupport.parkNanos()`：禁用当前线程进行线程调度，除非获得许可，或者超过指定的时间。
- `LockSupport.parkUntil()`：禁用当前线程进行线程调度，直到指定的时间，或者获得许可。



#### 1.2.7 TIMED_WAITING -> RUNNABLE

- 等待时间结束：当指定的等待时间结束，线程将变为可运行状态。
- `Object.notify()` 或 `Object.notifyAll()`：唤醒在此对象监视器上等待的单个或所有线程。
- `LockSupport.unpark(Thread)`：唤醒指定的线程。



#### 1.2.8 RUNNABLE -> RUNNABLE

- 线程调度：操作系统可能会在任何时候调度线程，即使线程已经在运行状态。这通常是为了公平地分配CPU时间给所有线程。
- `yield()`方法：事实上是调用了sleep(0)，但仅仅只是给CPU调度器一个提示(hint)，CPU调度器可能会忽略该提示，若没忽略提示，则让线程下CPU，切换其他线程占用CPU。



#### 1.2.9 RUNNABLE -> TERMINATED

- `run()` 方法结束：当线程的 `run()` 方法执行完毕，或者抛出未捕获的异常，线程将结束其生命周期，变为终止状态。



## 2. 创建线程的方式

### 2.1 继承Thread类

> Thread类继承自**Runnable接口**。
>
> Thread通过start()方法启动线程，源码如下：
>
> ```java
> // 使用synchronized关键字保证这个方法是线程安全的
> public synchronized void start() {
>  if (threadStatus != 0)
>      throw new IllegalThreadStateException();
> 
>  group.add(this);
> 
>  boolean started = false;
>  try {
>      // 使用native方法启动这个线程
>      start0();
>      // 如果没有抛出异常，那么started被设为true，表示线程启动成功
>      started = true;
>  } finally {
>      // 在finally语句块中，无论try语句块中的代码是否抛出异常，都会执行
>      try {
>          // 如果线程没有启动成功，就从线程组中移除这个线程
>          if (!started) {
>              group.threadStartFailed(this);
>          }
>      } catch (Throwable ignore) {
>          // 如果在移除线程的过程中发生了异常，我们选择忽略这个异常
>      }
>  }
> }
> ```
>
> 由Thread.start()源码可知，创建一个继承自Thread类的自定义类，通过覆写run()方法即可自定义线程运行执行的流程。

自定义一个类继承自Thread类，覆写run()方法，即可创建一个线程。通过调用自定义类对象的start()方法，即可启动线程。

举个例子：

```java
public class MyThread extends Thread{
    public void run() {
        for(int i = 0; i < 100; i++) {
            System.out.println(currentThread() + ": " + i);  // currentThread()获取当前线程名称
        }
    }
}
```

测试验证如下：

```java
public static void main(String[] args){
    for(int i = 0; i < 5; i++) {
        new MyThread().start();
    }
}
```

执行结果如下图，可以看到多个线程交替执行：

![thread_dsavdfg](.\images\thread_dsavdfg.png)



### 2.2 实现Runnable接口

> 由1.2.可以知道，Thread类的start()方法底层调用run()方法，而默认run()方法实现源码如下：
>
> ```java
> @Override
> public void run() {
>  if (target != null) {
>      target.run();
>  }
> }
> ```
>
> 其中，tartget是Thread类的一个private属性，类型为Runnable，可以通过如下构造函数来赋值：
>
> ```java
> // 带Runnable类型参数的构造函数
> public Thread(Runnable target) {
>  init(null, target, "Thread-" + nextThreadNum(), 0);
> }
> ```
>
> 由如上源码可以知道，Thread类的run()方法的默认实现方式会检查是否有传入Runnable对象，有的话则执行Runnable对象的run()方法；没有的话则run()方法可以视为空方法，由此可以得到第二个构造线程的方法。

自定义一个类实现Runnable接口，覆写Runnable接口的run()方法，接着将自定义类对象作为参数创建一个Thread对象，调用Thread对象的start()方法即可启动线程。

举个例子：

```java
public class MyThread implements Runnable{
    public void run() {
        for(int i = 0; i < 100; i++) {
            System.out.println(Thread.currentThread() + ": " + i);  // currentThread()获取当前线程名称
        }
    }
}
```

测试验证如下：

```java
public static void main(String[] args){
    MyThread t = new MyThread();
    for(int i = 0; i < 5; i++) {
        new Thread(t).start();
    }
}
```

执行结果如下图，可以看到多个线程交替执行：

![thread_dsavdfg](.\images\thread_dsavdfg.png)



### 2.3 实现Callable接口

通过前两种创建线程方式的源码解析可以发现，由于run()方法返回值为void，因此线程无法返回结果，由此引出第三种线程创建方式。

> Callable接口：内部仅仅声明了一个call()方法。
>
> 
>
> Future接口：定义了存储和操作运行结果的一套架构。
>
> 
>
> RunnableFuture接口：直接继承了Runnable接口和Future接口，没有额外添加新代码。
>
> 
>
> FutureTask类：实现了RunnableFuture接口，并将Runnable接口和Future接口两个接口架构进行实现和整合，使其既可以作为runnable对象来运行，也可以对线程运行结果进行操作。内部定义了一个Callable类型属性callable，可以借助如下构造函数赋值：
>
> ```java
> public FutureTask(Callable<V> callable) {
>      if (callable == null)
>          throw new NullPointerException();
>      this.callable = callable;
>      this.state = NEW; 
> }
> ```
>
> FutureTask覆写了run()方法，若属性callable不为空并且状态为new，则运行callable的call()方法（这里call()方法的作用相当于run()方法，只是run()方法用来封装获取结果的逻辑了，只能用call()方法来运行线程逻辑了），并将运行结果进行存储，源码如下：
>
> ```java
> public void run() {
> 	// 依旧是对线程状态判断，要求为新建态new
>  if (state != NEW ||
>      !UNSAFE.compareAndSwapObject(this, runnerOffset,
>                                   null, Thread.currentThread()))
>      return;
>  try {
>      Callable<V> c = callable;
>      if (c != null && state == NEW) {
>          V result;
>          boolean ran;
>          try {
>          	// 将线程运行结果存储在result属性中，并设置状态为运行成功
>              result = c.call();
>              ran = true;
>          } catch (Throwable ex) {
>          	// 运行异常则设置异常状态
>              result = null;
>              ran = false;
>              setException(ex);
>          }
>          if (ran)
>              set(result);
>      }
>  } finally {
>      runner = null;
>      int s = state;
>      if (s >= INTERRUPTING)
>          handlePossibleCancellationInterrupt(s);
>  }
> }
> ```
>
> 

自定义一个类实现Callable接口，覆写Callable接口的call()方法，接着将自定义类对象作为参数创建一个FutureTask对象，然后将FutureTask对象作为参数创建一个Thread对象，调用Thread对象的start()方法即可启动线程，线程运行结果可以通过FutureTask对象来获取（原因已在上面分析，具体可以去参考源码）。

举个例子：

```java
public class MyCallable implements Callable<String> {
    @Override
    public String call() throws Exception {
        for(int i = 0; i < 100; i++) {
            System.out.println(Thread.currentThread() + ": " + i);  // currentThread()获取当前线程名称
        }
        return Thread.currentThread().getName();
    }
}
```

测试验证如下：

```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
    MyCallable c = new MyCallable();
    List<FutureTask<String>> ftList = new ArrayList<>();
    for(int i = 0; i < 5; i++) {
        FutureTask<String> ft = new FutureTask<>(c);
        ftList.add(ft);
        Thread t = new Thread(ft);
        t.start();
    }
    for(FutureTask<String> ft: ftList) {
        System.out.println(ft.get());
    }
}
```

执行结果如下两图，可以看到多个线程交替运行，并且都有返回值：

![thread_ankvsn](.\images\thread_ankvsn.png)

![thread_ahrejni](.\images\thread_ahrejni.png)



## 3. 线程组



## 4.线程池

### 4.1 Executor框架介绍

`Executor` 框架是 Java 在 **JDK 1.5** 引入的并发框架的一部分，旨在**简化多线程编程、统一任务提交与执行机制、实现线程池复用和资源控制**，此外，还能有效避免this逃逸出现。。它是 `java.util.concurrent` 包中的核心模块之一，也是实现高性能并发程序的重要基础。

> ### this逃逸介绍
>
> 所谓 **this 逃逸**，就是这个“还没做完的对象”**被别人提前拿走了用**。比如说你刚创建了一个对象，还在构造函数里初始化各种属性；但你在初始化一半的时候，把这个对象注册给了某个“监听器”（比如：广播站）；那个监听器说：“好的，我拿到了你的 this”，就开始调用你对象里的方法！结果你都还没准备好（对象还没构造完），人家就开始调用你了，可能出错，可能得到的是不完整的结果。

如下是整个Executor框架的结构图，此外，还有个Executors类作为工具类来辅助使用这套框架：

![threadpool_ionvca](E:\各种资料\Java开发笔记\我的笔记\images\threadpool_ionvca.png)

#### 4.1.1 Executor

```java
public interface Executor {
    void execute(Runnable command);
}
```

- 最基础的接口，定义了 **任务提交的入口：`execute()` 方法**。
- 实现类会决定如何运行这个 `Runnable`。
- 不关心是新开线程、复用线程，还是延迟执行。



#### 4.1.2 ExecutorService

```java
public interface ExecutorService extends Executor {
    void shutdown();
    List<Runnable> shutdownNow();
    Future<?> submit(Runnable task);
    ...
}
```

- 是 **真正的线程池接口**，添加了：
  - **提交任务（submit）**
  - **关闭线程池（shutdown）**
  - **获取任务结果（Future）**



#### 4.1.3 ScheduledExecutorService

（定时任务支持）

```java
public interface ScheduledExecutorService extends ExecutorService {
    ScheduledFuture<?> schedule(...);
    ScheduledFuture<?> scheduleAtFixedRate(...);
}
```

- 增加了定时调度能力：可实现延时任务、周期任务。



#### 4.1.4 AbstractExecutorService

（骨架实现类）

```java
public abstract class AbstractExecutorService implements ExecutorService {
    // 实现了 submit 方法，但不关心线程池细节
}
```

- 抽象类，提供了一些公共方法的实现。
- 减少子类实现负担，例如将 `Runnable` 包装成 `FutureTask`。



#### 4.1.5 ThreadPoolExecutor

（最核心的线程池实现）

```java
public class ThreadPoolExecutor extends AbstractExecutorService
```

- 支持核心线程数、最大线程数、任务队列、拒绝策略等。
- 通过 `Executors.newFixedThreadPool()`、`newCachedThreadPool()` 底层都是它。

适合：大多数常规异步任务。



#### 4.1.6 ScheduledThreadPoolExecutor

（支持定时/周期任务）

```java
public class ScheduledThreadPoolExecutor extends ThreadPoolExecutor
    implements ScheduledExecutorService
```

- 支持延迟执行、周期执行。
- 通过 `Executors.newScheduledThreadPool()` 创建。
- 同时支持线程池功能和调度功能。

适合：心跳检测、日志轮询、定时任务等。



#### 4.1.7 ForkJoinPool

（任务分而治之）

```java
public class ForkJoinPool extends AbstractExecutorService
```

- 支持 **分治并行计算**，适合大任务切小任务并发处理。
- 使用的是“工作窃取”算法，适合 CPU 密集型任务。
- `Java 8` 中 `parallelStream()` 的底层线程池。

适合：图像处理、大规模数据处理等递归任务。




#### 4.1.8 ListeningExecutorService

（来自 Guava 扩展）

```java
public interface ListeningExecutorService extends ExecutorService {
    ListenableFuture<?> submit(...);
}
```

- 这是 Google Guava 提供的扩展接口，不是 JDK 自带。
- 可以监听任务完成（而非主动 get）——更方便响应异步结果。




### 4.2 ThreadPoolExecutor介绍

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.acc = System.getSecurityManager() == null ?
            null :
            AccessController.getContext();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

- corePoolSize：线程池中用来工作的核心线程数量。
- maximumPoolSize：最大线程数，线程池允许创建的最大线程数。
- keepAliveTime：超出 corePoolSize 后创建的线程存活时间或者是所有线程最大存活时间，取决于配置。
- unit：keepAliveTime 的时间单位。
- workQueue：任务队列，是一个阻塞队列，当线程数达到核心线程数后，会将任务存储在阻塞队列中。
- threadFactory ：线程池内部创建线程所用的工厂。
- handler：拒绝策略；当队列已满并且线程数量达到最大线程数量时，会调用该方法处理任务。



## 5. Java内存模型

在学习Java内存模型之前需要先了解JVM的内存结构[ [跳转到JVM内存结构](##1-jvm内存结构) ]。

Java内存模型（JMM）是Java语言规范的一部分，定义了多线程环境下共享变量的访问规则。

### 5.1 并发编程特性

**原子性**

​	一次操作或者多次操作，要么所有的操作全部都得到执行并且不会受到任何因素的干扰而中断，要么都不执行。

**可见性**

​	当一个线程对共享变量进行了修改，那么另外的线程都是立即可以看到修改后的最新值。

**有序性**

​	由于指令重排序问题，代码的执行顺序未必就是编写代码时候的顺序。

> **指令重排序可以保证串行语义一致，但是没有义务保证多线程间的语义也一致** ，所以在多线程下，指令重排序可能会导致一些问题。



### 5.2 Java并发问题产生原因

#### 5.2.1 多线程并发内存结构

> **CPU缓存模型**
>
> ​	为了缓解CPU和内存间读写速度差异导致的性能问题，现代计算机通常在CPU和内存间添加三级缓存，越接近CPU的缓存读写越快，但存储容量越小，如下图所示。
>
> ![cpu_cache_njkcnad](.\images\cpu_cache_njkcnad.png)
>
> ​	**CPU缓存的工作方式：** 先将内存数据加载到CPU缓存中，当 CPU 需要用到的时候就可以直接从CPU缓存中读取数据，当运算完成后，再将运算得到的数据写回内存中。但是，这样存在**内存缓存不一致性的问题** ！
>
> ​	**CPU 为了解决内存缓存不一致性问题可以通过制定缓存一致协议（比如 MESI 协议）或者其他手段来解决。** 
>
> 
>
> #### ~~MESI 协议~~
>
> ~~MESI 协议，全称为 Modified, Exclusive, Shared, Invalid，是一种高速缓存一致性协议。它是为了解决多处理器（CPU）在并发环境下，多个 CPU 缓存不一致问题而提出的。~~
> ~~MESI 协议定义了高速缓存中数据的四种状态：~~
>
> 1. ~~**Modified（M）**：表示缓存行已经被修改，但还没有被写回主存储器。在这种状态下，只有一个 CPU 能独占这个修改状态。~~
> 2. ~~**Exclusive（E）**：表示缓存行与主存储器相同，并且是主存储器的唯一拷贝。这种状态下，只有一个 CPU 能独占这个状态。~~
> 3. ~~**Shared（S）**：表示此高速缓存行可能存储在计算机的其他高速缓存中，并且与主存储器匹配。在这种状态下，各个 CPU 可以并发的对这个数据进行读取，但都不能进行写操作。~~
> 4. ~~**Invalid（I）**：表示此缓存行无效或已过期，不能使用。~~
>
> ~~MESI 协议的主要用途是确保在多个 CPU 共享内存时，各个 CPU 的缓存数据能够保持一致性。当某个 CPU 对共享数据进行修改时，它会将这个数据的状态从 S（共享）或 E（独占）状态转变为 M（修改）状态，并等待适当的时机将这个修改写回主存储器。同时，它会向其他 CPU 广播一个“无效消息”，使得其他 CPU 将自己缓存中对应的数据状态转变为I（无效）状态，从而在下次访问这个数据时能够从主存储器或其他 CPU 的缓存中重新获取正确的数据。~~
>
> ~~这种协议可以确保在多处理器环境中，各个 CPU 的缓存数据能够正确、一致地反映主存储器中的数据状态，从而避免由于缓存不一致导致的数据错误或程序异常。~~

​	Java多线程并发执行时，每个线程会有一个本地内存，同时所有线程会共用主内存，结构如下图：

![jvm_cmskaf](.\images\jvm_cmskaf.jpg)

​	**主内存**：所有线程创建的实例对象都存放在主内存中，不管该实例对象是成员变量，还是局部变量，类信息、常量、静态变量都是放在主内存中。为了获取更好的运行速度，虚拟机及硬件系统可能会让工作内存优先存储于寄存器和高速缓存中。

​	**本地内存**：每个线程都有一个私有的本地内存，本地内存存储了该线程以读 / 写共享变量的副本。每个线程只能操作自己本地内存中的变量，无法直接访问其他线程的本地内存。如果线程间需要通信，必须通过主内存来进行。本地内存是 JMM 抽象出来的一个概念，并不真实存在，它涵盖了缓存、写缓冲区、寄存器以及其他的硬件和编译器优化。

​	若没有JMM，则各个线程的本地内存与主内存间的交互会出现可见性问题，最终导致程序运行错误。



#### 5.2.2 重排序

​	Java 源代码会经历 **编译器优化重排 —> 指令并行重排 —> 内存系统重排** 的过程，最终才变成操作系统可执行的指令序列。

**（1）编译器重排序**

​	针对程序代码语而言，编译器可以在**不改变单线程程序语义的情况下**，可以对代码语句顺序进行调整重新排序。

**（2）指令集并行的重排序**

​	这个是针对于CPU指令级别来说的，处理器采用了指令集并行技术来将多条指令重叠执行，如果不存在数据依赖性，处理器可以改变语句对应的机器指令执行顺序。

**（3）内存重排序**

​	因为CPU缓存使用缓冲区的方式(Store Buffere )进行延迟写入，这个过程会造成多个CPU缓存可见性的问题，这种可见性的问题导致结果的对于指令的先后执行显示不一致，从表面结果上来看好像指令的顺序被改变了，内存重排序其实是造成可见性问题的主要原因所在。



### 5.3 JMM规则

#### 5.3.1 八种同步操作

​	八种同步操作是JMM的抽象描述，用于定义线程之间对共享变量的操作顺序和可见性规则。它们是JMM的逻辑模型的一部分，用于帮助程序员理解线程之间的内存交互。

![jmm_cnsknv](.\images\jmm_cnsknv.png)

- **锁定（lock)**: 作用于主内存中的变量，将他标记为一个线程独享变量。

- **解锁（unlock）**: 作用于主内存中的变量，解除变量的锁定状态，被解除锁定状态的变量才能被其他线程锁定。
- **read（读取）**：作用于主内存的变量，它把一个变量的值从主内存传输到线程的工作内存中，以便随后的 load 动作使用
- **load(载入)**：把 read 操作从主内存中得到的变量值放入工作内存的变量的副本中。
- **use(使用)**：把工作内存中的一个变量的值传给执行引擎，每当虚拟机遇到一个使用到变量的指令时都会使用该指令。
- **assign（赋值）**：作用于工作内存的变量，它把一个从执行引擎接收到的值赋给工作内存的变量，每当虚拟机遇到一个给变量赋值的字节码指令时执行这个操作。
- **store（存储）**：作用于工作内存的变量，它把工作内存中一个变量的值传送到主内存中，以便随后的 write 操作使用。
- **write（写入）**：作用于主内存的变量，它把 store 操作从工作内存中得到的变量的值放入主内存的变量中。

**规则**

- read和load，write和store必须成对出现，顺序执行（但不用连续执行）
- assign操作不允许丢弃，即，工作内存中变量改变必须同步给主内存
- use前必须有load，store前必须有assign
- 同一时间一个变量只能被一个线程lock，但该线程可对其lock多次，lock多少次，必须对应unlock对应次数才能解锁
- 如果一个线程lock了某个变量，改变量在工作内存中的值会被清空，使用前必须
- unlock前必须要把该变量写回主存



#### 5.3.2 happens-before规则

​	JDK1.5版本中的Java内存模型中引入了Happens-Before原则。如果两个操作不满足任意一个 happens-before 规则，那么这两个操作就没有顺序的保障，JVM 可以对这两个操作进行重排序。

![happens_before_sanjkvds](.\images\happens_before_sanjkvds.png)

- **程序次序规则**（Program Order Rule）：在**一个线程内**，按照程序代码顺序，书写在前面的操作先行发生于书写在后面的操作。准确地说，应该是控制流顺序而不是程序代码顺序，因为要考虑分支、循环等结构。
- **管程锁定规则**（Monitor Lock Rule）：一个unlock操作先行发生于后面对**同一个锁**的lock操作。这里必须强调的是同一个锁，而“后面”是指时间上的先后顺序。
- **volatile变量规则**（Volatile Variable Rule）：对一个volatile变量的写操作先行发生于后面对这个变量的读操作，这里的“后面”同样是指时间上的先后顺序。
- **线程启动规则**（Thread Start Rule）：Thread对象的start()方法先行发生于此线程的每一个动作。
- **线程终止规则**（Thread Termination Rule）：线程中的所有操作都先行发生于对此线程的终止检测，我们可以通过Thread.join()方法结束、Thread.isAlive()的返回值等手段检测到线程已经终止执行。
- **线程中断规则**（Thread Interruption Rule）：对线程interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生，可以通过Thread.interrupted()方法检测到是否有中断发生。
- **对象终结规则**（Finalizer Rule）：一个对象的初始化完成（构造函数执行结束）先行发生于它的finalize()方法的开始。
- **传递性**（Transitivity）：如果操作A先行发生于操作B，操作B先行发生于操作C，那就可以得出操作A先行发生于操作C的结论。



#### 5.3.3 内存屏障

​	在复杂的多线程环境下，编译器和处理器是根本无法分析代码指令的依赖关系的，因此程序员需要显示的告诉编译器和处理器哪些地方是存在逻辑依赖的，这些地方不能进行重排序。

​	**编译器层面**和**CPU层面**都提供了一套内存屏障来禁止重排序的指令，编码人员需要识别存在数据依赖的地方加上一个内存屏障指令，那么此时计算机将不会对其进行指令优化。

​	常见的内存屏障类型包括：

- **LoadLoad屏障**：放置在两个读操作之间，确保第一个操作的结果对第二个操作可见。
- **StoreStore屏障**：放置在两个写操作之间，确保第一个写操作的结果对第二个写操作可见。
- **LoadStore屏障**：放置在读操作之后，写操作之前，确保读操作的结果在写操作发生之前可见。
- **StoreLoad屏障**：放置在写操作之后，读操作之前，确保写操作的结果对后续的读操作可见。

​	内存屏障是JVM在运行时插入的，用于**确保JMM定义的同步操作能够正确执行**。它们是实现JMM同步操作的具体技术手段。



## 6. Java并发的设计思想

**Java 的锁都是基于对象的**，而一个对象的“锁”存放在对象内存空间的对象头的mark word部分，详细可见JVM对象内存结构部分[ [跳转到对象内存结构](#17-对象内存结构) ]。

### 6.1 中断协商机制

#### 6.1.1 概述

一个线程不应该由其他线程来强制中断或停止，而是应该由线程自己停止，所以Thread.stop，Thread.suspend，Thread.resume都已经被废弃了。

在Java中没有办法立即停止一条线程，然而停止线程却显得尤为重要，如取消一个耗时操作。因此Java提供了一种**用于停止线程**的协商机制——中断，即中断标识协商机制。

#### 6.1.2 原理

- interrupt()

  ```java
  public void interrupt();
  ```

  线程内部存在着一个名为interrupt flag的标识，如果一个线程被interrupt：

  - 如果线程处于**正常活动状态**，那么会将该线程的**中断标志设置为true**，仅此而已。被设置中断标志的线程将继续正常运行，不受影响。所以，interrupt()并不能真正的中断线程，需要被调用的线程自己进行配合才行。

    当线程的中断标志位设置为true后，如果执行了可中断的阻塞代码（下面有举例），那么就会立刻清除中断标志位并设置为false，然后不进入阻塞状态直接继续执行代码。

  - 如果线程处于**被阻塞状态**（即调用如下例举的可中断方法），在别的线程中调用当前线程对象的interrupt方法，那么线程将立即**退出被阻塞的状态**，**清除标志位**并**抛出一个InterruptedException异常**。

  > 可中断方法：
  >
  > Object的wait方法
  >
  > Object的wait(long)方法
  >
  > Object的wait(long, int)方法
  >
  > Thread的sleep(long)方法
  >
  > Thread的sleep(long, int)方法
  >
  > Thread的join方法
  >
  > Thread的join(long)方法
  >
  > Thread的join(long, int)方法
  >
  > InterruptibleChannel的io操作
  >
  > Selector的wakeup方法
  >
  > TimeUnit中的sleep()方法
  >
  > 其他方法

  **注意：**LockSupport中的park()方法也会被interrupt()方法中断，但是不会将中断标记清除并设置为false，而是继续保持true，并且不会抛出InterruptedException异常。

- interrupted()

  ```java
  public static boolean interrupted() {
      return currentThread().isInterrupted(true);
  }
  ```

  通过JNI方法isInterrupted(boolean ClearInterrupted)传入true参数来实现。ClearInterrupted参数表示是否清空中断标志位。

  判断线程是否被中断，并清除当前中断状态。

  注意该方法一共做了两件事：

  1. 返回当前线程的中断状态。
  2. 将当前线程的中断状态清零并重新设置为false，清除线程的中断状态。

  因此，如果连续两次调用该方法，无论第一次返回true还是false，第二次调用都是返回false。

- isInterrupted()

  ```java
  public boolean isInterrupted() {
      return isInterrupted(false);
  }
  ```
  
  通过JNI方法isInterrupted(boolean ClearInterrupted)传入false参数来实现。
  
  仅仅判断线程是否被中断，不会清空中断标志位。

#### 6.1.3 使用

##### 6.1.3.1 线程停止

- volatile变量实现

  ```java
  public class InterruptDemo {
      static volatile boolean isStop = false;
      
      public static void main(String[] args) {
          new Thread(() -> {
              while(true) {
                  if(isStop) {
                      System.out.println(Thread.currentThread().getName + "\t isStop被修改为true，程序停止");
                      break;
                  }
                  System.out.println("t1 ---hello volatile");
              }
          }, name:"t1").start();
          
          try {
              TimeUnit.MILLISECONDS.sleep(20);
          } catch(InterruptedException e) {
              e.printStackTrace();
          }
          
          new Thread(() -> {
              isStop = true;
          }, name:"t2").start();
      }
  }
  ```

- AtomicBoolean类对象实现

  ```java
  public class InterruptDemo {
      static AtomicBoolean atomicBoolean = new AtomicBoolean(false);
      
      public static void main(String[] args) {
          new Thread(() -> {
              while(true) {
                  if(atomicBoolean.get()) {
                      System.out.println(Thread.currentThread().getName + "\t atomicBoolean被修改为true，程序停止");
                      break;
                  }
                  System.out.println("t1 ---hello atomicBoolean");
              }
          }, name:"t1").start();
          
          try {
              TimeUnit.MILLISECONDS.sleep(20);
          } catch(InterruptedException e) {
              e.printStackTrace();
          }
          
          new Thread(() -> {
              atomicBoolean.set(true);
          }, name:"t2").start();
      }
  }
  ```

- Thread类自带中断方法实现

  ```java
  public class InterruptDemo {
      public static void main(String[] args) {
          Thread t1 = new Thread(() -> {
              while(true) {
                  if(Thread.currentThread().isInterrupted()) {
                      System.out.println(Thread.currentThread().getName + "\t isInterrupted()被修改为true，程序停止");
                      break;
                  }
                  System.out.println("t1 ---hello interrupt api");
              }
          }, name:"t1");
          t1.start();
          
          try {
              TimeUnit.MILLISECONDS.sleep(20);
          } catch(InterruptedException e) {
              e.printStackTrace();
          }
          
          new Thread(() -> {
              t1.interrupt();
          }, name:"t2").start();
      }
  }
  ```


##### 6.1.3.2 中断阻塞状态

```java
public class InterruptDemo {
    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            while(true) {
                if(Thread.currentThread().isInterrupted()) {
                    System.out.println(Thread.currentThread().getName()+"\t" +
                            "中断标志位："+Thread.currentThread().isInterrupted()+" 程序停止");
                    break;
                }

                try {
                    Thread.sleep(200);
                } catch (InterruptedException e) {
                    // 注意，在阻塞状态时被中断不仅抛出异常，还会清空中断标志位。
                    // 因此需要重新设置中断标志位，来保证依赖中断标志位的程序逻辑正确。
                    Thread.currentThread().interrupt();
                    e.printStackTrace();
                }

                System.out.println("----hello InterruptDemo");
            }
        }, "t1");
        t1.start();

        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        new Thread(() -> t1.interrupt(), "t2").start();
    }
}
```

在使用中断来打断阻塞状态时需要注意，在阻塞状态时被中断不仅抛出异常，还会清空中断标志位，因此需要在异常处理代码块中重新设置中断标志位，来保证依赖中断标志位的程序逻辑正确。



### 6.2 等待唤醒机制

#### 6.2.1 概述



#### 6.2.2 原理

##### 6.2.2.1 wait() / notify()

由于wait()和notify()的实现原理是基于synchronized的重量级锁状态下的monitor机制来实现的，因此需要先了解synchronized的重量级锁的实现原理：[跳转到synchronized重量级锁](######7.2.1.4 重量级锁)。

从monitor的实现原理可以知道，monitor对象内部维护者一个EntryList和一个WaitSet：

- 每当有一个线程要竞争获取锁对象的时候，都会被加入到EntryList，当占用锁对象的线程结束并释放锁对象后，EntryList里面的线程会共同竞争锁对象。

- 当线程占用锁对象运行过程中，调用了wait()方法，此时线程就会被加入到WaitSet中，直到被其他线程调用notify()/notifyall()方法或者等待时间结束后，才从WaitSet取出放入EntryList中继续竞争锁对象。

notify()方法随机唤醒一个WaitSet中的线程并放入EntryList，而notifyall()方法则唤醒WaitSet中的所有线程并放入EntryList，一旦线程被唤醒放入EntryList，即使没有立即获得到锁对象，后续也不需要重新被notify就可以直接竞争锁对象。

**注意：**由于wait()和notify()这套等待唤醒方法是基于monitor的，因此一旦使用了wait()和notify()这套等待唤醒方法，synchronized就会直接升级到重量级锁，放弃偏向锁或轻量级锁优化，此时线程间的同步会涉及到大量的内核切换造成性能下降。因此，若对性能要求高，请考虑使用 `LockSupport`、`ReentrantLock`、`Condition` 等并发工具，它们可以更细粒度地控制锁行为。

##### 6.2.2.2 await() / signal()



##### 6.2.2.3 LockSupport





#### 6.2.3 使用

##### 6.2.3.1 wait() / notify()

**限制**

- wait和notify方法必须要在同步块或者方法里面，否则会报错。

- wait和notify方法需要成对出现使用，先使用wait方法，后使用notify，否则可能会导致调用wait方法的线程一直陷入等待状态。

**正确使用示例**

```java
public class WaitNotifyDemo {
    public static void main(String[] args) {
        // 锁对象
        Object objectLock = new Object();
        
        // 等待线程t1
        new Thread(() -> {
           synchronized (objectLock) {
               System.out.println(Thread.currentThread().getName() + ": " + "获取锁...");
               try {
                   objectLock.wait();
               } catch (InterruptedException e) {
                   throw new RuntimeException(e);
               }
               System.out.println(Thread.currentThread().getName() + ": " + "被唤醒...");
           }
        }, "t1").start();

        // 保证唤醒线程t2在等待线程t1进入等待之后才去进行唤醒
        try {
            TimeUnit.MILLISECONDS.sleep(200);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }

        // 唤醒线程t2
        new Thread(() -> {
            synchronized (objectLock) {
                objectLock.notify();
                System.out.println(Thread.currentThread().getName() + ": " + "完成唤醒...");
            }
        }, "t2").start();
    }
}
```

运行结果：

```
t1: 获取锁...
t2: 完成唤醒...
t1: 被唤醒...
```

##### 6.2.3.2 await() / signal()

**限制**

- await和signal方法必须先调用condition对象对应的lock对象的lock()方法获取锁，否则会报错。要注意不用锁之后要调用unlock()方法释放锁。
- await和signal方法需要成对出现使用，先使用await方法，后使用signal，否则可能会导致调用await方法的线程一直陷入等待状态。

**对比**

- await() / signal()方法和wait() / notify()的限制是类似的，并没有解决wait() / notify()的限制，但是能够摆脱对synchronized的重量级锁的依赖，与synchronzied搭配使用的时候可以发挥偏向锁和轻量级锁优化的优势。

**正确使用示例**

```java
public class AwaitSignalDemo {
    public static void main(String[] args) {
        Lock lock = new ReentrantLock();
        Condition condition = lock.newCondition();
        Thread t1 = new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + "：开始线程");
            lock.lock();
            try {
                condition.await();
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            } finally {
                lock.unlock();
            }
            System.out.println(Thread.currentThread().getName() + "：线程被唤醒");
        }, "t1");
        t1.start();

        try {
            TimeUnit.MILLISECONDS.sleep(2000);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }

        Thread t2 = new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + "：开始线程");
            lock.lock();
            try {
                condition.signal();
            } finally {
                lock.unlock();
            }
            System.out.println(Thread.currentThread().getName() + "：唤醒线程t1");
        }, "t2");
        t2.start();
    }
}
```

运行结果：

```
t1：开始线程
t2：开始线程
t2：唤醒线程t1
t1：线程被唤醒
```



##### 6.2.3.3 LockSupport

**限制**

- LockSupport没有前面两种方法的限制
- LockSupport由于实现原理限制，因此提前使用unpark的话，被unpark线程最多只能存储一个通行证，因此连续的unpark最后只能唤醒一次线程，只有unpark和park方法交替使用，才能将线程唤醒多次。

**对比**

- LockSupport的park()和unpark()不需要获取锁就能直接让线程等待或者唤醒。
- LockSupport的park()和unpark()的实现原理使得其对park()和unpark()调用顺序没有要求，即使先使用unpark()方法对线程进行唤醒，后调用park()让线程进入等待，也能使得线程结束等待。
- LockSupport的park() / unpark()与await() / signal()方法一样，不会导致synchronized的锁升级为重量级锁。

**示例**

”先等待，后唤醒“示例：

```java
public class LockSupportDemo {
    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + ": 线程启动");
            LockSupport.park();
            System.out.println(Thread.currentThread().getName() + ": 线程结束阻塞");
        }, "t1");
        t1.start();

        try {
            TimeUnit.MILLISECONDS.sleep(200);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }

        new Thread(() -> {
            LockSupport.unpark(t1);
            System.out.println(Thread.currentThread().getName() + ": 唤醒t1线程");
        }, "t2").start();
    }
}
```

输出结果：

```
t1: 线程启动
t2: 唤醒t1线程
t1: 线程结束阻塞
```



”先唤醒，后阻塞“示例

```java
public class LockSupportDemo {
    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            System.out.println(Thread.currentThread().getName() + ": 线程启动");
            LockSupport.park();
            System.out.println(Thread.currentThread().getName() + ": 线程结束阻塞");
        }, "t1");
        t1.start();

        new Thread(() -> {
            LockSupport.unpark(t1);
            System.out.println(Thread.currentThread().getName() + ": 唤醒t1线程");
        }, "t2").start();
    }
}
```

输出结果：

```
t2: 唤醒t1线程
t1: 线程启动
t1: 线程结束阻塞
```



### 6.3 CAS

#### 6.3.1 概述

`CAS` 全名 `Compare and Swap` 。CAS 是比较并交换的意思，用于在硬件层面上提供原子性操作。在某些处理器架构（如x86）中，比较并交换通过指令 CMPXCHG 实现（（Compare and Exchange），一种原子指令），通过比较是否和给定的数值一致，如果一致则修改，不一致则不修改。

在 `CAS` 中，有这样三个值：

- V：要更新的变量(var)
- E：预期值(expected)
- N：新值(new)

比较并交换的过程如下：

判断 V 是否等于 E，如果等于，将 V 的值设置为 N；如果不等，说明已经有其它线程更新了 V，于是当前线程放弃更新，什么都不做。这里的**预期值 E 本质上指的是“旧值”**。当多个线程同时使用 CAS 操作一个变量时，只有一个会成功更新，其余均会失败，但失败的线程并不会被挂起，仅是被告知失败，并且允许再次尝试，当然也允许失败的线程放弃操作。

#### 6.3.2 原理

CAS 是一种原子操作，它是一种系统原语，是一条 CPU 的原子指令，从 CPU 层面已经保证它的原子性。

在 Java 中，有一个`Unsafe`类，它在`sun.misc`包中。它里面都是一些`native`方法，其中就有几个是关于 CAS 的：

```java
boolean compareAndSwapObject(Object o, long offset,Object expected, Object x);
boolean compareAndSwapInt(Object o, long offset,int expected,int x);
boolean compareAndSwapLong(Object o, long offset,long expected,long x);
```

这些CAS原子操作都判断待修改的变量的值是否等于与预期值expected，等于则将变量的值修改为目标值x并返回true；不等于则返回false。

Unsafe 对 CAS 的实现是通过 C++ 实现的，它的具体实现和操作系统、CPU 都有关系。

#### 6.3.3 使用

模板（假设多线程对int类型数据进行更新）：

```java
Unsafe unsafe = Unsage.getUnsafe();  // Unsafe类是单例模式实现的
public boolean add(Object o, long offset, int delta) {
    int v;
    do {
        v = unsafe.getIntVolatile(o, offset);
    } while(!unsafe.compareAndSwapInt(o, offset, v, v + delta));
}
```

这里Unsafe的getIntVolatile(Object, long)主要是用来获取当前时刻待修改的变量的值，这里用到了和volatile同理的方式获取。

通过不断轮询查看待修改变量是否发生改变，一旦不发生改变则完成更新操作。

具体的使用可以学习Unsafe类里面提供的其他方法，比如getAndAddInt()、getAndAddLong()等等方法，这些方法都利用CAS思想来实现线程安全的数据更新操作。

#### 6.3.4 问题

尽管 CAS 提供了一种有效的同步手段，但也存在一些问题，主要有以下三个：**ABA 问题**、**长时间自旋**、**多个共享变量的原子操作**。

##### 6.3.4.1 ABA 问题

所谓的 ABA 问题，就是一个值原来是 A，变成了 B，又变回了 A。这个时候使用 CAS 是检查不出变化的，但实际上却被更新了两次。

ABA 问题的解决思路是在变量前面追加上**版本号或者时间戳**。从 JDK 1.5 开始，JDK 的 atomic 包里提供了一个类`AtomicStampedReference`类来解决 ABA 问题。

这个类的`compareAndSet`方法的作用是首先检查当前引用是否等于预期引用，并且检查当前标志是否等于预期标志，如果二者都相等，才使用 CAS 设置为新的值和标志。

```
public boolean compareAndSet(V   expectedReference,
                              V   newReference,
                              int expectedStamp,
                              int newStamp) {
    Pair<V> current = pair;
    return
        expectedReference == current.reference &&
        expectedStamp == current.stamp &&
        ((newReference == current.reference &&
          newStamp == current.stamp) ||
          casPair(current, Pair.of(newReference, newStamp)));
}
```

先来看参数：

- expectedReference：预期引用，也就是你认为原本应该在那个位置的引用。
- newReference：新引用，如果预期引用正确，将被设置到该位置的新引用。
- expectedStamp：预期标记，这是你认为原本应该在那个位置的标记。
- newStamp：新标记，如果预期标记正确，将被设置到该位置的新标记。

执行流程：

①、`Pair<V> current = pair;` 这行代码获取当前的 pair 对象，其中包含了引用和标记。

②、接下来的 return 语句做了几个检查：

- `expectedReference == current.reference && expectedStamp == current.stamp`：首先检查当前的引用和标记是否和预期的引用和标记相同。如果二者中有任何一个不同，这个方法就会返回 false。
- 如果上述检查通过，也就是说当前的引用和标记与预期的相同，那么接下来就会检查新的引用和标记是否也与当前的相同。如果相同，那么实际上没有必要做任何改变，这个方法就会返回 true。
- 如果新的引用或者标记与当前的不同，那么就会调用 casPair 方法来尝试更新 pair 对象。casPair 方法会尝试用 newReference 和 newStamp 创建的新的 Pair 对象替换当前的 pair 对象。如果替换成功，casPair 方法会返回 true；如果替换失败（也就是说在尝试替换的过程中，pair 对象已经被其他线程改变了），casPair 方法会返回 false。

##### 6.3.4.2 长时间自旋

CAS 多与自旋结合。如果自旋 CAS 长时间不成功，会占用大量的 CPU 资源。

解决思路是让 JVM 支持处理器提供的**pause 指令**。pause 指令能让自旋失败时 cpu 睡眠一小段时间再继续自旋，从而使得读操作的频率降低很多，为解决内存顺序冲突而导致的 CPU 流水线重排的代价也会小很多。

##### 6.3.4.3 多个共享变量的原子操作

当对一个共享变量执行操作时，CAS 能够保证该变量的原子性。但是对于多个共享变量，CAS 就无法保证操作的原子性，这时通常有两种做法：

1. 使用`AtomicReference`类保证对象之间的原子性，把多个变量放到一个对象里面进行 CAS 操作；
2. 使用锁。锁内的临界区代码可以保证只有当前线程能操作。



### 6.4 AQS

#### 6.4.1 概述

**AQS**是`AbstractQueuedSynchronizer`的简称，即`抽象队列同步器`，从字面上可以这样理解:

- 抽象：抽象类，只实现一些主要逻辑，有些方法由子类实现；
- 队列：使用先进先出（FIFO）的队列存储数据；
- 同步：实现了同步的功能。

AQS 是一个用来构建锁和同步器的公共基础部分的抽象实现，是重量级基础框架及整个JUC体系的基石，主要用于解决锁分配给“谁”的问题。其实现原理大致就是一个抽象的FIFO对垒来完成资源获取线程的排队工作， 并通过一个int类变量表示持有锁的状态。

使用 AQS 能简单且高效地构造出应用广泛的同步器，比如 ReentrantLock，Semaphore，ReentrantReadWriteLock，SynchronousQueue，FutureTask等等，都是基于 AQS 的。



#### 6.4.2 原理

##### 6.4.2.1 AQS 数据结构

**AQS（AbstractQueuedSynchronizer）** 是 Java 并发包中用于构建同步器（如 ReentrantLock、Semaphore、CountDownLatch 等）的基础框架。它通过一个共享的**状态变量 `state`** 和一个基于**双向链表的等待队列**来管理线程的同步与通信。

![AQS_ivnsdjo](E:\各种资料\Java开发笔记\我的笔记\images\AQS_ivnsdjo.png)



###### 6.4.2.1.1 状态变量 `state`

```java
private volatile int state;
```

AQS 通过一个 `volatile` 的整数变量 `state` 来表示同步状态。该变量可以由AQS的子类定义其具体含义，具体如下：

| 同步器类型               | 同步模式    | `state` 代表含义                       |
| ------------------------ | ----------- | -------------------------------------- |
| `ReentrantLock`          | 独占        | 锁重入次数                             |
| `Mutex`（自定义）        | 独占        | 是否已被占用（0/1）                    |
| `FutureTask`             | 独占        | 任务执行状态                           |
| `Semaphore`              | 共享        | 剩余许可证数量                         |
| `CountDownLatch`         | 共享        | 倒计数值                               |
| `ReentrantReadWriteLock` | 独占 + 共享 | 高 16 位：读锁计数；低 16 位：写锁计数 |

为保证线程安全，AQS 提供了以下三个受保护的方法来访问和修改 `state`：

```java
protected final int getState() {
    return state;
}

protected final void setState(int newState) {
    state = newState;
}

protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

其中，`compareAndSetState` 是一个原子操作，依赖于底层的 `Unsafe` 类，通过 CAS（Compare-And-Swap）机制确保并发修改的正确性。



###### 6.4.2.1.2 等待队列（FIFO 双向队列）

当线程获取同步资源失败时，AQS 会将其封装成一个 `Node` 对象，加入到一个 FIFO 的 **等待队列（CLH 队列）** 中，直到它被唤醒。

AQS 使用两个指针：

- `head`：指向队列头节点（一个哑元节点，不代表任何线程）
- `tail`：指向队列尾部

```java
private transient volatile Node head;
private transient volatile Node tail;
```



###### 6.4.2.1.3 Node 节点结构

每个节点表示一个等待获取资源的线程，并保存着线程自身和前驱、后继节点的引用。结构如下：

```java
static final class Node {
    // 节点状态标识
    static final int CANCELLED =  1;
    static final int SIGNAL    = -1;
    static final int CONDITION = -2;
    static final int PROPAGATE = -3;

    volatile int waitStatus;
    volatile Node prev;   // 前驱节点
    volatile Node next;   // 后继节点
    volatile Thread thread; // 当前线程

    Node nextWaiter; // 用于Condition队列的指针（不同于next）
}
```

**关键字段说明：**

- `waitStatus`：标记当前节点的状态，如是否需要被唤醒或已取消等。

  | 枚举      | 含义                                                         |
  | :-------- | :----------------------------------------------------------- |
  | 0         | 当一个Node被初始化的时候的默认值。                           |
  | CANCELLED | 为1，表示节点因为超时或中断被取消，不再参与竞争。            |
  | SIGNAL    | 为-1，表示后继节点处于等待状态，当前节点在释放锁后需要通知它。 |
  | CONDITION | 为-2，表示当前节点在条件队列中，等待被 `signal()` 唤醒。     |
  | PROPAGATE | 为-3，表示共享模式下的传播行为，用于 `CountDownLatch` 等。   |

- `prev` / `next`：双向链表，用于维护阻塞队列。

- `thread`：记录当前节点关联的线程。

- `nextWaiter`：用于 `Condition` 条件队列的下一个节点，不同于阻塞队列的 `next`。

  | 模式      | 含义                           |
  | :-------- | :----------------------------- |
  | SHARED    | 表示线程以共享的模式等待锁     |
  | EXCLUSIVE | 表示线程正在以独占的方式等待锁 |



##### 6.4.2.2 加锁流程

**独占式锁**

![AQS_onvoab](E:\各种资料\Java开发笔记\我的笔记\images\AQS_onvoab.png)

> **疑惑：**
>
> 明明LockSupport.park()方法不会清除interrupt flag标志位，为什么还要大费周章的去判断并清除中断标志位，然后等出循环再判断是否被中断过重新设置中断标志位，明明可以就无论park是被正常唤醒还是中断唤醒，都不管中断标志位，毕竟他会一直保留着，直到被出队列后的业务逻辑代码清除。



**共享式锁**





##### 6.4.2.3 解锁流程

**独占式锁**

![AQS_ocbnas](E:\各种资料\Java开发笔记\我的笔记\images\AQS_ocbnas.png)

- unparkSuccessor(Node node)的compareAndSetWaitStatus这个CAS的目的是：**将当前节点的 `waitStatus` 从负数（`SIGNAL`）重置为 0，表示“我已经唤醒了后继节点，后面不用你再管了”**。但是即使 `CAS` 失败了（说明其他线程可能已经修改了 `waitStatus`，比如设置为 `CANCELLED`）：

  - **不会影响后面的线程唤醒逻辑**；
  - 也不会影响 AQS 队列结构或状态一致性；
  - 唤醒操作依然可以顺利进行。

  所以：**这一操作失败不重要，不值得为它付出性能代价重试**。



**共享式锁**









#### 6.4.3 使用

##### 6.4.3.4 支持的同步模式

AQS 支持两种同步模式：

- **独占模式（Exclusive）**：如 ReentrantLock，只有一个线程可以获取资源。
- **共享模式（Shared）**：如 Semaphore 或 CountDownLatch，多个线程可同时获取资源。

在共享模式下，唤醒线程时会传播给后续节点（节点的 `waitStatus` 为 `PROPAGATE`）。

#### 6.4.4 问题



## 7. Java并发工具介绍

### 7.1 volatile

#### 7.1.1底层原理

**有序性**

操作volatile变量的有序性通过**内存屏障**实现。

- 对于volatile变量的写操作，JVM会在写操作前插入StoreStore屏障，使得本次写操作之前的所有写操作执行完才会执行本次写操作，保证前面的所有写操作对本次写操作的可见性；在写操作后插入StoreLoad屏障，使得本次写操作执行完才会执行之后的读操作，保证本次写操作对于后面的读操作的可见性。

  ![volatile_mclnv](.\images\volatile_mclnv.jpg)

- 对于volatile变量的读操作，JVM会在读操作后插入LoadLoad屏障，使得本次读操作执行完才能只能后续的读操作，保证本次读操作对后续读操作的可见性；在读操作之后插入LoadStore屏障，保证在此次读操作执行完才能进行之后的写操作。

  ![volatile_asnjc](.\images\volatile_asnjc.jpg)

这些屏障确保volatile变量的读写操作对所有线程都是可见的，并且不会发生重排序。

**可见性**

操作volatile变量的可见性通过在每次线程进行volatile读的时候都从主内存读取，每次线程进行volatile写的时候都将数据写回主内存，从而保证了可见性。

**原子性**

volatile不保证操作的原子性。



#### 7.1.2 使用教程

- 先看下面未使用 volatile 的代码，假设writer和reader分别在两个不同线程中运行：

  ```
  class ReorderExample {
    int a = 0;
    boolean flag = false;
    public void writer() {
        a = 1;                   //1
        flag = true;             //2
    }
    public void reader() {
        if (flag) {                //3
            int i = a * a;         //4
            System.out.println(i);
        }
    }
  }
  ```

  因为重排序影响，所以最终的输出可能是 0。

  > ### 解释
  >
  > #### 理想情况
  >
  > 1. **线程A执行`writer`方法**：
  >    - `a = 1;` （操作1）
  >    - `flag = true;` （操作2）
  > 2. **线程B执行`reader`方法**：
  >    - `if (flag) {` （操作3）
  >    - `int i = a * a;` （操作4）
  >    - `System.out.println(i);` （操作5）
  >
  > 此时执行与起期望的逻辑顺序一样，输出为1。
  >
  > 
  >
  > #### 可能的特殊情况
  >
  > 在没有`volatile`关键字的情况下，JMM允许以下重排序：
  >
  > - **操作1和操作2之间的重排序**：`flag = true;` 可能被重排序到 `a = 1;` 之前。
  >
  > 假设发生了以下重排序：
  >
  > 1. **线程A**：
  >    - `flag = true;` （操作2）
  >    - `a = 1;` （操作1）
  > 2. **线程B**：
  >    - `if (flag) {` （操作3）
  >    - `int i = a * a;` （操作4）
  >    - `System.out.println(i);` （操作5）
  >
  > 由于指令重排序，`flag = true;`可能被重排序到`a = 1;`之前。因此，当线程B检查`flag`时，`flag`已经为`true`，但`a`的值可能还没有被更新为`1`。这导致线程B读取到的`a`的值仍然是`0`，从而计算出`i = a * a = 0`，最终输出`0`。

- 如果引入 volatile，我们再看一下代码：

  ```
  class ReorderExample {
    int a = 0;
    boolean volatile flag = false;
    public void writer() {
        a = 1;                   //1
        flag = true;             //2
    }
    public void reader() {
        if (flag) {                //3
            int i = a * a;         //4
            System.out.println(i);
        }
    }
  }
  ```

  这时候，volatile 会禁止指令重排序，这个过程建立在 happens before 关系的基础上：

  1. 根据程序次序规则，1 happens before 2; 3 happens before 4。
  2. 根据 volatile 规则，2 happens before 3。
  3. 根据 happens before 的传递性规则，1 happens before 4。

  上述 happens before 关系的图形化表现形式如下：

  ![volatile_onacncsa](.\images\volatile_onacncsa.jpg)

  > ## 解释
  >
  > 在`writer`方法中：
  >
  > - **`flag = true;`**：由于`flag`是`volatile`变量，会在写操作`flag = true;`之前插入`StoreStore`屏障和之后插入`StoreLoad`屏障。
  >   - **`StoreStore`屏障**：确保在`flag = true;`执行之前的所有普通写操作（如`a = 1;`）都已完成。
  >   - **`StoreLoad`屏障**：确保在`flag = true;`之后的所有读写操作不会被重排序到`flag = true;`之前。
  >
  > 这意味着`a = 1;`操作一定会在`flag = true;`之前完成，并且`flag = true;`操作会将`a`的值同步到主内存。
  >
  > 
  >
  > 在`reader`方法中：
  >
  > - **`if (flag)`**：由于`flag`是`volatile`变量，读操作`if (flag)`之后会插入`LoadLoad`屏障和`LoadStore`屏障。
  >   - **`LoadLoad`屏障**：确保在`if (flag)`之前后所有读操作都在if（flag）之后执行。
  >   - **`LoadStore`屏障**：确保在`if (flag)`之后的所有写操作不会被重排序到`if (flag)`之前。
  >
  > 这意味着，当`flag`为`true`时，`a`的值已经被写入主内存，并且`reader`方法能够读取到`a`的最新值。

- volatile 不适用的场景

  下面是变量自加的示例：

  ```
  public class volatileTest {
      public volatile int inc = 0;
      public void increase() {
          inc++;
      }
      public static void main(String[] args) {
          final volatileTest test = new volatileTest();
          for(int i=0;i<10;i++){
              new Thread(){
                  public void run() {
                      for(int j=0;j<1000;j++)
                          test.increase();
                  };
              }.start();
          }
          while(Thread.activeCount()>1)  //保证前面的线程都执行完
              Thread.yield();
          System.out.println("inc output:" + test.inc);
      }
  }
  ```

  测试输出：

  ```
  inc output:8182
  ```

  由于volatile不保证原子性，而++/--这类自增自减操作本身也不是原子操作，而是通过“取、改、赋值”三个步骤实现，这就导致并发对volatile变量进行更改的时候会出现并发问题。



#### 7.1.3 实战场景

**volatile 实现单例模式的双重锁**

下面是一个使用"双重检查锁定"（double-checked locking）实现的单例模式（Singleton Pattern）的例子。

```
public class Penguin {
    private static volatile Penguin m_penguin = null;

    // 一个成员变量 money
    private int money = 10000;

    // 避免通过 new 初始化对象，构造方法应为 private
    private Penguin() {}

    public void beating() {
        System.out.println("打豆豆" + money);
    }

    public static Penguin getInstance() {
        if (m_penguin == null) {
            synchronized (Penguin.class) {
                if (m_penguin == null) {
                    m_penguin = new Penguin();
                }
            }
        }
        return m_penguin;
    }
}
```

其中，使用 volatile 关键字是为了防止 `m_penguin = new Penguin()` 这一步被指令重排序。因为实际上，`new Penguin()` 这一行代码分为三个子步骤：

- 步骤 1：为 Penguin 对象分配足够的内存空间，伪代码 `memory = allocate()`。
- 步骤 2：调用 Penguin 的构造方法，初始化对象的成员变量，伪代码 `ctorInstanc(memory)`。
- 步骤 3：将内存地址赋值给 m_penguin 变量，使其指向新创建的对象，伪代码 `instance = memory`。

如果不使用 volatile 关键字，JVM 可能会对这三个子步骤进行指令重排。

- 为 Penguin 对象分配内存
- 将对象赋值给引用 m_penguin
- 调用构造方法初始化成员变量

这种**重排序会导致 m_penguin 引用在对象完全初始化之前就被其他线程访问到**。具体来说，如果一个线程执行到步骤 2 并设置了 m_penguin 的引用，但尚未完成对象的初始化，这时另一个线程可能会看到一个“半初始化”的 Penguin 对象。

假如此时有两个线程 A 和 B，要执行 `getInstance()` 方法：

```
public static Penguin getInstance() {
    if (m_penguin == null) {
        synchronized (Penguin.class) {
            if (m_penguin == null) {
                m_penguin = new Penguin();
            }
        }
    }
    return m_penguin;
}
```

- 线程 A 执行到 `if (m_penguin == null)`，判断为 true，进入同步块。
- 线程 B 执行到 `if (m_penguin == null)`，判断为 true，进入同步块。

如果线程 A 执行 `m_penguin = new Penguin()` 时发生指令重排序：

- 线程 A 分配内存并设置引用，但尚未调用构造方法完成初始化。
- 线程 B 此时判断 `m_penguin != null`，直接返回这个“半初始化”的对象。

这样就会导致线程 B 拿到一个不完整的 Penguin 对象，可能会出现空指针异常或者其他问题。

于是，我们可以为 m_penguin 变量添加 volatile 关键字，来禁止指令重排序，确保对象的初始化完成后再将其赋值给 m_penguin。



### 7.2 synchronized

#### 7.2.1 底层原理

在JDK1.6之前，synchronized底层使用的是重量级锁，在JDK1.6及以后的版本，synchronized优化出了锁升级的实现方式，有无锁、偏向锁、轻量级锁、重量级锁这几种锁状态，在JDK15废除了偏向锁机制。

![synchronized_monitor_pnckas](.\images\synchronized_monitor_pnckas.png)

![synchronized_monitor_xpsca](.\images\synchronized_monitor_xpsca.png)



**注意：**biased_lock为1并不代表此锁对象为偏向锁，而是代表该锁对象是可偏向的无锁状态，只有当MarkWord中存储了偏向的线程ID，此时才算是偏向锁状态。



##### 7.2.1.1 无锁

**概述**

无锁是一个对象被实例化后的初始状态，只要还没有任何线程竞争锁，那么它就一直保持无锁状态。



##### 7.2.1.2 偏向锁

**概述**

偏向锁用在单线程竞争，即**锁总是由同一线程多次获得**，偏向锁会偏向于第一个访问锁的线程，如果在接下来的运行过程中，该锁没有被其他的线程访问，则持有偏向锁的线程将永远不需要触发同步。也就是说，**偏向锁在资源无竞争情况下消除了同步语句**，连 `CAS` 操作都不做了，着极大地提高了程序的运行性能。但是如果线程间存在锁竞争，那么会太带来额外的所撤销的消耗。因此偏向锁适用于只有一个线程访问同步块的场景。

**实现原理**

在锁第一次被拥有的时候，记录下偏向线程的地址，这样偏向线程就一直持有者锁（后续这个线程进入和退出这段加了同步锁的代码块时，不需要再次加锁和释放锁。而是直接回去检查锁的MarkWord里面是不是放的自己的线程地址）。

当后续有线程希望获得锁时，就会先判断锁对象的MarkWord里存储的线程地址是否等于本线程地址：

- 如果是，表明该线程已经获得了锁，不需要花费 CAS 操作来加锁和解锁。

- 如果不是，就代表有另一个线程来竞争这个偏向锁。这个时候会尝试使用 CAS 来替换 Mark Word 里面的线程 ID 为新线程的 ID，这个时候要分两种情况：

  - 替换成功，表示之前的线程已经不占有锁了， Mark Word 里面的线程地址为新线程的地址，锁不会升级，仍然为偏向锁。

  - 替换失败，表示之前的线程仍占有锁，此时新旧线程形成竞争关系，那么会将偏向锁升级为轻量锁，然后以轻量锁的竞争方式进行锁的竞争。

**偏向锁机制**

当偏向锁机制开启后，所有对象都会变成可偏向状态，此时所有对象都处于偏向锁状态，只是MarkWord中的线程ID为0，可以简单理解为偏向锁机制开启后，默认的锁状态不是无锁状态，而是偏向锁状态。

锁的升级开销很大，如果应用程序里所有的锁通常处于竞争状态，那么偏向锁就会是一种累赘，对于这种情况，我们可以一开始就把偏向锁这个默认功能给关闭：

```
-XX:UseBiasedLocking=false
```

HotSpot JVM在启动时，默认会延迟4秒后才启用偏向锁机制。这个延迟时间可以通过JVM参数`-XX:BiasedLockingStartupDelay`来配置。默认情况下，`-XX:BiasedLockingStartupDelay`的值是**4000毫秒**（即4秒）。这意味着JVM启动后，偏向锁机制会在4秒后自动启用。如果需要调整这个延迟时间，可以通过设置`-XX:BiasedLockingStartupDelay`参数来实现。例如：

```
-XX:BiasedLockingStartupDelay=2000
```

**锁撤销**

如下两种情况进行锁撤销：

- 第一个线程正在执行synchronized方法(处于同步块)，它还没有执行完，其它线程来抢夺，该偏向锁会被取消掉并出现锁升级。
- 第一个线程执行完成synchronized方法(退出同步块)，则将对象头设置成无锁状态并撤销偏向锁，重新偏向。

**锁升级**

当且仅当偏向锁机制开启，并且有线程占用锁的时候从无锁升级为偏向锁。



##### 7.2.1.3 轻量级锁

**概述**

处于轻量级锁状态时，多线程利用CAS来轮询获取锁，因此竞争的线程不会阻塞，提高了程序的相应速度，但是始终得不到锁竞争的线程会不断自旋导致严重消耗CPU。因此轻量级锁适用于追求响应时间并且同步块执行速度非常快的场景。

**实现原理**

线程每次加锁，都会在线程的栈帧中分配一块锁记录的空间，锁记录里面包含了锁对象的Mark Word(displaced word)和指向锁对象的指针，而锁对象的对象头里面的ptr_to_lock_record是指向栈帧中锁记录的指针。

当线程第一次加锁时，锁记录就去记录完整的Mark Word和锁对象指针，当线程重入加锁时，同样会生成一个锁记录，但该锁记录里面的displaced word就为null，只记录了指向锁对象的指针，而锁对象里面的ptr_to_lock_record记录的是**线程栈顶**的锁记录，如下图所示：

![synchronized_lock_lnjsboa](.\images\synchronized_lock_lnjsboa.png)

当字节码解释器在执行monitorenter字节码轻度锁住一个对象时，就会在获取锁的线程的栈上显示或隐式分配一个lock record。其主要作用就是持有displaced word和锁对象的元数据，解释器可以使用lock record来检测非法的锁状态；隐式地充当重入机制的计数器。

轻量级锁加锁逻辑如下图所示：

![synchronized_lock_kasjbc](.\images\synchronized_lock_kasjbc.png)

**锁撤销**

在释放锁时，当前线程会使用`CAS`操作将`Displaced Mark Word`的内容复制回锁的`Mark Word`里面：

- 如果没有发生竞争，那么这个复制的操作就会成功。
- 如果有其他线程因为自旋多次导致轻量级锁升级成了重量级锁，那么`CAS`操作就会失败，此时就会释放锁并唤醒被阻塞的线程。

![synchronized_lock_lanoibf](.\images\synchronized_lock_lanoibf.png)



**锁升级**

- 处于偏向锁时，遇到两个及以上的线程竞争，此时从偏向锁升级为轻量级锁。

  如果两个线程都是活跃的，会发生竞争，此时偏向锁就会发生升级，也就是我们常常听到的锁膨胀。**偏向锁会膨胀成轻量级锁(lightweight locking)。**

  下面先简单地描述其升级的步骤：

  1. 在一个全局安全点（在这个时间点上没有字节码正在执行）停止拥有锁的线程。
  2. 拥有锁的线程在自己的栈桢中创建锁记录 LockRecord。
  3. 将锁对象的对象头中的MarkWord复制到线程的刚刚创建的锁记录中
  4. 将锁记录 LockRecord 中的Owner指针指向锁对象
  5. 将锁对象的对象头的 MarkWord 替换为指向锁记录的指针。

  其过程可以用如下图表示：

  ![synchronized_lock_noasb](.\images\synchronized_lock_noasb.jpeg)

  ![synchronized_lock_aohwd](.\images\synchronized_lock_aohwd.jpeg)

- 关闭偏向锁时，只要获取锁就会直接从无锁升级到轻量级锁。

- 在锁对象处于无锁状态的时候获取过一致性哈希（即调用过hashCode()方法），则之后获取锁就会直接进入轻量级锁状态，而不会进入偏向锁状态。

  > 主要是因为锁对象处于偏向锁状态时，对象头的`MarkWord`无法存储哈希值，只有到轻量级锁或者重量级锁才能存储哈希值，因此获取了一致性哈希后，为了保证哈希值只获取一次，因此需要一直存储着哈希值，也就不能用无法存储哈希值的偏向锁了，状态只能介于无锁、轻量级锁、重量级锁之间。



##### 7.2.1.4 重量级锁

**概述**

重量级锁依赖于每个对象内部都有的monitor锁来实现的，而monitor又依赖于操作系统的MutexLock(互斥锁)来实现，所以一般重量级锁也叫**互斥锁**。

由于需要在操作系统的内核态和用户态之间切换的，需要将线程阻塞挂起，切换线程的上下文，再恢复等操作，所以当synchronized升级成互斥锁，依赖monitor的时候，开销就比较大了，而这也是为什么说synchronized是一个很重的操作的原因了。

当然，升级成互斥锁之后，锁对象头的MarkWord内容也是会变化的，其锁标志位变成10，并且记录monitor的地址。

重量级锁状态下，线程竞争不适用自旋，不会消耗CPU，但是线程会进入阻塞，导致响应时间缓慢。因此重量级锁适用于追求吞吐量大且同步块执行速度较长的情景。

**实现原理**

- monitor机制

  **每个Java对象都有一个与之关联的Monitor**。这个Monitor在对象创建时自动创建，并且与对象的生命周期绑定。当对象被销毁时，其对应的Monitor也会被销毁。

  **Monitor是隐藏的**，我们无法直接操作Monitor，而是通过` synchronized`关键字或者`wait()`、`notify()`、`notifyAll()`等方法间接操作它。

  Monitor是synchronized实现互斥（注意专指实现互斥，因此专指实现重量级锁）的核心，当使用synchronized时，JVM会通过**Monitor（管程）**实现互斥。

  Monitor实现互斥的本质是依赖于底层操作系统的Mutex Lock实现，操作系统实现线程之间的切换需要从用户态转到内核态，成本非常高。

  Monitor其完整结构在JVM源码（如objectMonitor.hpp）中定义如下：

  ```c++
  class ObjectMonitor {
      void*     _header;        // 对象头（存储Mark Word）
      void*     _owner;         // 持有锁的线程指针
      volatile intptr_t  _recursions; // 重入次数
      ObjectWaiter* _EntryList; // 竞争锁的线程队列（阻塞态）
      ObjectWaiter* _WaitSet;   // 调用wait()后的线程队列（等待态）
      volatile int _WaitSetLock;// 保护WaitSet的锁
      // ... 其他字段省略
  };
  ```

  执行流程：

  - 线程通过monitorenter指令尝试获取Monitor所有权。

  - 若Owner为空，则当前线程成为Owner，recursions=1。
  - 若Owner是当前线程，recursions++（可重入性）。
  - 竞争失败则进入EntryList阻塞等待。

  关键队列说明：

  - EntryList
    - 当线程A持有锁时，线程B尝试获取锁失败，会被封装为ObjectWaiter节点加入EntryList，并进入BLOCKED状态。
    - 唤醒规则：锁释放时，JVM会从EntryList中选择线程（非公平模式下可能直接唤醒最新竞争者）。

  - WaitSet
    - 当线程调用wait()方法后，线程会释放锁并进入WaitSet，状态变为WAITING。
    - 唤醒条件：其他线程调用notify()/notifyAll()后，线程从WaitSet转移到EntryList重新竞争锁。

- 重量级锁实现

  在JVM中，重量级锁是基于进入和退出**Monitor对象**来实现的，无论是显式同步还是隐式同步。

  对于**方法级的同步**，它是隐式的，即**无需通过字节码指令来控制**，它实现在方法调用和返回操作中。JVM可以从方法常量池中的方法表结构中的**ACC_SYNCHRONIZED**访问标志来区分一个方法是否为同步方法。当方法调用时，调用指令会检查方法的ACC_SYNCHRONIZED访问标志是否被设置，如果设置了，执行线程将**先持有Monitor，然后执行方法**，最后在方法完成时释放Monitor。

  对于**代码块的同步**，它是利用**monitorenter**和**monitorexit**这两个字节码指令实现的。这些指令分别位于同步代码块的开始和结束位置。当JVM执行到monitorenter指令时，当前线程会尝试获取Monitor对象的所有权，如果获取成功，就会执行同步代码块，然后通过monitorexit指令释放Monitor对象。

**锁升级**

- 轻量级锁自选次数达上限

  当轻量级锁自旋达到一定次数时，就会从轻量级锁升级为重量级锁。

  JDK1.4引入自旋锁，但synchronized自旋锁是默认关闭的，并且最大允许自旋次数是固定的，默认为10次。

  JDK1.6中，将synchronized自旋锁默认开启，同时引入了自适应的自旋锁， 这种自旋锁不再固定最大允许自旋次数，JVM 会根据**历史的锁状态**、**CPU核心数**、**线程状态**等来判断是否继续自旋，自旋次数可能是 0（不自旋）、几次、甚至几十次。例如：

  - 如果前一次某个线程自旋成功获得了锁，下一次可能会自旋时间更长；
  - 如果长期得不到锁，JVM 会减少甚至禁用自旋。

  可以用`-XX:+UseSpinWaitHint`参数控制启用自旋等待优化（如发出 CPU PAUSE 指令），用`-XX:+UseAdaptiveSpin`参数控制开启自适应自旋策略。

- 使用了notify()/wait()这套等待唤醒机制方法

  由于wait()和notify()这套等待唤醒方法是基于monitor的，因此一旦使用了wait()和notify()这套等待唤醒方法，synchronized就会直接升级到重量级锁，放弃偏向锁或轻量级锁优化，此时线程间的同步会涉及到大量的内核切换造成性能下降。因此，若对性能要求高，请考虑使用 `LockSupport`、`ReentrantLock`、`Condition` 等并发工具，它们可以更细粒度地控制锁行为。



##### 7.2.1.5 锁消除

编译器在编译时检测到不可能存在竞争条件的代码块，并将其对应的锁消除掉，从而提高程序性能。



##### 7.2.1.6 锁粗化

将多个连续的细粒度锁合并成一个粗粒度锁，减少锁的获取和释放操作次数，提高执行效率。



#### 7.2.2 使用教程

synchronized 就是**可重入锁**，因此一个线程调用 synchronized 方法的同时，在其方法体内部调用该对象另一个 synchronized 方法是允许的。

> 从互斥锁的设计上来说，当一个线程试图操作一个由其他线程持有的对象锁的临界资源时，将会处于阻塞状态，但当一个线程再次请求自己持有对象锁的临界资源时，这种情况属于重入锁，请求将会成功。
>
> 代码示例：
>
> ```java
> public class AccountingSync implements Runnable{
>     static AccountingSync instance=new AccountingSync();
>     static int i=0;
>     static int j=0;
> 
>     @Override
>     public void run() {
>         for(int j=0;j<1000000;j++){
>             //this,当前实例对象锁
>             synchronized(this){
>                 i++;
>                 increase();//synchronized的可重入性
>             }
>         }
>     }
> 
>     public synchronized void increase(){
>         j++;
>     }
> 
>     public static void main(String[] args) throws InterruptedException {
>         Thread t1=new Thread(instance);
>         Thread t2=new Thread(instance);
>         t1.start();t2.start();
>         t1.join();t2.join();
>         System.out.println(i);
>     }
> }
> ```

synchronized 关键字最主要有以下 3 种应用方式：

- 同步方法，为当前对象（this）加锁，进入同步代码前要获得当前对象的锁；
- 同步静态方法，为当前类加锁（锁的是 Class 对象），进入同步代码前要获得当前类的锁；
- 同步代码块，指定加锁对象，对给定对象加锁，进入同步代码库前要获得给定对象的锁。



##### synchronized同步方法

**介绍**

对于 synchronized 同步静态方法，调用当前方法的线程获取被调用方法的对象的锁，保证同时刻下只有一个线程调用该对象的该方法。

**格式**

```java
public class Clazz{
	public synchronized void method() {
		...
    }
}
```

synchronized同步方法等价于如下代码块

```java
public class Clazz{
	public void method() {
		synchronized(this) {
			...
		}
    }
}
```



##### synchronized同步静态方法

**介绍**

对于 synchronized 同步静态方法，调用当前静态方法的线程获取方法所在类的 Class 对象的锁，因此任何其他线程在锁释放之前都无法调用该静态方法，保证了同时刻下只有一个线程调用该静态方法。

**格式**

```java
public class Clazz{
	public static synchronized void method() {
		...
    }
}
```

synchronized同步静态方法等价于如下代码块

```java
public class Clazz{
	public void method() {
		synchronized(Clazz.class) {
			...
		}
    }
}
```

由于静态成员变量不专属于任何一个对象，因此通过 Class 锁可以控制静态成员变量的并发操作。



##### synchronized同步代码块

**介绍**

某些情况下，编写的方法代码量比较多，存在一些比较耗时的操作，而需要同步的代码块只有一小部分，如果直接对整个方法进行同步，可能会得不偿失，此时可以使用同步代码块的方式对需要同步的代码进行包裹。

**格式**

```java
synchronized(object) {
    ...
}
```



#### 7.2.3 实战场景

##### synchronized同步方法

```java
public class AccountingSync implements Runnable {
    //共享资源(临界资源)
    static int i = 0;
    // synchronized 同步方法
    public synchronized void increase() {
        i ++;
    }
    @Override
    public void run() {
        for(int j=0;j<1000000;j++){
            increase();
        }
    }
    public static void main(String args[]) throws InterruptedException {
        AccountingSync instance = new AccountingSync();
        Thread t1 = new Thread(instance);
        Thread t2 = new Thread(instance);
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println("static, i output:" + i);
    }
}
```

输出结果：

```
static, i output:2000000
```

如果在方法 `increase()` 前不加 synchronized，因为 i++ 不具备原子性，所以最终结果会小于 2000000。

注意：一个对象只有一把锁，当一个线程获取了该对象的锁之后，其他线程无法获取该对象的锁，所以无法访问该对象的其他 synchronized 方法，但是其他线程还是可以访问该对象的其他非 synchronized 方法。



##### synchronized同步静态方法

```java
public class AccountingSyncClass implements Runnable {
    static int i = 0;
    /**
     * 同步静态方法,锁是当前class对象，也就是
     * AccountingSyncClass类对应的class对象
     */
    public static synchronized void increase() {
        i++;
    }
    // 非静态,访问时锁不一样不会发生互斥
    public synchronized void increase4Obj() {
        i++;
    }
    @Override
    public void run() {
        for(int j=0;j<1000000;j++){
            increase();
        }
    }
    public static void main(String[] args) throws InterruptedException {
        //new新实例
        Thread t1=new Thread(new AccountingSyncClass());
        //new新实例
        Thread t2=new Thread(new AccountingSyncClass());
        //启动线程
        t1.start();t2.start();
        t1.join();t2.join();
        System.out.println(i);
    }
}
```

输出结果:

```
2000000
```

由于 synchronized 关键字同步的是静态的 increase 方法，与同步实例方法不同的是，其锁对象是当前类的 Class 对象。



##### synchronized代码块

```java
public class AccountingSync2 implements Runnable {
    static AccountingSync2 instance = new AccountingSync2(); // 饿汉单例模式

    static int i=0;

    @Override
    public void run() {
        //省略其他耗时操作....
        //使用同步代码块对变量i进行同步操作,锁对象为instance
        synchronized(instance){
            for(int j=0;j<1000000;j++){
                i++;
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Thread t1=new Thread(instance);
        Thread t2=new Thread(instance);
        t1.start();t2.start();
        t1.join();t2.join();
        System.out.println(i);
    }
}
```

输出结果：

```
2000000
```

我们将 synchronized 作用于一个给定的实例对象 instance，即当前实例对象就是锁的对象，当线程进入 synchronized 包裹的代码块时就会要求当前线程持有 instance 实例对象的锁，如果当前有其他线程正持有该对象锁，那么新的线程就必须等待，这样就保证了每次只有一个线程执行 `i++` 操作。



### 7.3 LockSupport





### 7.4 ReentrantLock

#### 7.4.1 概述

`ReentrantLock` 是 Java 并发包 `java.util.concurrent.locks` 提供的一个可重入独占锁（reentrant exclusive lock）。与传统的 `synchronized` 关键字相比，`ReentrantLock` 提供了更灵活的锁控制能力，例如：

- **可中断锁获取**（`lockInterruptibly()`）
- **非阻塞锁尝试**（`tryLock()`）
- **定时获取锁**（`tryLock(timeout, unit)`）
- **公平锁选择**（构造函数中可选择公平/非公平）
- **可绑定多个条件队列**（通过 `newCondition()` 支持多个 `await()/signal()` 通信队列）

`ReentrantLock` 支持线程**重入**，即同一线程可以多次获得同一把锁，每获得一次锁计数器加一，释放时计数器减一，直到归零才真正释放锁。



#### 7.4.2 原理

##### 7.4.2.1 如何基于AQS实现类 

`ReentrantLock` 的锁实现是基于 `AQS` 的实现类的，因此需要先了解AQS思想和AQS接口：[跳转AQS介绍](###6.4 AQS)。

`ReentrantLock` 和 `AQS` 的体系架构图如下图：

![AQS_omvdjsn](E:\各种资料\Java开发笔记\我的笔记\images\AQS_omvdjsn.png)

从架构图可以看到，`ReentrantLock` 有内部类 `Sync` 实现了 `AQS` 接口，但是 `ReentrantLock` 的实现并不直接依赖内部类 `Sync` ，而是在 `ReentrantLock` **内部定义了一个 `Sync` 类型属性：**

```java
public class ReentrantLock implements Lock, java.io.Serializable {
    private final Sync sync;
}
```

然后又定义了 `NonfairSync` 和 `FairSync` 这两个内部类实现 `Sync` 赋值给该 `Sync` 属性，分别表示非公平锁和公平锁，并且 `ReentrantLock` 所有方法都是通过 `Sync` 的实现类实现的，比如 `lock()` 方法源码如下：

```java
public class ReentrantLock implements Lock, java.io.Serializable {
    public void lock() {
        sync.lock();
    }
}
```

而 `Sync` 这个内部类本身仅仅提供抽象方法并不实现 `lock()` 方法，而是交给其实现类 `FairSync` 和 `NonfairSync` 来实现：

> 该 `lock()` 方法并非从AQS接口继承而来，而是 `Sync` 类自己定义的抽象方法。

```java
public class ReentrantLock implements Lock, java.io.Serializable {
	abstract static class Sync extends AbstractQueuedSynchronizer {
        abstract void lock();
    }
}
```

通过构造器使得在创建 `ReentrantLock` 对象的时候就会决定该 `Sync` 类型属性引用的是公平锁还是非公平锁对象，源码如下：

```java
public class ReentrantLock implements Lock, java.io.Serializable {
	public ReentrantLock() {
        sync = new NonfairSync();
    }

    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
}
```

由 `ReentrantLock` 的构造器可见， `ReentrantLock` 对象默认采用非公平锁来实现，可以通过调用带参构造器来决定使用公平锁或非公平锁。

##### 7.4.2.2 ASQ的实现类介绍

上面已经介绍到 `ReentrantLock` 的所有方法是调用 `Sync` 的 `lock()` 抽象方法，而其中的 `lock()` 方法由 `Sync` 的实现类 `FairSync` 和 `NonfairLock` 来实现。

`Sync` 类实现如下：

```java
public class ReentrantLock implements Lock, java.io.Serializable {
	abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = -5179523762034025860L;

        abstract void lock();

        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }

        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }

        protected final boolean isHeldExclusively() {
            return getExclusiveOwnerThread() == Thread.currentThread();
        }

        final ConditionObject newCondition() {
            return new ConditionObject();
        }
        
        final Thread getOwner() {
            return getState() == 0 ? null : getExclusiveOwnerThread();
        }

        final int getHoldCount() {
            return isHeldExclusively() ? getState() : 0;
        }

        final boolean isLocked() {
            return getState() != 0;
        }

        private void readObject(java.io.ObjectInputStream s)
            throws java.io.IOException, ClassNotFoundException {
            s.defaultReadObject();
            setState(0); // reset to unlocked state
        }
    }
}
```

该类定义了**获取锁**、**非公平地尝试占有锁**、**尝试释放锁**、**判断当前线程是否占有锁**、**获取Condition类**、**获取当前占有锁的类**、获取当前state的值、判断当前是否上锁等方法。

`NonfairLock` 类实现如下：

```java
public class ReentrantLock implements Lock, java.io.Serializable {
	static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }
}
```

该类实现了 `Sync` 类的 `lock()` 抽象方法，同时定义了 `tryAcquire` 这个尝试获取锁的方法，由于是非公平锁，因此直接调用 `Sync` 类提供的非公平地尝试获取锁的方法。

`FairLock` 类实现如下：

```java
public class ReentrantLock implements Lock, java.io.Serializable {
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

        final void lock() {
            acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }
}
```

该类实现了 `Sync` 类的 `lock()` 抽象方法，同时定义了 `tryAcquire` 这个尝试获取锁的方法，并且是以公平锁的原理来实现。

##### 7.4.2.3 获取锁

获取锁的源码结构如下：

![ReentrantLock_opqnd](E:\各种资料\Java开发笔记\我的笔记\images\ReentrantLock_opqnd.png)

这里不对 `AQS` 的源码内容占用过多篇章，可跳转AQS查看AQS相关方法及其实现细节：[跳转到AQS介绍](###6.4 AQS)。

##### 7.4.2.4 释放锁

释放锁的源码结构如下：

![ReentrantLock_onxsan](E:\各种资料\Java开发笔记\我的笔记\images\ReentrantLock_onxsan.png)

这里不对 `AQS` 的源码内容占用过多篇章，可跳转AQS查看AQS相关方法及其实现细节：[跳转到AQS介绍](###6.4 AQS)。



### 7.5 ReentrantReadWriteLock





# 五、JVM

## 1. JVM内存结构

了解Java内存模型之前，需要先了解JVM内存结构（注意二者区别），JVM内存结构如下图所示，其中方法区和堆是所有线程共享，虚拟机栈、本地方法栈、程序计数器为线程私有。

![jmm_dsadvnk](.\images\jmm_dsadvnk.png)



### 1.1 程序计数器

​	程序计数器用来存放当前线程接下来将要执行的字节码指令、分支、循环、跳转、异常处理等信息。在任何时候CPU只执行其中一个线程中的指令，为了能够在多线程并发执行发生进程切换时能够回到线程正确的执行位置，每个线程都需要有一个程序计数器，并且各程序计数器之间互不影响，因此程序计数器为线程私有的。



### 1.2 虚拟机栈

​	虚拟机栈是线程私有，它的生命周期与线程相同，是在JVM运行时所创建，在线程中，方法在执行的时候都会创建一个名为栈帧的数据结构，主要用于存放局部变量表、操作栈、动态链接、方法出口等信息，如下图所示，方法的调用对应着栈帧在虚拟机栈中的压栈和弹栈过程。

![jvm_csanio](.\images\jvm_csanio.jpg)

#### 1.2.1 局部变量表

- **作用**：

  - 局部变量表用于存储方法参数和方法内部定义的局部变量。它是一块线性分配的内存空间，每个变量占用一个或多个槽位（Slot）。对于基本数据类型（如 `int`、`float`、`boolean` 等）和对象引用（`reference` 类型），每个变量占用一个槽位；而 `long` 和 `double` 类型的变量占用两个槽位。
  - 在方法执行过程中，局部变量表中的变量可以通过索引访问。例如，`iload_0` 指令表示将局部变量表中索引为 0 的 `int` 类型变量加载到操作数栈顶。
  - 局部变量表的大小在方法定义时就已经确定，它存储在方法的 `Code` 属性中。编译器会根据方法的参数和局部变量的数量来计算局部变量表的大小。

- **示例**：

  ```java
  public void exampleMethod(int a, int b) {
      int c = a + b;
  }
  ```

  在这个方法中，局部变量表会存储参数 `a` 和 `b`，以及局部变量 `c`。



#### 1.2.2 操作数栈

- **作用**：

  - 操作数栈是一个后进先出（LIFO）的栈结构，用于存储方法执行过程中的中间计算结果。它在方法执行过程中起到临时存储的作用，类似于 CPU 中的寄存器。
  - 操作数栈的大小在方法定义时也已经确定，存储在方法的 `Code` 属性中。操作数栈中的元素可以是任意 Java 数据类型的值，包括 `int`、`float`、`long`、`double`、`reference` 和 `returnAddress` 等。
  - JVM 的指令集大多是基于操作数栈的。例如，`iadd` 指令会从操作数栈中弹出两个 `int` 类型的值，将它们相加后将结果压入操作数栈。

- **示例**： 假设执行以下代码：

  ```java
  int a = 10;
  int b = 20;
  int c = a + b;
  ```

  对应的字节码操作可能如下：

  - 将 `a` 的值压入操作数栈。
  - 将 `b` 的值压入操作数栈。
  - 执行 `iadd` 指令，将栈顶的两个值相加，结果压入操作数栈。
  - 将结果存储到局部变量表中。



#### 1.2.3 动态链接

- **作用**：

  - 动态链接用于支持 Java 的动态绑定特性。在方法执行过程中，JVM 需要将方法调用中的符号引用（如类名、方法名和方法描述符）解析为具体的内存地址。动态链接确保了方法调用的正确性和灵活性。
  - 每个栈帧中都包含一个指向运行时常量池的引用，运行时常量池中存储了方法调用所需的符号引用信息。当方法被调用时，JVM 通过动态链接将这些符号引用解析为实际的方法地址。
  - 动态链接是 Java 多态实现的基础。例如，当调用一个对象的虚方法时，JVM 会通过动态链接找到该对象实际所属类的对应方法实现。

- **示例**：

  ```java
  class Parent {
      public void show() {
          System.out.println("Parent show");
      }
  }
  
  class Child extends Parent {
      @Override
      public void show() {
          System.out.println("Child show");
      }
  }
  
  public class Test {
      public static void main(String[] args) {
          Parent obj = new Child();
          obj.show(); // 输出 "Child show"
      }
  }
  ```

  在这个例子中，`obj.show()` 的调用通过动态链接找到 `Child` 类的 `show` 方法实现。

  

#### 1.2.4 方法出口

- **作用**：

  - 方法出口用于处理方法执行完成后的返回操作。当方法执行完毕后，JVM 需要将方法的返回值（如果有）传递给调用者，并恢复调用者的上下文环境（如恢复调用者的操作数栈和局部变量表）。
  - 方法出口还负责处理异常情况。如果方法执行过程中抛出了异常，JVM 会通过方法出口将异常信息传递给调用者，并根据异常处理机制决定是否继续执行调用者的代码。
  - 方法出口是方法调用生命周期的终点，它确保了方法调用的正确结束，并恢复线程的执行状态。

- **示例**：

  ```java
  public int add(int a, int b) {
      return a + b; // 方法出口将结果返回给调用者
  }
  ```

  在这个例子中，`add` 方法执行完成后，通过方法出口将结果返回给调用者。



### 1.3 本地方法栈

​	本地方法栈是线程私有的，Java中提供了调用本地方法的接口（Java Native Interface），也就是C/C++程序，在线程的执行过程中，经常会碰到调用JNI方法的情况，JVM为本地方法所划分的内存区域便是本地方法栈。



### 1.4 堆内存

​	堆内存是JVM中最大的一块内存区域，被所有的线程所共享，Java在运行期间创建的所有对象几乎都存放在堆内存中，并且堆内存区是垃圾回收器重点照顾的区域，因此有些时候堆内存也被称为“GC堆”。

​	堆内存一般会被细分为新生代和老年代，更细致的划分为Eden区、From Survivor区和To Survivor区，如下图所示。

![jvm_snckas](.\images\jvm_snckas.webp)



### 1.5 方法区

​	方法区又叫持久代，被所有线程共享，主要用于存储已经被虚拟机加载的类信息、常量、静态变量、即时编译器（JIT）编译后的代码等数据。方法区在JVM启动时被创建，大小固定，因此容易出现方法区内存分配不足而内存溢出。

​	Java虚拟机规范将方法区划分为堆内存的一个逻辑分区。

​	在HotSpot JVM中，方法区被细分为持久代和代码缓存区，代码缓存区主要用于存储编译后的本地代码以及JIT编译器生成的代码。



### 1.6 元空间

​	元空间（Meta Space）自JDK1.8版本之后取代了方法区，元空间同样是堆内存的一部分，相比较于方法区，元空间存在于本地内存而不是虚拟机内部，并且元空间的大小是动态的，因此不容易出现内存溢出的情况。



### 1.7 对象内存结构

HotSpot 虚拟机中，对象在内存中存储的布局可以分为三块区域：对象头（Header）、实例数据（Instance Data）和对齐填充（Padding）。

![object_memory_xnldlk](.\images\object_memory_xnldlk.png)

#### 1.7.1 对象头

HotSpot虚拟机的对象头分为两部分信息，第一部分用于存储对象自身运行时数据，如哈希码、GC分代年龄等，这部分数据的长度在32位和64位的虚拟机中分别为32位和64位。官方称为Mark Word。另一部分用于存储指向对象类型数据的指针，如果是数组对象的话，还会有一个额外的部分存储数组长度。

![object_memory_papfq](.\images\object_memory_papfq.png)

先简单介绍下对象头的形式，JVM中对象头的方式有以下两种（以32位JVM为例）：

普通对象：

![object_memory_apqnp](.\images\object_memory_apqnp.png)

数组对象：

![object_memory_nconajp](.\images\object_memory_nconajp.png)

##### 1.7.1.1 Mark Word
这部分主要用来存储对象自身的运行时数据，如hashcode、gc分代年龄等。mark word的位长度为JVM的一个Word大小，也就是说32位JVM的Mark word为32位，64位JVM为64位。
为了让一个字大小存储更多的信息，JVM将字的最低两个位设置为标记位，不同标记位下的Mark Word示意如下：

![object_memory_zmpbng](.\images\object_memory_zmpbng.png)

其中各部分的含义如下：

- lock：2位的锁状态标记位，由于希望用尽可能少的二进制位表示尽可能多的信息，所以设置了lock标记。该标记的值不同，整个mark word表示的含义不同。

![object_memory_mvjso](.\images\object_memory_mvjso.png)

- bias_lock：对象是否启动偏向锁标记，只占1个二进制位。为1时表示对象启动偏向锁，为0时表示对象没有偏向锁。

- age：4位的Java对象年龄。在GC中，如果对象在Survivor区复制一次，年龄增加1。当对象达到设定的阈值时，将会晋升到老年代。默认情况下，并行GC的年龄阈值为15，并发GC的年龄阈值为6。由于age只有4位，所以最大值为15，这就是-XX:MaxTenuringThreshold选项最大值为15的原因。
- identity_hashcode：25位的对象标识Hash码，采用延迟加载技术。调用方法System.identityHashCode()计算，并会将结果写到该对象头中。当对象被锁定时，该值会移动到管程Monitor中。
- thread：持有偏向锁的线程ID。
- epoch：偏向时间戳。
- ptr_to_lock_record：指向栈中锁记录的指针。
- ptr_to_heavyweight_monitor：指向monitor对象（也称为管程或监视器锁）的起始地址，每个对象都存在着一个monitor与之关联，对象与其monitor之间的关系有存在多种实现方式，如monitor对象可以与对象一起创建销毁或当前线程试图获取对象锁时自动生，但当一个monitor被某个线程持有后，它便处于锁定状态。



##### 1.7.1.2 class pointer
这一部分用于存储对象的类型指针，该指针指向它的类元数据，JVM通过这个指针确定对象是哪个类的实例。该指针的位长度为JVM的一个字大小，即32位的JVM为32位，64位的JVM为64位。

如果应用的对象过多，使用64位的指针将浪费大量内存，统计而言，64位的JVM将会比32位的JVM多耗费50%的内存。为了节约内存可以使用选项+UseCompressedOops开启指针压缩，其中，oop即ordinary object pointer普通对象指针。开启该选项后，下列指针将压缩至32位：

- 每个Class的属性指针（即静态变量）
- 每个对象的属性指针（即对象变量）
- 普通对象数组的每个元素指针

当然，也不是所有的指针都会压缩，一些特殊类型的指针JVM不会优化，比如指向PermGen的Class对象指针(JDK8中指向元空间的Class对象指针)、本地变量、堆栈元素、入参、返回值和NULL指针等。

 

##### 1.7.1.3 array length

如果对象是一个数组，那么对象头还需要有额外的空间用于存储数组的长度，这部分数据的长度也随着JVM架构的不同而不同：32位的JVM上，长度为32位；64位JVM则为64位。64位JVM如果开启+UseCompressedOops选项，该区域长度也将由64位压缩至32位。



#### 1.7.2 实例数据

存放类的属性数据信息，包括父类的属性信息，如果是数组的实例部分还包括数组的长度，这部分内存按4字节对齐。



#### 1.7.3 填充数据

由于虚拟机要求对象起始地址必须是8字节的整数倍。填充数据不是必须存在的，仅仅是为了字节对齐。



# 六、Java重要类

## 1. 并发

### 1.1 Unsafe

#### 1.1.1 概述

Unsafe这个类中有许多的native方法，通过字段偏移量（类似于C的指针），提供了Java语言与底层系统进行交互的接口，通过Unsafe可以直接操作底层系统，它具有直接内存管理、线程阻塞&唤醒的支持、CAS操作的支持、直接操作类、对象、变量等强大的功能。

#### 1.1.2 原理



#### 1.1.3 使用



### 1.2 AtomicStampedReference



## 2. 日志

### 2.1 Logger



### 2.2 LoggerFactory



## 3. 配置信息处理

### 3.1 CommandLine



### 3.2 Option



### 3.3 Properties



































