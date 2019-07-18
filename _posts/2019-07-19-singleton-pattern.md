---
layout: post
title:  "设计模式之单例模式"
date:   2019-07-17 22:30:03
categories: 设计模式
tags: 设计模式 单例模式
author: ddmcc
---

* content
{:toc}


## 什么是单例模式

确保一个类只有一个实例，并提供一个全局访问点。





## 如何设计

#### 饿汉模式

在类加载的时候就先创建类实例，任务线程访问的时候直接返回实例。

	public class SingletonPatternTest1 {

	    private static SingletonPatternTest1 instance = new SingletonPatternTest1();

	    private SingletonPatternTest1() { }

	    public static SingletonPatternTest1 getInstance() {
		return instance;
	    }

	}


这么做的好处是它是 **线程安全的** ，并且确保了只有一个实例存在。缺点是如果没有用到这个实例，这个实例也会被
创建，浪费资源。

#### 懒汉模式

延迟实例化。先不创建实例，在访问获取实例时，在判断是否已经创建。

	public class SingletonPatternTest1 {

	    private static SingletonPatternTest1 instance;

	    private SingletonPatternTest1() { }

	    public static SingletonPatternTest1 getInstance() {
		if (instance == null) {
		    instance = new SingletonPatternTest1();
		}
		return instance;
	    }
	}


它的优点是在第一次访问的时候才会被创建，避免了创建了没有使用而浪费资源的问题。但是每次访问都需要判断是否创建，
也会影响性能。而且在 **多线程的情况下**，它 **并不是线程安全的**。有可能会产生不同的对象。



#### 懒汉同步式

通过增加synchronized关键字，使得getInstance方法成同步方法，同时只能一个线程能访问。

	public class SingletonPatternTest1 {

	    private static SingletonPatternTest1 instance;

	    private SingletonPatternTest1() { }

	    public static synchronized SingletonPatternTest1 getInstance() {
		if (instance == null) {
		    instance = new SingletonPatternTest1();
		}
		return instance;
	    }
	}


确保了每次只有一个线程进入方法，解决了线程安全的问题。但因为是同步方法，所以会很影响性能。而且我们只需要确保第一次访问的时候不被重复创建实例，
在第一次创建之后，同步方法就成了累赘了。

#### 双重检查加锁

先判断判断实例是否已经创建，否则在进行加锁。保证了只有第一次才会进行同步。

	public class SingletonPatternTest1 {

	    private static volatile SingletonPatternTest1 instance;

	    private SingletonPatternTest1() { }

	    public static SingletonPatternTest1 getInstance() {
		if (instance == null) {
		    synchronized (SingletonPatternTest1.class) {
			if (instance == null) {
			    instance = new SingletonPatternTest1();
			}
		    }
		}
		return instance;
	    }
	}


>这里有个疑问就是：在设计模式书上说，没有volatile会使'双重检查加锁'失效。（原话是JDK1.4或更早的volatile实现会失效）
我们知道volatile是保证了可见性与禁止重排序，synchronized关键字不是已经保证了可见性和禁止重排序了吗？

答案：
synchronized并不能禁止指令重排序，编译器指令可能会重排序。假设new Instance()分为开辟内存地址，初始化对象，对引用变量进行赋值。指令重排序后可能顺序变成了
1,3,2。即另一线程拿到的可能是一个还没有完全初始化完成的实例。


#### 注册表式

Spring IOC就是用这种方式来管理单例bean的，虽然源码要复杂的多，但是设计思想还是差不多的！

	public class SingletonPattenTest {

		private SingletonPattenTest(){}

		private volatile static Map<Class<?>, Object> map = new HashMap<>();

		public static Object getInstance(Class<?> clazz) throws IllegalAccessException, InstantiationException {
			Object obj = map.get(clazz);
			if (obj == null) {
				synchronized (SingletonPattenTest.class) {
					if (obj == null) {
						obj = clazz.newInstance();
						map.put(clazz, obj);
					}
				}
			}
			return obj;
		}
	}


Spring源码

```java
protected <T> T doGetBean(
			final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
			throws BeansException {

		final String beanName = transformedBeanName(name);
		Object bean;

		// Eagerly check singleton cache for manually registered singletons.
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			if (logger.isDebugEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
			// Fail if we're already creating this bean instance:
			// We're assumably within a circular reference.
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// Check if bean definition exists in this factory.
			BeanFactory parentBeanFactory = getParentBeanFactory();
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				String nameToLookup = originalBeanName(name);
				if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
			}

			if (!typeCheckOnly) {
				markBeanAsCreated(beanName);
			}

			try {
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// Guarantee initialization of beans that the current bean depends on.
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dependsOnBean : dependsOn) {
						if (isDependent(beanName, dependsOnBean)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dependsOnBean + "'");
						}
						registerDependentBean(dependsOnBean, beanName);
						getBean(dependsOnBean);
					}
				}

				// Create bean instance.
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
						@Override
						public Object getObject() throws BeansException {
							try {
								return createBean(beanName, mbd, args);
							}
							catch (BeansException ex) {
								// Explicitly remove instance from singleton cache: It might have been put there
								// eagerly by the creation process, to allow for circular reference resolution.
								// Also remove any beans that received a temporary reference to the bean.
								destroySingleton(beanName);
								throw ex;
							}
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}

				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}

				else {
					String scopeName = mbd.getScope();
					final Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope '" + scopeName + "'");
					}
					try {
						Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
							@Override
							public Object getObject() throws BeansException {
								beforePrototypeCreation(beanName);
								try {
									return createBean(beanName, mbd, args);
								}
								finally {
									afterPrototypeCreation(beanName);
								}
							}
						});
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new BeanCreationException(beanName,
								"Scope '" + scopeName + "' is not active for the current thread; " +
								"consider defining a scoped proxy for this bean if you intend to refer to it from a singleton",
								ex);
					}
				}
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

		// Check if required type matches the type of the actual bean instance.
		if (requiredType != null && bean != null && !requiredType.isAssignableFrom(bean.getClass())) {
			try {
				return getTypeConverter().convertIfNecessary(bean, requiredType);
			}
			catch (TypeMismatchException ex) {
				if (logger.isDebugEnabled()) {
					logger.debug("Failed to convert bean '" + name + "' to required type [" +
							ClassUtils.getQualifiedName(requiredType) + "]", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
	}
```


#### 枚举式

	public enum SingletonPattenTest {
		INSTANCE;
	}
	
在通过 **SingletonPattenTest.INSTANCE** 就可以获得对象了

好处：
- 避免反射攻击的问题。 在普通的单例模式中，私有化构造函数并不能阻止创建唯一的实例，可以通过反射 **.setAccessible** 来创建实例。而枚举类并不允许通过反射来创建实例。

- 避免序列化问题问题。 任何一个readObject方法，不管是显式的还是默认的，它都会返回一个新建的实例，这个新建的实例不同于该类初始化时创建的实例。要解决的话可以重写方法readResolve，
- 它会在readObject之后被调用，并替换readObject返回的实例。但需要注意的是，如果单例实例中存在非transient对象引用，就会有被攻击的危险。可以在readResolve之前，对非transient对象引用域
进行操作。
而枚举序列化的时候仅仅是将枚举对象的name属性输出到结果中，反序列化的时候则是通过java.lang.Enum的valueOf方法来根据名字查找枚举对象。
同时，编译器是不允许任何对这种序列化机制的定制的，因此禁用了writeObject、readObject、readObjectNoData、writeReplace和readResolve等方法

- 避免线程安全问题。枚举类所有属性都会被声明称static类型，它是在类加载的时候初始化的，而类的加载和初始化过程都是线程安全的。所以，创建一个enum类型是线程安全的。


#### 静态内部类

	public class SingletonPattenTest{
		
	    private SingletonPattenTest(){}
	 
	    private static class SingletonHoler{
			
			private static SingletonPattenTest instance = new SingletonPattenTest();
		}
	 
		public static SingletonPattenTest getInstance(){
			return SingleTonHoler.instance;
		}
	}
	
优点是：外部类加载时并不需要立即加载内部类，内部类不被加载则不去初始化instance，故而不占内存。即当SingletonPattenTest第一次被加载时，
并不需要去加载SingleTonHoler，只有当getInstance()方法第一次被调用时，才会加载SingleTonHoler类并初始化instance，这种方法不仅能确保线程安全，也能保证单例的唯一性，同时也延迟了单例的实例化。

缺点：
- 需要两个类去做到这一点，虽然不会创建静态内部类的对象，但是其 Class 对象还是会被创建，而且是属于永久带的对象。
- 创建的单例，一旦在后期被销毁，不能重新创建。