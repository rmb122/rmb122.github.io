---
title: "LeetCode 刷题记"
date: 2018-01-19 23:25:50
tags: [leetcode, cpp]
---

不想玩游戏, 那就练练算法啥的吧, 也是挺好玩的.

<!-- more -->

1. Two Sum
==========

只会暴力算法, Hash Table学习中.  

Given an array of integers, return **indices** of the two numbers such that they add up to a specific target. You may assume that each input would have **_exactly_** one solution, and you may not use the _same_ element twice.  
**Example:**

Given nums = \[2, 7, 11, 15\], target = 9,

Because nums\[0\] + nums\[1\] = 2 + 7 = 9,
return \[0, 1\].

```cpp
class Solution {//复杂度为o(n²)
public:
    vector<int> twoSum(vector<int>& nums, int target) {
        for(int i=0;i<nums.size();i++)
        {
            for(int j=i+1;j<nums.size();j++)
            {
                if(nums[i]+nums[j]==target)
                {
                    vector<int> temp={i,j};
                    return temp;
                }
            }
        }
    }
};

class Solution {//复杂度为o(n)
public:
   vector<int> twoSum(vector<int>& nums, int target) {
		unordered_map<int,int> aMap;
		int anOtherNum;
		int anOtherNumAdd;
		for (int i = 0; i < nums.size(); ++i) {
			anOtherNum = target - nums[i];
			if (aMap.find(anOtherNum) != aMap.end()) {
				anOtherNumAdd = aMap[anOtherNum];
				vector<int> answer;
				answer.push_back(anOtherNumAdd);
				answer.push_back(i);
				return answer;
			}
			aMap.emplace(nums[i], i);
		}
	}
};
```

 2. Add Two Numbers
===================

You are given two **non-empty** linked lists representing two non-negative integers. The digits are stored in **reverse order** and each of their nodes contain a single digit. Add the two numbers and return it as a linked list. You may assume the two numbers do not contain any leading zero, except the number 0 itself. **Example**

Input: (2 -> 4 -> 3) + (5 -> 6 -> 4)
Output: 7 -> 0 -> 8
Explanation: 342 + 465 = 807.

```cpp
class Solution {
public:
    ListNode* addTwoNumbers(ListNode* l1, ListNode* l2) {
        bool add = false;
		ListNode *now = NULL;
		ListNode *last = new ListNode(0);
		ListNode *first = last;

		while (l1!=NULL || l2!=NULL)
		{
			if (l1 != NULL && l2 != NULL) {
				now = new ListNode(l1->val + l2->val);
				l1 = l1->next;
				l2 = l2->next;
				goto next;
			}
			if (l1 != NULL && l2 == NULL) {
				now = new ListNode(l1->val);
				l1 = l1->next;
				goto next;
			}
			if (l1 == NULL && l2 != NULL) {
				now = new ListNode(l2->val);
				l2 = l2->next;
				goto next;
			}
		next:
			last->next = now;
			if (add) now->val += 1; //进一位
			if (now->val > 9) {
				now->val -= 10; //满10减10同时使add=true, 使下一位进一
				add = true;
			}else{
				add = false;
			}
			if (add && l1 == NULL && l2 == NULL) //满10且两个链表都到头, 新建一位值为1, 应对(5),(5)之类的情况
			{
				last = now;
				now = new ListNode(1);
				last->next = now;
			}else{
				last = now;
			}
			
		}
		return first->next;
    }      
};
```

3. Longest Substring Without Repeating Characters
=================================================

Given a string, find the length of the **longest substring** without repeating characters. **Examples:** Given `"abcabcbb"`, the answer is `"abc"`, which the length is 3. Given `"bbbbb"`, the answer is `"b"`, with the length of 1. Given `"pwwkew"`, the answer is `"wke"`, with the length of 3. Note that the answer must be a **substring**, `"pwke"` is a _subsequence_ and not a substring.  

按照官方说明BF会超时233, 但是神奇的是我这个并没有.大概是C++大法好吧www.  

```cpp
class Solution {
public:
	int lengthOfLongestSubstring(string s) {
		string longest;
		string temp = "";
		for (int i = 0; i < s.size(); ++i) {
			for (; i < s.size(); ++i) {
				if (temp.find(s[i]) == s.npos) {
					temp.push_back(s[i]);
				}else{
					break;
				}
			}//i++
			if (temp.size() > longest.size()) {
				longest = temp;
			}

			if (i != s.size()) {
				i = s.rfind(s[i], i - 1);
			}
			temp.clear();
		}//i++
		return longest.size();
	}
};

class Solution {
public:
	int lengthOfLongestSubstring(string s) {
		unordered_map<char, int> aMap;//记录每个出现的字符最后一次被遍历到的地址
		unordered_map<char, int>::iterator itr;
		int add = -1;
		int longest = -1;
		int length;
		char chr;
		if (s.size() == 0) {
			return 0;
		}

		for (int i = 0; i < s.size(); ++i) {
			chr = s[i];
			itr = aMap.find(chr);

			if (itr == aMap.end()) { //如果这个字符没有被录入, 则将其录入
				aMap.emplace(chr, i);
			}
			else {				
				if (add < itr->second) {
					add = itr->second;//记录出现重复的字符的地址
				}			
				itr->second = i;//录入最后一次被遍历到的地址				
			}

			length = i - add;
			if (length > longest) {
				longest = length;
			}
		}
		return longest;
	}
};
```

4. Median of Two Sorted Arrays
==============================

There are two sorted arrays **nums1** and **nums2** of size m and n respectively. Find the median of the two sorted arrays. The overall run time complexity should be O(log (m+n)).  
**Example 1:**

nums1 = \[1, 3\]
nums2 = \[2\]

The median is 2.0

**Example 2:**

nums1 = [1, 2]
nums2 = [3, 4]

The median is (2 + 3)/2 = 2.5

标准答案的复杂度为o(log(min(m,n))), 然而并不能看懂, 我就贴上我这个o(n)的吧

```cpp
class Solution {
public:
	double findMedianSortedArrays(vector<int>& nums1, vector<int>& nums2) {
		multiset<int> aSet;
		double mid;
		int size;
		for (int i = 0; i < nums1.size(); ++i) {
			aSet.emplace(nums1[i]);
		}
		for (int i = 0; i < nums2.size(); ++i) {
			aSet.emplace(nums2[i]);
		}
		
		size = aSet.size();
		if (size % 2 == 0) {
			auto itr = aSet.begin();
			for (int i = 0; i < size / 2 - 1; ++i) {
				itr++;
			}
			mid = *itr++;
			mid += *itr;
			mid /= 2;
		}
		else {
			auto itr = aSet.begin();
			for (int i = 0; i < static_cast<int> (size / 2); ++i) {
				itr++;
			}
			mid = *itr;
		}
		return mid;
	}
};
```

5. Longest Palindromic Substring
================================

Given a string **s**, find the longest palindromic substring in **s**. You may assume that the maximum length of **s** is 1000. **Example:**

Input: "babad"

Output: "bab"

Note: "aba" is also a valid answer.

**Example:**

Input: "cbbd"

Output: "bb"

其实一般都最先想这种拓展两边的方法把0.0 马拉车算法真的太强了...

```cpp
class Solution {
public:
	string longestPalindrome(string s) {
        if(s.size() == 1)
        {
            return s;
        }
		string longest;
		unsigned int left;
		unsigned int right;
		unsigned int len1;
		unsigned int len2;
		size_t size = s.size();
		for (unsigned int i = 0; i < size; ++i)
		{
			left = i;
			right = i;
			while (left > 0 && right < size - 1 && s[left - 1] == s[right + 1])
			{
				left--;
				right++;
			}
			len1 = right - left + 1;

			left = i;
			right = i;
			if (s[left] == s[right + 1])
			{
				right++;
				while (left > 0 && right < size - 1 && s[left - 1] == s[right + 1])
				{
					left--;
					right++;
				}
			}			
			len2 = right - left + 1;
			if (len1 > len2 && len1 > longest.size())
			{
				longest = s.substr(i - len1 / 2, len1);
			}
			else
			{
				if (len2 > longest.size())
				{
					longest = s.substr(i - len2 / 2 + 1, len2);
				}
			}
		}
		return longest;
	}
};
```
```cpp
//向大佬低头.jpg 
/*
Manacher's 算法实际上相当于Expand Around Center算法的强化版, 
它通过记录每个字符的最长回文数和当前最后的回文位置, 实现了减少
部分重复的运算(如果了解的话会明白为什么是o(n)复杂度, 因为实际上每个字符只会与另一个字符比较一遍)
也就是每个字符只会被执行(str[i + count[i] + 1] == str[i - count[i] - 1])一次, 不过代来的
牺牲就是需要o(n)的空间复杂度, Expand Around Center可是o(1)的.
*/
class Solution {
public:
	string longestPalindrome(string s) {
		string str = "#";
		string longest;
		for (auto iter = s.begin(); iter != s.end(); ++iter)
		{
			str.push_back(*iter);
			str.push_back('#');
		}
		vector<int> count(str.size(), 0);

		int mx = 0;
		int id = 0;
		for (int i = 0; i < str.size(); ++i)
		{
			if (i < mx)
			{
				if (count[2 * id - i] > mx - i)
				{
					count[i] = mx - i;
				}
				else
				{
					count[i] = count[2 * id - i];
				}
			}

			while (i + count[i] < str.size() - 1 && i - count[i] >0 && (str[i + count[i] + 1] == str[i - count[i] - 1]))
			{
				count[i]++;
			}
		
			if (count[i] + i > mx)
			{
				mx = count[i] + i;
				id = i;
			}
		}

		for (int i = 0; i < count.size(); i ++)
		{
			if (count[i] > longest.size())
			{
				longest = s.substr(i / 2 - count[i] / 2, count[i]);
			}
		}
		return longest;
	}
};
```

6. ZigZag Conversion
====================

The string `"PAYPALISHIRING"` is written in a zigzag pattern on a given number of rows like this: (you may want to display this pattern in a fixed font for better legibility)

P   A   H   N
A P L S I I G
Y   I   R

And then read line by line: `"PAHNAPLSIIGYIR"`Write the code that will take a string and make this conversion given a number of rows:

string convert(string text, int nRows);

`convert("PAYPALISHIRING", 3)` should return `"PAHNAPLSIIGYIR"`.

这题感觉挺无脑的, ZigZag到底是啥题目也没说清楚...只要理解了下面这张大佬发在评论区的表就行了.(1315人踩这道题, 是赞的4倍233)

```
/*n=numRows
Δ=2n-2    1                           2n-1                         4n-3
Δ=        2                     2n-2  2n                    4n-4   4n-2
Δ=        3               2n-3        2n+1              4n-5       .
Δ=        .           .               .               .            .
Δ=        .       n+2                 .           3n               .
Δ=        n-1 n+1                     3n-3    3n-1                 5n-5
Δ=2n-2    n                           3n-2                         5n-4
*/
```

```cpp
class Solution {
public:
	string convert(string s, int numRows) {
		if (numRows == 1)
		{
			return s;
		}
		string answer;
		int size = s.size();
		int count;
		int shift;

		count = 0;
		for (int j = 0; (numRows * j - count) < size; j += 2)
		{
			answer.push_back(s[numRows * j - count]);
			count += 2;
		}

		
		if (numRows > 2)
		{
			shift = 2;
			for (int i = 1; i < numRows - 1; ++i)
			{
				count = 0;
				int j = 0;
				while (true)
				{
					if ((numRows * j + i - count) < size)
					{
						answer.push_back(s[numRows * j + i - count]);
						count += 2;
						j += 2;
					}
					else
					{
						break;
					}				
					if ((numRows * j + i - count - shift) < size)
					{
						answer.push_back(s[numRows * j + i - count - shift]);
					}	
					else
					{
						break;
					}				
				}
				shift+=2;
			}
		}

		count = 0;
		for (int j = 0; (numRows * j + numRows - 1 - count) < size; j += 2)
		{
			answer.push_back(s[numRows * j + numRows - 1 - count]);
			count += 2;
		}
		return answer; 
	}
};
```

7. Reverse Integer
==================

Given a 32-bit signed integer, reverse digits of an integer. **Example 1:**

Input: 123
Output:  321

**Example 2:**

Input: -123
Output: -321

**Example 3:**

Input: 120
Output: 21

**Note:** Assume we are dealing with an environment which could only hold integers within the 32-bit signed integer range. For the purpose of this problem, assume that your function returns 0 when the reversed integer overflows. Easy题, 没啥难度0.0, 但是大佬确实是大佬啊, 可以用短很多的代码写出来同样的效果.

```cpp
class Solution {
public:
	int reverse(int x) {
		int result = 0;
		getReverse(x, result);
		if (result == INT_MAX || result == INT_MIN)
		{
			return 0;
		}
		return result;
	}

	int getReverse(int x, int &result)
	{
		int digits;
		int remain = x % 10;
		x -= remain;
		x /= 10;
		if (x == 0)
		{
			result += remain;
			return 1;
		}
		else
		{
			digits = getReverse(x, result);
			result += remain * (pow(10, digits));
			digits++;
			return digits;
		}		
	}
};

class Solution {
public:
    int reverse(int x) {
        long answer = 0;
        while (x != 0) {
            answer = answer * 10 + x % 10;
            if (answer > INT_MAX || answer < INT_MIN) return 0;
            x /= 10;
        }
        return (int)answer;
    }
};
```

8. String to Integer (atoi)
===========================

Implement `atoi` to convert a string to an integer. **Hint:** Carefully consider all possible input cases. If you want a challenge, please do not see below and ask yourself what are the possible input cases. **Notes:** It is intended for this problem to be specified vaguely (ie, no given input specs). You are responsible to gather all the input requirements up front. **Requirements for atoi:** The function first discards as many whitespace characters as necessary until the first non-whitespace character is found. Then, starting from this character, takes an optional initial plus or minus sign followed by as many numerical digits as possible, and interprets them as a numerical value. The string can contain additional characters after those that form the integral number, which are ignored and have no effect on the behavior of this function. If the first sequence of non-whitespace characters in str is not a valid integral number, or if no such sequence exists because either str is empty or it contains only whitespace characters, no conversion is performed. If no valid conversion could be performed, a zero value is returned. If the correct value is out of the range of representable values, INT\_MAX (2147483647) or INT\_MIN (-2147483648) is returned.  

我只能说那么多踩是有原因的...题目看上去很长实际上该说的都没有说, 比如说指数, 又比如+-1是否正确等等.

```cpp
class Solution {
public:
	int toNum(char chr) {
		if (chr <= 57 && chr >= 48)
		{
			return chr - 48;
		}
		return -1;
	}

	int myAtoi(string str) {
		bool isNeg = false;
		bool haveSym = false;
		long long value = 0;

		auto itr = str.begin();
		while (true)
		{
			if (*itr == ' ')
			{
				++itr;
				continue;
			}
			else
			{
				break;
			}
		}

		if (\*itr == '-' || \*itr == '+')
		{
			if (*itr == '-') 
			{
				isNeg = true;
			}
			haveSym = true;
			itr++;
		}

		if (haveSym == true && (\*itr == '+' || \*itr == '-'))
		{
			return 0;
		}

		while (itr != str.end() && *itr != ' ')
		{
			int num = toNum(*itr);
			if (num == -1)
			{
				break;
			}
			if (isNeg)
			{
				value *= 10;
				value -= num;
			}
			else
			{
				value *= 10;
				value += num;
			}
			itr++;

			if (value > INT_MAX)
			{
				return INT_MAX;
			}
			if (value < INT_MIN)
			{
				return INT_MIN;
			}
		}

		if (value > INT_MAX)
		{
			return INT_MAX;
		}
		if (value < INT_MIN)
		{
			return INT_MIN;
		}
		return value;
	}
};
```

9. Palindrome Number
====================

Determine whether an integer is a palindrome. Do this without extra space.

**Some hints:**Could negative integers be palindromes? (ie, -1) If you are thinking of converting the integer to string, note the restriction of using extra space. You could also try reversing an integer. However, if you have solved the problem "Reverse Integer", you know that the reversed integer might overflow. How would you handle such case? There is a more generic way of solving this problem.

```cpp
class Solution {
public:
	bool isPalindrome(int x) {
		if (x < 0)
		{
			return false;
		}
		int reverse = 0;
		int copy = x;
		while (x >= 1)
		{
			reverse *= 10;
			int remain = (x % 10);
			reverse += remain;
			x -= remain;
			x /= 10;
		}
		return reverse == copy;
	}
};
```

10. Regular Expression Matching
===============================

Implement regular expression matching with support for `'.'` and `'*'`.

'.' Matches any single character.
'*' Matches zero or more of the preceding element.

The matching should cover the entire input string (not partial).

The function prototype should be:
bool isMatch(const char \*s, const char \*p)

Some examples:
isMatch("aa","a") → false
isMatch("aa","aa") → true
isMatch("aaa","aa") → false
isMatch("aa", "a*") → true
isMatch("aa", ".*") → true
isMatch("ab", ".*") → true
isMatch("aab", "c\*a\*b") → true

这道题为hard, 确实很有难度, 不看答案没有想到递归这个骚操作233 这里用迭代器就很方便, 省去了传参时的浪费.

```cpp
class Solution {
public:	
	string::iterator textEnd;
	string::iterator regEnd;

	bool isMatch(string s, string p) {
		textEnd = s.end();
		regEnd = p.end();
		return match(s.begin(), p.begin());
	}

	bool match(string::iterator text, string::iterator reg) {
		if (reg == regEnd) {
			return text == textEnd;
		}
		bool charMatch = false;
		if (text != textEnd && (\*reg == \*text || *reg == '.')) {
			charMatch = true;
		}
		if (regEnd - reg >= 2 && *(reg + 1) == '*') {
			return match(text, reg + 2) || (charMatch && match(text + 1, reg));
		}
		else{
			return charMatch && match(text + 1, reg + 1);
		}
	}
};
```

你还可以加上缓存, 避免重复的运算.加上后提升还是非常大的.

```cpp
class Solution {
public:	
	string::iterator textEnd;
	string::iterator regEnd;

	bool isMatch(string s, string p) {
		textEnd = s.end();
		regEnd = p.end();
		vector<vector<int>> cache(s.size() + 1, vector<int>(p.size() + 1, -1));
		return match(s.begin(), p.begin(), cache);
	}

	bool match(string::iterator text, string::iterator reg , vector<vector<int>> &cache) {
		int textIndex = textEnd - text;
		int regIndex = regEnd - reg;

		if (cache[textIndex][regIndex] != -1) {
			return cache[textIndex][regIndex];
		}
		if (reg == regEnd) {
			return text == textEnd;
		}
		bool charMatch = false;
		if (text != textEnd && (\*reg == \*text || *reg == '.')) {
			charMatch = true;
		}
		if (regEnd - reg >= 2 && *(reg + 1) == '*') {
			bool temp = match(text, reg + 2, cache) || (charMatch && match(text + 1, reg, cache));
			cache[textIndex][regIndex] = temp;
			return temp;
		}
		else{
			bool temp = charMatch && match(text + 1, reg + 1 ,cache);
			cache[textIndex][regIndex] = temp;
			return temp;
		}
	}
};
```