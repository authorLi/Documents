## 十三、使用Nginx作为HTTP负载均衡器

### 介绍

跨多个应用程序实例的负载均衡是一种优化资源利用率、最大化吞吐量、减少延迟和确保容错配置的常用技术。

可以使用Nginx作为HTTP的负载均衡器，以将流量分配到多个应用服务器，并使用Nginx改善Web应用fu'wu'qi服务器的性能，可伸缩性和可靠性。

### 负载均衡的方法

Nginx支持以下的负载均衡机制（方法）：

- 轮询——对应用服务器的请求以轮询的方式分发
- 最小链接——将下一个请求分配给活动连接数量最少的服务器
- IP哈希——使用哈希算法来确定下一个请求会被分配到那个服务器（基于客户端的IP地址）

### 默认的负载均衡策略配置

使用Nginx配置负载均衡的最简单的配置如下：

```xml
http {
		upstream myapp1 {
				server srv1.example.com;
				server srv2.example.com;
				server srv3.example.com;
		}
		
		server {
				listen 80;
				
				location / {
						proxy_pass http://myapp1;
				}
		}
}
```

在上面的例子中，有三个运行在同一应用上的实例srv1~srv3。当负载均衡方法没有特别指定时，Nginx默认使用的是轮询方法。所有请求都被代理到服务器组myapp1上，Nginx应用HTTP负载平衡来分发请求。

Nginx中的反向代理实现包括HTTP、HTTPS、FastCGI、uwsgi、SCGI、memcached和gRPC的负载均衡

想要为HTTPS配置负载均衡而不是HTTP，仅使用https协议即可

想要为FastCGI、uwsgi、SCGI、memcached和gRPC设置负载均衡只需使用各自的指令：fastcgi_pass、uwsgi_pass、scgi_pass、memcached_pass和grpc_pass即可。

### 最小连接的负载均衡

另一个负载均衡策略是最小连接策略。最小连接策略允许那些需要较长时间才能完成的情况下更公平的控制应用程序在实例上的负载

使用最小连接的负载均衡，Nginx就不会出现给让繁忙的应用服务器因过多的请求而过载的情况，而是将请求分配给不太繁忙的服务器。

当在server服务器组中配置least_conn指令时，Nginx就开启了最小连接策略

```xml
upstream myapp1 {
		least_conn;
		server.srv1.example.com;
		server.srv2.example.com;
		server.srv3.example.com;
}
```

### 会话持久性

请注意使用轮询策略和最小连接策略的负载均衡时，后续客户端的请求都可能分配给其他服务器，不能保证同一客户端总能分配到同一服务器处理。

如果需要将客户端绑定到一个特定的服务器上（换句话说就始终尝试特定服务器而言，使客户端的会话保持粘性或持久性）可以使用IP哈希的负载均衡策略。

当使用IP哈希的负载均衡策略时，会根据客户端的IP地址作为哈希的键来确定客户端的请求将由服务器组中的哪个服务器来处理。这个策略确保了同一服务器的请求总会被分配到同一服务器处理，除非此服务器是不可用的。

想要配置IP哈希这种负载均衡策略，仅仅需要添加一个ip_hash的指令到服务器组的配置中：

```xml
upstream myapp1 {
		ip_hash;
		server.srv1.example.com;
		server.srv2.example.com;
		server.srv3.example.com;
}
```

### 加权负载均衡

还可以通过服务器权重来进一步影响Nginx负载均衡算法

在上面的例子中，没有配置服务器权重，这就意味着所有指定的服务器都被视为对特定的负载均衡方法具有同等的资格。

特别是对于轮询策略，这也意味着服务器之间的请求分配大致相等——前提是有足够多的请求，并且能以统一的方式处理请求并能较快的完成请求。

挡在服务器中添加权重参数时，权重将成为负载均衡决策的一部分。

```xml
upstream myapp1 {
		server.srv1.example.com weight=3;
		server.srv2.example.com;
		server.srv3.example.com;
}
```

使用此配置，每5个新请求将按以下方式分布在应用程序实例中：三个请求到srv1中一个请求到srv2中，一个请求到srv3中。

类似的，你也可以在现在的Nginx版本中使用权重搭配最小连接策略和ip哈希策略

### 健康检查

Nginx中的反向代理实现包括里面自带的（或被动地）服务器健康检查。如果来自特定服务器的响应失败并出现错误，Nginx将会将此服务器标记为失败的服务器，并在一段时间内尽量避免为后续的请求发送到该服务器。

max_fails指令设置了在fail_timeout时间内连续不成功尝试连接服务器的次数，默认情况下max_fails被设置为1。如果它被设置为0，那么此服务器将不能提供健康检查功能。fail_timeout参数也同时指定了多长时间的失败可以把服务器定义为失败服务器。服务器故障后的fail_timeout间隔后，Nginx将开始使用实时客户端的请求来优雅的检测服务器的问题。如果检测成功，服务器会被标为活跃的服务器