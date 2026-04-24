那么介绍下今天的主角，内核 Netfilter 框架conntrack状态机制，也可以称为它是一种连接跟踪机制。

![图片](https://mmbiz.qpic.cn/mmbiz_png/H9YjqVWhgmvT2eGOZSGAnHfSWbscMVRdS8mHjm3tWrd8g1XJyzKvGytfiaoO9jL9uhsa0cP8AqcUvfPDkkmrDJA/640?wx_fmt=png&from=appmsg&randomid=xnder9u9&watermark=1&tp=webp&wxfrom=5&wx_lazy=1)

conntrack连接跟踪，实际上是一个内核模块 `nf_conntrack` ，可以使用命令 `lsmod | grep nf_conntrack` 来检查内核模块是否加载。

在加载 nf_conntrack模块后，连接跟踪机制就开始工作，它会判断每个经过的数据包是否属于已有的连接，如果不属于任何已存在的连接，那么 nf_conntrack模块就会为其新建一个 conntrack 条目，如果连接已经存在，则更新对应 conntrack 条目的状态、老化时间等信息。

在iptables里，数据包在连接跟踪连接中有五种不同状态：

- `NEW`**：**尝试建立一个新的连接。
    
- `ESTABLISHED`**：**已经存在的连接。
    
- `RELATED`**：**分配给一个正在发起新连接的数据包。当一个连接和某个已处于ESTABLISHED状态的连接有关系时，就被认为是RELATED的了。换句话说，一个连接要想是RELATED的，首先要有一个ESTABLISHED的连接。这个ESTABLISHED连接再产生一个主连接之外的连接，这 个新的连接就是RELATED的了，当然前提是conntrack模块要能理解RELATED。
    
- `INVALID`**：**报文是无效的，例如它不符合TCP状态图。
    
- `UNTRACKED`**：**一种特殊状态，可以由管理员分配，可以绕过对特定数据包的连接跟踪。
    

连接跟踪模块**目前只支持以下六种协议**：`TCP`、 `UDP`、 `ICMP`、 `DCCP`、`SCTP`、`GRE`。

![图片](https://mmbiz.qpic.cn/mmbiz_png/H9YjqVWhgmvT2eGOZSGAnHfSWbscMVRdAkFwjuspTOpySmR7a68unO4vdqv7xlEXq6erjeykCictWchEJEeHmJg/640?wx_fmt=png&from=appmsg&randomid=1edzysor&watermark=1&tp=webp&wxfrom=5&wx_lazy=1)

大家可以根据**Netfilter框架数据流图**加深理解，图中名为 conntrack 的方块则代表conntrack钩子函数。

除了本地产生的包由OUTPUT链处理外，所有连接跟踪都是在PREROUTING链里进行处理的，意思就是： iptables会在PREROUTING链里重新计算所有的状态。如果我们发送一个流的初始化包，状态就会在OUTPUT链里被设置为`NEW`，当我们收到回应的包时，状态就会在PREROUTING链里被设置为`ESTABLISHED`。

如果第一个包不是本地产生的，那就会在PREROUTING链里被设置为`NEW`状态。综上，所有状态的改变和计算都是在nat表中的PREROUTING链和OUTPUT链里完成的。

![图片](https://mmbiz.qpic.cn/mmbiz_png/H9YjqVWhgmvT2eGOZSGAnHfSWbscMVRdM7kJOzdxYBwBP1YhCZRutNMeaPyXNwDhKlOd0ukcbLbgXKxNy1j2iaA/640?wx_fmt=png&from=appmsg&randomid=ieahd9an&watermark=1&tp=webp&wxfrom=5&wx_lazy=1)

好啦，我们已经把理论知识搞明白了，那就实战。不服就干！

## conntrack工具使用

conntrack在用户态中也有一套命令程序，使用 `conntrack` 命令程序，需要安装conntrack-tools 安装包。

conntrack-tools包括：conntrack、conntrackd、nfct。

- conntrack：从用户空间查看和管理内核的连接跟踪表
    
- conntrackd：可以用在热备场景下在多台机器直接实时同步连接跟踪表。
    
- nfct：目前只支持nfnetlink_cttimeout。从长远来看，希望通过提供类似于nftables的语法来替代conntrack。
    

下面使用 `conntrack` 命令跟踪服务端IP `172.27.247.29`，端口是 `80`的网络连接请求。

我们可以使用如下参数来进行过滤

`conntrack -E -d 172.27.247.29 -p tcp --dport 80   `

下面将conntrack所跟踪的日志，进行解析

_图1_

![图片](https://mmbiz.qpic.cn/mmbiz_png/H9YjqVWhgmvT2eGOZSGAnHfSWbscMVRdkQmOMESVnhWAO3OZq2EaCY7jibx7ByicbLngOyEzbCRA4y9FTKRdqv8w/640?wx_fmt=png&from=appmsg&randomid=xbmmt8y1&watermark=1&tp=webp&wxfrom=5&wx_lazy=1)

_图2_

![图片](https://mmbiz.qpic.cn/mmbiz_png/H9YjqVWhgmvT2eGOZSGAnHfSWbscMVRdkrpicxgXVQpQQ8ogJKPEVwo0yeDBIZ8JpiacoexzNLwS8SWG8KicZCcUw/640?wx_fmt=png&from=appmsg&randomid=51bmpeys&watermark=1&tp=webp&wxfrom=5&wx_lazy=1)

conntrack命令支持多种参数，用于执行不同的操作：

- -L, --list：列出连接跟踪表中的所有条目。
    
- -G, --get：获取单个连接跟踪条目的信息。
    
- -D, --delete：从连接跟踪表中删除条目。
    
- -I, --create：创建一个新的连接跟踪条目。
    
- -U, --update：更新已存在的连接跟踪条目。
    
- -E, --event：监听连接跟踪事件。
    
- -d, 目标IP。
    
- -p, 协议。
    
- --dport，目标端口。
    

**连接状态**，描述TCP连接的当前状态：

- `ESTABLISHED`：已建立连接。
    
- `TIME_WAIT`：等待足够的时间以确保远程TCP接收到连接终止请求的确认。
    
- `CLOSE_WAIT`：CLOSE_WAIT状态表示对端（远程主机）已经关闭了连接的一半（发送了一个FIN包），并且本地端（你的计算机）已经接收到这个关闭请求，但是本地应用程序还没有关闭（或者说还没有调用close来关闭连接）。简单来说，CLOSE_WAIT状态意味着TCP连接在等待本地应用程序去关闭连接。
    
- `SYN_SENT`：客户端已发送一个连接请求（SYN包）给服务器，并等待服务器的确认。这表示客户端已经开始了TCP三次握手过程。
    
- `SYN_RECV`：服务器收到客户端的SYN包，并回应一个SYN+ACK包，等待客户端的确认。这个状态表示服务器端已经响应了连接请求，正在进行三次握手的第二步。
    
- `FIN_WAIT_1`：当一方（通常是客户端）决定关闭连接，并发送一个FIN包给对方，等待对方的确认。这标志着连接关闭过程的开始。
    
- `FIN_WAIT_2`：在发送FIN包并收到ACK包后，连接进入FIN_WAIT_2状态。在这个状态下，连接的关闭一方等待对方的FIN包。
    
- `CLOSE_WAIT`：当一方收到另一方的FIN包，即对方请求关闭连接时，它会发送一个ACK包作为回应，并进入CLOSE_WAIT状态。在这个状态下，等待本地应用程序关闭连接。
    
- `CLOSING`：在同时关闭的情况下，当双方几乎同时发送FIN包时，连接会进入CLOSING状态，表示双方都在等待对方的FIN包的确认。
    
- `LAST_ACK`：当处于CLOSE_WAIT状态的一方发送FIN包，并等待对方的最终ACK包时，连接进入LAST_ACK状态。
    
- `TIME_WAIT`：在收到对方的FIN包并发送ACK包后，连接进入TIME_WAIT状态。这个状态持续一段时间（2倍的MSL，最大报文生存时间），以确保对方收到了最终的ACK包。这也允许老的重复数据包在网络中消失。
    
- `CLOSED`：连接完全关闭，两端都释放了连接的资源。
    

## conntrack配合iptables使用

在iptables里，与连接跟踪是使用 `--state`匹配操作，我们能很容易地控制 “谁或什么能发起新的会话”。这样便让 iptables 成为了有状态的防火墙。

**案例1：通过 conntrack中 NEW 状态来限制网络连接访问web服务**

`NEW` 表示一个新的 TCP/UDP 连接的第一个数据包（如 TCP SYN 包），可预防CC攻击。

`iptables -A INPUT -m state --state NEW -s 117.128.29.83 -p tcp --dport 80 -j DROP   `

![图片](https://mmbiz.qpic.cn/mmbiz_png/H9YjqVWhgmvT2eGOZSGAnHfSWbscMVRd7tLVpOW1yTYQyrn6GTRKcB413aXMTO28lV6OtATYEicDlSWuGh8WByQ/640?wx_fmt=png&from=appmsg&randomid=9w01a897&watermark=1&tp=webp&wxfrom=5&wx_lazy=1)

**案例2: 入栈目的为80端口的网络请求，跳过连接跟踪机制。**

`iptables -t raw -A PREROUTING -i eth0 -p tcp --dport 80 --syn -j NOTRACK   `

**案例3: 对连接跟踪事件进行筛选，仅处理带有 `ASSURED` 标记或 `DESTROY` 事件的连接。**

`iptables -I PREROUTING -t raw -j CT --ctevents assured,destroy   `

- `assured`: 稳定连接，通常表示双向流量已确认。
    
- `destroy`: 连接被销毁（如超时、主动关闭）。