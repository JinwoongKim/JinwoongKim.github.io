---
title:  "예전의 나와 현재 나의 DP 풀이 차이"
categories: "coding_study"
tag: [dynamic programing]
---

[예전 글](http://jinwoongkim.net/diary/coding-study-start/) 에서 언급했듯이 최근 코딩 테스트에서 자꾸 떨어져서 공부중인데, 공교롭게도 떨어진 문제들이 죄다 DP 였다.. 

~~진짜 실무에서 DP 쓰는 사람 있나요?? ㅠㅠ~~ 

DP 의 실무성이 어떻든 간에, 떨어지는게 싫어서 리트코드의 [DP 세션]( https://leetcode.com/explore/learn/card/dynamic-programming/) 를 하나씩 따라가며 공부하던 중, 예전에 풀었던 DP 문제를 만났다 ㅎㅎ

뭔가 풀었다는건 기억이 나지만.. 어떻게 풀었는지는 도통 기억이 안 나서 일단 풀어보았다. 예전엔 되게 고생한거 같은데 이번엔 채 15분이 채 안 걸렸다 ㅎㅎ

<p align="center">
  <img src="/images/코쓱.png" />
</p>

 제출하고 이전기록을 보니 3년 전에 풀었더라.. 그때와 지금 코드가 사뭇 달라 재밌어 여기에 기록해본다 ㅎㅎ


# 3년전 코드

아마 이때는 제대로 못 풀었던 것으로 기억한다. 위에 주석처리된게 내가 DP를 정확히 모르니 나름 recursion이용해서 처음 시도한거고, 메모리나 실행시간에서 터졌을 것 같다.

그 이후 답지 보고, 고민하고, 공부하고, 아래쪽에 코드를 작성한거 같은데, 사실 지금 봐도 아래쪽 코드는 잘 이해가 안된다. 왠지 제대로 이해하지 못하고 답지를 배낀 것 같다 ㅎㅎ

```python
class Solution:
#    def __init__(self):
#        self.min_cost = float('inf')
#        
#    def helper(self, cost, step, cur_cost_sum):
#        if step >= len(cost):
#            self.min_cost = min(cur_cost_sum, self.min_cost)
#        else:
#            self.helper(cost, step+1, cur_cost_sum+cost[step])
#            self.helper(cost, step+2, cur_cost_sum+cost[step])
# 
#    def minCostClimbingStairs(self, cost: List[int]) -> int:
#               
#        self.helper(cost, 0, 0)
#        self.helper(cost, 1, 0)
#        
#        return self.min_cost

    def __init__(self):
        self.dp = dict()
        self.cost = None
        
    def helper(self, n):
        if n < 0:
            return 0
        
        if n not in self.dp:
            self.dp[n] = min(self.helper(n-1)+self.cost[n], self.helper(n-2)+self.cost[n])
        return self.dp[n]    
    
    def minCostClimbingStairs(self, cost: List[int]) -> int:
        self.cost = cost
        return min(self.helper(len(cost)-1), self.helper(len(cost)-2))
```

# 오늘 코드

```python
class Solution:
    def minCostClimbingStairs(self, cost: List[int]) -> int:
        n = len(cost)
        t = []
        
        #t[n] = cost[n] + min(t[n-1], t[n-2])
        
        if n == 2:
            return min(cost)
        
        
        t.append(cost[0])
        t.append(cost[1])
        
        for i in range(2, n):
            t.append(cost[i] + min(t[i-1], t[i-2]))
        
        return min(t[n-1], t[n-2])
```

다시 봐도 오늘 푼 코드가 더 깔끔한 것 같다. 주석 처리된 부분은 유도해낸 점화식이고, 사실상 아래쪽 for문 두 줄이 전부이다.

물론 이 코드는 leetcode 에서 고득점을 먹기 위한 trick 같은건 하지 않아 위의 코드보다 runtime이 길긴 하지만 난 오늘 짠 코드가 더 좋다.

---


DP를 푸는 방법은 top-down, bottom-up 두 가지가 있는데, 예전에 나는 top-down은 직관적이라고 생각하지만, bottom-up은 그렇지 않다고 생각했다. 하지만 DP 공부를 한두시간 해보고, 생각보다 bottom-up이 직관적이라는 생각을 했다. (물론 3년전엔 그렇게 생각 안 했지만..)

이렇게 말하니 왠지 DP 옹호론자(?)가 된 것 같지만, 생각보다 DP문제는 직관적으로 풀 수 있어 너무 재밌었다. 