# [详解JAVA对象实例化过程](https://zhuanlan.zhihu.com/p/187016666)



### 1. 对象的实例化过程

- 对象的实例化过程是分成两部分：类的加载初始化，对象的初始化。
- 要创建类的对象实例，需要先加载并初始化该类，main方法所在的类需要先加载和初始化。
- 类初始化就是执行<clinit>方法，对象实例化是执行<init>方法
- 一个子类要初始化需要先初始化父类



#### 1.1 类的加载过程

##### 1.虚拟机的类加载机制[classloader]

​		classloader顾名思义，即是类加载。虚拟机把描述类的数据从class字节码文件加载到内存，并对数据进行检验、转换解析和初始化，最终形成可以被虚拟机直接使用的Java类型，这就是虚拟机的类加载机制。

​		了解java的类加载机制，可以快速解决运行时的各种加载问题并快速定位其背后的本质原因，也是解决疑难杂症的利器。因此学好类加载原理也至关重要。

##### 2. 类加载初始化过程

![img](D:\学习\面试资料\java-interview\spring\image\20210601001.jpg)

- 类的加载机制:如果没有相应类的class，则加载class到方法区。对应着加载->验证->准备->解析-->初始化阶段

- - 加载：载入class对象，不一定是从class文件获取，可以是jar包，或者动态生成的class
  - 验证：校验class字节流是否符合当前jvm规范
  - 准备：为 **类变量** 分配内存并设置变量的初始值( **默认值** )。如果是final修饰的对象则是赋值声明值
  - 解析：将常量池的符号引用替换为直接引用
  - 初始化：执行类构造器<client>( **注意不是对象构造器** )，为 **类变量** 赋值，执行静态代码块。jvm会保证子类的<client>执行之前，父类的<client>先执行完毕

- 其中验证、准备、解析3个部分称为 连接

- <clinit>方法由 **静态变量赋值代码和静态代码块** 组成；先执行类静态变量显示赋值代码，再到静态代码块代码







####  1.2 对象实例化过程

- 对象实例化过程其实就是执行类构造函数 对应在字节码文件中的<init>()方法(称之为实例构造器)；<init>()方法由 **非静态变量、非静态代码块以及对应的构造器组成**

- - <init>()方法可以重载多个，类有几个构造器就有几个<init>()方法
  - <init>()方法中的代码执行顺序为：父类变量初始化，父类代码块，父类构造器，子类变量初始化，子类代码块，子类构造器。

- 静态变量，静态代码块，普通变量，普通代码块，构造器的执行顺序

![img](D:\学习\面试资料\java-interview\spring\image\20210601002.jpg)

- 具有父类的子类的实例化顺序如下

  ![img](D:\学习\面试资料\java-interview\spring\image\20210601003.jpg)





### 2. 类加载器和双亲委派规则

#### 2.1 类加载器

- - 通过一个类的全限定名来获取 **描述此类的二进制字节流** ，实现这个动作的代码模块称为类加载器
  - 任意一个类都需要其加载器和类本身来确定类在JVM的唯一性；每个类加载器都有自己的类名称空间，同一个类class由不同的加载器加载，则被JVM判断为不同的类



![img](D:\学习\面试资料\java-interview\spring\image\20210601004.jpg)

​		整个java虚拟机的类加载层次关系如上图所示，启动类加载器(Bootstrap Classloader)负责将<JAVA_HOME>/lib目录下并且被虚拟机识别的类库加载到虚拟机内存中。我们常用基础库，例如java.util.**，java.io.**，java.lang.**等等都是由根加载器加载。



#### 2.2 双亲委派模型

- - 启动类加载器有C++代码实现，是虚拟机的一部分。负责加载lib下的类库
  - 其他的类加载器有java语言实现，独立于JVM，并且继承ClassLoader
  - 扩展类加载器(Extention Classloader)负责加载JVM扩展类，比如swing系列、内置的js引擎、xml解析器等，这些类库以javax开头，它们的jar包位于<JAVA_HOME>/lib/ext目录中。
  - 应用程序加载器(Application Classloader)也叫系统类加载器，它负责加载用户路径(ClassPath)上所指定的类库。我们自己编写的代码以及使用的第三方的jar包都是由它来加载的自定义加载器(Custom Classloader)通常是我们为了某些特殊目的实现的自定义加载器，后面我们得会详细介绍到它的作用以及使用场景。
  - 不同的类加载器加载同一个class文件会导致出现两个类。而java给出解决方法是下层的加载器加委托上级的加载器去加载类，如果父类无法加载(在自己负责的目录找不到对应的类)，而交还下层类加载器去加载。如下图



![img](D:\学习\面试资料\java-interview\spring\image\20210601005.jpg)

### 3.进阶--自定义ClassLoader

假如我们的类不在classpath下，而我们又想读取一个自定义的目录下的class，如果做呢？

#### 读取自定义目录的类

示例读取c:/test/com/test.jdk/Key.class这个类。

```java
package com.test.jdk;

public class Key {
    private String key = "111111";
}
```

#### 自定义ClassLoader

```java
import org.apache.commons.io.IOUtils;

import java.io.FileInputStream;
import java.io.IOException;
import java.io.InputStream;

public class LocalClassLoader extends ClassLoader {

    private String path = "c:/test/";

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        Class<?> cls = findLoadedClass(name);
        if (cls != null) {
            return cls;
        }

        if (!name.endsWith(".Key")) {
            return super.loadClass(name);
        }

        try {
            InputStream is = new FileInputStream(path + name.replace(".", "/") + ".class");
            byte[] bytes = IOUtils.toByteArray(is);
            return defineClass(name, bytes, 0, bytes.length);
        } catch (IOException e) {
            e.printStackTrace();
        }

        return super.loadClass(name);
    }
}
```

#### 开始读取类

```java
public static void main(String[] args) {
    try {
        LocalClassLoader lcl = new LocalClassLoader();
        Class<?> cls = lcl.loadClass("com.test.jdk.Key");
        Field field = FieldUtils.getField(cls, "key", true);
        Object value = field.get(cls.newInstance());
        System.out.println(value);
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

自定义类加载器正常加载到类，程序最后输出：111111

#### URLClassLoader

上面自定义一个类加载器来读取自定义的目录，其实可以直接使用URLClassLoader就能读取，它已经实现了路径下类的读取逻辑。

```java
public static void main(String[] args) {
    try {
        URLClassLoader ucl = new URLClassLoader(new URL[]{new URL("c:/test/")});
        Class<?> cls = ucl.loadClass("com.test.jdk.Key");
        Field field = FieldUtils.getField(cls, "key", true);
        Object value = field.get(cls.newInstance());
        System.out.println(value);
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

