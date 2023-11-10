---
layout: post
title: "使用scapy和tcpdump检验标准网络栈行为"
categories:
  - 计算机网络
toc: true
toc_sticky: true
toc_label: 目录
---

## cs144-lab4的奇怪用例

cs144的lab4是要实现TCPConnection,实际上到了这一步的TCP已经很接近现实世界了，test中有一个就是使用自己写的TCPConnection搭载HTTP GET报文向真实网页发送请求（尽管除了Connection以外的部分都是教授帮忙写的），说实话lab4的难度相当大，个人认为不亚于6.824的raft和kvraft。

由于lab难度较大，本人实力不济，需要相当程度的“面向测试用例编程”，但是其中一个测试用例让我感到非常困惑，代码如下：

```c++
       // test #1: START -> SYN_SENT -> ACK (ignored) -> SYN -> SYN_RECV
        {
            TCPTestHarness test_1(cfg);

            // tell the FSM to connect, make sure we get a SYN
            test_1.execute(Connect{});
            test_1.execute(Tick(1));
            TCPSegment seg1 = test_1.expect_seg(ExpectOneSegment{}.with_syn(true).with_ack(false),
                                                "test 1 failed: could not parse SYN segment or invalid flags");

            test_1.execute(ExpectState{State::SYN_SENT});

            // send ACK only (no SYN yet)
            test_1.send_ack(WrappingInt32{0}, seg1.header().seqno + 1);
            test_1.execute(Tick(1));
            test_1.execute(ExpectState{State::SYN_SENT});

            // now send SYN
            const WrappingInt32 isn(rd());
            test_1.send_syn(isn);
            test_1.execute(Tick(1));
            test_1.execute(ExpectState{State::SYN_RCVD});

            test_1.execute(ExpectOneSegment{}.with_ack(true).with_syn(false).with_ackno(isn + 1));

            test_1.execute(ExpectBytesInFlight{1UL});
        }

        // test #2: START -> SYN_SENT -> SYN -> ACK -> ESTABLISHED
        {
            TCPTestHarness test_2(cfg);

            test_2.execute(Connect{});
            test_2.execute(Tick(1));

            TCPSegment seg = test_2.expect_seg(ExpectOneSegment{}.with_syn(true).with_ack(false),
                                               "test 2 failed: could not parse SYN segment or invalid flags");
            auto &seg_hdr = seg.header();

            test_2.execute(ExpectState{State::SYN_SENT});

            // send SYN (no ACK yet)
            const WrappingInt32 isn(rd());
            test_2.send_syn(isn);
            test_2.execute(Tick(1));

            test_2.expect_seg(ExpectOneSegment{}.with_syn(false).with_ack(true).with_ackno(isn + 1),
                              "test 2 failed: bad ACK for SYN");

            test_2.execute(ExpectState{State::SYN_RCVD});

            // now send ACK
            test_2.send_ack(isn + 1, seg_hdr.seqno + 1);
            test_2.execute(Tick(1));
            test_2.execute(ExpectNoSegment{}, "test 2 failed: got spurious ACK after ACKing SYN");
            test_2.execute(ExpectState{State::ESTABLISHED});
        }
```

测试用例中，描述了TCP端点发送了syn包之后，分别接收拆开的ack和syn，以及拆开的syn和ack的行为，按照测试用例的期望，发生的事情分别如下。

1、（端点）发送syn-接收ack-忽略ack-接收syn-发送ack-进入SYN_RECV状态

2、（端点）发送syn-接收syn-发送ack-接收ack-什么也不发送-进入ESTABLISHED状态

实际上，这两种情况都是违反RFC标准的，就我的理解而言，三次握手的实现中，应当做的是保证出现syn-syn/ack-ack的标准序列，而不是以任何方式尝试去替代这样的步骤。一方面，违反规范看上去很怪异且可能会带来风险（虽然具体我也没想到有什么风险），另一方面，强制保持这样的序列我认为并不是一件难事。具体来说，就是在SYN_SENT状态下，做出以下反应，以保证出现“标准序列”。

1）丢弃一切syn/ack和syn以外的包

2）对于syn/ack响应ack

3）对于syn响应syn/ack

## 使用scapy和tcpdump检验标准网络栈

但分析不管用，我决定自己去看一看真实的网络协议栈是怎么实现的，用到的工具是python的网络库scapy和tcpdump，以及telnet。

### 过程

#### scapy脚本

```python
from scapy.all import *
import time


# 定义一个函数来处理捕获的包
def handle_packet(packet):
    # 检查是否为 TCP 包，并且目的端口为 9999
    if TCP in packet and packet[TCP].dport == 9999:
        # 获取源 IP 和源端口
        src_ip = packet[IP].src
        src_port = packet[TCP].sport

        # 提取收到的SYN包的SEQ号
        seq_num = packet[TCP].seq

        # 创建并发送一个独立的 TCP SYN 包
        syn_packet = IP(dst=src_ip) / TCP(sport=9999, dport=src_port, flags="S")

        # 稍微延迟，然后发送一个带有正确ACK号的 TCP ACK 包
        ack_packet = IP(dst=src_ip) / TCP(
            sport=9999, dport=src_port, flags="A", ack=seq_num + 1
        )
        send(ack_packet)
        send(syn_packet)

        exit()


# 开始嗅探，过滤 TCP 和端口 9999 的包，不存储数据包
sniff(iface="eth0", filter="tcp and port 9999", prn=handle_packet, store=False)

```

scapy脚本用于嗅探端口，并且伪造tcp包。

拜托我最爱的gpt-4为我写了一个脚本，不得不说自从gpt相当程度地为我抹除了python的学习成本之后，它在我心目中的地位一下子就上升了，即插即用程度堪比shell的同时拥有极其强大的功能，写一些测试之类的确实好用。

这个脚本嗅探并过滤特定包，然后用handle函数进行处理，该函数中可以根据你的需要造包发包，把send那边改一下就可以了，很方便。

**坑点**：iface表示过滤的网络接口，如果你是localhost内发记得改成lo。

#### 防火墙设置

由于网络栈是一个操作系统级别的东西，而你在发送包的时候，希望造特定的包而不受到系统的影响，此时如果在目标端口上开启socket，那很难深入到其中去干涉它的行为；而如果你不开启socket，操作系统会检测到你发送过来的包，并且回一个rst包，这样就干扰了我们“控制tcp端点（现在可以认为是服务端）”的意图。

解决这个问题的一个办法是：不监听目标端点，只使用scapy脚本嗅探并且伪造包，并且使用防火墙将目标端口的所有rst包拦截，配置防火墙的一个流程可以是：

```shell
sudo nft add table inet filter
sudo nft add chain inet filter output { type filter hook output priority 0 \; }
sudo nft add rule inet filter output 'tcp sport 9999 tcp flags & (rst) == rst drop'
```

然后你可以通过以下命令查看防火墙规则

```shell
 sudo nft list ruleset
```

#### 具体流程

现在可以具体的造包和抓包流程了：启动scapy脚本，使用tcpdump命令抓包

```shell
tcpdump -i any port 9999
```

然后使用正规实现的网络栈向目标端口发送请求，观察tcpdump的结果即可。

**坑点**：关于这里，我一开始使用的是wsl向localhost发送消息，但发现scapy伪造的数据包无论如何都会被客户端丢弃，无奈只能使用windows向wsl发送。



### 结果

#### 正常发送syn-ack

在windows中使用telnet，wsl中使用scapy进行模拟发包（syn-ack），得到的结果

```
15:28:50.050422 eth0  In  IP DESKTOP-R370IH5.mshome.net.65228 > 192.168.86.144.9999: Flags [S], seq 3538371178, win 64240, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
15:28:50.199218 eth0  Out IP 192.168.86.144.9999 > DESKTOP-R370IH5.mshome.net.65228: Flags [S.], seq 225818447, ack 3538371179, win 65483, options [mss 65495,sackOK,nop,wscale 7,eol], length 0
15:28:50.199693 eth0  In  IP DESKTOP-R370IH5.mshome.net.65228 > 192.168.86.144.9999: Flags [.], ack 1, win 1026, length 0
```

可以看到正常完成了三次握手，telnet成功进入会话界面。

#### 分开发送syn和ack

模拟发包为syn，并且之后发送ack，得到的结果

```
15:35:05.617165 eth0  In  IP DESKTOP-R370IH5.mshome.net.65368 > 192.168.86.144.9999: Flags [S], seq 3995262156, win 64240, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
15:35:05.759385 eth0  Out IP 192.168.86.144.9999 > DESKTOP-R370IH5.mshome.net.65368: Flags [S], seq 0, win 8192, length 0
15:35:05.759822 eth0  In  IP DESKTOP-R370IH5.mshome.net.65368 > 192.168.86.144.9999: Flags [S.], seq 3995262156, ack 1, win 64240, options [mss 1460], length 0
15:35:06.339766 eth0  Out IP 192.168.86.144.9999 > DESKTOP-R370IH5.mshome.net.65368: Flags [.], ack 1, win 8192, length 0

```

可以看到，telnet接收到异常的syn之后，马上发送了syn-ack，并且维持了之前syn中的seq（也就是isn），这验证了我的说法，实际上也进入了telnet会话。

#### 只发送syn

```
15:42:27.336114 eth0  In  IP DESKTOP-R370IH5.mshome.net.49672 > 192.168.86.144.9999: Flags [S], seq 225320074, win 64240, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
15:42:27.479388 eth0  Out IP 192.168.86.144.9999 > DESKTOP-R370IH5.mshome.net.49672: Flags [S], seq 0, win 8192, length 0
15:42:27.479803 eth0  In  IP DESKTOP-R370IH5.mshome.net.49672 > 192.168.86.144.9999: Flags [S.], seq 225320074, ack 1, win 64240, options [mss 1460], length 0
15:42:28.345491 eth0  In  IP DESKTOP-R370IH5.mshome.net.49672 > 192.168.86.144.9999: Flags [S.], seq 225320074, ack 1, win 64240, options [mss 1460], length 0
```

与上面的结果基本相同，但是由于没有ack，telnet不断重发syn-ack。

#### 先发送ack，再发送syn

```
16:01:37.214990 eth0  In  IP DESKTOP-R370IH5.mshome.net.49935 > 192.168.86.144.9999: Flags [S], seq 3760926205, win 64240, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
16:01:37.349327 eth0  Out IP 192.168.86.144.9999 > DESKTOP-R370IH5.mshome.net.49935: Flags [.], ack 3760926206, win 8192, length 0
16:01:37.440540 eth0  Out IP 192.168.86.144.9999 > DESKTOP-R370IH5.mshome.net.49935: Flags [S], seq 0, win 8192, length 0
16:01:37.441001 eth0  In  IP DESKTOP-R370IH5.mshome.net.49935 > 192.168.86.144.9999: Flags [S.], seq 3760926205, ack 1, win 64240, options [mss 1460], length 0
16:01:38.218455 eth0  In  IP DESKTOP-R370IH5.mshome.net.49935 > 192.168.86.144.9999: Flags [S.], seq 3760926205, ack 1, win 64240, options [mss 1460], length 0
```

和上面本质上也一样，第一个ack被丢弃了，然后收到syn之后，不断重发syn-ack。

## 总结

我认为cs144-lab4中的这个测试用例确实是不合理的，也和windows网络栈真实的实现是相悖的。