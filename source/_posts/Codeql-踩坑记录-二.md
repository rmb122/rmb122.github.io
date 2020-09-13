---
title: Codeql 踩坑记录 (二)
date: 2020-03-31 23:06:52
tags: [codeql]
---


首先需要解决的就是上次留下的问题, 添加自定义的 taint track.  
在自带的 tests 中就有示例, 可以参考 `ql/python/ql/test/library-tests/taint/extensions/ExtensionsLib.qll`  

<!-- more -->

最后大概是这样  
```codeql
class AnyCallFlow extends DataFlowExtension::DataFlowNode {
     AnyCallFlow() {
         exists(CallNode call |
            call.getFunction().(AttrNode).getObject() = this
         )
     }
 
     override ControlFlowNode getASuccessorNode() {
         result.(CallNode).getFunction().(AttrNode).getObject() = this
     }
}
```

意思就是如果一个 funccall 中是 val.attr 类型的, 且 val 被 taint 了, 那么整个 CallNode 都将被 taint.  
然后加到 Configuration 里面就可以了  

```codeql
override predicate isExtension(TaintTracking::Extension extension) {
    extension instanceof AnyCallFlow
}
```

此时就能够识别 split 等方法了, 不过这样的结果肯定是增加误报率了.  
这里插一句, 最近在看南大开在 B 站上的软件分析课程, 讲的挺好, 这里其实就是 soundness completeness 问题, 在安全这一块还是 soundness 好一点, 所以最好还是牺牲虚警率来提高漏报率吧.  

最后按照官方库的方法, 封装一下, 最后的结果  

```codeql
import python
import semmle.python.security.TaintTracking
import semmle.python.web.flask.Request

class AnyCallFlow extends DataFlowExtension::DataFlowNode {
     AnyCallFlow() {
         exists(CallNode call |
            call.getFunction().(AttrNode).getObject() = this
         )
     }
 
     override ControlFlowNode getASuccessorNode() {
         result.(CallNode).getFunction().(AttrNode).getObject() = this
     }
}

class DangerousFunctionArg0 extends Value {
    DangerousFunctionArg0() {
        exists(Value val |
            this = val and
            (
                val = Value::named("subprocess.check_output") or
                val = Value::named("os.system") or 
                val = Value::named("os.popen") or 
                val = Value::named("eval") or 
                val = Value::named("exec") or
                val = Value::named("flask.render_template_string")
            )
        )
    }
}

class DangerousFunctionArg0Sink extends TaintSink {
    DangerousFunctionArg0Sink() {
        exists(
            CallNode call, DangerousFunctionArg0 dangerous_func |
            call.getFunction().pointsTo(dangerous_func) and
            call.getArg(0) = this
        )
    }

    override predicate sinks(TaintKind taint) {
        any()
    }
}

class SystemCommandExecution extends TaintTracking::Configuration {
    SystemCommandExecution() { this = "SystemCommandExecution Tracking" }

    override predicate isSource(DataFlow::Node src, TaintKind kind) {
        src.asCfgNode() instanceof FlaskRequestArgs
    }

    override predicate isSink(DataFlow::Node sink, TaintKind kind) {
        sink.asCfgNode() instanceof DangerousFunctionArg0Sink
    }

    override predicate isExtension(TaintTracking::Extension extension) {
         extension instanceof AnyCallFlow
    }
}

from SystemCommandExecution config, DataFlow::Node src, DataFlow::Node sink
where config.hasSimpleFlow(src, sink)
select sink, src
```

检测以下 sample, 一共 10 个漏洞, 都能找到, 还是不错的  
```python
import flask
import subprocess
from subprocess import check_output
from flask import request

app = flask.Flask(__name__)

def passby(i):
    return i.split('123')

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

@app.route('/index8')
def index8():
    tmp = request.args.get('c', 'ls')
    tmp = tmp + "i"
    return flask.render_template_string(tmp)

@app.route('/index9')
def index9():
    tmp = request.args.get('c', 'ls')
    tmp = tmp + "i"
    return flask.render_template_string("asd", t=tmp)

@app.route('/index10')
def index10():
    tmp = request.args.get('c', 'ls')
    tmp = passby(tmp + "i")
    return flask.render_template_string("asd", t=tmp)

@app.route('/index11')
def index11():
    tmp = request.args.get('c', 'ls')
    tmp = passby(tmp + "i")
    return eval(tmp)

@app.route('/index12')
def index12():
    tmp = request.args.get('c', 'ls')
    tmp = passby(tmp + "i")
    return flask.render_template_string(tmp)

app.run()
```

最后, 其实感觉编写最大的难点还是需要思维的转换, 这种声明式的语言像 SQL 一样, 是告诉程序, 希望在 xx 地方是 xx, 且 xx 里面的 yy 是 zz 这样.  
需要一点时间来转变思维吧, 之后是这个官方 python 接口库感觉本身写的就有点乱 (逃, 各种类似的对象, 又是 PythonFunctionCall, CallNode 的, 同样的目的可以由一万种不同的方式达成. 感觉对新手确实不太友好. 等后续文档跟上吧.  
