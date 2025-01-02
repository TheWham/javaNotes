## 堆栈的用法

__基本介绍:__堆栈内部是由一个``array``和一个``index``组成有一个``maxsize``来表示最大容量

基本结构

```java
public class stack
{
    int [] array;
    int maxsize;
    int index;
   	public stack(int maxSize)
    {
        index = -1;
        this.maxsize = maxSize;
        array = new int[maxSize];
    }
}
```

* 判断栈是否为空

  ```java
  public boolean isEmpty()
  {
      return this.index == -1;
  }
  ```

* 判断栈是否满

  ```java
  public boolean isFull()
  {
      return this.index == maxsize - 1;
  }
  ```

* push 操作 (将元素压入栈)

```java
public void push(int element) throws Exception
{
    if(this.index < maxsize - 1)
    this.array[++ index] = element;
    else throw new Exception("stack is full !");
}
```

* pop 操作(将元素出栈)

> 从栈顶依次的弹出

```java
public int pop() throws RuntimeException
{
    if(this.index >= 0)
        return array[index --];
    else throw new Exception("stack is empty!");
}
```

* peek 操作(返回栈顶)

```java
public int peek()
{
    if(!isEmpty()) return array[index];
    else return throw new RuntimeException("stack is empty!");
}
```



