## 网络安全第五章实验报告 
### 实验目的
* 掌握网络扫描之端口状态探测的基本原理
### 实验环境
* python + scapy 2.4.3
* nmap
### 网络拓扑结构 
![](img/网络拓扑.png) 

身份 | 虚拟机名称 |  网卡 | IP  
-|-|-|-
网关 | debian-getaway | intnet1 |172.16.111.1|
攻击者主机 | kali-attacker | intnet1 |172.16.111.118|
目标靶机 | victim-kali-1 | intnet1 |172.16.111.108 |
### 实验要求 
##### 完成以下扫描技术的编程实现
* [ √ ] TCP connect scan / TCP stealth scan
* [ √ ] TCP Xmas scan / TCP fin scan / TCP null scan
* [ √ ] UDP scan
* 上述每种扫描技术的实现测试均需要测试端口状态为：开放、关闭和过滤状态时的程序执行结果
* 提供每一次扫描测试的抓包结果并分析与课本中的扫描方法原理是否相符？如果不同，试分析原因
* 在实验报告中详细说明实验网络环境拓扑、被测试 IP 的端口状态是如何模拟的
* （可选）复刻nmap的上述扫描技术实现的命令行参数开关 
### 实验过程
* 端口状态模拟 
```
1. 关闭状态
sudo ufw disable
systemctl stop apache2
systemctl stop dnsmasq 关闭端口
2. 开放状态
systemctl start apache2 开启服务开放TCP端口
systemctl start dnsmasq 开启服务开放UDP端口
3. 被过滤状态
sudo ufw enable && sudo ufw deny 80/tcp
sudo ufw enable && sudo ufw deny 53/udp
```
* TCP connect scan
```
#攻击者主机编写TCPconnect.py
from scapy.all import *
def tcpconnect(dst_ip,dst_port,timeout=10):
    pkts=sr1(IP(dst=dst_ip)/TCP(dport=dst_port,flags="S"),timeout=timeout)
    if (pkts is None):
        print("FILTER")
    elif(pkts.haslayer(TCP)):
        if(pkts[1].flags=='AS'):
            print("OPEN")
        elif(pkts[1].flags=='AR'):
                print("CLOSE")
tcpconnect('172.16.111.108',80)
``` 
  * Closed 
    靶机检测自身端口状态
    ![](img/靶机检测自身端口状态.png) 
    攻击者主机运行TCP connect scan python脚本 
    ![](img/tcpc_closed.png) 
    靶机Wireshark抓包
    ![](img/1.png) 
    攻击者主机nmap复刻 
    ![](img/n_tcpc_closed.png)
    分析：TCP三次握手机制，攻击者主机向靶机发送连接请求后，靶机相应端口处于关闭状态，靶机将会向攻击者返回[RST,ACK]包，抓包结果与预期结果一致。 
  * Open 
    攻击者主机运行TCP connect scan python脚本 
    ![](img/tcpc_opend.png) 
    靶机Wireshark抓包 
    ![](img/2.png) 
    攻击者主机nmap复刻 
    ![](img/n_tcpc_open.png) 
    分析：TCP三次握手机制，攻击者主机向靶机发送连接请求后，收到靶机返回[SYN/ACK]数据包，抓包结果与预期结果一致。
  * Filtered 
    攻击者主机运行TCP connect scan python脚本 
    ![](img/tcpc_filterd.png) 
    靶机Wireshark抓包 
    ![](img/3.png) 
    攻击者主机nmap复刻 
    ![](img/n_tcpc_filterd.png) 
    分析：TCP三次握手机制，攻击者主机向靶机发送连接请求后，没有得到任何响应，抓包结果与预期结果一致。 
* TCP stealth scan 
    原理与TCP connect scan类似 
```
#攻击者主机编写tcpstealth.py
from scapy.all import *
def tcpstealthscan(dst_ip , dst_port , timeout = 10):
    pkts = sr1(IP(dst=dst_ip)/TCP(dport=dst_port,flags="S"),timeout=10)
    if (pkts is None):
        print ("Filtered")
    elif(pkts.haslayer(TCP)):
        if(pkts.getlayer(TCP).flags == 0x12):
            send_rst = sr(IP(dst=dst_ip)/TCP(dport=dst_port,flags="R"),timeout=10)
            print ("Open")
        elif (pkts.getlayer(TCP).flags == 0x14):
            print ("Closed")
        elif(pkts.haslayer(ICMP)):
            if(int(pkts.getlayer(ICMP).type)==3 and int(stealth_scan_resp.getlayer(ICMP).code) in [1,2,3,9,10,13]):
                print ("Filtered")
tcpstealthscan('172.16.111.108',80)
``` 
  * Closed
    ![](img/tcps_closed.png) 
    靶机Wireshark抓包 
    ![](img/4.png) 
    攻击者主机nmap复刻 
    ![](img/n_tcps_closed.png) 
  * Open 
    ![](img/tcps_opend.png) 
    靶机Wireshark抓包 
    ![](img/5.png) 
    攻击者主机nmap复刻 
    ![](img/n_tcps_opend.png) 
  * Filtered 
    ![](img/tcps_filterd.png) 
    靶机Wireshark抓包 
    ![](img/6.png) 
    攻击者主机nmap复刻 
    ![](img/n_tcps_filterd.png) 
* TCP Xmas scan 
```
#攻击者主机编写tcpxmas.py
from scapy.all import *
def Xmasscan(dst_ip , dst_port , timeout = 10):
    pkts = sr1(IP(dst=dst_ip)/TCP(dport=dst_port,flags="FPU"),timeout=10)
    if (pkts is None):
        print ("Open|Filtered")
    elif(pkts.haslayer(TCP)):
        if(pkts.getlayer(TCP).flags == 0x14):
            print ("Closed")
    elif(pkts.haslayer(ICMP)):
        if(int(pkts.getlayer(ICMP).type)==3 and int(pkts.getlayer(ICMP).code) in [1,2,3,9,10,13]):
            print ("Filtered")
Xmasscan('172.16.111.108',80)
``` 
  * Closed
    ![](img/tcpx_closed.png) 
    靶机Wireshark抓包 
    ![](img/7.png) 
    攻击者主机nmap复刻 
    ![](img/n_tcpx_closed.png) 
    分析：Xmas发送TCP请求，在靶机端口关闭状态下，靶机响应[RST，ACK]，抓包结果与预期结果一致。 
  * Open 
    ![](img/tcpx_opend.png) 
    靶机Wireshark抓包 
    ![](img/8.png) 
    攻击者主机nmap复刻 
    ![](img/n_tcpx_opend.png)  
    分析：Xmas发送TCP请求，在靶机端口开放状态下，靶机无响应，抓包结果与预期结果一致。
  * Filtered 
    ![](img/tcpx_filterd.png) 
    靶机Wireshark抓包 
    ![](img/9.png) 
    攻击者主机nmap复刻 
    ![](img/n_tcpx_filterd.png) 
    分析：Xmas发送TCP请求，在靶机端口被过滤状态下，靶机无响应，抓包结果与预期结果一致。 
*  TCP fin scan
   原理与TCP Xmas scan类似
```
#攻击者主机编写tcpfin.py
from scapy.all import *
def finscan(dst_ip , dst_port , timeout = 10):
    pkts = sr1(IP(dst=dst_ip)/TCP(dport=dst_port,flags="F"),timeout=10)#发送FIN包
    if (pkts is None):
        print ("Open|Filtered")
    elif(pkts.haslayer(TCP)):
        if(pkts.getlayer(TCP).flags == 0x14):
            print ("Closed")
    elif(pkts.haslayer(ICMP)):
        if(int(pkts.getlayer(ICMP).type)==3 and int(pkts.getlayer(ICMP).code) in [1,2,3,9,10,13]):
            print ("Filtered")
finscan('172.16.111.108',80)
``` 
  * Closed
    ![](img/tcpf_closed.png) 
    靶机Wireshark抓包 
    ![](img/10.png) 
    攻击者主机nmap复刻 
    ![](img/n_tcpf_closed.png) 
  * Open 
    ![](img/tcpf_opend.png) 
    靶机Wireshark抓包 
    ![](img/11.png) 
    攻击者主机nmap复刻 
    ![](img/n_tcpf_opend.png) 
  * Filtered 
    ![](img/tcpf_filterd.png) 
    靶机Wireshark抓包 
    ![](img/12.png) 
    攻击者主机nmap复刻 
    ![](img/n_tcpf_filterd.png) 
*  TCP null scan 
   原理与TCP Xmas scan类似 
```
#攻击者主机编写tcpnull.py
from scapy.all import *
def nullscan(dst_ip , dst_port , timeout = 10):
    pkts = sr1(IP(dst=dst_ip)/TCP(dport=dst_port,flags=""),timeout=10)
    if (pkts is None):
        print ("Open|Filtered")
    elif(pkts.haslayer(TCP)):
        if(pkts.getlayer(TCP).flags == 0x14):
            print ("Closed")
    elif(pkts.haslayer(ICMP)):
        if(int(pkts.getlayer(ICMP).type)==3 and int(pkts.getlayer(ICMP).code) in [1,2,3,9,10,13]):
            print ("Filtered")
nullscan('172.16.111.108',80)
``` 
  * Closed
    ![](img/tcpn_closed.png) 
    靶机Wireshark抓包 
    ![](img/13.png) 
    攻击者主机nmap复刻 
    ![](img/n_tcpn_closed.png) 
  * Open 
    ![](img/tcpn_opend.png) 
    靶机Wireshark抓包 
    ![](img/14.png) 
    攻击者主机nmap复刻 
    ![](img/n_tcpn_opend.png) 
  * Filtered 
    ![](img/tcpn_filterd.png) 
    靶机Wireshark抓包 
    ![](img/15.png) 
    攻击者主机nmap复刻 
    ![](img/n_tcpn_filterd.png) 
* UDP scan 
```
#攻击者主机编写udp.py
from scapy.all import *
def udpscan(dst_ip,dst_port,dst_timeout = 10):
    resp = sr1(IP(dst=dst_ip)/UDP(dport=dst_port),timeout=dst_timeout)
    if (resp is None):
        print("Open|Filtered")
    elif (resp.haslayer(UDP)):
        print("Open")
    elif(resp.haslayer(ICMP)):
        if(int(resp.getlayer(ICMP).type)==3 and int(resp.getlayer(ICMP).code)==3):
            print("Closed")
        elif(int(resp.getlayer(ICMP).type)==3 and int(resp.getlayer(ICMP).code) in [1,2,9,10,13]):
            print("Filtered")
        elif(resp.haslayer(IP) and resp.getlayer(IP).proto==IP_PROTOS.udp):
            print("Open")
udpscan('172.16.111.108',53)
``` 
  * Closed
    ![](img/udp_closed.png) 
    靶机Wireshark抓包 
    ![](img/16.png) 
    攻击者主机nmap复刻 
    ![](img/n_udp_closed.png) 
    分析：UDP扫描属于开放式扫描，靶机udp/53 端口关闭状态下，对攻击者主机并无任何响应，抓包结果与预期结果一致。 
  * Open 
    ![](img/udp_opend.png) 
    靶机Wireshark抓包 
    ![](img/17.png) 
    攻击者主机nmap复刻 
    ![](img/n_udp_opend.png) 
    分析：UDP扫描属于开放式扫描，靶机udp/53 端口开启状态下，对攻击者主机并无任何响应，无法判断被过滤或开启，抓包结果与预期结果一致。
  * Filtered 
    ![](img/udp_filterd.png) 
    靶机Wireshark抓包 
    ![](img/18.png) 
    攻击者主机nmap复刻 
    ![](img/n_udp_filterd.png) 
    分析：UDP扫描属于开放式扫描，靶机udp/53 端口被过滤状态下，对攻击者主机并无任何响应，无法判断被过滤或开启，抓包结果与预期结果一致。 

#### 参考资料 
* https://c4pr1c3.github.io/cuc-ns/chap0x05/main.html 
* https://c4pr1c3.github.io/cuc-ns-ppt/chap0x05.md.html#/3/1 
* https://github.com/CUCCS/2020-ns-public-AlinaZxy/blob/chap05/chap05/chap05.md
* https://github.com/CUCCS/2020-ns-public-jerrymajerry/blob/chap0x05/%E7%AC%AC%E4%BA%94%E7%AB%A0%E5%AE%9E%E9%AA%8C/%E5%AE%9E%E9%AA%8C%E6%8A%A5%E5%91%8A.md
