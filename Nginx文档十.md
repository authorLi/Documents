## 十、为Windows而设计的Nginx

Windows版使用的Nginx是本地Win32 API（而非Cygwin仿真层）。当前版本仅仅使用了select()和poll()连接处理方法。因此它没有很高的性能和很好的可伸缩性。由于这个问题和一些其他的问题，windows版本的Nginx被视为是测试版本的Nginx。目前，除了XSLT过滤器、图像过滤器、GeoIP模块和嵌入式Perl语言之外，它提供的版本几乎与Unix版本的Nginx相同。

要下载Windows版本的Nginx，请下载最新主线发行版，因为现在主线发行版包括了已知的修补程序。然后解压缩，到nginx-1.17.4目录下，并运行nginx。这有一个例子：

```xml
cd c:\
unzip nginx-1.17.4.zip
cd nginx-1.17.4
start nginx
```

你可以运行tasklist命令来查看Nginx进程：

```xml
C:\nginx-1.17.4>tasklist /fi "imagename eq nginx.exe"

Image Name           PID Session Name     Session#    Mem Usage
=============== ======== ============== ========== ============
nginx.exe            652 Console                 0      2 780 K
nginx.exe           1332 Console                 0      3 112 K
```

其中有一个是主进程，其他的都是工作进程。 如果Nginx没有启动，你可以在错误日志目录的logs/error.log文件中找到答案。如果这个log文件没有被创建，那么应在Windows的日志中报告其原因。如果没有展示出期望的页面，也可以在logs/error.log文件中找到原因。

Nginx/Windows会使用运行目录的目录作为配置中相对路径的前缀。在上面的例子中，前缀就是*C:\nginx-1.17.4\\*。配置文件中的路径必须是使用正斜杠以Unix样式指定：

```xml
access_log   logs/site.log;
root         C:/web/html;
```

Nginx/Windows使用作为标准的控制台应用程序（不是服务）运行可以使用以下命令管理：

```xml
nginx -s stop			快速关机
nginx -s quit			优雅的关机
nginx -s reload		修改配置文件，使用修改后的文件创建新的工作线程，并优雅的关闭老的工作线程
nginx -s reopen		重新打开日志文件
```

### 已知的问题

尽管可以启动多个工作线程，但是仅仅有一个可以做任何事

不支持UDP代理方式

### 未来可能做到的改进

作为服务运行

使用I/O完成端口作为连接处理方法

在单个进程中使用多个工作线程