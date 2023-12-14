

## 靶机地址

https://www.vulnhub.com/entry/bbs-cute-102,567/

## 信息收集

进行全端口扫描，确认目标开放端口和服务

```shell
nmap -n -v -sS --max-retries=0 -p- 172.16.33.9
```

<img src="https://cos.kevinc.ltd/file/download?fileId=835" style="zoom:50%;" />

对开放端口进行版本的扫描

```shell
nmap -sV -p22,80,88,110,995 -A 172.16.33.9
```

通过对各自服务的信息收集，并无入手点。对POP3服务进行过暴破，无结果

使用对web服务（80端口）进行目录及文件的发现

```shell
dirsearch -u http://172.16.33.9

# 如果要指定目录和指定扩展名，记得加参数 -f 进行文件发现，否则拼接出来全部是目录

dirsearch -u 172.16.33.9 -e php -f  --wordlists=/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt --thread=1000 
```

结果如下：

<img src="https://cos.kevinc.ltd/file/download?fileId=836" style="zoom:33%;" />

尝试多个页面后，在`index.php`页面中发现有版本信息 `CuteNews 2.1.2`



## 利用漏洞拿shell

使用`searchsploit`搜寻相关版本漏洞利用代码，有一个远程命令执行的exp --> 48800.py

<img src="https://cos.kevinc.ltd/file/download?fileId=837" style="zoom:33%;" />

初次利用并不成功，查看源代码，结合目录扫描的结果发现所有路径都多了 一个`/CuteNews`

<img src="https://cos.kevinc.ltd/file/download?fileId=838" style="zoom:33%;" />

删掉后重新利用，成功获得低权限shell

使用`netcat`给`kali`弹回来一个shell, 这样使用起来更方便

<img src="https://cos.kevinc.ltd/file/download?fileId=839" style="zoom:50%;" />

使用 python 换一个更舒服的 shell

```shell
python -c "import pty;pty.spawn('/bin/bash')"
```



## 提权

传上去一个`linpeas.sh`跑一下，发现一些带`suid`位的文件可利用

<img src="https://cos.kevinc.ltd/file/download?fileId=840" style="zoom:33%;" />

可以验证一下

```shell
find / -perm -u=s -type f 2>dev/null

# find / -perm /4000 -type f -user root 2>/dev/null
```

通过查资料得含有`suid`位的`hping3`可提权

进入`hping3`界面，查看权限，发现有查看`root`用户的权限。

<img src="https://cos.kevinc.ltd/file/download?fileId=841" style="zoom:50%;" />

也可以直接修改`/etc/passwd`和`/etc/shadow`文件，直接获得root权限



## 参考资料

[hping3 提权](https://www.cnblogs.com/zlgxzswjy/p/14115306.html)
