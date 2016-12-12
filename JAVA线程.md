##目录
* [什么是线程](#什么是线程)
* [中断线程](#中断线程)
* [线程状态](#线程状态)
* [线程属性](#线程属性)
* [同步](#同步)
	* [银行转账的例子](#银行转账的例子)
	* [锁对象](#锁对象)
	* [条件对象](#条件对象)
	* [总结锁和条件的关键之处](#总结锁和条件的关键之处)
	* [synchronized关键字](#synchronized关键字)
	* [同步阻塞](#同步阻塞)
	
什么是线程
-----------
下面是一个没有使用多线程的程序，点击start可以开始运行程序，但是点击close将无法停止程序，其他任务（如close方法）都将被阻塞，直到
当前任务完成。
```java
//主要任务代码
Ball ball = new Ball();
panel.add(ball);
for(int i = 0;i < STEPS;i++){
	ball.move(panel.getBounds());
	panel.paint(panel.getGraphics);
	Thread.sleep(DELAY);
}
```
所以，应该将这段工作代码放置在一个独立的线程中,无论何时点击start，都将启动一个新线程。
```python
class BallRunnable implements Runnable{

	//主要代码，其余代码省略
	public void run(){
		try{
			for(int i = 0;i < STEPS;i++){
				ball.move(panel.getBounds());
				panel.paint(panel.getGraphics);
				Thread.sleep(DELAY);
			}
		}catch(InterruptedException exception){
		}
	}
}
Ball b = new Ball();
panel.add(b);
Runnable r = new BallRunnable(b,panel);
Thread t = new Thread(r);
t.start();
```
 * Thread(Runnable target) 构造一个新线程，用于调用给定target的`run()`方法
 * void start() 启动这个线程，将引发调用`run()`方法。这个方法将立即返回，并且新线程将并行运行
 * void run() 调用关联Runnable的`run()`方法

中断线程
-----------
参考JAVA核心技术，[博客](http://blog.csdn.net/wxwzy738/article/details/8516253)<br>
当线程的`run()`方法执行方法体最后一条语句，并经由return语句返回时，或者出现在方法中没有捕获的异常，线程将终止。（早起版本的`stop()`已被弃用）<br>
 * void interrupt() 向线程发送中断请求。线程的中断状态将被设置为true。如果目前该线程被一个sleep调用阻塞，那么InterruptedException异常将被抛出。
 * static boolean interrupted() 测设当前线程是否被中断,返回线程的上次的中断状态。注意，这是一个静态方法，这一调用会产生副作用----它将当前线程的中断状态重置为false。
 * boolean isInterrupted() 测试线程是否被终止。不像静态的中断方法，这一调用不改变线程的中断状态。
 * static Thread currentThread() 返回代表当前执行线程的Thread对象<br>
 
中断是通过调用Thread.interrupt()方法来做的. 这个方法通过修改了被调用线程的中断状态来告知那个线程, 说它被中断了,但也仅仅只是告诉它而已，并不会终止它。 对于非阻塞中的线程, 只是改变了中断状态, 即Thread.isInterrupted()将返回true,但其本身还是将继续运行; 对于可取消的阻塞状态中的线程, 比如等待在这些函数上的线程, Thread.sleep(), Object.wait(), Thread.join(), 这个线程收到中断信号后, 会抛出InterruptedException, 同时会把中断状态置回为true.但调用Thread.interrupted()会对中断状态进行复位。<br>
**使用场景**<br>
在某个子线程中为了等待一些特定条件的到来, 你调用了Thread.sleep(10000), 预期线程睡10秒之后自己醒来, 但是如果这个特定条件提前到来的话, 来通知一个处于Sleep的线程。又比如说.线程通过调用子线程的join方法阻塞自己以等待子线程结束, 但是子线程运行过程中发现自己没办法在短时间内结束, 于是它需要想办法告诉主线程别等我了. 这些情况下, 就需要中断.

```java
public class Thread3 extends Thread{
    public void run(){  
        while(true){  
            if(Thread.currentThread().isInterrupted()){  
                System.out.println("Someone interrupted me.");  
            }  
            else{  
                System.out.println("Thread is Going...");  
            }
        }  
    }  

    public static void main(String[] args) throws InterruptedException {  
        Thread3 t = new Thread3();  
        t.start(); 
        Thread.sleep(3000);  
        t.interrupt();  
    }  
} 
```
在main线程sleep的过程中由于t线程中`isInterrupted()`为false所以不断的输出`”Thread is going”`。当调用t线程的`interrupt()`后t线程中`isInterrupted()`为true。此时会输出Someone interrupted me.**而且注意到线程并不会因为中断信号而停止运行。因为它只是被修改一个中断信号而已。**<br>
所以，在Java核心技术中有这样一句话：**"没有任何语言方面的需求要求一个被中断的程序应该终止。中断一个线程只是为了引起该线程的注意，被中断线程可以决定如何应对中断 "**。某些线程是如此重要以至于应该处理完异常后继续执行，而不理会中断。跟普遍的情况是，线程将简单地将中断作为一个终止的请求。
```java
//Interrupted的经典使用代码    
    public void run(){    
            try{    
                 ....    
                 while(!Thread.currentThread().isInterrupted()&& more work to do){    
                        // do more work;    
                 }    
            }catch(InterruptedException e){    
                        // thread was interrupted during sleep or wait    
            }    
            finally{    
                       // cleanup, if required    
            }    
    }    
```
在上面代码中，while循环需要不停的检查自己的中断状态。当外部线程调用该线程的interrupt 时，使得中断状态置位即变为true。这时该线程将终止循环，不在执行循环中的do more work了。<br>
这说明: interrupt中断的是线程的某一部分业务逻辑，前提是线程需要检查自己的中断状态(isInterrupted())。<br>
**下面是使用共享变量来真正中断（停止）一个线程-----非阻塞线程**
```java
class Test2 {
	volatile static boolean stop = false;

	public static void main(String args[]) {
		Runnable runnable = new Runnable() {

			@Override
			public void run() {
				while (!stop) {
					System.out.println("Thread is running...");
					long time = System.currentTimeMillis();
					while ((System.currentTimeMillis() - time < 1000) && (!stop)) {
					}
				}
				System.out.println("Thread exiting under request...");
			}
		};

		Thread thread = new Thread(runnable);
		System.out.println("Starting thread...");
		thread.start();
		try {
			Thread.sleep(3000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.println("Asking thread to stop...");

		stop = true;
		try {
			Thread.sleep(3000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.println("Stopping application...");
	}

}
	//	Starting thread...
//	Thread is running...
//	Thread is running...
//	Thread is running...
//	Thread is running...
//	Asking thread to stop...
//	Thread exiting under request...
//	Stopping application...
}

```

**下面是使用共享变量来真正中断（停止）一个线程-----阻塞线程**<br>
这里通过构建一个Thread类的子类定义一个线程，不过这种方法已不推荐，推荐使用上面一段代码的Runnable来实现
```java
class Example3 extends Thread {  
  volatile boolean stop = false;  
  public static void main( String args[] ) throws Exception {  
   Example3 thread = new Example3();  
   System.out.println( "Starting thread..." );  
   thread.start();  
   Thread.sleep( 3000 );  
   System.out.println( "Asking thread to stop..." );  
   thread.stop = true;//如果线程阻塞，将不会检查此变量  
   thread.interrupt();  
   Thread.sleep( 3000 );  
   System.out.println( "Stopping application..." );  
   //System.exit( 0 );  
  }  
  
  public void run() {  
    while ( !stop ) {  
     System.out.println( "Thread running..." );  
      try {  
      Thread.sleep( 1000 );  //阻塞线程
      } catch ( InterruptedException e ) {  
      System.out.println( "Thread interrupted..." );  
      }  
    }  
   System.out.println( "Thread exiting under request..." );  
  }  
}  
Starting thread... 

Thread running... 

Thread running... 

Thread running... 

Asking thread to stop... 

Thread interrupted... 

Thread exiting under request... 

Stopping application... 
```
Thread.interrupt()方法不会中断一个正在运行的线程。这一方法实际上完成的是，在线程受到阻塞时抛出一个中断信号，这样线程就得以退出阻塞的状态。更确切的说，如果线程被Object.wait, Thread.join和Thread.sleep三种方法之一阻塞，那么，它将接收到一个中断异常（InterruptedException），从而提早地终结被阻塞状态。 

线程状态
-----------
* New（新创建）例如 new Thread(r)
* Runnable（可运行）调用start方法，一旦线程开始运行，不必始终保持运行。
* Blocked（被阻塞）当一个线程试图获取一个内部的对象锁，而该锁被其他线程所持有，则该线程进入阻塞状态。
* Waiting（等待）当线程等待另一个线程通知调度器一个条件时，它自己进入等待状态。
* Timed waiting（计时等待）有几个方法有一个超时参数，调用它们进入计时等待，一直保持到超时期满或者接收到适当的通知。
* Terminated（被终止）因为run()方法正常退出而自然死亡，或是因为一个没有捕获的异常终止了run方法而意外死亡

**线程状态图**<br>
![iamge](https://github.com/shishengjia/Android-NetworkArchitecture-Design/blob/master/%E7%BA%BF%E7%A8%8B%E7%8A%B6%E6%80%81%E5%9B%BE.png)

线程属性
------------
**线程优先级**<br>

* void setPriority(int newPriority) 设置线程的优先级。一般使用Thread.NORM_PRIORITY优先级
* static int MIN_PRIORITY 最小优先级，值为1
* static int NORM_PRIORITY 默认优先级，值为5
* static int MAX_PRIORITY 最高优先级，值为10
* static void yield() 导致当前执行线程处于让步状态，如果有其他的可运行线程具有至少与此线程同样高的优先级，那么这些线程接下来会被调度。

**守护线程**<br>

* void setDaemon(boolean isDaemon) 标识该线程为守护线程或用户线程，这一方法必须在线程启动之前调用。<br>
	守护线程的唯一用途是为其他线程提供服务，例如计时线程，它定时发送信号给其他线程或清空过时的高速缓存项的线程。当只剩下守护线程时，虚拟机就退出了。另外，守护线程应该永远不去访问固有资源，如文件，数据库，因为它会在任何时候甚至在一个操作的中间发生中断。

同步
-----------
银行转账的例子
-------------
```java
package test;
/**
 * 银行类
 * @author shi
 */
public class Bank {
	
	//账户数组
	private double[] accounts;
	
	
	/**
	 * 初始化账户数组及对应存款
	 * @param num
	 * @param initialBalance
	 */
	public Bank(int num, double initialBalance) {
		accounts = new double[num];
		for (int i = 0; i < accounts.length; i++)
			accounts[i] = initialBalance;
	}
	
	/**
	 * 打钱到另一个账户
	 * @param from
	 * @param to
	 * @param amount
	 */
	public void transfer(int from,int to,double amount){
		//转账账户余额小于转账金额，直接返回
		if(accounts[from]<amount)
			return;
		System.out.print(Thread.currentThread());
		accounts[from] -=amount;
		System.out.printf("%10.2f from %d to %d",amount,from,to);
		accounts[to] += amount;
		System.out.printf("Total Balance: %10.2f\n", getTotalBalance());
	}
	
	/**
	 * 返回总金额
	 * @return
	 */
	private double getTotalBalance() {
		double sum = 0;
		for(int i = 0;i<accounts.length;i++)
			sum +=accounts[i];
		return sum;
	}
	
	/**
	 * 返回银行账户数量
	 * @return
	 */
	public int size(){
		return accounts.length;
	}
}
```
```java
package test;
/**
 * TransferRunnable 将钱从指定账户打到目标账户
 * @author shi
 */
public class TransferRunnable implements Runnable {
	
	private Bank bank;
	private int fromAccount;
	private double maxAmount;
	private int DELAY = 10;
	
	public TransferRunnable(Bank bank, int fromAccount, double maxAmount) {
		this.bank = bank;
		this.fromAccount = fromAccount;
		this.maxAmount = maxAmount;
	}

	@Override
	public void run() {
		while(true){
			//随机生成一个目标账户
			int toAccount = (int)(bank.size()*Math.random());
			//随机生成一个转账金额
			double amount = maxAmount*Math.random();
			bank.transfer(fromAccount, toAccount, amount);
			try {
				Thread.sleep((int)(DELAY*Math.random()));
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	}
}
```
```java
package test;

public class Test {

	public static final int ACCOUNT_Num = 100;
	public static final int INITIAL_BALANCE = 1000;

	public static void main(String[] args) {
		Bank bank = new Bank(ACCOUNT_Num, INITIAL_BALANCE);
		for (int i = 0; i < ACCOUNT_Num; i++) {
			TransferRunnable r = new TransferRunnable(bank, i, INITIAL_BALANCE);
			Thread t = new Thread(r);
			t.start();
		}
	}
}
//部分测试结果
//Thread[Thread-44,5,main]    370.42 from 44 to 13Total Balance:  100000.00
//Thread[Thread-13,5,main]    614.65 from 13 to 48Total Balance:  100000.00
//Thread[Thread-37,5,main]     29.50 from 37 to 99Total Balance:  100000.00
//Thread[Thread-12,5,main]    851.90 from 12 to 8Total Balance:   99353.53
//Thread[Thread-4,5,main]    566.35 from 4 to 24Total Balance:   98739.28
//Thread[Thread-25,5,main]      1.03 from 25 to 93Total Balance:   98739.28
```
可以看出，银行总额在不断变化。假定两个线程同时执行`accounts[to] += amount;`<br>
问题在于这不是原子操作，该指令可能的处理方式。

 * 1.将accounts[to]加载到寄存器
 * 2.增加amount
 * 3.将结果写回到accounts[tp]
 
 假定第一个线程执行步骤1和2，然后被剥夺了运行权。假定此时第二个线程被唤醒并修改了accounts数组中的同一项，然后第一个线程被唤醒并完成第三个步，这样
 这一动作擦去了第二个线程所做的更新，总金额不再正确。<br>
 
 锁对象
 ------------
 **用锁对象来解决这个同步问题**<br>
 ReentrantLock保护代码块的基本结构
 ```java
 myLock.lock();
 try{
 	
 }finally{
 	myLock.unLock();//为了确保异常抛出这种情况，解锁需要在finally执行，防止其他线程被永远阻塞
 }
 ```
 下面使用一个锁来保护Bank类的transfer方法
 ```java
 package test;

import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * 银行类(其余代码省略)
 * @author shi
 */
public class Bank {
	// 用锁来保护Bank类的transfer方法
	private Lock bankLock = new ReentrantLock();
	/**
	 * 打钱到另一个账户
	 * 
	 * @param from
	 * @param to
	 * @param amount
	 */
	public void transfer(int from, int to, double amount) {
		
		//上锁
		bankLock.lock();
		try {
			// 转账账户余额小于转账金额，直接返回
			if (accounts[from] < amount)
				return;
			System.out.print(Thread.currentThread());
			accounts[from] -= amount;
			System.out.printf("%10.2f from %d to %d", amount, from, to);
			accounts[to] += amount;
			System.out.printf("Total Balance: %10.2f\n", getTotalBalance());
		} finally {
			//解锁
			bankLock.unlock();
		}
	}
}
 ```
 每一个Bank对象都有自己的ReentrantLock对象。如果两个线程视图访问同一个Bank对象，那么锁以串行方式进行服务。但如果两个线程访问不同的Bank对象，
 每一个线程得到不同对象的锁对象，两个线程不会发生阻塞。<br>
 另外，锁是可重入的，**因为线程可以反复地获得已经持有的锁**。锁保持一个持有计数(holdcount)来跟踪对lock方法的嵌套调用。由于这一特性，被一个
 锁保护的代码可以调用另一个使用相同的锁的方法<br>
 例如，给getTotalBalance方法也加上锁。
 ```java
 /**
	 * 返回总金额
	 * 
	 * @return
	 */
	private double getTotalBalance() {
		bankLock.lock();
		try{
			double sum = 0;
			for (int i = 0; i < accounts.length; i++)
				sum += accounts[i];
			return sum;
		}finally{
			bankLock.unlock();
		}
	}
 ````
 `transfer`方法调用`getTotalBalance`方法，这也会封锁`bankLock`对象，此时bankLock对象的持有计数为`2`.当`getTotalBalance`方法退出的时候，持有计数变为`1`，
 当`transfer`方法退出的时候，持有计数为`0`，此时线程释放锁。
 
 条件对象
 ------------
 线程进入临界区，发现在某一条件满足后它才能执行。这时要使用一个条件对象来管理那些已经获得锁但是却不能做有用工作的线程。<br>
 ```java
 if(bank.getBalance[from] >= amount)
 	bank.transfer(from,to,amount);
 ```
 这段代码是不允许的，因为当前线程完全有可能在成功地完成测试，且在调用transfer方法之前被中断。<br>
 当账户中没有足够余额时，应该等待另一个线程向用户注入资金。但由于这一线程获得了对bankLock的排他性访问，别的线程是没有进行存款操作机会的，这是就
 需要条件对象。<br>
 一个锁可以拥有一个或多个相关的条件对象，通过`newCondition`方法来获得一个条件对象。
 ```java
 Condition sufficientFunds = bankLock.newCondition();
 ```
 调用sufficientFunds.await()方法，使当前线程被阻塞，并放弃锁，这样可以使得另一个线程进行增加账户余额的操作。<br>
 **当然，等待获得锁的线程和调用await()方法的线程存在本质上的不同。一旦一个线程调用await方法，它进入该条件的等待集。当锁可用时，该线程不能马上解除
 阻塞，直到另一个线程调用同一条件上的signalAll方法时为止。**<br>
 
 所以，当另一个线程完成转账时，他应该调用`sufficientFunds.sinalAll()`;<br>
 
 这一次调用重新激活因为这一条件而等待的所有线程。它们再次可运行，试图重新进入该对象。一旦锁可用，将从`await`调用返回，获得该锁并从被阻塞的地方继续
 执行。<br>
 
 所以，在该程序中，当一个账户余额发生改变时，等待的线程应该有机会检查余额。所以当完成转账时，应该调用`signalAll`方法。需要注意的是`signalAll`方法不会
 立即激活一个等待线程。它仅仅解除等待线程的阻塞，以便这些线程可以在当前线程退出后同步方法后，通过竞争实现对对象的访问。<br>
 另一个方法是signal，则是随机接触等待集中某个线程的阻塞状态。这比接触所有线程的阻塞更加有效，但是，如果随机选择的线程发现自己仍然不能运行，那么它
 将再次被阻塞。如果没有其他线程再次调用`signal`，那么系统就死锁了。<br>
 **当一个线程拥有某个条件的锁时，它仅仅可以在改条件上调用await,signalAll,signal方法**<br>
 **下面是改进后的Bank类**
 ```java
/**
 * 银行类
 * @author shi
 */
public class Bank {

	// 账户数组
	private double[] accounts;
	// 用锁来保护Bank类的transfer方法
	private Lock bankLock;
	//条件对象
	private Condition sufficientFunds;

	/**
	 * 初始化账户数组及对应存款
	 * 
	 * @param num
	 * @param initialBalance
	 */
	public Bank(int num, double initialBalance) {
		accounts = new double[num];
		for (int i = 0; i < accounts.length; i++)
			accounts[i] = initialBalance;
		bankLock = new ReentrantLock();
		//获得该锁的条件对象
		sufficientFunds = bankLock.newCondition();
	}

	/**
	 * 打钱到另一个账户
	 * @param from
	 * @param to
	 * @param amount
	 * @throws InterruptedException 
	 */
	public void transfer(int from, int to, double amount) throws InterruptedException {
		//上锁
		bankLock.lock();
		try {
			while(accounts[from]<amount)
				sufficientFunds.await();
			System.out.print(Thread.currentThread());
			accounts[from] -= amount;
			System.out.printf("%10.2f from %d to %d", amount, from, to);
			accounts[to] += amount;
			System.out.printf("Total Balance: %10.2f  \n", getTotalBalance());
            sufficientFunds.signalAll();
		} finally {
			//解锁
			bankLock.unlock();
		}
	}

	/**
	 * 返回总金额
	 * 
	 * @return
	 */
	private double getTotalBalance() {
		bankLock.lock();
		try{
			double sum = 0;
			for (int i = 0; i < accounts.length; i++)
				sum += accounts[i];
			return sum;
		}finally{
			bankLock.unlock();
		}
	}

	/**
	 * 返回银行账户数量
	 * @return
	 */
	public int size() {
		return accounts.length;
	}
	
	/**
	 * 返回指定账户余额
	 * @param id
	 * @return
	 */
	public double getBalance(int id){
		return accounts[id];
	}
}

 ```
总结锁和条件的关键之处
---------------------
* 锁用来保护代码片段，任何时刻只能有一个线程执行被保护的代码
* 锁可以管理试图进入被保护代码段的线程
* 锁可以拥有一个或多个相关的条件对象
* 每个条件对象管理哪些已经进入被保护代码但还不能运行的线程方法

synchronized关键字
------------------
从1.0版开始，java中每一个对象都有一个内部锁，如果一个方法用synchronized关键字声明，那对象的锁将保护整个方法，也就是说要调用该方法，线程必须获得内部的对象锁。<br>
内部对象锁只有一个相关条件。wait方法添加一个线程到等待集中，notifyAll/notify方法解除等待线程的阻塞状态。即：<br>
* intrinsicCondition.await() == wait()
* intrinsicCondition.signalAll() == notifyAll()

所以Bank类的transfer和getTotalBalance方法可以这样实现
```java
public synchronized void transfer(int from, int to, double amount) throws InterruptedException {
		//上锁
		bankLock.lock();
		try {
			while(accounts[from]<amount)
				await();
			System.out.print(Thread.currentThread());
			accounts[from] -= amount;
			System.out.printf("%10.2f from %d to %d", amount, from, to);
			accounts[to] += amount;
			System.out.printf("Total Balance: %10.2f  \n", getTotalBalance());
        		notifyAll();
		}
	}
//getTotalBalance方法并没有使用条件对象，所以代码和最初版本一样就行
private double getTotalBalance() {
		double sum = 0;
		for (int i = 0; i < accounts.length; i++)
			sum += accounts[i];
		return sum;
	}

```
当然，将静态方法声明为synchronized也是合法的。<br>
**内部锁和条件的局限**<br>
* 不能中断一个正在试图获得锁的线程
* 试图获得锁时不能设定超时
* 每个锁仅有单一的条件，可能是不够的<br>
所以，最好既不使用Lock/Condition也不使用synchronized关键字。在许多情况下可以使用java.util.concurrent包中的一种机制，它会处理所有的加锁。当然，如果synchronized适合程序，那么可以尽量使用它。如这个银行转账程序。Lock/Condition结构比较独特，在需要使用它的这种独特性时才使用。<br>


同步阻塞
----------
每一个java对象都有一个锁。线程可以通过调用同步方法获得锁。当然也可以通过进入一个同步阻塞来获得锁。<br>
```java
synchronized(obj){
	....
}
```
 于是它获得obj的锁<br>
 对于Bank类的transfer方法
 ```java
 private Object lock = new Object();
 pubic void transfer(int from,int to,int amount){
 	synchronized(lock){
		...
	}
 }
 ```
 
 有时程序员使用一个对象的锁来实现额外的原子操作，实际上称为客户端锁定.例如，Vector类。现在假定在Vector<Double>中存储银行余额
 ```java
 public void transfer(Vector<Double> accounts,int form,int to,int amount){
 	synchronized(accounts){
		acccounts.set(from,accounts.get(from) - amount);
		accounts.set(to,accounts.get(to) + amount);
	}
 }
 ```
 这个方法可以工作，但完全依赖于一个事实，即Vector类对自己的所有可修改方法都使用内部锁。然而Vector类的文档并没有给出这样的承诺。所以客户端锁定是非常
 脆弱的，通常不推荐使用。
 
