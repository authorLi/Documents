## 十五、Nginx是如何处理TCP\UDP会话的

一个来自客户端的TCP\UDP会话会被称作**”阶段“**的连续步骤进行处理：

*post-accept*

第一阶段是发生在接收到客户端的连接之后的。ngx_stream_realip_module模块就是在这一步调用的。

*pre-access*

对访问进行初步检查。ngx_stream_limit_conn_module模块就是在这一步调用的。

*access*

实际客户端数据处理之前的访问限制。ngx_stream_access_module在这一步被调用。

*SSL*

TLS\SSL终止。ngx_stream_ssl_module模块在这一步被调用。

*preread*

将数据的初始字节读取到预读缓冲区中，以允许ngx_stream_ssl_preread_module模块在处理数据之前对其进行分析。

*content*

强制阶段，通常实际代理到上游服务器，或将指定值返回给客户端，是实际处理数据的阶段。

*log*

记录客户端会话处理结果的最后阶段。ngx_stream_log_module模块在这一步被调用。