## 希尔排序

> 基本思路 : 希尔排序可以看作是插入排序的递归操作
>
> 也就是将一个数组先以``gap``的距离将数组进行插入操作,也就是从之前的``1 ``变成了`` gap`` ,而``gap ``是 从 ``arr.length / 2 ``一直到 ``0`` 所以他的时间复杂度 是 ``nlogn`` 级别的 最坏是``n^2logn``

```java
public void shellSort(int[] arr)
{
    int n = arr.length / 2;
    for(int gap = n; gap > 0; gap /= 2)
    {
		for(int i = gap; i < arr.length; i ++)
        {
            int j = i - gap;
            int key = arr[i];
            while(j >= 0 && arr[j] > key)
            {
                arr[j + gap] = arr[j];
                j -= gap;
            }
            arr[j + gap] = key;
        }
    }
}
```

