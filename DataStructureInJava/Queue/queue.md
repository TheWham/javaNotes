## 队列问题

* 队列的基本构造

  > 队列是由队首和队尾组成的,其中也是用数组来维持队列,也有一个maxsize表示队列容量
  >
  > 队列的基本操作有 入队 出队 遵循 "先入先出"的原则
  >
  > 入队 : 队尾前移
  >
  > 出队 : 队首前移

  ```java
  public class Queue
  {
      private int front;
      private int rear;
      private int maxSize;
      private int[] array;
      public Queue(int maxSize)
      {
          front = -1;
          rear = -1;
          this.maxSize = maxSize;
          array = new int[maxSize];
      }
  }
  ```

* 判断是否为空

  ```java
  public boolean isEmpty()
  {
      return rear == front;
  }
  ```

* 判断是否队满

  ```java
  public boolean isFull()
  {
      return front == maxSize - 1;
  }
  ```

  

* insert操作

  ```java
  public void insert(int element) throws Exception
  {
      if(!isFull())
      {
          array[++ rear] = element; 
      }
      else throw new RuntimeException("queue is full!");
  }
  ```

* pop操作

  ```java
  public int pop() throws Exception
  {
      if(!isEmpty())
      {
          return array[++ front];
      }
      else throw new RuntimeException("queue is empty!");
  }
  ```

* 遍历

  ```java
  public void show()
  {
      //注意要 front + 1 否则会造成溢出
      for(int i = front + 1; i <= rear; i ++)
          System.out.println(array[i] + " ");
  }
  ```

  ## **建造循环队列**

  > 以上普通队列是固定的无法循环使用(原来队列满之后只能做出队操作无法再做入队操作)
  >
  > 基本原理是将长度进行取余操作,以及rear 和 front 是一直在变化的 就可以控制 maxSize不变而 队列的数字可以循环利用
  >
  > 循环对立相当于``rear`` 和 `` front`` 围绕着一个大小为 maxSize 的圈转

  * isFull 函数发生变化  isEmpty不变
  
    ```java
    public boolean isFull()
    {
        //rear + 1 是因为下标从0开始
        return (rear + 1) % maxSize == front;
    }
    ```

  * 初始化 front , rear 不再是 -1 而是 0
  
    ```java
    rear = 0;
    front = 0;
    ```

  * validSize表示现在数组中存在多少元素
  
    ```java
    public int validSize()
    {
        return (rear - front + maxSize) % maxSize;
    }
    ```

  * pop
  
    ```java
    public int pop() throws Exception
    {
        if(isEmpty())
       	throw new RuntimeException("queue is empty!");
        int value = array[front];
        front = (front + 1) % maxSize; // 防止front溢出
    }
    ```
  
  * add
  
    ```java
    public void add(int element) throws Exception
    {
       if(isFull())
       throw new RuntimeException("queue is full!");
       array[rear] = element;
       rear = (rear + 1) % maxSize;
    }
    ```

  * show

    ```java
    public void show()
    {
        for(int i = getFront(); i <= validSize() + getFront; i ++)
            System.out.println(array[i % maxSize] + " ");
    }
    ```
    
    Whole codes show
    
    ```java
    import java.io.*;
    import java.util.*;
    import java.lang.*;
    import static java.lang.System.*;
    
    public class CircleArrayQueueDemo
    {
    	public static void main(String[] args) 
    	{
    		CircleArrayQueue test = new CircleArrayQueue(6);
    		Boolean flag = true;
    		while(true)
    		{
    			out.println("------------------------------------");
    			out.println("h(getHeadElement)");
    			out.println("s(showQueue)");
    			out.println("a(addQueue)");
    			out.println("p(popQueue)");
    			out.println("e(exit)");
    			out.println();
    			out.println("please do your choice:");
    			Scanner sc = new Scanner(in);
    			String str = sc.nextLine();
    			switch(str)
    			{
    				case "h" :
    				try
    				{ 
    					test.getHeadQueue();
    				}
    				catch(RuntimeException e)
    				{
    					out.println(e.getMessage());
    				}
    				break;
    
    				case "a" : 
    					out.print("input the number what you want to add:");
    					int value = sc.nextInt();
    					out.println();
    					try
    					{
    						test.addQueue(value);
    						out.println("add successfully!");
    					}catch(RuntimeException e)
    					{
    						out.println(e.getMessage());
    					}
    					break;
    
    				case "p" : 
    					try
    					{
    						test.popQueue();
    						out.println("pop successfully!");
    					}catch(RuntimeException e)
    					{
    						out.println(e.getMessage());
    					}
    					
    					 break;
    
    				case "e" : flag = false; break;
    				
    				case "s" : 
    					test.showQueue(); 
    					break;	 
    			}
    			if(!flag) break;
    			out.println();
    		}
    	}
    }
    class CircleArrayQueue
    {
    	private int front;
    	private int rear;
    	private int[] data;
    	private int maxSize;
    
    	public CircleArrayQueue(int arrayMaxSize)
    	{
    		maxSize = arrayMaxSize;
    		data = new int[arrayMaxSize];
    		front = 0;
    		rear = 0;
    	}
    
    	public boolean isFull()
    	{
    		return (rear + 1) % maxSize == front;
    	}
    
    	public boolean isEmpty()
    	{
    		return rear == front;
    	}
    
    	public void addQueue(int value)
    	{
    		if(isFull())
    		{ 
    			throw new RuntimeException("queue was fulled,couldn't to add element!");
    		}
    		data[rear] = value;
    		rear = (rear + 1) % maxSize; 
    	}
    
    	public int popQueue()
    	{
    		if(isEmpty())
    		{
    			throw new RuntimeException("queue is empty,couldn't to pop element!");
    		}	
    		int value = data[front];
    		front = (front + 1) % maxSize;
    		return value;
    	}
    
    	public int getHeadQueue()
    	{
    		if(isEmpty())
    		{
    			throw new RuntimeException("queue is empty,couldn't to getHeadElement！");
    		}
    		return data[front];
    	}
    
    	public int getFront()
    	{
    		return  this.front;
    	}
    
    	public void showQueue()
    	{
    		for(int i = this.getFront(); i < validSize() + front; i++)
    			out.printf("data[%d] = %d\n",i % maxSize,data[i % maxSize]);
    	}
    	public int validSize()
    	{
    		return (maxSize + rear - front) % maxSize;
    	}
    }
    ```
    
    
  
  ​                                                                                                                  