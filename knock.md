# 芝麻开门！

### `openssh` 登录

`openssh` 是一种 `ssh` 的开源实现，它有两种安全级别的安全认证
一级是基于密码的安全验证
二级是基于密钥的安全验证
默认登录端口为 `22`

##### 密码登录
```bash
# ssh 默认端口 -p 22 
# ssh [你的可登陆用户名]@[你的服务器 ip ]
# 这里我在环境中设置了变量 hwip 为自己的服务器 ip, 请使用自己的服务器 ip
ssh root@$hwip
sudo apt install fish
```
##### 密钥登录

ssh-keygen 是生成密钥的工具之一。 
```bash
ssh-keygen
# 设置密码，这里先不设置, 按回车
```
将密钥上传到服务器
```bash
# ssh-copy-id [你的用户名]@[你的 ip4 地址]
ssh-copy-id root@$hwip
```

### 简单修改 `ssh` 使之更安全

##### 禁止密码登录 
实际上，一旦你设置了密钥登录。
你已经密钥可以无密码访问这个主机
你就可以编辑 /etc/ssh/sshd_config 文件来禁止密码认证。

```bash
vim /etc/ssh/ssh_config
# 搜索字符串 PasswordAuthentication，将默认行更改为 no：
```


##### 修改默认登录端口 
```bash
vim /etc/ssh/ssh_config
# 这里为了方便教学，保留 22 端口
# 添加端口
ListenAddress 0.0.0.0:22
ListenAddress 0.0.0.0:5522
# 保存并重启 SSH 服务器
sudo systemctl restart sshd
```
同样的你还可以进行一些常见设置
```bash
Port 2222 # 更改登录端口

ListenAddress #监听的IP

Protocol 2 #SSH版本选择

HostKey /etc/ssh/ssh_host_rsa__key  # 私钥保存位置
AllowUsers    用户1 用户2  # 制定用户登录
AllowGroups   用户组1 用户组2  # 指定组登录
Banner /etc/issue  # 要添加漂亮的欢迎消息（例如，输出 /etc/issue 文件），请配置 Banner 选项：
ServerKeyBits  # 1024

ServerFacility AUTH  # 日志记录ssh登陆情况

#KeyRegenerationInterval 1h  # 重新生成服务器密钥的周期

#ServerKeyBits 1024  # 服务器密钥的长度

LogLevel INFO  # 记录sshd日志消息的级别

#PermitRootLogin yes  # 是否允许root远程ssh登录

#RSAAuthentication yes  # 设置是否开启ras密钥登录方式

#PubkeyAuthentication yes  # 设置是否开启公钥验登录方式

#AuthorizedKeysFile .ssh/authorized_keys  # 设置公钥验证文件的路径

#PermitEmptyPasswords no  # 设置是否允许空密码的账号登录

X11Forwarding yes  # 设置是否允许X11转发

GSSAPIAuthentication yes  # GSSAPI认证开启
```

### 不秘密的端口

##### 扫描端口

攻击经常在攻击服务器之前使用端口扫描程序对打开的端口进行自动扫描。
所以更改默认的 `ssh` 端口不是保护服务器的安全方法
把端口隐藏才是保护SSH服务器的更好方法。
例如，如果要设置端口22的端口断开功能，则仅当您依次请求端口10001、10002、10003时，此端口才会打开,平常都处于无效的状态，正确完成序列后，防火墙将为您打开端口22。

# nmap $hwip
# 这里为了快速，直接指定 5522 端口

```bash
nmap -p 22 $hwip
```


###### 安装并设置 `iptables`

更新并安装 `iptables`
```bash
sudo apt update && sudo apt install iptables iptables-persistent
```

将需要允许所有已建立的连接和正在进行的会话通过 `iptables`
```bash
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

阻止 `5522` 端口被传入数据
```bash
iptables -A INPUT -p tcp --dport 5522 -j REJECT
```

保存 `iptables` 规则
```bash
sudo iptables-save > /etc/iptables/rules.v4
```


### 安装和配置 `Knockd`

安装命令 
```bash
sudo apt install knockd -y
```
一旦安装了端口敲门服务后，你将需要启用敲门服务才能使用。可以通过编辑 `/etc/default/knockd` 文件来做到这一点：

```bash
vim /etc/default/knockd
START_KNOCKD=1
KNOCKD_OPTS="-i eth0" // 这里该为自己的网卡，可用 ip -4 a 查看
```

接下来，你可以通过编辑 `/etc/knockd.conf` 文件来配置敲门：

```bash
vim /etc/knockd.conf
```

##### 配置例子

** 注意这里与模板不同，命令用 -I 置顶而不是 -A 追加 **
```bash
[options] 
    logfile = /var/log/knockd.log

[openSSH] 
    sequence = 1145,1451,4514 
    seq_timeout = 20 
    tcpflags = syn 
    command = /sbin/iptables -I INPUT -s %IP% -p tcp --dport 5522 -j ACCEPT

[closeSSH] 
    sequence = 4514,1451,1145 
    seq_timeout = 20 
    command = /sbin/iptables -D INPUT -s %IP% -p tcp --dport 5522 -j ACCEPT 
    tcpflags = syn
```
```bash
logfile = /var/log/knockd.log 设置日志，可用 tail -f /var/log/knockd.log 跟踪
sequence 设置为自己想要的端口序列。
seq_timeout 设置为20，用于敲门后有效时间以免超时时间过小出错。
command为要添加的防火墙命令，在openSSH下插入一条开启 5522端口的防火墙规则 -I 为插入到规则最前面，最先生效，以防止过滤所有端口的情况将此条规则吃掉。
closeSSH目的是下为删除（-D）之前插入到开启5522的规则. ```

##### 启动敲门服务
```bash
sudo systemdctl start knockd  #<-- 启动 knockd 服务
sudo systemdctl stop knockd  #<-- 停止 knockd 服务
sudo systemdctl knockd restart #<-- 重启 knockd 服务
sudo systemdctl enable knockd  #<-- 开机自启动 knockd 服务状态
sudo systemdctl status knockd  #<-- 查看 knockd 服务状态
```

好了，我们查看当前的防火墙规则

```bash
iptables -L -n 
```

##### 华为云打开端口


##### telent 敲门

```bash
telnet $hwip 1145
telnet $hwip 1451
telnet $hwip 4514
```


强烈推荐:
(arch-OpenSSH)[https://wiki.archlinuxcn.org/wiki/OpenSSH]
(wikipedia)[https://zh.wikipedia.org/zh-hk/Secure_Shell]
(iptables防火墙常规应用命令使用整理)[https://hiyae.com/document/iptables-command.html]

参考文献:
(001.SSH配置文件)[https://www.cnblogs.com/itzgr/p/9888726.html]
(Linux 远程连接之 SSH 新手指南)[https://linux.cn/article-13726-1.html]
(在Ubuntu Linux上使用端口敲门保护SSH服务器)[https://hiyae.com/document/ubuntu-install-knockd.html]
(在 Ubuntu 中配置 SSH 的完整指南)[https://linux.cn/article-15175-1.html]
(腾讯云主机上部署端口敲门Knock服务)[https://cloud.tencent.com/developer/article/1832711]
(linux中的knockd服务--端口敲门)[https://www.cnblogs.com/guangdelw/p/18402272]
