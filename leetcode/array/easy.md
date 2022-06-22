---
title: 数组-简单题
comments: true
---



虽然有些不是最优解，大体上还是挺让我惊讶的，竟然做出来了..



<!--more-->

| 最近提交时间 | 题目                                                         | 题目难度 | 提交次数 |
| :----------- | :----------------------------------------------------------- | :------- | :------- |
| 2022-5-30    | [#66 加一](https://leetcode.cn/problems/plus-one/)           | 简单     | 4 次     |
| 2022-5-30    | [#35 搜索插入位置](https://leetcode.cn/problems/search-insert-position/) | 简单     | 7 次     |
| 2022-5-30    | [#27 移除元素](https://leetcode.cn/problems/remove-element/) | 简单     | 5 次     |
| 2022-5-30    | [#26 删除有序数组中的重复项](https://leetcode.cn/problems/remove-duplicates-from-sorted-array/) | 简单     | 1 次     |
| 2022-5-31    | [#88 合并两个有序数组](https://leetcode.cn/problems/merge-sorted-array/) | 简单     | 3 次     |
| 2022-6-01    | [#29 有效的括号](https://leetcode.cn/problems/valid-parentheses/solution/you-xiao-de-gua-hao-by-leetcode-solution/) | 简单     | 1 次     |
| 2022-6-01    | [#9 回文数](https://leetcode.cn/problems/palindrome-number/) | 简单     | 5 次     |



今天搞了四道关于数组的简单题，过程上都是大差不差，很难一次性把答案写的完美，都需要调整一下亦或是语法写错了...



印象比较深刻的是 27 和 66 花的时间稍微久一些，很多情况第一时间没有考虑到导致提交次数比较多。还有一个使用了 二分算法的，在判断取 start 还是 end 的时候比较纠结，但是后来还是巧妙地化解了。



[#66 加一](https://leetcode.cn/problems/plus-one/)

```go
func plusOne(digits []int) []int {
    // 直接对最后一位进行加一操作
    digits[len(digits)-1]++

    // 处理进位的情况
    for i := len(digits) - 1; i >= 0; i-- {
        // 因为 0 对 任何数取余都是 0 所以要排除 0 的情况
        // 这也导致我多提交一次..尬
        if digits[i]!=0 && digits[i]%10 == 0 {
            digits[i] = 0
            if i >= 1 {
                digits[i-1]++
            } else {
                res := []int{1}
                for j := 0; j < len(digits); j++ {
                    res = append(res, 0)
                }
                return res
            }
        }
    }

    return digits
}
```



[#35 搜索插入位置](https://leetcode.cn/problems/search-insert-position/)

```go
func searchInsert(nums []int, target int) int {
    start := 0
    end := len(nums) - 1

    // 直接用两个 if 处理越界的情况
    if nums[end] < target {
        return len(nums)
    }

    if nums[start] >= target {
        return 0
    }

    var midIndex int

    // 二分的方式进行处理
    for {
        midIndex = (start + end) / 2

        if nums[midIndex] >= target {
            end = midIndex
        } else {
            start = midIndex
        }


        if start == end-1 {
            if nums[start] <= target {
                return end
            } else {
                return start
            }
        }
    }
}
```



[#27 移除元素](https://leetcode.cn/problems/remove-element/)

```go
// 我写的 n^2 的时间复杂度
func removeElement(nums []int, val int) int {
    for index := 0; index < len(nums); index++ {
        if nums[index] == val {
            for i := index+1; i < len(nums); i ++ {
                nums[i - 1] = nums[i]
            }
            index--
            nums = nums[:len(nums)-1]
        }
    }
    return len(nums)
}

// 人家这个 O(N) 的时间复杂度
func removeElement(nums []int, val int) int {
    left := 0
    for _, v := range nums { // v 即 nums[right]
        if v != val {
            nums[left] = v
            left++
        }
    }
    return left
}
```



[#26 删除有序数组中的重复项](https://leetcode.cn/problems/remove-duplicates-from-sorted-array/)

```go
func removeDuplicates(nums []int) int {
    for index, num := range nums {
        // 找出最后一个重复数字的索引，然后保留一个， 其余的切掉
        last := 0
        for i := index+1; i < len(nums) && num == nums[i]; i++ {
            last = i
        }
        if last != 0 {
            nums = append(nums[:index], nums[last:]...)
        }
    }
    return len(nums)
}
```



暴力解法没什么不好的，先写出来，再考虑优化的事情，一个好的算法必须要经历千百次的调试。（我总结的hh）



[#88 合并两个有序数组](https://leetcode.cn/problems/merge-sorted-array/)  

这道题我用了时间复杂度为 O(n) 的，题目中给定了两个数组，nums1 的大小就是合并之后的大小...这个让我有点懵逼。

```go
func merge(nums1 []int, m int, nums2 []int, n int) {
    var data []int
	var i, j int
	for i < m && j < n {
		if nums1[i] <= nums2[j] {
			data = append(data, nums1[i])
			i++
		} else {
			data = append(data, nums2[j])
			j++
		}
	}

	if i != m {
		data = append(data, nums1[i:]...)
	}

	if j != n {
		data = append(data, nums2[j:]...)
	}
    
}
```



[#9 回文数](https://leetcode.cn/problems/palindrome-number/)  

这个是最初版，表面上看起来没啥问题，也能通过一部分测试用例，但是，这里判断数字位数有一个 bug，比如 101 这种数字，算出来的位数是 1。不符合预期。。

```go
func isPalindrome(x int) bool {
    // 如题目描述的，负数不属于回文数，直接处理即可
    if x < 0 {
        return false
    }

    // 计算这个数字的位数
    var count int
    temp := x
    for temp % 10 != 0 {
        count++
        temp /= 100
    }

    // 倒序 然后进行比较
    var result int
    temp1 := x
    for temp1 > 0 {
        data := temp1 % 10
        result += data * int(math.Pow(float64(10), float64(count-1)))
        temp1 /= 10
        count--
    }
    if result == x {
        return true
    }

    return false
}
```



为了解决上述计算位数的问题，我将这个数字转换成字符串，但是后续的算法没有改变，这里试试使用字符串进行判断的方式，尝试一下，（先写 blog 再尝试）。即，将算位数的地方修改成如下的形式：

`  dig := utf8.RuneCountInString(strconv.Itoa(x))`



下边这个是直接转换成字符串的方式进行处理：

> 竟然一次通过了，unbelievable..

```go
func isPalindrome(x int) bool {
    if x < 0 {
        return false
    }

    data := strconv.Itoa(x)
    count := utf8.RuneCountInString(data)

    for first, last := 0, count-1; first <= count-1 && last >= 0; {
        if data[first] != data[last] {
            return false
        }
        // go 中不支持 这种写法 for ; ; xx++, xx++
        first++
        last--
    }
    return true
}
```



  [#29 有效的括号](https://leetcode.cn/problems/valid-parentheses/solution/you-xiao-de-gua-hao-by-leetcode-solution/) 

有效的括号，这里没搞懂啥意思啊，`([)]` 这样是不行的吗？一开始没读懂题的意思，理解成了这个字符串中的括号是否成对，如下：

```go
func isValid(s string) bool {
	var one []int32
	var two []int32
	var three []int32

	for _, val := range s {
		switch val {
		case '{':
			one = append(one, val)
		case '}':
			if len(one) != 0 {
				one = one[:len(one)-1]
			} else {
				return false
			}
		case '[':
			two = append(two, val)
		case ']':
			if len(two) != 0 {
				two = two[:len(two)-1]
			} else {
				return false
			}
		case '(':
			three = append(three, val)
		case ')':
			if len(three) != 0 {
				three = three[:len(three)-1]
			} else {
				return false
			}
		}
	}

	if len(one) == 0 && len(two) == 0 && len(three) == 0 {
		return true
	}
	return false
}
```

采用了三个栈的方式求解，相当的暴力hh，后来看了别人的解答才明白这个到底是什么意思，如 `([)]` 这种形式，就是说第一个右括号匹配最后一个左括号，这时候是 `[)` 应该给人家返回 false，如下：

```go
func isValid1(s string) bool {
	var one []int32

	for _, val := range s {
		switch val {
		case '{', '[', '(':
			one = append(one, val)
		case '}', ']', ')':
			if len(one) != 0 {
				data := one[len(one)-1]
				if data+2 == val || data+1 == val {
					one = one[:len(one)-1]
				} else {
					return false
				}
			} else {
				return false
			}

		}
	}

	if len(one) == 0 {
		return true
	}

	return false
}
```

果然啊，人家要的就是这种方式，其实这里考察的是**栈**..

