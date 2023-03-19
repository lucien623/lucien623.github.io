---
title: Deadlock in Java
date: 2023-01-05 15:59:32
tags: 
  - Java 
categories:
  - 技术
description: no description 
---

### 形成 deadlock 的四个条件

1. `no preemption（禁止抢夺）`：线程在未使用完资源之前不能被强制剥夺
2. `hold and wait（持有和等待）`：一个线程可以在等待时持有资源
3. `mutual exclusion（互斥）`：资源只能同时分配给一个线程使用，无法多个线程共用
4. `circular waiting（循环等待）`：一系列线程互相持有其它线程所需要的资源，形成一种头尾相接的等待关系。

在介绍死锁的一些类型之前先介绍一下 Java 中的 `synchronized` 关键字，`synchronized` 可以有三种使用方式：

1. 修饰实例方法（锁作用于当前实例）    

    ```java
    public synchronized void method() {}
    ```

2. 修饰静态方法（锁作用于 class 对象）  

    ```java
    public static synchronized void method() {}
    ```

3. 修饰对象（锁作用于括号中的对象）   

    ```java
    public void method() {
    	//此时锁作用于括号中的 this 即当前实例
        synchronized (this) {
    
        }
    }
    ```
 

### deadlock 的几种类型

1. 静态死锁
    
    ```java
    public class LockDemo {
        private final Object objectA = new Object();
        private final Object objectB = new Object();
    
        public void methodA() throws InterruptedException {
            synchronized (objectA) {
                Thread.sleep(3000);
                synchronized (objectB) {
                    System.out.println("methodA");
                }
            }
        }
    
        public void methodB() throws InterruptedException {
            synchronized (objectB) {
                Thread.sleep(3000);
                synchronized (objectA) {
                    System.out.println("methodB");
                }
            }
        }
    
        public static void main(String[] args) {
            LockDemo demo = new LockDemo();
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        demo.methodA();
                    } catch (InterruptedException e) {
                        throw new RuntimeException(e);
                    }
                }
            }).start();
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        demo.methodB();
                    } catch (InterruptedException e) {
                        throw new RuntimeException(e);
                    }
                }
            }).start();
        }
    }
    ```

    如上代码所示，当两个线程分别执行 `methodA` 和 `methodB` 方法时，第一个线程调用持有了对象 `objectA` 的锁，等待对象 `objectB` 的锁，而第二个线程则相反，持有对象 `objectB` 的锁，等待对象 `objectA` 的锁，这样就形成了 circular waiting，导致了死锁。防止这种死锁形成的方法是在两个方法中对两个对象加锁顺序保持一致。

2. 动态死锁

	```java
	public class DynamicLockDemo {

	    public void dynamicMethod(Object objA, Object objB) throws InterruptedException {
	        synchronized (objA) {
	            Thread.sleep(3000);
	            synchronized (objB) {
	                // TODO:
	            }
	        }
	    }

	    public static void main(String[] args) {
	        DynamicLockDemo demo = new DynamicLockDemo();
	        Object objA = new Object();
	        Object objB = new Object();
	        new Thread(new Runnable() {
	            @Override
	            public void run() {
	                try {
	                    demo.dynamicMethod(objA, objB);
	                } catch (InterruptedException e) {
	                    throw new RuntimeException(e);
	                }
	            }
	        }).start();

	        new Thread(new Runnable() {
	            @Override
	            public void run() {
	                try {
	                    demo.dynamicMethod(objB, objA);
	                } catch (InterruptedException e) {
	                    throw new RuntimeException(e);
	                }
	            }
	        }).start();
    	}
	}
	```

	如上所示，当两个线程分别调用 dynamicMethod 方法时，第一个线程获取了对象 objA 的锁，等待 objB 的锁，第二个线程获取了对象 objB 的锁，等待 objA 的锁，从而形成了死锁。解决这种死锁的方法如下：

	```java
	public void dynamicMethod(Object objA, Object objB) throws InterruptedException {
        // 通过 System.identityHashCode 防止重写 hashCode()
        int hashA = System.identityHashCode(objA);
        int hashB = System.identityHashCode(objB);

        if (hashA < hashB) {
            synchronized (objA) {
                Thread.sleep(3000);
                synchronized (objB) {
                    // TODO:
                }
            }
        } else if (hashA > hashB) {
            synchronized (objB) {
                Thread.sleep(3000);
                synchronized (objA) {
                    // TODO:
                }
            }
        } else {
            synchronized (lock) {
                synchronized (objA) {
                    Thread.sleep(3000);
                    synchronized (objB) {
                    }
                }
            }
        }
	}
	```

	通过比较 identityHashCode 的方式，始终把较小值的对象放在前面保证不同线程调用的时候锁顺序相同从而避免产生死锁（当 hashcode 相同时，则在外层再加一个对象锁）。
