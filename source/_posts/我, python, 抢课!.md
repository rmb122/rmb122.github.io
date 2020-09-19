---
title: "我, python, 抢课!"
date: 2017-12-29 19:41:53
tags: [python, misc]
---

坑爹的选课系统. 人品不够, 技术来凑233 之前一直用的 selenium. 第一次使用 python 自带的 urllib, 顺便网上的教程真的是太乱了... 还是得靠自己摸索. 

<!-- more -->

```python
from urllib import request
from urllib import parse
from http.cookiejar import MozillaCookieJar
import json 
import _sha1
import time
import os

studentID=11111 #, 
turnID=163 #这一次是163
courseList={252504,252509} #在这里添加课程ID
YourUsername='Here is your username'
YourPassword='Here is your password'

def getOpener(cookieData):
    opener=request.build_opener()
 
    Host=list()
    Host.append('Host')
    Host.append('jxglstu.hfut.edu.cn')
 
    UA=list()
    UA.append('User-Agent')
    UA.append('Mozilla/5.0 (X11; Linux x86_64; rv:52.0) Gecko/20100101 Firefox/52.0')
 
    Referer=list()
    Referer.append('Referer')
    Referer.append('http://jxglstu.hfut.edu.cn/eams5-student/for-std/course-select/'+str(studentID)+'/turn/'+str(turnID)+'/select')
 
    Cookie=list()
    Cookie.append('Cookie')
    Cookie.append(cookieData)
 
    opener.addheaders.clear()
    opener.addheaders.append(Host)
    opener.addheaders.append(UA)
    opener.addheaders.append(Referer)
    opener.addheaders.append(Cookie) #构造HEADER
    return opener
 
def getRequestID(lessonID):
    request.urlopen('http://jxglstu.hfut.edu.cn/eams5-student/for-std/course-select') #必须GET一下这个, 不然后台还没注册这个SESSION, 导致HTTP500
 
    postData={'studentAssoc':studentID,'lessonAssoc':lessonID,'courseSelectTurnAssoc':turnID,'scheduleGroupAssoc':'','virtualCost':'0'}
    postData=parse.urlencode(postData).encode(encoding='UTF8')
    response=request.urlopen('http://jxglstu.hfut.edu.cn/eams5-student/ws/for-std/course-select/add-request',postData)
    requestID=response.read()
 
    return requestID
 
def charge(requestID):
    postData={'studentId':studentID,'requestId':requestID}
    postData=parse.urlencode(postData).encode(encoding='UTF8')
 
    response=request.urlopen('http://jxglstu.hfut.edu.cn/eams5-student/ws/for-std/course-select/add-drop-response',postData)
    result=response.read()
    result=result.decode()
    print(result)
 
 
def getCookie(username,password): 
    aCookieJar=MozillaCookieJar()
    handler=request.HTTPCookieProcessor(aCookieJar)
    opener=request.build_opener(handler)
 
    UA=list()
    UA.append('User-Agent')
    UA.append('Mozilla/5.0 (X11; Linux x86_64; rv:52.0) Gecko/20100101 Firefox/52.0')
 
    opener.addheaders.clear()
    opener.addheaders.append(UA)  #构造HEADER
 
    try:
        response=opener.open('http://jxglstu.hfut.edu.cn/eams5-student/login-salt') #得到盐
        salt=response.read().decode('UTF8')
    except: #服务器有时候会404 应该是防DDOS
        print("服务器大姨妈来啦, 等一下再试")
        os.system("pause")
 
    salt=salt+'-'+password
    sha1Processer=_sha1.sha1()
    sha1Processer.update(salt.encode())
    password=sha1Processer.hexdigest()  #密码加盐 
 
    postData={'username':username,'password':password,'captcha':''}
    postData=json.dumps(postData,separators=(',',':')).encode(encoding='utf-8') #将密码用户名转换为JSON
    
    header={'Content-Type':'application/json'}
    opener.open(request.Request("http://jxglstu.hfut.edu.cn/eams5-student/login", headers=header),postData) #必须自己构造一个Request, 不然因为没有Content-Type会被自动替换成Urlencode
   
    cookie=''
    for cookieData in aCookieJar:
        cookie=cookie+cookieData.name+'='+cookieData.value+';' #构造cookie
 
    cookie=cookie[0:len(cookie)-1] #删掉最后一个;号
    return cookie
    
 
opener=getOpener(getCookie(YourUsername,YourPassword))
request.install_opener(opener) #替代request的opener
 
while True:   
    for courseNumber in courseList:
        print('Now in ',courseNumber)
        charge(getRequestID(courseNumber))
        time.sleep(0.1)
```