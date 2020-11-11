+++
title = "Creating a VPN in Go - Part 1"
description = "Creating our own encrypted VPN in Go"
date = 2020-04-03
slug = "vpn_part_1"

[taxonomies]
tags = ["go", "golang", "vpn"]
+++

Recently I have been looking at the [Wireguard](https://www.wireguard.com/) protocol for my personal networking setup and also because I was quite fascinated by the simplicity of it. I have always wondered how VPNs generally work. So, I wanted to create a encrypted VPN in Go just to learn how VPNs generally work. I will document this in multiple blog posts as documentation and to show how this is not some magic.

> Note: This VPN is just created to learn and not to use in Production.

We will create a point to point to tunnel, where the traffic will go over UDP. We can do this with TCP too, but TCP over TCP is generally a bad idea, since any kind of packet loss is amplified since the underlying transport is also TCP. UDP is just a thin wrapper over IP packets and quite lightweight. This VPN will only work on Linux since that's what I am most familiar with.

For creating a VPN, we will need to create a TUN/TAP interface which is a virtual network kernel interface. TUN interface is better in our case since this will strip out the ethernet frames in the packet and only gives us raw IP packets to deal with.

We will use [https://github.com/songgao/water](https://github.com/songgao/water) to create a TUN interface. `Water` gives a simple `io.ReadWriteCloser` to work with. This is how the code looks like for creating a TUN interface.

Let's call our VPN `sartunnel`.

```go
// main.go
package main

import (
	"log"
	"time"

	"github.com/songgao/water"
)

const (
	// Name will be the name of the tunnel
	Name = "tun0"
)

func main() {
	config := water.Config{
		DeviceType: water.TUN,
	}
	config.Name = Name

	// Create a tunnel
	inf, err := water.New(config)
	if err != nil {
		log.Fatalf("error while creating a tun interface: %v", err)
    }
    defer inf.Close()

	log.Printf("tunnel created with name: %s", inf.Name())

	time.Sleep(10 * time.Minute)
}
```
We are using a `DeviceType` of `water.TUN` to create a TUN interface called `tun0`. We are using a `time.Sleep` so that the program doesn't exit and stays up. A better way to exit is to catch the termination signal. For now `time.Sleep` would suffice.

We need `sudo` to run this.

```bash
$ go build && sudo ./sartunnel
2020/04/03 21:34:46 tunnel created with name: tun0
```

In another terminal if you `ip addr | grep tun0` you will see a tunnel created.

```bash
$ ip a | grep tun0
339: tun0: <POINTOPOINT,MULTICAST,NOARP> mtu 1500 qdisc noop state DOWN group default qlen 500
```

Yay! A TUN interface is created, but you can see that it's down and there is no IP address assigned to it. We have to use `ip` command to assign an IP address, set MTU for the interface and bring the interface up.

```bash
$ sudo ip addr add 192.168.9.10/24 dev tun0
$ sudo ip link set dev tun0 mtu 1300
$ sudo ip link set dev tun0 up
$ ip a | grep tun0
355: tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN group default qlen 500
    inet 192.168.9.10/24 scope global tun0
```

Now we can see that there is an IP address assigned to it and there is an MTU set. Instead of `time.Sleep`, let's catch the termination signal.

```go
package main

import (
	"log"
	"os"
	"os/signal"
	"syscall"

	"github.com/songgao/water"
)

const (
	// Name will be the name of the tunnel
	Name = "tun0"
)

func main() {
	sigs := make(chan os.Signal, 1)
	done := make(chan bool, 1)

	signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)

	go func() {
		<-sigs
		done <- true
	}()

	config := water.Config{
		DeviceType: water.TUN,
	}
	config.Name = Name

	// Create a tunnel
	inf, err := water.New(config)
	if err != nil {
		log.Fatalf("error while creating a tun interface: %v", err)
    }
    defer inf.Close()

	log.Printf("tunnel created with name: %s", inf.Name())
	<-done
	log.Println("Exiting....")
}
```

Now, the program won't exit until it receives a termination signal.

We have an interface but we are not actually doing anything with it. Let's read the packets from there.

```go
// Read packets from the interface
r := bufio.NewReader(inf)
packet := make([]byte, 1500)
for {
    n, err := r.Read(packet)
    if err != nil {
        log.Printf("error reading from the interface: %v", err)
        break
    }

    log.Printf("got packet: %v", packet[:n])
}
```

We are allocating a packet of 1500 buffer size. We are reading from the interface and printing `n` number of bytes we read.

Since everytime we restart, we have to use `ip` to set everything, let's do it from the program itself. We can use `netlink` api to programmatically set everything. We will use [https://github.com/vishvananda/netlink](https://github.com/vishvananda/netlink) library.

```go
// Create a tunnel
inf, err := water.New(config)
if err != nil {
    log.Fatalf("error while creating a tun interface: %v", err)
}
defer inf.Close()

link, err := netlink.LinkByName(Name)
if err != nil {
    log.Fatalf("error while getting link: %v", err)
}
addr, _ := netlink.ParseAddr(IPAddr)
if err != nil {
    log.Fatalf("error parsing ip address: %v", err)
}

err = netlink.LinkSetMTU(link, 1300)
if err != nil {
    log.Fatalf("error setting MTU: %v", err)
}

err = netlink.AddrAdd(link, addr)
if err != nil {
    log.Fatalf("error assigning ip address: %v", err)
}

err = netlink.LinkSetUp(link)
if err != nil {
    log.Fatalf("error bringing up the link: %v", err)
}

log.Printf("tunnel created with name: %s", inf.Name())

// Read packets from the interface
r := bufio.NewReader(inf)
```

Instead of printing bytes, let's parse the packets read from the interface.

```go
for {
    n, err := r.Read(packet)
    if err != nil {
        log.Printf("error reading from the interface: %v", err)
        break
    }

    hdr, err := ipv4.ParseHeader(packet[:n])
    if err != nil {
        log.Printf("error while parsing ip header: %v", err)
        continue
    }

    log.Printf("got packet: %+v", hdr)
}
```


Instead of using `bufio` let's use [github.com/dolmen-go/contextio](github.com/dolmen-go/contextio) which provides a reader which can be cancelled when the program terminates. This is how our program looks like after adding context cancellation.

```go
package main

import (
	"context"
	"log"
	"os"
	"os/signal"
	"syscall"

	"github.com/dolmen-go/contextio"
	"github.com/songgao/water"
	"github.com/vishvananda/netlink"
	"golang.org/x/net/ipv4"
)

const (
	// Name will be the name of the tunnel
	Name = "tun0"

	// IPAddr will be the IP address added to the interface
	IPAddr = "192.168.9.10/24"
)

func main() {
	sigs := make(chan os.Signal, 1)
	done := make(chan bool, 1)

	signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)

	ctx, cancel := context.WithCancel(context.Background())

	go func() {
		<-sigs
		cancel()
		done <- true
	}()

	config := water.Config{
        DeviceType: water.TUN,
	}
    config.Name = Name

	// Create a tunnel
	inf, err := water.New(config)
	if err != nil {
		log.Fatalf("error while creating a tun interface: %v", err)
	}
	defer inf.Close()

	link, err := netlink.LinkByName(Name)
	if err != nil {
		log.Fatalf("error while getting link: %v", err)
	}
	addr, _ := netlink.ParseAddr(IPAddr)
	if err != nil {
		log.Fatalf("error parsing ip address: %v", err)
	}

	err = netlink.LinkSetMTU(link, 1300)
	if err != nil {
		log.Fatalf("error setting MTU: %v", err)
	}

	err = netlink.AddrAdd(link, addr)
	if err != nil {
		log.Fatalf("error assigning ip address: %v", err)
	}

	err = netlink.LinkSetUp(link)
	if err != nil {
		log.Fatalf("error bringing up the link: %v", err)
	}

	log.Printf("tunnel created with name: %s", inf.Name())

	// Read packets from the interface
	r := contextio.NewReader(ctx, inf)
	packet := make([]byte, 1500)
	for {
		n, err := r.Read(packet)
		if err != nil {
			log.Printf("error reading from the interface: %v", err)
			break
		}

		hdr, err := ipv4.ParseHeader(packet[:n])
		if err != nil {
			log.Printf("error while parsing ip header: %v", err)
			continue
		}

		log.Printf("got packet: %+v", hdr)
	}

	<-done
	log.Println("Exiting....")
}

```

Let's run this and ping the `192.168.9.11`. The pings will be routed through the interface, so the kernel will send the packets to the tunnel.
If you ping `192.168.9.10`, the kernel will reply automatically to pings and the packets won't be sent to our program.


```bash
$ ping 192.168.9.11
```

---

```bash
$ go build && sudo ./sartunnel
2020/04/04 01:03:24 got packet: ver=4 hdrlen=20 tos=0x0 totallen=84 id=0x3761 flags=0x2 fragoff=0x0 ttl=64 proto=1 cksum=0x6fe2 src=192.168.9.10 dst=192.168.9.11
2020/04/04 01:03:25 got packet: ver=6 hdrlen=0 tos=0x0 totallen=0 id=0x8 flags=0x1 fragoff=0x1aff ttl=254 proto=128 cksum=0x0 src=0.0.0.0 dst=52.84.50.179
2020/04/04 01:03:25 got packet: ver=4 hdrlen=20 tos=0x0 totallen=84 id=0x378c flags=0x2 fragoff=0x0 ttl=64 proto=1 cksum=0x6fb7 src=192.168.9.10 dst=192.168.9.11
2020/04/04 01:03:26 got packet: ver=4 hdrlen=20 tos=0x0 totallen=84 id=0x37b4 flags=0x2 fragoff=0x0 ttl=64 proto=1 cksum=0x6f8f src=192.168.9.10 dst=192.168.9.11
2020/04/04 01:03:27 got packet: ver=4 hdrlen=20 tos=0x0 totallen=84 id=0x37d1 flags=0x2 fragoff=0x0 ttl=64 proto=1 cksum=0x6f72 src=192.168.9.10 dst=192.168.9.11
2020/04/04 01:03:27 got packet: ver=4 hdrlen=20 tos=0x0 totallen=154 id=0x2f8b flags=0x2 fragoff=0x0 ttl=1 proto=17 cksum=0x901b src=192.168.9.10 dst=239.255.255.250
2020/04/04 01:03:27 got packet: ver=4 hdrlen=24 tos=0xc0 totallen=40 id=0x0 flags=0x2 fragoff=0x0 ttl=1 proto=2 cksum=0x3a47 src=192.168.9.10 dst=224.0.0.22
```

We can see the ping packets now.

So we have created a TUN interface from Go and started parsing IP packets received through it. We will connect to a remote address in the next part and start sending the packets we received to the remote server.

The code until now can be accessed at `part-1` tag at [https://github.com/iamd3vil/sartunnel](https://github.com/iamd3vil/sartunnel/tree/part-1). I will update this post, once part 2 is published.
