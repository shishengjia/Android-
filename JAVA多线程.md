#目录
  * [什么是线程](#什么是线程)
  * [中断线程](#中断线程)
  
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
所以，在Java核心技术中有这样一句话："没有任何语言方面的需求要求一个被中断的程序应该终止。**中断一个线程只是为了引起该线程的注意，被中断线程可以决定如何应对中断 **"。某些线程是如此重要以至于应该处理完异常后继续执行，而不理会中断。跟普遍的情况是，线程将简单地将中断作为一个终止的请求。
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
