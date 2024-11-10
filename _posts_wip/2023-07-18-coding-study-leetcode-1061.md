---
title: 오랜만의 코딩 스터디(WIP)
categories: coding study
tags:
  - coding
  - study
  - leetcode
toc: true
toc_sticky: true
published: false
---

https://leetcode.com/problems/lexicographically-smallest-equivalent-string/description/


```python
class Solution:
    def smallestEquivalentString(self, s1: str, s2: str, baseStr: str) -> str:

        d = dict()
        r_d = dict()
        small = dict()
        group_id = 0

        for a, b in zip(s1, s2):
            if a == b:
                continue

            s, l = (a, b) if a < b else (b, a)

            # if both are not in d
            # make a new group and make s as a smallest one
            if s not in d and l not in d:
                d[s] = d[l] = group_id
                r_d[group_id] = [s, l]
                small[group_id] = s
                group_id += 1
            # if l not in d but s in d
            # put l to s's group
            elif s in d and l not in d:
                d[l] = d[s]
                r_d[d[s]].append(l)
            # if l in d but not s in d
            # add s to l's group and compare s with l's small one and update
            elif s not in d and l in d:
                d[s] = d[l]
                r_d[d[l]].append(s)
                if s < small[d[s]]:
                    small[d[s]] = s
            # if l in d and s in d
            # combine two groups if need, and compare their smallest ones
            # otherwise, do nothing
            else:
                if d[s] != d[l]:
                    # combine
                    if small[d[s]] < small[d[l]]:
                        s_group_id, l_group_id = d[s], d[l]
                    else:
                        s_group_id, l_group_id = d[l], d[s]
                        
                    for n in r_d[l_group_id]: # get l's group's numbers
                        d[n] = s_group_id
                    r_d[s_group_id] += r_d[l_group_id]
                    #del r_d[l_group_id]
                else:
                    continue

        ans = ""
        for c in baseStr:
            if c in d:
                ans += small[d[c]]
            else:
                ans += c

#        print(ans)
        return ans
```