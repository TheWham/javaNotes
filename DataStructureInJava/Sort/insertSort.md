## 插入排序

> 插入排序的基本思路 :
>
> 以从小到大为例:
>
> 插入排序是在数组序列中拿最前面的一个数和他相邻数之间进行比较.
>
> 就是先用临时指针 j 指向数组的第一位数,将key作为相邻的数,然后如果arr[ j ] > key 
>
> 就先让key位置的数 = arr[ j ] ; 然后 j --,  直到 找arr[ j ] < key 或者 j 到达最左端为止  (这一步的意思也就是找到 <= key的数将它 = key)

```java
public void insertSort(int[] arr)
{
    for(int i = 1; i < arr.length; i ++)
    {
        int j = i - 1; //j 在 i 的左边
        int key = arr[i];
        while(j >= 0 && arr[j] > key)
        {
            arr[j + 1] = arr[j]; //因为以相邻数进行比较的 所以 只需 j + 1 即可
            j --;
        }
        arr[j ++] = key;
    }
}
```

