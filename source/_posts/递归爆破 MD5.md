---
title: "递归爆破 MD5"
date: 2018-01-06 01:14:38
tags: [python, writeup]
---

做到一题要用爆破的, 刚刚好拿来练练递归. 题目如下:

> python大法好！ 这里有一段丢失的md5密文 e9032???da???08????911513?0???a2 要求你还原出他并且加上nctf{}提交 已知线索 明文为： TASC?O3RJMV?WDJKX?ZM   
题目来源：安恒杯

<!-- more -->

```python
import _md5
import sys

sys.setrecursionlimit(1000000) #增加递归的上限

table=''
for i in range(0,26):
    table+=chr((ord('a')+i))
    table+=chr((ord('A')+i))

for i in range(0,10):
    table+=chr((ord('0')+i)) #构造字符表a-zA-Z0-9

part1='TASC'
part2='O3RJMV'
part3='WDJKX'
part4='ZM'

def test(astr):
    flag=part1+astr[0]+part2+astr[1]+part3+astr[2]+part4
    md5Processer=_md5.md5()
    md5Processer.update(flag.encode())
    md5=md5Processer.hexdigest()

    if md5[0:5] =='e9032':
        print(flag)
        print(md5)

def bf(i,astr): 
    for char in table:    
        temp=astr 
        if i==0:
           test(temp)        
           break     
        temp+=char
        bf(i-1,temp)

       
bf(3,'')
```

不一会就爆破出来了, 明文是TASCJO3RJMVKWDJKXLZM 算出MD5加上格式 `flag：nctf{e9032994dabac08080091151380478a2}`