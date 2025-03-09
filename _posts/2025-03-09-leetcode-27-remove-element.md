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

## 솔루션

```python
class Solution:
    def removeElement(self, nums: List[int], val: int) -> int:
        index = 0
        for i in range(len(nums)):
            if nums[i] != val:
                nums[index] = nums[i]
                index += 1
        return index
```

- 와, 결국 왼쪽부터 val이 아닌 애들을 채워주는 방식..
- 만약 안 채웠으면 그냥 0 리턴..

## 느낀점
- 저 정도의 시간을 투자하고도 이런 쉬운 문제를 틀리다니... 내가 코테 면접관으로 들어갈 자격이 있나 생각하고 ㅠ
- 틀린 이유가
	- 내 생각이 틀리거나
	- 내가 생각한거랑 다르게 짰거나
- 인데, 전자는 솔루션을 봐야하고 후자는 코딩을 연습해야 한다고 생각, 나는 코딩 연습을 해야 될 거 같다..
- 라고 솔루션 보기 전에 생각했는데, 그냥 솔루션을 어지간하면 보고 문제 푸는 감각을 좀 익혀야 될 듯
- 안 그러면 꾸준히 안 할 듯
