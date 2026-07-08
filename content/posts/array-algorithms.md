+++
title = "Array Algorithms"
date = 2022-07-12T12:25:04-07:00

[taxonomies]
categories = ["CS"]
tags = ["algorithms", "arrays"]
+++

Arrays store elements in contiguous memory. Access is O(1). Search and modification require iterating. This guide covers six algorithms that appear repeatedly in array problems: binary search, two pointers, sliding window, prefix sums, divide and conquer, and the Dutch national flag.

<!--more-->

## Binary search

Binary search finds an element in a sorted array. Each comparison halves the search space. Time complexity is O(log n).

The midpoint calculation avoids overflow:

```python
mid = left + (right - left) // 2  # not (left + right) // 2
```

In languages with fixed-width integers, `left + right` can overflow. Python handles big integers natively, so overflow is not a concern. The subtraction form remains good practice.

```python
def search(nums, target):
    left, right = 0, len(nums) - 1
    while left <= right:
        mid = left + (right - left) // 2
        if nums[mid] < target:
            left = mid + 1
        elif nums[mid] > target:
            right = mid - 1
        else:
            return mid
    return -1
```

Binary search requires sorted input. For unsorted data, use a hash set (O(n) space, O(1) time) or linear search (O(n) time).

## Two pointers

Two pointers scan an array from both ends, moving toward each other based on a condition. This avoids nested loops.

**Example:** Remove element in-place. Keep elements not equal to `val` at the front of the array.

```python
def remove_element(nums, val):
    k = 0
    for num in nums:
        if num != val:
            nums[k] = num
            k += 1
    return k
```

One pointer iterates through the array. The other tracks the position to write valid elements. Time: O(n). Space: O(1).

**Example:** Sorted two-sum. Find two numbers in a sorted array that sum to `target`.

```python
def two_sum(nums, target):
    left, right = 0, len(nums) - 1
    while left < right:
        s = nums[left] + nums[right]
        if s < target:
            left += 1
        elif s > target:
            right -= 1
        else:
            return [left, right]
    return None
```

## Sliding window

Sliding window maintains a variable-size window over the array. Expand the window by moving the right boundary. Shrink from the left when the window violates a constraint. This pattern solves subarray problems efficiently.

**Example:** Maximum subarray sum (Kadane's algorithm).

```python
def max_sub_array(nums):
    max_sum = current = nums[0]
    for num in nums[1:]:
        current = max(num, current + num)
        max_sum = max(max_sum, current)
    return max_sum
```

`current` represents the maximum subarray sum ending at the current index. `max` tracks the global maximum. When `current + num` is less than `num`, starting a new subarray yields a better sum than extending the previous one.

**Example:** Longest subarray with sum less than `k` (all elements positive).

```python
def max_length_less_than_k(nums, k):
    left = sum_val = 0
    max_len = 0
    for right in range(len(nums)):
        sum_val += nums[right]
        while sum_val >= k:
            sum_val -= nums[left]
            left += 1
        max_len = max(max_len, right - left + 1)
    return max_len
```

When the sum exceeds `k`, shrink from `left` until valid. This works for arrays with positive elements only. Arrays with negative elements require prefix sums and monotonic queues.

## Prefix sums

Prefix sums precompute cumulative sums. Range sum queries become O(1) after O(n) preprocessing.

```python
def prefix_sums(nums):
    prefix = [0]
    for num in nums:
        prefix.append(prefix[-1] + num)
    return prefix

def range_sum(prefix, left, right):
    return prefix[right + 1] - prefix[left]
```

Prefix sums also solve subarray problems. Count subarrays with sum equal to `k`:

```python
from collections import defaultdict

def subarray_sum(nums, k):
    count = defaultdict(int)
    count[0] = 1
    sum_val = result = 0
    for num in nums:
        sum_val += num
        result += count[sum_val - k]
        count[sum_val] += 1
    return result
```

We track how many times each prefix sum has occurred. If `sum_val - k` occurred before, the subarray between that position and now sums to `k`.

## Divide and conquer

Divide and conquer splits the problem, solves recursively, and combines results. Merge sort and quick sort are examples.

**Merge sort** divides the array into halves, recursively sorts each half, then merges.

```python
def merge_sort(nums):
    if len(nums) <= 1:
        return nums
    mid = len(nums) // 2
    left = merge_sort(nums[:mid])
    right = merge_sort(nums[mid:])
    return merge(left, right)

def merge(left, right):
    result = []
    i = j = 0
    while i < len(left) and j < len(right):
        if left[i] <= right[j]:
            result.append(left[i])
            i += 1
        else:
            result.append(right[j])
            j += 1
    result.extend(left[i:])
    result.extend(right[j:])
    return result
```

Time: O(n log n). Space: O(n) for the merge step. Merge sort is stable. It preserves the relative order of equal elements.

## Dutch national flag

Dutch national flag partitions an array into three sections in one pass. It solves grouping problems where elements fall into exactly three categories.

**Example:** Sort an array of 0s, 1s, and 2s.

```python
def sort_colors(nums):
    low = mid = 0
    high = len(nums) - 1
    while mid <= high:
        if nums[mid] == 0:
            nums[low], nums[mid] = nums[mid], nums[low]
            low += 1
            mid += 1
        elif nums[mid] == 1:
            mid += 1
        else:
            nums[mid], nums[high] = nums[high], nums[mid]
            high -= 1
```

Three pointers maintain three regions: 0s at `[0, low)`, 1s at `[low, mid)`, and 2s at `(high, n-1]`. Elements between `mid` and `high` are unknown. One pass sorts the array.

Python's tuple unpacking swaps elements without a temporary variable.

## Pattern selection

Choose the right pattern for the problem:

- **Binary search**: Sorted array, find element or insertion point
- **Two pointers**: Sorted array, pairwise comparison, or in-place removal
- **Sliding window**: Subarray problems with contiguous elements
- **Prefix sums**: Range sum queries or subarray sum counting
- **Divide and conquer**: Sorting, recursive partitioning
- **Dutch national flag**: Three-way partitioning
