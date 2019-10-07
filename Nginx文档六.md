## 六、debugging 日志（调试日志）

为了启用调试日志功能，Nginx需要在构建时配置支持debugging：

```xml
./configure --with-debug ...
```

然后需要使用error_log指令设置debug等级

```xml
error_log /paht/to/log debug;
```

为了证实Nginx已经启用了debugging支持，运行nginx -v命令如下：

```xml
configure arguments： --with-debug
```

预先构建的Linux软件包提供了开箱即用的调试日志支持，运行如下命令可以启用nginx-debug二进制文件（1.9.8）

```xml
service nginx stop
service nginx-debug start
```

然后设置debug等级。Windows上的nginx二进制版本总是支持调试日志。所以仅仅设置debug等级即可。

注意：在没有指定debug等级的情况下重新定义日志将不能启用调试日志。在下面的例子中，在服务器级别上重新定义日志将不能给这个服务器提供调试日志功能：

```xml
error_log /path/to/log dubug;
http {
		server {
				error_log /path/to/log debug;
		}
}
```

为了避免这种情况，要么重新定义日志的行应该被注释掉，要么debug还应添加级别规范：

```xml
error_log /path/to/log debug;
http {
		server {
				error_log /path/to.log debug;
		}
}
```

### 选定客户端的调试日志

也可以仅对特定的客户端地址进行调试日志：

```xml
error_log /path/to/log;

events {
		debug_connection 192.168.1.1;
		debug_connection 192.16.10.0/24;
}
```

### 记录到循环内存缓冲区

调试日志可以被写到循环内存缓冲区：

```xml
error_log memory:32m debug;
```

debug即使在高负载下，在该级别上登录到内存缓冲区也不会对性能产生重大影响。在这种情况下，可以用gdb脚本来提取日志：

```xml
set $log = ngx_cycle->log

while $log->write != ngx_memory_writer
		set $log = $log->next
end

set $buf = (ngx_log_memory_buf_t *) $log->wdata
dump binary memory debug_log.txt $bug->start $bug->end
```

