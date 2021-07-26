# Spring源码学习记录





## 1.BeanFactory是spring容器的根接口，也是容器的入口

## 2.Spring的三个接口名字：

beanFactory、AbstractAutowiredCapableBeanFactory、DefaultListableBeanFactory，查看这三个接口与XmlClassPathApplicationContext的继承实现关系

## 3.BeanFactoryPostProcrssor ：

增强器/后置处理器，动态修改BeanDeifinition中bean的定义信息。                
例如，配置文件中${jdbc.url}的值替换，应用到了子类 PlaceHolderConfigurerSupport来实现。

## 4.BeanPostProcessor：

增强器/后置处理器，修改/增强bean信息。
AbstractAutoProxyCreator 是 BeanPostProcessor的子类，抽象自动代理创建器，是AOP在Spring中的应用，通过切面，自动创建代理实现类。

##  5.Aware接口：

如果某个类A需要获取spring中的某些对象，比如，applicationContext,environment等，则需要，A implements ApplicationContextAware接口，重写对应setApplicationContext方法，进行set注入。
aware接口的作用：
当Spring容器创建的bean对象，在进行具体操作时，如果需要容器的某个对象，只需要实现这个对象的Aware接口，来满足当前需要。(如A implements ApplicationContextAware，重写对应setApplicationContext方法，进行set注入。)

## 6.在Spring容器中，不同的阶段处理不同的工作，需要怎么做

观察者模式：监听器，监听事件，多播器（广播器）
AbstractApplicationContext中的refresh方法中，有写到 注册事件广播器[initApplicationEventMulticaster()]和注册事件监听器[register listener()]来完成观察者模式

## 7.BeanFactory和FactoryBean的区别：

1.BeanFactory和FactoryBean都是用来创建bean的
2.当使用BeanFactory的时候，必须遵循完整的创建过程，这个过程由Spring来控制的。
3.使用FactoryBean时，只需要调用getObject方法就可以获取具体的对象，整个对象的创建过程由用户自己控制，更加灵活。

## 8.获取class的三种方式：

1. Class class  =  Class.forename(beanName);
2. Class class  = MyClass.class ;
3. A a = New A( );
Class class  = a. get class();
9.根据Class实例化bean对象
Constructor ctor = class.getConstructor();
Object OBJ = ctor.newInstance();
或者
第二种:
Test test = Class.forName(Test).newInstance();

## 10.学懂Spring源码，入口先读懂Abstract ApplicationContext.refresh()方法

，方法里面有10多个方法，读懂了也就理解学会了Spring。



## 11.Spring的三级缓存及解决循环依赖问题

> 作者：敲代码的小姐姐
> 链接：https://juejin.cn/post/6967240428496093215

### 循环依赖

  BeanFactory作为bean工厂管理我们的单例bean，那么肯定需要有个缓存来存储这些单例bean，在Spring中就是一个Map结构的缓存，key为beanName，value为bean。在获取一个bean的时候，先从缓存中获取，如果没有获取到，那么触发对象的实例化和初始化操作，完成之后再将对象放入缓存中，这样就能实现一个简单的bean工厂。

  由于Spring提供了依赖注入的功能，支持将一个对象自动注入到另一个对象的属性中，这就可能出现循环依赖（区别于dependsOn）的问题：A对象在创建的时候依赖于B对象，而B对象在创建的时候依赖于A对象。

<img src="D:\学习\面试资料\java-interview\spring\image\20210618001.image" alt="image.png" style="zoom:50%;" />

可以看到，由于A依赖B，所以创建A的时候触发了创建B，而B又依赖A，又会触发获取A，但是此时A正在创建中，还不在缓存中，就引发了问题。

### 多级缓存

三级缓存在spring中是如何体现的呢？我们来到DefaultSingletonBeanRegistry中：

> ~~~java
> //一级缓存，也就是单例池，存储最终对象
> private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
> //二级缓存，存储早期对象
> private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);
> //三级缓存，存储的是一个函数接口，
> private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
> ~~~
>
> ​		其实所谓三级缓存，在源码中就是3个Map，一级缓存用于存储最终单例对象，二级缓存用于存储早期对象，三级缓存用于存储函数接口。

### 一级缓存

> 一级缓存可以不等对象初始化之后再存入缓存，设置在实例化完成后存入一级缓存，可以解决循环依赖问题。
>
> 一级缓存可以解决循环依赖问题，但引发获取bean时需要加互斥锁，导致Spring性能降低，故引出二级缓存。

  前面引出了循环依赖的问题，那么该如何解决呢？其实很简单，单单依赖一级缓存就能解决。对于一级缓存，我们不再等对象初始化完成之后再存入缓存，而是等对象实例化完成后就存入一级缓存，由于此时缓存中的bean还没有进行初始化操作，可以称之为早期对象，也就是将A对象提前暴露了。这样B在创建过程中获取A的时候就能从缓存中获取到A对象，最后在A对象初始化工作完成后再更新一级缓存即可，这样就解决了循环依赖的问题。

  但是这样又引出了另一个问题：早期对象和完整对象都存在于一级缓存中，如果此时来了其它线程并发获取bean，就可能从一级缓存中获取到不完整的bean，这明显不行，那么我们不得已只能在从一级缓存获取对象处加一个互斥锁，以避免这个问题。

  而加互斥锁也带来了另一个问题，容器刷新完成后的普通获取bean的请求都需要竞争锁，如果这样处理，在高并发场景下使用spring的性能一定会极低。

### 二级缓存

  既然只依赖一级缓存解决循环依赖需要靠加锁来保证对象的安全发展，而加锁又会带来性能问题，那该如何优化呢？答案就是引入另一个缓存。这个缓存也是一个Map结构，key为beanName，value为bean，而这个缓存可以称为二级缓存。我们将早期对象存到二级缓存中，一级缓存还是用于存储完整对象（对象初始化工作完成后），这样在接下来B创建的过程中获取A的时候，先从一级缓存获取，如果一级缓存没有获取到，则从二级缓存获取，虽然从二级缓存获取的对象是早期对象，但是站在对象内存关系的角度来看，二级缓存中的对象和后面一级缓存中的对象（指针）都指向同一对象，区别是对象处于不同的阶段，所以不会有什么问题。

  既然获取bean的逻辑是先从一级缓存获取，没有的话再从二级缓存获取，那么也可能出现其它线程获取到不完整对象的问题，所以还是需要加互斥锁。不过这里的加锁逻辑可以下沉到二级缓存，因为早期对象存储在二级缓存中，从一级缓存获取对象不用加锁，这样的话当容器初始化完成之后，普通的getBean请求可以直接从一级缓存获取对象，而不用去竞争锁。

### 二级缓存，当循环依赖遇上AOP

  似乎二级缓存已经解决了循环依赖的问题，看起来也非常简单，但是不要忘记Spring提供的另一种特性：AOP。Spring支持以CGLIB和JDK动态代理的方式为对象创建代理类以提供AOP支持，在前面总结bean生命周期的文章中已经提到过，代理对象的创建(通常)是在bean初始化完成之后进行（通过BeanPostProcessor后置处理器）的，而且按照正常思维来看，一个代理对象的创建也应该在原对象完整的基础上进行，但是当循环依赖遇上了AOP就不那么简单了。

  还是在前面A和B相互依赖的场景中，试想一下如果A需要被代理呢？由于二级缓存中的早期对象是原对象，而代理对象是在A初始化完成之后再创建的，这就导致了B对象中引用的A对象不是代理对象，于是出现了问题。

  要解决这问题也很简单，把代理对象提前创建不就行了？也就是如果没有循环依赖，那么代理对象还是在初始化完成后创建，如果有循环依赖，那么就提前创建代理对象。那么怎么判断发生了循环依赖呢？在B创建的过程中获取A的时候，发现二级缓存中有A，就说明发生了循环依赖，此时就为A创建代理对象，将其覆盖到二级缓存中，并且将代理对象复制给B的对应属性，解决了问题。当然，最终A初始化完成之后，在一级缓存中存放的肯定是代理对象。

  如果在A和B相互依赖的基础上，还有一个对象C也依赖了A：A依赖B，B依赖A，A依赖C，C依赖A。那么在为A创建代理对象的时候，就要注意不能重复创建。

可以在对象实例化完成之后，将其beanName存到一个Set结构中，标识对应的bean正在创建中，而当其他对象创建的过程依赖某个对象的时候，判断其是否在这个set中，如果在就说明发生了循环依赖。


### 三级缓存

  虽然仅仅依靠二级缓存能够解决循环依赖和AOP的问题，但是从解决方案上看来，维护代理对象的逻辑和getBean的逻辑过于耦合，Spring没有采取这种方案，而是引入了另一个三级缓存。三级缓存的key还是为beanName，但是value是一个函数（ObjectFactory#getObject方法），在该函数中执行获取早期对象的逻辑：getEarlyBeanReference方法。 在getEarlyBeanReference方法中，Spring会调用所有SmartInstantiationAwareBeanPostProcessor的getEarlyBeanReference方法，通过该方法可以修改早期对象的属性或者替换早期对象。这个是Spring留给开发者的另一个扩展点，虽然我们很少使用，不过在循环依赖遇到AOP的时候，代理对象就是通过这个后置处理器创建。


### Spring三级缓存源码实现

 那么三级缓存在spring中是如何体现的呢？我们来到DefaultSingletonBeanRegistry中：

~~~java
//一级缓存，也就是单例池，存储最终对象
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
//二级缓存，存储早期对象
private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);
//三级缓存，存储的是一个函数接口，
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
~~~

其实所谓三级缓存，在源码中就是3个Map，一级缓存用于存储最终单例对象，二级缓存用于存储早期对象，三级缓存用于存储函数接口。

  在容器刷新时调用的十几个方法中，finishBeanFactoryInitialization方法主要用于实例化单例bean，在其逻辑中会冻结BeanDefinition的元数据，接着调用beanFactory的preInstantiateSingletons方法，循环所有被管理的beanName，依次创建Bean，我们这里主要关注创建bean的逻辑，也就是AbstractBeanFactory的doGetBean方法（该方法很长，这里只贴了部分代码）：

~~~java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
		//解析FactoryBean的name(&)和别名
		final String beanName = transformedBeanName(name);
		Object bean;
		//尝试从缓存中获取对象
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			//包含了处理FactoryBean的逻辑，可以通过&+beanName获取原对象，通过beanName获取真实对象
			//FactoryBean的真实bean有单独的缓存factoryBeanObjectCache（Map结构）存放
			//如果是普通的bean，那么直接返回对应的对象
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}
		else {
			//只能解决单例对象的循环依赖
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}
			//如果存在父工厂，并且当前工厂中不存在对应的BeanDefinition，那么尝试到父工厂中寻找
			//比如spring mvc整合spring的场景
			BeanFactory parentBeanFactory = getParentBeanFactory();
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				//bean的原始名称
				String nameToLookup = originalBeanName(name);
				if (parentBeanFactory instanceof AbstractBeanFactory) {
					return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
							nameToLookup, requiredType, args, typeCheckOnly);
				}
				else if (args != null) {
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else {
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
			}
			try {

				//处理dependsOn的依赖(不是循环依赖)
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							//循环depends-on 抛出异常
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						registerDependentBean(dep, beanName);
						try {
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
					}
				}
				//创建单例bean
				if (mbd.isSingleton()) {
					//把beanName 和一个ObjectFactory类型的singletonFactory传入getSingleton方法
					//ObjectFactory是一个函数，最终创建bean的逻辑就是通过回调这个ObjectFactory的getObject方法完成的
					sharedInstance = getSingleton(beanName, () -> {
						try {
							//真正创建bean的逻辑
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							destroySingleton(beanName);
							throw ex;
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}
		return (T) bean;
	}
~~~

