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

## 결과 : Wrong Answer

<img src="/images/test.png" width=300>
<img src="/images/IMG-20250307094724630.png" width=300>

## 느낀점
- 저 정도의 시간을 투자하고도 이런 쉬운 문제를 틀리다니... 내가 코테 면접관으로 들어갈 자격이 있나 생각하고 ㅠ
- 틀린 이유가
	- 내 생각이 틀리거나
	- 내가 생각한거랑 다르게 짰거나
- 인데, 전자는 솔루션을 봐야하고 후자는 코딩을 연습해야 한다고 생각, 나는 코딩 연습을 해야 될 거 같다..
- 
