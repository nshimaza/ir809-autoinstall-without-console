# ir809-autoinstall-without-console
Provision Cisco IR809 without USB console

# Motivation

This repo demonstrates how you can provision IR809 without using USB console.
IR809 comes with USB mini-B console port with covering plate.  You have to
use screw driver to remove the cover to access the console port.  There is
no typical RJ45 Cisco console port on this product.  This is a little bit
annoying.

Fortunately, IR809 comes with a bunch of Cisco IOS features for automated
provisioning such like AutoInstall, WSMA, Plag and Play Agent.  With those
features you can get initial access to IOS CLI via telnet or SSH, so that
you can perform farther configurations.

# Usage

1. Obtain appropriate IOS bundle image (in this demo it is
   ir800-universalk9-bundle.SPA.157-3.M4b.bin)
1. Set up a TFTP server with address 192.168.0.253 (follow configuration in
   the next section)
1. Put IOS bundle image, `network-confg`, and `common.cfg` to `tftpboot`
   folder
1. Set up DHCP server (follow configuration in the next section)
1. Connect GE0 port of your brand new IR809, TFTP server, and the DHCP
   server all together via Ethernet
1. Power the IR809
1. Wait for 20 to 30 minutes
1. Verify you can telnet to the IR809's address 192.168.0.1 with username
   `root` and password `cisco`

## Server configurations

The TFTP server must have IP address 192.168.0.253/24.  The DHCP server
can be separeted from the TFTP server or co-hosted with the TFTP server.
In any case, don't use 192.168.0.1.  It will be used by the IR809 later.

Provide appropriate address pool for network 192.168.0/24 by the DHCP server
(e.g. from 192.168.0.100 to 192.168.0.200).  Provide the TFTP server address
to the pool via DHCP option 150.

**IMPORTANT**: You must provide TFTP server address via option 150.

This is sample configuration of `dhcpd.conf` for `isc-dhcpd`.

```
option ciscotftp code 150 = ip-address;

subnet 192.168.0.0 netmask 255.255.255.0 {
  range 192.168.0.100 192.168.0.200;
  option ciscotftp 192.168.0.253;
}
```

## Verify

Once the AutoInstall process succeed, you can telnet to the IR809's address
192.168.0.1 and login to IOS with username:root / password:cisco.

# How does it works?

When IR809 bootstraps, it checks if `startup-config` presents.  If there is
no `startup-config` in `nvram:`, IR809 kicks AutoInstall feature.  It tries
to obtain IP address on GigabitEthernet0 interface and tries to download
its configuration from obtained TFTP server via DHCP.  In this demo, it
requests configuration from `tftp://192.168.0.253/network-confg`.

In simplest use case, you can directly provide your final configuration with
the file `network-confg` but in this demo we will perform IOS bundle
installation with the first configuration.

In `network-confg` we have EEM (Embedded Event Manager) applet.  It monitors
PnP discovery process aborts due to absence of PnP server.  Once PnP
discovery aborted, the EEM applet starts execution.

An EEM applet is basically a sequence of command line input but also supports
some variable manipulations and flow control with logics.  In this demo the
applet check if specified IOS bundle image is already on the `flash:` file
system on the IR809 and download from the TFTP server when it doesn't exist.
After downloading completed, it performs disk re-partitioning for IOx guest
OS and `bundle install` against the bundle image.  With this process, all
IOS and IOx are refreshed.

Finally, the applet downloads startup-config from
`tftp://192.168.0.253/common.cfg`, save it to NVRAM, then reload the router.

After router rebooted, you will get IR809 configured with following
configurations.

- A user on IOS with username root, privilege 15, and password cisco.
  You can use the user to login to IOS via telnet and login to IOx Local
  Manager via browser.
- interface GigabitEthernet0:  192.168.0.1/24
- interface GigabitEthernet1:  192.168.1.1/24
- interface GigabitEthernet2:  192.168.2.1/30 (for IOx access)
- line configurations for telnet access
- Some necessary stuffs for IOx (IPv6, DHCP pool, NAT, virtual console etc)

# Caveat

The `network-confg` in this demo doesn't work with factory reset procedure.
You have to perform factory reset against IR809 alone (without connecting
the TFTP server), then you have to connect the IR809 and reload or power
cycle it.

# Next step

For simplicity, this demonstration configure any IR809 to exact same
configuration.  You can differentiate configuration per IR809 unit basis
using serial number.
