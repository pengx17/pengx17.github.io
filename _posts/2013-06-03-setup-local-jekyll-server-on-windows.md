---
layout: post
keywords: blog
description: blog
title: "在Windows上建立Jekyll平台"
categories: [learning, jekyll]
tags: [jekyll]
group: archive
icon: file-o
featured: true
---

为了本地调试搭建好的博客，我们得在当前机器上安装一个Jekyll服务，在本地生成完毕后再把更新push到github上面。
笔者主要的开发环境是在Windows上，但[Jekyll官方文档](http://jekyllrb.com/docs/installation/)并没有提供在Windows上面安装的方法。
按照<a href="#reference">网上的各种教程</a>，总算是成功在Win 7/8上面搭建成功了Jekyll。相对于Mac和Ubuntu(Debian)，在Windows上部署Jekyll有一些额外的修改。
为了给后来人提供方便，这篇文章我就介绍一下如何一步一步在Windows 7 x64上搭建一个支持中文的Jekyll系统。

<script>
$(function () {
    $('#tip-tutorial').tooltip(
    {title: '见本文末尾', 
    delay: { hide: 300 }
    });
})
</script>

<!-- more -->

***

**首先**, 从官网上下载`Ruby on Windows`，`Ruby DevKit`和`Python 2.7`。
下载页：[http://rubyinstaller.org/downloads/](http://rubyinstaller.org/downloads/)。在写这篇文章的时候，作者安装的分别是: [Ruby 2.0.0-p195 (x64)](http://rubyforge.org/frs/download.php/76958/ruby-2.0.0-p195-x64-mingw32.7z) 和 [DevKit-mingw64-64-4.7.2-20130224-1432-sfx.exe](http://rubyforge.org/frs/download.php/76808/DevKit-mingw64-64-4.7.2-20130224-1432-sfx.exe)。

<div class="alert alert-warning">
  <strong>Warning!</strong>
  注意，Jekyll现在不支持Python 3.0+版本！
</div>
<div class="alert alert-info">
  <strong>Notice!</strong> 
  安装好Ruby和Python后，别忘了把可执行文件夹加入到PATH下面。
</div>

安装好上述的文件包后，以<span class="label label-info">管理员模式</span>打开控制台，进入`Ruby DevKit`的目录下（比如D:\rubydevkit），执行下面的语句，对DevKit初始化：

    D:\rubydevkit\>ruby dk.rb init
    
然后，继续安装：

    D:\rubydevkit>ruby dk.rb install
        
接下来，设置gem环境，删除默认的下载源，然后把下载源改成淘宝的镜像（原因你懂的）。

    D:\rubydevkit>gem sources --remove https://rubygems.org/
    D:\rubydevkit>gem sources -a http://ruby.taobao.org/
    D:\rubydevkit>gem sources -l
    *** CURRENT SOURCES ***

    http://ruby.taobao.org
    # 请确保只有 ruby.taobao.org
    D:\rubydevkit>gem install rails
    
现在就可以在gem环境下安装Jekyll了，执行:

    D:\rubydevkit>gem install jekyll

<div class="alert alert-warning">
  <h4>Warning!</h4>
  现在还没有完!
  为了确保Jekyll能正确使用，我们还得对Jekyll的代码加上两个补丁。
</div>

1，[Jekyll的header和tag默认不支持UTF-8](http://log.medcl.net/item/2012/04/jekyll-encounter-encoding-problems/)，我们需要修改以下两个文件：

```diff
--- rubypath\lib\ruby\gems\2.0.0\gems\jekyll-1.0.2\lib\jekyll\convertible.rb
+++ rubypath\lib\ruby\gems\2.0.0\gems\jekyll-1.0.2\lib\jekyll\convertible.rb
@@ 搜索替换下面这行
-   source = File.read(@file)
+   source = File.read(@file, :encoding => "utf-8")

--- rubypath\lib\ruby\gems\2.0.0\gems\jekyll-1.0.2\lib\jekyll\tags\include.rb
+++ rubypath\lib\ruby\gems\2.0.0\gems\jekyll-1.0.2\lib\jekyll\tags\include.rb
-   self.content = File.read(File.join(base, name))
+   self.content = File.read(File.join(base, name), :encoding => "utf-8")
```
2，[修复Windows上面关于分页的问题](https://github.com/mojombo/jekyll/pull/1058):

```diff
--- rubypath\lib\ruby\gems\2.0.0\gems\jekyll-1.0.2\lib\jekyll\generators\pagination.rb
+++ rubypath\lib\ruby\gems\2.0.0\gems\jekyll-1.0.2\lib\jekyll\generators\pagination.rb
@@ -92,8 +92,9 @@ def self.subdirectories_identical(paginate_path, page_dir)
-   format = File.basename(site_config['paginate_path'])
-   format.sub(':num', num_page.to_s)

+   format = site_config['paginate_path']
+   format = format.sub(':num', num_page.to_s)
+   File.basename(format)
```

<div class="alert alert-success">
  <h4>Done！</h4>
  这应该就是所有要做的工作了。/dance
</div>

---

<h3 id="reference">参考文档</h3>

1. [windows下安装jekyll](http://aotee.com/windows-installation-jekyll)
1. [Fixing pagination on windows](https://github.com/mojombo/jekyll/pull/1058)
1. [Jekyll遭遇编码问题](http://log.medcl.net/item/2012/04/jekyll-encounter-encoding-problems/)
1. [RubyGem淘宝镜像](http://ruby.taobao.org/)
1. [像黑客一样写博客——Jekyll入门](http://www.soimort.org/posts/101/) <span class="label label-info">推荐阅读</span>
