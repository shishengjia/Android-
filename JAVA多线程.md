#目录
  * [什么是线程](#什么是线程)
  
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
当线程的`run()`方法执行方法体最后一条语句，并经由return语句返回，或者出现在方法中没有捕获的异常，线程将终止。（早起版本的`stop()`已被弃用）
