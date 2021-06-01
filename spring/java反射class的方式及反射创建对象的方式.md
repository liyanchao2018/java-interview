# java反射class的三种方式，反射创建对象的两种方式



反射中，欲获取一个类或者调用某个类的方法，首先要获取到该类的 Class 对象。

### 1、获取Class对象
在 Java API 中，提供了获取 Class 类对象的三种方法：

#### 1.1 Class.forName(类的完全限定名)

第一种，使用 Class.forName 静态方法。

前提：已明确类的全路径名。



#### 1.2 类名.class()

第二种，使用 .class 方法。

说明：仅适合在编译前就已经明确要操作的 Class



#### 1.3 对象.getClass()

第三种，使用类对象的 getClass() 方法。

适合有对象示例的情况下

代码示例：



~~~java
package com.reflection;
 
/**
 * Created by Liuxd on 2018-08-15.
 */
public class User {
    private String name;
    private Integer age;
 
    public User() {
    }
 
    public User(String name, Integer age) {
        this.name = name;
        this.age = age;
    }
 
    public String getName() {
        return name;
    }
 
    public void setName(String name) {
        this.name = name;
    }
 
    public Integer getAge() {
        return age;
    }
 
    public void setAge(Integer age) {
        this.age = age;
    }
 
    @Override
    public String toString() {
        return "User{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
~~~



~~~java
package com.reflection;
 
/**
 * Created by Liuxd on 2018-08-15.
 */
public class TestReflection {
 
    public static void main(String[] args) {
//      第一、通过Class.forName方式
        Class clazz1 = null;
        try {
            clazz1 = Class.forName("com.reflection.User");
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
 
//      第二、通过对象实例方法获取对象
        Class clazz2 = User.class;
 
//      第三、通过Object类的getClass方法
        User user = new User();
        Class clazz3 = user.getClass();
 
        System.out.println(clazz1);
        System.out.println(clazz2);
        System.out.println(clazz3);
    }
}
~~~



### 2、获取对象实例

共两种方法：

2.1、直接用字节码文件获取对应实例

~~~java
// 调用无参构造器 ，若是没有，则会报异常

Object o = clazz.newInstance();　　
~~~

2.2、有带参数的构造函数的类，先获取到其构造对象，再通过该构造方法类获取实例：

~~~java
/ /获取构造函数类的对象

Constroctor constroctor = clazz.getConstructor(String.class,Integer.class); /

// 使用构造器对象的newInstance方法初始化对象

Object obj = constroctor.newInstance("龙哥", 29); 
~~~

代码示例：

~~~java
package com.reflection;
 
import java.lang.reflect.Constructor;
 
/**
 * Created by Liuxd on 2018-08-15.
 */
public class TestReflection {
 
    public static void main(String[] args) {
//      第一、通过Class.forName方式
        Class clazz1 = null;
        try {
            clazz1 = Class.forName("com.reflection.User");
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
 
//      第二、通过对象实例方法获取对象
        Class clazz2 = User.class;
 
//      第三、通过Object类的getClass方法
        User user = new User();
        Class clazz3 = user.getClass();
 
        System.out.println(clazz1);
        System.out.println(clazz2);
        System.out.println(clazz3);
 
        User user1 = null;
        try {
             user1 =(User) clazz1.newInstance();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
        user1.setName("终结者");
        user1.setAge(1500);
        System.out.println("user1:"+user1.toString());
 
 
        User user2 = null;
        try {
            // 获取构造函数
            Constructor constroctor = clazz2.getConstructor(String.class,Integer.class);
            // 通过构造器对象的newInstance方法进行对象的初始化
            user2 = (User) constroctor.newInstance("龙哥",29);
        } catch (Exception e) {
            e.printStackTrace();
        }
        System.out.println("user2:"+user2.toString());
 
    }
}
~~~

