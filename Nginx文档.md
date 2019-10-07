# Nginx文档

## 一、从源码中编译

### configure命令

构建使用的是configure命令。它定义了系统的许多方面，包括nginx在连接过程中被允许使用的方法。最后他会创建一个makefile。

configure命令支持以下参数：

--help：显示帮助信息

--prefix=path：定义了保留服务器文件的目录，即定义安装路径。这个目录也会被configure设置为相对路径并且在nginx.conf配置文件中。它设置*/usr/local/nginx*为默认安装目录。

--sbin-path=path：为nginx定义一个可执行文件的名称。这个名称只在安装时起作用。默认名是：prefix/sbin/nginx

--modules-path=path：定义了nginx的动态模块安装的路径。默认路径是 prefix/modules

--config-path=path：为一个nginx.conf配置文件命名。如果需要这么做，那么你就可以编写不同的nginx配置文件，你可以通过在命令参数-c file说明它。它的默认名为 prefix/config/nginx.conf

