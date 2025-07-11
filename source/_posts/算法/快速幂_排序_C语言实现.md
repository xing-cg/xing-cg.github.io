---
title: 快速幂_排序_C语言实现
categories:
  - - C
  - - 算法
tags: 
date: 2024/1/18
updated: 2024/1/24
comments: 
published:
---
# FastPower - 快速幂

在ACM中或有的面试中会考到。核心问题是如何加速算法。

$a^n$即n个a相乘。如何转化？

例：$a^{13_{(10)}}$的计算。

我们可以发现：$a^{4_{(10)}}\cdot a^{4_{(10)}} = (a^{4_{(10)}})^{2_{(10)}} = a^{8_{(10)}}$

更一般地，（以下为10进制）
$$
a^1\\
(a^{1})^2 = a^{2}\\
(a^{2})^2 = a^{4}\\
(a^{4})^2 = a^{8}\\
(a^{8})^2 = a^{16}\\
...
$$
则$a^{13_{(10)}}$可以转化为$a^{1101_{(2)}} = a^{8_{(10)}}\cdot a^{4_{(10)}}\cdot a^{1_{(10)}}$，如此，$a^{13_{(10)}}$的计算不用把a傻傻地连乘13次，而是转化为需要计算3个小的数即可。

所以，整个计算过程可以利用前面计算过程中的中间值进行计算，提高效率。

## 具体算法

把指数看作二进制，对指数最后一位判断是否为1（与1按位与），为1则记录下来处理，之后右移1位，接着判断下一位，如此循环。

```c
long long fast_pow(long a, unsigned n)
{
    long long product = 1;
    while(n)
    {
        if(n & 1)
        {
            product *= a;
        }
        a *= a;  // 每次对应 a^2  a^4  a^8 ...
        n >>= 1;
    }
    return product;
}
```

# 随机数

```c
#include<stdio.h>
#include<time.h>
#define SIZE 13
void show(int * arr, int len)
{
    for(int i = 0; i < len; ++i)
    {
        printf("%i  ", arr[i]);
    }
    printf("\n");
}
bool isSort(int * arr, int len)
{
    for(int i = 0; i < len - 1; ++i)
    {
        if(arr[i] > arr[i + 1])
            return false;
    }
    return true;
}
int main()
{
    int arr[SIZE] = { 0 };
    // 设置一个随即因子
    srand((unsigned)(time(NULL)));
    for(int i = 0; i < SIZE; ++i)
    {
        arr[i] = rand() % 100;
    }
    show(arr, SIZE);  //展示原始数据
    XXX(arr);         //调用排序算法
    show(arr, SIZE);  //展示排后数据
    if(isSort(arr, SIZE))
        printf("数据已经完全有序\n");
    else printf("数据无序\n");
}
```

# 排序算法

有八种典型的排序算法，可归于五大类：

1. 插入排序
    1. 直接插入排序
    2. 希尔排序
2. 选择排序
    1. 简单选择排序
    2. 堆排序
3. 交换排序
    1. 冒泡排序
    2. 快速排序
4. 归并排序
5. 基数排序
## 稳定性

两要素：
1. 两个元素相同
2. 排序后这两个元素是否能保持原来的顺序
## 直接插入排序

插入排序大多数都是两层循环。

直接插入排序的思想：将待排序数据分为两部分，左边为有序的，右边是无序的。从右边拿一个数据插入到左边（为了方便，每次拿右边的第一个数据），需保持左边继续有序，直到右边没有数据。过程可以参照扑克牌插入排序。

### 《算法导论》写法

```c
void straight_insert_sort(int a[], int n)
{
    for(int i = 1; i < n; ++i)
    {
        int tmp = a[i];
        for(int j = i - 1; j > -1; --j)//j指示欲插入的位置的前一个
        {
            if(a[j] > tmp)//移动
            {
                a[j + 1] = a[j];
            }
            else break;//j不减1
        }
        a[j + 1] = tmp;
    }
}
```

for循环体中的条件判断可以与for表达式合并

```c
void straight_insert_sort(int a[], int n)
{
    for(int i = 1; i < n; ++i)
    {
        int tmp = a[i];
        for(int j = i - 1; tmp < a[j] && j > -1; --j)
        {
            a[j + 1] = a[j];
        }
        //j指示插入位置的前一个位置
        a[j + 1] = tmp;
    }
}
```

### 《算法》写法

```c
void straight_insert_sort(int a[], int n)
{
    for(int i = 1; i < n; ++i)
    {
        for(int j = i; a[j - 1] > a[j] && j > 0; --j)
        {
            swap(&a[j], &a[j - 1]);
        }
    }
}
```

## 希尔排序 - 插入排序的优化

```c
void shell_sort(int a[], int n)
{
    for(int k = n / 2; k > 0; k /= 2) // k为组距或组数
    {
        // 组内进行"直接插入排序"
        for(int i = k; i < n; i += k)
        {
            for(int j = i; j > k - 1 && a[j - k] > a[j]; j -= k)
            {
                swap(&a[j - k], &a[j]);
            }
        }
    }
}
```

```c
void shell_sort(int a[], int n)
{
    for(int k = n / 2; k > 0; k /= 2) // k为组距或组数
    {
        // 组内进行"直接插入排序"
        for(int i = k; i < n; i += k)
        {
            int tmp = a[i];
            for(int j = i - k; j > -1 && a[j] > tmp; j -= k)
            {
                a[j + k] = a[j];
            }
            a[j + k] = tmp;
        }
    }
}
```

## 简单选择排序

```c
void selection_sort(int a[], int n)
{
    int min_idx;
    for(int i = 0; i < n; ++i)
    {
        min_idx = i;
        for(int j = i + 1; j < n; ++j)
        {
            if(a[j] < a[min_idx])
            {
                min_idx = j;
            }
        }
        swap(&a[i], &a[min_idx]);
    }
}
```

优化：i可以优化为`i < n - 1`

```c
void selection_sort(int a[], int n)
{
    int min_idx;
    for(int i = 0; i < n - 1; ++i)
    {
        min_idx = i;
        for(int j = i + 1; j < n; ++j)
        {
            if(a[j] < a[min_idx])
            {
                min_idx = j;
            }
        }
        swap(&a[i], &a[min_idx]);
    }
}
```

## 堆排序

堆的作用是产生一个极值元素。

堆是一种完全二叉树。（区分概念：二叉树、满二叉树、完全二叉树）

总体的排序过程：首先要保证整个二叉树是一个大（小）根堆，即**每一个父节点都要比其子节点大（小）**，如此就能保证整个树的根节点就是最大（小）的节点，即产生了一个极值元素，我们把它和未排序的最后一个叶子节点互换值，而交换完之后，根节点打乱了大（小）根堆的结构，需要重新保证整个二叉树是一个大（小）根堆，才能产生极值元素与最后位置交换。如此，循环往复，每找到一个极值点，就换其到最后（归到已排序数列中），就可以达到排序的目的。

**其中，找极值元素，即保证树为大（小）根堆的过程，是一个递归过程**：因为大（小）根堆的概念就是：**完全二叉树中，每一个父节点都要比其子节点大（小）**。由于涉及到树，并且涉及到层层相扣的大小关系，因此堆化的过程是一个递归过程。

如果把数据抽象为堆结构去排序，那么首先我们需要建立堆。需要从最后一个非叶子节点进行逆向遍历，依次以节点为根进行堆化。

### 堆化函数

注意！！！堆化函数只适用于：只有根节点不是大（小）根堆，而其他节点符合大（小）根堆 的情况。

```c
void heapify(int a[], int root_idx, int last_idx)
{
    int left = root_idx * 2 + 1; //root最小值可能为0
    int right = left + 1;
    // 判断该根节点是否有子节点，如果left超过了last_idx值则退出
    if(left > last_idx) return;
    int max = root_idx;
    // 运行到这里时，left > last_idx，即last_idx <= left，由于是完全二叉树的逻辑结构，因此max一定有左子树，所以我们直接看[left]和[max]的大小关系
    if(a[left] > a[max])
    {
        max = left;
    }
    // 但我们不能确定是否max有右子树，需要判断
    if(right <= last_idx)
    {
        if(a[right] > a[max])
        {
            max = right;
        }
    }
    if(max != root_idx)
    {
        exchange(a, &a[root_idx], &a[max]);
        heapify(a, max, last_idx);
    }
}
```
### 建堆

注意！！！堆化函数只适用于只有根节点不是大（小）根堆，而其他节点符合大（小）根堆 的情况。

因此在堆化之前我们应该先建立一个大（小）根堆！

怎么建立大（小）根堆呢？我们要利用上面这个堆化函数。而刚才一直在反复强调，堆化函数只能用于除了根节点没有堆化，其他节点已全部堆化的情况。所以：只能从非叶子节点逆着依次堆化！最后一个非叶子节点为：`(last_idx - 1) / 2`。（`last_idx`表示数组下标，从0开始）

```c
void create_heap(int a[], int root_idx, int last_idx)
{
    for(int i = (last_idx - 1) / 2; i > -1; --i)
    {
        heapify(a, i, last_idx);
    }
}
```

### 排序

如上文，每次产生一个极值元素，和未排序的最后一个叶子节点互换值，就可以达到排序的目的。

```c
void heap_sort(int a[], int n)
{
    create_heap(a, 0, n - 1);
    for(int i = n - 1; i > 0; )
    {
        exchange(a, &a[0], &a[i]);
        --i;
        heapify(a, 0, i);
    }
}
```

## 冒泡排序

```c
void bubble_sort(int a[], int n)
{
    for(int i = 0; i < n - 1; ++i)
    {
        for(int j = n - 1; j > 0 + i; --j)
        {
            if(a[j - 1] > a[j])
            {
                swap(&a[j - 1], &a[j]);
            }
        }
    }
}
```

优化：

```c
void bubble_sort(int a[], int n)
{
    for(int i = 0; i < n - 1; ++i)
    {
        int is_exchanged = 0;
        for(int j = n - 1; j > 0 + i; --j)
        {
            if(a[j - 1] > a[j])
            {
                swap(&a[j - 1], &a[j]);
                is_exchanged = 1;
            }
        }
        if(!is_exchanged)
            return;
    }
}
```

## 快速排序

在快速排序中，最重要的即为划分过程，每个数的位置将在每一次划分后确定。

那么，基准值（pivot）其实可以从未定序列（除了确定位置的其他值）中随机选择。

在不同教材中，讲解的快速排序往往有两大方案，一是每次选择左首为基准，二是每次选择右尾为基准进行划分。

在《算法导论》中，选择的是右尾；在《算法》（普林斯顿大学）中选择的是左首。
在bubo讲解中，选择的是右尾；在老杨派系的讲解下，选择的是左首。
在网络上，大部分选择的是左首。

除了基准位置的选取不同。迭代方向也有所不同：分为单向（单轴）法（需要分左右）、双轴法等。

这些各式各样的方法，无所谓其细节，其实达到的效果都是一样的，即我们需要关注其核心本质，不要忘了快速排序中，划分函数的初心是什么。

所以，我们可以想象，如果每次选择左首划分，那么其实给元素的位置交换多了一些麻烦，即出现不同的情况。

而如果选择右尾划分，我们可以发现，划分区域的扩张过程很简洁，那么元素的位置交换也不需要太多的分情况，所以思路相对简洁。

总的来说，划分的方式有三大要素：左首、右尾划分；单向、双向划分；左、右划分（如果是双向划分则是先左后右或先右后左）

则可分为：

1. 左首、单向、左划分；
2. 左首、单向、右划分；
3. 左首、双向、先左后右划分；
4. 左首、双向、先右后左划分；
5. 右尾、单向、左划分；
6. 右尾、单向、右划分；
7. 右尾、双向、先左后右划分；
8. 右尾、双向、先右后左划分；
### 3：左首、双向（先左）划分（《算法》 - 普林斯顿大学）

```c
//此方法是前置++，i和j的位置最后不一定重合，会错开1位！
//由于是以左首划分，那么最后替换到左首的值则应该是一个小于（等于）左首的值。
//以上两个条件决定了，必须选择a[j]（即j停下来的位置的值）与左首交换，则j就作为确定的划分界限。

//另外，左右划分的顺序无所谓！先左先右都可以！
void quick_sort(int a[], int left, int right)
{
    if (left >= right) return;
    int i = left, j = right + 1;//为了便于下面++i和--j的代码统一性，所以从left到right+1，实际上只考察到了left+1到right的部分
    int pivot = a[left];
    while(1)
    {
        while(a[++i] <= pivot)
            if(i == right) break;
        while(a[--j] > pivot)
            if(j == left) break;
        if(i >= j) break;
        swap(&a[i], &a[j]);
    }
    swap(&a[left], &a[j]);// a[left]为左首元素，即基准值
    quick_sort(a, left, j - 1);
    quick_sort(a, j + 1, right);
}
```

### 4：左首、双向、先右后左划分（仿照bubo的“右尾、双向、先左后右划分”方法写的，见7）

```c
void quick_sort(int a[], int left, int right)
{
    if (left >= right) return;
    int l = left + 1, pivot = left, r = right;
    while (l < r)
    {
        if (a[r] > a[pivot])
            --r;
        else if (a[l] <= a[pivot]) 
            ++l;
        else
            swap(&a[l], &a[r]);
    }
    if (a[r] > a[pivot])
    {
        swap(&a[r - 1], &a[pivot]);
        pivot = r - 1;
    }
    else
    {
        swap(&a[r], &a[pivot]);
        pivot = r;
    }
    quick_sort(a, left, pivot - 1);
    quick_sort(a, pivot + 1, right);
}
```

上面的代码是初版，最后的`if-else`想复杂了，以下是优化：

```c
void quick_sort(int a[], int left, int right)
{
    if (left >= right) return;
    int l = left + 1, pivot = left, r = right;
    while (l < r)
    {
        if (a[r] > a[pivot])
            --r;
        else if (a[l] <= a[pivot])
            ++l;
        else
            swap(&a[l], &a[r]);
    }
    if (a[l] < a[pivot])
    {
        swap(&a[l], &a[pivot]);
        pivot = l;
    }
    quick_sort(a, left, pivot - 1);
    quick_sort(a, pivot + 1, right);
}
```

### 5：右尾、单向、左划分（北航算法）

```c
void quick_sort(int a[], int left, int right)
{
    if(left >= right) return;
    int i = left, j = left;
    int pivot = a[right];
    while(j < right)
    {
        if(a[j] > pivot)
        {
            ++j;
        }
        else
        {
            swap(&a[i + 1], &a[j]);
            ++i;
            ++j;
        }
    }
    a[right] = a[i + 1];
    a[i + 1] = pivot;
}
```

`if-else`优化：

```c
void quick_sort(int a[], int left, int right)
{
    if(left >= right) return;
    int i = left, j = left;
    int pivot = a[right];
    while(j < right)
    {
        if(a[j] <= pivot)
        {
            swap(&a[i + 1], &a[j]);
            ++i;
        }
        ++j;
    }
    a[right] = a[i + 1];
    a[i + 1] = pivot;
}
```

以上程序，`i`的初始值错误，不应是`left`，而是`left - 1`：怎么记住呢？我们可以想象一下，如果划分两个元素，那么`right = 1`，`left = 0`。如果`i`一开始和`j`都等于`left`，那么如果`a[j] <= pivot`，则`a[i + 1]`和`a[j]`交换，这时`i + 1`是`right`，造成小数在右，所以错误。

实际上，`i`、`j`指示器分别代表左、右部分的末端。两个极端的情况就是：如果`i`超出数组左边界（值为`-1`）则说明左部分为空，即没有比基准值小于（等于）的值；如果`j`超出待排数组右边界（当以右尾划分时，`j`超出边界时值为`right`）则说明右部分为空，即没有比基准值大的值。

遍历数组过程中，主要依托`j`的行进，如果遇到的值比基准值大，则`j`单独进步；如果遇到值比基准值小（等），则需要与`右部分`的首位置替换，如此，遇到的小的值去了`左部分`，原先首位置的值换到了`右部分`的最末端。最后，`i`和`j`同时进一步。

```c
void quick_sort(int a[], int left, int right)
{
    if(left >= right) return;
    int i = left - 1, j = left;
    int pivot = a[right];
    while(j < right)
    {
        if(a[j] <= pivot)
        {
            swap(&a[i + 1], &a[j]);
            ++i;
        }
        ++j;
    }
    a[right] = a[i + 1];
    a[i + 1] = pivot;
    quick_sort(a, left, i);
    quick_sort(a, i + 2, right);
}
```

### 7：右尾、双向、先左后右划分：（bubo）

```c
//这个版本下，left和right必定相等
//但是，存在一种情况，即只有两个元素进行划分，如2、3；
//那么此时left和right一上来就相等，都指向2
//如果我们不去做a[l]和a[pivot]的大小对比，那么最后就会划分为3、2（3为基准），此时发现，2比3小，却错误的排到后面了。
//所以，必须在后面加上a[l]和右尾值的大小对比！才能进行替换

//另外，此方法不能交换划分顺序！即必须先左划分，后右划分，否则错误！
void quick_sort(int a[], int left, int right)
{
    if (left >= right) return;
    //此时若只有两个元素，则出现l == r
    int l = left, pivot = right, r = right - 1;
    while (l < r)
    {
        if (a[l] <= a[pivot])
            ++l;
        else if (a[r] > a[pivot])
            --r;
        else
            swap(&a[l], &a[r]);
    }
    if (a[l] > a[pivot])
    {
        swap(&a[l], &a[pivot]);
        pivot = l;
    }
    quick_sort(a, left, pivot - 1);
    quick_sort(a, pivot + 1, right);
}
```

### 8：右尾、双向、先右后左划分；（仿照“左首、双向（先左）划分”《算法》写的）

```c
void quick_sort(int a[], int left, int right)
{
    if (left >= right) return;
    int i = left - 1, j = right;//为了便于下面++i和--j的代码统一性，所以从left-1到right，实际上只考察到了left到right-1的部分
    int pivot = a[right];
    while (1)
    {
        while (a[--j] > pivot)
            if (j == left) break;
        while (a[++i] <= pivot)
            if (i == right) break;
        if (i >= j) break;
        swap(&a[i], &a[j]);
    }
    swap(&a[right], &a[i]);// a[right]为右尾元素，即基准值 //必须和a[i]交换，否则不对
    quick_sort(a, left, i - 1); // 必须是i - 1，否则不对
    quick_sort(a, i + 1, right);// 必须是i + 1，否则不对
}
```

其实我们发现，先左后右划分也正确，但是后三句中的基准位置是`i`，不能是`j`

```c
void quick_sort(int a[], int left, int right)
{
    if (left >= right) return;
    int i = left - 1, j = right;//为了便于下面++i和--j的代码统一性，所以从left-1到right，实际上只考察到了left到right-1的部分
    int pivot = a[right];
    while (1)
    {
        while (a[++i] <= pivot)
            if (i == right) break;
        while (a[--j] > pivot)
            if (j == left) break;
        if (i >= j) break;
        swap(&a[i], &a[j]);
    }
    swap(&a[right], &a[i]);// a[right]为右尾元素，即基准值 //必须和a[i]交换，否则不对
    quick_sort(a, left, i - 1); // 必须是i - 1，否则不对
    quick_sort(a, i + 1, right);// 必须是i + 1，否则不对
}
```

### 总结

```
若用前置++，i和j的位置最后不一定重合，可能会错开1位！
由于是以左首划分，那么最后替换到左首的值则应该是一个小于（等于）左首的值。
以上两个条件决定了，必须选择a[j]（即j停下来的位置的值）与左首交换，则j就作为确定的划分界限。
相反，如果以右尾划分，那么替换到右尾的值则应该是一个大于（等于）右尾的值。则必须选择a[i]与右尾交换，则i就作为确定的划分界限。
```

综上所述，快排划分函数的细节需要注意：

1. 如果你选择的是双向划分——最终的`i`、`j`指针是一定重合吗？有可能错开1位吗？
    1. 如果错开1位，则先左先右无所谓！
    2. 如果一定重合，则必须有个先左或先右的顺序！**（为什么呢？）**
2. 你选择以左首或右尾划分，决定了你最后替换左首（右尾）的条件和具体位置！这个和`i`、`j`错不错开没关系！如果选择左首划分，则需要拿比左首小的值与之替换；如果选择右尾划分，则需要拿比右尾大的值与之替换；
    1. 如果你的方法有可能使`i`、`j`错开，那么`i`停下的位置一定是比基准值大于（等于）的；`j`停下的位置一定是比基准值小于（等于）的
    2. 如果你的方法使`i`、`j`最后一定重合，那么由于两个元素时`i`、`j`也恰巧重合，所以不能直接替换左首（右尾），需要比较`i`（或`j`，他俩一样，无所谓），如果你选择左首划分，则判断左首是否比`i`（或`j`）位置的值小（等）；如果你选择右尾划分，则判断右尾是否比`i`（或`j`）位置的值大（等）——若不是，则交换值，更新基准下标（pivot）；

## 归并排序

归并排序涉及到分治（Divide & Conquer）策略的思想。分治策略有很多例子，比如MapReduce用于大规模数据集（大于1TB）的并行运算。涉及"Map（映射）"和"Reduce（归约）"。

需要注意，min和max是绝对下标，即要排序的整个数组的下标。

```c
void divide_conquer(int a[], int min, int max, int b[])
{
    if(min >= max) return;              //终止条件
    int mid = (max - min) / 2 + min;    //找到中间位置(奇数时左边多1个)
    divide_conquer(a, min, mid, b);     //处理完之后，此次递归时的min~max的左半侧已经有序
    divide_conquer(a, mid + 1, max, b); //处理完之后，此次递归时的min~max的有半侧已经有序
    conquer(a, min, max, b);            //conquer即合并两边数组的前提是左右双边均已有序
}
```

conquer即合并两边数组的前提是左右双边均已有序。

```c
void conquer(int a[], int min, int max, int b[])
{
    int mid = (max - min) / 2 + min;
    int i = min;      //左半边
    int j = mid + 1;  //右半边
    int k = min;      //合并的新空间的下标
    while(k <= max)
    {
        if(i <= mid && j <= max)//i和j都没到结尾，则需要一一比对大小
        {
            if(a[i] < a[j])
            {
                b[k] = a[i++];
            }
            else b[k] = a[j++];
        }
        else if(i > mid)        //左半侧已结束，则右半侧(j)剩下的可直接全部放入b
        {
            b[k] = a[j++];
        }
        else b[k] = a[i++];     //右半侧已结束，右半侧(i)剩下的可直接全部放入b
        ++k;
    }
    // 别忘了b数组中处理完成之后还要复制一份给a数组
    for(int k = min; k <= max; ++k)
    {
        a[k] = b[k];
    }
}
```

```c
void merging_sort(int a[], int n)
{
    int * b = (int *)malloc(n * sizeof(a[0]));
    divide_conquer(a, 0, n - 1, b);
    // 无须再把b数组复制给a数组，因为再conquer过程中b数组已经复制给了a数组
    free(b);
}
```

## 基数排序

用的少，考的少，基本上已经退出历史舞台，暂不讨论。
