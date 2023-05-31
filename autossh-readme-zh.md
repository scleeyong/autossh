# autossh 版本 1.4
-------------------

# 构建和安装 Autossh
--------------------

从 1.4 版本开始，autossh 现在使用 autoconf。因此，构建过程现在是众所周知的：

	./configure
	make
	make install

请查看 autossh.host 文件以获取一个示例的包装脚本。

# 用法
-----
	autossh -M <port>[:echo_port] [-f] [SSH OPTIONS]
  
# 描述
autossh 是一个启动 ssh 的程序，并监控它，如果它死掉或停止传输流量，会重新启动它。

最初的想法和机制来自 rstunnel（可靠的 SSH 隧道）。在 1.2 版本中，方法发生了变化：autossh 现在使用 ssh 构建一个 SSH 转发的循环（从本地到远程，从远程到本地），然后发送测试数据，并期望收到回应。（这个想法要感谢 Terrence Martin。）

在 1.3 版本中，增加了一种新的方法（感谢 Ron Yorston）：可以指定一个远程回显服务的端口，该服务将回显测试数据。这避免了担心远程机器上的所有端口号冲突的拥堵和麻烦。当无法使用回显服务时，仍然可以使用循环转发的方法。

autossh 有三个自己的参数：

 -M <port>[:echo_port]，用于指定要使用的基本监视端口，或者可以选择指定监视端口和回显服务端口。
  
	当没有指定回显服务端口时，这个端口和它的上一个端口（port# + 1）不应该被其他进程使用。autossh会在基本监视端口上发送测试数据，并在上一个端口上接收返回的数据。例如，如果你指定"-M 20000"，autossh将设置转发，使其可以在端口20000上发送数据，并在20001上接收返回的数据。
	另外，可以指定一个远程回显服务的端口。如果想使用标准的inetd回显服务，这个端口应该是端口7。当指定了回显端口时，只使用指定的监视端口，并且在两个方向上传递监视消息。
	许多人禁用了回显服务，甚至禁用了inetd，所以请确保远程机器上可用该服务。有些操作系统允许指定该服务只监听本地主机（环回接口），这对于此用途足够。
	回显服务也可以是更复杂的东西：比如监控一组SSH隧道的守护进程。
	-M 0 将关闭监视功能，autossh只会在SSH退出时重新启动SSH。
	例如，如果你使用的是最近版本的OpenSSH，你可能希望尝试使用ServerAliveInterval和ServerAliveCountMax选项，使SSH客户端在发现与服务器断开连接时退出。在许多方面，这可能是比监视端口更好的解决方案。

 -f     
  在运行ssh之前，导致autossh进入后台运行。-f标志将从传递给ssh的参数中去除。需要注意的是，autossh中的-f与ssh中的-f存在一个重要的区别：当与autossh一起使用时，ssh将无法要求输入密码或密钥密码。使用-f时，"starting gate"时间（参见AUTOSSH_GATETIME）将被设置为0。

 -V     
  使autossh显示其版本号并退出。
  
autossh会将所有其他参数传递给ssh。还有一些其他的设置，但这些都是通过环境变量控制的。ssh似乎在为选项占用越来越多的字母，这似乎是避免冲突的最简单方法。

autossh尝试区分它监视的ssh进程的退出方式，并采取相应的行动。规则如下：

如果ssh进程正常退出（例如，某人在交互会话中输入"exit"），autossh将退出而不是重新启动；
如果autossh本身收到SIGTERM、SIGINT或SIGKILL信号，它会认为这是有意地发送的信号，并在杀死子ssh进程后退出；
如果autossh本身收到SIGUSR1信号，它将杀死子ssh进程并启动一个新的进程；
定期（默认为每10分钟）autossh尝试在监视转发端口上传递流量。如果失败，autossh将杀死子ssh进程（如果它仍在运行）并启动一个新的进程；
如果子ssh进程以其他任何原因死亡，autossh将尝试启动一个新的进程。
  
## 启动行为：

   - 如果第一次尝试的ssh会话以退出状态1失败，autossh将假设存在语法问题或连接设置问题，并退出而不是重试；
   - 现在有一个"starting gate"时间。如果第一个ssh进程在启动的最初几秒内失败，autossh会假设它从未离开"starting gate"，并退出。这用于处理初始身份验证失败、连接失败等情况。默认情况下，这个时间是30秒，可以进行调整（参见下面的AUTOSSH_GATETIME环境变量）。
   - 注意：如果将AUTOSSH_GATETIME设置为0，则上述两种行为都被禁用。这对于例如在启动时启动autossh非常有用。当使用-f标志启动autossh时，"starting gate"时间也被设置为0。

## 持续失败：

   - 如果ssh连接失败，并且重启尝试在快速连续的情况下失败，autossh将开始延迟尝试重启，并逐渐增加间隔，最长不超过autossh的轮询时间（通常为10分钟）。可以通过向autossh发送信号，比如SIGHUP（"kill -HUP"）来"激励"它重新尝试。

# 连接设置
-----------

由于连接必须在无人值守的情况下建立，使用autossh需要设置某种形式的自动身份验证。推荐的方法是使用ssh-agent的RSAAuthentication。示例包装脚本尝试检查当前环境是否有正在运行的代理，并在没有代理时启动一个代理。

强调一点，必须确保ssh在单独使用时正常工作，并且能够设置所需的会话，然后再尝试在autossh下运行它。

如果你正在进行隧道传输，并且使用的是不支持-N标志的旧版本ssh，建议你升级（你的版本存在安全漏洞）。如果无法升级，你可以像rstunnel一样给ssh指定一个要运行的命令，例如"sleep 99999999999"。

# 禁用连接监控
-----------------

使用监控端口值"0"（"autossh -M 0"）将禁用监控端口的使用；autossh将仅对信号和ssh进程的终止做出响应。
  
# 环境变量
------------

可以设置以下环境变量：

- AUTOSSH_DEBUG：将日志级别设置为LOG_DEBUG，如果操作系统支持，将syslog设置为将日志条目复制到stderr。
- AUTOSSH_FIRST_POLL：初始轮询时间（默认为AUTOSSH_POLL）。
- AUTOSSH_GATETIME：ssh在被视为成功连接之前必须保持运行的时间。默认为30秒。如果设置为0，则禁用此行为，并且即使第一次尝试运行ssh失败，autossh也会重试。
- AUTOSSH_LOGFILE：设置autossh使用指定的日志文件，而不是syslog。
- AUTOSSH_LOGLEVEL：日志级别，与syslog使用的级别相对应；范围为0-7，其中7最详细。
- AUTOSSH_MAXLIFETIME：设置进程在终止ssh子进程并退出之前应存在的最长时间（以秒为单位）。
- AUTOSSH_MAXSTART：指定ssh应启动的次数。负数表示不限制ssh启动的次数。默认值为-1。
- AUTOSSH_MESSAGE：将自定义消息附加到回显字符串中（最大64字节）。
- AUTOSSH_NTSERVICE：当设置为"yes"时，设置autossh以NT服务的形式在cygrunsrv下运行。如果尚未设置，则添加了ssh的-N标志，并将日志输出设置为stdout，并更改ssh退出时的行为，以便即使在正常退出时也会重新启动。
- AUTOSSH_PATH：ssh可执行文件的路径，以防与编译时的路径不同。
- AUTOSSH_PIDFILE：将autossh的进程ID写入指定的文件。
- AUTOSSH_POLL：轮询时间（以秒为单位），默认为600秒。更改此值还会更改第一次轮询时间，除非使用AUTOSSH_FIRST_POLL将其设置为其他值。如果轮询时间小于两倍的网络超时时间（默认为15秒），则网络超时时间将调整为轮询时间的一半。
- AUTOSSH_PORT：设置监控端口。主要用于防止ssh在某个时间占用了-M标志。但是由于这种可能性的存在，AUTOSSH_PORT将覆盖-M标志。 
  
# SSH选项
在使用autossh时，有两个特定的OpenSSH选项非常有用：

1）在客户端上使用ExitOnForwardFailure=yes，以确保转发成功，当autossh假设连接已正确建立时。

2）在服务器端使用ClientAliveInterval，以确保如果客户端连接关闭，则在服务器端关闭监听套接字。

日志和Syslog
autossh使用LOG_USER设施将日志记录到syslog中。您的syslog可能需要配置以接受此设施的消息。通常在/etc/syslog.conf中进行此配置。
