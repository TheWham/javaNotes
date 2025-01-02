## 二叉搜索树

>二叉树搜索树又称二叉排序树(BST), 它的一般规律是树的根结点的``val``大于它的**左结点** 小于它的**右结点**.
>
>二叉搜索树的主要特点:
>
>* 能够快速的增删改查结点(时间复杂度_0(log n)_)
>* **中序遍历有序输出：** 对 BST 进行中序遍历会按照升序（或降序，取决于树的结构）输出树中的所有节点值。这使得 BST 可以用于实现排序算法。
>* 尽管 BST 具有上述优点，但它也有一些局限性。当 BST 的结构接近线性链条时（例如，插入值按照升序排列），查找、插入和删除操作的时间复杂度可能会退化为 O(n)，其中 n 是树中节点的数量。为了克服这个问题，通常采用平衡二叉搜索树(如 AVL 树和红黑树)来保持树的平衡性，以确保操作的平均时间复杂度保持在 O(log n)。

### 二叉搜索树的实现:

1. 基本结构:

   ```java
   class Node
   {
       public int value;
       public Node left;
       public Node right;
       
       public Node(int value)
       {
           this.value = value;
           this.left = null;
           this.right = null;
   	}
       
       public void indexOrder(Node root)
       {
           while(root != null)
           {
   			indexOrder(root.left);
                System.out.println(root.value + " ");
                indexOrder(root.right);
           }
   	}
   }
   
   public class BinarySearchTree
   {
       public Node root;
       
       public BinarySearchTree(Node root)
       {
           this.root = root;
   	}
       public void indexOrder()
       {
           if(root == null) System.out.println("树为空无法遍历!");
           else root.indexOrder(root);
       }
   }
   ```

2. 二叉搜索树的插入操作:

   * 递归无返回值写法

     ```java
     public void add(Node targetNode)
     {
         if(targetNode == null) 
             return;
         if(targetNode.value < root.value)
         {
             //目标结点小于根结点想根结点的左边递归插入
             if(root.left == null)
                 root.left = targetNode;
             else root.left.add(targetNode);
         }
         else
         {
             //目标结点大于根结点想根结点的左边递归插入
             if(root.right == null)
                 root.right = targetNode;
             else root.right.add(targetNode);
         }
     }
     ```

   * 递归有返回值写法

     >比较难以弄清,它主要是先递归搜索树合适的位置然后插入, 搜索位置的过程中是记录上一层记录的结果,为了回溯时将其串在一起,因为递归搜索,返回值先存入栈中,递归完成之后再依次从栈顶的值返回,第一次返回仅仅是目标结点,然后第二次回溯返回的是连接有目标结点的父节点,再一次返回直到返回到根节点为止.

     ```java
     public Node insert(Node root, int value)
     {
         if(root == null)
         {
             //表示已经找到要插入的位置
             root = new Node(value);
             return root;
         }
         if(value < root.value)
         {
             //记录上一层的状态,回溯时候再重新将树的结点连接在一起
             root.left = insert(root.left, value);
         }
         else if(value > root.value)
         {
             root.right = insert(root.right, value);
         }
         //回溯返回根结点
         return root;
     }
     ```

3. 二叉搜索树的搜索操作:

   ```java
   public Node search(int value)
   {
       if(root.value == value) 
           return root;
      	if(root.value > value)
       {
           // 如果左结点为null就找不到目标值了
           if(root.left == null)
               return null;
           else return root.left.search(node);
       }
       else
       {
           if(root.right == null)
               return null;
           else root.right.search(node);
       }
   }
   ```

4. 二叉搜索树的搜索父结点操作

   ```java
   public Node searchParent(Node node)
   {
    if(root.left != null && root.left.value == node.value || root.right != null && root.right.value == node.value)
   	return root;
   	if(root.left != null && root.value > node.value) 
           root.left.searchParent(node);
       else if(root.right != null && root.value <= node.value) 
           root.right.searchParent(node);
       return null;
   }
   ```

5. 二叉搜索树的删除操作

   >删除操作有3种情况
   >
   >1. ### 目标结点为叶子结点:
   >
   >   * 判断目标结点是父结点的左子树还是右子树,然后直接**置空**.
   >
   >2. ### 目标结点有一个子结点:
   >
   >   * 先判断目标结点是父结点的左子树还是右子树,然后判断子结点是目标结点的左子树还是右子树**然后直接将父结点的子结点指向目标结点的子节点**.
   >
   >3. ### 目标结点有两个子结点:
   >
   >   * 先判断目标结点是**父结点**的左子树还是右子树,然后寻找目标结点的**右子树**中的**最小值**,找到最小值后将目标结点换成最小值,然后再删除最小值.(_**之所以找目标结点的右子树的最小值是因为,这样删除后,最小值的右边都是比它大,左边还是都比它小的,仍然如何二叉搜索树**_)

   * 无返回值型

   ```java
   public void deleteNode(int value)
   {
       if(root == null) 
           return;
       else
       {
           Node targetNode = search(value);
           
           if(targetNode == null) return null;
           
           if(root.left == null && root.right == null)
           {
               root = null;
               return;
           }       
           Node parentNode = searchParent(targetNode);
           // 第一种情况
          	if(targetNode.left == null && targetNode.right == null)
           {
               if(parentNode.left != null && parentNode.left.value == targetNode.value)
                   parentNode.left = null;
               else if(parentNode.right != null && parentNode.right.value == targetNode.value)
                   parentNode.right = null;
           }
           //第三种情况
           else if(targetNode.left != null && targetNode.right != null)
           {
               //寻找目标结点右子树的最小值
   			int minValue = finMinRightSubTree(targetNode.right);
                targetNode.value = minValue;
           }
           //第二种情况
           else if(targetNode.left != null)
           {
               if(parentNode != null) // 为了避免删除的结点时根节点,因为根结点没有parent 所以调用其parent的子结点会报错
               {
                   if(parentNode.left != null && parentNode.left.value = value)
                   parentNode.left = targetNode.left;
              	   else if(parentNode.right != null && parentNode.right.value == value)
                   parentNode.right = targetNode.left;
               }
               else root = targetNode.left;
           }
           else
           {
               if(parentNode != null)
               {
                  if(parentNode.left != null && parentNode.left.value = value)
                   parentNode.left = targetNode.right;
                 else if(parentNode.right != null && parentNode.right.value == value)
                   parentNode.right = targetNode.right; 
               }
               else root = targetNode.right;
           }
       }
   }
   private int finMinRightSubTree(Node node)
   {
       Node temp = node;
       while(temp.left != null)
           temp = temp.left;
       //删除最小值结点
   	deleteNode(temp.value);
       return temp.value;
   }
   ```

   * 有返回值型

     >代码简单,但是过程不好理解

   ```java
   public TreeNode deleteRec(TreeNode root, int key)
       {
           if (root == null)
           {
               return root;
           }
   
           if (key < root.val)
           {
               // 可以理解为删除结点并连接操作
               root.left = deleteRec(root.left, key);
           }
           else if (key > root.val)
           {
               root.right = deleteRec(root.right, key);
           }
           else
           {
               // 当前节点就是要删除的节点
               if (root.left == null)
               {
                   // 情况1|2：没有左子节点或没有子节点
                   return root.right;
               }
               else if (root.right == null)
               {
                   // 情况2：没有右子节点
                   return root.left;
               }
               
               // 情况3：有两个子节点
               // 找到右子树的最小值节点，即右子树中的最左节点
               root.val = findMinValue(root.right);
               // 删除右子树中的最小值节点
               root.right = deleteRec(root.right, root.val);
           }
   
           return root;
       }
   
       private int findMinValue(TreeNode node)
       {
           int minValue = node.val;
           while (node.left != null)
           {
               minValue = node.left.val;
               node = node.left;
           }
           return minValue;
       }
   ```

   