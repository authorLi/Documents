## 三、控制Nginx 

Nginx可以使用信号控制。默认情况下，Nginx主进程的进程ID被写在/usr/local/nginx/logs目录下的nginx.pid文件中。你可以在配置的时候修改它的名字，或者在nginx.conf文件中使用pid指令修改。主进程支持以下几种信号：

```xml
TEAM,INT					快速关机
QUIT							优雅的关机
HUP								改变配置，紧接着修改时区（仅供FreeBSD和Linux系统），用新配置开启一个新的工作进程，优雅									的关闭老的工作进程
USR1							重新打开日志文件
USR2							升级可执行文件
WINCH							优雅的关闭工作进程
```

虽然不需要，单独的工作进程也可以使用信号控制。支持的信号如下所示：

```xml
TEAM,INT					快速关闭
QUIT							优雅的关闭
USR1							重新打开日志文件
WINCH							调试异常终止（需要开启debug断点）
```

### 改变配置

为了Nginx能够重新读取配置文件，需要发送一个HUP信号给Nginx的主进程。主进程先会检查语法异常，然后尝试应用新的更改配置，那就是，打开日志文件和监听新的端口。如果失败了，它会回滚配置并继续使用老的配置文件。如果成功了，它会开启一个新的工作进程并且会通知老的工作进程使其优雅的关闭。老的工作进程会关闭监听的端口并继续服务于老的客户端。当所有的老客户端请求处理完毕，老的工作工作进程就会关闭。

让我们举例说明，想象一下Nginx运行在FreeBSD，运行以下命令：

```xml
ps -aw -o pid,ppid,user,%cpu,vsz,wchan,command | egrep '(nginx|PID)'
```

会出现：

```xml
  PID  PPID USER    %CPU   VSZ WCHAN  COMMAND
33126     1 root     0.0  1148 pause  nginx: master process /usr/local/nginx/sbin/nginx
33127 33126 nobody   0.0  1380 kqread nginx: worker process (nginx)
33128 33126 nobody   0.0  1364 kqread nginx: worker process (nginx)
33129 33126 nobody   0.0  1364 kqread nginx: worker process (nginx)
```

如果发送HUP信号到Nginx主进程，那么输出会变为：

```xml
  PID  PPID USER    %CPU   VSZ WCHAN  COMMAND
33126     1 root     0.0  1164 pause  nginx: master process /usr/local/nginx/sbin/nginx
33129 33126 nobody   0.0  1380 kqread nginx: worker process is shutting down (nginx)
33134 33126 nobody   0.0  1368 kqread nginx: worker process (nginx)
33135 33126 nobody   0.0  1368 kqread nginx: worker process (nginx)
33136 33126 nobody   0.0  1368 kqread nginx: worker process (nginx)
```

其中一个PID为33219的老工作进程仍然会继续工作，但一段时间后它就退出了：

```XML
  PID  PPID USER    %CPU   VSZ WCHAN  COMMAND
33126     1 root     0.0  1164 pause  nginx: master process /usr/local/nginx/sbin/nginx
33134 33126 nobody   0.0  1368 kqread nginx: worker process (nginx)
33135 33126 nobody   0.0  1368 kqread nginx: worker process (nginx)
33136 33126 nobody   0.0  1368 kqread nginx: worker process (nginx)
```

### 轮转日志文件

为了去轮转日志文件，它们首先需要被重命名。在那之后，发送需要发送USR1信号到Nginx的主进程中。主进程就会重新打开所有当前已经打开的日志文件，并为它们在运行着的工作进程下分配一个非特权的用户，作为其拥有者。在成功重新打开后，主进程会关闭所有打开的文件，并且发送信息给工作进程命令它们都重新打开文件。工作进程会立即关闭老的文件并打开新文件。这样做的结果是，老的文件几乎可以立即用于后期处理，例如压缩。

### 快速升级可执行文件

为了升级服务的可执行性，应该首先用新的可执行文件替换老的可执行文件。在那之后，USR2信号会被发送到Nginx主进程中。首先会先将其具有进程ID的文件重命名为带有.oldbin后缀的新文件，例如：/usr/local/nginx/logs/nginx.oldbin，然后启动一个新的可执行文件，该文件依次启动新的工作进程：

```xml
  PID  PPID USER    %CPU   VSZ WCHAN  COMMAND
33126     1 root     0.0  1164 pause  nginx: master process /usr/local/nginx/sbin/nginx
33134 33126 nobody   0.0  1368 kqread nginx: worker process (nginx)
33135 33126 nobody   0.0  1380 kqread nginx: worker process (nginx)
33136 33126 nobody   0.0  1368 kqread nginx: worker process (nginx)
36264 33126 root     0.0  1148 pause  nginx: master process /usr/local/nginx/sbin/nginx
36265 36264 nobody   0.0  1364 kqread nginx: worker process (nginx)
36266 36264 nobody   0.0  1364 kqread nginx: worker process (nginx)
36267 36264 nobody   0.0  1364 kqread nginx: worker process (nginx)
```

在那之后所有的工作进程（老的和新的）都会继续接受请求。如果WINCH信号被发送到第一个主进程，他将会发送信息到他的工作进程命令他们优雅的关闭，然后它们就会开始陆续退出：

```xml
  PID  PPID USER    %CPU   VSZ WCHAN  COMMAND
33126     1 root     0.0  1164 pause  nginx: master process /usr/local/nginx/sbin/nginx
33135 33126 nobody   0.0  1380 kqread nginx: worker process is shutting down (nginx)
36264 33126 root     0.0  1148 pause  nginx: master process /usr/local/nginx/sbin/nginx
36265 36264 nobody   0.0  1364 kqread nginx: worker process (nginx)
36266 36264 nobody   0.0  1364 kqread nginx: worker process (nginx)
36267 36264 nobody   0.0  1364 kqread nginx: worker process (nginx)
```

一段时间后，就只有新的工作进程在处理请求了：

```xml
  PID  PPID USER    %CPU   VSZ WCHAN  COMMAND
33126     1 root     0.0  1164 pause  nginx: master process /usr/local/nginx/sbin/nginx
36264 33126 root     0.0  1148 pause  nginx: master process /usr/local/nginx/sbin/nginx
36265 36264 nobody   0.0  1364 kqread nginx: worker process (nginx)
36266 36264 nobody   0.0  1364 kqread nginx: worker process (nginx)
36267 36264 nobody   0.0  1364 kqread nginx: worker process (nginx)
```

需要注意的是老的主进程并不会关闭他监听的端口，它甚至可以在需要的情况下重新运行它的工作进程。如果由于某些原因这个新的可执行文件无法正常工作，可以进行如下操作

- 发送HUP信号给老的主进程。老的主进程将在不读取新配置文件的情况下开启新的工作进程。在那之后，所有新的进程可以通过发送QUIT信号给新的主进程被优雅的关闭。
- 发送TEAM信号给新的主进程。它将会发送信息给它的工作进程让他门立即退出，然后它们几乎都会立即退出（如果新的进程由于某些原因并不存在，需要发送KILL信号给这些工作进程以让它们强制退出）。当新的主进程退出老的主进程将会自动创建新的工作进程。

如果新的主进程退出，老的主进程会丢弃带有进程ID的后缀为.oldbin的后缀以恢复原来的状态。

如果升级成功，也需要发送QUIT信号给老的主进程，并且只有新的进程会存活下来：

```xml
  PID  PPID USER    %CPU   VSZ WCHAN  COMMAND
36264     1 root     0.0  1148 pause  nginx: master process /usr/local/nginx/sbin/nginx
36265 36264 nobody   0.0  1364 kqread nginx: worker process (nginx)
36266 36264 nobody   0.0  1364 kqread nginx: worker process (nginx)
36267 36264 nobody   0.0  1364 kqread nginx: worker process (nginx)
```





