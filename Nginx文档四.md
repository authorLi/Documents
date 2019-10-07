## 四、连接处理方法

Nginx支持多种连接处理方法。特定方法的可用性依赖于使用的平台。在支持多种方法的平台上，Nginx会自动选择效率最高的方法。然而，如果你需要的话，你可以使用use指令明确的指定连接处理方法。

Nginx支持如下的几种连接处理方法：

- select——标准方法。支持模块会在缺少更高效的方法的平台上自动构建。可以使用--with-select_module和  --without-select_module配置参数强制启用和不启用构建此模块
- poll——标准方法。支持模块会在缺少更高效的方法的平台上自动构建。可以使用--with-poll_module和--without-poll_module配置参数强制启用和不启用构建此模块
- kqueue——应用于FreeBSD 4.1+，OpenBSD 2.9+，NetBSD 2.0和Mac OS的更高效的方法
- epoll——应用于Linux 2.6+ 的更高效的方法
- /dev/poll——应用于Solaris 7 11/99+，HP/UX 11.22+ （eventpoint），IRIX 6.5.15+ Tru64 UNIX 5.1A+ 的更高效的方法
- eventport——事件端口，应用于Solaris 10+ 的方法（由于已知问题，推荐使用/dev/poll方法代替）