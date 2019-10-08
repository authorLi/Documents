## 十四、配置HTTPS服务器

想要配置HTTPS服务器，必须要在server块中的监听套接字上启动该参数，并应指定服务器证书和私钥文件的位置:

```xml
server {
		listen 443 ssl;
		server_name www.example.com;
		ssl_certificate www.example.com.crt;
		ssl_certificate_key www.example.com.key;
		ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
		ssl_ciphers HIGH:!aNULL:!MD5;
}
```

服务器证书是公共实体。它被送到每个链接服务器的客户端。私钥是一个安全实体，他应该被保存在一个访问受限的文件中。但是，私钥文件对于Nginx的主进程必须是可读的。私钥也可以与证书保存在同一文件中。

```xml
ssl_certificate www.example.com.cert;
ssl_certificate_key www.example.com.cert;
```

在这种情况下文件的访问也应该受到严格的限制。尽管证书和私钥被放到同一文件中，也仅仅只有证书会被发送到客户端。

指令ssl_protocols和ssl_ciphers可用于限制链接，使其仅包括SSL/TLS的强版本和密码。默认情况下，ssl_protocols指令的值为”TLSv1 TLSv1.1 TLSv1.2“，而ssl_ciphers指令的值为”HIGH!:aNULL!:MD5“，所以不需要对它们进行明确的配置。注意，这些指令的默认值有时会被修改多次。

### HTTPS服务器优化

SSL操作会额外的消耗CPU的资源，在多处理器系统上，应该同时运行多个工作进程，不可少于CPU内核的数量。最消耗CPU的就是在SSL握手时。有两种方式可以最小化每个客户端的这些操作的数量。第一种是通过启用keepalive用一个链接来发送多个请求，第二种是重用SSL会话参数以避免SSL用于并行连接和后续链接。这个会话存储在工作进程之间相互共享的会话缓存中，并且可以通过ssl_session_cache指令进行配置。一兆字节的缓存大约包含4000个会话。默认的缓存超时时间为5分钟，可以使用ssl_session_timeout指令来对超时时间进行修改。这有一个为有十兆字节会话缓存的多核系统进行优化的配置：

```xml
worker_processes auto;
http {
		ssl_session_cache shared:SSL:10m;
		ssl_session_timeout 10m;
		
		server {
				listen 443 ssl;
				server_name www.example.com;
				keepalive_timeout 70;
				ssl_certificate www.example.com.crt;
				ssl_certificate_key www.example.com.key;
				ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
				ssl_ciphers HIGH:!aNULL:!MD5;
		}
}
```

### SSL证书链

一些浏览器可能会抱怨由知名证书颁发机构签署的证书，而其他浏览器可能会毫无理由的接受这些证书。发生这种情况的原因是颁发机构已使用中间证书对服务器证书进行了签名，该中间证书在随特定浏览器分发的众所周知的受信任颁发证书颁发机构的证书库中不存在。在这种情况下，授权机构将提供一束链接的证书，这些证书应与已签名的服务器证书串联在一起。服务器证书必须出现在组合文件中的链接证书之前。

```xml
$ cat www.example.com.crt bundle.crt > www.example.com.chained.crt
```

最终文件应在ssl_certificate指令中使用

```xml
server {
		listen 443 ssl;
		server_name www.example.coml;
		ssl_certificate www.example.com.chained.crt;
		ssl_certificate_key www.example.com.key;
}
```

如果服务器证书与分发包的连接顺序错误，那么Nginx将无法正常启动，并显示如下结果信息：

```xml
SSL_CTX_use_PrivateKey_file(".../www.example.com.key")failed
	(SSL: error:0B080074:x509 certificate routines:
    X509_check_private_key:key values mismatch)
```

因为Nginx尝试将私钥与捆绑包的第一个证书一起使用而不是服务器的证书。

浏览器通常会存储它们接收到的并且被受信任的专业机构签名的中间证书。所以你活跃使用的浏览器可能已经具有所需要的那些中间证书，并且也许就不会抱怨没有链式捆绑包发出的证书。为了确保服务器发送出了完整的证书链可以使用openssl的命令，如下例：

```xml
$ openssl s_client -connect www.godaddy.com:443
...
Certificate chain
 0 s:/C=US/ST=Arizona/L=Scottsdale/1.3.6.1.4.1.311.60.2.1.3=US
     /1.3.6.1.4.1.311.60.2.1.2=AZ/O=GoDaddy.com, Inc
     /OU=MIS Department/CN=www.GoDaddy.com
     /serialNumber=0796928-7/2.5.4.15=V1.0, Clause 5.(b)
   i:/C=US/ST=Arizona/L=Scottsdale/O=GoDaddy.com, Inc.
     /OU=http://certificates.godaddy.com/repository
     /CN=Go Daddy Secure Certification Authority
     /serialNumber=07969287
 1 s:/C=US/ST=Arizona/L=Scottsdale/O=GoDaddy.com, Inc.
     /OU=http://certificates.godaddy.com/repository
     /CN=Go Daddy Secure Certification Authority
     /serialNumber=07969287
   i:/C=US/O=The Go Daddy Group, Inc.
     /OU=Go Daddy Class 2 Certification Authority
 2 s:/C=US/O=The Go Daddy Group, Inc.
     /OU=Go Daddy Class 2 Certification Authority
   i:/L=ValiCert Validation Network/O=ValiCert, Inc.
     /OU=ValiCert Class 2 Policy Validation Authority
     /CN=http://www.valicert.com//emailAddress=info@valicert.com
...
```

当使用SNI测试配置参数时，使用-servername选项是非常重要的，因为openssl默认情况下不适用SNI

在这个例子中，www.GoDaddy.com的服务器证书#0的主题（”s“）是由发行人（”i“）签署的，这个发行人就是证书#1的主题。签署证书#1的正是证书#2的主题。证书#2是由知名的发行人ValiCert，Inc进行签署的，这些证书会被存储在浏览器的内置证书库中。

如果没有添加证书捆绑包，则仅显示服务器证书#0。

### 单个HTTP/HTTPS服务器

可以配置一个同时处理HTTP和HTTPS请求的服务器：

```xml
server {
		listen 80;
		listen 443 ssl;
		server_name www.example.com;
		ssl_certificate www.example.com.cert;
		ssl_certificate_key www.example.com.key;
}
```

如上所示，0.7.14SSL之前，无法为各个监听套接字选择性的启用SSL。尽可以使用ssl指令为整个服务器启用SSL，从而无法设置单个HTTP/HTTPS服务器。通过在listen指令中添加ssl参数才解决了这个问题。因此不建议在现代版本中使用ssl参数

### 基于名称的HTTPS服务器

当配置两个或多个服务器监听同一端口时会有一个常见的问题。

```xml
server {
		listen 443 ssl;
		server_name www.example.com;
		ssl_certificate www.example.com.crt;
		...
}

server {
		listen 443 ssl;
		server_name www.example.org;
		ssl_certificate www.example.org.crt;
		...
}
```

应用以上配置浏览器会接受到默认服务器的证书，即www.example.com与请求的服务器名称无关。这是由于SSL协议行为引起的。在浏览器发送HTTP请求之前SSL连接就已经被创建了，并且Nginx不知道请求的服务名。因此，它仅仅会提供一个默认服务器的证书。

解决此问题的最古老的最可靠的方法就是为每个单独的HTTPS服务器分配一个单独的IP地址：

```xml
server {
		listen 192.168.1.1:443 ssl;
		server_name www.example.com;
		ssl_certificate www.example.com.crt;
		...
}

server {
		listen 192.168.1.2:443 ssl;
		server_name www.example.org;
		ssl_certificate www.example.org.crt;
		...
}
```

### 具有多个名称的SSL证书

现在也有其他方法实现多个HTTPS服务器共享一个IP地址。但是他们都有缺点，一种方法是在SubjectAltName证书字段中使用具有多个名称的证书，例如www.exmaple.com和www.example.org。然而SubjectAltName字段的长度是有限制的。

另一种是使用具有通配符名称的证书，例如，*.exmaple.org。通配符证书可保证所有指定域的所有子域，但只能在一个级别上。这个证书会匹配www.example.org和www.sub.example.org。

这两种方法也可以结合使用。一个整数在SubjectAltName字段中可以包含明确的或具有通配符的名称，例如example.org或*.example.org。

最好将带有多个名称的证书文件和它的秘钥文件放在配置的http级别，以在所有服务器中继承其单个内存副本：

```xml
ssl_certificate common.crt;
ssl_certificate_key common.key;

server {
		listen 443 ssl;
		server_name www.example.com;
}

server {
		listen 443 ssl;
		server_name www.example.org;
}
```

### 服务器名称指示

多个HTTPS服务器共享一个IP地址的更常用的方法是TLS服务器名称指示扩展名(SNI)。它允许浏览器在SSL握手期间就传递请求的服务器名称。因此服务器将知道哪个是用于连接的证书。大多数现代服务器一般都支持SNI，尽管某些旧客户端或特殊客户端不支持SNI。

仅仅能在SNI中传递域名。但是如果请求包含文字IP地址，则某些浏览器会错误地传递服务器的IP地址作为它的名称。

为了在Nginx中使用SNI，必须在构建Nginx二进制文件的OpenSSL库以及在运行时动态链接到的库都受支持。如果OpenSSL使用配置选项--enable-tlsext构建，则从0.9.8f都支持SNI。自从0.9.8j版本开始，都默认启用这个选项。如果Nginx构建时是启用了支持SNI的，则当使用”-V“指令时，Nginx将显示：

```xml
$ nginx -V
	...
	TLS SNI support enabled
	...
```

但是如果启用了SNI的Nginx动态链接到不支持SNI的OpenSSL类库，则Nginx将显示以下警告：

```xml
nginx was built with SNI support，however，now it is linked
dynamically an OpenSSL library which has no tlsext support，
therefore SNI is not available
```

### 兼容性

自从0.8.21和0.7.62版本开始，”-V“指令可以显示SNI的状态

0.7.14版本开始listen的ssl指令就一直是支持的。0.8.21版本之前它都是只能与default指令同时使用的

0.5.23版本开始就支持SNI了

0.5.6版本之后才开始支持SSL会话缓存共享

1.9.1版本之后，默认的SSL协议为TLSv1、TLSv1.1、TLSv1.2（如果支持OpenSSL类库的话）

0.7.65和0.8.18版本之前，默认的SSL协议是SSLv2、TLSv1、TLSv1.1、TLSv1.2（如果支持OpenSSL类库的话）

0.7.64和0.8.17版本之前，默认的SSL协议是SSLv2、SSLv3、TLSv1

1.0.5版本之后SSL的默认密码为”HIGH:!aNULL:!MD5“

0.7.65和0.8.20版本之后SSL默认密码为”HIGH:!ADH:!MD5“

0.8.19版本默认的SSL密码是”ALL:!ADH:RC4+RSA:+HIGH:+MEDIUM“

0.7.64和0.8.18版本之前SSL的默认密码是”ALL:!ADH:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP“