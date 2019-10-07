## 二、入门

nginx有一个主进程和几个工作进程，主进程的作用主要是读取和评估配置，并维持工作进程。工作进程对请求做实际上的处理。nginx使用基于事件的模型和依赖操作系统的机制有效的分发请求到工作进程。工作进程的数量在配置文件中被定义，可以针对给定的配置固定，也可以自动调整为可用CPU内核数量。

Nginx和其模块的工作方式在配置文件中被确定。默认情况下，配置文件的名称为nginx.conf并且被放置在 /usr/local/nginx/conf，/etc/nginx或/usr/local/etc/nginx文件夹下

### 启动，停止，重新装载配置文件

想要启动Nginx，运行可运行文件即可。一旦Nginx启动，就可以用-s参数调用可执行文件对Nginx进行控制。

当标识可能是一下几种时：

- stop——快速停止
- quit——优雅的停止
- reload——重新装载配置文件
- reopen——重新打开日志文件

例如：想要Nginx在处理完当前请求后再停止工作进程的话就使用以下命令：*nginx -s quit*

如果不重新装载配置文件或不重启Nginx的话，对配置文件的改变是不会生效的。想要重新装载配置文件可以使用以下命令：*nginx -s reload*

一旦主进程接收到了重新装载配置文件的信号，主进程会先检验配置文件语法的合法性，并且会开始进行尝试应用其中提供的配置。如果成功重新装载了，主线程会开启一个新的工作线程并且向老的工作线发送信息请求他们停止工作。否则，主进程会回滚这些改变并继续使用老的配置进行工作。老的工作进程收到了停止运行的命令后会停止接受新的连接并且继续处理当前的请求直到所有的请求都被处理完。在那之后老的工作进程会退出。

信号也可能在Unix工具（例如kill实用工具）的帮助下被送到Nginx进程。在这个过程中，信号可以直接被送到已知进程ID的进程中。默认情况下，Nginx的主进程的进程ID会被写到/usr/local/nginx/logs或/var/run目录下的nginx.pid文件中。例如，如果主进程的ID是1628，想要去发送退出信号到Nginx以实现优雅的关闭，就执行：

*kill -s QUIT 1628*

想要获得所有的运行中的Nginx进程列表，可以使用ps命令，例如可以这样写：*ps -ax | grep nginx*

### 配置文件的结构

在配置文件中Nginx包含很多被指定命令控制的模块。这些指令分为简单指令和块指令。一个简单指令包括以空格隔开的名字和参数并且以分号（；）结尾。一个块指令的构成和简单指令差不多，但是它并不是以分号结尾，而是以括号（{}）括起来的一系列额外命令结尾。如果一个块指令在括号内有其他指令，他就被称作上下文（例如：events、http、server和location）

放置在任何上下文外部的配置文件中的指令都被认为是在主上下文中。events和http指令属于主上下文，server在http中，location在server中。

剩下的行首有井号（#）的都被认为是注释

### 提供静态内容访问服务

web服务器的一个重要任务就是分发文件（例如图片或静态HTML文件）。你将实现一个实例，其中根据请求，文件将从不同的本地目录提供/data/www（这里将包含一些HTML文件）和/data/images（这里包含一些图片），这可能需要编辑配置文件并且在server块中的http块中编辑两个location块

首先，创建/data/www目录并且编写一个index.html并创建一个/data/images目录存储一些图片文件

下一步，打开配置文件，默认的配置文件已经包含了server块的几个实例，大部分已被注释掉。现在注释掉所有原来的server块，编写一个新的server块

```xml
http {
		server{}
}
```

通常来说，配置文件可以包括几个server块，这些快通过它们监听的端口号和服务名称的不同来区分。一旦Nginx确定了哪个server块处理请求，它就会根据server块内的location指令的参数测试请求标头中指定的URI

```xml
location / {
		root /data/www;
}
```

这个location块中选定了“/”前缀与请求中的URI相比。对于匹配上的请求，此URI将会被添加到root指令指定的路径，在此例中就是被添加到/data/www下以形成本地文件系统上所请求文件的路径。如果有多个location块，Nginx将选择前缀最长的串，location上提供了最短的前缀（长度为1），因此只有在所有其他location块都未匹配的情况下，才会匹配该location块

下一步，增加第二个location块：

```xml
location /images/ {
		root /data;
}
```

它将匹配以/images/开头的请求（location / 也会匹配这个URI，但是由于它是路径最短的，所以不悠闲匹配）

最后server的配置是像这样的：

```xml
server {
		location / {
			root /data/www;
		}
		
		location /images/ {
			root /data;
		}
}
```

这已经是服务器的工作配置，它会监听8080端口并在本机用http://localhost/是可访问的。在以URI /images/开头的请求的响应中，服务器将要从/data/images目录中发送文件。例如，http://localhost/images/example.png这样的请求的响应，Nginx将会发送 /data/images/目录下的example.png文件。如果这个文件不存在，Nginx将会响应404错误。如果请求不以/images/开头，那么它将要匹配到 /data/www这个目录。例如，响应http://localhost/some/example.html这个请求，Nginx将要发送/data/www/some/目录下example.html文件。

为了应用这个新配置，启动Nginx（如果它未启动的话）或者发送重新载入配置文件的信号到Nginx的主进程，只要执行以下命令：

```xml
nginx -s reload
```

如果某些功能无法工作，你可以尝试在/usr/local/nginx/logs目录下的access.log和error.log文件。

### 设置一个简单的代理服务

Nginx的一种常用用法就是将其作为代理服务器使用，Nginx将接收请求，然后把请求转发给内部网络上的服务器，并将从服务器上取得的数据返回给客户端，Nginx对外表现为一个服务器（伪装成服务器）

我们将配置一个简单的代理服务器，它们服务于对本地目录中图片的请求，并发送所有其他的请求到代理服务器。在这个例子中，所有的服务都被定义在一个简单的Nginx实例中。

第一步，通过在Nginx的配置文件中添加一个server块来定义一个代理服务器，如下：

```xml
server {
		listen 8080;
		root /data/up1;

		location / {
		}
}
```

这是一个简单的监听8080端口的服务器（就以前来说，listen命令如果没有特别说明将会使用80端口）并且此服务会映射到本地的/data/up1目录。创建这个目录并编写一个index.html文件放在里面。需要注意的是，这里配置的root指令是在server的上下文中，这样当location块中的root指令无法匹配上时，就会匹配server块下的root

下一步，使用上一步的配置并对其修改使其成为代理服务器配置。在第一个location块中，将*proxy_pass*指令与参数中指定的代理服务器的协议，名称和端口（本例中为http://localhost:8080）放置在一起

```xml
server {
		location / {
				proxy_pass http://localhost:8080;
		}

		location /images/ {
				root /data;
		}
}
```

我们将修改第二个location块，使它将以/images开头的请求映射到本地目录/data/images转变为使其匹配典型的图片文件扩展名，修改如下所示：

```xml
location ~ \.(gif|jpg|png)$ {
		root /data/images;
}
```

这个参数将以正则表达式匹配所有以.gif、.jpg或.png结尾的URI请求。正则表达式以~开头。与此相关的请求将被映射到/data/images目录

当Nginx来选择一个location块来处理请求时，它首先会检查location指令的前缀匹配，记住location匹配到的最长的前缀，然后会检查正则表达式，如果有匹配到正则表达式，则Nginx会选择此location配置项，否则它将选择之前记住的匹配项。

代理服务器最终配置如下：

```xml
server {
		location / {
				proxy_pass http://localhost:8080;
		}
		location ~ \.(gif|jpg|png)$ {
				root /data/images;
		}
}
```

此服务器将会过滤以.gif、.jpg或.png结尾的请求并且将它们映射到本地的/data/images目录（通过添加URI到root指令的参数）并将所有其他请求发送到上面配置好的代理服务器上。

为了新的配置能够生效，请按照前几节的的说明发送信号到Nginx的主进程

### 设置FastCGI代理

Nginx可用于将请求路由到FastCGI服务器，该服务器运行使用各种框架和汇编语言（例如PHP）构建的应用程序

最基本的使用FastCGI服务的配置除了使用fastcgi_pass代替proxy_pass指令外，还要设置好fastcgi_pass指令传递给FastCGI服务器的参数。假设FastCGI可以通过localhost:9000访问到。先把前一部分写的配置进行修改，修改proxy_pass指令修改为fastcgi_pass指令，并将参数改为localhost:9000。在PHP中*SCRIPT_FILENAME*参数被用于定义脚本名称，*QUERY_STRING*参数被用于传递请求参数。最终配置如下：

```xml
server {
		location / {
				fastcgi_pass localhost:9000;
				fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
				fastcgi_param QUERY_STRING		$query_string;
		}

		location ~ \.(gif|jpg|png)$ {
				root /data/images;
		}
}
```

这样设置就可以实现一个服务器，该服务器将通过FastCGI协议将除静态图片请求之外的所有请求路由到在localhost:9000上运行的代理服务器