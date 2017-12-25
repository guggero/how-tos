# HOW TO setup IPv6 networking using Docker

This guide will explain how you can set up IPv6 networking for your Docker container on Ubuntu 16.04.
Most likely everything will work quite similarly on other Linux distributions but I will focus on Ubuntu.

## Requirements

This text assumes that your system meets the following requirements:

### Server with natively routed IPv6 network
Of course you can only (easily) use IPv6 if your provider natively supports IPv6. This usually means that you have a whole /64 or even
/48 network routed to your server. You can check if that is the case with the command `ifconfig`:
```bash
root@dockerhost ~/ # ifconfig
eth0 Link encap:Ethernet HWaddr 90:1b:0e:00:aa:bb 
 inet addr:88.99.11.22 Bcast:88.99.11.255 Mask:255.255.255.255
 inet6 addr: fe80::921b:eff:febd:f718/64 Scope:Link
 inet6 addr: 2a01:4f8:a:b::2/64 Scope:Global
```
Important in this example is the address with the Scope `Global`. Addresses that begin with `fe80` or `fd00`, or have the Scope `Link`
are local addresses that are not reachable from the internet. But we see that we have a globally routed `/64` net available in our example:
`2a01:4f8:a:b::/64`. Whenever you see this address/network I assume that you replace it with your specific network address when issuing
the commands shown in this guide.

You can check if the whole IPv6 really works with the following command:
```bash
root@dockerhost ~/ # ping6 -n -c4 google.com
PING google.com(2a00:1450:4001:811::200e) 56 data bytes
64 bytes from 2a00:1450:4001:811::200e: icmp_seq=1 ttl=56 time=5.19 ms
64 bytes from 2a00:1450:4001:811::200e: icmp_seq=2 ttl=56 time=5.19 ms
64 bytes from 2a00:1450:4001:811::200e: icmp_seq=3 ttl=56 time=5.19 ms
64 bytes from 2a00:1450:4001:811::200e: icmp_seq=4 ttl=56 time=5.20 ms

--- google.com ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3003ms
rtt min/avg/max/mdev = 5.194/5.196/5.201/0.088 ms
```

### A recent version of Docker

You need to have at least version 1.10 of Docker installed. For example a version of `docker-ce` that begins with 17 is fine.

### A basic understanding of IPv6

this guide assumes that you have a basic understanding of what IPv6 is, how an address or a network looks like and what the notation
of for example `/64` for the network length means and how a subnet would look like.

## Prepare Docker

We now prepare the Docker Engine for IPv6 with the following steps:

* Add route to interface `docker0`:
  * `sudo ip -6 route add 2a01:4f8:a:b::/64 dev docker0` 
  * This will route all IPv6 traffic that is addressed to the network `2a01:4f8:a:b::/64` to the docker interface `docker0`.
* Enable IPv6 packet forwarding in the Linux kernel:
  * `sudo sysctl -w net.ipv6.conf.default.forwarding=1`
  * `sudo sysctl -w net.ipv6.conf.all.forwarding=1`
* Configure the Docker daemon by editing `/etc/default/docker`:
  * `DOCKER_OPTS="--dns 8.8.8.8 --dns 8.8.4.4 --ipv6 --fixed-cidr-v6='2a01:4f8:a:b::/64'"`
* Restart the Docker daemon:
  * `sudo systemctl docker restart`
* Create a Docker network that has IPv6 enabled:
  * `docker network create --driver bridge --ipv6 --subnet=2a01:4f8:a:b:c::/80 ipv6`
  * With this we create a Docker network with the name `ipv6` for the sub net `2a01:4f8:a:b:c::/80`. Because we might want to create multiple
  networks that are used by Docker, we create a `/80` network with the suffix `c`. Of course this can be anything you choose from
  `0000` to `ffff`. Or you could even assign more bytes to the subnet but then you would need to adjust the length of `/80`.

If everything worked as expected, the following command should output something similar as shown here:

```bash
root@dockerhost ~/ # docker network inspect ipv6
[{
 "Name": "ipv6",
 "Id": "8777534b045364232f1b9706172ed0ac8c68ce5c60bb2ab538994858e4c1b2e4",
 "Created": "2017-05-12T09:15:59.367836503+02:00",
 "Scope": "local",
 "Driver": "bridge",
 "EnableIPv6": true,
 "IPAM": {
 "Driver": "default",
 "Options": {},
 "Config": [{
     "Subnet": "172.19.0.0/16",
     "Gateway": "172.19.0.1"
   },
   {
     "Subnet": "2a01:4f8:a:b:c::/80"
   }
 ]},
 "Internal": false,
 "Attachable": false,
 "Ingress": false,
 "Containers": {},
 "Options": {},
 "Labels": {}
}]
```

## Add a container to the network

Finally we can add a container to the Docker network we just created:

`docker network connect --ip6 '2a01:4f8:a:b:c::5' ipv6 [container_name]`

With this, the container `container_name` will be reachable with the address `2a01:4f8:a:b:c::5`. 
Naturally, this only works if the software inside the container is built to handle IPv6 and listens on both stacks. 
Sometimes there is a configuration option or a command line flag to enable IPv6.
If you have to configure a listen address, configure it to `[::]` which is the IPv6 equivalent of `0.0.0.0`, also known as
*listen on all addresses that are available*.
It's easiest to consult the software/container's manual to find out how you can enable dual stack mode.

If you don't want your container to be connected to the default network but only IPv6, you can use the command line parameter to do so:

```bash
docker run \
 -d \
 -p 1234:1234 \
......
 --network ipv6 \
 --ip6 '2a01:4f8:a:b:c::5' \
 --name [container_name] \
 [image_name]
```

To check if that worked, you can use the command `docker network inspect ipv6`.

If your container is really reachable through IPv6, you can use an online tool like [ipv6-test.com](http://ipv6-test.com/validate.php).

## Security considerations

If you are an experienced Docker user, you are used to the fact that usually you use a NAT network. So a container is not reachable from
the outside per default, you only forward some ports explicitly to it.

With IPv6 this is no longer the default! If you assign an IPv6 address to a container, it is directly exposed to the network through that
address! All ports that are open are reachable.

There are additional tools that implement the missing IPv6-NAT functionality, for example
[docker-ipv6nat](https://github.com/robbertkl/docker-ipv6nat).

But since NAT was never really meant as a security feature but was implemented to mitigate the scarcity of IPv4 addresses, the proper way
to secure your containers would be a firewall setup, for example with `iptables`.
