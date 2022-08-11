---
title: Codeql 踩坑记录
date: 2020-03-30 23:24:40
tags: [codeql]
---

## 安装

入口: https://help.semmle.com/codeql/codeql-cli/procedures/get-started.html  
下载地址: https://github.com/github/codeql-cli-binaries/releases  
license: https://securitylab.github.com/tools/codeql/license  

<!--more-->

license 里面有写:  

Further, except (and only to the extent) permitted by applicable law or applicable third-party license, you will not (and have no right to):

work around any technical limitations in the Software that only allow you to use it in certain ways;
**reverse engineer**, decompile or disassemble the Software;
remove, minimize, block, or modify any notices of GitHub or its suppliers in the Software;
use the Software in any way that is against the law; or
share, publish, distribute or lend the Software, provide or make available the Software as a hosted solution (whether on a standalone basis or combined, incorporated or integrated with other software or services) for others to use, or transfer the Software or these Terms to any third party.

GitHub CodeQL can only be used on codebases that are released under an OSI-approved open source license, or to perform academic research. It can't be used to generate CodeQL databases for or during automated analysis, continuous integration or continuous delivery, whether as part of normal software engineering processes or otherwise. For these uses, contact the sales team.

大概意思就是: 主程序是闭源的, 但是检测规则是开源的, 其中一些 [extractor](https://github.com/github/codeql-go) 也开源了, 当然其实 java 写的未混淆, 跟开源差不多.  
用于学术研究或者检测开源项目都随便使用, 但是拿来商业等操作, 就需要购买商业 license.  

主程序就在 tools/codeql.jar, 也没做混淆啥的. 还是挺良心的.  
安装包里面自带了运行环境和支持语言的 extractor (甚至自带 java 运行环境), 开箱即用.  

## 测试

先写个测试用例看看
```python
# $ cat flask-example/app.py
import flask
import subprocess

app = flask.Flask(__name__)

@app.route('/')
def index():
    return subprocess.check_output(flask.request.args.get('c', 'ls'))

app.run()
```

建立 database  
```sh
/home/rmb122/repos/codeql/codeql database create codeql-database --language=python --source-root flask-example
```

然后就喜闻乐见的出错了 233
```
[2020-03-30 17:45:00] [build] [INFO] [8] Extracted module _lzma in 22ms
[2020-03-30 17:45:01] [build] [INFO] [3] Extracted file /usr/lib/python3.8/site-packages/cryptography/hazmat/backends/openssl/dsa.py in 765ms
[2020-03-30 17:45:01] [build] [INFO] [5] Extracted file /usr/lib/python3.8/site-packages/cryptography/hazmat/primitives/serialization/ssh.py in 404ms
[2020-03-30 17:45:01] [build] [INFO] [1] Extracted file /usr/lib/python3.8/site-packages/cryptography/hazmat/backends/openssl/encode_asn1.py in 1572ms
[2020-03-30 17:45:01] [build] [ERROR] [1] Failed to extract module _decimal: libmpdec.so.2: cannot open shared object file: No such file or directory
[2020-03-30 17:45:01] [build] [TRACEBACK] [1] "semmle/worker.py", line 220, in _extract_loop
[2020-03-30 17:45:01] [build] [TRACEBACK] [1] "semmle/extractors/super_extractor.py", line 47, in process
[2020-03-30 17:45:01] [build] [TRACEBACK] [1] "semmle/extractors/builtin_extractor.py", line 14, in process
[2020-03-30 17:45:01] [build] [INFO] [5] Extracted module _io in 50ms
[2020-03-30 17:45:01] [build] [INFO] [8] Extracted file /usr/lib/python3.8/site-packages/cryptography/hazmat/primitives/asymmetric/dh.py in 373ms
[2020-03-30 17:45:01] [build] [INFO] [1] Extracted file /usr/lib/python3.8/site-packages/werkzeug/wrappers/cors.py in 85ms
[2020-03-30 17:45:01] [build] [INFO] [7] Extracted file /usr/lib/python3.8/argparse.py in 5850ms
[2020-03-30 17:45:01] [build] [INFO] [2] Extracted file /usr/lib/python3.8/tarfile.py in 5602ms
[2020-03-30 17:45:01] [build] [INFO] [3] Extracted file /usr/lib/python3.8/site-packages/werkzeug/debug/repr.py in 611ms
[2020-03-30 17:45:01] [build] [INFO] [4] Extracted file /usr/lib/python3.8/site-packages/jinja2/parser.py in 2626ms
[2020-03-30 17:45:03] [build] [ERROR] Process 6 timed out
[2020-03-30 17:45:04] [build-err] Traceback (most recent call last):
[2020-03-30 17:45:04] [build-err]   File "/home/rmb122/repos/codeql/python/tools/index.py", line 19, in <module>
[2020-03-30 17:45:04] [build-err]     buildtools.index.main()
[2020-03-30 17:45:04] [build-err]   File "/home/rmb122/repos/codeql/python/tools/python3src.zip/buildtools/index.py", line 110, in main
[2020-03-30 17:45:04] [build-err]   File "/usr/lib/python3.8/subprocess.py", line 364, in check_call
[2020-03-30 17:45:04] [build-err]     raise CalledProcessError(retcode, cmd)
[2020-03-30 17:45:04] [build-err] subprocess.CalledProcessError: Command '['python3', '-S', '/home/rmb122/repos/codeql/python/tools/python_tracer.py', '-v', '-z', 'all', '-c', '/home/rmb122/temp/codeql/codeql-database/working/trap_cache', '-p', '/usr/lib/python3.8/site-packages', '-R', '/home/rmb122/temp/codeql/flask-example']' returned non-zero exit status 1.
[2020-03-30 17:45:04] [ERROR] Spawned process exited abnormally (code 1; tried to run: [/home/rmb122/repos/codeql/python/tools/autobuild.sh])
A fatal error occurred: Exit status 1 from command: [/home/rmb122/repos/codeql/python/tools/autobuild.sh]
```

问题出在这  
Failed to extract module _decimal: libmpdec.so.2: cannot open shared object file: No such file or directory  

repl 里面运行一下, 果然也是如此  
```
$ python
Python 3.8.2 (default, Feb 26 2020, 22:21:03) 
[GCC 9.2.1 20200130] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import _decimal
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
ImportError: libmpdec.so.2: cannot open shared object file: No such file or directory
>>> 
```

神奇的居然没有装, pacman -S mpdecimal 装一个就完事了.  
这应该只扫了用到的库, 没有扫全部的库, 挺快的, 点个赞.  

接下来真正开始写 ql  
教程在这:  
https://help.semmle.com/QL/learn-ql/  
https://help.semmle.com/QL/ql-handbook/  
https://help.semmle.com/qldoc/  

官方的规则库:  
https://github.com/Semmle/ql.git  
学习一哈.  

官方自带的 ql 里面有预先定义好一些数据结构, 直接 import python 就能用了, .qll 是 library, .ql 是真正的查询语句.  
正式运行还需要定义一个 qlpack.yml, 意义跟 pom.xml 差不多吧, 定义包名和依赖  
https://help.semmle.com/codeql/codeql-cli/reference/qlpack-overview.html  

```yml
name: com-rmb122-test
version: 0.0.1
libraryPathDependencies: codeql-python
extractor: python
```

在 `$HOME/.config/codeql/config` 里面设置 ql repo 的路径, 这样才能被 import, 感觉现在这种手动配置好蛋痛 =.= 感觉跟自己编译 cpp 一样链接一堆库  
而且文档 u1s1, 挺乱的, 不过刚被 github 收购, 可以理解, 希望之后能更方便一点.  
```
--search-path <path to ql repo>
```
注意不要写成 `--search-path=<path to ql repo>`, 不然识别不了... 坑了我好久.  

然后保存
```codeql
import python

from Function f
where f.getName().matches("index")
select f, "This is a function called get..."
```

```sh
/home/rmb122/repos/codeql/codeql query run test.ql -d ../codeql-database/
```
就能运行了, 或者用 vscode 插件也行, 更方便一点. 可以找到我们写的 index 函数.  

## 简单的 Taint demo

照着自带的示例, 可以依葫芦画瓢, 写出
```codeql
import python
import semmle.python.security.TaintTracking
import semmle.python.web.flask.Request

class SystemCommandExecution extends TaintTracking::Configuration {
    SystemCommandExecution() { this = "SystemCommandExecution Tracking" }

    override predicate isSource(DataFlow::Node src, TaintKind kind) {
        exists(FlaskRequestArgs taintSrc |
            src.asCfgNode() = taintSrc 
            // and taintSrc.isSourceOf(kind) 这里示例用的 get, 不是 dickstring 这个 kind, 所以需要注释掉
        )
    }

    override predicate isSink(DataFlow::Node sink, TaintKind kind) {
        exists(
            CallNode call |
            call.getFunction().pointsTo(Value::named("subprocess.check_output")) and
            call.getArg(0) = sink.asCfgNode()
        )
    }
}

from SystemCommandExecution config, DataFlow::Node src, DataFlow::Node sink
where config.hasSimpleFlow(src, sink)
select sink, src
```

再检测一下威力加强版示例
```python
import flask
import subprocess
from subprocess import check_output
from flask import request

app = flask.Flask(__name__)

@app.route('/index')
def index():
    return subprocess.check_output(flask.request.args.get('c', 'ls'))

@app.route('/index2')
def index2():
    tmp = flask.request.args.get('c', 'ls')
    tmp = tmp.split('|')
    return subprocess.check_output(tmp)

@app.route('/index3')
def index3():
    tmp = flask.request.args.get('c', 'ls')
    tmp = tmp.split('|')
    return check_output(tmp)

@app.route('/index4')
def index4():
    tmp = request.args.get('c', 'ls')
    tmp = tmp.split('|')
    return subprocess.check_output(tmp)

@app.route('/index5')
def index5():
    tmp = flask.request.args.get('c', 'ls')
    tmp = tmp + "i"
    return subprocess.check_output(tmp)

@app.route('/index6')
def index6():
    tmp = request.args.get('c', 'ls')
    tmp = tmp + "i"
    return subprocess.check_output(tmp)

@app.route('/index7')
def index7():
    tmp = request.args.get('c', 'ls')
    tmp = tmp + "i"
    return check_output(tmp)

app.run()
```

结果是 index, index5-7 都能检测出来, 2-4 没检测出来. 原因应该是 spilt 的结果没被 taint 到.  
将 isSink 替换成
```
override predicate isSink(DataFlow::Node sink, TaintKind kind) {
        "1" = "1"
}
```

也可以观察到 `tmp = tmp.split('|')` 的返回值是没被 select 到的. 也可以验证这一点.  

这样测试下来, 感觉最大的问题还是编写起来测试太费劲了, 就算只更改一点内容, 编译和运行都要 1~2 分钟才能完成. 对 database 也是同理, 只能重新生成, 没有更新的选项, 浪费了大量的时间.  
剩下的可能是自带的 taint tracking 不够给力, 还好官方给了相关的接口 (DataFlowExtension), 需要自己研究下添加额外 taint 的方法, 至少 split 肯定是得加进去的吧.
