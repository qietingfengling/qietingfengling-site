---
title: "深入理解KMP字符串模式匹配算法"
date: 2018-03-21T09:32:45+08:00
subtitle: ""
tags: [go,algorithm]
---
# 背景
冲着KMP名字来研究的算法

# 使用场景
在一个长字符串中定位一个子字符串的高效算法

# 变量定义

S:原字符串；

subS:待定位子字符串;

next:移动规则，len(next)==len(subS);

j:subS的下标;

k:大于等于1小于j的下标;

i: S的下标

# 逻辑理解

对于子字符串subS，当S匹配到某个位置`S[i]`和`subS[k]`不相等的时候，
对于subS，我们需要知道一套规则来指示我们S中的i需要向右移动多少位，
subS中的j需要向左移动多少位。

算法的主要思想是让subS中的j向左移动的位数尽可能少，
当`subS[:k+1] == subS[j-1-k:j]`时候，subS只需要从k+1位开始匹配。



# 公式理解

1. `next[0]=-1`,表示任何subS的第一个字符串的移动规则都是-1
2. `next[j]=-1`,表示 `subS[0]==subS[j]&&(subS[:k+1]!=subS[j-1-k:j] || subS[k]==subS[j]) `
3. `next[j]=k`, 表示 `subS[:k+1] == subS[j-1-k:j] && subS[k+1]!=subS[j]`
4. `next[j]=0`,不满足1、2、3三个条件的其他所有情况。

以上各种规则转化为字符串S的下标i移动位数和子字符串subS的下标j的移动位数分别如下：

1. `-1` i加1，既S向后移动1位开始匹配；j恢复到0，既subS从头开始匹配；
2. `k` i不变，既S从上一次匹配的位置开始匹配；j恢复到k，既subS从第k位开始匹配；
3. `0` i不变，既S从上一次匹配的位置开始匹配；j恢复到0，既subS从头开始匹配；

# 实例
`subS=="ababa"`

| 下标 | 0 | 1 | 2 | 3 | 4 |
|:--------|:---------:|:-------:|:-------:|:-------:|:-------:|
| subS | a| b | a | b | a |
| next | -1| 0 | -1 | 0 | -1 |
| 满足公式 | 1| 4 | 2 | 4 | 2 |

`subS=="acaaaacabbbb"`

| 下标 | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 |
|:--------|:---------:|:-------:|:-------:|:-------:|:-------:|:---------:|:-------:|:-------:|:-------:|:-------:|:-------:|:-------:|
| subS | a| c | a | a | a | a| c | a | b | b | b | b |
| next | -1| 0 | -1 | 1 | 1 | 1 | 0 | -1 | 3 | 0 | 0 | 0 |
| 满足公式 | 1| 4 | 2 | 3 | 3 | 3| 4 | 2  | 3 | 4 | 4 | 4 |



# 代码实现
```go
package KMP

type Kmp struct {
	subS string
	next []int
}

func New(subS string) *Kmp {
	kmp := &Kmp{
		subS: subS,
	}
	kmp.next = make([]int, len(subS))
	kmp.getNext()
	return kmp
}

//get next rule
func (kmp *Kmp) getNext() {
	j := 0
	k := -1
	kmp.next[0] = -1
	subSLen := len(kmp.subS)
	for j < subSLen-1 {
		if k == -1 || kmp.subS[j] == kmp.subS[k] {
			j += 1
			k += 1
			if kmp.subS[j] != kmp.subS[k] {
				kmp.next[j] = k
			} else {
				kmp.next[j] = kmp.next[k]
			}
		} else {
			k = kmp.next[k]
		}
	}
}

func (kmp *Kmp) KmpContains(s string) bool {
	return kmp.KmpGetIndex(s) > -1
}

// kmp search
func (kmp *Kmp) KmpGetIndex(s string) int {
	sLen := len(s)
	subSLen := len(kmp.subS)
	index := 0
	i := 0
	j := 0
	for i < sLen && j < subSLen {
		if s[i] == kmp.subS[j] {
			i += 1
			j += 1
		} else {
			index += j - kmp.next[j]
			if kmp.next[j] != -1 {
				j = kmp.next[j]
			} else {
				j = 0
				i += 1
			}
		}
	}
	if j == subSLen {
		return index
	} else {
		return -1
	}

}
```

