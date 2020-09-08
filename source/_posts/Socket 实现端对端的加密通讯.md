---
title: "Socket 实现端对端的加密通讯"
date: 2018-01-11 17:43:16
tags: [python, misc]
---

对这方面比较感兴趣, 写了一个小DEMO吧.只是勉强能用而已. 可以加上断线等待重连和通过RSA加密传输AES密钥来实现更快的加密.

<!-- more -->

```python
import socket
import rsa
import threading
from time import sleep

mySendAddress=('0.0.0.0',0)
myListenAddress=('0.0.0.0.',0)
hisAddress=('0.0.0.0',0)

pubKey,priKey=rsa.newkeys(1024)
pubKeyBin=pubKey.save_pkcs1()
hisPubKey=rsa.PublicKey(1,1)

def startListen():
    global hisPubKey
    sck=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
    sck.bind(myListenAddress)
    sck.listen()
    con,add=sck.accept()
    hisPubKeyBin=con.recv(2048) #得到对方的公钥
    hisPubKey=rsa.PublicKey.load_pkcs1(hisPubKeyBin)
    while True:             
        data=con.recv(2048)
        data=rsa.decrypt(data,priKey) #用自己的私钥解密
        print('>>Get a message from',add[0])
        print('>>>Data:',data.decode(),end='\n>')

def sentMessage(address,text,sck):    
    global hisPubKey
    text=rsa.encrypt(text.encode(),hisPubKey) #用对方的公钥加密信息
    try:
        sck.sendall(text)
    except ConnectionRefusedError:
        print('>>>He is now online yet')   

t1=threading.Thread(target=startListen)
t1.start()

sck_send=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
sck_send.bind(mySendAddress)

print('Wait for his response')
while True:
    try:
        sck_send.connect(hisAddress) #如果连接成功退出循环
        sck_send.sendall(pubKeyBin) #送出自己的公钥        
        print('He is online')        
        break
    except ConnectionRefusedError:
        sleep(0.5)
        pass

while True:
    text=input('>') 
    sentMessage(hisAddress,text,sck_send)
```