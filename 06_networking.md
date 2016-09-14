## Container networking basics
We will now run network services (accepting requests) in containers.

At the end of this lesson, you will be able to: 
* Run a network service in a container.
* Manipulate container networking basics.
* Find a container's IP address.

We will also explain the different network models used by Docker.

----

### A simple webserver

Run the Docker Hub image nginx, which contains a basic web server:
```bash
docker run -d -P nginx
```

Docker will download the image from the Docker Hub.
* `-d` tells Docker to run the image in the background.
* `-P` tells Docker to make this service reachable from other computers.  
(-P is the short version of --publish-all.)

But, how do we connect to our web server now?

----

### Connect to our webserver

Find the port of the webserver:
```bash
docker ps
... PORTS                                           NAMES
... 0.0.0.0:32772->80/tcp, 0.0.0.0:32771->443/tcp   clever_chandrasekhar
```

* The webserver is running on port 80/443 inside the container
* Those ports are mapped to port 32772 and 32771 on the host  

Connect:
```bash
open http://0.0.0.0:32772
```

This will open the default browser and connect to the container on port 32772

----

### Why mapping ports?

* We are out of IPv4 addresses.
* Containers cannot have public IPv4 addresses.
* They have private addresses.
* Services have to be exposed port by port.
* Ports have to be mapped to avoid conflicts.

----

### Manual allocation of ports
If you want to set port numbers yourself, no problem:
```bash
docker run -d -p 80:80 nginx
docker run -d -p 8000:80 nginx
```

* We are running two NGINX web servers.
* The first one is exposed on port 80.
* The second one is exposed on port 8000.

Note: the convention is port-on-host:port-on-container

----

### Bringing container to your infrastructure

There are (at least) three ways to integrate containers in your network.
* Start the container, letting Docker allocate a public port for it.  
Then retrieve that port number and feed it to your configuration.
* Pick a fixed port number in advance, when you generate your configuration.
Then start your container by setting the port numbers manually.  
* Use an overlay network, connecting your containers with e.g. VLANs, tunnels, plugins,…

----

### Locating the container IP address

We can use the docker inspect command to find the IP address of the container.

```bash
docker inspect --format '{{ json .NetworkSettings.IPAddress }}' $(docker ps -lq)
"172.17.0.3"
```

* `docker inspect` is an advanced command, that can retrieve a ton of information about our containers.
* Here, we provide it with a format string to extract exactly the private IP address of the container.

----

### Networking before Engine 1.9
A container could use one of the following drivers:
* bridge (default)
* none
* host
* container

----

### Bridge mode (default)
* By default, the container gets a virtual eth0 interface.  
(In addition to its own private lo loopback interface.)
* That interface is provided by a veth pair.
* It is connected to the Docker bridge.  
(Named docker0 by default; configurable with --bridge.)
* Addresses are allocated on a private, internal subnet.  
(Docker uses 172.17.0.0/16 by default; configurable with --bip.)
* Outbound traffic goes through an iptables MASQUERADE rule.
* Inbound traffic goes through an iptables DNAT rule.
* The container can have its own routes, iptables rules, etc.

----

### Bridge network

![Bridge](networking_bridge.png)

----

### The null driver

* Container is started with docker run --net none ...
* It only gets the lo loopback interface. No eth0.
* It can't send or receive network traffic.
* Useful for isolated/untrusted workloads.

----

### The host driver

* Container is started with docker run --net host ...
* It sees (and can access) the network interfaces of the host.
* it can bind any address, any port (for ill and for good).
* Network traffic doesn't have to go through NAT, bridge, or veth.
* Performance = native

----

### The container driver
* Container is started with docker run --net container:id …
* It re-uses the network stack of another container.
* It shares with this other container the same interfaces, IP address(es), routes, iptables rules, etc.
* Those containers can communicate over their lo interface.  
(i.e. one can bind to 127.0.0.1 and the others can connect to it.)

----

### Networking after Engine 1.9

All the previous drivers are still available.
Docker now has the notion of a network, and a new top-level command to manipulate
and see those networks: docker network.
```bash
docker network ls
NETWORK ID          NAME                   DRIVER              SCOPE
1670debb4a0d        bridge                 bridge              local               
9cfb79738802        dockerzipkin_default   bridge              local               
74edd3fad0d5        friday                 bridge              local               
7e51bf6c1c8f        ghost_default          bridge              local               
75e8b87360a8        host                   host                local               
f0ece9ce4283        none                   null                local               
52dbfccfa3f5        test                   bridge              local               
72614ed08fb1        weave                  weavemesh           local
```

----

### What is a network in Docker?

* Conceptually, a network is a virtual switch.
* It can be local (to a single Engine) or global (across multiple hosts).
* A network has an IP subnet associated to it.
* A network is managed by a driver.
* A network can have a custom IPAM (IP allocator).
* Containers with explicit names are discoverable via DNS.
* All the drivers that we have seen before are available.
* A new multi-host driver, overlay, is available.
* More drivers can be provided by plugins (OVS, VLAN...)

----

### Creating a network
Let's create a network.
```bash
docker network create workshop-tage
```
The network is now visible with the `network ls` command:
```bash
docker network ls
NETWORK ID          NAME                   DRIVER              SCOPE
1670debb4a0d        bridge                 bridge              local               
9cfb79738802        dockerzipkin_default   bridge              local               
74edd3fad0d5        friday                 bridge              local               
7e51bf6c1c8f        ghost_default          bridge              local               
75e8b87360a8        host                   host                local               
f0ece9ce4283        none                   null                local               
52dbfccfa3f5        test                   bridge              local               
72614ed08fb1        weave                  weavemesh           local               
d1294237576a        workshop-tage          bridge              local      
```

----

### Connecting containers to a network

We will create two named containers on this network.
First, let's create this container in the background.
```bash
docker run -dti --name con1 --net workshop-tage alpine sh
```
Now, create another container in this network in the foreground.
```bash
docker run -ti --name con2 --net workshop-tage alpine sh
```

----

### Communication between containers
From our new container con2, we can resolve and ping con1, using its assigned name:
```bash
ping -c4 con1
PING con1 (172.18.0.2): 56 data bytes
64 bytes from 172.18.0.2: seq=0 ttl=64 time=0.111 ms
64 bytes from 172.18.0.2: seq=1 ttl=64 time=0.098 ms
64 bytes from 172.18.0.2: seq=2 ttl=64 time=0.101 ms
64 bytes from 172.18.0.2: seq=3 ttl=64 time=0.130 ms

--- con1 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.098/0.110/0.130 ms
```

How does it work?

----

### Resolving container addresses

Before Docker Engine 1.10:
Docker updates `/etc/hosts` each time containers are added/removed.
```
172.18.0.2 con1
172.18.0.2 con1.workshop-tage
```
After Docker Engine 1.10:
As of Docker 1.10, the docker daemon implements an embedded DNS server which provides built-in service discovery for any container created with a valid name or net-alias or aliased by link.

----

### Connecting to multiple networks

Let's create a new network and container in this network:
```bash
docker network create ch-open
docker run -ti --name con3 --net ch-open alpine sh
```
con3 can’t ping a container in a different network
```
ping -c4 con1
ping: bad address 'con1'
```

You need to connect the container to the new network:
```bash
docker network connect workshop-tage con3
 michael.mueller  ... / projects / docker-workshop-1d / c  master  docker attach con3
/ # ping -c1 con1
PING con1 (172.18.0.2): 56 data bytes
64 bytes from 172.18.0.2: seq=0 ttl=64 time=0.104 ms

--- con1 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 0.104/0.104/0.104 ms
```

----

### Implementation details
With the "bridge" network driver, each container joining a network receives a new virtual interface.
```bash
docker run --net container:con3 alpine ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: sit0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN qlen 1
    link/sit 0.0.0.0 brd 0.0.0.0
3: ip6tnl0@NONE: <NOARP> mtu 1452 qdisc noop state DOWN qlen 1
    link/tunnel6 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00 brd 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00
4: ip6gre0@NONE: <NOARP> mtu 1448 qdisc noop state DOWN qlen 1
    link/[823] 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00 brd 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00
349: eth0@if350: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:ac:13:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.19.0.2/16 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe13:2/64 scope link 
       valid_lft forever preferred_lft forever
351: eth1@if352: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:ac:12:00:04 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.4/16 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe12:4/64 scope link 
       valid_lft forever preferred_lft forever
```

----

### Multi-host networking
Out of the scope for this workshop!
Very short instructions:
* deploy a key/value store (Consul, etcd, Zookeeper)
* add two extra flags to your Docker Engine
* you can now create networks using the overlay driver!

When you create a network on one host with the overlay driver, it appears automatically on all other hosts.  
Containers placed on the same networks are able to resolve and ping as if they were local.  
The overlay network is based on VXLAN and store neighbor info in a key/value store.

----

### Do it yourself
* Create a network
* Create a nginx container in this network
* Create a ubuntu container in this network
* From the ubuntu container use `curl` to access the default website of nginx

Hint: curl needs to be installed `apt-get update && apt-get install -y curl`

----

### Possible Solution
```bash
docker run -dtiP --name web --net workshop nginx
docker run -ti --name ubuntu --net workshop ubuntu
curl -i http://web:80
```

----

### Summary

We've learned how to:
* Expose a network port.
* Manipulate container networking basics.
* Find a container's IP address.
* Create private networks for groups of containers.
