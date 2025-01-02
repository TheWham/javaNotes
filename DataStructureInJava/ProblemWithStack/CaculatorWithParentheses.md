## 中缀表达式计算

> 计算思路:准备两个栈 numberStack 和 operatorStack
>
>  先依次处理只含有'+','-','*','/','()'和数字的字符串. 如果是**数字**压入numberStack ; 如果是**运算符** operatorStack不是空并且 运算符的**优先级小于栈顶运算符的优先级** 的话 就将 numberStack栈顶上的两个数字pop出来, 再结合着operatorStack pop出栈顶的运算符计算,**计算的结果push到numberStack** . 如果operatorStack为空或者 优先级高于operator Stack栈顶的优先级 那么直接将他入operator栈.
>
> 等到字符串遍历完成之后就要清空numberStack 和 operatorStack, 依次取出numberStack的两个元素 与 operatorStack的运算符进行四则运算. 结果 push到numberStack ,最后 numberStack栈中只有一个元素就是最终的结果
>
> 如果表达式含有括号遇到 '(' 入栈继续遍历直到遇到 ')' 为止 然后将括号里面的先进行运算结果压入numberStack 在将 '(' 移除operatorStack栈

* calculate(四则运算)

  ```java
  public int calculate(int num1, int num2, int operator)
  {
      //这里图个方便就用int表示运算符了
      if(operator == '*') return num1 * num2;
      //须注意的地方
      if(operator == '/') return num2 / num1;
      if(operator == '+') return num2 + num1;
      else return num2 - num1;
  }
  ```

  

* eval(运算操作)

  ```java
  public void eval(Stack<Integer> numberStack, Stack<Character> operatorStack)
  {
      int num1 = numberStack.pop();
      int num2 = numberStack.pop();
      //这里需要注意先后因为除法和减法运算要按顺序, 一般是num2在num1下面也就是先入栈的 所以用num2 ['/'|'-'] num1
      int operator = operatorStack.pop();
      // 进行了字符转换
      numberStack.push(calculate(num1, num2, operator));
  }
  ```

* 处理整数操作(整数可能是多位的)

  ```java
  //要新建一下指针指向字符串中第一个数字出现的位置,然后用它进行遍历最后再将原指针指向新指针的位置
  int newIndex = index;
  //建一个字符串用来存多位整数
  StringBuilder numberString = new StringBuilder();
  while(newIndex < EXPRESSION.length && Character.isDigit(ch))
  {
      numberString.append(ch);
      //处理边界
      if(newIndex < EXPRESSION.length - 1) ch = EXPRESSION.charAt(++ newIndex);
      else newIndex ++;
  }
  //还原
  index = newIndex - 1;
  numberStack.push(Integer.parseInt(numberString.toString()));
  ```

* priority(处理优先级) 和 isOperator(判断运算符)

  ```java
  public int priority(int operator)
  {
      if(operator == '*' || operator == '/') return 3;
      else if(operator == '+' || operator == '-') return 2;
      else return 1;
  }
  public boolean isOperator(char ch)
  {
      return ch == '+' || ch == '-' || ch == '/' || ch == '*';
  }
  ```

Whole codes show

```java
import java.util.*;
import java.lang.*;
import java.io.*;

public class Main
{
     public void main(String [] args)
	{
		String EXPERSSION = "231-2319+321*312*231-2312+2-3*67";
		String EXPRESSION = "((1+8)/(1+2))";
        System.out.println("answer is : " + solution(EXPRSSION));
	}
    public static int solution(String expression)
    {
        Stack<Integer> numberStack = new Stack<>();
        Stack<Character> operatorStack = new Stack<>();
        int index = 0;
        while(index < expression.length)
        {
			char ch = expression.charAt(index);
            if(isOperator(ch))
            {
                if(!operatorStack.isEmpty() && priority(ch) <= priority(operatorStack.peek()))
                eval(numberStack, operatorStack);
                operatorStack.push(ch);
            }
            else if(ch == '(')
            {
                operatorStack.push(ch);
            }
            else if(ch == ')')
            {
                while(operatorStack.peek() != '(')
                    eval(numberStack, operatorStack);
                operatorStack.pop();
            }
            else
            {
                int newIndex = index;
                StringBuilder numberString = new StringBuilder();
                for(newIndex < expression.length && Character.isDigit(ch))
                {
                    numberSting.append(ch);
                    if(newIndex < expression.length - 1) ch = expression.charAt(++ newIndex);
                    else newIndex ++;
                }
                numberStack.push(Integer.parseInt(numberString.toString()));
                index = newIndex - 1;
            }
            index ++;
        }
        while(!operatorStack.isEmpty())
            eval(numberStack, operatorStack);
        return numberStack.peek();
	}
    public int calculate(int num1, int num2, int operator)
    {
        //这里图个方便就用int表示运算符了
        if(operator == '*') return num1 * num2;
        //须注意的地方
        if(operator == '/') return num2 / num1;
        if(operator == '+') return num2 + num1;
        else return num2 - num1;
    }
    public void eval(Stack<Integer> numberStack, Stack<Character> operatorStack)
    {
        int num1 = numberStack.pop();
        int num2 = numberStack.pop();
        //这里需要注意先后因为除法和减法运算要按顺序, 一般是num2在num1下面也就是先入栈的 所以用num2 ['/'|'-'] num1
        int operator = operatorStack.pop();
        // 进行了字符转换
        numberStack.push(calculate(num1, num2, operator));
    }
    public int priority(int operator)
    {
        if(operator == '*' || operator == '/') return 3;
        else if(operator == '+' || operator == '-') return 2;
        else return 1;
    }
    public boolean isOperator(char ch)
    {
        return ch == '+' || ch == '-' || ch == '/' || ch == '*';
    }
}

```

