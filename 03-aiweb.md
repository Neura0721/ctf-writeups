# AI Web 1.0 渗透实战 Writeup

> **目标：** AI Web 1.0 靶机（`172.16.0.17`）
> **攻击路径：** 端口扫描 → robots.txt 情报 → 递归目录爆破 → 敏感信息泄露 → 注入点探测
> **关键漏洞：** 信息泄露、SQL 注入

---

## 一、信息收集

### 1.1 端口扫描

```bash
masscan -p 0-65535 --rate=1000 172.16.0.17
nmap -p- 172.16.0.17
nmap -p 80 -sC -A 172.16.0.17
```

**结果：**

| 端口 | 服务 | 版本 |
|---|---|---|
| 80/tcp | http | Apache httpd |

**关键信息：**

- `http-title: AI Web 1.0`
- `http-robots.txt: 2 disallowed entries`
  - `/m3diNf0/`
  - `/se3reTdir777/uploads/`

> 💡 **robots.txt 直接泄露了两条敏感路径**，这是本题最重要的突破口。

---

## 二、目录爆破与路径发现

### 2.1 递归目录扫描

```bash
dirsearch -u http://172.16.0.17 -r
```

**扫描结果：**

| 路径 | 状态码 | 说明 |
|---|---|---|
| `/index.html` | 200 | 首页 |
| `/robots.txt` | 200 | 泄露敏感路径 |
| `/server-status` | 403 | 禁止访问 |
| `/m3diNf0/info.php` | 200 | **信息泄露** |

### 2.2 访问隐藏路径

直接访问 `/m3diNf0/` 目录被禁止，但访问其下的 `info.php` 成功：

```
http://172.16.0.17/m3diNf0/info.php
http://172.16.0.17/se3reTdir777/uploads/
```

> 💡 **思路：** 目录被禁止不代表其下的文件不可访问。uploads 目录的存在也暗示了后续可能存在文件上传点。

---

## 三、敏感信息泄露

访问 `info.php` 后，页面返回了大量敏感信息：

**1. Web 服务绝对路径**

```
/home/www/html/web1x443290o2sdf92213/m3diNf0/info.php
```

**2. 环境变量（PATH）**

```
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

**3. 应用根目录**

```
/home/www/html/web1x443290o2sdf92213
```

**4. 其他泄露项**

```
/m3diNf0/info.php
/usr/sbin/sendmail -t -i
```

### 危害分析

| 泄露内容 | 可利用价值 |
|---|---|
| Web 绝对路径 | 为文件包含、任意文件读取、日志投毒提供准确路径 |
| 环境变量 | 判断系统类型、已装组件、可执行程序位置 |
| 应用根目录 | 定位配置文件（如数据库连接信息） |

---

## 四、注入点探测

### 4.1 手工测试

访问 `http://172.16.0.17/se3reTdir777/` 后发现参数位置传入不同 id 时，页面返回内容存在差异 —— **怀疑存在 SQL 注入**。

### 4.2 Burp Suite 抓包

使用 Burp Suite 拦截请求，保存为 `1.txt`：

```
GET /se3reTdir777/?id=1 HTTP/1.1
Host: 172.16.0.17
...
```

### 4.3 SQLMap 自动化探测

```bash
sqlmap -r 1.txt --dbs
```

**结果：** 成功探测到注入点，并列出数据库信息。

---

## 五、总结

| 阶段 | 关键手段 |
|---|---|
| 端口扫描 | nmap -sC 带默认脚本，直接读出 robots.txt 内容 |
| 路径发现 | robots.txt 泄露 + 递归目录爆破 |
| 信息泄露 | `info.php` 暴露绝对路径与环境变量 |
| 注入探测 | Burp 抓包 + SQLMap 自动化利用 |

### 核心启示

1. **`nmap -sC` 别省** —— 默认脚本能直接读出 robots.txt，效率远高于手工访问
2. **robots.txt 是情报源** —— 开发者想藏起来的路径，恰恰是最有价值的
3. **目录禁止 ≠ 文件不可访问** —— 403 的是目录列表，不是目录下的具体文件
4. **信息泄露是「铺垫型」漏洞** —— 单看不致命，但为后续的文件包含、注入、日志投毒提供了必要条件

---

## 附：信息泄露常见来源

| 来源 | 典型文件 |
|---|---|
| 探针文件 | `info.php`、`phpinfo.php`、`test.php` |
| 版本控制 | `.git/`、`.svn/` |
| 备份文件 | `.bak`、`.zip`、`.sql`、`wwwroot.rar` |
| 配置文件 | `config.php`、`web.config`、`.env` |
| 爬虫协议 | `robots.txt`、`sitemap.xml` |
| 错误页面 | 报错回显中的绝对路径与堆栈信息 |
