## 快使排序

> 基本思路 : 快速排序是根据一个pivot 然后将小于pivot 的值放在左边,大于pivot的值放在右边
>
> 而pivot的确定就以数组的中间值为例

```java
public void quickSort(int [] arr, int left, int right)
{
    if(left >= right) return;
    int front = left - 1, rear = right + 1; //因为后面携程front ++ , rear -- 所以前面要先退一下
    int pivot = arr[left + right >> 1];
    while(front < rear)
    {
        do front ++; while(arr[front] < pivot);
        do rear --; while(arr[rear] > pivot);
        if(front < rear)
        {
            int temp = arr[front];
            arr[front] = arr[rear];
            arr[rear] = temp;
        }
	}
    quickSort(arr, left, front - 1);
    quickSort(arr, rear + 1, right);
}
```

