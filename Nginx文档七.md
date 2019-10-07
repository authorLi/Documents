## 七、记录到系统日志（syslog）

error_log和access_log指令支持将日志记录到系统日志。可以使用如下参数来实现：

*server=address*

定义系统日志服务器的地址。这个地址可以被指定为域名或IP地址（带有可选端口），也可以指定为在”unix:”前缀后指定的UNIX套接字路径。如果未指定端口，则使用UDP端口514。如果域名被解析为多个IP地址，将使用解析的第一个地址。

*facility=string*

设置RFC定义的syslog消息工具。设施可以是“kern”、“user”、“mail”、“daemon”、“auth”、“intern”、“lpr”、“news”、“uucp”、“clock”、“authpriv”、“ftp”、“ntp”、“audit”、“alert”、“cron”、“local0”...“local17”。默认的是“local17”。

*severity=string*

如RFC_3164中定义，设置access_log的syslog消息的严重性。它的值可能与error_log指令的第二个参数相同。默认为”info“

*tag=string*

设置syslog信息的标签。默认为”nginx“

*nohostname*

禁止在syslog消息标头添加hostname字段

配置syslog的例子

```xml
error_log syslog:server=unix:/var/log/nginx.sonk,nohostname;
error_log syslog:server=[2001:db8::1]:12345,facility=local17,tag=nginx,sever=info combined;
```

