## 十一、Nginx如何处理请求

### 基于名称的虚拟服务器

Nginx会首先确定那个服务器会处理该请求。让我们看一下下面的一个有三个监听80端口的简单配置吧：

```xml
server {
		listen 80;
		server_name	example.org www.example.org;
		...
}

server {
		listen 80;
		server_name example.net www.example.net;
		...
}

server {
		listen 80;
		server_name example.com www.example.com;
		...
}
```

在此配置中，Nginx仅测试请求的标头字段“Host”来确定将请求路由到哪个服务器上。如果它的值与任何服务器名都不匹配，或者请求根本不包含此标头字段，则Nginx将会将此请求路由到监听此端口的默认服务器。在上面的配置中，默认服务器是第一个服务器——这是Nginx的标准默认行为。你也可以在listen指令后面用default_server参数明确的设置默认的服务器。

```xml
server {
		listen 80 default_server;
		server_name example.net www.example.net;
		...
}
```

default_server这个参数是自0.8.21版本以后采用的，之前是采用default参数

需要注意的是default_server参数是listen的参数而不是server_name的参数

### 如何防止使用未定义的服务器名称处理请求

如果不允许请求标头不带有“Host”的情况，你可以定义一个像如下的只会丢弃请求的服务器：

```xml
server {
		listen 80;
		server_name "";
		return 444;
}
```

像这样，在server_name后面设置一个空字符串，这样就会匹配所有没有“Host”请求标头的请求，并且将返回Nginx特殊的非标准444状态码，以关闭连接。

自从版本0.8.48开始服务器server_name “”这样的设置是可以省略的。在早期的版本中，默认使用主机名作为默认的server_name

### 基于名称和基于IP的混合虚拟服务器

让我们来看一下复杂的虚拟服务器监听不同IP地址的配置：

```xml
server {
		listen 192.168.1.1:80;
		server_name example.org www.example.org;
}

server {
		listen 192.168.1.1:80;
		server_name example.net www.example.net;
}

server {
		listen 192.168.1.2:80;
		server_name example.com www.example.com;
}
```

在这个配置中，Nginx首先会根据server块的listen指令后面的内容测试IP地址和端口号。然后测试匹配了IP地址和端口号的那个server块的server_name的请求标头的“Host”字段。如果请求的服务器未找到，那么将交由默认服务器处理。例如，在192.168.1.1:80这个端口上接收到www.example.com的请求，将路由到192.168.1.1:80的默认服务器（就是第一个根据IP地址和端口号匹配到的服务器）处理，因为在这个端口号中没有没有定义处理www.example.com的服务器

如前所述，默认服务器是监听端口的属性，每个不同的端口上应该定义着不同的默认服务

```xml
server {
		listen 192.168.1.1:80;
		server_name example.org www.example.org;
}

server {
		listen 192.168.1.1:80 default_server;
		server_name example.net www.example.net;
}

server {
		listen 192.168.1.2:80 default_server;
		server_name example.com www.example.com;
}
```

### 简单地PHP网站配置

现在让我们来看一下Nginx如何选择location来处理一个典型的简单PHP网站的请求：

```xml
server {
		listen 80;
		server_name example.org www.example.org;
		root /data/www;

		location / {
				index index.html index.php;
		}

		location ~* \.(gif|jpg|png)$ {
				expires 30d;
		}

		location ~ \.php$ {
				fastcgi_pass localhost:9000;
				fastcgi_param SCRIPT_FILENAME
											$document_root$fastcgi_script_name;
				include fastcgi_param;
		}
}
```

无论列出的顺序如何，Nginx首先都会搜索匹配特定文字字符串的前缀的那个location。在上面的配置中，仅有的前缀配置为“/”，又因为他会匹配任意请求，所以它会被放到最后匹配。然后Nginx就会根据配置的顺序检查所有带有正则表达式的location配置，一旦匹配上，Nginx机会停止搜索并且使用这个匹配上的location。如果没有任何正则表达式匹配上这个请求，Nginx就会使用之前找到的那个前缀的location进行匹配。

需要注意的是，所有类型的location仅测试后面没有跟参数的请求行的URI部分。这样做的原因是提供的参数列表的参数总是不固定的，像如下这种情况：

```xml
/index.php?user=john&page=1
/index.php?page=1&user=john
```

此外，任何人都可以在请求的参数列表中请求任何内容：

```xml
/index.php?user=john&something+else&page=1
```

现在，让我们看一下上面的配置是如何处理接收到的请求的

- 一个/logo.gif的请求发送过来，并且首先匹配到了前缀为“/”的location，然后又匹配到了									“~ /.(gif|jpg|png)$”的正则表达式的location。因此它会被后一个location处理，使用root指令/data/www所以映射到了/data/www/logo.gif，并且该文件会被发送到客户端
- 一个/index.php的请求也会优先匹配到前缀为“/”的location，然后匹配到“~ /.php\$”。因此，它会被后一个location处理。并且请求会被传递到监听localhost:9000的FastCGI服务器上。fastcgi_param指令将FastCGI 的SCRIPT_FILENAME属性设置为了/data/www/index.php，并且FastCGI执行了这个文件。$document_root这个参数名等价于root指令，参数值$fastcgi_script_name等价于请求的URI，例如：index.php
- 一个about.html请求仅会匹配到前缀为“/”的location，因此它就会被这个location处理。使用到root指令 /data/www，请求将会被映射到/data/www/about.html，并且文件会被发送到客户端
- 处理“/”请求时比较复杂的。它仅会被前缀为“/”的location匹配。因此它会被这个location处理。然后index指令会通过它的参数index.html、index.php和root指令/data/www去验证索引文件是否存在。如果/data/www/index.html不存在，但是/data/www/index.php文件存在，Nginx会做一个内部的重定向到index.php，然后Nginx会重新开始匹配index.php就好像这个请求是客户端发过来的。最后正如我们看到的，index.php最终会被FastCGI服务器进行处理。