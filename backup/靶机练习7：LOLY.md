

## 靶机地址

https://www.vulnhub.com/entry/loly-1,538/

## 信息收集

扫描全端口，进行服务发现

```
nmap -n -v -sS -max-retries=0 -p- 172.16.33.25
```

![https://cos.kevinc.ltd/file/download?fileId=1019](https://cos.kevinc.ltd/file/download?fileId=1019)

发现只有80端口的web服务

进行进一步的版本发现和目录扫描（使用了nmap 的vuln脚本）

```
sudo nmap -p80 -script=vuln 172.16.33.25
```

![https://cos.kevinc.ltd/file/download?fileId=1020](https://cos.kevinc.ltd/file/download?fileId=1020)

发现有目录`/wordpress`和`wordpress/wp-login.php`，其中含有wordpress的登录页面

```
dirsearch -u 172.16.33.25 -w  /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50
```

![https://cos.kevinc.ltd/file/download?fileId=1021](https://cos.kevinc.ltd/file/download?fileId=1021)

## 利用漏洞获取shell

访问`/wordpress/wp-login.php`，随机输入账号密码，提示无法访问服务`loly.lc`

![https://cos.kevinc.ltd/file/download?fileId=1022](https://cos.kevinc.ltd/file/download?fileId=1022)

需要在`/etc/hosts`进行域名解析

```
#oscp
172.16.33.25 loly.lc
```

重新访问，提示未知用户名

由于服务是`wordpress`，可以使用工具`wpscan`进行用户名枚举

```
wpscan --url http://172.16.33.25/wordpress -e
```

![https://cos.kevinc.ltd/file/download?fileId=1023](https://cos.kevinc.ltd/file/download?fileId=1023)

获取用户名为`loly`，继续破解密码

```
wpscan --url http://172.16.33.25/wordpress -U loly -P /usr/share/wordlists/rockyou.txt
```

![https://cos.kevinc.ltd/file/download?fileId=1025](https://cos.kevinc.ltd/file/download?fileId=1025)

使用获得的用户名和密码登录`wordpress`后台

![https://cos.kevinc.ltd/file/download?fileId=1026](https://cos.kevinc.ltd/file/download?fileId=1026)

登录后台后找不到`Appearance`下的`Editor`，无法将木马写到404页面中

发现`AdRotate`的`Manage Media`中有上传点，可以尝试上传木马

使用`/usr/share/webshells/php`下的`php-reverse-shell.php`，将地址和端口改一下，改名后压缩为`shell.zip`

![https://cos.kevinc.ltd/file/download?fileId=1027](https://cos.kevinc.ltd/file/download?fileId=1027)

在`Settings`处可以看到上传路径

![https://cos.kevinc.ltd/file/download?fileId=1028](https://cos.kevinc.ltd/file/download?fileId=1028)

在kali端监听设置端口，并在网页中访问`wordpress/wp-content/banners/shell.php`,获得shell

![https://cos.kevinc.ltd/file/download?fileId=1029](https://cos.kevinc.ltd/file/download?fileId=1029)

## 提权

尝试上传 linpeas.sh 到靶机，发现不行，最后发现 `/tmp`目录中才能上传成功

通过`linpeas.sh`的提示，发现有现成的`exploit`可利用

![https://cos.kevinc.ltd/file/download?fileId=1030](https://cos.kevinc.ltd/file/download?fileId=1030)

> 刚开始用了40871.c这个exploit, 发现在kali上编译成功后靶机上执行不了，直接传到靶机上也编译不成功
> 

使用`searchsploit`找到`exploit`，在kali中进行编译

```
searchsploit -m 45010
gcc 45010.c -o exploit
```

传到靶机上，赋予执行权限后执行

![https://cos.kevinc.ltd/file/download?fileId=1031](https://cos.kevinc.ltd/file/download?fileId=1031)

成功获得root 权限，可查看 root.txt

## 参考资料

[Wordpress网站渗透测试](https://blog.csdn.net/weixin_51957047/article/details/124985217)