# Nmap_Bypass_IDS

扫描参数：nmap -sS -sV -O -vv 1.1.1.1

### nmap 系统识别绕过ids检测

nmap使用-O参数扫描时会发送tcp，udp，icmp 然后在根据响应的tcp window，TTL，IPID等对比指纹库识别操作系统，IDS识别nmap扫描一般都是根据UDP data区域填充的'C'字符串,ICMP填充的是0（正常windows下是a-z，Linux下是0-9）

alert udp $EXTERNAL_NET 10000: -> $HOME_NET 10000: (msg:"ET SCAN NMAP OS Detection Probe"; dsize:300; content:"CCCCCCCCCCCCCCCCCCCC"; fast_pattern:only; content:"CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC"; depth:255; content:"CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC"; within:45; classtype:attempted-recon; sid:2018489; rev:3; metadata:created_at 2014_05_20, updated_at 2014_05_20;)

https://raw.githubusercontent.com/nmap/nmap/master/osscan2.cc

static u8 patternbyte = 0x43; /* character 'C' */ 替换为  static u8 patternbyte = 0x46; /* character 'F' */

### nmap UA 修改

alert http $EXTERNAL_NET any -> $HOME_NET any (msg:"ET SCAN Nmap Scripting Engine User-Agent Detected (Nmap Scripting Engine)"; flow:to_server,established; content:"Mozilla/5.0 (compatible|3b| Nmap Scripting Engine"; nocase; http_user_agent; depth:46; reference:url,doc.emergingthreats.net/2009358; classtype:web-application-attack; sid:2009358; rev:5; metadata:created_at 2010_07_30, updated_at 2010_07_30;)

修改nselib/http.lua

USER_AGENT = stdnse.get_script_args('http.useragent') or "Mozilla/5.0 (compatible; )"

### TCP window 修改tcp window 窗口大小

nmap 默认扫描的tcp window size 大小是1024，将其修改为10240来绕过ids

alert tcp $EXTERNAL_NET any -> $HOME_NET any (msg:"ET SCAN NMAP -sS window 1024"; fragbits:!D; dsize:0; flags:S,12; ack:0; window:1024; threshold: type both, track by_dst, count 1, seconds 60; reference:url,doc.emergingthreats.net/2009582; classtype:attempted-recon; sid:2009582; rev:3; metadata:created_at 2010_07_30, updated_at 2010_07_30;)

tcpip.cc:729:    tcp->th_win = htons(1024); /* Who cares */

 if (window)
 
    tcp->th_win = htons(10240); /* Who cares */

### 修改nmap-service-probes

alert tcp $EXTERNAL_NET any -> $HOME_NET $HTTP_PORTS (msg:"ET SCAN NMAP SIP Version Detect OPTIONS Scan"; flow:established,to_server; content:"OPTIONS sip|3A|nm SIP/"; depth:19; classtype:attempted-recon; sid:2018317; rev:1; metadata:created_at 2014_03_25, updated_at 2014_03_25;)

-Probe TCP SIPOptions q|OPTIONS sip:nm SIP/2.0\r\nVia: SIP/2.0/TCP nm;branch=foo\r\nFrom: <sip:nm@nm>;tag=root\r\nTo: <sip:nm2@nm2>\r\nCall-ID: 50000\r\nCSeq: 42 OPTIONS\r\nMax-Forwards: 70\r\nContent-Length: 0\r\nContact: <sip:nm@nm>\r\nAccept: application/sdp\r\n\r\n|

去掉OPTIONS sip

### 修改3389 cookie

nmap-service-probes  
Probe TCP TerminalServerCookie q|\x03\0\0*%\xe0\0\0\0\0\0Cookie: mstshash=nmap\r\n\x01\0\x08\0\x03\0\0\0|  
nselib/rdp.lua  
local cookie = "mstshash=nmap"  

### Zmap识别
很多网络扫描器会使用固定的tcp window或者ipid，这些都可以作为特征识别的一部分。  
例如在使用zmap扫描时，通过抓包分析发现zmap使用了固定的IPID 54321以及固定的window值 65535，那么我可以写一个IDS规则来检测它  
```
alert tcp any any -> any any (msg:"Detect ZMAP scan"; dsize:0; flags:S; window:65535; id:54321; threshold: type both, track by_src, count 60, seconds 60; sid:19010201; rev:1; metadata:created_at 2019_01_02, by al0ne;)
```

alert 是记录告警日志， tcp $EXTERNAL_NET any -> $HOME_NET any 这些对应着协议 源ip 源端口 方向 目标ip 目标端口，这里是匹配任意ip和任意端口，
dsize代表tcp data区大小扫描时没有附加其他数据所以为0，flags:S 是指tcp的flags为s即三次握手时syn请求，并且计算阀值60秒内出现60才告警 

修改绕过  
src/probe_modules/packet.c  
```
tcp_header->th_win = htons(65535);
iph->ip_id = htons(54321);
```
