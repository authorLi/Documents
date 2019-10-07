## 十二、服务器名称

服务器的名称可以使用server_name这个指令定义，它被定义在server块中用于确定请求使用哪个server块进行处理。这些服务器名可以被定义为确定的名称、带有通配符的名称和带有正则表达式的名称。

```xml
server {
		listen 80;
		server_name example.org www.example.org;
}

server {
		listen 80;
		server_name *.example.org;
}

server {
		listen 80;
		server_name mail.*;
}

server {
		listen 80;
		server_name ~^(?<user>.+)\.example\.net$;
}
```

当通过名称搜索虚拟服务器时，当名称匹配到多种变体时，例如既匹配到了通配符名称又匹配到了正则表达式名称将会根据以下顺序选择第一个匹配到的服务器：

```xml
1.确定的名臣
2.以*开头的最长的通配符名称，例如：*.example.org
3.以*结尾的最长的正则表达式名称，例如：mail.*
4.第一个匹配到的正则表达式的配置（按照它们在配置文件中出现的顺序）
```

### 通配符名称

包括*的通配符名称的\*只能出现在名称的开头或结尾或点边界(.)附近。像“www.\*.example.org”和“ww\*.example.org”是无效的。然而这些名称可以使用正则表达式被指定，例如：“~^www\\..+\\.example\\.org$”和“~^w.\*\\.example\\.org\$”。\*可以匹配多个名称部分。名称为“\*.example.org”不仅可以匹配“www.example.org”还可以匹配“www.sub.example.org”

可以使用“\*.example.org”形式的特殊通配符名称来匹配到确定的名称“example.org”和通配符名称“\*.example.org”

### 正则表达式名称

Nginx使用的正则表达式与Perl语言（PCRE）使用的正则表达式相兼容。要使用正则表达式，必须要以波浪号开头。

```xml
server_name ~^www\d+\.example\.net$;
```

否则它会被认为是一个确定的名称，或者如果表达式里包含一个*，则可能被认为是一个通配符名称。不要忘了加“^”和“$”来确定正则表达式的范围，虽然这两个符号在语法上是不必须的，但在逻辑上是必须的。同样要注意域名的点（.）前面需要加一个反斜线（\）来转义。包含字符“{”和“}”的正则表达式需要加引号

```xml
server_name "~^(?<name>\w\d{1,3}+)\.example\.net$";
```

否则Nginx将无法启动并报出以下错误：

```xml
directive "server_name" is not terminated by ";" in ...
```

正确命名的正则表达式捕获后可以在以后用作变量：

```xml
server {
		server_name ~^(www\.)?(?<domain>.+)$;
    
    location / {
      		root /sites/$domain;
      }
}
```

PCRE类库使用以下语法支持命名捕获：

```xml
?<name>			Perl 5.10兼容语法，PCRE-7.0版本开始支持
?'name'			Perl 5.10兼容语法，PCRE-7.0版本开始支持
?P<name>		Python 兼容语法，PCRE-5.0版本开始支持
```

如果Nginx没有成功启动并且报出以下错误：

```xml
pcre_compile() failed: unrecognized character after (?< in ...
```

这就意味着的你的PCRE类库的版本太老了并且"?P<name>"参数需要被替换，也可以使用数字形式捕获：

```xml
server {
		server_name ~^(www\.)?(.+)$;
		
		location / {
			root /sites/$2;
		}
}
```

然而这种用法只能用于简单的情况（像上面这样），因为数字引用很容易被重写。

### 杂项名称

有些服务器名称经过特殊处理

如果要求处理服务器块中没有”Host”字段并且不是默认值的请求，则需要定义一个空的服务器名：

```xml
server {
		listen 80;
		server_name example.org www.example.org;
}
```

如果server块中没有定义server_name指令，那么Nginx将使用空的名称作为服务器名称。

在这种情况下，直到0.8.48版本的Nginx都是用主机名作为服务器名称

如果服务器名称定义为“$hostname”（0.9.4），则使用主机名作为服务器名称

如果有人使用IP地址而不是服务器名称进行请求，”Host“请求标头字段将包含IP地址并且可以使用IP地址作为服务器名称来处理请求

```xml
server {
		listen 80;
		server_name example.org
								www.example.org
								""
								192.168.1.1
								;
}
```

在匹配所有的例子中，可以看到一个奇怪的名称 ”_“

```xml
server {
		listen 80 default_server;
		server_name _;
		return 444;
}
```

其实这个名称没什么特殊的，它只是无数从未与任何真实名称相交叉的无效域之一。也同样可以使用其他无效域”——“或”!@#“。

直到Nginx0.6.25版本都支持这个可能被认为是匹配所有的”*“号。它从不用作匹配所有的通配符名称。相反它提供了现在server_name_in_redirect指令提供的功能。现在不推荐使用特殊服务器名称”\*”，推荐使用server_name_in_redirect指令。注意不能使用server_name指令去指定匹配所有的服务器名称或默认服务器。那是listen指令的属性而不是server_name的属性。倒是可以将listen指令设置为监听“\*:80”或“\*:8080”，并指定其中一个是端口“\*:8080”的默认服务器，另一个是监听“\*:80”端口的默认服务器

```xml
server {
		listen 80;
		listen 8080 default_server;
		server_name example.net;
}

server {
		listen 80 default_server;
		listen 8080;
		server_name example.org;
}
```

### 国际化名称

应在server_name指令中使用ASCII（Punycode）表示形式指定国际化域名（IDNs）：

```xml
server {
		listen 80;
		server_name xn--elafmkfd.xn--80akhbyknj4f;
}
```

### 优化

确定的名称、以\*开头的通配符名称和以*结尾的正则表达式名称被存储在绑定监听端口的三个哈希表中。哈希表的大小在配置阶段进行了优化，以便可以和找到CPU缓存未命中的最少的名称。

首先会到存储确定名称的哈希表中进行搜索，如果没有找到，则会去以\*开头的通配符名称哈希表中去搜索，如果还没找到，则去以\*结尾的正则表达式名称的哈希表中去搜索。

搜索通配符名称的哈希表要比搜索确定名称的哈希表要慢，因为名称是通过域名部分进行搜索的。要注意“.example.org”这个名称是存储在通配符名称的哈希表中而不是确定名称的哈希表中。

正则表达式是按照顺序测试的，因此搜索正则表达式名称的哈希表是最慢的，并且效率是不可估量的。

因为以上三个原因，如果可能的话最好使用确定的名称。例如，如果最常访问的服务器名为“example.org”或“www.example.org”，明确的进行以下定义可以试试效率更高一些：

```xml
server {
		listen 80;
		server_name example.org www.example.org *.example.org;
}
```

这样的定义比只在server_name中定义一个*.example.org更高效。

如果定义了大量的服务器名称，又或者定义了异常长的服务器名称。则很有必要调节http块中的server_names_hash_max_size和server_names_bucket_max_size这两个指令。关于server_names_bucket_max_size指令的默认值，它可能是32、64或其他值，它的默认值由CPU缓存行的大小而定。如果默认值是32，并且服务器名称被定义为“too.long.server.name.example.org”，那么Nginx将不能启动，并报出以下错误：

```xml
could not build the server_names_hash,
you should increase server_names_bucker_max_size: 32
```

因为这个原因，你应该把server_names_bucket_max_size这个指令的值增加到下一个2的幂次方

```xml
http {
		server_names_bucket_max_size 64;
}
```

如果定义了大量的服务器名称，另一个错误也可能会报出来：

```xml
could not build server_names_hash,
you should increase either server_names_hash_max_size: 512
or server_names_bucket_max_size: 32
```

为了解决这个问题，你可以先尝试去设置server_names_hash_max_size指令的值为接近服务器名称数量的值。仅在如果这个方法不奏效或者Nginx的启动时间过长时再去增加server_names_bucket_max_size指令的值。

如果一个服务器是监听这个端口的唯一服务器，这样的话Nginx将不会在去测试服务器名称（更不会去构建监听此端口的服务器名的哈希表）。然而这种情况有一个例外，如果这个唯一的服务器名称是一个捕获的正则表达式，则Nginx必须要执行该正则表达式以进行捕获。

### 兼容性

- 0.9.4版本后支持“$hostname”这样的特殊的服务器名称
- 0.8.48版本以后支持默认的服务器名为空即为“”
- 0.8.25版本以后支持以捕获的正则表达式命名服务器名
- 0.7.40版本以后支持正则表达式捕获
- 0.7.12版本以后支持服务器名为空即为“”
- 0.6.25版本以后支持使用通配符名称和正则表达式名称作为首选服务器名称
- 0.6.7版本以后支持正则表达式名称
- 0.6.0版本以后支持使用形如“example.*”的通配符名称
- 0.3.18版本以后支持使用形如“.example.org”的特殊服务器名称
- 0.1.13版本以后支持形如“*.example.org”的通配符名称