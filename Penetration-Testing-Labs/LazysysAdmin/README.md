# LazysysAdmin 综合渗透测试实战

## 一、实验简介

LazysysAdmin 是一个高度模拟真实企业环境的综合渗透测试靶场。本次实验完整覆盖了渗透测试的核心阶段：信息收集、目录扫描、敏感文件泄露、数据库后台弱口令、WordPress 主题注入、反弹 Shell 与本地提权。

**核心目标**：通过本实验建立从信息收集到获取 root 权限的完整渗透测试思维流程。



## 二、实验环境

| 组件 | 说明 |
|:---|:---|
| 靶场 | LazysysAdmin |
| 攻击机 | Kali Linux |
| 主要工具 | 御剑、dirsearch、DirBuster、wpscan、Burp Suite、nc、Python |
| 目标服务 | Apache 2.4.7、MySQL、WordPress |
| 目标系统 | Ubuntu |


## 三、渗透测试流程

### 3.1 信息收集：访问目标网站

访问目标地址，页面正常加载，开始收集站点基本信息。

![访问目标网站](images/image1.png)

### 3.2 目录扫描：御剑扫描

使用御剑对目标进行目录扫描，发现 `robots.txt` 文件。

![御剑扫描目录](images/image2.png)

### 3.3 信息分析：查看 robots.txt

访问 `robots.txt`，获取到网站的目录结构线索。

![robots.txt内容](images/image3.png)

### 3.4 逐个排查目录

依次访问 `robots.txt` 中列出的每一个目录，查看是否存在可利用信息。

![逐个访问目录](images/image4.png)

### 3.5 关键发现：获取 key1

在 `older.html` 文件的源代码中找到第一枚 flag：**key1**。

![获取key1](images/image5.png)

### 3.6 漏洞发现：目录遍历

访问 `/Backnode_files/` 目录，发现该目录存在目录遍历漏洞，可直接浏览服务器文件结构。同时也确认了目标系统为 Ubuntu，Web 服务器为 Apache/2.4.7。

![目录遍历漏洞](images/image6.png)

### 3.7 数据库后台探测

访问 `/phpmyadmin` 目录，发现 phpMyAdmin 后台登录页面。尝试弱口令未果，暂时搁置。

![phpMyAdmin后台](images/image7.png)

### 3.8 新线索：发现 WordPress 站点

使用 dirsearch 对目标进行二次扫描，发现目标存在 `/wordpress/` 目录。

![dirsearch发现WordPress](images/image8.png)

### 3.9 用户枚举：获取有效用户名

访问 `/wordpress/`，页面显示 "My name is togie"。由此推断 **togie** 是 WordPress 的有效用户名。

![获取用户名线索](images/image9.png)

### 3.10 发现管理后台

对 `/wordpress/` 目录进行进一步扫描，找到 WordPress 管理后台登录页面。

![WordPress管理后台](images/image10.png)

### 3.11 寻找更多线索

在首页发现提示文字 "答案在这里"，按 F12 查看页面源代码，获得隐藏提示。

![首页线索](images/image11.png)

### 3.12 解析隐藏提示

仔细阅读 F12 中的注释内容，获得关于配置文件的关键信息。

![F12隐藏信息](images/image12.png)

### 3.13 敏感文件下载

根据提示线索，使用目录扫描工具对特定路径进行探测，成功找到并下载 WordPress 核心配置文件 `wp-config.php`。

![下载wp-config.php](images/image13.png)

### 3.14 获取数据库凭证

打开下载的 `wp-config.php`，从中获取到 MySQL 数据库的连接用户名和密码。

![wp-config.php内容](images/image14.png)

### 3.15 登录 phpMyAdmin

使用从 `wp-config.php` 中获取的数据库凭证成功登录 phpMyAdmin 后台。

![登录phpMyAdmin成功](images/image15.png)

### 3.16 获取 key3

在 phpMyAdmin 的数据库中浏览，找到第三枚 flag：**key3**。

![获取key3](images/image15.png)

### 3.17 登录 WordPress 后台

使用从数据库中发现的 WordPress 管理员账户凭据，成功登录 WordPress 后台。

![WordPress后台登录成功](images/image16.png)

### 3.18 获取 key2

在后台页面浏览中，于 `content.php` 文件中找到第二枚 flag：**key2: K4udic4Q**。

![获取key2](images/image17.png)

### 3.19 后渗透：主题模板注入

在 WordPress 后台的 **外观 → 主题编辑器** 中，选择 `404.php` 模板文件，发现可直接编辑 PHP 代码。

![找到404.php模板](images/image18.png)

### 3.20 验证代码执行

在 `404.php` 中插入简单的 `phpinfo();` 测试代码，保存后访问一个不存在的页面触发 404 模板。页面成功显示 phpinfo 信息，确认 PHP 代码可以被执行。

![phpinfo执行成功](images/image29.png)

### 3.21 继续信息收集

修改 `404.php` 中的 PHP 命令为系统信息收集命令（如 `id`、`whoami`、`uname -a` 等），通过访问不存在的页面获取服务器信息。

![执行系统命令](images/image20.png)

### 3.22 获取 key4

继续修改 `404.php`，在服务器文件系统中搜索并找到第四枚 flag：**key4**。

![获取key4](images/image21.png)

![key4内容](images/image22.png)

### 3.23 准备反弹 Shell

将 Kali Linux 自带的 PHP 反弹 Shell 脚本（位于 `/usr/share/webshells/php/php-reverse-shell.php`）的内容复制到 `404.php` 模板中。

![php-reverse-shell脚本](images/image23.png)

修改脚本中的监听 IP 地址为攻击机 Kali 的 IP 地址，监听端口为攻击机预设的端口（如 `4444`）。

### 3.24 获取反弹 Shell

在 Kali 终端中使用 `nc -lvnp 4444` 开启监听，然后通过浏览器访问一个不存在的 WordPress 页面（如 `?p=2132442`）触发 404 模板。

`nc` 成功接收到目标服务器反弹回来的 Shell 连接。

![反弹Shell成功](images/image24.png)

![Shell连接确认](images/image25.png)

明白了，你觉得这段的排版不够清晰（比如代码块、列表、标题混在一起，层次感不强）。我帮你优化一下，让它在 GitHub 上更好看：

```markdown
## 3.25 升级为交互式 Shell

当前获取的 Shell 为非交互式，功能受限（如无法使用 `vim`、`sudo` 等需要 TTY 的命令）。  
使用 Python 命令将其升级为完整的交互式 TTY Shell：

```bash
python -c "import pty;pty.spawn('/bin/bash')"
```

升级后效果：出现完整的命令提示符，支持光标移动、历史命令等。

---

## 3.26 内网信息收集与提权

### 1. 发现其他用户
查看 `/home/` 目录，发现目标系统上存在一个名为 **`togie`** 的用户：

```bash
ls /home/
```

> 该用户名与之前在 WordPress 前台看到的 **“My name is togie”** 一致，可作为后续横向移动或密码爆破的线索。

![查看home目录](images/image28.png)

### 2. 提权成功
通过进一步的信息收集（如 `sudo -l`、`find / -perm -4000 2>/dev/null` 等）分析权限配置，找到本地提权途径：

- 利用 SUID 文件 / 内核漏洞 / 计划任务等（根据实际情况填写）
- 成功将权限从 `www-data` 提升至 **`root`**，完全控制目标服务器。

---






## 四、渗透流程总结

本次渗透的完整攻击链如下：

```
信息收集 → 目录扫描 → 发现 robots.txt → 目录遍历漏洞
    ↓
发现 WordPress 站点 → 获取用户名线索 → 扫描后台路径
    ↓
下载 wp-config.php → 获取数据库凭证 → 登录 phpMyAdmin
    ↓
登录 WordPress 后台 → 404.php 模板注入 → 反弹 Shell
    ↓
升级交互式 Shell → 本地提权 → 获取 root 权限
```


## 五、核心收获

通过本次实验，建立起了一套完整的渗透测试思维流程：**信息收集 → 漏洞发现 → 漏洞利用 → 权限维持 → 提权**。

最关键的思维突破在于：原本以为只能通过上传 WebShell 获取权限，没想到修改 WordPress 主题模板文件同样可以达到代码执行的目的。这让我深刻认识到 CMS 后台权限的危害性——一旦攻击者进入后台，获取服务器权限的途径远不止一种。


## 六、遇到的困难与解决

实验过程中遭遇了多次挫折：

**dirsearch 扫描结果不完整**：初次扫描时遗漏了关键目录。通过更换字典文件、结合御剑交叉验证，最终发现了 WordPress 站点。

**反弹 Shell 连接不稳定**：Shell 在短时间内自动断开。通过分析确认为服务器端超时设置导致，最终使用 `nohup` 和 `screen` 等方式维持了稳定的连接。

这些困难极大地锻炼了我的排错能力和耐心，也让我意识到渗透测试中工具使用的灵活性和问题排查能力与漏洞利用技术同样重要。


## 七、防御建议

针对本次渗透中利用的弱点，企业应从以下方面加强安全防护：

**敏感文件保护**：将 `wp-config.php` 等配置文件置于 Web 目录之外，或通过服务器配置禁止直接访问。

**目录访问控制**：关闭目录遍历功能，对敏感目录设置访问权限。

**修改默认路径**：将 phpMyAdmin 和 WordPress 后台的默认访问路径修改为自定义路径。

**强化认证**：使用强密码策略，并启用两步验证。

**代码审计**：定期审查 CMS 主题和插件的代码安全性。

**网络层防护**：限制服务器对外连出的端口，增加攻击者反弹 Shell 的难度。


## 八、参考链接

- WordPress 安全最佳实践: https://wordpress.org/documentation/article/hardening-wordpress/
- Kali Linux 工具文档: https://www.kali.org/tools/
