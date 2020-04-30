---
title: 单例模式
date: 2016-08-21 21:57:13
tags: [设计模式]
---

单例（Singleton）模式可能是被使用最广泛的一个设计模式之一，也可能是面试中被问到或被要求书写最多的一个设计模式了。该设计模式的主要目的是要求在整个系统中只出现一个类的实例。
下面采用Java语言描述单例模式的几种实现方式。
<!--more--> 
### 懒汉&&线程不安全
	
懒汉式单例在调用取得实例方法的时候实例化对象。

	public class Singleton{
		private staic Singleton singleton = null;
		private Singleton()　{ }
		public static Singleton getInstance(){
			if(singleton == null){
				singleton = new Singleton();
			}
			return singleton;		
		}
	}
从代码中可以看出，采用私有的构造函数，表明这个类是不可能形成实例了，这样可以防止类出现多个实例。所以，采用一个静态的方式（getInstance()）来让其形成实例。在getInstance()方法中，先判断是否已经形成实例，如果已经形成则直接返回，否则创建实例，保存在自己类的私有成员中。     
但是，问题来了，因为是全局性的实例，这种方式最大的问题就是不支持多线程，在多线程情况下，多个线程同时调用getInstance()的话，那么，可能会有多个线程同时通过(singleton == null)的条件检查，最终创建出多个实例。所以，应加入同步机制以适应多线程环境。    

### 懒汉&&线程安全

所以，加入synchronized做同步。得到如下所示的线程安全版本。

	public class Singleton{
		private static Singleton singleton = null;
		private Singleton() { }
		public static synchronized Singleton getInstance(){
			if(singleton == null){
				singleton = new Singleton();
			}
			return singleton;
		}
	}

### 饿汉
	
饿汉式单例在单例类被加载的时候，就实例化一个对象交给自己的引用。

	public class Singleton {
		private static final Singleton singleton = new Singleton();
		private Singleton(){ } 
		public static Singleton getInstance(){
			return singleton;
		}
	}
这种方式在实际中比较常用，基于classloader机制避免了多线程的同步问题，没有加锁，执行效率较高。但是，仍美中不足的是：singleton在类加载的时候就实例化，这样就算是getInstance()没有被调用，类也被初始化了。这样的话，有时会与我们想要的行为不一样，比如，在类的构造函数中，有一些事需要依赖于别的类，我们希望它能在第一次getInstance()时才被真正创建。这样，可以自己控制真正的类创建的时刻，而不是把类的创建委托给类装载器。   
所以又出现了下面这种方式。
	
### 静态内部类

	public class Singleton{
		private static class SingletonHolder{
			private static Singleton singleton = new Singleton();
		}		
		private Singleton() { }
		public static Singleton getInstance(){
			return SingletonHolder.singleton;
		} 
	}
这样的话，由于SingletonHolder是私有的，除了getInstance()之外没有方法访问它，因此只有在getInstance()被调用时才会真正创建。

### 双重锁校验锁

	public class Singleton{
		private volatile static Singleton singleton = null;
		private Singleton() { }
		public static Singleton getInstance(){
			if(singleton == null){
				synchronized(Singleton.class){
					if(singleton == null){
						singleton = new Singleton();
					}
				}
			}
			return singleton;
		}
	}
第一个if判定是说，如果实例创建了，就不需要同步了，直接return singleton就可以了。不然的话，就开始同步线程。第二个if判定是说，如果被同步的线程中，已经创建了对象，那么就不用创建了。这样，保证了多线程条件下的安全。    
此外，由于`singleton = new Singleton();`不是原子操作，所以使用volatile关键字禁止指令重排序优化。

### 枚举

	public enum Singleton{
		INSTANCE;
	}

这样，简直不要太简单哦！是的，虽然他可能还包含实例变量和实例方法。默认枚举实例的创建是线程安全的，所以不需要担心线程安全的问题。当然，需要自行负责枚举中的其他任何方法的线程安全。      
这样，通过Singleton.INSTANCE来访问，这比上述的getInstance()方法简单多了。而且，更重要的是，枚举单例，JVM对序列化有保证。 

### 总结

总结一下上述的单例模式实现，一般情况下，不建议使用两种懒汉方式，建议使用饿汉方式。当要求延迟加载时，使用静态内部类形式。而如果涉及到反序列化创建对象时，可以尝试用枚举方式。   

 
	