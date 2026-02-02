# Docker Deep Dive

## Table of Contents

1. [Networking](#1-networking)
    - [1.1. Single Host Bridge Network](#11-single-host-bridge-network)
        - [1.1.1. Attach containers to default bridge network](#111-attach-containes-to-default-bridge-network)
        - [1.1.2. Create our own bridge network (ps-bridge)](#112-create-our-own-dridge-network-ps-bridge)
        - [1.1.3. Connect existing container to our bridge network](#113-connect-existing-container-to-our-bridge-network)
        - [1.1.4. Understand port mapping](#114-understand-port-mapping)
    - [1.2. Multi-host Overlay Networks](#12-multi-host-overlay-networks)
        - [1.2.1. Create docker swarm cluster of 3 nodes](#121-create-docker-swarm-cluster-of-3-nodes)
        - [1.2.2. Inspect default overlay ingress network](#122-create-new-overlay-network)
        - [1.2.3. Create new overlay network "ps-overlay"](#123-create-new-overlay-network-ps-overlay)
        - [1.2.4. Create new overlay attachable network](#124-create-new-overlay-attachable-network)
    - [1.3. MACVLAN](#13-macvlan)
    - [1.4. IPVLAN](#14-ipvlan)
    - [1.5. Service Discovery](#15-service-discovery)
        - [1.5.1. Create a service and test connectivity and load balancing](#151-create-a-service-and-test-connectivity-and-load-balancing)
        - [1.5.2. Ingress Networking](#152-ingress-networking)
        - [1.5.3. External Load Balancer](#153-external-load-balancer)
## 1. Networking
* On the same machine, docker containers communicate through local docker socket (without going through internet)
* Display available network drivers

```sh
$ docker info --format '{{.Plugins.Network}}'
[bridge host ipvlan macvlan null overlay]
```
### 1.1. Single Host Bridge Network

#### 1.1.1. Attach containes to default bridge network

```sh
$ docker network ls
NETWORK ID     NAME                            DRIVER    SCOPE
03604b758f46   bridge                          bridge    local
2a5eabeb0cd9   host                            host      local
48906b56089c   none                            null      local

$ docker run -dit --name ctr1 alpine ash
$ docker run -dit --name ctr2 alpine ash

$ docker network inspect bridge
...
"Scope": "local",
        "Driver": "bridge",
        "EnableIPv4": true,
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "IPRange": "",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
...
"Containers": {
            "cc63a2381e7a4e2ff8baa59ef96e74dead3c4b00244d4744e3d270a974b34334": {
                "Name": "ctr1",
                "EndpointID": "3a66b273dde80a221700796641115eaccdd5adfe2112a557c212aa6e3b50055b",
                "MacAddress": "b6:9f:b8:87:d5:8e",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            },
            "fbf411d8b60dfba23fda575ccf9938a833a1b28c6df9ff42e65a81b049f75135": {
                "Name": "ctr2",
                "EndpointID": "166a6bb12a9e564cf78da4a2190ad05dab866e2cc088cbfdcaa65ea90c14482b",
                "MacAddress": "76:c9:ae:dc:d3:3b",
                "IPv4Address": "172.17.0.3/16",
                "IPv6Address": ""
            }
        },

....

$ docker attach ctr1
/ # ip addr show
...
eth0@if8: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
    link/ether b6:9f:b8:87:d5:8e brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever

/ # ping -c 4 172.17.0.3
PING 172.17.0.3 (172.17.0.3): 56 data bytes
64 bytes from 172.17.0.3: seq=0 ttl=64 time=0.092 ms
64 bytes from 172.17.0.3: seq=1 ttl=64 time=0.066 ms
64 bytes from 172.17.0.3: seq=2 ttl=64 time=0.066 ms
64 bytes from 172.17.0.3: seq=3 ttl=64 time=0.066 ms

/ # ping -c 4 app.pluralsight.com
PING app.pluralsight.com (104.17.192.111): 56 data bytes
64 bytes from 104.17.192.111: seq=0 ttl=63 time=20.885 ms
64 bytes from 104.17.192.111: seq=1 ttl=63 time=11.002 ms
64 bytes from 104.17.192.111: seq=2 ttl=63 time=18.338 ms
64 bytes from 104.17.192.111: seq=3 ttl=63 time=14.193 ms

```

#### 1.1.2. Create our own dridge network ps-bridge
```sh
$ docker network create --driver bridge ps-bridge

$ docker network inspect ps-bridge

[
    {
        "Name": "ps-bridge",
        "Id": "f6cba76e90bb4e6270f3039e0ab8e86ee1e13f3c8b72edfa2e9ad5a468de3d0f",
        "Created": "2026-02-01T09:20:14.539300754Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv4": true,
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.21.0.0/16",
                    "IPRange": "",
                    "Gateway": "172.21.0.1"
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
        "Options": {
            "com.docker.network.enable_ipv4": "true",
            "com.docker.network.enable_ipv6": "false"
        },
        "Labels": {},
        "Containers": {},
        "Status": {
            "IPAM": {
                "Subnets": {
                    "172.21.0.0/16": {
                        "IPsInUse": 3,
                        "DynamicIPsAvailable": 65533
                    }
                }
            }
        }
    }
]

$ docker run -dit --name ctr3 --network ps-bridge alpine ash
$ docker run -dit --name ctr4 --network ps-bridge alpine ash

$ docker network inspect ps-bridge
[
    {
        "Name": "ps-bridge",
        "Id": "f6cba76e90bb4e6270f3039e0ab8e86ee1e13f3c8b72edfa2e9ad5a468de3d0f",
        "Created": "2026-02-01T09:20:14.539300754Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv4": true,
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.21.0.0/16",
                    "IPRange": "",
                    "Gateway": "172.21.0.1"
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
        "Options": {
            "com.docker.network.enable_ipv4": "true",
            "com.docker.network.enable_ipv6": "false"
        },
        "Labels": {},
        "Containers": {
            "070e19dab8879159672a24b526b7fd6b962c0fb1897038c5e7a9084a904ef31c": {
                "Name": "ctr3",
                "EndpointID": "c29b8e0359d5e45e617e311766bef87a196c48078c89994ce0bf3cf54582e4ab",
                "MacAddress": "c6:7e:86:6a:34:6c",
                "IPv4Address": "172.21.0.2/16",
                "IPv6Address": ""
            },
            "828e973f202dee92a6f53a4d55d4f4ebf053bb9edef827e2865a7d7d19593df7": {
                "Name": "ctr4",
                "EndpointID": "e5439e9ffb76992402079984ac4b3150c5ae835f6a6362bf507b04360fd6e4a1",
                "MacAddress": "0a:bb:72:45:e6:7b",
                "IPv4Address": "172.21.0.3/16",
                "IPv6Address": ""
            }
        },
        "Status": {
            "IPAM": {
                "Subnets": {
                    "172.21.0.0/16": {
                        "IPsInUse": 5,
                        "DynamicIPsAvailable": 65531
                    }
                }
            }
        }
    }
]

$ docker attach ctr3
/ # ping -c 4 172.21.0.3
PING 172.21.0.3 (172.21.0.3): 56 data bytes
64 bytes from 172.21.0.3: seq=0 ttl=64 time=0.082 ms
64 bytes from 172.21.0.3: seq=1 ttl=64 time=0.083 ms
64 bytes from 172.21.0.3: seq=2 ttl=64 time=0.075 ms
64 bytes from 172.21.0.3: seq=3 ttl=64 time=0.073 ms

--- 172.21.0.3 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.073/0.078/0.083 ms

/ # ping -c 4 ctr4
PING ctr4 (172.21.0.3): 56 data bytes
64 bytes from 172.21.0.3: seq=0 ttl=64 time=0.053 ms
64 bytes from 172.21.0.3: seq=1 ttl=64 time=0.078 ms
64 bytes from 172.21.0.3: seq=2 ttl=64 time=0.089 ms
64 bytes from 172.21.0.3: seq=3 ttl=64 time=0.095 ms

--- ctr4 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.053/0.078/0.095 ms

/ # ping -c 4 app.pluralsight.com
PING app.pluralsight.com (104.17.181.111): 56 data bytes
64 bytes from 104.17.181.111: seq=0 ttl=63 time=11.970 ms
64 bytes from 104.17.181.111: seq=1 ttl=63 time=10.041 ms
64 bytes from 104.17.181.111: seq=2 ttl=63 time=10.066 ms
64 bytes from 104.17.181.111: seq=3 ttl=63 time=9.672 ms

--- app.pluralsight.com ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 9.672/10.437/11.970 ms

/ # ping -c 4 ctr2
ping: bad address 'ctr2'
/ # ping -c 4 172.17.0.3
PING 172.17.0.3 (172.17.0.3): 56 data bytes

--- 172.17.0.3 ping statistics ---
4 packets transmitted, 0 packets received, 100% packet loss
```

#### 1.1.3. Connect existing container to our bridge network
```sh
$ docker network connect ps-bridge ctr1

# => Now the container ctr1 is existing into the both bridge networks default bridge and ps-bridge

$ docker attach ctr1

/ # ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0@if15: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
    link/ether 02:2c:90:9a:ba:22 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
3: eth1@if16: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
    link/ether c6:49:43:36:ab:1a brd ff:ff:ff:ff:ff:ff
    inet 172.21.0.2/16 brd 172.21.255.255 scope global eth1
       valid_lft forever preferred_lft forever


/ # ping -c 4 ctr4
PING ctr4 (172.21.0.3): 56 data bytes
64 bytes from 172.21.0.3: seq=0 ttl=64 time=0.264 ms
64 bytes from 172.21.0.3: seq=1 ttl=64 time=0.061 ms
64 bytes from 172.21.0.3: seq=2 ttl=64 time=0.062 ms
64 bytes from 172.21.0.3: seq=3 ttl=64 time=0.108 ms

--- ctr4 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.061/0.123/0.264 ms

# => Now ctr1 can ping ctr4 and also name resolution works
```

#### 1.1.4. Understand port mapping
```sh
$ docker run -dit --name web --network ps-bridge --publish 5000:8080 nigelpoulton/pluralsight-docker-ci

# => we created a container of image nigelpoulton/pluralsight-docker-ci that maps the host port 5000 to container port 8080
# => to reach this web app we should use http://localhost:5000
```

### 1.2. Multi-host Overlay Networks
We will work with 3 nodes (vms). Each node has docker installed on it.

#### 1.2.1. Create docker swarm cluster of 3 nodes
Cluster of 3 nodes (M1, M2, M3):
```sh
M1:# docker node ls
[to do]
M1:# docker network ls
[to do]
M1:# docker swarm init --advertise-addr=192.168.193.121
[to do]
M2:# docker swarm join --token [token]
M3:# docker swarm join --token [token]

M1:# docker node ls
[to do]
M1:# docker network ls
[to do]

M1:# docker network inspect ingress
...
"Ingress": true,
"Peers": [
    { "Name": ...., "IP": "..."},
    { "Name": ...., "IP": "..."},
    { "Name": ...., "IP": "..."},
]
```

#### 1.2.2. create new overlay network
Inspect default overlay ingress network created by swarm
```sh
M1:# docker network ls
[to do]

M1:# docker network inspect ingress
...
"Ingress": true,
"Peers": [
    { "Name": ...., "IP": "..."},
    { "Name": ...., "IP": "..."},
    { "Name": ...., "IP": "..."},
]
```

#### 1.2.3. Create new overlay network "ps-overlay"

```sh
M1:# docker network create --driver overlay --subnet=10.11.0.0/16 --gateway=10.11.0.2 --opt encrypted ps-overlay

M1:# docker network ls
[to do]
M2:# docker network ls
[to do]
not yet visible on node M2 and M3 because overlay networks are lasy. We need to attach to them containers inside the node where to be visible.

M1:# docker service create --name ps-svc --replicas=3 --network ps-overlay alpine sleep infinity

M2:# docker network ls
[to do]

M1:# docker service ps ps-svc
[to do]

M2:# docker network inspect ps-overlay
[to do]
"Containers": {
    .....
    "Name": "ps-svc.1<ID2>"
}

# We use docker exec because we can not attach to sleepy process
M1:# docker exec -it ps-svc.1<ID1>
/ # ping -c 4 ps-svc.1<ID2>
```

#### 1.2.4. create new overlay attachable network
The bridge newtork that we created previously ps-overlay is not attacgable from manual created containers
```sh
M1:# docker container run -dit --name willfail --network ps-overlay alpine ash
[to do]
Error response from daemon: Could not attach to network ps-overlay

M1:# docker network create --driver=overlay --attachable ps-attacher

M1:# docker container run -dit --name willpass --network ps-attacher alpine ash

M2:# docker container run -dit --name ps-pinger --network ps-attacher alpine ash 

M2: docker attach ps-pinger
/ # ping -c 4 willpass
success
```

### 1.3. MACVLAN
MACVLAN allows Docker containers to appear as **physical devices on the local network**.
Each container gets its own **MAC address** and IP from the LAN, and communicates directly
with the external network without NAT.

It requires a Linux host with **kernel 3.9+ (4.x+ recommended)** and a network interface
configured in **promiscuous mode**, since multiple MAC addresses share the same physical NIC.

MACVLAN is typically used for:
- Legacy applications that require direct L2 access
- Network monitoring tools
- Appliances that must be discoverable on the LAN
- Environments where port mapping or NAT is not acceptable

⚠️ Limitation: containers cannot communicate with the Docker host by default.

### 1.4. IPVLAN

IPVLAN is similar to MACVLAN but uses a **single MAC address** shared by the host
while assigning **multiple IP addresses** to containers.

Unlike MACVLAN, IPVLAN does **not require promiscuous mode**, making it easier to
deploy on networks with strict security policies or switch limitations.

It requires a Linux host with **kernel 4.2+** and is typically used when:
- The network does not allow multiple MAC addresses per port
- You need better scalability than MACVLAN
- You want containers to integrate into the LAN at Layer 3

IPVLAN operates in two modes:
- **L2 mode**: containers behave like hosts on the same subnet
- **L3 mode**: routing is required between the host and containers

### 1.5. Service Discovery
Working on docker cluster of 3 nodes <br/>

### Service Discovery

Docker Swarm provides **built-in service discovery** using an internal DNS server.
Each service is automatically assigned a **DNS name** that resolves to the
IP address of one or more service tasks.

Service discovery works **only inside user-defined overlay networks**.

Key points:
* Services can reach each other using the **service name**, not IP addresses
* DNS resolution is **dynamic** and updates automatically when tasks scale or move
* Load balancing is performed at the **service level** (VIP mode by default)
* No external service registry (Consul, etcd) is required

When a service is created, Docker assigns it a Virtual IP (VIP).
Requests sent to the service name are resolved to this VIP.
The Swarm internal load balancer then distributes traffic
to available service tasks.

* Service Discovery modes (important distinction)

Docker Swarm supports two service discovery modes:

1. VIP (default)
   - A single virtual IP per service
   - Built-in load balancing

2. DNSRR (DNS Round Robin)
   - DNS returns multiple task IPs
   - Load balancing handled by the client
```sh
$ docker service create \
  --name web \
  --network app-net \
  --endpoint-mode dnsrr \
  nginx
```

* Not provided by the ingress network
* Not dependent on published ports
* Not visible outside the Swarm cluster

#### 1.5.1 Create a service and test connectivity and load balancing
```sh
M1:# docker network create --driver overlay --attachable marvel

M1:# docker service create --name shield --network marvel --replicas 3 nigelpoulton/k8sbook:shield-01

M1:# docker service ls

M1:# docker network inspect 
# => container Name shield.2.<ID1>
M2:# docker network inspect 
# => container Name shield.2.<ID2>

M1:# docker attach shield.2.<ID1>
/ # ping -c 4 shield.2.<ID2>
success
^ctrl PQ

M1:# docker container run -dit --name zephyr-one --network marvel alpine ash

M1:# docker attach zephyr-one 
/ # ping -c 4 shield.2.<ID2>
success 

# ping service directly
/ # ping -c 4 shield
success 

/ # ping -c 4 shield
/ # ping -c 4 shield
# At each time we see different address responds
# => load balancing
^ctrl PQ

M1:# docker network create --driver overlay --attachable theframework 

M1:# docker service create --name hydra --network theframework --replicas 3 nigelpoulton/k8sbook:hydra-ingress

M1:# docker attach zephyr-one 
/ # ping -c 4 hydra
bad address 'hydra' 
```
#### 1.5.2. Ingress Networking
Service connectivity works because Docker Swarm automatically creates
an **ingress overlay network** when Swarm mode is initialized.
This network enables external access and internal load balancing
for services that publish ports.

* We should **not deploy application services directly on the default ingress network**.
* The ingress network is used for **service routing and load balancing**, not service discovery.
* Behind the scenes, Docker uses two main components on each node:
  - **docker_gwbridge**: connects the ingress network to the host network
  - **ingress-sbox** (sandbox): handles packet processing and routing for ingress traffic

When traffic arrives from outside the cluster, it enters through the
docker_gwbridge, is processed by the ingress sandbox, forwarded to the
ingress overlay network, and finally routed to the service task
attached to the application’s custom overlay network.

```sh
M1:# docker network ls
...
docker_gwbridge bridge  local
ingress         overlay swarm

M1:# docker network inspect ingress
...
Containers:
  "ingress-sbox": .... "IPv4Address": "10.0.0.2/24"

M1:# docker network inspect docker_gwbridge
...
Containers:
  "ingress-sbox": .... "IPv4Address": "172.18.0.2/16"

# => ingress-sbox:
#     10.0.0.2 (ingress overlay)
#     172.18.0.02 (docker_gwbridge)
```

```sh
M1:# docker network create --driver overlay --attachable -o encrypted app-net

M1:# docker service create --name app-svc --replicas 2 --network app-net --publish published=5000,target=8080 nigelpoulton/k8sbook:shield-01
```

When accessing the application from outside the cluster using:

<docker-node-public-ip>:5000

the request can be sent to **any Swarm node**, even if the service
task is not running on that node.

Docker Swarm’s **internal load balancer** (routing mesh) forwards
each request to one of the available service replicas.
As a result, successive requests may be handled by **different containers**.

#### 1.5.3. External Load Balancer

In production environments, it is common to place an **external load balancer**
(e.g. AWS ELB / ALB, HAProxy, NGINX, F5) in front of the Swarm cluster.

The external load balancer:
* Targets all Swarm nodes
* Forwards traffic to the published application port (5000)
* Distributes traffic across nodes

Docker Swarm then performs **internal load balancing**
to route requests to the appropriate service replicas.

* The external load balancer balances traffic across nodes,
while Docker Swarm balances traffic across service replicas.

* The routing mesh allows a published service port to be reachable
on every node, regardless of where the containers are running.