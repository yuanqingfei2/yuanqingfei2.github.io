---
title: 快速排序面面观
date: "2019-06-19T17:08:00.000Z"
description: "快排序的常见语言实现"
---

偶遇一个有趣网站，把quick-sort的各种语言实现都呈现出来，很有趣，分享之。

## 快排序

[快排序](https://en.wikipedia.org/wiki/Quicksort)是比较好的排序方法，平均性能大概是O(nlogn)。 基本思路如下：

* 在数组中找一个元素，称之为 **pivot**  
* 分片：把所有小于 **pivot**　的元素都放在它前面，反之放在后面。 
* 把以上两步递归调用直到整个数组都变得有序。 

## 实现

### C

```c
void quicksort(int *A, int len) {
  if (len < 2) return;
 
  int pivot = A[len / 2];
 
  int i, j;
  for (i = 0, j = len - 1; ; i++, j--) {
    while (A[i] < pivot) i++;
    while (A[j] > pivot) j--;
 
    if (i >= j) break;
 
    int temp = A[i];
    A[i]     = A[j];
    A[j]     = temp;
  }
 
  quicksort(A, i);
  quicksort(A + i, len - i);
}
```

### Java

Java 1.5 

```java
public static <E extends Comparable<? super E>> List<E> quickSort(List<E> arr) {
    if (arr.isEmpty())
        return arr;
    else {
        E pivot = arr.get(0);
 
        List<E> less = new LinkedList<E>();
        List<E> pivotList = new LinkedList<E>();
        List<E> more = new LinkedList<E>();
 
        // Partition
        for (E i: arr) {
            if (i.compareTo(pivot) < 0)
                less.add(i);
            else if (i.compareTo(pivot) > 0)
                more.add(i);
            else
                pivotList.add(i);
        }
 
        // Recursively sort sublists
        less = quickSort(less);
        more = quickSort(more);
 
        // Concatenate results
        less.addAll(pivotList);
        less.addAll(more);
        return less;
    }
}
```

Java 1.8

```java
public static <E extends Comparable<E>> List<E> sort(List<E> col) {
    if (col == null || col.isEmpty())
        return Collections.emptyList();
    else {
        E pivot = col.get(0);
        Map<Integer, List<E>> grouped = col.stream()
                .collect(Collectors.groupingBy(pivot::compareTo));
        return Stream.of(sort(grouped.get(1)), grouped.get(0), sort(grouped.get(-1)))
                .flatMap(Collection::stream).collect(Collectors.toList());
    }
}
```

### Python

命令型

```python
def quickSort(arr):
    less = []
    pivotList = []
    more = []
    if len(arr) <= 1:
        return arr
    else:
        pivot = arr[0]
        for i in arr:
            if i < pivot:
                less.append(i)
            elif i > pivot:
                more.append(i)
            else:
                pivotList.append(i)
        less = quickSort(less)
        more = quickSort(more)
        return less + pivotList + more
```

函数型

```python
def qsort(list):
    if not list:
        return []
    else:
        pivot = list[0]
        less = [x for x in list     if x <  pivot]
        more = [x for x in list[1:] if x >= pivot]
        return qsort(less) + [pivot] + qsort(more)
```

### Scala

```scala
 def sort[T <: Ordered[T]](xs: List[T]): List[T] = {
    xs match {
      case Nil => Nil
      case x :: xx => {
        val (lo, hi) = xx.partition(_ < x)
        sort(lo) ++ (x :: sort(hi))
      }
    }
  }
```

更抽象的版本，连容器也参数化了

```scala
 def sort[T, C[T] <: scala.collection.TraversableLike[T, C[T]]]
    (xs: C[T])
    (implicit ord: scala.math.Ordering[T],
      cbf: scala.collection.generic.CanBuildFrom[C[T], T, C[T]]): C[T] = {
    // Some collection types can't pattern match
    if (xs.isEmpty) {
      xs
    } else {
      val (lo, hi) = xs.tail.partition(ord.lt(_, xs.head))
      val b = cbf()
      b.sizeHint(xs.size)
      b ++= sort(lo)
      b += xs.head
      b ++= sort(hi)
      b.result()
    }
  }
```

### Lisp

```lisp
(defun quicksort (list)
  (when list
    (destructuring-bind (x . xs) list
      (nconc (quicksort (remove-if (lambda (a) (> a x)) xs))
	     `(,x)
	     (quicksort (remove-if (lambda (a) (<= a x)) xs))))))
```

### 

```haskell
qsort [] = []
qsort (x:xs) = qsort [y | y <- xs, y < x] ++ [x] ++ qsort [y | y <- xs, y >= x]
```

## 总结

* 真正的主流语言如Java和Python都支持命令和函数两种方式实现。 
* 纯粹的函数型语言在实现此类算法上更有表达能力，比如Haskell，是最精简的实现，才2行。  
* Scala语言看起来是抽象程度最高的实现，不仅是元素，甚至容器都范型化了。这也对得起它比较高的学习曲线了。 
* [rosettacode](https://rosettacode.org/)真是个好网站，非常有心得收集了这些实现，是学习算法也语言的好地方，值得推荐。 

## 感谢

* https://rosettacode.org/

* https://en.wikipedia.org/wiki/Quicksort
