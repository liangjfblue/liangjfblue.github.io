---
layout:     post                  
title:      go
subtitle:   wktmltopdf实战
date:       2019-08-11          
author:     Liangjf                  
header-img: img/post_bg_82.jpg
catalog: true                      
tags:                       
    - go
---

# wktmltopdf实战

有个需求是把订单的内容生成pdf，经过寻找，找到了```go-wktmltopd```


wktmltopdf工具是使用Webkit引擎来将HTML网页转换为PDF文件，关于wkhtmltopdf工具的相关信息可以参见：

```http://code.google.com/p/wkhtmltopdf/```


```http://www.oschina.net/p/wkhtmltopdf/```


## 步骤
## 1、安装wktmltopdf
下载地址：[wktmltopdf](https://wkhtmltopdf.org/downloads.html)

直接在这里下载自己机器对应的版本就ok了。因为 `go-wkhtmltopdf` 实质是通过使用`wkhtmltopdf` 工具来实现html和pdf的转换的。所以 `wkhtmltopdf` 必须先安装。

## 2、检查是否正常使用
> wktmltopdf test.html test.pdf

`test.html` 的内容：

    <html>
    <head>
    <meta charset="utf-8">
    <title>测试测试测试</title>
    </head>
    <body>
        <h1>我的第一个标题</h1>
        <p>我的第一个段落。</p>
    </body>
    </html>

可能会报错：

> wkhtmltopdf: error while loading shared libraries: libpng12.so.0: cannot open shared object file: No such file or directory

可能还有其他的，按照错误提示，百度n多解决方法了，就是缺少对应的动态库。

比如`libjpeg`，`libpng`等。因为 `wkhtmltopdf` 是先把 `html` 转换为图片，然后再转换为pdf的。

例如安装libpng12
> sudo yum install libpng12

## 3、再次执行命令
> wktmltopdf test.html test.pdf

可以看到当前目录生成了 test.pdf 文件

但是中文格式有点问题：

![中文格式](https://github.com/liangjfblue/liangjfblue.github.io/blob/master/img/post_go_pdf2.png?raw=true)

中文重叠了。（这是另外一个`html`文件生成的，因为在写这篇文章时，我已解决这个问题，所以就重现不了，但是情况是一样的。）

## 4、中文乱码或者中文重叠
### 4.1、在这个网站下载简体中文字体 
> http://www.font5.com.cn/font_search.php?

### 4.2、把字体文件放在以下目录（没有就创建）
> /usr/share/fonts/chinese/TrueType/

## 5、再次执行转换命令，中文就正常显示了。

![中文正常](https://github.com/liangjfblue/liangjfblue.github.io/blob/master/img/post_go_pdf1.png?raw=true)

如果还是中文乱码，可能是你的html不是中文编码，可以在开头加入：
> <meta http-equiv="content-type" content="text/html;charset=utf-8">

## 6、总结
`wktmltopdf` 应该是应用最广的 `html` 转pdf工具了，因为他支持css格式，可以原样的转换为pdf格式。

