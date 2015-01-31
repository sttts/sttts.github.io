---
layout: post
title: Poor Men's Cluster-Unique IPs for Docker Clusters
date:   2015-01-31 18:22:07
categories: docker network
---

Following up from {% post_url 2015-01-22-weave %} I want to line out how to setup Docker in a way that every container gets its own cluster-wide unique IP, without any overlay network layer. This for sure is neither rocket science, nor anything new. But as I see so many people using Docker in a cluster environment, still fighting with port management, I want to write down how this can be implemented in a few steps:

##  0. The Goal and What We Need

We aim at

- a cluster-wide, unique, automatically assigned IP address for every container
- which is routable (switchable) within a private network
- without any DHCP or other complicated infrastructure
- without an overlay network.

We need

- a large enough free IP range (`10.1.0.0/16` in this example)
- a private network interface (`eth1` in this example)
- `bridge-tools` installed on two Docker hosts (Ubuntu 14.04 in this example).

## 1. Basic Idea

We setup our own custom bridge `br0` with `eth1` inside on each node. We use this as the Docker bridge – instead of the usual `docker0`.

We assign a subnet of the private IP range for each node, without overlaps. We pass this range as `--fixed-cidr` to Docker. Docker will select container IPs inside this range by itself, without any help of external DHCP services.

## 2. Setting up the Bridge

We setup the bridge in `/etc/network/interfaces`:

```
auto br0
iface br0 inet static
    address 10.1.0.1
    netmask 255.255.0.0
    bridge_ports eth1
    bridge_stp off
    bridge_fd 0
```

on each node. We choose IPs of the shape `10.1.0.x` as host addresses. 

Then we start the bridge with

```bash
$ ifup br0
```

If there was an address on `eth1` before, make sure to remove it before this with

```bash
$ ifconfig eth1 0.0.0.0
```

## 3. Changing the Docker Daemon Network Settings

We tell Docker about our bridge and the network IP range to use in `/etc/default/docker`:

```bash
DOCKER_OPTS="--bridge=br0 --fixed-cidr=10.1.1.0/24"
```

Here, the network range `10.1.1.0` must be chosen according to the host, i.e. the host with `10.1.0.x` gets `10.1.x.0/24` as its IP range for the containers.

Now restart Docker and check that Docker assigns the right addresses:

```bash
$ restart docker
$ docker run -it ubuntu /bin/bash
root@508b511dab3e:/# ip addr show dev eth0
7: eth0: <BROADCAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:0a:01:01:02 brd ff:ff:ff:ff:ff:ff
    inet 10.1.1.2/16 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:aff:fe01:102/64 scope link 
       valid_lft forever preferred_lft forever
```

## 4. Cross Host Connectivity

Make sure IPv4 forwarding is active:

```bash
$ cat /proc/sys/net/ipv4/ip_forward
1
```

If you use VirtualBox to test this (e.g. in Vagrant), make sure to allow promiscuous mode in the VM network settings.

Test that you can access container ports on another host:

```bash
node2 $ docker run -itd sttts/python-ubuntu:latest python -m SimpleHTTPServer 80
29351797f33af256c60d79b5ad24fccb4f4a83b5a3bb5bd204e9eab38db77308
{% raw %}
node2 $ docker inspect --format '{{ .NetworkSettings.IPAddress }}' 2935
{% endraw %}
10.1.2.5

node1 $ curl -I 10.1.2.5:8080
HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/2.7.6
Date: Sat, 31 Jan 2015 17:12:02 GMT
Content-type: text/html; charset=ANSI_X3.4-1968
Content-Length: 810
```

Your network works. Congratulations.
