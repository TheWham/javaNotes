##          归并排序

> 基本思路 : 将一个序列一直分成两等份, 一直分到每组只有两个元素, 再将这两个数排序. 最后再递归逆向排序,将分的两组重新比对按照顺序合成一个序列,因为这两组也是由其它小份按照顺序合成的.
>
> 对比时要借助一个辅助数组来存储, 最后这个数组就是排序好的结果

```java
public class Main
{
    static final int N = 100000;
    static int[] temp = new int[N];
    public static void mergeSort(int [] array, int left, int right)
    {
        if(left >= right) return;
        int mid = left + right >> 1;
        mergeSort(array, left, mid); //左递归
        mergeSort(array, mid + 1, right); //右递归
        int L = left, R = mid + 1, k = 0;
        while(L <= mid && R <= right) // 避免越界
            if(array[L] <= array[R]) temp[k ++] = array[L ++];
        	else temp[k ++] = array[R ++];
        // 然后可能这两组数组还没有全部都存入temp中
        while(L <= mid) temp[k ++] = array[L ++];
        while(R <= right) temp[k ++] = array[R ++];
        for(int i = left, j = 0; i <= right; i ++, j ++)
            array[i] = temp[j];
    }
}

```

