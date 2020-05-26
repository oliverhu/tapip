Very useful project to understand how tcp/ip works at source code level. Better than reading the TCP/IP Illustrated series out of no where.. it is appararently simpler than reading the Linux source code, but this project is more complicated than I thought.. taking some learning notes.

Some useful docs are under /doc folder.

```
$ cloc ./
      80 text files.
      78 unique files.
       7 files ignored.

github.com/AlDanial/cloc v 1.82  T=0.05 s (1488.4 files/s, 163686.6 lines/s)
-------------------------------------------------------------------------------
Language                     files          blank        comment           code
-------------------------------------------------------------------------------
C                               37            525            749           4913
C/C++ Header                    22            229             82           1234
make                            10             45              4            184
Bourne Shell                     2              9             32             49
Markdown                         1              8              0             44
JSON                             2              0              0             31
-------------------------------------------------------------------------------
SUM:                            74            816            867           6455
-------------------------------------------------------------------------------
```

# ip_frag.c

> IP fragmentation

IP fragmentation is an Internet Protocol (IP) process that breaks packets into smaller pieces (fragments), so that the resulting pieces can pass through a link with a smaller maximum transmission unit (MTU) than the original packet size. The fragments are reassembled by the receiving host.

# loop.c

> Loop device

`eno1` is the onboard Ethernet (wired) adapter.

`lo` is a loopback device. You can imagine it as a virtual network device that is on all systems, even if they aren't connected to any network. It has an IP address of 127.0.0.1 and can be used to access network services locally. For example, if you run a webserver on your machine and browse to it with Firefox or Chromium on the same machine, then it will go via this network device.

There is no wi-fi adapter listed. lspci and lsusb may help you find them in the first place at which point you need to figure out why it isn't working.

# veth.c

> virtual ethernet device

Very important file. `veth_poll()` is the function to poll event sent to the veth device.
Try
```
@@ -125,8 +126,11 @@ void veth_poll(void)
                pfd.revents = 0;
                /* one event, infinite time */
                ret = poll(&pfd, 1, -1);
-               if (ret <= 0)
+               if (ret <= 0) {
                        perrx("poll /dev/net/tun");
+               } else {
+                       dbg("polled a message.");
+               }
                /* get a packet and handle it */
                veth_rx();
        }
```
and ping 10.0.0.1. You will see "polled a message" for every icmp message. After each package received, `veth_rx()` will process the packet.

The size of the packet will be the net_mtu size + the header size of an ethernet frame, thus `network_device-> net_mtu + ETH_HRD_SZ`.

`calloc()` gives you a zero-initialized buffer, while `malloc()` leaves the memory uninitialized.

When we compile from scratch, we saw this warning:
```
pkb.c: In function ‘alloc_pkb’:
pkb.c:38:12: warning: taking address of packed member of ‘struct pkbuf’ may result in an unaligned pointer value [-Waddress-of-packed-member]
   38 |  list_init(&pkb->pk_list);
```
What is a packed member??
```
What is Packing
Packing, on the other hand prevents compiler from doing padding means remove the unallocated space allocated by structure.

In case of Linux we use __attribute__((__packed__))  to pack structure.
In case of Windows (specially in Dev c++) use  # pragma pack (1) to pack structure.
firmcodes.com/structure-padding-and-packing-in-c-example/
```
Not sure how to fix besides disabling packed here.. tba.

After `pkb` is allocated, `veth_recv()` is called to read the data from tap device (note tap is ethernet level device, tun is ip level device).

The package is then passed to the upper ip layer via `net_in(veth, pkb)`

In `net_in()` method, it calls into

# net.c

> Ethernet -> IP level transition

`eth_init()` casts the pkb's data into `ether` struct, which represents an ethernet frame. By comparing some bits, we know if the package is PKT_BROADCAST or PKT_MULTICAST. Based on ethernet proto type, we can parse the eframe in either ARP or IP format (there are more, but the ones we care are ip and arp).

In the `./tapip` shell, typing `debug ip` to get into l2 debug mode. When we ping 10.0.0.1 from localhost, you will see
```
[17340]ip_in ip 10.0.0.2 -> 10.0.0.1(20/84 bytes)
[17340]rt_output ip Find route entry from 10.0.0.1 to 10.0.0.2
[17340]ip_send_out ip 10.0.0.1 -> 10.0.0.2(20/84 bytes)
[17340]ip_in ip 10.0.0.2 -> 239.255.255.250(20/200 bytes)
[17340]ip_forward ip 10.0.0.2 -> 239.255.255.250(20/200 bytes) forwarding
[17340]ip_in ip 10.0.0.2 -> 10.0.0.1(20/84 bytes)
[17340]rt_output ip Find route entry from 10.0.0.1 to 10.0.0.2
[17340]ip_send_out ip 10.0.0.1 -> 10.0.0.2(20/84 bytes)
```
Localhost and our tap device are connected via a 10.0.0.1/24 mask. When you run `ifconfig` in localhost, you get:
```
tap0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.0.2  netmask 255.255.255.0  broadcast 10.0.0.255
        inet6 fe80::9c06:4ff:feb4:79a  prefixlen 64  scopeid 0x20<link>
        ether 9e:06:04:b4:07:9a  txqueuelen 1000  (Ethernet)
        RX packets 31  bytes 3270 (3.2 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 171  bytes 31574 (31.5 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```
In the tapip shell, you type `ifconfig` and get:
```
[net shell]: ifconfig
lo        HWaddr 00:00:00:00:00:00
          IPaddr 127.0.0.1
          mtu 1500
          RX packet:0 bytes:0 errors:0
          TX packet:0 bytes:0 errors:0
veth      HWaddr 00:34:45:67:89:ab
          IPaddr 10.0.0.1
          mtu 1500
          RX packet:98 bytes:19310 errors:0
          TX packet:10 bytes:841 errors:0

--- NOTE: this nic isnt in tapip, it is in remote machine ---
tap0      HWaddr 9e:06:04:b4:07:9a
          IPaddr 10.0.0.2
          mtu 1500
          RX packet:0 bytes:0 errors:0
          TX packet:0 bytes:0 errors:0
```

After understanding the package type based on the ethernet prototype, we can finally move up to the next layer. ip & arp.

# ip_in.c

> ip layer

`ip_in(struct netdev *, struct pkbuf *)` first does some sanity check and use nothl to convert the bytes between network to host. The package was later sent forwardto `ip_recv_route(pkbuf *)`. I don't know what does `route.c` do here, but it checks against a linked list and make sure the ip_dst of the package is for our host. If yes, invoke `ip_recv_local(pkb)` otherwise `ip_forward(pkb)`. Not sure how forwarding works here, but it seems the default TOP 1 topology doesn't support forwarding. We will look at ip_forward function later. The package will later be cast into an `ip` struct.

We will check if this ip package is a fragment or via checking the `fragment offset` field

*Fragment Offset*

The fragment offset field is measured in units of eight-byte blocks. It is 13 bits long and specifies the offset of a particular fragment relative to the beginning of the original unfragmented IP datagram. The first fragment has an offset of zero. This allows a maximum offset of (213 – 1) × 8 = 65,528 bytes, which would exceed the maximum IP packet length of 65,535 bytes with the header length included (65,528 + 20 = 65,548 bytes).

_ip_frag.c_ will lookup and assumble the complete ip packet and return full ip package if a complete (by checking if frag_flags matches 0x00000001) ip packet is assembled, otherwise return NULL. Next we will move up to the transport layer.

To test functions in `ip_frag.c`, we can write a few lines of Pythons script:
```
>>> import socket
>>> s = socket.socket(socket.AF_INET,socket.SOCK_DGRAM)
>>> s.sendto('helloworld' * 10000,('10.0.0.2',12345))
```

In the `tapip` shell, you will see (in debug ip mode):
```
[net shell]: debug ip
enter ^C to exit debug mode
[54514]ip_in ip 10.0.0.2 -> 10.0.0.1(20/1500 bytes)
[54514]ip_reass ip ID:7262 RS:0 DF:0 MF:1 OFF:0 bytes size:1500 bytes
[54514]ip_in ip 10.0.0.2 -> 10.0.0.1(20/1500 bytes)
[54514]ip_reass ip ID:7262 RS:0 DF:0 MF:1 OFF:1480 bytes size:1500 bytes
[54514]ip_in ip 10.0.0.2 -> 10.0.0.1(20/1500 bytes)
[54514]ip_reass ip ID:7262 RS:0 DF:0 MF:1 OFF:2960 bytes size:1500 bytes
[54514]ip_in ip 10.0.0.2 -> 10.0.0.1(20/1500 bytes)
[54514]ip_reass ip ID:7262 RS:0 DF:0 MF:1 OFF:4440 bytes size:1500 bytes
[54514]ip_in ip 10.0.0.2 -> 10.0.0.1(20/1500 bytes)
[54514]ip_reass ip ID:7262 RS:0 DF:0 MF:1 OFF:5920 bytes size:1500 bytes
[54514]ip_in ip 10.0.0.2 -> 10.0.0.1(20/1418 bytes)
[54514]ip_reass ip ID:7262 RS:0 DF:0 MF:0 OFF:7400 bytes size:1418 bytes
[54514]reass_frag ip resassembly success(20/8818 bytes)
```
`pkbdbg(pkbuf* )` is a useful function to print the details of a pkbuf structure.

`https://github.com/oliverhu/tapip/blob/master/doc/test` contains a list of ways if we want to test tcp/udp connections.

sock is the socket in the network stack domain, socket is the user facing part.

`_ntohs` converts values between host and network byte order. We need to apply this function when we are trying to print out stuff.

Turn on tcpstate debug to understand the state changes.

```
[net shell]: debug -n tcp tcpstate
[net shell]: snc -d  -b 0.0.0.0:12345
[36950]recv_packet (TCP): bind 0.0.0.0:12345
[36950]recv_tcp_packet (TCP): listen with backlog:10

[36949]tcp_in tcp 40 bytes, real 40 bytes
[36949]tcp_segment_init tcp from 10.0.0.2:55072 to 10.0.0.1:12345	seq:3563457793(0:1) ack:0 SYN
[36949]tcp_dbg_state tcpstate LISTEN
[36949]tcp_listen tcpstate LISTEN
[36949]tcp_listen tcpstate 1. check rst
[36949]tcp_listen tcpstate 2. check ack
[36949]tcp_listen tcpstate 3. check syn
[36949]tcp_set_state tcpstate State from CLOSED to SYN-RECV
[36949]tcp_send_synack tcp send SYN(12345679)/ACK(4096) [WIN -731509502] to 10.0.0.1:55072
[36949]tcp_in tcp 20 bytes, real 20 bytes
[36949]tcp_segment_init tcp from 10.0.0.2:55072 to 10.0.0.1:12345	seq:3563457794(0:0) ack:12345680 ACK
[36949]tcp_dbg_state tcpstate SYN-RECV
[36949]tcp_process tcpstate 1. check seq
[36949]tcp_process tcpstate 2. check rst
[36949]tcp_process tcpstate 3. NO check security and precedence
[36949]tcp_process tcpstate 4. check syn
[36949]tcp_process tcpstate 5. check ack
[36949]tcp_synrecv_ack tcpstate Passive three-way handshake successes!
[36949]tcp_set_state tcpstate State from SYN-RECV to ESTABLISHED
[36949]tcp_process tcpstate 6. check urg
[36949]tcp_process tcpstate 7. segment text
[36949]tcp_process tcpstate 8. check fin
[36950]recv_tcp_packet (TCP): Three-way handshake successes: from 10.0.0.2:55072

[36949]tcp_in tcp 24 bytes, real 20 bytes
[36949]tcp_segment_init tcp from 10.0.0.2:55072 to 10.0.0.1:12345	seq:3563457794(4:4) ack:12345680 PSH|ACK
[36949]tcp_dbg_state tcpstate ESTABLISHED
[36949]tcp_process tcpstate 1. check seq
[36949]tcp_process tcpstate 2. check rst
[36949]tcp_process tcpstate 3. NO check security and precedence
[36949]tcp_process tcpstate 4. check syn
[36949]tcp_process tcpstate 5. check ack
[36949]tcp_process tcpstate SND.UNA 12345680 < SEG.ACK 12345680 <= SND.NXT 12345680
[36949]tcp_process tcpstate 6. check urg
[36949]tcp_process tcpstate 7. segment text
[36949]tcp_process tcpstate 8. check fin
[36949]tcp_send_ack tcp send ACK(3563457798) [WIN 4092] to 10.0.0.2:55072
[36950]recv_tcp_packet (TCP): starting _read()...
123
[36949]tcp_in tcp 20 bytes, real 20 bytes
[36949]tcp_segment_init tcp from 10.0.0.2:55072 to 10.0.0.1:12345	seq:3563457798(0:1) ack:12345680 FIN|ACK
[36949]tcp_dbg_state tcpstate ESTABLISHED
[36949]tcp_process tcpstate 1. check seq
[36949]tcp_process tcpstate 2. check rst
[36949]tcp_process tcpstate 3. NO check security and precedence
[36949]tcp_process tcpstate 4. check syn
[36949]tcp_process tcpstate 5. check ack
[36949]tcp_process tcpstate SND.UNA 12345680 < SEG.ACK 12345680 <= SND.NXT 12345680
[36949]tcp_process tcpstate 6. check urg
[36949]tcp_process tcpstate 7. segment text
[36949]tcp_process tcpstate 8. check fin
[36949]tcp_set_state tcpstate State from ESTABLISHED to CLOSE-WAIT
[36949]tcp_send_ack tcp send ACK(3563457799) [WIN 4096] to 10.0.0.2:55072
[36950]recv_tcp_packet (TCP): last _read() return 0
[36950]tcp_send_fin tcp send FIN(12345680)/ACK(3563457799) [WIN 4096] to 10.0.0.2:55072
[36949]tcp_in tcp 20 bytes, real 20 bytes
[net shell]: [36949]tcp_segment_init tcp from 10.0.0.2:55072 to 10.0.0.1:12345	seq:3563457799(0:0) ack:12345681 ACK
[36949]tcp_lookup_sock_establish Found a socket from tcp_lookip
[net shell]: [36949]tcp_recv found socket.
[36949]tcp_dbg_state tcpstate LAST-ACK
[net shell]: [36949]tcp_process tcpstate 1. check seq
[36949]tcp_process tcpstate 2. check rst
[net shell]: [36949]tcp_process tcpstate 3. NO check security and precedence
[36949]tcp_process tcpstate 4. check syn
[36949]tcp_process tcpstate 5. check ack
[36949]tcp_process tcpstate SND.UNA 12345680 < SEG.ACK 12345681 <= SND.NXT 12345681
[36949]tcp_set_state tcpstate State from LAST-ACK to CLOSED
[36949]tcp_send_ack tcp send ACK(3563457799) [WIN 4096] to 10.0.0.2:55072
```