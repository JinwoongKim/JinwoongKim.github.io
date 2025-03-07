---
title: "[Leetcode] 27. Remove Element"
categories: coding study
tags:
  - leetcode
  - easy
published: true
---
https://leetcode.com/problems/remove-element/description/

## 설계 (소요시간: 15분)
- two pointers
- T= O(N)
- S=O(1)
- edge case : # of V > # of !V
- termination condition : lp >= rp
## 제출 답안지 (소요시간: 5분)

```python
def removeElement(self, nums: List[int], val: int) -> int:
	lp = 0
	rp = len(nums)-1

	while lp < rp:
		while nums[lp] != val:
			lp+=1
			if lp == len(nums)-1:
				break


		while nums[rp] == val:
			rp-=1
			if rp == 0:
				break
		
		if nums[lp] == val and nums[rp] != val:
			nums[lp] = nums[rp]
			nums[rp] = '_'
	
	return rp+1
```

## 검산 (소요시간: 13분)

## 제출

![](blog/images/blog/_posts/2025-03-07-leetcode-27-remove-element/IMG-20250307094724630.png)
![](../images/blog/_posts/2025-03-07-leetcode-27-remove-element/IMG-20250307094724630.png)
![[blog/images/blog/_posts/2025-03-07-leetcode-27-remove-element/IMG-20250307094733194.png]]
![[blog/images/blog/_posts/2025-03-07-leetcode-27-remove-element/IMG-20250307094733194.png]]
<img src="/images/blog/_posts/2025-03-07-leetcode-27-remove-element/IMG-20250307094733194.png" width=300>
<img src="/images/IMG_3744.jpeg" width=300>
## 결과 : Wrong Answer



## 느낀점
이번에 느낀점은, 생각보다 설계가 중요하다는 것
그리고 손으로 검산하는게 생각보다 중요하다는 것
특별히 배운 자료 구조는 없다!