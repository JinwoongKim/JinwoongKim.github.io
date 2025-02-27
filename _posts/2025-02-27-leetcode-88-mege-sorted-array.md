---
title: "[Leetcode] 88. Merge Sorted Array"
categories: coding study
tags:
  - leetcode
  - merge
  - array
published: true
---
### LeetCode - Merge Sorted Array
🔗 [문제 링크](https://leetcode.com/problems/merge-sorted-array/description/)
## **제출 답안지 (소요시간: 20분)**

_(이때는 20분 제한을 두고 풀었다.)_

```python
class Solution:
    def merge(self, nums1: List[int], m: int, nums2: List[int], n: int) -> None:
        """
        Do not return anything, modify nums1 in-place instead.
        """
        i0, i1, i2 = m+n-1, m-1, n-1

        if m == 0:
            nums1[:] = nums2
        elif n == 0:
            pass
        else:
            while i1 >= 0 and i2 >= 0:
                if nums1[i1] < nums2[i2]:
                    nums1[i0] = nums2[i2] 
                    i2 -= 1
                elif nums1[i1] >= nums2[i2]:
                    nums1[i0] = nums1[i1] 
                    nums1[i1] = 0
                    i1 -= 1
                i0 -= 1
            
            while i2 >= 0:
                nums1[i0] = nums2[i2]
                i2 -= 1
```

복잡하게 생각하지 않고, **포인터를 세 개 사용**했다.

- 하나는 **nums1의 제일 마지막 인덱스**를 가리키고,
- 하나는 **nums1의 마지막 숫자 이전 값을**,
- 나머지 하나는 **nums2의 마지막 값을 가리키도록** 설정했다.

<img src="images/blog/_posts/2025-02-26-leetcode-88-mege-sorted-array/IMG-20250226092629238.png" width="200">

메인 알고리즘은 매우 단순하다.

1. **i1과 i2를 비교하여 더 큰 숫자를 i0에 넣는다.**
2. **i1을 넣었으면 한 칸 왼쪽으로 이동**
3. **i2를 넣었으면 한 칸 왼쪽으로 이동**

## 결과 : Wrong Answer

오랜만 + 20분 안에 풀려고 해서 힘들긴 했지만
이거 예전에 한 번 풀었던 문제인데.. 당황스러웠다 ㅠ

솔루션을 보자 [링크](https://leetcode.com/problems/merge-sorted-array/solutions/5714203/video-simple-solution-coding-exercise)

```python
class Solution:
    def merge(self, nums1: List[int], m: int, nums2: List[int], n: int) -> None:
        midx = m - 1
        nidx = n - 1 
        right = m + n - 1

        while nidx >= 0:
            if midx >= 0 and nums1[midx] > nums2[nidx]:
                nums1[right] = nums1[midx]
                midx -= 1
            else:
                nums1[right] = nums2[nidx]
                nidx -= 1

            right -= 1
```

와우.. 놀라우리 만치 단순..
내가 너무 어렵게 생각했나 싶다가도, 생각보다 까다로운 케이스가 있었던 듯

솔루션 공부 후 다시 풀고, 10분만에 제출 완료

```python
class Solution:
    def merge(self, nums1: List[int], m: int, nums2: List[int], n: int) -> None:
        """
        Do not return anything, modify nums1 in-place instead.
        """
        dest = m+n-1
        n1_idx = m-1
        n2_idx = n-1
        
        # until nums2 moves all its elements
        while n2_idx >= 0:
            # if nums1 has larger one, move it to dest
            if nums1[n1_idx] > nums2[n2_idx] and n1_idx >= 0:
                nums1[dest] = nums1[n1_idx]
                n1_idx -= 1
            # otherwise, move nums2's element even if they are the same
            else:
                nums1[dest] = nums2[n2_idx]
                n2_idx -= 1
            dest -= 1
```

Beat 100%? 말이 되나 이게..?ㅋㅋ

![[blog/images/blog/_posts/2025-02-27-leetcode-88-mege-sorted-array/IMG-20250227091134723.png]]


2020년에 제출한 솔루션을 보자

```python
class Solution:
    def merge(self, nums1: List[int], m: int, nums2: List[int], n: int) -> None:
        """
        Do not return anything, modify nums1 in-place instead.
        """
        top = len(nums1)-1
        last1 = top-len(nums2)
        last2 = len(nums2)-1
        
        while (last1 >= 0 and last2 >= 0):
            if nums1[last1] > nums2[last2]:
                nums1[top] = nums1[last1]
                top -= 1 
                last1 -= 1 
            else:    
                nums1[top] = nums2[last2]
                top -= 1 
                last2 -= 1 
        
        while last1 >= 0:
            nums1[top] = nums1[last1]
            top -= 1 
            last1 -= 1 
            
        while last2 >= 0:
            nums1[top] = nums2[last2]
            top -= 1 
            last2 -= 1 
         
```

확실히 풀긴 했는데, 상당히 복잡한 듯

## 느낀점
이번에 느낀점은, 생각보다 설계가 중요하다는 것
그리고 손으로 검산하는게 생각보다 중요하다는 것
특별히 배운 자료 구조는 없다!