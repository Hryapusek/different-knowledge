# **Linux — ifupdown cheat sheet**

Main configuration file:

```
/etc/network/interfaces
```

Additional configuration files:

```
/etc/network/interfaces.d/
```

## **1. Basic configuration — DHCP**

Automatically obtain an IP address, gateway and DNS information from a DHCP server.

```
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp
```

## **2. Static IP address**

```
auto eth0
iface eth0 inet static
    address 192.168.1.10
    netmask 255.255.255.0
    gateway 192.168.1.1
```

Alternatively, using CIDR notation:

```
auto eth0
iface eth0 inet static
    address 192.168.1.10/24
    gateway 192.168.1.1
```

The `gateway` option creates a default route.

## **3. Static IP without a gateway**

Useful for communicating with hardware on an isolated network.

```
auto eth0
iface eth0 inet static
    address 192.168.10.100/24
```

The kernel automatically creates the connected route:

```
192.168.10.0/24 dev eth0
```

No gateway is necessary for communicating within this subnet.

## **4. Multiple network interfaces**

One interface provides internet access, while another communicates with local equipment.

```
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address 192.168.1.10/24
    gateway 192.168.1.1

auto eth1
iface eth1 inet static
    address 192.168.10.100/24
```

Resulting routing table:

```
default via 192.168.1.1 dev eth0
192.168.1.0/24 dev eth0
192.168.10.0/24 dev eth1
```

Generally, configure only one default gateway unless you deliberately need multiple default routes or policy-based routing.

## **5. Multiple IP addresses on one interface**

Useful when equipment uses different subnets but is connected to the same physical switch.

```
auto eth0
iface eth0 inet static
    address 192.168.1.10/24
    gateway 192.168.1.1

    up ip addr add 192.168.2.10/24 dev eth0
    down ip addr del 192.168.2.10/24 dev eth0
```

The interface will have two IPv4 addresses and two connected subnet routes.

## **6. Static routes to remote subnets**

Suppose another router, `192.168.1.50`, provides access to the `10.0.0.0/24` network.

```
auto eth0
iface eth0 inet static
    address 192.168.1.10/24
    gateway 192.168.1.1

    post-up ip route replace 10.0.0.0/24 via 192.168.1.50 dev eth0
    pre-down ip route del 10.0.0.0/24 via 192.168.1.50 dev eth0 || true
```

Resulting routing table:

```
default via 192.168.1.1 dev eth0
192.168.1.0/24 dev eth0
10.0.0.0/24 via 192.168.1.50 dev eth0
```

This means traffic destined for `10.0.0.0/24` will go through a different gateway than ordinary internet traffic.

## **7. Static IP with DNS**

```
auto eth0
iface eth0 inet static
    address 192.168.1.10/24
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8 1.1.1.1
```

Note: `dns-nameservers` requires compatible DNS integration, such as `resolvconf`.

## **8. Manual configuration**

Bring up an interface without automatically assigning it an IP address.

```
auto eth0
iface eth0 inet manual
```

Useful for interfaces that are configured by custom scripts or used as bridge members.

## **9. Splitting configurations into files**

`/etc/network/interfaces`:

```
auto lo
iface lo inet loopback

source /etc/network/interfaces.d/*
```

`/etc/network/interfaces.d/eth0`:

```
auto eth0
iface eth0 inet dhcp
```

`/etc/network/interfaces.d/eth1`:

```
auto eth1
iface eth1 inet static
    address 192.168.10.100/24
```

## **Configuration keywords**

| Keyword             | Meaning                                |
| ------------------- | -------------------------------------- |
| `auto`              | Bring up interface at boot             |
| `allow-hotplug`     | Bring up when device is detected       |
| `iface`             | Define interface configuration         |
| `inet`              | IPv4 address family                    |
| `inet6`             | IPv6 address family                    |
| `static`            | Static configuration                   |
| `dhcp`              | Automatic configuration via DHCP       |
| `manual`            | No automatic IP configuration          |
| `pre-up`            | Execute command before configuration   |
| `up` / `post-up`    | Execute command after configuration    |
| `down` / `pre-down` | Execute command before deconfiguration |
| `post-down`         | Execute command after deconfiguration  |
| `source`            | Include another configuration file     |

**Important:** `allow-hotplug` refers to detection of the network device, not necessarily plugging in an Ethernet cable.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://manpages.debian.org\&sz=32)

Debian Manpages+1



## **Management commands**

```
# Bring interface up
sudo ifup eth0

# Bring interface down
sudo ifdown eth0

# Reconfigure interface
sudo ifdown eth0 && sudo ifup eth0

# Bring up all interfaces marked auto
sudo ifup -a

# Restart networking service
sudo systemctl restart networking

# Show parsed interface configuration
ifquery eth0

# List configured automatic interfaces
ifquery --list

# Dry run with verbose output
sudo ifup --no-act --verbose eth0

# Check networking service logs
journalctl -u networking -b
```

Be careful with `ifdown` and restarting networking when connected via SSH. You might disconnect yourself.

## **Debugging commands**

```
ip -br link                  # Interface states
ip -br addr                  # Interface addresses
ip -s link                   # Interface statistics
ip route                     # Routing table
ip route get 192.168.10.20    # Route selection
ip neigh                     # ARP / neighbor table
ip monitor all               # Monitor kernel changes
ss -tulnp                    # TCP/UDP sockets
sudo tcpdump -ni eth0         # Capture traffic
cat /etc/resolv.conf          # DNS configuration
```

## **Documentation**

The complete documentation is already available on your machine if the corresponding man pages are installed:

```
man interfaces
man ifup
man ifdown
man ifquery
man ip-route
```

Online: [Debian interfaces(5)](https://manpages.debian.org/trixie/ifupdown/interfaces.5.en.html?utm_source=chatgpt.com) and [Debian ifup(8)](https://manpages.debian.org/trixie/ifupdown/ifup.8.en.html?utm_source=chatgpt.com).

The most important thing to remember about `ifupdown` is that it isn't itself responsible for routing packets. It simply configures the kernel and executes commands at different stages of the interface lifecycle.

And those `up`, `post-up`, `pre-down` directives are particularly powerful because you can execute arbitrary shell commands there.