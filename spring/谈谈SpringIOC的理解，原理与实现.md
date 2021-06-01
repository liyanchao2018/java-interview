突出一些技术名词（核心概念，接口，类，关键方法）

一个问题能占用面试官多少时间？问的越多可能露馅越多

当面试管问道一个你熟悉的点的时候，一定要尽量拖时间。

### 1.谈谈Spring IOC的理解，原理与实现？

**总：**

- 控制反转：理论思想，原来的对象是由使用者来进行控制的，有了spring后，可以把整个对象交给spring帮我们进行管理
  - DI：依赖注入，把对应的属性的值注入到具体的对象中。@Autowired @populateDean 完成属性值的注入



- spring容器：存储对象，使用map结构来存储，在spring中一般使用三级缓存，singletonObject存放完整的bean对象，整个bean的生命周期，从创建到使用到销毁的过程，全部都是由容器来管理（bean的生命周期）。

**分：**

1.一般聊ioc容器的时候，要涉及到容器的创建过程（beanFactory.DefaultListableBeanFactory）。向bean工厂中设置一些参数（BeanPostProcessor，Aware接口的子类）等等属性。

2.加载解析bean对象，准备要创建的bean对象的定义对象beanDefinition，（xml或者注解的解析过程）

3.beanFactoryPostProcessor的处理，此处是扩展点，PlaceHolderConfigurSupport处理占位符，ConfigurationClassPostProcessor

4.BeanPostProcessor的注册功能，方便后续对bean对象完成具体的扩展功能。

5.通过反射的方式将beanDefinition对象实例化成具体的bean对象

6.bean对象的初始化过程（填充属性，调用aware子类的方法，调用BeanPostProcessor前置处理方法，调用init-method方法，调用BeanPostProecessor的后置处理方法）

7.生成完整的bean对象，通过getBean方法可以直接获取

8.销毁过程

对面试官说：这是我对IOC的整体理解，包含了一些详细的处理过程，您看下有什么问题，可以指点我一下（允许你把整个流程说完）











### 2.谈一下spring IOC的底层实现



### 3.描述一下bean的生命周期



### 4.Spring 是如何解决循环依赖的问题的



### 5.Bean Factory与FactoryBean有什么区别



### 6.Spring中用到的设计模式



