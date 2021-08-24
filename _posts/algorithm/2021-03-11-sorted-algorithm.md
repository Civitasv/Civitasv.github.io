---
layout: post
title: "排序算法总结"
subtitle: "选择排序、插入排序、希尔排序、归并排序、快速排序"
excerpt: "对常用排序算法进行总结"
date: 2021-03-11
author: "Civitasv"
catalog: true
# header-style: text
header-img: "img/post-bg-sort.jpg"
tags:
  - 排序
  - 算法
---

> 排序就是将一组对象按照某种逻辑顺序重新排列的过程。

## 选择排序

对于数组 arr，从小至大排序，算法步骤如下：

1. 对于`arr[i]`，寻找区间`[i + 1,n]`最小值 min 索引 index；

2. 交换`arr[i]`和`arr[index]`。

特点：

1. 运行时间和输入时是否有序无关；

2. 数据移动最少，仅有`N`次交换。

代码实现：

```java
public static class Selection {
    public static void sort(Comparable[] arr) {
        for (int i = 0; i < arr.length - 1; i++) {
            Comparable ele = arr[i];
            int min = i;
            for (int j = i + 1; j < arr.length; j++)
                if (less(arr[j], arr[min]))
                    min = j;
            exch(arr, min, i);
        }
    }
}
```

## 插入排序

对于数组 arr，从小至大排序，算法步骤如下：

1. 对于`arr[i]`，保持左方有序，判断区间`[0, i]`，若`arr[i] < arr[i - 1]`，则使得大值右移；

2. 直到`arr[i] >= arr[i - 1]`结束右移，插入`arr[i]`。

代码实现：

```java
public static class Insert {
    public static void sort(Comparable[] arr) {
        for (int i = 1; i < arr.length; i++) {
            Comparable temp = arr[i]; // 要插入的元素
            int j = i;
            while (j > 0 && less(temp, arr[j - 1])) {
                arr[j] = arr[j - 1];
                j--;
            }
            if (j != i) { // 存在变化
                arr[j] = temp;
            }
        }
    }
}
```

## 希尔排序

希尔排序基于插入排序优化得来。

> 对于大规模乱序数组插入排序很慢，因为它只会交换相邻的元素，因此元素只能一点一点的从数组的一端移动到另一端，这很不高效。

希尔排序的思想时使数组中任意间隔为`h`的元素都是有序的，这样的数组被称为**h 有序数组**。换句话说，一个 h 有序数组就是由 h 个互相独立的有序数组编织在一起组成的一个数组。这样，在进行排序时，如果 h 很大，我们就能把元素移动到很远的地方，为实现更小的 h 有序创造方便。

代码实现：

```java
public static class Shell {
    public static void sort(Comparable[] arr) {
        int N = arr.length;
        int h = 1;
        while (h < N / 3) h = 3 * h + 1;
        while (h >= 1) {
            for (int i = h; i < N; i++) {
                Comparable temp = arr[i]; // 要插入的元素
                int j = i;
                while (j >= h && less(temp, arr[j - h])) {
                    arr[j] = arr[j - h];
                    j -= h;
                }
                if (j != i) {
                    arr[j] = temp;
                }
            }
            h /= 3;
        }
    }
}
```

## 归并排序

归并排序算法思路：要将一个数组排序，可以先递归地将它分成两半进行排序，然后将结果归并起来。

算法实现：

```java
public static class Merge {
    private static Comparable[] aux;

    public static void sort(Comparable[] arr) {
        aux = new Comparable[arr.length];
        sort(arr, 0, arr.length - 1);
    }

    // 递归
    public static void sort(Comparable[] arr, int lo, int hi) {
        if (lo >= hi) return;
        int mid = lo + (hi - lo) / 2;
        sort(arr, lo, mid);
        sort(arr, mid + 1, hi);
        merge(arr, lo, mid, hi);
    }

    // 自底向上
    public static void sort2(Comparable[] arr) {
        aux = new Comparable[arr.length];
        for (int sz = 1; sz < arr.length; sz = sz + sz) {
            for (int lo = 0; lo < arr.length - sz; lo += sz + sz) {
                merge(arr, lo, lo + sz - 1, Math.min(lo + sz + sz - 1, arr.length - 1));
            }
        }
    }

    private static void merge(Comparable[] arr, int lo, int mid, int hi) {
        // 将arr[lo...mid] 和 arr[mid+1,hi]归并
        int i = lo, j = mid + 1;
        for (int k = lo; k <= hi; k++)
            aux[k] = arr[k];
        for (int k = lo; k <= hi; k++) {
            if (i > mid) arr[k] = aux[j++];
            else if (j > hi) arr[k] = aux[i++];
            else if (less(aux[i], aux[j])) arr[k] = aux[i++];
            else arr[k] = aux[j++];
        }
    }
}
```

## 快速排序

快速排序基于分治思想，将一个数组分成两个子数组，两部分独立地进行排序。

快速排序和归并排序是互补的：归并排序是将数组分成两个子数组分别排序，并将有序的子数组归并以将整个数组排序；

而快速排序将数组排序的方式则是当两个子数组都有序时整个数组自然就有序了。

快速排序的左半部分不大于某个值，右半部分不小于某个值，那么两部分分别排好序后，自然就有序了。

算法实现：

```java
private static void sort(int[] arr) {
    sort(arr, 0, arr.length - 1);
}

private static void sort(int[] arr, int start, int end) {
    if (start >= end) {
        return;
    }

    // partition
    int index = partition(arr, start, end);

    // sort left
    sort(arr, start, index - 1);
    // sort right
    sort(arr, index + 1, end);
}

private static int partition(int[] arr, int start, int end) {
    int partition = arr[end]; // 使用该数据作为分割
    int counter = start; // counter 左侧为所有小于 partition 的数据

    for (int i = start; i < end; i++) {
        if (arr[i] < partition) { // 移动到左侧
            exch(arr, i, counter++);
        }
    }
    exch(arr, end, counter);
    return counter;
}

private static void exch(int[] arr, int a, int b) {
    int temp = arr[a];
    arr[a] = arr[b];
    arr[b] = temp;
}

```
