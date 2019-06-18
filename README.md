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

You can also access to IOx Local Manager via https://192.168.0.1:8443/ with
same login credential above.

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
applet checks if specified IOS bundle image is already on the `flash:` file
system on the IR809.  If the bundle image doesn't present, download it from
the TFTP server.  After downloading completed, it performs disk
re-partitioning for IOx guest OS and `bundle install` against the bundle
image.  With this process, all IOS and IOx are refreshed.

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

See `common.cfg` for more detail.

# Caveat

The `network-confg` in this demo doesn't work with factory reset procedure.
You have to perform factory reset against IR809 alone (without connecting
the TFTP server), then you have to connect the IR809 and reload or power
cycle it.

# Next step

For simplicity, this demonstration configure any IR809 to exact same
configuration.  You can differentiate configuration per unit basis using
serial number.

Replace following part of `network-confg`

```
 action 006.0 puts "Download startup-config start"
 action 006.1 cli command "copy tftp://192.168.0.253/common.cfg startup-config" pattern "Destination"
 action 006.2 cli command ""
 action 006.3 puts "Download startup-config done"
```

with following.

```
 action 006.0 puts "Download startup-config start"
 action 006.1 set serial "none"
 action 006.2 cli command "show version | include Processor board ID"
 action 006.3 regexp "Processor board ID ([0-9a-zA-Z]*)" "$_cli_result" dummy serial
 action 006.4 cli command "copy tftp://192.168.0.253/$serial.cfg startup-config" pattern "Destination"
 action 006.5 cli command ""
 action 006.6 puts "Download startup-config done"
 action 007   reload
```

Or, replace entire `network-confg` with `network-confg-with-serial`
(rename `network-confg-with-serial` to `network-confg`).

Now the EEM applet will download `tftp://192.168.0.253/<serial number>.cfg`
instead of `tftp://192.168.0.253/common.cfg`.  For example, if your serial
number is FCW2111004V, the applet downloads `FCW2111004V.cfg` from the
TFTP server.

## Using web server instead of TFTP server

In this demo, we solely used TFTP server for configuration source but you
can use HTTP server at the final stage.  The `copy` command of Cisco IOS
allows http URL schema such like `http://192.168.0.253/FCW2111004V.cfg`
for configuration source as well as TFTP so that you can develop custom
web application to generate configuration dynamically.  Though it can be done
with TFTP but it is much easier with HTTP.

# Additional notes for Mac users

Start temporary TFTP server on Mac.

```console
sudo launchctl load -w /System/Library/LaunchDaemons/tftp.plist
```

Default folder for TFTP server is `/private/tftpboot`.

Verify TFTP server is running.

```shell-session
$ sudo lsof -i:69
Password:
COMMAND PID USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
launchd   1 root    9u  IPv6 0x6e2ffc2a55c8e63d      0t0  UDP *:tftp
launchd   1 root   49u  IPv6 0x6e2ffc2a55c8e63d      0t0  UDP *:tftp
launchd   1 root   50u  IPv4 0x6e2ffc2a55c93515      0t0  UDP *:tftp
launchd   1 root   51u  IPv4 0x6e2ffc2a55c93515      0t0  UDP *:tftp
$
```

Stop TFTP server.

```console
sudo launchctl unload -w /System/Library/LaunchDaemons/tftp.plist
```

Install `isc-dhcpd` via brew.

```console
brew install isc-dhcp
```

You will find `/usr/local/etc/dhcpd.conf` for its configuration file.

Run DHCP server foreground.  Replace `en?` with your NIC.

```console
sudo /usr/local/sbin/dhcpd -f -d -cf /usr/local/etc/dhcpd.conf en?
```
