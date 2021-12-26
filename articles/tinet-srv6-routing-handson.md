---
title: "tinet を用いた SRv6 動作検証"
emoji: "🍣"
type: "tech"
topics: ["network", "srv6", "tinet"]
published: false
---

https://github.com/tinynetwork/tinet は、 YAML 形式の設定ファイルを記述することで任意の Network Namespace で分割された複数コンテナをローカルマシン内に簡単に展開する事ができるツールです。

tinet を実行すると `ip` (iproute2), `docker`, `ovs-vsctl` (Open vSwitch) コマンドや任意のコマンドを標準出力してくれ、これをパイプ越しにシェルに食わせることで、設定ファイルに記述した環境を構築出来ます。

tinet によりユーザは設定ファイルを書くだけでネットワークトポロジを簡単に構築可能なため、ルーティングプロトコルの動作検証を行う際にとても便利です。

今回はこの tinet を利用して、 examples として配置されている設定を使い SRv6 の動作を確認しました。

# 検証ネットワークの説明

今回の動作検証では以下を利用しました。

* https://github.com/tinynetwork/tinet/tree/17e20e789e8d252bee8bc47c5eb1fe8685d7950b/examples/basic_srv6/linux/hands_on

Router が 8 台おり、ネットワーク構成は以下のようになっています。

# 構築

構築手順です。

## 環境情報

* Ubuntu 20.04.3 (5.4.0-81-generic)
* Docker version 20.10.11
* tinet v0.0.2

上記のインストール手順は省略します。

## 構築手順

tinet を利用し環境を構築します。

* `tinet up` : iproute2 と docker を利用して、任意の Network Namespace へのインタフェースを持つコンテナを構築

```bash
tinet up -c examples/basic_srv6/linux/hands_on/spec.yaml | sudo sh -x
```

* 実行結果

:::details クリックで展開

```bash
$ tinet up -c examples/basic_srv6/linux/hands_on/spec.yaml | sudo sh -x
+ docker run -td --net none --name R1 --rm --privileged --hostname R1 -v /tmp/tinet:/tinet slankdev/frr
+ mkdir -p /var/run/netns
+ docker inspect R1 --format {{.State.Pid}}
+ PID=67003
+ ln -s /proc/67003/ns/net /var/run/netns/R1
+ docker run -td --net none --name R2 --rm --privileged --hostname R2 -v /tmp/tinet:/tinet slankdev/frr
+ mkdir -p /var/run/netns
+ docker inspect R2 --format {{.State.Pid}}
+ PID=67112
+ ln -s /proc/67112/ns/net /var/run/netns/R2
+ docker run -td --net none --name R3 --rm --privileged --hostname R3 -v /tmp/tinet:/tinet slankdev/frr
+ mkdir -p /var/run/netns
+ docker inspect R3 --format {{.State.Pid}}
+ PID=67192
+ ln -s /proc/67192/ns/net /var/run/netns/R3
+ docker run -td --net none --name R4 --rm --privileged --hostname R4 -v /tmp/tinet:/tinet slankdev/frr
+ mkdir -p /var/run/netns
+ docker inspect R4 --format {{.State.Pid}}
+ PID=67379
+ ln -s /proc/67379/ns/net /var/run/netns/R4
+ docker run -td --net none --name R5 --rm --privileged --hostname R5 -v /tmp/tinet:/tinet slankdev/frr
+ mkdir -p /var/run/netns
+ docker inspect R5 --format {{.State.Pid}}
+ PID=67461
+ ln -s /proc/67461/ns/net /var/run/netns/R5
+ docker run -td --net none --name R6 --rm --privileged --hostname R6 -v /tmp/tinet:/tinet slankdev/frr
+ mkdir -p /var/run/netns
+ docker inspect R6 --format {{.State.Pid}}
+ PID=67543
+ ln -s /proc/67543/ns/net /var/run/netns/R6
+ docker run -td --net none --name R7 --rm --privileged --hostname R7 -v /tmp/tinet:/tinet slankdev/frr
+ mkdir -p /var/run/netns
+ docker inspect R7 --format {{.State.Pid}}
+ PID=67639
+ ln -s /proc/67639/ns/net /var/run/netns/R7
+ docker run -td --net none --name R8 --rm --privileged --hostname R8 -v /tmp/tinet:/tinet slankdev/frr
+ mkdir -p /var/run/netns
+ docker inspect R8 --format {{.State.Pid}}
+ PID=67719
+ ln -s /proc/67719/ns/net /var/run/netns/R8
+ ip link add net0 netns R1 type veth peer name net1 netns R2
+ ip netns exec R1 ip link set net0 up
+ ip netns exec R2 ip link set net1 up
+ ip link add net0 netns R2 type veth peer name net0 netns R3
+ ip netns exec R2 ip link set net0 up
+ ip netns exec R3 ip link set net0 up
+ ip link add net1 netns R3 type veth peer name net0 netns R4
+ ip netns exec R3 ip link set net1 up
+ ip netns exec R4 ip link set net0 up
+ ip link add net2 netns R3 type veth peer name net0 netns R6
+ ip netns exec R3 ip link set net2 up
+ ip netns exec R6 ip link set net0 up
+ ip link add net3 netns R3 type veth peer name net0 netns R7
+ ip netns exec R3 ip link set net3 up
+ ip netns exec R7 ip link set net0 up
+ ip link add net4 netns R3 type veth peer name net0 netns R8
+ ip netns exec R3 ip link set net4 up
+ ip netns exec R8 ip link set net0 up
+ ip link add net1 netns R4 type veth peer name net0 netns R5
+ ip netns exec R4 ip link set net1 up
+ ip netns exec R5 ip link set net0 up
+ make -C /home/vagrant/git/srdump install_docker
make: *** /home/vagrant/git/srdump: No such file or directory.  Stop.
+ ip netns del R1
+ ip netns del R2
+ ip netns del R3
+ ip netns del R4
+ ip netns del R5
+ ip netns del R6
+ ip netns del R7
+ ip netns del R8
```

:::

* `tinet conf` : spec.yaml の `.node_configs` に与えられたコマンドを、 `docker exec` を通じて任意の Node に対し実行

```bash
$ tinet conf -c examples/basic_srv6/linux/hands_on/spec.yaml | sudo sh -x
```

* 実行結果
    * `sysctl: cannot stat` は、 コンテナ内の DOWN しているインタフェースに対し設定しようとして発生したエラーで、今回の検証に影響なし
    * `ZEBRA: Disabling MPLS support (no kernel support)` は、今回 MPLS を利用しないため影響なし

:::details クリックで展開

```bash
$ tinet conf -c examples/basic_srv6/linux/hands_on/spec.yaml | sudo sh -x
+ docker exec R1 bash -c enable_seg6_router.py | sh
sysctl: cannot stat /proc/sys/net/ipv6/conf/tunl0/disable_ipv6: No such file or directory
sysctl: cannot stat /proc/sys/net/ipv6/conf/tunl0/seg6_enabled: No such file or directory
+ docker exec R1 /usr/lib/frr/frr start
2021/12/05 07:35:14 warnings: ZEBRA: Disabling MPLS support (no kernel support)
+ docker exec R1 ip addr add cafe:1::1/64 dev net0
+ docker exec R1 vtysh -c conf t -c interface lo -c  ipv6 address fc00:1::/64 -c router ospf6 -c  interface lo area 0.0.0.0 -c  ospf6 router-id 10.255.0.1 -c  interface net0 area 0.0.0.0
+ docker exec R2 bash -c enable_seg6_router.py | sh
sysctl: cannot stat /proc/sys/net/ipv6/conf/tunl0/disable_ipv6: No such file or directory
sysctl: cannot stat /proc/sys/net/ipv6/conf/tunl0/seg6_enabled: No such file or directory
+ docker exec R2 /usr/lib/frr/frr start
2021/12/05 07:35:15 warnings: ZEBRA: Disabling MPLS support (no kernel support)
+ docker exec R2 ip addr add cafe:2::2/64 dev net0
+ docker exec R2 ip addr add cafe:1::2/64 dev net1
+ docker exec R2 ip -6 route add default via cafe:2::3
+ docker exec R2 vtysh -c conf t -c interface lo -c  ipv6 address fc00:2::/64 -c router ospf6 -c  ospf6 router-id 10.255.0.2 -c  interface lo area 0.0.0.0 -c  interface net0 area 0.0.0.0 -c  interface net1 area 0.0.0.0
+ docker exec R2 ip route add cafe:4::5 encap seg6 mode encap segs fc00:6::1,fc00:7::1,fc00:4::1 dev net0
+ docker exec R3 bash -c enable_seg6_router.py | sh
sysctl: cannot stat /proc/sys/net/ipv6/conf/tunl0/disable_ipv6: No such file or directory
sysctl: cannot stat /proc/sys/net/ipv6/conf/tunl0/seg6_enabled: No such file or directory
+ docker exec R3 /usr/lib/frr/frr start
2021/12/05 07:35:17 warnings: ZEBRA: Disabling MPLS support (no kernel support)
+ docker exec R3 ip addr add cafe:2::3/64 dev net0
+ docker exec R3 ip addr add cafe:3::3/64 dev net1
+ docker exec R3 ip addr add cafe:5::3/64 dev net2
+ docker exec R3 ip addr add cafe:6::3/64 dev net3
+ docker exec R3 ip addr add cafe:7::3/64 dev net4
+ docker exec R3 vtysh -c conf t -c interface lo -c  ipv6 address fc00:3::/64 -c router ospf6 -c  ospf6 router-id 10.255.0.3 -c  interface lo area 0.0.0.0 -c  interface net0 area 0.0.0.0 -c  interface net1 area 0.0.0.0 -c  interface net2 area 0.0.0.0 -c  interface net3 area 0.0.0.0 -c  interface net4 area 0.0.0.0
+ docker exec R4 bash -c enable_seg6_router.py | sh
sysctl: cannot stat /proc/sys/net/ipv6/conf/tunl0/disable_ipv6: No such file or directory
sysctl: cannot stat /proc/sys/net/ipv6/conf/tunl0/seg6_enabled: No such file or directory
+ docker exec R4 /usr/lib/frr/frr start
2021/12/05 07:35:19 warnings: ZEBRA: Disabling MPLS support (no kernel support)
+ docker exec R4 ip addr add cafe:3::4/64 dev net0
+ docker exec R4 ip addr add cafe:4::4/64 dev net1
+ docker exec R4 vtysh -c conf t -c interface lo -c  ipv6 address fc00:4::/64 -c router ospf6 -c  ospf6 router-id 10.255.0.4 -c  interface lo area 0.0.0.0 -c  interface net0 area 0.0.0.0 -c  interface net1 area 0.0.0.0
+ docker exec R4 ip route add fc00:4::1/128 encap seg6local action End.DX6 nh6 cafe:4::5 dev net0
+ docker exec R5 bash -c enable_seg6_router.py | sh
sysctl: cannot stat /proc/sys/net/ipv6/conf/tunl0/disable_ipv6: No such file or directory
sysctl: cannot stat /proc/sys/net/ipv6/conf/tunl0/seg6_enabled: No such file or directory
+ docker exec R5 /usr/lib/frr/frr start
2021/12/05 07:35:21 warnings: ZEBRA: Disabling MPLS support (no kernel support)
+ docker exec R5 ip addr add cafe:4::5/64 dev net0
+ docker exec R5 vtysh -c conf t -c interface lo -c  ipv6 address fc00:5::/64 -c router ospf6 -c  ospf6 router-id 10.255.0.5 -c  interface lo area 0.0.0.0 -c  interface net0 area 0.0.0.0 -c  interface net1 area 0.0.0.0
+ docker exec R6 bash -c enable_seg6_router.py | sh
sysctl: cannot stat /proc/sys/net/ipv6/conf/tunl0/disable_ipv6: No such file or directory
sysctl: cannot stat /proc/sys/net/ipv6/conf/tunl0/seg6_enabled: No such file or directory
+ docker exec R6 /usr/lib/frr/frr start
2021/12/05 07:35:23 warnings: ZEBRA: Disabling MPLS support (no kernel support)
+ docker exec R6 ip addr add cafe:5::6/64 dev net0
+ docker exec R6 vtysh -c conf t -c interface lo -c  ipv6 address fc00:6::/64 -c router ospf6 -c  ospf6 router-id 10.255.0.6 -c  interface net0 area 0.0.0.0 -c  interface lo area 0.0.0.0
+ docker exec R6 ip route add fc00:6::1/128 encap seg6local action End dev net0
+ docker exec R7 bash -c enable_seg6_router.py | sh
sysctl: cannot stat /proc/sys/net/ipv6/conf/tunl0/disable_ipv6: No such file or directory
sysctl: cannot stat /proc/sys/net/ipv6/conf/tunl0/seg6_enabled: No such file or directory
+ docker exec R7 /usr/lib/frr/frr start
2021/12/05 07:35:25 warnings: ZEBRA: Disabling MPLS support (no kernel support)
+ docker exec R7 ip addr add cafe:6::7/64 dev net0
+ docker exec R7 vtysh -c conf t -c interface lo -c  ipv6 address fc00:7::/64 -c router ospf6 -c  ospf6 router-id 10.255.0.7 -c  interface net0 area 0.0.0.0 -c  interface lo area 0.0.0.0
+ docker exec R7 ip route add fc00:7::1/128 encap seg6local action End dev net0
+ docker exec R8 bash -c enable_seg6_router.py | sh
sysctl: cannot stat /proc/sys/net/ipv6/conf/tunl0/disable_ipv6: No such file or directory
sysctl: cannot stat /proc/sys/net/ipv6/conf/tunl0/seg6_enabled: No such file or directory
+ docker exec R8 /usr/lib/frr/frr start
2021/12/05 07:35:27 warnings: ZEBRA: Disabling MPLS support (no kernel support)
+ docker exec R8 ip addr add cafe:7::8/64 dev net0
+ docker exec R8 vtysh -c conf t -c interface lo -c  ipv6 address fc00:8::/64 -c router ospf6 -c  ospf6 router-id 10.255.0.8 -c  interface net0 area 0.0.0.0 -c  interface lo area 0.0.0.0
+ docker exec R8 ip route add fc00:8::1/128 encap seg6local action End dev net0
```
:::

# 動作検証

検証についてまとめます。

## 疎通確認①: 初期状態

* R1 から R5 に対して ping を打つ

```bash
$ sudo docker exec -it R1 ping cafe:4::5
PING cafe:4::5(cafe:4::5) 56 data bytes
64 bytes from cafe:4::5: icmp_seq=1 ttl=61 time=0.133 ms
64 bytes from cafe:4::5: icmp_seq=2 ttl=61 time=0.271 ms
64 bytes from cafe:4::5: icmp_seq=3 ttl=61 time=0.256 ms
64 bytes from cafe:4::5: icmp_seq=4 ttl=61 time=0.253 ms
64 bytes from cafe:4::5: icmp_seq=5 ttl=61 time=0.259 ms
```

* R6 で tcpdump を実行

```bash
$ docker exec -it R6 tcpdump
TODO
```

* R7 で tcpdump を実行

```bash
$ docker exec -it R7 tcpdump
14:00:08.537733 IP6 cafe:2::2 > fc00:7::1: srcrt (len=6, type=4, segleft=1[|srcrt]
14:00:08.537755 IP6 cafe:2::2 > fc00:4::1: srcrt (len=6, type=4, segleft=0[|srcrt]
14:00:09.565720 IP6 cafe:2::2 > fc00:7::1: srcrt (len=6, type=4, segleft=1[|srcrt]
14:00:09.565732 IP6 cafe:2::2 > fc00:4::1: srcrt (len=6, type=4, segleft=0[|srcrt]
14:00:10.149341 IP6 fe80::6cf7:f5ff:febd:c0bf > ff02::5: OSPFv3, Hello, length 40
14:00:10.149763 IP6 fe80::44a0:29ff:feda:b037 > ff02::5: OSPFv3, Hello, length 40
...
```

* R6 で tcpdump を実行

```bash
$ docker exec -it R8 tcpdump
... (OSPFv3 Hello パケットのみ出力される)
```


* R7 でパケットヘッダを確認

```bash

$ docker exec -it R7 tcpdump -X
14:25:01.465788 IP6 cafe:2::2 > fc00:7::1: srcrt (len=6, type=4, segleft=1[|srcrt]
        0x0000:  600f 3be6 00a0 2b3c cafe 0002 0000 0000  `.;...+<........
        0x0010:  0000 0000 0000 0002 fc00 0007 0000 0000  ................
        0x0020:  0000 0000 0000 0001 2906 0401 0200 0000  ........).......
        0x0030:  fc00 0004 0000 0000 0000 0000 0000 0001  ................
        0x0040:  fc00 0007 0000 0000 0000 0000 0000 0001  ................
        0x0050:  fc00 0006 0000 0000 0000 0000 0000 0001  ................
        0x0060:  600f 3be6 0040 3a40 cafe 0001 0000 0000  `.;..@:@........
        0x0070:  0000 0000 0000 0001 cafe 0004 0000 0000  ................
        0x0080:  0000 0000 0000 0005 8000 c3f7 0179 06f1  .............y..
        0x0090:  bdcb ac61 0000 0000 ed1a 0700 0000 0000  ...a............
        0x00a0:  1011 1213 1415 1617 1819 1a1b 1c1d 1e1f  ................
        0x00b0:  2021 2223 2425 2627 2829 2a2b 2c2d 2e2f  .!"#$%&'()*+,-./
        0x00c0:  3031 3233 3435 3637                      01234567
14:25:01.465815 IP6 cafe:2::2 > fc00:4::1: srcrt (len=6, type=4, segleft=0[|srcrt]
        0x0000:  600f 3be6 00a0 2b3b cafe 0002 0000 0000  `.;...+;........
        0x0010:  0000 0000 0000 0002 fc00 0004 0000 0000  ................
        0x0020:  0000 0000 0000 0001 2906 0400 0200 0000  ........).......
        0x0030:  fc00 0004 0000 0000 0000 0000 0000 0001  ................
        0x0040:  fc00 0007 0000 0000 0000 0000 0000 0001  ................
        0x0050:  fc00 0006 0000 0000 0000 0000 0000 0001  ................
        0x0060:  600f 3be6 0040 3a40 cafe 0001 0000 0000  `.;..@:@........
        0x0070:  0000 0000 0000 0001 cafe 0004 0000 0000  ................
        0x0080:  0000 0000 0000 0005 8000 c3f7 0179 06f1  .............y..
        0x0090:  bdcb ac61 0000 0000 ed1a 0700 0000 0000  ...a............
        0x00a0:  1011 1213 1415 1617 1819 1a1b 1c1d 1e1f  ................
        0x00b0:  2021 2223 2425 2627 2829 2a2b 2c2d 2e2f  .!"#$%&'()*+,-./
        0x00c0:  3031 3233 3435 3637                      01234567

...
```

Segment Routing Header (SRH) より以下のことが分かります。

* Segment List には R4 (`fc00:0004::1`), R6 (`fc00:0006::1`), R7 (`fc00:0007::1`) がある
    * R2 にて `R6 -> R7 -> R4` の順番でルーティングさせるよう設定されているため、Segment List の値は意図通り
* `14:25:01.465788 IP6 cafe:2::2 > fc00:7::1` のパケットの Segments Left の値は `01`
    * R2 (`cafe:2::2`) を src, R7 (`fc00:0007::1`) を dst としたパケット (つまり R6 から転送された inbound packet) の Segment は `fc00:0007::1` であることを指す
* `14:25:01.465815 IP6 cafe:2::2 > fc00:4::1` のパケットの Segments Left の値は `00`
    * R2 (`cafe:2::2`) を src, R4 (`fc00:0004::1`) を dst としたパケット (つまり R7 から転送する outbound packet) の Segment は `fc00:0004::1` であることを指す

以上より、`R1 -> (R2 -> R3 -> R6 -> R3 -> R7 -> R3 -> R4) -> R5` とルーティングされていることが分かる。また `( )` で囲われた部分は `R2 -> R6 -> R7 -> R4` と Segment Routing されていることが分かる。

## 疎通確認② R2 において、Segment Routing を行うエントリをルーティングテーブルから削除する

TODO

以上より、OSPF で広報された `cafe:5::/64` のエントリにマッチし `R1 -> R2 -> R3 -> R4 -> R5` と通常の IPv6 ルーティングされていることが分かる。

## 疎通確認③: R2 において、SR Node の SID を Segment List に R8 を追加する

TODO

以上より、新たに追加された R8 を通り `R1 -> (R2 -> R3 -> R6 -> R3 -> R7 -> R3 -> R8 -> R3 -> R4) -> R5` とルーティングされていることが分かる。また `( )` で囲われた部分は `R2 -> R6 -> R7 -> R8 -> R4` と Segment Routing されていることが分かる。

# 感想

もともと Segment Routing どころか IPv6 もろくに触ったことが無かったのですが、この機会にどこにどのように設定することでソースルーティングが実現できるのかを動かしながら理解することが出来ました。
今回利用した SRv6 Function 以外にどのような Function があるのか、それにより新たに何が出来るようになるのかは分かっていないので、別途学んでいきたいです。

あとは、 初めて tinet を利用してみましたが、今まで libvirt や OpenStack を使い VM を用いて構築していたものがとても簡単に構築できるようになり感動しました。これからもネットワーク検証の機会があれば使っていこうと思います。
