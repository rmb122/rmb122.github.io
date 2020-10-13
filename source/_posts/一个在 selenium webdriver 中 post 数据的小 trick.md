---
title: "一个在 selenium webdriver 中 post 数据的小 trick"
date: 2017-12-16 12:49:13
tags: [misc, python]
---

在爬英语听力答案中遇到需要 post 数据的需求, 而 webdriver 原生并没有提供 post 数据的功能. 不过想了想, 可以通过 JavaScript 来构造一个虚拟表单然后发送就行了.  语句如下 网上可以找到

<!-- more -->

```python
js='''
   function post(URL, PARAMS) {        
   var temp = document.createElement("form");        
   temp.action = URL;        
   temp.method = "post";        
   temp.style.display = "none";        
   for (var x in PARAMS) {        
   var opt = document.createElement("textarea");        
   opt.name = x;        
   opt.value = PARAMS[x];           
   temp.appendChild(opt);        
   }        
   document.body.appendChild(temp);        
   temp.submit();        
   return temp;        
   }            
   //调用方法      
   post(URL,{'xxx1':'xxx2','xxx3':'xxx4'});
   '''
driver.execute_script(js)
```