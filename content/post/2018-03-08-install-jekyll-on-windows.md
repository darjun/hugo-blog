---
layout:		post
title:		"在Windows上安装Jekyll"
subtitle: 	"\"Start a blog trip\""
# description: "教你如何一步步在Windows上安装jekyll，常见问题及解决"
date:		2018-03-07T11:18:00
author:		"darjun"
image:	"/img/post-bg-2015.jpg"
tags:
    - 环境搭建
URL: "/2018/03/08/install-jekyll-on-windows/"
categories: [ 
    "Tools"
]
---

## 背景
最近想试试用Jekyll在Github搭建blog。选取网站模板，修改域名等等这些网上都有很详细的教程了，文末会附上链接，这里就不再赘述了。本文主要记录在Windows本地安装jekyll环境的过程，遇到的问题及如何解决的。

## 安装环境

### 1. 安装Ruby
在Windows上使用RubyInstaller安装比较方便，去[Ruby官网][1]下载最新版本的RubyInstaller。注意32位和64位版本的区分。

注意：勾选添加到PATH选项，以便在命令行中使用。

![添加PATH](/img/in-post/windows-jekyll/ruby-install.png)

安装完成界面：

![Ruby安装完成](/img/in-post/windows-jekyll/msys2-install.png)

这里需要勾选安装msys2，后面安装gem和jekyll时会用到：

![安装msys2和development toolchain](/img/in-post/windows-jekyll/ruby-installer2.png)

### 2. 安装RubyGems
Windows中下载[ZIP格式][2]比较方便，下载后解压到任意路径。打开Windows的cmd界面，输入命令：
```
$ cd {unzip-path} // unzip-path表示解压的路径
$ ruby setup.rb
```

### 3. 安装Jekyll
在cmd中输入:
```
$ gem install jekyll
```

### 4. 安装jekyll-paginate
在cmd中输入：
```
$ gem install jekyll-paginate
```

### 5. 验证安装完成
在cmd中输入：
```
$ jekyll -v
```

输出版本说明安装完成（我的版本为3.7.3）：
```
jekyll 3.7.3
```

### 6. 开启本地实时预览
切换到仓库所在目录，在cmd中输入:
```
$ jekyll serve
```

## 遇到问题及解决
### 1. gem install jekyll时报错，而且还是乱码！
```
C:\User>gem install jekyll
Temporarily enhancing PATH for MSYS/MINGW...
Building native extensions. This could take a while...
ERROR:  Error installing jekyll:
    ERROR: Failed to build gem native extension.

    current directory: C:/Ruby25-x64/lib/ruby/gems/2.5.0/gems/http_parser.rb-0.6.0/ext/ruby_http_parser
C:/Ruby25-x64/bin/ruby.exe -r ./siteconf20180308-3672-ueo7ea.rb extconf.rb
creating Makefile

current directory: C:/Ruby25-x64/lib/ruby/gems/2.5.0/gems/http_parser.rb-0.6.0/ext/ruby_http_parser
 make "DESTDIR=" clean
 'make' �����ڲ����ⲿ���Ҳ���ǿ����еĳ���
���������ļ���

current directory: C:/Ruby25-x64/lib/ruby/gems/2.5.0/gems/http_parser.rb-0.6.0/ext/ruby_http_parser
make "DESTDIR="
'make' �����ڲ����ⲿ���Ҳ���ǿ����еĳ���
 ���������ļ���

 make failed, exit code 1

Gem files will remain installed in C:/Ruby24-x64/bin/ruby_builtin_dlls/Ruby24-x6
4/lib/ruby/gems/2.4.0/gems/http_parser.rb-0.6.0 for inspection.
Results logged to C:/Ruby24-x64/bin/ruby_builtin_dlls/Ruby24-x64/lib/ruby/gems/2
.4.0/extensions/x64-mingw32/2.4.0/http_parser.rb-0.6.0/gem_make.out
```

参考oneclick/rubyinstaller2的[issue #98][3]。

首先cmd中输入：
```
$ chcp 850
```

切换编码之后安装：
```
$ gem install jekyll
```

下面是报错：
```
Temporarily enhancing PATH for MSYS/MINGW...
Building native extensions. This could take a while...
ERROR:  Error installing jekyll:
        ERROR: Failed to build gem native extension.

    current directory: C:/Ruby25-x64/lib/ruby/gems/2.5.0/gems/http_parser.rb-0.6.0/ext/ruby_http_parser
C:/Ruby25-x64/bin/ruby.exe -r ./siteconf20180308-3672-ueo7ea.rb extconf.rb
creating Makefile

current directory: C:/Ruby25-x64/lib/ruby/gems/2.5.0/gems/http_parser.rb-0.6.0/ext/ruby_http_parser
make "DESTDIR=" clean
'make' 不是内部或外部命令，也不是可运行的程序或批处理文件。

current directory: C:/Ruby25-x64/lib/ruby/gems/2.5.0/gems/http_parser.rb-0.6.0/ext/ruby_http_parser
make "DESTDIR="
'make' 不是内部或外部命令，也不是可运行的程序或批处理文件。

make failed, exit code 1

Gem files will remain installed in C:/Ruby25-x64/lib/ruby/gems/2.5.0/gems/http_p
arser.rb-0.6.0 for inspection.
Results logged to C:/Ruby25-x64/lib/ruby/gems/2.5.0/extensions/x64-mingw32/2.5.0
/http_parser.rb-0.6.0/gem_make.out
```

原来是没有make指令，上面的步骤其实已经安装了msys2，所以不会出现问题。对于没有勾选的童鞋，可以在cmd中输入下面命令来安装：
```
$ ridk install
```

安装完成之后再次安装jekyll和jekyll-paginate就ok了。

### 2. jekyll serve启动报错
```
Incremental build: disabled. Enable with --incremental
      Generating...
jekyll 3.7.3 | Error:  Permission denied @ rb_sysopen - C:/Users/username/NTUSER.DAT
```

这是因为jekyll默认使用4000端口，而4000是FoxitProtect（福昕阅读器的一个服务）的默认端口。网上有教程说kill掉FoxitProtect的进程，但是我觉得首先这个比较麻烦，其次重启计算机时FoxitProtect是默认启动的，除非关闭这个服务，这样又可能带来其他问题。所以最简单的办法还是指定端口：
```
$ jekyll serve -P 5555
```

### 参考链接
1. [Github Pages + Jekyll 独立博客一小时快速搭建&上线指南][4]
2. [RubyInstaller2 issue98][3]
3. [Install Ruby and the Ruby DevKit][5]

[1]:https://rubyinstaller.org/downloads/
[2]:https://rubygems.org/pages/download
[3]:https://github.com/oneclick/rubyinstaller2/issues/98
[4]:http://playingfingers.com/2016/03/26/build-a-blog/
[5]:http://jekyll-windows.juthilo.com/1-ruby-and-devkit/