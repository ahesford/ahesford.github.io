# A Home Router Built on Void Linux and ZFSBootMenu

Back around 1998, I built a home router from a Dell Dimension with a 100-MHz
Pentium, some tens of megabytes of RAM, no hard disk and PicoBSD on a
high-density, 3.5-inch floppy. Our cable company offered us a cable modem that
could pull 256 kbps down and used a 56k modem for uploads. We started on thin
coax and eventually upgraded to twisted pair and an 8-port hub. Our demands
grew, and eventually it became more convenient to use pre-built home routers
with integrated wireless access points.

After two decades, I bought several eero Pro units to distribute around my home
for relatively painless "mesh" Wi-Fi. These are "Pro" in name only. The feature
that distinguishes these devices from regular eero units is not improved
control and configurability, but a third wireless band. With Ethernet backhaul,
this setup works well enough, provided you don't care too much about controlling
the devices through a mobile application *via Amazon, over the Internet*.
Losing control of your network when your Internet connection is down and having
too few knobs to tweak when you want to do anything that isn't bog standard
gets old pretty fast. I've thought several times about using my old Asus
RT-AC66U as the router and using the eero devices as dumb access points, but
figured I might as well go all in on custom.

## Hardware Requirements

Wireless networking isn't as important to me as it used to be. A few years
back, I ran twisted pair to strategic places in my house, and most devices with
Ethernet adapters are already wired directly into the network. For handheld
devices and entertainment units with no Ethernet, I want good throughput and
relatively seamless roaming. The existing eero units do this job just fine, and
they can be put in "bridge" mode to disable routing and firewall capabilities.

CPU, memory and storage requirements are pretty modest. I've got a 400/20 Mbps
cable link that I want to be able to saturate. If the local fiber company ever
gets around to building out my neighborhood, I'd like to expand that to a
symmetric 1 Gbps. To go beyond that, I'd have to rewire the house and upgrade
all other equipment anyway.

Some weeks ago, I happened upon
[an article](https://battlepenguin.com/tech/installing-void-linux-with-a-serial-terminal/)
about a Lanner FW-7541C running Void Linux (via a post on
[r/voidlinux](https://reddit.com/r/voidlinux)
that I can no longer find). The two-core, four-thread Atom D525 CPU, 4 GB DDR3
RAM and 30-GB SSD seem to provide all the power I need for a home router. The
device provides six independent Ethernet interfaces. I've already configured
switching for my network around the two-port eero box, so having more than two
interfaces is nice but unnecessary. As a bonus, the author of the article has
already confirmed that our standard Void kernel recognizes all of the hardware.
I grabbed a unit off eBay for the same price quoted in the original article.

## Basic Installation

The article linked above provides details for configuring a Void live image to
use a serial console. This works fine for me, except I avoid deploying Linux or
UNIX systems without ZFS, so I needed to include the Void `zfs` package in the
initramfs. From a clone of the
[`void-mklive`](https://github.com/void-linux/void-mklive)
repo,

    sudo ./mklive.sh -C "console=ttyS0,115200n8" -p zfs

gives me an ISO image I can write to a USB drive and boot on the Lanner. From
there, I abandoned the original article about running Void on the Lanner and
moved to the
[ZFSBootMenu Wiki](https://github.com/zbm-dev/zfsbootmenu/wiki/Void-Linux----Single-disk-syslinux-MBR)
for a HOWTO on setting up ZFSBootMenu with syslinux on BIOS systems. I made a
few  trivial deviations from the HOWTO:

- Instead of [hrmpf](https://github.com/leahneukirchen/hrmpf), which is a Void
  live image with a bunch of extra utilities, I used my custom live image with
  `zfs` and a serial console.

- Rather than `base-system`, I like to install `base-voidstrap`, which does not
  pull in the `linux` metapackage. For a recent LTS kernel, I manually
  installed `linux5.10` and `linux5.10-headers`; installing the `linux-base`
  metapackage pulls in some firmware (not all of which is needed; ignoring this
  is an exercise for the reader).

- The ZFSBootMenu HOWTO uses `Components.ImageDir: /boot/syslinux/zfsbootmenu`
  as the location where ZFSBootMenu kernel and initramfs pairs will be
  installed, but I shortened this to `/boot/sylinux/zbm`.

- I prefer not to version my ZFSBootMenu images, so I set `Components.Versions:
  false`. This causes ZFSBootMenu to create images named `vmlinuz-bootmenu` and
  `initramfs-bootmenu.img`, with a single backup of the prior versions as
  `vmlinuz-bootmenu-backup` and `initramfs-bootmenu-backup.img`, respectively.

- After letting ZFSBootMenu generate an initial `/boot/syslinux/syslinux.cfg`,
  I disabled syslinux configuration with `Components.syslinux.Enabled: false`
  in the ZFSBootMenu configuration file. Because my ZBM images will always have
  the same name, I don't need to rewrite the syslinux configuration each time
  and, because I need to make syslinux work over serial console, the
  configuration needs some customization beyond what ZFSBootMenu will write.

At the conclusion of the HOWTO, I finalized my `/boot/syslinux/syslinux.cfg`
file to contain

    SERIAL 0 115200 0x8
    UI menu.c32
    PROMPT 0
    
    MENU TITLE Choose a ZFSBootMenu image to boot
    TIMEOUT 50
    TOTALTIMEOUT 6000
    DEFAULT zbmskip
    
    LABEL zbmskip
    MENU LABEL ZFSBootMenu (Default)
    LINUX /zbm/vmlinuz-bootmenu
    INITRD /zbm/initramfs-bootmenu.img
    APPEND zfsbootmenu ro loglevel=4 zbm.skip zbm.lines=30 zbm.columns=90 console=ttyS0,115200n8
    
    LABEL zbmshow
    MENU LABEL ZFSBootMenu (Menu)
    LINUX /zbm/vmlinuz-bootmenu
    INITRD /zbm/initramfs-bootmenu.img
    APPEND zfsbootmenu ro loglevel=4 zbm.show zbm.lines=30 zbm.columns=90 console=ttyS0,115200n8
    
    LABEL zbmbackup
    MENU LABEL ZFSBootMenu (Fallback)
    KERNEL /zbm/vmlinuz-bootmenu-backup
    INITRD /zbm/initramfs-bootmenu-backup.img
    APPEND zfsbootmenu ro loglevel=4 zbm.timeout=300 zbm.lines=30 zbm.columns=90 console=ttyS0,115200n8

This left me with a functioning Void installation on the Lanner.

## Initial Router Configuration

To manage routing, it is helpful to treat the network interface that will be
connected to your ISP (in my case, via cable modem) separately from all other
interfaces. It is also useful to give static names to all interfaces and do
away with the horrible "predictable naming" that is enabled by default. The
"predictable naming" can be disabled with an argument on the kernel command
line, but a more permanent solution is to mask the naming rule with

    mkdir -p /etc/udev/rules.d
    ln -s /dev/null /etc/udev/rules.d/80-net-name-slot.rules

Next, assign convenient and obvious names to each of the six interfaces based
on MAC addresses. Create the file `/etc/udev/rules.d/70-net-macs.rules` with contents

    SUBSYSTEM=="net", ACTION=="add", KERNEL=="eth*", ATTR{address}=="aa:bb:cc:dd:ee:f0", NAME="wan0"
    SUBSYSTEM=="net", ACTION=="add", KERNEL=="eth*", ATTR{address}=="aa:bb:cc:dd:ee:f1", NAME="lan0.0"
    SUBSYSTEM=="net", ACTION=="add", KERNEL=="eth*", ATTR{address}=="aa:bb:cc:dd:ee:f2", NAME="lan0.1"
    SUBSYSTEM=="net", ACTION=="add", KERNEL=="eth*", ATTR{address}=="aa:bb:cc:dd:ee:f3", NAME="lan0.2"
    SUBSYSTEM=="net", ACTION=="add", KERNEL=="eth*", ATTR{address}=="aa:bb:cc:dd:ee:f4", NAME="lan0.3"
    SUBSYSTEM=="net", ACTION=="add", KERNEL=="eth*", ATTR{address}=="aa:bb:cc:dd:ee:f5", NAME="lan0.4"

Make sure to replace the placeholder addresses in `ATTR{address}` with the
actual MAC address for each interface. You'll need to create a host-only
initramfs image to make sure these rules are applied when your system
configures devices:

    cat >> /etc/dracut.conf.d/hostonly.conf  <<EOF
    hostonly=yes
    hostonly_cmdline=no
    EOF

Finally, `xbps-reconfigure -f linuxX.Y` for whatever versioned `linuxX.Y`
packages you have installed. (Even if you rely on the unversioned `linux`
metapackage, you will need to use the name of the versioned package that is
pulled in as a dependency.)

Unless you are doing sophisticated stuff on your internal network, in which
case you probably don't this guide anyway, it is convenient to lump all
internal interfaces into a single bridge by adding the following lines to
`/etc/rc.local`:

    # Create a bridge for all but wlan0
    ip link add name lan0 type bridge
    
    # Bring up and attach devices to the bridge
    for _dev in 0 1 2 3 4; do
            ip link set dev lan0.${_dev} up
            ip link set dev lan0.${_dev} master lan0
    done
    unset _dev
    
    # Bring up the bridge
    ip link set dev lan0 up
    
    # Create a static IP address on the lan0 interface
    ip addr add 192.168.1.1/24 brd + dev lan0

Finally, set some sysctls to allow IP forwarding:

    cat > /etc/sysctl.d/ip_forwarding.conf <<EOF
    net.ipv4.ip_forward = 1
    net.ipv6.conf.all.forwarding = 1
    EOF

On internal interfaces, I also like to enable "stable privacy" addresses and
use router advertisements for SLAAC configuration:

    cat > /etc/sysctl.d/ipv6_addressing.conf <<EOF
    net.ipv6.conf.all.addr_gen_mode = 3
    net.ipv6.conf.default.addr_gen_mode = 3
    net.ipv6.conf.all.accept_ra = 1
    net.ipv6.conf.default.accept_ra = 1
    EOF

For IPv6 subnet delegation from the ISP to the internal network, we will rely
on `dhcpcd`, which was already pulled in by `base-voidstrap`. Add some
additional directives to its configuration and enable the service:

    cat >> /etc/dhcpcd.conf <<EOF
    # Ignore resolv.conf updates
    nohook resolv.conf
    
    # Only listen on WAN interface
    allowinterfaces wan0
    
    noipv6rs
    interface wan0
      ipv6rs
      ia_na 1
      ia_pd 2 lan0/0
    EOF

    ln -s /etc/sv/dhcpcd /var/service

Except for the `nohook resolv.conf` directive, these directives come right from
the [`dhcpcd.conf(5)`](https://man.voidlinux.org/dhcpcd.conf.5)
manual page, but I've inverted the suggested `denyinterfaces eth2` blacklist in
favor of the more restrictive `allowinterfaces wan0` whitelist. The
`resolv.conf` hook is disabled because we will rely on `dnsmasq` for DNS rather
than whatever lousy servers the ISP suggests.


At this point, plugging your ISP device into the `wan0` port (after taking
steps to clear any MAC binding it may have) should result in `dhcpcd` grabbing
a public IPv4 address and, if supported, IPv6 address. The `ia_pd` directive in
your `dhcpcd.conf` will also result in your `lan0` interface receiving a public
IPv6 address of the form `<prefix>::1/<mask>` for some arbitrary prefix and
mask length. (In my case, the mask length is 64 bits.) This is *not* sufficient
to provide any connectivity to other devices on your network.

## Firewalls and Masquerading

Before going further, it is probably desirable to configure a firewall. I
prefer `nftables` for this task, so start by installing necessary components:

    xbps-install -S nftables runit-nftables

The `runit-nftables` package installs a runit core service that will configure
your firewall before running any regular services, so there is no need to
enable the fake `nftables` service provided in the `nftables` package.

There are a number of guides for configuring firewalls and IPv4 masquerading
with nftables. Here, I'll just dump a basic firewall configuration:

    # Clear all prior state
    flush ruleset
    
    table inet firewall {
      # Packets bound for the local host
      chain input {
        type filter hook input priority 0
        policy drop
    
        # allow established/related connections
        ct state {established, related} accept
    
        # early drop of invalid connections
        ct state invalid drop
    
        # allow from loopback
        iif lo accept
    
        # allow from local bridge
        iif lan0 accept
    
        # allow icmp, igmp
        ip protocol icmp accept
        ip6 nexthdr icmpv6 accept
        ip protocol igmp accept
    
        # Allow dhcpv6
        udp dport dhcpv6-client accept
    
        # allow ssh
        tcp dport ssh accept
    
        # allow ssh to local host
        tcp dport 222 accept
    
        # everything else
        reject with icmpx type port-unreachable
      }
    
      # Packets not bound for the local host (e.g., virtualization guests)
      chain forward {
        type filter hook forward priority 0
        policy drop
    
        # Drop invalid packets
        ct state invalid drop
    
        # Allow all outgoing on wan interface
        oif wan0 accept
    
        # Allow incoming on wan interface for related and established
        iif wan0 ct state related, established accept
    
        # allow inbound ssh
        iif wan0 tcp dport ssh accept
      }
    
      # Outbound packets
      chain output {
        type filter hook output priority 0
      }
    }
    
    table ip nat {
      chain prerouting {
        type nat hook prerouting priority -100
    
        # Forward traffic from wan interface to LAN
        iif wan0 tcp dport ssh dnat 192.168.1.2 comment "default ssh port forwarding"
        iif wan0 tcp dport 222 redirect to ssh comment "local ssh port redirection"
      }
    
      chain postrouting {
        type nat hook postrouting priority 100
    
        # Masquerade outgoing traffic
        oif wan0 masquerade
      }
    }

These rules allow outbound IPv4 connections originating from the internal
network to be masqueraded as the router. Furthermore, inbound SSH connections
are forwarded to a default host (here, 192.168.1.2) while connections on port
222 are directed to the SSH server running on the router itself.

## DHCP and DNS for the Internal Network

The router knows how to get an IPv4 address and, when applicable, IPv6 address
on `wan0` using `dhcpcd`. It will also request prefix delegation and assign a
`::1` IPv6 address to `lan0` using the prefix delegated by the ISP. The
firewall has been configured to restrict most inbound traffic, pass all
outbound traffic and, for IPv4, masquerade outbound packets that originate from
the internal network. If you only use IPv4 and all devices on your internal
network will be statically configured, nothing more is required. However, for
dynamic configuration, the router needs to run a DHCP server and also issue
router advertisements for a delegated IPv6 prefix. There are several programs
that can provide these services. I like `dnsmasq` because it will do both and,
as a bonus, provide a DNS cache for the hosts on my network. Install with

    xbps-install -S dnsmasq

Configuration is straightforward, with directives in `/etc/dnsmasq.conf` that
map cleanly to command-line options for the server. My basic configuration is

    # Ignore /etc/resolv.conf and statically configure servers
    no-resolv
    
    # Upstream servers: CloudFlare
    server=1.1.1.1
    server=2606:4700:4700::1111
    server=1.0.0.1
    server=2606:4700:4700::1001
    
    # Upstream servers: Google
    server=8.8.8.8
    server=2001:4860:4860::8888
    server=8.8.4.4
    server=2001:4860:4860::8844
    
    # Disable bogus private reverse lookups
    bogus-priv
    
    # Only listen for DHCP and DNS on lan0
    interface=lan0
    
    # IPv4 private DHCP range
    dhcp-range=192.168.1.100,192.168.1.250,12h
    
    # Do router advertisements for all subnets where we're doing DHCPv6
    enable-ra
    
    # For SLAAC and stateless DHCPv6, pull IPv6 prefix
    # from the ::1 address assigned to lan0 by dhcpcd
    dhcp-range=::1,constructor:lan0,slaac,ra-stateless,24h
    
    # Become the authoritative DHCP server on this network
    dhcp-authoritative
    
    # Reserve 192.168.1.2 for MAC aa:bb:cc:dd:ee:ff
    # Give it the hostname "ssh"
    dhcp-host=aa:bb:cc:dd:ee:ff,192.168.1.2,ssh
    
    # If a DHCP client claims that its name is "wpad", ignore that.
    # This fixes a security hole. see CERT Vulnerability VU#598349
    dhcp-name-match=set:wpad-ignore,wpad
    dhcp-ignore-names=tag:wpad-ignore

Finally, configure the `dnsmasq` service to start with

    ln -s /etc/sv/dnsmasq /var/service

At this point, you can plug your ISP device into the `wan0` port and plug one
or more internal hosts (or switches) into any of the `lan0` ports that have
been bridged. The internal devices should receive a private IPv4 address from
the router and be able to ping the public Internet. If your ISP gives you an
IPv6 address and delegates a prefix, your internal devices should configure
themselves with public IPv6 addresses.

## Niceties

Because the router is a generic Void system, it can be extended with whatever
features you prefer. I like to use `avahi` for mDNS resolution, so I install
that and (for glibc) the mDNS resolver for NSS:

    xbps-install -S avahi nss-mdns

Firewall notwithstanding, `avahi` should completely ignore `wan0`. Edit the
configuration file `/etc/avahi/avahi-daemon.conf` and add (or uncomment) the
line

    allow-interfaces=lan0

under the `[server]` section. The configure the server to start with

    ln -s /etc/sv/avahi-daemon /var/service

If you care about logging, which you should, I recommend `socklog`:

    xbps-install -S socklog-void
    ln -s /etc/sv/{nanoklogd,socklog-unix} /var/service

I also enable `sshd` so that I can access my router remotely (using the port
redirect rule in the firewall), so run

    ln -s /etc/sv/sshd /var/service

I also require key-based authentication for SSH, but won't describe the process
here. With an SSH server running, I also like to block repeated login attempts.
(They will always fail when password authentication is prohibited, but it makes
for noisy logs.) The `sshguard` package can dynamically modify firewall rules
to block repeat attackers. With `nftables`, the process looks like

    xbps-intall -S sshguard
    sed -i '/BACKEND=/a BACKEND="/usr/libexec/sshg-fw-nft-sets"' /etc/sshguard.conf
    ln -s /etc/sv/sshguard-socklog /var/service

The service assumes you are using `socklog`. If everything is working the command

    nft list tables

should show, among other things, `table ip sshguard` and `table ip6 sshguard`.
As apparent attacks are logged by the system, `sshguard` will dynamically add
hosts to sets within these tables.

## Conclusion

Any Linux system that you're comfortable administering can make a powerful home
router. For me, Void is a great platform; it provides a good selection of
kernels, is generally up to date (especially the packages I care about) and has
top-notch ZFS support. Using ZFSBootMenu to boot the Void environment has
additional benefit, providing me with a perfect recovery environment should
a bad update or other catastrophe prevent the device from booting normally. I
could also, but have not yet, incorporate an SSH server into the ZFSBootMenu
image for remote recovery operations using
[dracut-crypt-ssh](https://github.com/dracut-crypt-ssh/dracut-crypt-ssh).
