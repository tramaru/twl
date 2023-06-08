# Docker の DNS ってどうなっているの？

## 目次
- Docker network に関して触ってみた
- Embedded DNS server を触ってみた

## Docker network に関して触ってみた
Nginx のコンテナを作成し Port 番号と IP アドレスを確認する。
| container | port | IP |
| nginx | 80 | 172.17.0.2 |

```console
% docker run -p 80:80 --name nginx -d nginx
% docker port nginx
80/tcp -> 0.0.0.0:80
% docker inspect --format '{{ .NetworkSettings.IPAddress }}' nginx
172.17.0.2
```

Docker network の一覧を取得する。

```console
% docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
d7372ca404c7   bridge    bridge    local
```

bridge network 配下にあるコンテナの情報を確認する。
nginx コンテナは、`172.17.0.2/16` の IP を持って存在していることがわかる。

```console
% docker network inspect bridge
[
  {
    "Name": "bridge",
    "Id": "d7372ca404c76a996a4261f16c7897a2ed6d622359bb3ab31e5954abb7c1b968",
    "Created": "2023-06-07T14:26:45.994556875Z",
    "Scope": "local",
    "Driver": "bridge",
    "EnableIPv6": false,
    "IPAM": {
      "Driver": "default",
      "Options": null,
      "Config": [
        {
            "Subnet": "172.17.0.0/16",
            "Gateway": "172.17.0.1"
        }
      ]
    },
    "Internal": false,
    "Attachable": false,
    "Ingress": false,
    "ConfigFrom": {
        "Network": ""
    },
    "ConfigOnly": false,
    "Containers": {
      "cde8c8ef7218ae7eccfb40755cfce896b1a98b68aa0609649dc7eed726b1a8c4": {
        "Name": "nginx",
        "EndpointID": "17cd13ee74486e6892485966f9b84d57baf6b91cce4d4989598cc60e76b345cb",
        "MacAddress": "02:42:ac:11:00:02",
        "IPv4Address": "172.17.0.2/16",
        "IPv6Address": ""
      }
    },
    "Options": {
      "com.docker.network.bridge.default_bridge": "true",
      "com.docker.network.bridge.enable_icc": "true",
      "com.docker.network.bridge.enable_ip_masquerade": "true",
      "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
      "com.docker.network.bridge.name": "docker0",
      "com.docker.network.driver.mtu": "65535"
    },
    "Labels": {}
  }
]
```

自分自身で docker network を作り出すこともできる。

```console
% docker network create test_network
c1a49e49f7bb01a0eac8f5d24cbe7b3db21eeeed504c0098fcc8a99b66e4bd7c
% docker network ls
NETWORK ID     NAME           DRIVER    SCOPE
c1a49e49f7bb   test_network   bridge    local
```

作成した `test_network` 配下に新しく作成した `nginx_second` を結びつけてみる。

```console
% docker run -d --name nginx-second --network test_network nginx
6a4a6b2044f5178ded59607e91b0ff19fc3ca7e4dabdfba6110cf2611bd94db0

% docker network inspect test_network
[
  {
    "Name": "test_network",
    "Id": "c1a49e49f7bb01a0eac8f5d24cbe7b3db21eeeed504c0098fcc8a99b66e4bd7c",
    "Scope": "local",
    "Driver": "bridge",
    "IPAM": {
      "Driver": "default",
      "Options": {},
      "Config": [
        {
          "Subnet": "172.19.0.0/16",
          "Gateway": "172.19.0.1"
        }
      ]
    },
    "Containers": {
      "6a4a6b2044f5178ded59607e91b0ff19fc3ca7e4dabdfba6110cf2611bd94db0": {
        "Name": "nginx-second",
        "EndpointID": "e2472b58e10fb22a1f7a5c995bcf04ec660be417c1db62b8660b737a2446fbd3",
        "MacAddress": "02:42:ac:13:00:02",
        "IPv4Address": "172.19.0.2/16",
        "IPv6Address": ""
      }
    },
  }
]
```

事前に作成していた `nginx` コンテナも `test_netwrok` に結びつけることができる。 
```console
% docker network connect test_network nginx
% docker network inspect test_network
[
    {
      "Name": "test_network",
      "Id": "c1a49e49f7bb01a0eac8f5d24cbe7b3db21eeeed504c0098fcc8a99b66e4bd7c",
      "Created": "2023-06-07T22:21:06.062927007Z",
      "Scope": "**local**",
      "Driver": "bridge",
      "IPAM": {
        "Driver": "default",
        "Options": {},
        "Config": [
          {
            "Subnet": "172.19.0.0/16",
            "Gateway": "172.19.0.1"
          }
        ]
      },
      "Containers": {
        "6a4a6b2044f5178ded59607e91b0ff19fc3ca7e4dabdfba6110cf2611bd94db0": {
          "Name": "nginx-second",
          "EndpointID": "e2472b58e10fb22a1f7a5c995bcf04ec660be417c1db62b8660b737a2446fbd3",
          "MacAddress": "02:42:ac:13:00:02",
          "IPv4Address": "172.19.0.2/16",
          "IPv6Address": ""
        },
        "cde8c8ef7218ae7eccfb40755cfce896b1a98b68aa0609649dc7eed726b1a8c4": {
          "Name": "nginx",
          "EndpointID": "1cd035590cee1f58a8aca7b5d1f51bc8e966f670aa91060d6d1f10bca0f5a7b6",
          "MacAddress": "02:42:ac:13:00:03",
          "IPv4Address": "172.19.0.3/16",
          "IPv6Address": ""
        }
      },
    }
]
```
nginx コンテナを見ると下記の２つの network につながっているのが確認できる
- bridge
- test_network

```console
% docker inspect nginx
"Networks": {
  "bridge": {
    "IPAMConfig": null,
    "Links": null,
    "Aliases": null,
    "NetworkID": "d7372ca404c76a996a4261f16c7897a2ed6d622359bb3ab31e5954abb7c1b968",
    "EndpointID": "17cd13ee74486e6892485966f9b84d57baf6b91cce4d4989598cc60e76b345cb",
    "Gateway": "172.17.0.1",
    "IPAddress": "172.17.0.2",
    "IPPrefixLen": 16,
    "IPv6Gateway": "",
    "GlobalIPv6Address": "",
    "GlobalIPv6PrefixLen": 0,
    "MacAddress": "02:42:ac:11:00:02",
    "DriverOpts": null
  },
  "test_network": {
    "IPAMConfig": {},
    "Links": null,
    "Aliases": [
        "cde8c8ef7218"
    ],
    "NetworkID": "c1a49e49f7bb01a0eac8f5d24cbe7b3db21eeeed504c0098fcc8a99b66e4bd7c",
    "EndpointID": "1cd035590cee1f58a8aca7b5d1f51bc8e966f670aa91060d6d1f10bca0f5a7b6",
    "Gateway": "172.19.0.1",
    "IPAddress": "172.19.0.3",
    "IPPrefixLen": 16,
    "IPv6Gateway": "",
    "GlobalIPv6Address": "",
    "GlobalIPv6PrefixLen": 0,
    "MacAddress": "02:42:ac:13:00:03",
    "DriverOpts": {}
  }
}
```

紐付けた `test_network` から `nginx` コンテナを外すこともできる
```console
% docker network disconnect test_network nginx             
% docker % docker inspect nginx
"NetworkSettings": {
  "Bridge": "",
  "SandboxID": "dbe7c2658ff3d0e3ce50ede340160f6c0fe9da813f191be64df44c020402c5fb",
  "HairpinMode": false,
  "LinkLocalIPv6Address": "",
  "LinkLocalIPv6PrefixLen": 0,
  "Ports": {
    "80/tcp": [
      {
        "HostIp": "0.0.0.0",
        "HostPort": "80"
      }
    ]
  },
  "SandboxKey": "/var/run/docker/netns/dbe7c2658ff3",
  "SecondaryIPAddresses": null,
  "SecondaryIPv6Addresses": null,
  "EndpointID": "17cd13ee74486e6892485966f9b84d57baf6b91cce4d4989598cc60e76b345cb",
  "Gateway": "172.17.0.1",
  "GlobalIPv6Address": "",
  "GlobalIPv6PrefixLen": 0,
  "IPAddress": "172.17.0.2",
  "IPPrefixLen": 16,
  "IPv6Gateway": "",
  "MacAddress": "02:42:ac:11:00:02",
  "Networks": {
    "bridge": {
      "IPAMConfig": null,
      "Links": null,
      "Aliases": null,
      "NetworkID": "d7372ca404c76a996a4261f16c7897a2ed6d622359bb3ab31e5954abb7c1b968",
      "EndpointID": "17cd13ee74486e6892485966f9b84d57baf6b91cce4d4989598cc60e76b345cb",
      "Gateway": "172.17.0.1",
      "IPAddress": "172.17.0.2",
      "IPPrefixLen": 16,
      "IPv6Gateway": "",
      "GlobalIPv6Address": "",
      "GlobalIPv6PrefixLen": 0,
      "MacAddress": "02:42:ac:11:00:02",
      "DriverOpts": null
    }
  }
}
```

## Docker DNS を触ってみた

そもそも DNS って何が嬉しいの？
コンテナは立ち上げるたびに IP アドレスが動的に変化していく、なので IP アドレスを常に指定するのはとても大変である。
なのでコンテナのサービス名で指定できるのはすごく助かる。

### DNS を確認するための準備
先ほどと同様に Docker network を作ります。
その中に、2 つの Ubuntu コンテナを入れて、お互いの疎通を確認する形でみていきます。

`hikido_network` の作成
```
% docker network create hikido_network
7e7c87c12cd9e2363bba17b0600aa4aef7456f179d47ebdd17c8f277f2863398
```

利用するシンプルな `ubuntu` イメージ用の Dockerfile を作成し、２つのコンテナを立ち上げる。
```
# Dockerfile
FROM ubuntu:latest

RUN apt-get update && \
  apt-get install -y iputils-ping netcat dnsutils && \
  apt-get clean && \
  rm -rf /var/lib/apt/lists/*

CMD ["tail", "-f", "/dev/null"]
```
```
% docker build -t ubuntu .
```
```
% docker run -d --name first --network hikido_network ubuntu
9452d3f199efab041aa683a8b07d498620df1a2f735c4f3b442f265f4ac8b8b3
% docker run -d --name second --network hikido_network ubuntu
1a942235497e1d6b0e8180d1db662a27645cd261a404ea88f6c9bc927b88aad8
% docker ps
CONTAINER ID   IMAGE     COMMAND               CREATED         STATUS         PORTS     NAMES
1a942235497e   ubuntu    "tail -f /dev/null"   4 minutes ago   Up 4 minutes             second
9452d3f199ef   ubuntu    "tail -f /dev/null"   4 minutes ago   Up 4 minutes             first
```

`hikido_network` のなかに作成した２つの `ubuntu` コンテナがいるのを確認できる。

```console
% docker network inspect hikido_network
[
  {
    "Name": "hikido_network",
    "Id": "7e7c87c12cd9e2363bba17b0600aa4aef7456f179d47ebdd17c8f277f2863398",
    "Created": "2023-06-07T22:52:34.653598923Z",
    "Scope": "local",
    "Driver": "bridge",
    "EnableIPv6": false,
    "IPAM": {
      "Driver": "default",
      "Options": {},
      "Config": [
        {
          "Subnet": "172.20.0.0/16",
          "Gateway": "172.20.0.1"
        }
      ]
    },
    "Containers": {
      "1a942235497e1d6b0e8180d1db662a27645cd261a404ea88f6c9bc927b88aad8": {
        "Name": "second",
        "EndpointID": "3491774df0bcf6fdd65323647a82d9a5a845b2e3945b010fd9e7d8d2a9869ee3",
        "MacAddress": "02:42:ac:14:00:03",
        "IPv4Address": "172.20.0.3/16",
        "IPv6Address": ""
      },
      "9452d3f199efab041aa683a8b07d498620df1a2f735c4f3b442f265f4ac8b8b3": {
        "Name": "first",
        "EndpointID": "5b81fe1f0b208692c1ef5ae1e1e95718563680cfca33448387be7212f01d139b",
        "MacAddress": "02:42:ac:14:00:02",
        "IPv4Address": "172.20.0.2/16",
        "IPv6Address": ""
      }
    },
  }
]
```

## DNS を利用して通信できるか確認

お互いの `ubuntu` コンテナがコンテナ名で疎通ができることを確認し問題なさそう。
```console
% docker container exec -it first ping -c 3 second
PING second (172.20.0.3) 56(84) bytes of data.
64 bytes from second.hikido_network (172.20.0.3): icmp_seq=1 ttl=64 time=0.123 ms
64 bytes from second.hikido_network (172.20.0.3): icmp_seq=2 ttl=64 time=0.221 ms
64 bytes from second.hikido_network (172.20.0.3): icmp_seq=3 ttl=64 time=0.291 ms

--- second ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2034ms
rtt min/avg/max/mdev = 0.123/0.211/0.291/0.068 ms
```
```console
docker % docker container exec -it second ping -c 3 first
PING first (172.20.0.2) 56(84) bytes of data.
64 bytes from first.hikido_network (172.20.0.2): icmp_seq=1 ttl=64 time=0.142 ms
64 bytes from first.hikido_network (172.20.0.2): icmp_seq=2 ttl=64 time=0.163 ms
64 bytes from first.hikido_network (172.20.0.2): icmp_seq=3 ttl=64 time=0.150 ms

--- first ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2038ms
rtt min/avg/max/mdev = 0.142/0.151/0.163/0.008 ms
```

## どのように DNS で名前解決しているんだろう？

アタッチされている network から取得した、DNS サーバーの情報が `/etc/resolv.conf` に記載される。
DNS サーバーは下記の IP で動いていることがわかる。
- https://docs.docker.com/network/#dns-services
- この DNS サーバーを embedded DNS server と呼び、Docker daemon 上で動いているそう。
```console
/# cat /etc/resolv.conf
nameserver 127.0.0.11
options ndots:0
```

実際に `second` コンテナの IP を取得するために `nslookup` を利用して問い合わせてみると、
`127.0.0.11` に確認しに行ってるので、使われていることが分かる。
```console
% docker container exec -it first bash
/# nslookup second
Server:         127.0.0.11
Address:        127.0.0.11#53

Non-authoritative answer:
Name:   second
Address: 172.20.0.3
```

### 次の課題
では、実際にどのようにこの DNS サーバーが実装されているのかを確認してみたい

## 参考にした記事
- https://docs.docker.com/network/
- https://vegibit.com/how-does-docker-dns-work/
- https://dev.classmethod.jp/articles/docker-service-discovery/

