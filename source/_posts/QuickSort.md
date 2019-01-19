title: 'QuickSort Application in java'
date: 2018-11-25 23:07:22
tags: Algorithm
categories: [算法]
top: true
---
![I love it when a plan comes together.](http://ww1.sinaimg.cn/large/006Cwrd9gy1fxskn2tpksj31hc0u0guq.jpg)
### 前言：
**对各种基本排序有了解的人都会知道，各种单一的排序都有他自己合适的使用场景，快速排序是综合表现最好的。
而实际应用中的排序可要考虑的实在是太多了，看jdk的排序是怎么做的.**

    java中的Arrays.Sort()方法是我们常用的排序方法，有心的人肯定点进去源码里面看过的，随着jdk的变化这个排序也有持续的变动，说明维护的人
    还是很愿意花精力在这个方法上的，代码对基本算法没有足够了解的人来说看起来还是很吃力的(我说的是以前的我)，花了点时间整理下这个算法。
### 涉及的算法
1. 插入排序(之前有用binary insertion,既二分法找到插入点后copy)
2. 归并排序
3. 快速排序
    - 单轴双切分快排序(带等号)
    - 双轴三切分快排
4. 计数排序(用于数值范围小的情况，byte，short，char类型的时候)
5. timSort(用于分析本身排序情况)

### 大致思路
尽量发掘各自单一排序算法自己优势，当有合适使用条件的时候就使用适合的基本排序：
1. 小数据量：插入排序
2. 适中：快速排序
    - 结合插入排序
    - 划分区间的时候，带等号使用单轴快排，否则双轴快排
    - 递归调用自己 
3. 大量：先分用timSort分析数据本身排序状况，
    - 衡量指标：run(单调升降序长度)和runs(归并次数)
    - 结构化可以就用归并，否则就用快排归并
    
### 具体代码(jdk1.8)
``` java
    /**
     * Sorts the specified range of the array using the given
     * workspace array slice if possible for merging
     *
     * @param a the array to be sorted
     * @param left the index of the first element, inclusive, to be sorted
     * @param right the index of the last element, inclusive, to be sorted
     * @param work a workspace array (slice)
     * @param workBase origin of usable space in work array
     * @param workLen usable size of work array
     */
    static void sort(int[] a, int left, int right,
                         int[] work, int workBase, int workLen) {
            // 对于数据量少的直接使用快排
            if (right - left < QUICKSORT_THRESHOLD) {// 286
                sort(a, left, right, true); // 基于插入，单双轴的快排见下面
                return;
            }
    
            /*  timSort分析数据排序情况， run[i]是第i个run的开始
             *  一个run是一节单调区间(升序或者降序)
             */
            int[] run = new int[MAX_RUN_COUNT + 1];
            int count = 0; run[0] = left;
    
            for (int k = left; k < right; run[count] = k) {
                if (a[k] < a[k + 1]) { // 
                    while (++k <= right && a[k - 1] <= a[k]);
                } else if (a[k] > a[k + 1]) { // 降序
                    while (++k <= right && a[k - 1] >= a[k]);
                    for (int lo = run[count] - 1, hi = k; ++lo < --hi; ) {
                        int t = a[lo]; a[lo] = a[hi]; a[hi] = t;
                    }
                } else { // 相等
                    for (int m = MAX_RUN_LENGTH; ++k <= right && a[k - 1] == a[k]; ) {
                        if (--m == 0) { // 等号多了直接将这里短丢到快排里面
                            sort(a, left, right, true);
                            return;
                        }
                    }
                }
    
                /* 如果数组不是高度结构化用快排代替归并(要合并的次数count太多了)
                 */
                if (++count == MAX_RUN_COUNT) {// 目前MAX_RUN_COUNT = 67
                    sort(a, left, right, true); // 基于插入，单双轴的快排见下面
                    return;
                }
            }
            
            // 下面是基于O(n)辅助空间的非递归归并，贴进来总体太长
            // ...... 归并
            // 结束
        }
```  
下面是核心快排 
```` java
    /**
     * Sorts the specified range of the array by Dual-Pivot Quicksort.
     *
     * @param a the array to be sorted
     * @param left the index of the first element, inclusive, to be sorted
     * @param right the index of the last element, inclusive, to be sorted
     * @param leftmost indicates if this part is the leftmost in the range
     */
        private static void sort(int[] a, int left, int right, boolean leftmost) {
            int length = right - left + 1;
    
            // 使用插入排序
            if (length < INSERTION_SORT_THRESHOLD) { // INSERTION_SORT_THRESHOLD = 47
                // 左边是否是最大
                if (leftmost) {
                    // 普通的插入排序
                    for (int i = left, j = i; i < right; j = ++i) {
                        int ai = a[i + 1];
                        while (ai < a[j]) {
                            a[j + 1] = a[j];
                            if (j-- == left) {
                                break;
                            }
                        }
                        a[j + 1] = ai;
                    }
                } else {
                    /*
                     * 跳过最长升序
                     */
                    do {
                        if (left >= right) {
                            return;
                        }
                    } while (a[++left] >= a[left - 1]);
    
                    /*
                     * 这里也不是普通的的插入排序，
                     * 使用的是双元素插入法更优。
                     */
                    for (int k = left; ++left <= right; k = ++left) {
                        int a1 = a[k], a2 = a[left];
    
                        if (a1 < a2) {
                            a2 = a1; a1 = a[left];
                        }
                        while (a1 < a[--k]) {
                            a[k + 2] = a[k];
                        }
                        a[++k + 1] = a1;
    
                        while (a2 < a[--k]) {
                            a[k + 1] = a[k];
                        }
                        a[k + 1] = a2;
                    }
                    int last = a[right];
    
                    while (last < a[--right]) {
                        a[right + 1] = a[right];
                    }
                    a[right + 1] = last;
                }
                return;
            }
    
            // 快速得到接近七等分的长度 9/64的长度 + 1
            int seventh = (length >> 3) + (length >> 6) + 1;
            // 各个等分点
            int e3 = (left + right) >>> 1; // The midpoint
            int e2 = e3 - seventh;
            int e1 = e2 - seventh;
            int e4 = e3 + seventh;
            int e5 = e4 + seventh;
    
            // Sort these elements using insertion sort
            if (a[e2] < a[e1]) { int t = a[e2]; a[e2] = a[e1]; a[e1] = t; }
    
            if (a[e3] < a[e2]) { int t = a[e3]; a[e3] = a[e2]; a[e2] = t;
                if (t < a[e1]) { a[e2] = a[e1]; a[e1] = t; }
            }
            if (a[e4] < a[e3]) { int t = a[e4]; a[e4] = a[e3]; a[e3] = t;
                if (t < a[e2]) { a[e3] = a[e2]; a[e2] = t;
                    if (t < a[e1]) { a[e2] = a[e1]; a[e1] = t; }
                }
            }
            if (a[e5] < a[e4]) { int t = a[e5]; a[e5] = a[e4]; a[e4] = t;
                if (t < a[e3]) { a[e4] = a[e3]; a[e3] = t;
                    if (t < a[e2]) { a[e3] = a[e2]; a[e2] = t;
                        if (t < a[e1]) { a[e2] = a[e1]; a[e1] = t; }
                    }
                }
            }
    
            // Pointers
            int less  = left;  // The index of the first element of center part
            int great = right; // The index before the first element of right part
    
            if (a[e1] != a[e2] && a[e2] != a[e3] && a[e3] != a[e4] && a[e4] != a[e5]) {
                /*
                 * Use the second and fourth of the five sorted elements as pivots.
                 * These values are inexpensive approximations of the first and
                 * second terciles of the array. Note that pivot1 <= pivot2.
                 */
                int pivot1 = a[e2];
                int pivot2 = a[e4];
    
                /*
                 * The first and the last elements to be sorted are moved to the
                 * locations formerly occupied by the pivots. When partitioning
                 * is complete, the pivots are swapped back into their final
                 * positions, and excluded from subsequent sorting.
                 */
                a[e2] = a[left];
                a[e4] = a[right];
    
                /*
                 * Skip elements, which are less or greater than pivot values.
                 */
                while (a[++less] < pivot1);
                while (a[--great] > pivot2);
                
                // 双轴三切分
    
                /*
                 * Partitioning:
                 *
                 *   left part           center part                   right part
                 * +--------------------------------------------------------------+
                 * |  < pivot1  |  pivot1 <= && <= pivot2  |    ?    |  > pivot2  |
                 * +--------------------------------------------------------------+
                 *               ^                          ^       ^
                 *               |                          |       |
                 *              less                        k     great
                 *
                 * Invariants:
                 *
                 *              all in (left, less)   < pivot1
                 *    pivot1 <= all in [less, k)     <= pivot2
                 *              all in (great, right) > pivot2
                 *
                 * Pointer k is the first index of ?-part.
                 */
                outer:
                for (int k = less - 1; ++k <= great; ) {
                    int ak = a[k];
                    if (ak < pivot1) { // Move a[k] to left part
                        a[k] = a[less];
                        /*
                         * Here and below we use "a[i] = b; i++;" instead
                         * of "a[i++] = b;" due to performance issue.
                         */
                        a[less] = ak;
                        ++less;
                    } else if (ak > pivot2) { // Move a[k] to right part
                        while (a[great] > pivot2) {
                            if (great-- == k) {
                                break outer;
                            }
                        }
                        if (a[great] < pivot1) { // a[great] <= pivot2
                            a[k] = a[less];
                            a[less] = a[great];
                            ++less;
                        } else { // pivot1 <= a[great] <= pivot2
                            a[k] = a[great];
                        }
                        /*
                         * Here and below we use "a[i] = b; i--;" instead
                         * of "a[i--] = b;" due to performance issue.
                         */
                        a[great] = ak;
                        --great;
                    }
                }
    
                // Swap pivots into their final positions
                a[left]  = a[less  - 1]; a[less  - 1] = pivot1;
                a[right] = a[great + 1]; a[great + 1] = pivot2;
    
                // 切分后递归
                sort(a, left, less - 2, leftmost);
                sort(a, great + 2, right, false);
    
                /* 
                 * 如果中间太大包含了大于 length 4/7 的长度
                 * If center part is too large (comprises > 4/7 of the array),
                 * swap internal pivot values to ends.
                 */
                if (less < e1 && e5 < great) {
                    /*
                     * Skip elements, which are equal to pivot values.
                     */
                    while (a[less] == pivot1) {
                        ++less;
                    }
    
                    while (a[great] == pivot2) {
                        --great;
                    }
    
                    /*
                     *   left part         center part                  right part
                     * +----------------------------------------------------------+
                     * | == pivot1 |  pivot1 < && < pivot2  |    ?    | == pivot2 |
                     * +----------------------------------------------------------+
                     *              ^                        ^       ^
                     *             less                      k     great
                     *
                     * Invariants:
                     *
                     *              all in (*,  less) == pivot1
                     *     pivot1 < all in [less,  k)  < pivot2
                     *              all in (great, *) == pivot2
                     *
                     * Pointer k is the first index of ?-part.
                     */
                    outer:
                    for (int k = less - 1; ++k <= great; ) {
                        int ak = a[k];
                        if (ak == pivot1) { // Move a[k] to left part
                            a[k] = a[less];
                            a[less] = ak;
                            ++less;
                        } else if (ak == pivot2) { // Move a[k] to right part
                            while (a[great] == pivot2) {
                                if (great-- == k) {
                                    break outer;
                                }
                            }
                            if (a[great] == pivot1) { // a[great] < pivot2
                                a[k] = a[less];
                                a[less] = pivot1;
                                ++less;
                            } else { // pivot1 < a[great] < pivot2
                                a[k] = a[great];
                            }
                            a[great] = ak;
                            --great;
                        }
                    }
                }
    
                // 第三个切分
                sort(a, less, great, false);
    
            } else { // 单轴快排
                int pivot = a[e3];
    
                /*
                 *   left part    center part              right part
                 * +-------------------------------------------------+
                 * |  < pivot  |   == pivot   |     ?    |  > pivot  |
                 * +-------------------------------------------------+
                 *              ^              ^        ^
                 *             less            k      great
                 *
                 * Invariants:
                 *
                 *   all in (left, less)   < pivot
                 *   all in [less, k)     == pivot
                 *   all in (great, right) > pivot
                 */
                for (int k = less; k <= great; ++k) {
                    if (a[k] == pivot) {
                        continue;
                    }
                    int ak = a[k];
                    if (ak < pivot) { // Move a[k] to left part
                        a[k] = a[less];
                        a[less] = ak;
                        ++less;
                    } else { // a[k] > pivot - Move a[k] to right part
                        while (a[great] > pivot) {
                            --great;
                        }
                        if (a[great] < pivot) { // a[great] <= pivot
                            a[k] = a[less];
                            a[less] = a[great];
                            ++less;
                        } else { // a[great] == pivot
                            a[k] = pivot;
                        }
                        a[great] = ak;
                        --great;
                    }
                }
    
                /*
                 * 切分后递归
                 */
                sort(a, left, less - 1, leftmost);
                sort(a, great + 1, right, false);
            }
        }
````
我承认要有点耐心才能看完，如果你看完了，看别的代码那就是小菜一碟了 - -。

   
    
    
    