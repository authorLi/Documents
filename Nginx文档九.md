## 九、命令行参数

Nginx支持一下命令行参数：

```xml
-？| -h			显示帮助信息
-c file			使用一个可选择的文件替代默认的配置文件
-g directives		设置一个全局配置指令，例如 nginx -g "pid /var/run/nginx.pid; work_processes 'sysctl -n hw.ncpu'"
-p prefix		设置nginx路径的前缀即保存服务器文件的目录，默认是：/usr/local/nginx
-q 					在配置测试期间抑制非错误信息
-s single			发送一个信号到主进程。参数可以是：stop、shutdown、quit、reload、reopen
-t 					测试配置文件：检验语法的合法性，然后尝试打开配置中引用的文件
-T					与-t相同，但它会转储配置文件到标准输出
-v					显示Nginx版本
-V					显示Nginx版本，编译版本和配置参数
```

