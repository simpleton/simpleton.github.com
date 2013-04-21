---
author: sunsj1231
comments: true
date: 2011-03-31 08:01:14
layout: post
slug: '%e8%bd%ac%e8%bd%bdlinux-tcpip-%e5%8d%8f%e8%ae%ae%e6%a0%88%e7%9a%84%e5%85%b3%e9%94%ae%e6%95%b0%e6%8d%ae%e7%bb%93%e6%9e%84socket-buffersk_buff'
title: '[转载]Linux TCP/IP 协议栈的关键数据结构Socket Buffer(sk_buff )'
wordpress_id: 39001
categories:
- coding
---




sk_buff结构可能是linux网络代码中最重要的数据结构，它表示接收或发送数据包的包头信息。它在<include/linux/skbuff.h>中定义，并包含很多成员变量供网络代码中的各子系统使用。




这个结构在linux内核的发展过程中改动过很多次，或者是增加新的选项，或者是重新组织已存在的成员变量以使得成员变量的布局更加清晰。它的成员变量可以大致分为以下几类：






  * Layout 布局


  * General 通用


  * Feature-specific功能相关


  * Management functions管理函数




<!-- more -->




这个结构被不同的网络层（MAC或者其他二层链路协议，三层的IP，四层的TCP或UDP等）使用，并且其中的成员变量在结构从一层向另一层传递时改变。L4向L3传递前会添加一个L4的头部，同样，L3向L2传递前，会添加一个L3的头部。添加头部比在不同层之间拷贝数据的效率更高。由于在缓冲区的头部添加数据意味着要修改指向缓冲区的指针，这是个复杂的操作，所以内核提供了一个函数skb_reserve（在后面的章节中描述）来完成这个功能。协议栈中的每一层在往下一层传递缓冲区前，第一件事就是调用skb_reserve在缓冲区的头部给协议头预留一定的空间。




_skb_reserve_同样被设备驱动使用来对齐接收到包的包头。如果缓冲区向上层协议传递，旧的协议层的头部信息就没什么用了。例如，L2的头部只有在网络驱动处理L2的协议时有用，L3是不会关心它的信息的。但是，内核并没有把L2的头部从缓冲区中删除，而是把有效荷载的指针指向L3的头部，这样做，可以节省CPU时间。




#### 1. 网络参数和内核数据结构




就像你在浏览TCP/IP规范或者配置内核时所看到的一样，网络代码提供了很多有用的功能，但是这些功能并不是必须的， 比如说，防火墙，多播，还有其他一些功能。大部分的功能都需要在内核数据结构中添加自己的成员变量。因此，sk_buff里面包含了很多像#ifdef这样的预编译指令。例如，在sk_buff结构的最后，你可以找到：



    
    <span style="color:#0000ff;">struct</span> sk_buff {<br></br>
                 ... ... ...<br></br>
            <span style="color:#0000ff;">#ifdef</span> CONFIG_NET_SCHED<br></br>
                 _ _u32     tc_index;<br></br>
            <span style="color:#0000ff;">#ifdef</span> CONFIG_NET_CLS_ACT<br></br>
                 _ _u32     tc_verd;<br></br>
                 _ _u32     tc_classid;<br></br>
            <span style="color:#0000ff;">#endif<br></br>
            #endif</span><br></br>
            }




它表明，tc_index只有在编译时定义了CONFIG_NET_SCHED符号才有效。这个符号可以通过选择特定的编译选项来定义（例如："Device Drivers Networking supportNetworking options QoS and/or fair queueing"）。这些编译选项可以由管理员通过make config来选择，或者通过一些自动安装工具来选择。




前面的例子有两个嵌套的选项：CONFIG_NET_CLS_ACT（包分类器）只有在选择支持"QoS and/or fair queueing"时才能生效。




顺便提一下，QoS选项不能被编译成内核模块。原因就是，内核编译之后，由某个选项所控制的数据结构是不能动态变化的。一般来说，如果某个选项会修改内核数据结构（比如说，在sk_buff里面增加一个项tc_index），那么，包含这个选项的组件就不能被编译成内核模块。




你可能经常需要查找是哪个make config编译选项或者变种定义了某个#ifdef标记，以便理解内核中包含的某段代码。在2.6内核中，最快的，查找它们之间关联关系的方法， 就是查找分布在内核源代码树中的kconfig文件中是否定义了相应的符号（每个目录都有一个这样的文件）。在   
2.4内核中，你需要查看_Documentation/Configure.help_文件。




**2. Layout Fields**




有些sk_buff成员变量的作用是方便查找或者是连接数据结构本身。内核可以把sk_buff组织成一个双向链表。当然，这个链表的结构要比常见的双向链表的结构复杂一点。




就像任何一个双向链表一样，sk_buff中有两个指针next和prev，其中，next指向下一个节点，而   
prev指向上一个节点。但是，这个链表还有另一个需求：每个sk_buff结构都必须能够很快找到链表头节点。为了满足这个需求，在第一个节点前面会插入另一个结构sk_buff_head，这是一个辅助节点，它的定义如下：



    
            <div style="margin-top:0;margin-bottom:0;">
            <div style="margin-top:0;margin-bottom:0;">
            <div style="margin-top:0;margin-bottom:0;"><span style="color:#0000ff;">struct </span>sk_buff_head {</div>
            <div style="margin-top:0;margin-bottom:0;">   <span style="color:#8e7cc3;">/* These two members must be first. */</span><span style="color:#cfe2f3;">   </span> </div>
            <div style="margin-top:0;margin-bottom:0;">    <span style="color:#0000ff;">struct</span> sk_buff     * next;    </div>
            <div style="margin-top:0;margin-bottom:0;">    <span style="color:#0000ff;">struct </span>sk_buff     * prev;       </div>
            <div style="margin-top:0;margin-bottom:0;">    _ _u32         qlen;    </div>
            <div style="margin-top:0;margin-bottom:0;">    spinlock_t     lock;    </div>
            <div style="margin-top:0;margin-bottom:0;"> };</div>
            </div>
            </div>
            




qlen代表链表元素的个数。lock用于防止对链表的并发访问。




sk_buff和sk_buff_head的前两个元素是一样的：next和prev指针。这使得它们可以放到同一个链表中，尽管sk_buff_head要比sk_buff小得多。另外，相同的函数可以同样应用于sk_buff和sk_buff_head。




为了使这个数据结构更灵活，每个sk_buff结构都包含一个指向sk_buff_head的指针。这个指针的名字是list。图1会帮助你理解它们之间的关系。




##### Figure 1. List of sk_buff elements




![](http://blogimg.chinaunix.net/blog/upfile/070330110542.jpg)







其他有趣的成员变量如下：




struct sock *sk  
这是一个指向拥有这个sk_buff的sock结构的指针。这个指针在网络包由本机发出或者由本机进程接收时有效，因为插口相关的信息被L4(TCP或 UDP)或者用户空间程序使用。如果sk_buff只在转发中使用(这意味着，源地址和目的地址都不是本机地址)，这个指针是NULL。




unsigned int len  
这是缓冲区中数据部分的长度。它包括主缓冲区中的数据长度(data指针指向它)和分片中的数据长度。它的值在缓冲区从一个层向另一个层传递时改变，因为往上层传递，旧的头部就没有用了，而往下层传递，需要添加本层的头部。len同样包含了协议头的长度。







unsigned int data_len  
和len不同，data_len只计算分片中数据的长度。







unsigned int mac_len  
这是mac头的长度。




atomic_t users  
这是一个引用计数，用于计算有多少实体引用了这个sk_buff缓冲区。它的主要用途是防止释放sk_buff后，还有其他实体引用这个sk_buff。因此，每个引用这个缓冲区的实体都必须在适当的时候增加或减小这个变量。这个计数器只保护sk_buff结构本身，而缓冲区的数据部分由类似的计数器 (dataref)来保护.  
有时可以用atomic_inc和atomic_dec函数来直接增加或减小users，但是，通常还是使用函数skb_get和kfree_skb来操作这个变量。







unsigned int truesize  
这是缓冲区的总长度，包括sk_buff结构和数据部分。如果申请一个len字节的缓冲区，alloc_skb函数会把它初始化成len+sizeof(sk_buff)。



    
    <span style="color:#0000ff;">struct</span> sk_buff *alloc_skb(<span style="color:#0000ff;">unsigned int</span> size,<span style="color:#0000ff;">int </span>gfp_mask)<br></br>
            {<br></br>
                  ... ... ...<br></br>
                  skb->truesize = size + sizeof(<span style="color:#0000ff;">struct </span>sk_buff);<br></br>
                  ... ... ...<br></br>
            }




当skb->len变化时，这个变量也会变化。




unsigned char *head  
unsigned char *end  
unsigned char *data  
unsigned char *tail  
它们表示缓冲区和数据部分的边界。在每一层申请缓冲区时，它会分配比协议头或协议数据大的空间。head和end指向缓冲区的头部和尾部，而data和 tail指向实际数据的头部和尾部，参见图2。每一层会在head和data之间填充协议头，或者在tail和end之间添加新的协议数据。图2中右边数据部分会在尾部包含一个附加的头部。




##### Figure 2. head/end versus data/tail pointers




![](http://blogimg.chinaunix.net/blog/upfile/070330110558.jpg)







void (*destructor)(...)  
这个函数指针可以初始化成一个在缓冲区释放时完成某些动作的函数。如果缓冲区不属于一个socket，这个函数指针通常是不会被赋值的。如果缓冲区属于一个socket， 这个函数指针会被赋值为sock_rfree或sock_wfree(分别由skb_set_owner_r或skb_set_owner_w函数初始化)。这两个sock_xxx函数用于更新socket的队列中的内存容量。




#### 3. General Fields




本节描述sk_buff的主要成员变量，这些成员变量与特定的内核功能无关：




struct timeval stamp  
这个变量只对接收到的包有意义。它代表包接收时的时间戳，或者有时代表包准备发出时的时间戳。它在netif_rx里面由函数net_timestamp设置，而netif_rx是设备驱动收到一个包后调用的函数。




struct net_device *dev  
这 个变量的类型是net_device，net_device它代表一个网络设备。dev的作用与这个包是准备发出的包还是刚接收的包有关。当收到一个包时，设备驱动会把sk_buff的dev指针指向收到这个包的设备的数据结构，就像下面的vortex_rx里的一段代码所做的一样，这个函数属于 3c59x系列以太网卡驱动，用于接收一个帧。(drivers/net/3c59x.c)：



    
    <span style="color:#0000ff;">static int</span> vortex_rx(<span style="color:#0000ff;">struct </span>net_device *dev)<br></br>
            {<br></br>
                        ... ... ...<br></br>
                     skb->dev = dev;<br></br>
                        ... ... ...<br></br>
                     skb->protocol = eth_type_trans(skb, dev);<br></br>
                     netif_rx(skb); /* Pass the packet to the higher layer */<br></br>
                        ... ... ...<br></br>
            }




当一个包被发送时，这个变量代表将要发送这个包的设备。在发送网络包时设置这个值的代码要比接收网络包时设置这个值的代码复杂。有些网络功能可以把多个网络设备组成一个虚拟的网络设备(也就是说，这些设备没有和物理设备直接关联)，并由一个虚拟网络设备驱动管理。当虚拟设备被使用时，dev指针指向虚拟设备的net_device结构。而虚拟设备驱动会在一组设备中选择一个设备并把dev指针修改为这个设备的net_device结构。因此，在某些情况下， 指向传输设备的指针会在包处理过程中被改变。







struct net_device *input_dev  
这是收到包的网络设备的指针。如果包是本地生成的，这个值为NULL。对以太网设备来说，这个值由eth_type_trans初始化,它主要被流量控制代码使用。







struct net_device *real_dev  
这个变量只对虚拟设备有意义，它代表与虚拟设备关联的真实设备。例如，Bonding和VLAN设备都使用它来指向收到包的真实设备。




union {...} h  
union {...} nh  
union {...} mac




这些是指向TCP/IP各层协议头的指针：h指向L4，nh指向L3，mac指向L2。每个指针的类型都是一个联合，包含多个数据结构，每一个数据结构都表示内核在这一层可以解析的协议。例如，h是一个包含内核所能解析的L4协议的数据结构的联合。每一个联合都有一个raw变量用于初始化，后续的访问都是通过协议相关的变量进行的。




当接收一个包时，处理n层协议头的函数从n-1层收到一个缓冲区，它的skb->data指向层n协议的头。处理 n层协议的函数把本层的指针(例如，L3对应的是skb->nh指针)初始化为skb->data，因为这个指针的值会在处理下一层协议时改 变(skb->data将被初始化成缓冲区里的其他地址)。在处理n层协议的函数结束时，在把包传递给n+1层的处理函数前，它会把skb- >data指针指向n层协议头的末尾，这正好是n+1层协议的协议头(参见图3)。




发送包的过程与此相反，但是由于要为每一层添加新的协议头，这个过程要比接收包的过程复杂。




##### Figure 3. Header's pointer initializations while moving from layer two to layer three




![](http://blogimg.chinaunix.net/blog/upfile/070330110612.jpg)







struct dst_entry dst  
这个变量在路由子系统中使用。




char cb[40]  
这 是一个"control buffer"，或者说是一个私有信息的存储空间，由每一层自己维护并使用。它在分配sk_buff结构时分配(它目前的大小是40字节，已经足够为每一层存储必要的私有信息了)。在每一层中，访问这个变量的代码通常用宏实现以增强代码的可读性。例如，TCP用这个变量存储tcp_skb_cb结构，这个结构在include/net/tcp.h中定义：



    
    <span style="color:#0000ff;">struct </span>tcp_skb_cb {<br></br>
                 ... ... ...<br></br>
                 _ _u32         seq;         <span style="color:#674ea7;">/* Starting sequence number */</span><br></br>
                 _ _u32         end_seq;    <span style="color:#674ea7;"> /* SEQ + FIN + SYN + datalen*/</span><br></br>
                 _ _u32         when;       <span style="color:#674ea7;"> /* used to compute rtt's     */</span><br></br>
                 _ _u8          flags;       <span style="color:#674ea7;">/* TCP header flags.         */</span><br></br>
                 ... ... ...<br></br>
            };




下面这个宏被TCP代码用来访问cb变量。在这个宏里面，有一个简单的类型转换：



    
    <span style="color:#0000ff;">#define</span> TCP_SKB_CB(_ _skb)     ((<span style="color:#0000ff;">struct </span>tcp_skb_cb *)&((_ _skb)->cb[0]))




下面的例子是TCP子系统在收到一个分段时填充相关数据结构的代码：



    
    <span style="color:#0000ff;">int </span>tcp_v4_rcv(<span style="color:#0000ff;">struct </span>sk_buff *skb)<br></br>
            {<br></br>
                     ... ... ...<br></br>
                     th = skb->h.th;<br></br>
                     TCP_SKB_CB(skb)->seq = ntohl(th->seq);<br></br>
                     TCP_SKB_CB(skb)->end_seq = (TCP_SKB_CB(skb)->seq + th->syn + th->fin +<br></br>
                                                 skb->len - th->doff * 4);<br></br>
                     TCP_SKB_CB(skb)->ack_seq = ntohl(th->ack_seq);<br></br>
                     TCP_SKB_CB(skb)->when = 0;<br></br>
                     TCP_SKB_CB(skb)->flags = skb->nh.iph->tos;<br></br>
                     TCP_SKB_CB(skb)->sacked = 0;<br></br>
                     ... ... ...<br></br>
            }




如果想要了解cb中的参数是如何被取出的，可以查看net/ipv4/tcp_output.c中的tcp_transmit_skb函数。这个函数被TCP用于向IP层发送一个分段。




unsigned int csum  
unsigned char ip_summed  
表示校验和以及相关状态标记。




unsigned char cloned  
一个布尔标记，当被设置时，表示这个结构是另一个sk_buff的克隆。在"克隆和拷贝缓冲区"一节中有描述。




unsigned char pkt_type  
这个变量表示帧的类型，分类是由L2的目的地址来决定的。可能的取值都在include/linux/if_packet.h中定义。对以太网设备来说，这个变量由eth_type_trans函数初始化。   
类型的可能取值如下：




PACKET_HOST  
包的目的地址与收到它的网络设备的L2地址相等。换句话说，这个包是发给本机的。




.PACKET_MULTICAST  
包的目的地址是一个多播地址，而这个多播地址是收到这个包的网络设备所注册的多播地址。




PACKET_BROADCAST  
包的目的地址是一个广播地址，而这个广播地址也是收到这个包的网络设备的广播地址。




PACKET_OTHERHOST  
包的目的地址与收到它的网络设备的地址完全不同(不管是单播，多播还是广播)，因此，如果本机的转发功能没有启用，这个包会被丢弃。




PACKET_OUTGOING  
这个包将被发出。用到这个标记的功能包括Decnet协议，或者是为每个网络tap都复制一份发出包的函数。




PACKET_LOOPBACK  
这个包发向loopback设备。由于有这个标记，在处理loopback设备时，内核可以跳过一些真实设备才需要的操作。




PACKET_FASTROUTE  
这个包由快速路由代码查找路由。快速路由功能在2.6内核中已经去掉了。







_ _u32 priority  
这个变量描述发送或转发包的QoS类别。如果包是本地生成的，socket层会设置priority变量。如果包是将要被转发的， rt_tos2priority函数会根据ip头中的Tos域来计算赋给这个变量的值。这个变量的值与DSCP(DiffServ CodePoint)没有任何关系。




unsigned short protocol  
这个变量是高层协议从二层设备的角度所看到的协议。典型的协议包括IP，IPV6和ARP。完整的列表在include/linux/if_ether.h 中。由于每个协议都有自己的协议处理函数来处理接收到的包，因此，这个域被设备驱动用于通知上层调用哪个协议处理函数。每个网络驱动都调用 netif_rx来通知上层网络协议的协议处理函数，因此protocol变量必须在这些协议处理函数调用之前初始化。




unsigned short security  
这是包的安全级别。这个变量最初由IPSec子系统使用，但现在已经作废了。




#### 4. Feature-Specific Fields




linux内核是模块化的，你可以选择包含或者删除某些功能。因此，sk_buff结构里面的一些成员变量只有在内核选择支持某些功能时才有效，比如防火墙(netfilter)或者qos：




unsigned long nfmark  
_ _u32 nfcache  
_ _u32 nfctinfo  
struct nf_conntrack *nfct  
unsigned int nfdebug  
struct nf_bridge_info *nf_bridge  
这些变量被netfilter使用(防火墙代码)，内核编译选项是"Device Drivers->Networking support-> Networking options-> Network packet filtering"和两个子选项"Network packet filtering debugging"和"Bridged IP/ARP packets filtering"




union {...} private  
这个联合结构被高性能并行接口(HIPPI)使用。相应的内核编译选项是"Device->Drivers ->Networking support ->Network device support ->HIPPI driver support"




_ _u32 tc_index  
_ _u32 tc_verd  
_ _u32 tc_classid  
这些变量被流量控制代码使用，内核编译选项是"Device Drivers ->Networking->support ->Networking options ->QoS and/or fair queueing"和它的子选项"Packetclassifier API"




struct sec_path *sp  
这个变量被IPSec协议用于跟踪传输的信息。




#### 5. Management Functions




有很多函数，通常都比较短小而且简单，内核用这些函数操作sk_buff的成员变量或者sk_buff  
链表。图4会帮助我们理解其中几个重要的函数。我们首先来看分配和释放缓冲区的函数，然后是一些通过移动指针在缓冲区的头部或尾部预留空间的函数。




如果你看过include/linux/skbuff.h和net/core/skbuff.c中的函数，你会发现，基本上每个函数都有两个版本，名字分别是do_something和__do_something。通常第一种函数是一个包装函数，它会在第二种函数的基础 上增加合法性检查或者锁。一般来说，类似__do_something的函数不能被直接调用(除非满足特定的条件，比如说锁)。那些违反这条规则而直接引 用这些函数的不良代码会最终被更正。




##### Figure 4. Before and after: (a)skb_put, (b)skb_push, (c)skb_pull, and (d)skb_reserve




![](http://blogimg.chinaunix.net/blog/upfile/070330110625.jpg)







**5.1. Allocating memory: alloc_skb and dev_alloc_skb**




alloc_skb是net/core/skbuff.c里面定义的，用于分配缓冲区的函数。我们已经知道，数据缓冲区和缓冲区的描述结构(sk_buff结构)是两种不同的实体，这就意味着，在分配一个缓冲区时，需要分配两块内存(一个是缓冲区，一个是缓冲区的描述结构 sk_buff)。




alloc_skb调用函数kmem_cache_alloc从缓存中获取一个sk_buff结构，并调用kmalloc分配缓冲区(如果有缓存的话，它同样从缓存中获取内存)。简化后的代码如下：



    
         skb = kmem_cache_alloc(skbuff_head_cache, gfp_mask & ~_ _GFP_DMA);<br></br>
                 ... ... ...<br></br>
                 size = SKB_DATA_ALIGN(size);<br></br>
                 data = kmalloc(size + sizeof(struct skb_shared_info), gfp_mask);







在调用kmalloc前，size参数通过SKB_DATA_ALIGN宏强制对齐。在函数返回前，它会初始化结构中的一些变量，最后的结构如图5所示。在图5右边所示的内存块的底部，你能看到对齐操作所带来的填充区域。




##### Figure 5. alloc_skb function




![](http://blogimg.chinaunix.net/blog/upfile/070330110636.jpg)







dev_alloc_skb也是一个缓冲区分配函数，它主要被设备驱动使用，通常用在中断上下文中。这是一个 alloc_skb函数的包装函数，它会在请求分配的大小上增加16字节的空间以优化缓冲区的读写效率，它的分配要求使用原子操作 (GFP_ATOMIC)，这是因为它是在中断处理函数中被调用的。



    
    <span style="color:#0000ff;">static inline struc</span><span style="color:#0000ff;">t</span> sk_buff *dev_alloc_skb(<span style="color:#0000ff;">unsigned int</span> length)<br></br>
            {<br></br>
                 <span style="color:#0000ff;">return</span> _ _dev_alloc_skb(length, GFP_ATOMIC);<br></br>
            }<br></br>
            <br></br>
            <span style="color:#0000ff;">static inline struct</span> sk_buff *_ _dev_alloc_skb(<span style="color:#0000ff;">unsigned int</span> length, <span style="color:#0000ff;">int</span> gfp_mask)<br></br>
            {<br></br>
                 <span style="color:#0000ff;">struct </span>sk_buff *skb = alloc_skb(length + 16, gfp_mask);<br></br>
                 <span style="color:#0000ff;">if </span>(likely(skb))<br></br>
                         skb_reserve(skb, 16);<br></br>
                 <span style="color:#0000ff;">return </span>skb;<br></br>
            }




如果没有体系架构相关的实现，缺省使用__dev_alloc_skb的实现。




**5.2. Freeing memory: kfree_skb and dev_kfree_skb**




这两个函数释放缓冲区，并把它返回给缓冲池(缓存)。kfree_skb可以直接调用，也可以通过包装函数 dev_kfree_skb调用。后面这个函数一般被设备驱动使用，与之功能相反的函数是dev_alloc_skb。dev_kfree_skb仅是一个简单的宏，它什么都不做，只简单地调用kfree_skb。这些函数只有在skb->users为1地情况下才释放内存(没有人引用这个结构)。 否则，它只是简单地减小   
skb->users。如果缓冲区有三个引用者，那么只有第三次调用dev_kfree_skb或kfree_skb时才释放内存。




图6中的流程图显示了释放一个缓冲区所需要的步骤。当sk_buff释放后，dst_release同样会被调用以减小相关dst_entry数据结构的引用计数。




如果destructor被初始化过，相应的函数会在此时被调用.




在图5中，我们看到，一个简单的场景是：一个sk_buff结构与另一个内存块相关，这个内存块里存储的是真正的数据。 当然，内存块底部的skb_shared_info数据结构可以包含指向其他分片的指针(参见图5)。如果存在分片，kfree_skb同样会释放这些分片所占用的内存。最后，kfree_skb 把sk_buff结构返回给skbuff_head_cache缓存。




**5.3. Data reservation and alignment: skb_reserve, skb_put, skb_push, and skb_pull**




skb_reserve可以在缓冲区的头部预留一定的空间，它通常被用来在缓冲区中插入协议头或者在某个边界上对齐。这 个函数改变data和tail指针，而data和tail指针分别指向负载的开头和结尾，图4(d)展示了调用skb_reserve(skb,n)的结果。这个函数通常在分配缓冲区之后就调用，此时的   
data和tail指针还是指向同一个地方。




如果你查看某个以太网设备驱动的收包函数(例如，_drivers/net/3c59x.c_中的vortex_rx)， 你就会发现它在分配缓冲区之后，在向缓冲区中填充数据之前，会调用下面的函数：



    
    skb_reserve(skb, 2);    <span style="color:#674ea7;"> /* Align IP on 16 byte boundaries */</span>




##### Figure 6. kfree_skb function




![](http://blogimg.chinaunix.net/blog/upfile/070330110647.jpg)







由于**以太网帧的头部长度是14个八位组**，这个函数把缓冲区的头部指针向后移动了2个字节。这样，紧跟在以太网头部之后的IP头部在缓冲区中存储时就可以在16字节的边界上对齐。如图7所示。




##### Figure 7. (a) before skb_reserve, (b) after skb_reserve, and (c) after copying the frame on the buffer




![](http://blogimg.chinaunix.net/blog/upfile/070330110659.jpg)







图8展示了一个在发送过程中使用skb_reserve的例子。




##### Figure 8. Buffer that is filled in while traversing the stack from the TCP layer down to the link layer




![](http://blogimg.chinaunix.net/blog/upfile/070330110745.jpg)









  1. 





当TCP发送数据时，它根据一些条件分配一个缓冲区(比如，TCP的最大分段长度(mss)，是否支持散读散写I/O等








  2. 





TCP在缓冲区的头部预留足够的空间(用skb_reserve)用于填充各层的头部(如TCP，IP，链路层等)。MAX_TCP_HEADER参数是各层头部长度的总和，它考虑了最坏的情况：由于tcp层不知道将要用哪个接口发送包，它为每一层预留了最大的头部长度。它甚至考虑了出现多个IP头的可能性(如果内核编译支持IP over IP， 我们就会遇到多个IP头的情况)。








  3. 





把TCP的负载拷贝到缓冲区。需要注意的是：图8只是一个例子。TCP的负载可能会被组织成其他形式。例如它可以存储到分片中。








  4. 





TCP层添加自己的头部。








  5. 





TCP层把缓冲区传递给IP层，IP层同样添加自己的头部。








  6. 





IP层把缓冲区传递给邻居层，邻居层添加链路层头部。












当缓冲区在协议栈中向下层传递时，每一层都把skb->data指针向下移动，然后拷贝自己的头部，同时更新skb->len。这些操作都使用图4中所展示的函数完成。




skb_reserve函数并没有把数据移出或移入缓冲区，它只是简单地更新了缓冲区的两个指针，这两个指针在图4(d)中有描述。



    
    <span style="color:#0000ff;">static inline void </span>skb_reserve(<span style="color:#0000ff;">struct </span>sk_buff *skb, <span style="color:#0000ff;">unsigned int</span> len)<br></br>
            {<br></br>
                 skb->data+=len;<br></br>
                 skb->tail+=len;<br></br>
            }




skb_push在缓冲区的开头加入一块数据，而skb_put在缓冲区的末尾加入一块数据。与skb_reserve  
类似，这些函数都不会真的往缓冲区中添加数据，相反，它们只是移动缓冲区的头指针和尾指针。数据由其他函数拷贝到缓冲区中。skb_pull通过把head指针往前移来在缓冲区的头部删除一块数据。图4展示了这些函数是如何工作的。




**2.1.5.4. The skb_shared_info structure and the skb_shinfo function**




如图5所示，在缓冲区数据的末尾，有一个数据结构skb_shared_info，它保存了数据块的附加信息。这个数据结构紧跟在end指针所指的地址之后(end指针指示数据的末尾)。下面是这个结构的定义：



    
    <span style="color:#0000ff;">struct </span>skb_shared_info {<br></br>
                 atomic_t         dataref;<br></br>
                 <span style="color:#0000ff;">unsigned int</span>     nr_frags;<br></br>
                 <span style="color:#0000ff;">unsigned short</span>   tso_size;<br></br>
                 <span style="color:#0000ff;">unsigned short</span>   tso_seqs;<br></br>
                 <span style="color:#0000ff;">struct sk_buff</span>   *frag_list;<br></br>
                 skb_frag_t       frags[MAX_SKB_FRAGS];<br></br>
            };




dataref表示数据块的"用户"数，这个值在下一节(克隆和拷贝缓冲区)中有描述。nf_frags， frag_list和frags用于存储IP分片。skb_is_nonlinear函数用于测试一个缓冲区是否是分片的，而skb_linearize可以把分片组合成一个单一的缓冲区。组合分片涉及到数据拷贝，它将严重影响系统性能。




有些网卡硬件可以完成一些传统上由CPU完成的任务。最常见的例子就是计算L3和L4校验和。有些网卡甚至可以维护L4 协议的状态机。在下面的例子中，我们主要讨论TCP段卸载TCP segmentation offload，这些网卡实现了TCP层的一部分功能。tso_size和tso_seqs就在这种情况下使用。




需要注意的是：sk_buff中没有指向skb_shared_info结构的指针。如果要访问这个结构， 就需要使用skb_info宏，这个宏简单地返回end指针：



    
    #define skb_shinfo(SKB)     ((struct skb_shared_info *)((SKB)->end))




下面的语句展示了如何使用这个宏来增加结构中的某个成员变量的值：



    
    skb_shinfo(skb)->dataref++;




**2.1.5.5. Cloning and copying buffers**




如果一个缓冲区需要被不同的用户独立地操作，而这些用户可能会修改sk_buff中某些变量的值(比如h和nh值)，内核没有必要为每个用户复制一份完整的sk_buff以及相应的缓冲区。相反，为提高性能，内核克隆一个缓冲区。克隆过程只复制sk_buff结构， 同时修改缓冲区的引用计数以避免共享的数据被提前释放。克隆缓冲区使用skb_clone函数。




一个使用包克隆的场景是：一个接收包的过程需要把这个包传递给多个接收者，例如包处理函数或者一个或多个网络模块。




被克隆的sk_buff不会放在任何链表中，同时也不会有到socket的引用。原始的和克隆的sk_buff中的 skb->cloned值都被置为1。克隆包的skb->users值被置为1，这样，在释放时，可以先释放sk_buff结构。同时，缓冲 区的引用计数(dataref)增加1(因为有多个sk_buff结构指向它)。图9展示了克隆缓冲区的例子。




##### Figure 9. skb_clone function




![](http://blogimg.chinaunix.net/blog/upfile/070330110728.jpg)







skb_cloned函数可以用来测试skb的克隆状态。




图9展示了一个分片缓冲区的例子，这个缓冲区的一些数据保存在分片结构数组frags中。




skb_share_check用于检查引用计数skb->users，如果users变量表明skb是被共享的， 则克隆一个新的sk_buff。




如果一个缓冲区被克隆了，这个缓冲区的内容就不能被修改。这就意味着，访问数据的函数没有必要加锁。因此，当一个函数不仅要修改sk_buff，而且要修改缓冲区内容时， 就需要同时复制缓冲区。在这种情况下，程序员有两个选择。如果他知道所修改的数据在skb->start和skb->end  
之间，他可以使用pskb_copy来复制这部分数据。如果他同时需要修改分片中的数据，他就必须使用skb_copy。图10展示了pskb_copy和skb_copy的结果。skb_shared_info结构也可以包含一个   
sk_buff的链表(链表名称是frag_list)。这个链表在pskb_copy和skb_copy中的处理方式与frags数组的处理方式相同(图10忽略了frag_list)。




##### Figure 10. (a) pskb_copy function and (b) skb_copy function







![](http://blogimg.chinaunix.net/blog/upfile/070330110824.jpg)







在决定克隆或复制一个缓冲区时，子系统的程序员不能预测其他内核组件(其他子系统)是否需要使用缓冲区里的原始数据。内核是模块化的，它的状态变化是不可预测的，因此，每个子系统都不知道其他子系统是如何操作缓冲区的。因此，内核程序员需要记录它们对缓冲区的修改，并且在修改缓冲区前，复制一个新的缓冲区，以避免其他子系统在使用缓冲区的原始数据时出现错误。




**2.1.5.6. List management functions**




这些函数管理sk_buff的链表(也被称作队列)。在<include/linux/skbuff.h>和<net/core/skbuff.c>中有函数完整列表。以下是一些经常使用的函数：




skb_queue_head_init  
初始化sk_buff_head结构，创建一个空队列。




skb_queue_head, skb_queue_tail  
把一个缓冲区加入队列的头部或尾部。




skb_dequeue, skb_dequeue_tail  
从队列的头部或尾部取下一个缓冲区。第一个函数的名字应该是skb_dequeue_head，以保持和其他函数的名字风格一致。




skb_queue_purge  
清空一个队列。




skb_queue_walk  
Runs a loop on each element of a queue in turn.




这些函数都是原子操作，它们必须先获取sk_buff_head中的自旋锁，然后才能访问队列中的元素。否则，它们有可能被其他异步的添加或删除操作打断，比如在定时器中调用的函数，这将导致链表出现错误而使得系统崩溃。




因此，每个函数的实现都采用下面这种形式：



    
    <span style="color:#0000ff;">static inline</span> <em>function_name</em> ( <em>parameter_list</em> )<br></br>
            {<br></br>
                     <span style="color:#0000ff;">unsigned long </span>flags;<br></br>
            <br></br>
                     spin_lock_irqsave(...);<br></br>
                     _ _ _<em>function_name</em> ( <em>parameter_list</em> )<br></br>
                     spin_unlock_irqrestore(...);<br></br>
            }




这些函数先获取锁，然后调用一个以两个下划线开头的同名函数(这个函数做具体的操作，而且不需要锁)，然后释放锁。



