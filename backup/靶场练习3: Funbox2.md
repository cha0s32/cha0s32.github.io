## 信息收集阶段

扫描端口

```shell
sudo nmap -p- -n -v -sS --max-retries=0 172.16.33.30
```

<img src="https://cos.kevinc.ltd/file/download?fileId=706" style="zoom:33%;" />

发现开放端口21，22，80，扫描版本

```shell
sudo nmap -p21,22,80 -sV -A 172.16.33.30
```

<img src="https://cos.kevinc.ltd/file/download?fileId=708" style="zoom:33%;" />

扫描一下敏感目录，没发现有什么有价值的信息，除了一个robots.txt, 访问得到路径 /logs/ , nmap 已经扫出来了。



## 利用漏洞拿Shell

访问80端口网页，无有用信息，是一个apache 默认页面。

<img src="https://cos.kevinc.ltd/file/download?fileId=709" style="zoom:33%;" />

访问路径 /logs , 显示没有该页面。

<img src="https://cos.kevinc.ltd/file/download?fileId=710" style="zoom:33%;" />

尝试从 FTP 服务入手，尝试匿名登录，成功登录！

<img src="https://cos.kevinc.ltd/file/download?fileId=711" style="zoom:33%;" />

查看所有文件，看看有什么有价值的

<img src="https://cos.kevinc.ltd/file/download?fileId=712" style="zoom:33%;" />

可以全部下载下来解压查看，由于 `tom.zip` 权限跟其他`zip`权限不同，可以重点关注一下。同时查看一了一下`.@users` 和`.@admins`两个文件

<img src="https://cos.kevinc.ltd/file/download?fileId=707" style="zoom:33%;" />

直接解压发现需要密码，使用`john`工具进行暴破解压。

```shell
zip2john tom.zip > tom.hash 
# 提取zip文件哈希

cp /usr/share/wordlists/rockyou.txt .
sudo john tom.hash --wordlist=rockyou.txt
```

<img src="https://cos.kevinc.ltd/file/download?fileId=713" style="zoom:33%;" />

暴破出密码后，用密码解压，得到私钥 `id_rsa`

使用私钥登录服务器

```shell
ssh -i id_rsa john@172.16.33.30
```

<img src="https://cos.kevinc.ltd/file/download?fileId=714" style="zoom:33%;" />

看看根目录下有什么文件，看到很多`history` 文件，查看一下，发现`mysql_history`文件有密码。

<img src="https://cos.kevinc.ltd/file/download?fileId=716" style="zoom:33%;" />

<img src="https://cos.kevinc.ltd/file/download?fileId=717" style="zoom:33%;" />

登录成功，查看一下权限，没有 root 权限。尝试使用`sudo -s` 提权，输入刚才得到的密码，成功提权。

<img src="https://cos.kevinc.ltd/file/download?fileId=718" style="zoom:33%;" />



## 绕过rbash

尝试进入 `/root` 拿flag，发现 是有限制的 shell，要尝试绕过。

```shell
echo $SHELL
```

发现是`/bin/rbash`

尝试可不可以换shell ,  使用`bash` 或着 `sh`，成功绕过。

<img src="https://cos.kevinc.ltd/file/download?fileId=719" style="zoom:33%;" />